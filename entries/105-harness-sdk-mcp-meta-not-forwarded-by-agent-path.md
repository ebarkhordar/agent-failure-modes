# The wrapper forwards one key out of an untyped invocation context and drops the rest, so a parameter the underlying client accepts has no route from the caller

- **Repo:** strands-agents/harness-sdk
- **Surface:** `strands-py/src/strands/tools/mcp/mcp_agent_tool.py`, `MCPAgentTool.stream`
  (`:113`, the `call_tool_async` call), against `MCPClient.call_tool_async` in
  `tools/mcp/mcp_client.py` (`:875`), whose signature accepts `meta` at `:881`
- **Class:** configuration wiring & documented contracts
- **Report:** reproduced publicly on
  [issue #3517](https://github.com/strands-agents/harness-sdk/issues/3517#issuecomment-5113995077)
  (open). No PR from us yet: the repair adds public API surface, this repo runs a design review
  before that lands, and the naming question the report opens (what the key is called, and what
  happens when a construction-time value and a per-request value are both set) is the
  maintainers' to answer, not ours to pick unilaterally.

## Root cause

There are two routes to the same MCP tool call. A caller can invoke
`MCPClient.call_tool_async` directly, and that signature takes `meta` (`:881`), which the client
sends as the request's `_meta`. The other route is the one the framework exists to provide: the
model decides to call the tool, and the call is issued by `MCPAgentTool.stream`.

`stream` builds that call at `mcp_agent_tool.py:113` with five arguments. One of them is
`cancel_signal`, and it is read out of the ambient invocation context at `:118`:

```python
result = await self.mcp_client.call_tool_async(          # :113
    tool_use_id=tool_use["toolUseId"],
    name=self.mcp_tool.name,
    arguments=tool_use["input"],
    read_timeout_seconds=self.timeout,
    cancel_signal=getattr(invocation_state.get("agent"), "_cancel_signal", None),   # :118
)
```

`meta` is not in that list, so on the model-driven route it is never set. Measured against a
stdio MCP server whose tool echoes the `_meta` it received, both routes in one process:

```
DIRECT (call_tool_async(meta={"trace_id": "abc"}))
  -> {"meta_seen": {"progressToken": null, "trace_id": "abc"}}
MODEL-DRIVEN (Agent + MCPAgentTool.stream)
  -> {"meta_seen": {"progressToken": null}}
```

The detail that makes this hard to see is the `progressToken`. The transport populates `_meta`
on its own, so the field arrives on both routes and is merely incomplete on one. A server
checking whether it received `_meta` gets yes in both cases.

`invocation_state` is not a closed structure. It is a dict threaded from the public entry points
(`Agent.__call__` at `agent.py:704`, `invoke_async` at `:786`, `stream_async` at `:1071`, each
taking an `invocation_state` parameter) down to this call site, and a caller-supplied key was
measured arriving intact: `invoke_async(prompt, invocation_state={"meta": {...}})` reaches
`stream` with `invocation_state["meta"] == {"trace_id": "abc"}`. The value is already there. The
line that would forward it sits one line above the line that forwards `cancel_signal`.

## Invariant violated

When a low-level client accepts a parameter and a high-level wrapper is the path most callers
actually use, the wrapper is the real API surface. A parameter reachable only by bypassing the
wrapper is not supported, it is documented. The gap is invisible from either side read alone:
the client's signature says `meta` is a first-class argument, and the wrapper's call site looks
complete, because a call site always looks complete. Nothing at either location states that the
two lists are supposed to agree.

The sharper rule is about the shape of the context object. **An untyped dict threaded through a
call chain has no schema, so what gets forwarded is decided per key, at each call site, by
whoever needed that key.** `cancel_signal` is forwarded because someone implementing cancellation
needed it there. Nothing generalised that to "this call site forwards what the invocation carries",
because there is no declaration of what the invocation carries. Adding a key upstream is free and
silent, and it arrives nowhere. That is the opposite of the property a dict-shaped context appears
to offer, and the appearance is why it keeps being used for this: it looks like a bag that flows
through, and it is really a set of independently hand-wired single keys.

A corollary worth stating on its own, because it inverts the usual severity ordering: **a field
that arrives partially populated is worse than one that does not arrive.** An absent `_meta` fails
the receiving end's first check and gets reported. A present `_meta` carrying only the framework's
own housekeeping key passes every structural check and loses exactly the application's values, so
the failure surfaces later, somewhere else, as trace data that is missing for reasons no one can
attribute.

## Trigger

Any MCP tool call issued through an `Agent` where the server is expected to see request-scoped
metadata: a trace or correlation id, a tenant id, anything a caller sets per invocation. The
direct-client path carries it and the agent path does not, so the same value tested through the
client and shipped through the agent silently stops arriving.

## Repro

Clean `python:3.12-slim`, `PYTHONPATH` onto `strands-py/src`, `mcp` 1.29.0, with `__file__`
provenance printed so the imported modules are the checkout's. A stdio MCP server exposes one
tool that returns the `_meta` it received. The same tool is then called twice in one process,
once through `MCPClient.call_tool_async(meta={"trace_id": "abc"})` and once by letting a model-
driven `Agent` select it, and the two returned `meta_seen` values are compared. The
`invocation_state` measurement used a pass-through recorder around the real `MCPAgentTool.stream`,
so the recorded dict is the one the production method received.

Not checked: what the forwarding key should be named, and whether a construction-time default and
a per-request value should merge or override when both are set. Both are design questions raised
in the report rather than measured.

Verified 2026-07-29 at HEAD `f2482f41`; the line citations above were re-read at that commit.
Reported, not fixed upstream at the time of writing.
