# One of three sibling tools skips the shared timeout resolution, so the config ceiling silently does not bind there

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/server.py` (`execute_code`, ~line 810)
- **Class:** configuration wiring & documented contracts
- **Fix:** [PR #282](https://github.com/datalayer/jupyter-mcp-server/pull/282) (merged 2026-07-18; follows [#270](https://github.com/datalayer/jupyter-mcp-server/pull/270) and issue [#264](https://github.com/datalayer/jupyter-mcp-server/issues/264))

## Root cause

Three tools run code with a `timeout`: `execute_cell`, `insert_execute_code_cell`,
and `execute_code`. The first two resolve the caller's `timeout` against the config
before running:

```python
config = get_config()
effective_timeout = config.execution_timeout if timeout == 0 else min(timeout, config.max_execution_timeout)
```

`execute_code` did neither. It forwarded its raw `timeout` straight through, with a
hardcoded `Field(le=3600)` as its only bound. Two consequences followed. First,
`max_execution_timeout` did not bind: an operator lowering the ceiling saw it enforced
on two of the three tools and silently ignored on the third, which accepted any value
up to 3600. Second, `timeout=0` was not the config default here; the other two tools
document `0` as "use config default", while `execute_code` forwarded `0` verbatim to
`asyncio.wait_for(timeout=0)`, which expires immediately. The `le=3600` cap was a stale
duplicate, raised in #257 thirteen days before the config landed in #266 and never
reconciled with `max_execution_timeout`.

## Invariant violated

When a setting is documented once for a family of sibling operations, each sibling is a
separate site that must be made to honor it; the setting is an assertion about code, and
only the caller that reads it makes it true (the same shape as entry 028, one tool deeper).
A resolution step shared by two of three code paths is not shared by the third until the
third also calls it. When a config value has a family of consumers, enumerate the family
and check each member reads it, rather than trusting that "the config is wired" once one
consumer does. A per-tool duplicate bound (`le=3600`) that predates the config is not a
second safety net, it is a divergent source of truth that hides the gap.

## Trigger

Operator sets `--max-execution-timeout` below 3600 and calls `execute_code` with a larger
value (ceiling not enforced), or calls `execute_code` with `timeout=0` expecting the
documented config default (expires immediately instead).

## Repro

Reproduced on HEAD in a clean `python:3.12-slim` container (`server.__file__` confirmed
installed source). A new `tests/test_execute_code_timeout.py` captures the timeout that
reaches the tool, so resolution is observable without a live kernel: with
`execution_timeout=45`, `max_execution_timeout=60`, `timeout=3600` resolves to 60 and
`timeout=0` resolves to 45 (both fail on `main`, which passes the raw value through);
`timeout=30` and omitting `timeout` stay 30 (unchanged). Verified at the resolution
boundary, not driven end-to-end through a kernel. The default (`execute_code` still
defaults to `timeout=30` while its siblings default to `0`) was left as a public-behaviour
call for the maintainer. Merged by echarles.
