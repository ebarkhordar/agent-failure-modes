# A documented setting is an assertion about code, and only a caller can make it true

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/config.py` (`JupyterMCPConfig`), `jupyter_mcp_server/CLI.py`
- **Class:** configuration wiring & documented contracts
- **Fix:** [PR #270](https://github.com/datalayer/jupyter-mcp-server/pull/270) (merged 2026-07-16; issue [#264](https://github.com/datalayer/jupyter-mcp-server/issues/264))

## Root cause

Two defects with one shape.

`execution_timeout` and `max_execution_timeout` were added to `JupyterMCPConfig`, and
README.md:32 plus the tools reference documented `JUPYTER_MCP_EXECUTION_TIMEOUT` as the
way to set them. That variable exists in no Python source. It appears in those two
documentation files and nowhere else: `grep -rn JUPYTER_MCP_EXECUTION_TIMEOUT
--include='*.py' .` returns nothing. `JupyterMCPConfig` is a plain pydantic `BaseModel`,
not `BaseSettings`, so it reads no environment, and no `--execution-timeout` option fed
the existing `set_config` call. Every operator who followed the README exported the
variable and silently kept the 120s default.

Separately, `config.py:38` shipped the field description `Set to 0 for unlimited (use
with caution)`. Zero is not unlimited at either seam it reaches. It flows straight into
`asyncio.wait_for`, and into `execute_code_local`, where `timeout_ms = timeout * 1000`
makes `remaining_ms = max(0, timeout_ms - elapsed_ms)` zero on the first loop iteration,
so the execution expires before the kernel can answer. That second defect is unreachable
today only because nothing can set the value: the wiring that fixes the first is what
makes the second live.

## Invariant violated

A documented configuration knob is an assertion about code: that a value entering at the
documented surface (env var, flag, config file) reaches the code that reads it.
Documentation cannot establish that path. Only a caller can. The failure is silent in the
worst direction, because the system starts, runs, and honors its default, so the
operator's evidence that the setting took effect is identical to the evidence that it did
not.

The check that generalizes is to trace from the published name inward, not from the field
outward. Grepping the source for the documented variable is a complete test of the first
defect and takes seconds. It works precisely because a reviewer reading `config.py` sees a
well-formed field with a type, a default, and a description, and has no reason to ask who
supplies it. The field was real. The path was what was missing, and a field is the thing
you look at while the path is the thing you have to go find.

The second defect generalizes differently: a field description is untested code. `Set to 0
for unlimited` was false in the same commit that shipped it, and no test could have caught
it, because a docstring executes nothing. A sentinel value with a documented meaning (`0`
= unlimited, `-1` = disabled, `None` = default) is a branch. It is real only if some line
reads it and branches on it. Otherwise the sentinel travels on as an ordinary number into
arithmetic that quietly gives it the opposite meaning: here, the value documented as "no
limit" is the one value that guarantees immediate expiry.

## Trigger

Any deployment that configures the timeout the documented way. For the first defect,
exporting `JUPYTER_MCP_EXECUTION_TIMEOUT` at all. For the second, taking the description
at its word via a programmatic `set_config(execution_timeout=0)`.

## Repro

Clean Docker `python:3.11-slim` against HEAD `5ce3fc1`, installed from a fresh checkout
with provenance printed on both sides (`/src/jupyter_mcp_server/CLI.py`) to confirm the
executed source was the checkout and not an installed copy. The first attempt at this
repro imported the image's own baked-in copy of the package and had to be re-pinned, which
is the reason the provenance print is not optional.

Driven end to end through the real `start` command with only the blocking `mcp.run`
patched out: on `5ce3fc1`, `JUPYTER_MCP_EXECUTION_TIMEOUT=1800` gives
`get_config().execution_timeout == 120`, and `=0` starts the server anyway.

The zero-expires-instantly behavior was executed against a real ipykernel through
`execute_code_local` on `5ce3fc1`: `timeout=30` returns `['hello from a real kernel']` in
0.223s, while `timeout=0` returns `['[TIMEOUT ERROR: Code execution exceeded 0 seconds]']`
in 0.115s with zero outputs.

**Not verified:** no live JupyterLab or Datalayer runtime was exercised. The wiring is
proven at `get_config()` and at the execution seam, not by watching a long-running cell
survive to 1800s.
