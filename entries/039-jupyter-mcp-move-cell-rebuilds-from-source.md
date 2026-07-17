# A move implemented as a re-creation keeps only the fields the constructor takes

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/tools/move_cell_tool.py::_move_cell_ydoc` and
  `::_move_cell_websocket`
- **Class:** tool side-effect scope
- **Fix:** [PR #273](https://github.com/datalayer/jupyter-mcp-server/pull/273)
  (merged 2026-07-17; issue [#272](https://github.com/datalayer/jupyter-mcp-server/issues/272))

## Root cause

`move_cell` has three backends implementing one contract, and two of them
implement it as a re-creation rather than a reorder:

```python
deleted = nb.delete_cell(source_index)
cell_type = deleted.get("cell_type", "code")
cell_source = deleted.get("source", "")
nb.insert_cell(target_index, cell_source, cell_type)
```

`NotebookModel.insert_cell(index, source, cell_type)` routes to
`nbformat.v4.new_code_cell(source)`, so the rebuilt cell has exactly the two
fields that were passed in. Everything else the `deleted` node still holds is
dropped: `outputs`, `execution_count`, `metadata` and tags, attachments, and the
cell `id`. `_move_cell_file` reorders the cell object itself (`_apply_move`),
which is why the file backend was never affected.

## Invariant violated

A move changes a cell's position and nothing else. Source, cell type, outputs,
execution count, metadata and attachments survive, identically on all three
backends.

The mechanism generalizes cleanly: delete-then-insert is a move only if the
insert accepts everything the delete returned. The signature of the re-insertion
primitive **is** the list of what survives. `insert_cell(index, source,
cell_type)` cannot preserve outputs, because it has no parameter to receive them.
So the data loss is legible in the call signature and completely invisible in the
control flow, which is correct: the cell is removed from one index and appears at
another, which is what a move does. Reading the function tells you nothing.
Reading what the constructor accepts against what the object holds tells you
everything.

This is why the entry sits under tool side-effect scope alongside
[027](027-jupyter-mcp-use-notebook-ui-focus-steal.md). Both are an operation whose
real effect exceeds its name: there a tool reached out to the UI its caller never
asked it to touch, here a reorder destroys state its caller never asked it to
discard. The question that finds both is the same one, asked about the verb in the
tool's name: what else, besides that verb, does this actually do.

## The test that named the invariant and asserted a tautology

`test_move_cell_preserves_outputs` already existed, with the docstring "Moving a
cell that has been executed should preserve its outputs." It could not fail.
`read_cell` returns `[header, source, *outputs]`, the test joined the whole result
into one string and asserted `"hello" in cell_text`, and the cell's source is
literally `print('hello')`. The assertion was matching the source, which a move
never loses.

The generalizable rule: an assertion that searches for a needle in a haystack
containing the input is not testing the output. It is testing that the input was
echoed. The intent here was recorded correctly, in the test name and in the
docstring, and the intent is not what runs. A test whose subject and whose fixture
share a string can pass for the wrong reason forever, and the giveaway is
available statically: the expected value appears in the setup.

The replacement unpacks the outputs region and compares against a read taken
before the move. Execution count is compared before against after rather than
pinned to a literal, because the kernel's counter depends on what ran earlier in
the session: a hardcoded `1` passes in `MCP_SERVER` mode and fails in
`JUPYTER_SERVER` mode for reasons unrelated to this bug.

## Why the existence proof is not the whole claim

A fails-on-main test shows the payload now survives the path it runs. It cannot
establish "nothing else rebuilds a cell", which is a claim over all code. That
half was settled by parsing rather than by running: calls to the lossy primitive
`NotebookModel.insert_cell(...)` in the package are exactly four.

| call site | verdict |
|---|---|
| `insert_cell_tool.py:85` (`_insert_cell_ydoc`) | correct: creating a genuinely new cell from source, nothing to preserve |
| `insert_cell_tool.py:177` (`_insert_cell_websocket`) | correct: same |
| `move_cell_tool.py:91` (`_move_cell_ydoc`) | the bug, fixed |
| `move_cell_tool.py:152` (`_move_cell_websocket`) | the bug, fixed |

`delete_cell(...)` has exactly two call sites, both in `move_cell_tool.py`, both
fixed. So the complete set of places that can destroy a cell payload on a move is
the two lines the PR changes.

## Trigger

Any `move_cell` in ydoc or websocket mode on a cell that carries anything beyond
its source: outputs, an execution count, metadata, tags, or attachments. No
configuration, no concurrency.

## Repro

Docker `python:3.12-slim`, non-editable `pip install .`, module provenance printed
on both sides, source sha256 `4e50c846...` on main against `20e73f9d...` on the
branch. Live end-to-end against a real Jupyter server and a real ipykernel, on
main, `MCP_SERVER` mode:

```
BEFORE read_cell(1): ['=====Cell 1 | type: code | execution count: 1=====', "print('hello')", 'hello\n']
AFTER  read_cell(2): ['=====Cell 2 | type: code | execution count: N/A=====', "print('hello')"]
```

On the branch the same sequence returns `execution count: 1` and the `hello`
output at the new index. Both modes CI runs: 39 passed each. Reverting only the
source file and keeping the tests fails `test_move_cell_preserves_outputs` and
`TestMoveCellPreservesPayload` (`assert 'hello' in ''`, and `assert [] ==
[{...'text': 'hello\n'...}]`). The added unit tests mock only the connection
manager: `delete_cell` and `insert` are the real `NotebookModel` methods, so no
writer of the cell payload is mocked out.

**Not verified:** `_move_cell_ydoc` was not executed. A headless run creates no
collaborative room, so `get_notebook_model()` returns `None` and the tool
correctly falls back to the file path. Its identical twin `_move_cell_websocket`
was exercised over the same `NotebookModel` type, and `_move_cell_ydoc` was read
by eye. Both call sites carry the same five-line block, but one is reported from
execution and one from reading. Attachments on markdown cells are read from the
code (`new_markdown_cell(source)` takes no attachments), not measured.

Not a regression: `move_cell` has a single commit (`c45d8ec`, "Add move_cell tool
(#226)") and has not been touched since, so this is original behavior.

## Fix

Reinsert the deleted cell through `NotebookModel.insert(index, value: dict)`,
which hands the whole cell to `YNotebook.create_ycell(value)`. That path rebuilds
source, metadata and outputs as CRDT types and keeps every other key, including an
incoming `id`. Four insertions and four deletions across the two call sites, no
public API change and no new dependency.

Preserving the cell `id` is a behavior change that was flagged rather than
assumed: today a move re-mints it, the fix keeps it on the grounds that a move
should not change cell identity, and the maintainer was asked to say if they want
the opposite.
