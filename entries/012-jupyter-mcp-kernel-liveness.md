# Restart can't recover an idle-culled kernel: dead binding, no self-heal

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `restart_notebook_tool.py` (JUPYTER_SERVER mode)
- **Class:** resource liveness & reconnection
- **Fix:** [PR #267](https://github.com/datalayer/jupyter-mcp-server/pull/267) (merged; issue [#260](https://github.com/datalayer/jupyter-mcp-server/issues/260))

## Root cause

In JUPYTER_SERVER mode, `restart_notebook` calls `restart_kernel(kernel_id)`
on the kernel id cached at `use_notebook` time. If that kernel was idle-culled
or lost to a server restart, the Jupyter server answers `404: Kernel does not
exist`; the tool catches the error and returns "Failed to restart" instead of
provisioning a new kernel. The cached id is never liveness-checked, so the
agent is left holding a permanently dead binding it has no path to recover.

## Invariant violated

A cached handle to an external, cullable resource must be revalidated on use,
and the recovery operation must actually recover. A "restart" issued against a
kernel that no longer exists should self-heal (start a fresh kernel and rebind
it), not report the loss as a terminal failure. The one operation whose job is
to restore a working state is exactly the one that must not assume the old
state still exists.

## Trigger

The kernel is idle-culled (or the server restarts) between `use_notebook` and
`restart_notebook`; the still-cached `kernel_id` now 404s.

## Repro

Integration in the JUPYTER_SERVER harness: start the extension, delete the
kernel out-of-band (`DELETE /api/kernels/<id>`), call `restart_notebook`.
On HEAD it returns Failed; routing the 404 into a start-plus-rebind path
returns success. Fix extracts a reprovision helper covered by a parametrized
unit test. A sibling issue (#21) tracks the same class on the MCP_SERVER side,
so the maintainers already recognize the defect.
