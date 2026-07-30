# A configured value that reaches only the telemetry path, so the instrument an operator would check reports the setting that was never applied

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight-api-slim/hindsight_api/engine/providers/claude_code_llm.py`: the
  two `ClaudeAgentOptions(...)` constructions at `:236` (`call()`) and `:539`
  (`call_with_tools()`), against the uses of `self.model` at `:310`, `:325`, `:341`, `:595`
  and `:606`
- **Class:** configuration wiring & documented contracts
- **Report:** triaged publicly on
  [issue #2881](https://github.com/vectorize-io/hindsight/issues/2881#issuecomment-5081797483)
  (filed by another user running the claude-code provider, closed as completed).
- **Fix:** [PR #2980](https://github.com/vectorize-io/hindsight/pull/2980) (merged
  2026-07-28), which pins the configured model in both constructions.

## Root cause

The provider spawns a CLI through the Claude Agent SDK. `ClaudeAgentOptions` accepts a
`model` argument, and neither of the two constructions in the file passes one, so the
spawned CLI runs whatever model it defaults to. `self.model` is not unused: it reaches
`metrics.record_llm_call` at `:310` and `:595`, the span recorder at `:325`, and the
`slow llm call` log line at `:341` and `:606`. Every knob that sets it
(`HINDSIGHT_API_RETAIN_LLM_MODEL` and its per-scope siblings) is therefore fully wired into
observability and connected to nothing that runs.

Two details change what the fix has to be.

**The sibling provider disagrees, so this is a gap rather than a convention.** `CodexLLM`,
the other subscription-auth CLI-backed provider, puts `"model": self.model` in both of its
request payloads (`codex_llm.py:438`, `:824`) and even normalises a leading `openai/` off it
at `:172`. There is no shared "CLI-backed providers do not pin the model" rule to preserve.

**Passing the value activates a default that is currently inert.** An unset per-scope model
already resolves provider-aware: `claude-code` with no model set falls through to
`_get_default_model_for_provider` (`config.py:2555`, same shape at `:2583`, `:2608`, `:2463`)
which returns `claude-sonnet-4-5-20250929` (`config.py:753`). Today that string reaches
nothing. With the one-line fix, every deployment that never set a model stops riding the
CLI's own current default and starts running a pinned 2025-09-29 snapshot id.

## Invariant violated

**A configured value that reaches only the telemetry path is worse than one that reaches
nothing, because the telemetry is the instrument an operator would use to detect the drop.**
The natural check after setting a model is to look at the logs, the metrics labels or the
trace and confirm the name you set. Here all three are populated from the same field the SDK
never receives, so the check passes precisely because of the code that fails. An operator can
only find this by comparing output quality or provider billing against a config they have
already confirmed, which is a far more expensive instrument than the one they were handed. A
setting is applied when the consumer receives it, and a name printed beside a call is
evidence about the printer.

**Wiring up a dead knob is not a no-op for the users who never touched it.** The intuition
that a fix is safe for the untouched population is backwards when the fix connects a value
that was being discarded: the untouched population is exactly the set now moved onto whatever
the default resolver returns, and they are the least prepared for the change because they
never made a choice at all. Before connecting a discarded value, resolve what it evaluates
to when unset, and count who is on that branch. Here the answer is a pinned snapshot from a
specific date, which ages differently from the CLI's rolling default that those deployments
have been getting.

**When one member of a provider family differs from its siblings, the first question is which
one is the outlier.** The same difference supports two opposite fixes, and only the history
of the family tells you which. Reading the sibling first is cheap and it changes the patch.

## Trigger

Any deployment on the claude-code provider that sets a model. The mismatch is silent by
construction: telemetry reports the configured name, the CLI runs its default, and the
answers are plausible either way.

## Repro

The completeness claim is an enumeration over code structure, so it was answered by parsing
rather than by matching strings: an AST walk over `hindsight_api/` at `main` `ed120a256`
finds exactly two `ClaudeAgentOptions(...)` constructions, both in `claude_code_llm.py`, and
neither carries a `model` argument.

Executed in a clean `python:3.12-slim` container: the default resolver was run against
HEAD's `config.py` and returned `claude-sonnet-4-5-20250929` for `claude-code` with no model
set, and `ClaudeAgentOptions` was installed and its dataclass fields read at both 0.2.82,
the floor declared in `pyproject.toml:86`, and 0.2.128. `model` is `str | None = None` in
both, so the one-line fix is valid at the declared minimum and not only at the version the
reporter runs.

**Not verified, and this bounds the entry:** the CLI itself was never spawned. That the
spawned process runs its own default model is the reporter's observation plus the absence of
the parameter, not a measurement of ours, and no claim is made here about which model a given
subscription tier actually falls back to.

Verified 2026-07-26 at `ed120a256`. Fixed upstream by PR #2980, merged 2026-07-28.
