# A token count that already includes cache reads, priced as if none of it were cached

- **Repo:** openlit/openlit
- **Surface:** `openlit/instrumentation/claude_agent_sdk/utils.py::set_chat_span_attributes`
  and `openlit/instrumentation/litellm/utils.py::common_chat_logic`
- **Class:** streaming & usage accounting
- **Fix:** [PR #1379](https://github.com/openlit/openlit/pull/1379) (in review;
  issue [#1378](https://github.com/openlit/openlit/issues/1378))

## Root cause

`get_chat_model_cost` can price cache-read and cache-write tokens at their own
rates, but only when the caller passes the cache counts and declares that the
`prompt_tokens` it is handing over already includes them
(`prompt_tokens_include_cache=True`); the helper then re-prices the cached slice
at `cacheReadPrice`/`cacheCreationPrice` and subtracts it from the prompt base.
Two adapters skipped that contract. The `claude_agent_sdk` adapter builds
`input_tokens = uncached + cache_read + cache_creation` and forwarded that
inclusive total with no cache arguments. The `litellm` adapter forwarded
litellm's normalized `prompt_tokens`, which is likewise cache-inclusive, the same
way. So the cached fraction of every prompt was billed at full prompt price. For
Claude models, the only models in the bundled pricing that define a
`cacheReadPrice` (set at 10x the read discount), a heavily cached prompt is
overcharged in proportion to its cached fraction: the PR measures a 2.23x
`gen_ai.usage.cost` on a 6200-token call that was mostly cache reads.

A second, independent defect sat in the litellm path: cache-creation tokens were
read from `completion_tokens_details.cached_tokens`, a field litellm never
populates, while litellm actually reports them under
`prompt_tokens_details.cache_creation_tokens`. The count was therefore always
zero, so both the cost and the `gen_ai.usage.cache_creation_input_tokens` span
attribute silently under-reported cache writes as none.

## Invariant violated

A token total and the price applied to it must agree on what the total contains.
When an adapter's `prompt_tokens` is cache-inclusive, the cost call has to say so
and pass the cache counts, otherwise the priced quantity and the counted quantity
are different sets and every cached token is billed twice-over at the wrong rate.
A shared costing helper that supports cache rates only closes the gap at the call
sites that opt in; each adapter that assembles an inclusive total is a separate
site that must opt in. And a usage field must be read from the key the source
actually populates: reading cache-creation from a field that provider never sets
does not fail loudly, it reports a confident zero.

## Trigger

Any Claude call through the `claude_agent_sdk` or `litellm` instrumentation whose
prompt reuses a cache (`cache_read_input_tokens > 0`). The larger the cached
fraction, the larger the cost inflation. Non-Claude models are unaffected because
no bundled model defines cache prices, so `prompt_tokens_include_cache=True` is
inert for them.

## Repro

Clean `python:3.12-slim` container, no network, against the fix branch, with each
module's `__file__` and sha256 asserted equal to the checkout. Driving
`claude_agent_sdk` `extract_usage` then the cost call for a 6200-token inclusive
total (200 uncached, 5000 cache-read, 1000 cache-write) yields `$0.010350`, the
ground-truth cache-aware cost, where `main` bills `$0.023100` (2.23x). litellm's
own `AnthropicConfig().calculate_usage` (litellm 1.92.0) was confirmed to return
`prompt_tokens = 6200` and to expose the cache-creation count under
`prompt_tokens_details.cache_creation_tokens`, not under `completion_tokens_details`.
Two regression tests fail on `main` and pass on the branch; they are dict-driven
and import neither SDK. Not verified end to end: the repo's Python SDK tests are
disabled in CI (`python-tests.yml` is held), so they were run locally, not by a
live provider call. Scope was held to the two adapters that were confirmed to
build an inclusive total and route Claude; `strands/processor.py` also omits the
cache arguments but sources its count from an external SDK whose inclusiveness
could not be verified offline, so it was left out rather than guessed at.
