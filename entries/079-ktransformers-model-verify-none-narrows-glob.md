# Passing the "no specific files" sentinel to a hasher makes it fall back to a `*.safetensors`-only default, so a first-time verify reports every `.json`/`.py` as missing on a healthy model

- **Repo:** kvcache-ai/ktransformers
- **Surface:** `kt-kernel/python/cli/commands/model.py` (the `files_list=files_to_hash if files_to_verify else None` call, around lines 2453-2458) and `kt-kernel/python/cli/utils/model_verifier.py::calculate_local_sha256` (default `file_pattern="*.safetensors"`), reached from `verify_model` and `pre_operation_verification`
- **Class:** error handling & success reporting
- **Fix:** [PR #2104](https://github.com/kvcache-ai/ktransformers/pull/2104) (in
  review; issue [#2100](https://github.com/kvcache-ai/ktransformers/issues/2100))

## Root cause

`kt model verify` builds the set of files it means to hash as
`files_to_hash` over the patterns `['*.safetensors', '*.json', '*.py']`, then hands
it to the hasher as:

```python
files_list = files_to_hash if files_to_verify else None
```

`files_to_verify` is the optional list of specific files for a *re-verification*.
On a first verify it is `None`, so this expression passes `None`, meaning "no
explicit list", and `calculate_local_sha256` then falls back to its own default,
`file_pattern="*.safetensors"`. So exactly when the caller has computed the full
three-pattern set, it discards it and the hasher covers only the weight shards. The
same defaulting appears in `verify_model_integrity_with_progress` and in
`pre_operation_verification`, which runs before every `run` / `quant`.

The remote side, `fetch_model_sha256`, returns hashes for `.safetensors`, `.json`,
and `.py`. When the local map holds only safetensors, every remote `.json` and
`.py` has no local counterpart, so verification reports them as missing and marks a
perfectly healthy model `sha256_status="failed"` with a "weights may be corrupted"
message. A second defect compounds it: the local map was keyed by basename, so
`config.json` and `inference/config.json` collide, which is why the reporter's 16
missing files span subdirectories that a top-level glob could never have reached.
The sibling `verify_model_integrity` enumerates all three patterns correctly, which
is strong evidence the narrow default here was unintended rather than a design
choice.

## Invariant violated

When a helper accepts an optional set and falls back to a *narrower* default for
the unspecified case, a caller that has already computed the broad set must pass it
explicitly; passing the "unspecified" sentinel to mean "use everything I care
about" instead invokes the narrow default and silently checks a subset. The sentinel
means "I have no opinion, use your default," and the caller here has a very strong
opinion (three patterns) precisely on the path where it passes the sentinel. The
second rule is about keys: when file identities can repeat across directories, a
map must be keyed by repository-relative path, not basename, or entries from
different directories overwrite each other. Together they turn a health check into
a false alarm, the most corrosive kind of verification bug, because it teaches the
user to distrust a tool that is telling the truth about a model that is fine.

## Trigger

`kt model verify <model>` (or any `run` / `quant`, which calls
`pre_operation_verification`) on a healthy model whose directory contains `.json` /
`.py` files alongside the safetensors shards, run without an explicit re-verify
list. The command reports the non-weight files as missing and the model as
corrupted.

## Repro

Reproduced this session in a clean `python:3.11-slim` container against the fault
commit `d1a3ed8`, using the fix's `per_commit/test_model_verifier_relpath.py`
applied over the unfixed `model_verifier.py`: the local hash map comes back keyed
`{"config.json": ...}` with the subdirectory files absent, so
`assertIn("inference/config.json", local)` fails and the suite reports 1 failure and
4 errors. On the fix branch (recursive `rglob` plus repository-relative-path
matching, applied to the two reachable consumers) the same five tests pass. The two
`verify_model_integrity*` functions carry the same defect but are imported and never
called, so they were left untouched and noted as dead rather than claimed fixed.
