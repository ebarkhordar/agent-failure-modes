# A singleton captures its data root at construction and is constructed at import, so one reader follows the environment and its sibling is frozen on the path chosen before the process was configured

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/database/encrypted_db.py`, `DatabaseManager.__init__`
  (`:162`, `self.data_dir = get_data_directory() / "encrypted_databases"`) with the
  module-level singleton `db_manager = DatabaseManager()` (`:1260`, the file's last line),
  against `get_auth_db_path()` in `database/auth_db.py` (`:31-33`), which calls
  `get_data_directory()` fresh on every call
- **Class:** configuration wiring & documented contracts
- **Report:** reproduced publicly on
  [issue #5257](https://github.com/LearningCircuit/local-deep-research/issues/5257#issuecomment-5103100048)
  (open). Triage only, no PR from us: the reporter arrived with the correct root cause and a
  proposed patch and was fixing it himself, so the useful contribution was measuring what his
  repro and his patch actually do, not racing him to the diff.

## Root cause

`get_data_directory()` reads the `LDR_DATA_DIR` environment variable when it is called.
`DatabaseManager.__init__` calls it once, at `:162`, and stores the result on the instance.
The module then instantiates the singleton at `:1260`, so that one call happens at import.

Whichever event comes first wins for the life of the process, and nothing later can move it.
Import the module before the environment is set and the singleton is bound to the
platformdirs root permanently. That is not an exotic ordering: it is what a test suite does,
because collection imports the world and fixtures set the environment afterwards.

The damage is not that a setting is ignored. It is that the setting is honoured by some
readers and not others, so the process runs with two data roots at once. The auth database
resolves its path through `get_auth_db_path()`, a fresh `get_data_directory()` call, and
lands in the configured directory. Per-user encrypted databases resolve through
`_get_user_db_path()` (`:367-369`), which reads the cached `self.data_dir`, and land in the
platformdirs root. Measured in one process at `e5ff013c`:

```
at import time, data_dir = /root/.local/share/local-deep-research/encrypted_databases
LDR_DATA_DIR set to     = /tmp/tmpt02u1fv8
get_data_directory()    = /tmp/tmpt02u1fv8
get_auth_db_path()      = /tmp/tmpt02u1fv8/ldr_auth.db
db_manager.data_dir     = /root/.local/share/local-deep-research/encrypted_databases
_get_user_db_path(u)    = /root/.local/share/.../ldr_user_0fb265a4573777a0.db
```

The user rows say the account exists and the encrypted database backing it was written
somewhere the operator did not choose, which is why the reported symptom is an accumulation
of orphaned `ldr_user_<hash>.db` files under the platformdirs root rather than an error.

An AST classification of all 12 `get_data_directory()` call sites by enclosing scope found
two more module-level captures, `utilities/db_utils.py:19` and `web/models/database.py:11`.
Both bind a `DATA_DIR` constant with no reader anywhere in the repo, so both look dead, but
`web/models/database.py:15` runs `os.makedirs(DATA_DIR, exist_ok=True)`, which means merely
importing that module creates the platformdirs root as a side effect.

## Invariant violated

A value derived from mutable external state has to be resolved where it is used, or resolved
once at a moment the program chooses. Module import is neither. Import time is not a phase
the program controls: it is decided by the first statement anywhere in the process that
happens to name the module, which in a test suite is collection order. Binding configuration
there means the configuration API is really the import graph, and no caller can see it.

The failure mode worth generalising is what happens when only *some* readers cache. A setting
that is uniformly ignored is a visible bug: the feature does not work, someone files it on the
first run. A setting honoured by one path and cached-stale on another produces a process with
two answers to one question, and the paths that still work are what make it survive. Here the
login flow works, registration works, the tests pass, and the only evidence is files piling up
in a directory nobody looks at. So the rule is not "do not cache", it is that a cache of a
configured value must be the *only* reader of it. One resolver, called by everyone, or every
caller reading fresh. A mixture is strictly worse than either.

Two corollaries this case demonstrates, both with teeth:

**An ordering bug's reproduction has to pin the ordering, or it silently tests the benign
interleaving.** The repro in the issue sets `LDR_DATA_DIR` and then imports `db_manager`,
which is the one order in which `:162` captures the correct value. Run verbatim in a clean
container it prints the "Expected" line and passes on unfixed code. The reporter had the root
cause exactly right and had written a repro that could not show it, which is the normal
outcome when the trigger is an order rather than an input: the author knows what they meant,
and the script records what they typed. A regression test written from that script would be
green before and after the fix. The test has to assert the variable is unset, import, touch
the attribute, and only then set the variable.

**Do not key an idempotence latch on a boolean when it should be keyed on identity.** The
obvious repair, and the one proposed, is to make `data_dir` a lazy property behind a
`_data_dir_initialized` flag. That moves the capture without removing it: the latch records
*that* initialization ran, not *what* it ran against, so the first access still wins and the
property never revisits the correct directory. Key the guard on the resolved path and the
same code is correct.

The reason that repair is dangerous rather than merely incomplete is the second thing the
initializer does. `:163` creates the directory and `:170` chmods it to `0o700`, for the reason
the comment at `:164-169` gives: SQLite's WAL and SHM sidecars, and the plaintext unencrypted
fallback, are not covered by the per-file `0o600`. Meanwhile `create_user_database` (`:460`)
does its own `db_path.parent.mkdir(parents=True, exist_ok=True)` at `:480`. So when the latch
sends the initializer to the wrong path, the directory the process actually writes to still
gets created, by that downstream mkdir, at umask. The feature keeps working. Only the
hardening is lost. **When a lazy initializer performs both a creation and a hardening step,
and any downstream caller independently performs the creation, the hardening is the only part
that can go missing, and it goes missing silently:** no exception, no missing file, no failing
functional test, because every functional assertion is about the file existing and the file
does exist. Whether the directory is `0o700` or `0o755` is not something the feature's own
tests are looking at.

## Trigger

Any process that imports `local_deep_research.database.encrypted_db` before `LDR_DATA_DIR` is
set in the environment. Per-user encrypted databases and their `.salt` sidecars are then
written under the platformdirs data root for the life of the process, while the auth database
goes to the configured root. Test suites hit this by construction.

## Repro

Clean `python:3.12-slim`, `PYTHONPATH=src`, runtime dependencies installed (`loguru`,
`sqlalchemy`, `platformdirs`, `sqlalchemy-utc`, `pydantic`, `pydantic-settings`, `dynaconf`,
`toml`, `sqlcipher3-binary`, `alembic`), at HEAD `e5ff013c`, with the import placed first so
the ordering is the one under test:

```python
assert "LDR_DATA_DIR" not in os.environ
from local_deep_research.database.encrypted_db import db_manager
_ = db_manager.data_dir
os.environ["LDR_DATA_DIR"] = str(Path(tempfile.mkdtemp()).resolve())
print(db_manager._get_user_db_path("someuser"))
```

Both readers were driven in one process, so the split shown above is the same commit and the
same environment variable, differing only in which function resolved the path.

**Scope of what was measured.** The two-reader split, the reporter's repro printing the
"Expected" line on unfixed code, and the permission loss under the proposed patch were all
executed. The permission figures come from applying that patch, not from shipped code:
`0o755` patched against `0o700` unpatched, same container, umask 022. Applied verbatim the
patch does not import at all, since `:162` still assigns to what is now a property without a
setter, so `:162-163` and the `:170` chmod have to be removed before it runs, and the
measurement is of that corrected form.

Verified 2026-07-28 at HEAD `e5ff013c`. Reported, not fixed upstream at the time of writing.
