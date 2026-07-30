# A guard added for one pathological input is written against that input's exception type, so its sibling escapes and is reclassified from recoverable to fatal

- **Repo:** UKGovernmentBEIS/inspect_ai
- **Surface:**
  `src/inspect_sandbox_tools/src/inspect_sandbox_tools/_in_process_tools/_text_editor/text_editor.py:193`
  (`Path(path_str).resolve()`) and `:211` (`except OSError`), with the classification at
  `_util/json_rpc_helpers.py:91` and `:97` and the host-side mapping at
  `_error_mapper.py:24` and `:26`
- **Class:** error handling & success reporting
- **Report:** triaged publicly on
  [issue #4566](https://github.com/UKGovernmentBEIS/inspect_ai/issues/4566#issuecomment-5035919858)
  (filed by a maintainer, closed as completed).
- **Fix:** [PR #4659](https://github.com/UKGovernmentBEIS/inspect_ai/pull/4659) (merged
  2026-07-29), which widens the clause to `except (OSError, ValueError)` and pins the
  null-byte case with a regression test beside the existing invalid-path ones.

## Root cause

`_validated_path` exists to turn a bad model-supplied path into a tool error the model can
read and correct. It resolves the path inside a `try` and catches `OSError`, which was the
right type for the case it was written for: an over-long filename raises `OSError`
(`ENAMETOOLONG`), and `test_validated_path_rejects_too_long_filename` pins that.

A path containing a NUL byte does not raise `OSError`. `Path.resolve()` raises `ValueError`,
which is not an `OSError` subclass, so the clause does not see it:

```
[too-long-path]  ToolException: "Invalid path 'aaaa...'"                        # caught
[null-byte-path] ValueError ESCAPED: 'lstat: embedded null character in path'   # not caught
Path.resolve raised: ValueError | isinstance OSError: False
```

What the two paths become is the whole defect. A `ToolException` is serialised as JSON-RPC
`-32099` and mapped host-side to `ToolError`, which is handed back to the model as a
correctable result. Anything else takes the `except Exception` branch, becomes `-32098`, and
is mapped to `RuntimeError`, which kills the sample. So one character in a path decides
between "the model is told its path was invalid and tries again" and "the run dies", and the
line that decides it names an exception class rather than a category of input.

## Invariant violated

**An `except` clause is a policy statement about a set of exception types, while the code
around it was reasoned about as a set of inputs, and the two sets coincide only by
accident.** The author's intent here reads clearly in the message it raises: reject invalid
paths. Invalid paths are an input class. `OSError` is a type, and the stdlib picks which
type each invalid path produces, on rules that have nothing to do with this function.
Anything the standard library chooses to signal with a different type falls straight through
a guard that looks, in review, like it covers the case. Enumerate the exception types the
calls inside the `try` can raise, not the scenarios you had in mind when you wrote it.

**At a boundary that sorts failures into recoverable and fatal, an uncaught exception is not
lost, it is promoted.** The usual worry about a missing catch is a swallowed error. Here the
error is loud, and that is what makes it worse: an RPC layer with a default `except
Exception` branch will faithfully deliver it as the most severe category it has. A gap in a
narrow catch upstream therefore reappears downstream as a maximum-severity verdict, which is
the opposite of what the narrow catch was trying to achieve. The exception taxonomy is the
recoverability contract, and every unhandled type defaults to the harshest end of it.

A smaller note with a general edge: the `ValueError` message wording is platform-dependent
(`embedded null character in path` here, `embedded null byte` in the reporter's traceback)
while the type is stable across both. A fix that matched on message text would have been
correct on one platform and silently ineffective on the other.

## Trigger

A model-supplied path containing a NUL byte reaching `text_editor()`. Deliberate abuse is
the obvious route, but the ordinary one is an accident: a truncated tool-argument string, a
decoding artifact, or a path assembled from a binary read. Nothing in the tool's schema
rejects it earlier.

## Repro

Docker `python:3.12-slim`, only `inspect_sandbox_tools` installed, at `main`
`a85e2ddbc1b5d1101696e8d1c57c8a4f5c67bcba`. `_validated_path` was called directly with the
two pathological paths and with a control:

```python
_validated_path("a" * 5000, "view")          # too-long  -> ToolException
_validated_path("/repo/foo\x00bar", "view")  # null byte -> ValueError escapes
```

The unit doubles as the regression test, since after the widening the second call must raise
`ToolException`.

**Not verified, and this bounds the entry:** the JSON-RPC hop and the host-side mapping were
read at the lines cited, not driven end to end, so the `-32098` to `RuntimeError` leg is a
static claim about the code path rather than an observed run. `bash_session` was searched for
the same shape and no `Path(...).resolve()` on a model-supplied path was found there, but it
was not exercised.

Verified 2026-07-21 at `a85e2ddb`. Fixed upstream by PR #4659, merged 2026-07-29.
