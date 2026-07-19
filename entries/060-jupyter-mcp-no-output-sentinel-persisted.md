# A display sentinel for the tool response is persisted as a real cell output

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/tools/execute_cell_tool.py::ExecuteCellTool._write_outputs_to_cell`
  (the `raw_outputs`-empty fallback branch), with the sentinel produced at
  `jupyter_mcp_server/utils.py:562`
- **Class:** message-conversion boundaries
- **Fix:** [PR #286](https://github.com/datalayer/jupyter-mcp-server/pull/286) (merged)
  (merged 2026-07-19; issue [#285](https://github.com/datalayer/jupyter-mcp-server/issues/285))

## Root cause

In `JUPYTER_SERVER` file mode, a cell that emits nothing (`x = 1`, an `import`, a
`def`) makes `execute_via_execution_stack` return the display sentinel
`["[No output generated]"]` while leaving `raw_outputs` empty. `_write_outputs_to_cell`
then takes its `raw_outputs`-empty fallback, which writes the formatted strings
back to the cell. That branch only special-cases strings beginning with `[ERROR:`,
`[TIMEOUT ERROR:`, or `[PROGRESS:`; the `[No output generated]` sentinel matches
none of them, so it was written as an `execute_result` with
`data={'text/plain': '[No output generated]'}` and stamped with the cell's
execution count. A cell that produced nothing was persisted to the `.ipynb` with a
fabricated result output, visible on every reopen.

The sentinel exists for one channel only: it is the message the MCP tool hands
back to the calling agent so the response is not empty. It was never meant to
enter `cell.outputs`, which is the persisted notebook document. The two channels
share one string and the fallback did not separate them.

## Invariant violated

A string produced for the tool's response is not the same artifact as the cell's
persisted output. The sentinel that tells the caller "nothing came back" must not
survive into the document as if a value had come back. An output-less cell
persists no output.

The mechanism is the same one that puts this entry next to
[044](044-jupyter-mcp-output-type-sniffed-from-string.md): a value crosses a
boundary carrying meaning that is valid on one side and forbidden on the other.
There a string was sniffed to guess an output *type*; here a string minted for
display is persisted as an output *value*. Both are legible only by asking what
the string was *for*, because on the persistence side it is a well-formed
`execute_result` that nothing downstream can tell apart from a genuine one.

## Residual of a prior fix

This is not a regression from unrelated work. [#278](https://github.com/datalayer/jupyter-mcp-server/pull/278)
fixed the real-output fidelity path in this exact function and *intentionally* kept
the string fallback for the error, timeout, and progress cases, which are correct:
those strings carry information the caller needs and are the best available
representation when `raw_outputs` is empty. Only the empty-output sentinel was
wrong to persist. A fix that widened the guard blindly would have dropped the
error and timeout strings too, so the change had to name the one string to exclude
rather than the ones to keep.

## Fix

In the fallback branch, skip the `[No output generated]` sentinel so an
output-less cell persists no output. The sentinel is still returned to the caller
(the tool response is unchanged), and the cell still receives its execution count.
The `[ERROR:`, `[TIMEOUT ERROR:`, and `[PROGRESS:` strings are untouched.

## The scenarios were pinned one test each

The PR body named three behaviors, and each got its own assertion rather than one
test standing in for the set:

| scenario | test |
|---|---|
| empty sentinel NOT persisted | `test_no_output_sentinel_is_not_persisted` (sentinel yields `cell.outputs == []`, execution count still set; fails on `main`, passes on the branch) |
| error string STILL persisted | `test_error_string_still_persisted_without_raw_outputs` (an `[ERROR: ...]` string is still written as a stream output) |
| timeout string STILL persisted | the existing `test_falls_back_to_formatted_strings_without_raw_outputs` |

The point of the second and third is that the fix is a subtraction, and a
subtraction has to prove it took away only the one thing. A single test on the
empty case would have been consistent with a change that silently dropped the
error and timeout strings as well.

## Trigger

Any `execute_cell` in `JUPYTER_SERVER` file mode on a cell that produces no
output. No configuration beyond that mode, no concurrency.

## Repro

Docker `python:3.12-slim`, `pip install -e .`, on HEAD `a043d85`. Driving
`execute_via_execution_stack` with an ExecutionStack that reports no outputs, then
persisting through `_write_outputs_to_cell`: the `.ipynb` cell gained an
`execute_result` carrying `[No output generated]`. With the fix, `cell.outputs == []`.
The full `tests/test_execute_cell_output_fidelity.py` module passes (12 tests).

**Not verified:** not driven end-to-end against a live kernel. The empty-cell to
sentinel step (`utils.py:562`) is exercised with a stub ExecutionStack rather than
a real ExecutionStack response, and the persisted-output defect is exercised
directly through `_write_outputs_to_cell`, which is where the sentinel was written.
