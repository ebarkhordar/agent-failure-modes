# A documented kernel-targeting parameter silently discarded in the default server mode

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/server.py` (`execute_code` handler), `jupyter_mcp_server/execute_code_tool.py` (`_execute_via_notebook_manager`)
- **Class:** configuration wiring & documented contracts
- **Fix:** [PR #280](https://github.com/datalayer/jupyter-mcp-server/pull/280) (merged), [issue #279](https://github.com/datalayer/jupyter-mcp-server/issues/279)

## Root cause

`execute_code` exposes a public `kernel_id` parameter so a caller can target a
specific kernel. The handler resolves that argument in only one of the two server
modes. In `JUPYTER_SERVER` mode `server.py` reads `kernel_id` and routes the
execution to the named kernel. In `MCP_SERVER` mode, which is the default, the
same call routes through `execute_code_tool.py`'s `_execute_via_notebook_manager`,
whose signature has no `kernel_id` parameter at all. The argument is accepted at
the tool boundary, then dropped on the floor, and the code runs on the current
notebook's kernel instead of the one the caller named.

The failure is invisible because nothing on the path objects. The parameter is
documented, the call is well formed, no branch rejects it, and the tool returns
output. It is simply the wrong kernel's output, reported as success.

## Invariant violated

A documented public parameter must either take effect or raise. A third outcome,
accepting the argument and silently ignoring it, is the worst of the three,
because the caller has no signal that their intent was discarded and the return
value looks correct.

More narrowly: when the same public tool has two backend paths for two runtime
modes, every documented parameter has to be honored (or rejected) on both paths.
A parameter wired into one backend and absent from the other backend's signature
is a no-op in whichever mode routes to the second backend, and if that mode is
the default, the parameter is a no-op for most users.

## Trigger

Calling `execute_code` with `kernel_id` set while the server runs in `MCP_SERVER`
mode (the default). The ordinary case for anyone using kernel targeting: the
parameter exists precisely to be used, and the default mode is the one most
deployments run.

## Repro

The kernel-targeting call is issued in `MCP_SERVER` mode against a session with
more than one kernel, and the output is compared against the kernel the caller
named. The reproduction and the fix are in the linked PR, which routes the
`kernel_id` through the `MCP_SERVER` backend so a targeted call lands on the
kernel it asked for.
