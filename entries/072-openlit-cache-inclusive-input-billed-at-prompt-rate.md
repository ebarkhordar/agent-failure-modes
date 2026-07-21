# Two Anthropic adapters fold cache tokens into prompt_tokens but forward no cache args to the cost call, so cache-read tokens are billed at the full prompt rate (litellm 2.84x, claude_agent_sdk 3.74x overcharge)

- **Repo:** openlit/openlit
- **Surface:** `sdk/python/src/openlit/instrumentation/litellm/utils.py` (cost call at `:436`, extraction at `:407-410`/`:767-770`, span attrs at `:573-583`) and `sdk/python/src/openlit/instrumentation/claude_agent_sdk/utils.py` (`:324` -> `_calculate_cost` `:430`; `extract_usage` `:166` builds an inclusive `total_input`)
- **Class:** streaming & usage accounting
- **Report:** [issue #1378](https://github.com/openlit/openlit/issues/1378), fixed in [PR #1379](https://github.com/openlit/openlit/pull/1379) (merged 2026-07-21).

## Root cause

`get_chat_model_cost(...)` prices cache-read tokens at `cacheReadPrice` only when
the caller forwards `cache_read_tokens`/`cache_creation_tokens`, and it must be
told `prompt_tokens_include_cache=True` when the provider's `prompt_tokens`
already includes those cache tokens. Both adapters extract the cache token counts
and publish them as span attributes, then call the cost function with only the
inclusive `prompt_tokens` and no cache args. The cost function therefore prices
every cache-read token at `promptPrice`, which for `claude-*` is 10x
`cacheReadPrice`. The span and the bill disagree inside one function: the
attributes report the cache tokens the cost math has already overcharged.

A second, independent defect in the litellm path: it reads
`_cache_creation_input_tokens` from `usage.completion_tokens_details.cached_tokens`,
an attribute that does not exist on `CompletionTokensDetailsWrapper`, so the
cache-creation count is silently always 0. The correct source is
`prompt_tokens_details.cache_creation_tokens`.

## Invariant violated

A cost function that accepts a token total plus optional cache-token args carries
a precondition: whoever passes an inclusive total (cache tokens already folded
into `prompt_tokens`) must also pass the cache args and the include-cache flag, or
the cache tokens are billed at the uncached rate. An adapter that has already
extracted the cache counts (proven by publishing them as span attributes) may not
then drop them at the cost call. Where one value is both reported and billed, the
two computations must read the same decomposition of the input.

## Trigger

An Anthropic model (`claude-*`, the only family in openlit's `pricing.json`
carrying `cacheReadPrice`) routed through either the litellm or the
claude_agent_sdk instrumentation, on a request that reads cached input tokens. The
overcharge scales with the cached fraction. Adapters whose `prompt_tokens` are
cache-exclusive (native anthropic) or that already forward the flag (openai,
langchain) are unaffected, which is why the defect hid: three of the five paths
were correct.

## Repro

Clean container (`python:3.12-slim`, `--network=none`) against `main` HEAD
`a3d6f17`, module provenance confirmed by sha256 of the loaded modules. The same
request and model driven through both adapters from the real SDK usage types
offline: native anthropic cost $0.013200 (ground truth) vs litellm $0.037500 =
2.841x; ground truth $0.017250 vs claude_agent_sdk $0.064500 = 3.739x. Only the
cost arithmetic over the usage object is claimed and executed; no live-provider
billing was observed and none is needed, since the defect is in openlit's own
math. The 37 call sites of `get_chat_model_cost` were enumerated by ast-parsing
(not grep) to confirm only cache-inclusive claude-routing adapters are affected:
only 22 of 165 priced models carry `cacheReadPrice` and all 22 are `claude-*`, so
a violator must be both cache-inclusive and route `claude-*`. The fix (forward
both cache args and `prompt_tokens_include_cache=True` at each cost call, and
correct the cache-creation source field) landed in
[PR #1379](https://github.com/openlit/openlit/pull/1379); two new regression tests
fail on `main` and pass on the branch.
