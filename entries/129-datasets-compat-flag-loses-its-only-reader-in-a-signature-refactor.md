# A refactor about a signature deleted the only reader of a compatibility flag, so the writer still runs and the legacy cache key it exists to reproduce is silently wrong

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/utils/_dill.py:73-84` (`Pickler._batch_setitems`, the deleted early
  return) against `src/datasets/builder.py:494` (`DatasetBuilder._check_legacy_cache2`, the
  surviving `patch.object` writer)
- **Class:** configuration wiring & documented contracts
- **Report:** [issue #8410](https://github.com/huggingface/datasets/issues/8410) (open)
- **Fix:** [PR #8411](https://github.com/huggingface/datasets/pull/8411) (open, in review)

## Root cause

`Pickler._legacy_no_dict_keys_sorting` exists so that one caller can ask the hasher to behave
the way `datasets` 2.15.0 behaved. 2.15.0 hashed a dict without sorting its items; every release
since sorts them, so the same `data_files` dict hashes to a different value. `_check_legacy_cache2`
turns the flag on for exactly one call, to rebuild the `config_id` that 2.15.0 would have written
into the cache directory name:

```python
# src/datasets/builder.py:493-495
with patch.object(Pickler, "_legacy_no_dict_keys_sorting", True):
    config_id = self.config.name + "-" + Hasher.hash({"data_files": self.config.data_files})
```

PR #7817 rewrote `_batch_setitems` to take the extra argument Python 3.14 passes, and in the same
hunk removed the two lines that read the flag:

```diff
-    def _batch_setitems(self, items):
-        if self._legacy_no_dict_keys_sorting:
-            return super()._batch_setitems(items)
+    def _batch_setitems(self, items, *args, **kwargs):
         # Ignore the order of keys in a dict
```

That PR is about the 3.14 signature and says nothing about the legacy branch. What is left is a
flag with a class-level default, a writer, and nobody reading it. Both of its two remaining
occurrences in `src/` look load-bearing to a reader, and neither is a read:

```console
$ grep -rn '_legacy_no_dict_keys_sorting' src/
src/datasets/builder.py:494:            with patch.object(Pickler, "_legacy_no_dict_keys_sorting", True):
src/datasets/utils/_dill.py:30:    _legacy_no_dict_keys_sorting = False
```

`_check_legacy_cache2` builds `{cache_dir}/{namespace}___{name}/{config_id}/0.0.0/{hash}` from
that `config_id` and returns it only `if os.path.isdir(...)`. A `config_id` that no longer matches
what 2.15.0 wrote means the directory is not found, the lookup misses, and the caller proceeds to
build the dataset again.

## Invariant violated

**A flag is a contract between a writer and a reader, and the language checks neither end.**
`patch.object` is the one tool here that checks anything: without `create=True` it raises
`AttributeError: <class> does not have the attribute` when the name is absent, which is why
deleting the class-level default at `_dill.py:30` would have broken the writer loudly. Deleting
the reader does not, because the assertion `patch.object` makes is that the attribute exists,
never that anybody consumes it. An unused local is a lint error in most projects. An unused class attribute written
through a context manager in another module is invisible to every tool the project runs, so the
half-deleted feature reports as healthy code, and the `with` block still reads as if it does
something.

**A refactor scoped by a signature still edits the body, and review attention follows the stated
subject.** #7817 is a Python 3.14 compatibility change: the argument list is the subject, and a
reviewer checking it verifies that `*args`/`**kwargs` reach `super()`. The early return sat in
the first two lines of the same body, inside the edit's blast radius and outside its topic. A
diff cannot mark which of its lines are incidental, so the only defence is reading the hunk
rather than the intent, and the intent is what the title, the description and the reviewer's
attention are all pointed at.

**Compatibility code is dead code from the test suite's point of view.** `_check_legacy_cache2`
only does anything when a 2.14 or 2.15 era cache directory is already on disk. A test run starts
with an empty cache directory, so the branch is either never entered or enters and correctly finds
nothing, and it produces the same green result whether the flag works or not. The trigger is a
state that the environment verifying the code never constructs, which is the same shape as a
dependency cap that no test can check because resolution happens at install time. Coverage over
the function is not the property that helps; the missing input is.

**The failure is a miss and not an error, so it costs the user time and not a traceback.** The
lookup returns `None`, the caller falls through to a normal build, and the dataset is regenerated
or re-downloaded. Nobody sees a stack trace, nobody files a bug, and the only symptom is that an
upgrade appears to have thrown away a cache that is still sitting on disk under a name nothing
computes any more.

## Trigger

Upgrading from `datasets` 2.14/2.15 to 4.4.0 or later with an existing cache and a `data_files`
dict whose keys are not already in sorted order. `{"train": ..., "test": ...}` is the ordinary
case and is not sorted. 4.4.0 is the first release containing #7817 (`git tag --contains
f7c8e46ec`), so the behaviour changes exactly at that boundary.

## Repro

One venv per version on Python 3.11 in a clean container, so no interpreter difference can
confound the comparison:

```python
from unittest.mock import patch

from datasets.fingerprint import Hasher
from datasets.utils._dill import Pickler

obj = {"train": ["train.csv"], "test": ["test.csv"]}
print("normal:", Hasher.hash(obj))
with patch.object(Pickler, "_legacy_no_dict_keys_sorting", True):
    print("legacy:", Hasher.hash(obj))
```

| datasets | `normal` | `legacy` | matches 2.15.0 |
|---|---|---|---|
| 2.15.0 | `711511d8f1d9bc25` | n/a | n/a |
| 4.3.0 | `cfd3b51f0f8e9fd8` | `711511d8f1d9bc25` | yes |
| 4.4.0 | `cfd3b51f0f8e9fd8` | `cfd3b51f0f8e9fd8` | no |
| main @ `b7cb10b0` | `cfd3b51f0f8e9fd8` | `cfd3b51f0f8e9fd8` | no |
| main @ `b7cb10b0` + the two restored lines | `cfd3b51f0f8e9fd8` | `711511d8f1d9bc25` | yes |

The differential ran in `python:3.11-slim` against the bind-mounted clone, `datasets`
5.0.2.dev0 with `__file__` under the mount, `dill` 0.4.1. With the two lines stripped and the
new tests present: 2 failed, 1 passed. With them restored: 3 passed. `tests/test_fingerprint.py`
went from 24 passed and 9 skipped to 26 passed and 9 skipped, with nothing regressing.

**Not verified:** no real 2.15.0 cache directory was built and loaded under HEAD. The hash
mismatch is executed; the consequence for the directory lookup is read from
`_check_legacy_cache2`, where the path is built from `config_id` and returned only if
`os.path.isdir` succeeds. The issue and the PR both say so.

Verified 2026-08-06 at `b7cb10b0`. Reported and a fix offered; the issue also offers the other
resolution, retiring the 2.14/2.15 compat path outright, since which one is right is a decision
about how long that cache layout is worth carrying.
