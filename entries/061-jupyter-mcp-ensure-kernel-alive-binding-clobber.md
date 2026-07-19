# A dead-kernel restart resets a notebook's server/token/path binding to config defaults

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/notebook_manager.py` (`ensure_kernel_alive`, `add_notebook`)
- **Class:** shared mutable state & partial-rebuild binding loss
- **Fix:** [PR #288](https://github.com/datalayer/jupyter-mcp-server/pull/288) (merged), [issue #287](https://github.com/datalayer/jupyter-mcp-server/issues/287)

## Root cause

`ensure_kernel_alive` regenerates a kernel when the current one has died. To
re-register the notebook it called `add_notebook(name, new_kernel)`, and
`add_notebook` unconditionally rebuilds the whole `notebook_info` entry, filling
`server_url`, `token`, and `path` from the config defaults whenever they are not
passed. The restart path did not pass them, so the real binding a caller set with
`use_notebook(name, path=...)` was overwritten by the default document's url,
token, and path.

The kernel was the only thing that needed to change. Rebuilding the entire entry
to swap it discarded the three fields that identify which notebook and server the
entry points at. After the restart, the next connection built from that entry
targets the default notebook, so subsequent read, edit, and execute operations
run against the wrong document while every call looks successful.

## Invariant violated

Regenerating a dead kernel must preserve the notebook's server_url/token/path
binding. A kernel restart changes only the kernel, never which notebook or server
the entry points at.

More generally: when a maintenance operation needs to mutate one field of a
shared record, mutate that field in place. Rebuilding the whole record through a
constructor that defaults the fields you did not pass turns a targeted kernel swap
into a silent reset of every field the constructor fills, and the binding fields
are exactly the ones a restart has no business touching. The correct pattern
already existed one file over in the restart tool, which re-passes `path`
deliberately; the liveness path did not mirror it.

## Trigger

A notebook is opened with a non-default path, its kernel dies, and any operation
routes through the liveness check that regenerates the kernel. The ordinary case
for a long-running session: kernels die from idle culling or crashes, and the
recovery path is meant to be transparent.

## Repro

`NotebookManager` is a plain in-memory dict, so no live kernel is needed: register
a notebook with a non-default path, mark its kernel dead, call
`ensure_kernel_alive`, and assert the entry's `path` is unchanged. The test fails
before the fix (path reset to the config default) and passes after. The fix swaps
only the kernel field in place when the notebook is already registered, falling
back to `add_notebook` otherwise. Both are in the linked PR.
