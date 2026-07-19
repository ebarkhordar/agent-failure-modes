# A guard that asks a question the cancellation it just issued has not answered yet

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/tools/execute_cell_tool.py::ExecuteCellTool.execute`,
  the `stream=True` branch
- **Class:** initialization & control flow
- **Fix:** [PR #276](https://github.com/datalayer/jupyter-mcp-server/pull/276) (merged)
  (merged 2026-07-17; issue [#275](https://github.com/datalayer/jupyter-mcp-server/issues/275))

## Root cause

Two defects at the end of one branch. Both are absent from the non-streaming
branch of the same function, which is what marks them as oversights rather than
intent.

**The timeout discards its own log.** The monitoring loop cancels the execution
task, appends `[TIMEOUT at ...s: Cancelling execution]`, and breaks. The
completion block below is then guarded by:

```python
if not execution_task.cancelled():
    ...
    await execution_task
```

`Task.cancelled()` is `False` immediately after `cancel()`, because the task has
not processed the cancellation yet. The guard therefore always passes on the very
path it was written to exclude, and the `await` inside it raises `CancelledError`.
That derives from `BaseException`, so the `except Exception` below does not catch
it: it leaves `execute()`, the timeout log is dropped, and `AFTER_EXECUTE` never
fires. The non-streaming branch instead returns partial outputs plus a
`[TIMEOUT ERROR: ...]` marker.

**The terminal drain assumes a str.** Outputs landing between the last one-second
poll and completion are drained with `if extracted.strip():`. `extract_output` is
declared `-> Union[str, ImageContent]` and returns `ImageContent` for `image/png`
when `ALLOW_IMG_OUTPUT` is on, the default. The `AttributeError` is swallowed by
the same `except Exception`, so the image is replaced by
`[ERROR: 'ImageContent' object has no attribute 'strip']`, which the caller cannot
tell apart from a real execution error. The monitoring loop 28 lines above already
type-checks the same union with `isinstance(extracted, str)`.

## Invariant violated

A timeout returns the partial log it collected. An output that survives the
monitoring loop survives the terminal drain.

Underneath both: **cancellation is a request, not a state transition.**
`cancel()` schedules a `CancelledError`; `cancelled()` reports whether the task
has already finished by absorbing one. Between those two facts is the entire gap
this bug lives in. The guard reads as a question about what the code just did
(`did I cancel this?`), but it asks what the *task* has done about it (`has it
finished cancelling?`), and the answer at that line is always no, because nothing
has awaited it yet. The code holds the answer already, in the branch it just took.
It threw that away and asked the object instead, which cannot know yet.

That is the general shape: a guard that re-derives a fact the control flow already
established, by interrogating a collaborator that learns it strictly later. It is
always dead code, and it is invisible, because the guard names the right concept.
The fix is not a better predicate but a local flag (`timed_out`), which is just
the control flow remembering what it did.

The second defect generalizes the other way: a union return type is a contract
that every consumer must discharge, so the count of consumers is the count of
places to check. This function had two, the poll and the drain, and only the poll
was written against the declaration. A `Union[str, X]` narrowed by calling a `str`
method is not narrowing, it is a bet, and `except Exception` pays it off silently.

## Trigger

- Defect 1: `execute_cell(stream=True)` on a cell that runs past `timeout_seconds`.
- Defect 2: `execute_cell(stream=True)` on a cell whose image output lands after
  the final poll, which a fast `plt.show()` does.

Neither is reachable with `stream=False`. `grep -rn 'stream=True' tests/` returned
nothing before this fix: the flag is user-facing (`server.py:618`) and had no test
coverage, which is why both survived the three refactors that moved this code
(#28 introduced the guard, #95 and #111 moved it).

## Minimal repro

Verified on HEAD `4d07938d` in a clean `python:3.12-slim` container, driving the
real `ExecuteCellTool.execute(mode=ServerMode.MCP_SERVER, stream=True, ...)` with
in-memory stand-ins for the kernel and the notebook connection, so the control
flow and return value under test are the real code.

Defect 2, a cell whose only output is an `image/png` at completion:

```
RESULT: ['[PROGRESS: 0.0s elapsed, 0 outputs so far]',
         '[COMPLETED in 1.0s]',
         "[ERROR: 'ImageContent' object has no attribute 'strip']"]
image preserved? False
```

Defect 1, a cell that runs past `timeout_seconds`:

```
*** RAISED CancelledError ***
timeout log returned? False
```

The primitives behind defect 1, printed from the same container:

```
cancel() -> True
cancelled() immediately after cancel() -> False
await task RAISED: CancelledError
isinstance(CancelledError(), Exception) -> False
```

## Fix

Track the timeout in a local `timed_out` flag and gate the completion block on it,
so the timeout path returns `outputs_log` like the non-streaming branch. Mirror the
monitoring loop's `isinstance` check on the drain path so `ImageContent` passes
through, keeping the empty-string filter for strings. Nine lines, one function, no
signature change and no behavior change on the paths that already worked. Two
regression tests in `tests/test_execute_cell_stream.py`, one per defect, both
failing on main and passing on the branch.

## Related

[028](028-jupyter-mcp-execution-timeout-unwired.md) is the same timeout surface:
there the configured timeout reached no code that read it, here the timeout fires
and throws away what it collected. A setting is only as real as the path that
consumes it, and that path had never been executed by a test either.
