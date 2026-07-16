# A capability flag is not consent to use the capability

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/tools/use_notebook_tool.py::execute`
- **Class:** tool side-effect scope
- **Fix:** [PR #269](https://github.com/datalayer/jupyter-mcp-server/pull/269) (merged 2026-07-16; issue [#218](https://github.com/datalayer/jupyter-mcp-server/issues/218), user-filed)

## Root cause

`use_notebook` is conceptually a connection operation: register a notebook, start
a kernel, hand the agent a handle. On HEAD `a116ed6` it also ran this, near the
end of `execute`:

```python
context = get_server_context()
if context.is_jupyterlab_mode():
    # ... docmanager_open(...)
```

`docmanager_open` opens the notebook in JupyterLab, and opening a document
activates its tab. So every `use_notebook` call yanked the human's active tab to
whatever the agent had just decided to work on.

The condition is the whole bug. `is_jupyterlab_mode()` is driven by the
`JUPYTERLAB` environment variable, which **defaults to true**, and it answers a
question about deployment: is there a JupyterLab UI attached to this server?
The code used that answer to decide something else entirely: whether the user
wants their screen taken over. Those are different questions, and only the first
one had an input.

The fix adds the missing input rather than changing the existing one: a separate
`open_notebook_in_ui` config field (`OPEN_NOTEBOOK_IN_UI`, CLI
`--open-notebook-in-ui`, extension trait), defaulting to **false**, and gates
the open on both:

```python
if context.is_jupyterlab_mode() and get_config().open_notebook_in_ui:
```

## Invariant violated

A flag that describes what the environment *can* do is not a statement that the
user *wants* it done. `JUPYTERLAB=true` means a UI exists; it does not mean
"interrupt me." Reusing a capability flag as a consent flag looks economical and
reads fine at the call site, precisely because the name is true in both
sentences. The tell is that the flag has no way to express "yes, there is a UI,
and no, do not touch it": a single boolean was doing two jobs and the second job
had no off switch.

The generalization for agent tooling: a tool's side effects should be scoped to
what its caller asked for. An MCP tool call is issued by a model, but the side
effect here lands on a human who did not issue it and is not watching the tool
call. Any effect that crosses from the agent's workspace into the user's
attention (focus, navigation, notifications, window state) needs its own opt-in
and should default off, because the cost of a wrong default is paid by someone
who cannot see why it happened. Agents also call these tools far more often than
humans do: a human opens a notebook and stays there, while an agent may switch
notebooks dozens of times in a session, so a side effect that is merely
tolerable at human frequency becomes unusable at agent frequency. The frequency
change is what turns a defensible default into a bug.

Note also which way the default moved. When a capability flag and a consent flag
are untangled, the consent flag does not inherit the capability flag's default.
`JUPYTERLAB` defaults true because the UI is usually there; `OPEN_NOTEBOOK_IN_UI`
defaults false because being interrupted is usually not wanted. Splitting the
flag while keeping the old default would have preserved the bug under a better
name.

## Trigger

Any `use_notebook` call against a server running in the default configuration
(`JUPYTERLAB=true`) with a JupyterLab UI open. The agent switching notebooks
pulls the human's active tab each time, including while the human is typing in
another tab.

## Repro

**Verified:** the gating and the defaults. `tests/test_use_notebook_ui_open.py`
runs in a clean `python:3.10-slim` container against the repo HEAD and asserts
that the UI-open path is skipped when `open_notebook_in_ui` is false (the
default) and taken when it is true and JupyterLab mode is on, with
`jupyter_mcp_server.__file__` confirmed to resolve to the repo checkout. The
full suite is green on all 15 CI legs.

**Not verified:** the focus-steal itself. Observing a tab actually being
activated requires a live JupyterLab front end plus `jupyter-mcp-tools` driving
a real browser session, which was not reproduced headless in a container. The
side effect is established by reading the call into `docmanager_open` on HEAD
and by the issue reporter's first-hand account, not by watching a tab move. The
root cause is a source read; the fix's gating behavior is executed and observed.
