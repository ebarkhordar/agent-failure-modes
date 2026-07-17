# Extracted, returned, and never counted: a metric that reads zero because nobody feeds it

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight_api/llm/openai_compatible_llm.py`, the `record_llm_call`
  sites in `call()` and `call_with_tools()`
- **Class:** streaming & usage accounting
- **Fix:** [PR #2758](https://github.com/vectorize-io/hindsight/pull/2758) (merged)

## Root cause

`OpenAICompatibleLLM` pulls `cached_tokens` and `thoughts_tokens` out of the
provider's usage object on both call paths and passes them to `TokenUsage`. It
does not pass them to `metrics.record_llm_call`, which sits a few lines below and
already accepts both kwargs, keeping a bucketed counter for each
(`llm_tokens_cached_input`, `llm_tokens_thoughts`). The values are extracted,
used for the return contract, and dropped on the metrics path.

Two distinct effects, worth keeping apart because they have different histories:

1. **Reasoning tokens reach no counter at all, and this is a regression.** #2378
   made `output_tokens` visible-only by subtracting `thoughts_tokens`, and that
   subtraction sits directly above `record_llm_call`. So reasoning was removed
   from the metrics path rather than moved onto `llm_tokens_thoughts`. Before
   #2378 those tokens were at least still counted inside `output_tokens`.
2. **`cached_input_tokens` has always read 0 here.** This provider has never
   passed it since the counter landed in #1936; `gemini_llm` is the only provider
   that does. That half is a gap rather than a regression.

## Invariant violated

Recorded `output_tokens` plus recorded `thoughts_tokens` equals the provider's
`completion_tokens`: every billed output token lands on exactly one counter,
never zero and never two.

The mechanism behind effect 1 is the one to carry away. A refactor that makes a
number more accurate for one consumer can silently remove it from another. #2378
was right: `output_tokens` should mean visible output, and subtracting reasoning
improved the return contract. One line below, a second consumer received the same
variable and wanted the opposite thing from it, and the change served the nearer
one. Neither consumer is named at the subtraction site. A variable feeding two
sinks with different definitions of correctness is a fork in the code that looks
like a single value, and the fix is not to pick a side but to notice that
subtracting from a value is only half of moving it: the other half has to land
somewhere.

The second mechanism is that a counter which exists and reads zero is worse than
a counter that does not exist. `llm_tokens_cached_input` pinned at 0 on a
dashboard does not read as "nobody reports this", it reads as "caching is not
happening", which is a claim about the world. Its presence in the metrics schema
is what makes the code look like it reports the value. This is the same shape as
entry [026](026-ldr-download-queue-no-atomic-claim.md), where a `PROCESSING`
status existed in the model and was never written: a mechanism that is declared
and never fed gets read by everyone as evidence. When auditing telemetry, do not
check that the counter exists. Find the write, and check it is on the path that
produces the number.

The rationale for the counter was already written next to it in `metrics.py`:

> Surfacing them as a distinct counter is required for honest cost attribution: a
> workload that "looks cheap" by output volume can be silently expensive if the
> model is doing long reasoning chains.

That comment introduces reasoning tokens as the Gemini 2.5+ family, which is
where they came from, but the counter name and the `record_llm_call` signature
are both provider-neutral, and #2378 established that OpenAI-compatible reasoning
models report the same counts. The workload the comment describes, an o-series or
deepseek-r1 run that reads as cheap, is exactly the one this provider serves.

## Trigger

Any OpenAI-compatible reasoning model (o-series, deepseek-r1) for the reasoning
half, and any prompt-cached workload through this provider for the cached half.
No configuration and no failure: the calls succeed and the cost attribution is
wrong.

## Repro

Clean `python:3.12-slim` container, package installed into real site-packages
rather than imported from a source dir, against `main` at `9676fc16`. Usage shape
`prompt=2000 (cached=1024), completion=83 (reasoning=64)`:

| | `record_llm_call` receives |
|---|---|
| `main` | `output_tokens=19`, no `thoughts_tokens`, no `cached_input_tokens` |
| branch | `output_tokens=19, thoughts_tokens=64, cached_input_tokens=1024` |

On `main` that is 19 of 83 billed output tokens counted, and 0 of 1024 cached. For
the regression half, #2378's parent (`701de329`) was installed in the same
container and given the same probe: `record_llm_call(output_tokens=83)`, so all
billed output was visible to the counters before that change.

Three tests added to `tests/test_token_usage_cached_thoughts.py`, asserting
against a `MagicMock(spec=MetricsCollector)`. All three fail on `main`
(`KeyError: 'thoughts_tokens'`) and pass on the branch. The existing provider
tests patch `get_metrics_collector` without asserting on it, which is why this
survived them: mocking a collaborator and never checking what it received tests
that the call does not crash.

**Not verified:** no live provider call. The usage objects are `SimpleNamespace`
mocks, and the `completion=83, reasoning=64` pair is quoted from #2378's own
commit message rather than measured. The claim is only that HEAD drops these
fields given that usage shape. Verification stops at the provider-to-collector
boundary: the kwargs the provider passes are asserted, and `record_llm_call` was
read to confirm each lands on a counter, but no live Prometheus endpoint was
scraped.

## Fix

Pass `cached_input_tokens` and `thoughts_tokens` at the two call sites that parse
a usage object. Four added kwargs, no logic or signature change. The other two
`record_llm_call` sites are deliberately untouched: the `tool_use_failed` fallback
has no usage object, and the Ollama native path reads
`prompt_eval_count`/`eval_count`, which carry no reasoning or cached counts.

`anthropic_llm.py` and `litellm_llm.py` have the same `cached_input` gap and were
left out on purpose, following #2378's precedent of scoping to one provider and
offering the rest as a follow-up.
