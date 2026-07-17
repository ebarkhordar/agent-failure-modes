# A structured output flattened to a string, then re-typed by sniffing its prefix

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/tools/execute_cell_tool.py::_write_outputs_to_cell`, `jupyter_mcp_server/utils.py::execute_via_execution_stack`
- **Class:** round-trip fidelity, type reconstruction
- **Fix:** [PR #278](https://github.com/datalayer/jupyter-mcp-server/pull/278) (merged), [issue #277](https://github.com/datalayer/jupyter-mcp-server/issues/277)

## Root cause

Type information was discarded one frame above the code that needed it, and then
guessed back from the rendered text. In JUPYTER_SERVER mode with the notebook not
open in a collaborative session, `execute_cell` writes results directly to the
`.ipynb`. `ExecutionStack` hands `execute_via_execution_stack` a list of
already nbformat-shaped outputs in `result["outputs"]`, each carrying its own
`output_type`. That function returns only `safe_extract_outputs(...)`, a list of
display strings, so the type is gone before the caller ever sees it.

`_write_outputs_to_cell` then reconstructs a type from the string alone: a
`[ERROR:`, `[TIMEOUT ERROR:` or `[PROGRESS:` prefix becomes `stream`, and
everything else becomes `execute_result`. The prefixes are the only evidence left,
and they are not evidence about type: `safe_extract_outputs` returns a bare
traceback with no `[ERROR:` prefix at all. So `print()` stdout persisted as
`execute_result`, and a traceback persisted as `execute_result`. No path on this
branch ever wrote `output_type: error`. The synthesized `execute_result` also
received `execution_count: None` while its own cell received `max + 1`.

## Invariant violated

Every output persisted to the `.ipynb` carries the `output_type` the kernel
reported for it. More generally: a value's type travels with the value or it is
lost. A serialization that keeps only the human-readable rendering has thrown
away the discriminant, and any later attempt to recover it is a heuristic over
presentation, not a decode. The correct data was in memory one frame up the whole
time.

## Trigger

Any cell containing `print()` or raising an exception, executed through the
Jupyter Server extension while no human has the notebook open in a browser tab.
`get_jupyter_ydoc` returns `None` outside a collaborative session, which selects
the file-mode write-back. This is the ordinary headless case, not a degenerate one.

## Repro

A real `python3` kernel driven through the real `ExecutionStack`, the real
`safe_extract_outputs` and the real `_write_outputs_to_cell`, ending at the file
on disk, nothing mocked:

```
                   kernel emitted   persisted   main     branch
print('hello')     stream           ->          WRONG    ok
1/0                error            ->          WRONG    ok
2                  execute_result   ->          ok       ok
```

The fix threads an optional `raw_outputs` list into
`execute_via_execution_stack`, which appends the kernel's nbformat-shaped outputs
to it. The return value, and so the MCP tool's public return value, is unchanged.
`_write_outputs_to_cell` persists those outputs as they arrived and deletes the
prefix-sniffing heuristic. The string fallback is kept for the timeout path,
which reaches the write-back with strings only.

## Note

`nbformat.validate` passes on both sides, before and after. Nothing was ever
schema-invalid, which is exactly why this survived: there is no validator to
point at and no crash to trace. The failure is silent and lands downstream, in
any tooling that keys on `output_type`. A notebook whose cells raised contains
no `error` output at all, so a papermill-style failure detector or an error
scraper reads it as a clean run.

The two comments in the codebase labelling this behavior a deliberate
simplification live in `utils.execute_cell_local`, which has zero callers in the
package. They describe a path nobody runs, so they were never evidence about the
live one. One function away, `execute_code_local` already built correctly typed
outputs from kernel messages: the repo's own code contained both the defect and
its answer, and the distinguishing fact was which of the two was reachable.
