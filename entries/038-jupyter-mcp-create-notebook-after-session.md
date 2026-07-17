# A skeleton only one branch writes, and a session pointed at a file that does not exist yet

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/tools/use_notebook_tool.py::UseNotebookTool.execute`,
  the `JUPYTER_SERVER` branch of `use_mode="create"`
- **Class:** initialization & control flow
- **Fix:** [PR #271](https://github.com/datalayer/jupyter-mcp-server/pull/271) (merged 2026-07-17)

## Root cause

One tool, two backend branches that are meant to be equivalent, and the
`JUPYTER_SERVER` one gets two things wrong.

**The skeleton is discarded.** Above the branch, `execute` builds a `content`
dict holding a markdown cell ("New Notebook Created by Jupyter MCP Server"). The
`MCP_SERVER` branch passes it on
(`server_client.contents.create_notebook(notebook_path, content=content)`). The
`JUPYTER_SERVER` branch called `await contents_manager.new(model={'type':
'notebook'}, path=notebook_path)`, which never references `content` at all. On
that branch the variable is dead, so the same tool call produced a different
notebook depending on the mode.

**The file is created last.** `_start_kernel_local(..., path=notebook_path)` and
then `session_manager.create_session(path=notebook_path, ...)` both ran before the
file was written, so a Jupyter session briefly existed for a notebook path that
did not exist yet. On `main` the recorded call order is literally
`['start_kernel', 'create_session', 'new']`.

## Invariant violated

In create mode the notebook file exists, with its intended content, before
anything is pointed at its path.

The generalizable half is about where to look when one operation has two backend
implementations. The value at risk is the one constructed **above** the branch.
`content` is built once, before the split, which is what makes both branches read
as if they use it, and the fact that one of them does use it is what makes the
other's silence invisible. A local that is live in one arm and ignored in the
other draws no warning from any linter, because from the linter's point of view
the variable is used. So the parity check cannot be "is this value used", it has
to be per-branch: for each value computed before a fork, does every arm consume
it, and if not, is that deliberate.

The ordering half generalizes as: creating a reference to a resource before the
resource exists is not made safe by the gap being short. Here the file does
appear a moment later, and under a standard backend nothing observably breaks,
which is precisely the condition under which such an ordering survives for years
and then fails on a backend that reads the path eagerly.

## Trigger

`use_notebook(use_mode="create")` in `JUPYTER_SERVER` mode. Every such call
produces a notebook with no cells, where the same call in `MCP_SERVER` mode
produces the intended one-cell skeleton.

## Repro

Clean `python:3.12-slim` container on `main` at `2113db0`, with
`print(module.__file__)` confirming the code under test was the checkout and not
an installed copy.

`tests/test_use_notebook_create_local.py` drives
`UseNotebookTool().execute(mode=ServerMode.JUPYTER_SERVER, ...)` with in-memory
recording stand-ins for the contents, kernel and session managers, so it needs no
live server. Both tests fail on `main` and pass on the branch; the recorded call
order quoted above is what one of them asserts.

The consequence of the dropped skeleton was checked against real `jupyter_server`
2.20.0 `AsyncFileContentsManager` rather than read from its source:

- `new(model={'type': 'notebook'})` writes a 72-byte notebook with **0 cells**
- `new(model={'type': 'notebook', 'content': ..., 'format': 'json'})` writes a
  198-byte notebook with the **1** skeleton cell

Existing suites that run without a live server are unchanged: 82 passed, 1
skipped. The 52 errors in that run need a live server and are identical on `main`
and on the branch.

**Not verified, and this is the important limit:** the JupyterLab hang reported in
[#246](https://github.com/datalayer/jupyter-mcp-server/issues/246) could not be
reproduced (it needs JupyterHub plus `jupyter_server_documents` and jupyter-ai
3.0), so this entry makes **no claim** that the fix addresses that hang. Under
standard `jupyter_server` the old code writes a valid, if empty, notebook rather
than the 0-byte file described in that thread, so the 0-byte observation looks
backend-specific and the causal chain was not confirmed. What is demonstrated
here is the mode disparity (0 cells against 1) and the call order. #246 is the
report that led to the code, not something this entry says was explained.

The `MCP_SERVER` branch, where the reordering also applies, was not exercised
against a live server. One consequence of the reorder was flagged upstream for
the maintainer's call: if kernel startup now fails, the created file is left
behind, where previously it would not have been created.

## Fix

Move the `if use_mode == "create":` block above the kernel and session block, and
pass the skeleton on the local branch via `model={'type': 'notebook', 'content':
content, 'format': 'json'}`. The block moves unchanged apart from that model
argument.
