# to_thread(coroutine_fn) returns an un-awaited coroutine, so a sync wrapper silently breaks on the default async ContentsManager

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/jupyter_extension/backends/local_backend.py`, the
  nine `self.contents_manager.{get,new,save}` call sites
- **Class:** sync/async boundary
- **Fix:** [PR #284](https://github.com/datalayer/jupyter-mcp-server/pull/284) (merged)
  (merged 2026-07-18; issue [#283](https://github.com/datalayer/jupyter-mcp-server/issues/283))

## Root cause

`LocalBackend` reaches the notebook file system through
`serverapp.contents_manager`, wrapping every call as
`await asyncio.to_thread(self.contents_manager.get/new/save, ...)`.
`asyncio.to_thread` runs the callable in a worker thread and returns its result.
jupyter-server's default contents manager, `AsyncLargeFileManager`, defines
`get`/`new`/`save` as coroutine functions, so the worker thread calls the
coroutine function, gets back a coroutine object, and returns that object
un-awaited instead of a model dict.

Nothing raises at the boundary. The breakage surfaces one call later, differently
per caller, which is what made it read as several unrelated symptoms:

- `_list_notebooks_recursive` subscripts the coroutine (`model['type']`), the bare
  `except Exception` swallows the `TypeError`, and `list_notebooks` returns `[]`.
- `get_notebook_content`, `create_notebook`, and the save paths raise
  `TypeError: 'coroutine' object is not subscriptable` or leave a coroutine
  un-awaited.
- `notebook_exists` reports every path as present: the un-awaited coroutine is
  truthy, so the guard passes and the lookup body never runs.

## Invariant violated

Every `contents_manager` call in `LocalBackend` is awaited correctly for BOTH a
synchronous and an asynchronous ContentsManager. The default install is async, so
the un-guarded path is the one most users hit.

This is the mirror of the tool-side bug fixed in
[#281](https://github.com/datalayer/jupyter-mcp-server/pull/281) and it sits next
to corpus entry [006](006-anthropic-cache-tokens-lost.md) only loosely; the sharper
neighbour is any sync/async adapter. #281 had the opposite shape,
`await contents_manager.get(...)` (correct for async, broken for a sync manager);
this backend is `to_thread(contents_manager.get, ...)` (correct for sync, broken
for the default async). One codebase carried both polarities of the same mistake
in two files, because each was written against whichever manager the author had in
front of them. The general form: a call site that hard-codes one side of a
may-be-async contract is wrong for the other side, and which side is "the default"
is a property of the dependency, not of the code you are reading.

## Trigger

Any `LocalBackend` operation on a stock install, where `contents_manager` resolves
to `AsyncLargeFileManager`. No configuration beyond the defaults, no concurrency.
A test suite that pins a synchronous manager sees none of it.

## Repro

Docker `python:3.12-slim`, non-editable `pip install .`, current HEAD,
`jupyter_server==2.20.0`, `jupyter_core==5.9.1`, default `contents_manager`
resolving to `AsyncLargeFileManager`. Added
`tests/test_local_backend_contents_manager.py`, parametrized over a synchronous and
an asynchronous ContentsManager (same shape as the repo's existing
`tests/test_contents_manager_sync.py`), covering `get_notebook_content`,
`list_notebooks`, `notebook_exists` (present and absent), `create_notebook`, and a
get+save round trip through `append_cell`.

On `main` the five async cases fail:
`get_notebook_content`/`create_notebook`/`append_cell` raise `TypeError`,
`list_notebooks` returns `[]`, and `notebook_exists` reports a missing path as
present. With the fix all ten (five sync + five async) pass, and the pre-existing
`test_contents_manager_sync.py` still passes, so the tool-side path is unaffected.

**Not verified:** the manual MCP-client end-to-end flow from CONTRIBUTING was not
run. The save paths other than `append_cell`
(`insert_cell`/`delete_cell`/`overwrite_cell`/`execute_cell`) use the identical
wrapped call and were covered by inspection, not each exercised individually.

## Fix

Wrap the nine `contents_manager` calls in `jupyter_core.utils.ensure_async`, which
awaits an awaitable and passes a plain value through, so both manager flavours
work. This is the same primitive #281 applied to the tool files. The call set was
enumerated by parsing the module (nine `self.contents_manager.{get,new,save}`
calls, no other `contents_manager` access); all nine are wrapped. The one
genuinely-synchronous `to_thread` in the file, `client.get_iopub_msg`, is left
alone, because there `to_thread` is doing its real job of offloading a blocking
sync call.
