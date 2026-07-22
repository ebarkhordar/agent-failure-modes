# One provider adapter reads only the visible-output token field its siblings fold reasoning into, so output tokens and the cost derived from them undercount every thinking-model call

- **Repo:** confident-ai/deepeval
- **Surface:** `deepeval/models/llms/gemini_model.py::_token_cost` (the `output_tokens = getattr(usage, "candidates_token_count", None)` read that feeds `calculate_cost`)
- **Class:** streaming & usage accounting
- **Fix:** [PR #2943](https://github.com/confident-ai/deepeval/pull/2943) (in review;
  self-discovered, no issue)

## Root cause

`_token_cost` maps google-genai's `usage_metadata` onto deepeval's cost record by setting
`input_tokens = prompt_token_count` and `output_tokens = candidates_token_count`, then
multiplies `output_tokens` by the model's output price in `calculate_cost`. For a Gemini
2.5/3.x model with thinking enabled, `candidates_token_count` counts only the visible
answer. The reasoning tokens are reported in a separate field, `thoughts_token_count`, and
Google's own usage definition makes `total_token_count = prompt_token_count +
candidates_token_count + thoughts_token_count`. Google bills the thoughts tokens at the
output rate, so reading only `candidates_token_count` drops them from both the reported
output count and the derived cost.

deepeval's OpenAI adapter does not have this gap: `openai_model.py` reads
`completion_tokens`, which already bundles the model's reasoning tokens into the one output
figure. So the two adapters normalize into the same cost record from different provider
fields, and only the Gemini path leaves a billed-output segment out. The undercount scales
with how much the model reasoned, which means it is zero on a trivial prompt and largest on
exactly the reasoning-heavy calls a cost report most needs to be right about.

## Invariant violated

When several provider adapters feed one shared cost or usage record, each must map its
provider's fields so that `output_tokens` counts every token the model generated and was
billed for as output, not only the tokens that reached the visible response. A per-provider
field name is an implementation detail of that provider's wire format; the shared record is
read downstream by cost and budget code that knows nothing about which fields it came from,
so each adapter owes the same semantic total on its own. An adapter that copies the
provider's visible-answer field straight into `output_tokens` silently under-reports for any
provider that accounts reasoning separately, and the gap is invisible to a sibling adapter
whose provider happens to pre-sum reasoning into the field it reads.

The reusable rule: cross-adapter consistency is a property of the destination contract, not
of any one source format. "The OpenAI path is correct" does not transfer to the Gemini path,
because the two providers put the same billed quantity in differently-shaped fields; the
only way to keep the record honest is to reconcile each adapter against the destination's
definition of output, provider by provider.

## Trigger

Any `generate` call to a Gemini thinking model (2.5 or 3.x with thinking enabled) whose
`usage_metadata` carries a non-zero `thoughts_token_count`. The returned cost then reports
`output_tokens = candidates_token_count` and a cost computed from it, both short by the
thoughts-token count times the output price. Non-thinking models report no thoughts field,
so their cost is unaffected, which is why the shipped tests never caught it: their
`usage_metadata` fixtures set `candidates_token_count` alone and assert `output_tokens`
equals it, staying inside the case where the two counts coincide.

## Repro

Reproduced in a clean `python:3.12-slim` container against HEAD, module provenance confirmed
by `__file__`, with a mock `usage_metadata` and no live Gemini API. A fake response with
`prompt_token_count=1000`, `candidates_token_count=500`, `thoughts_token_count=300` is run
through `_token_cost` for `gemini-2.5-flash`: on HEAD the record reports `output_tokens=500`
and a cost priced on 500 output tokens; with the fix (`output_tokens +=
thoughts_token_count or 0`) it reports `output_tokens=800` and a cost priced on 800, so both
the count and the cost now reflect what Google bills. The added regression test
(`test_gemini_generate_folds_thinking_tokens_into_output_cost`) asserts `cost.output_tokens
== 800` and the matching cost; it fails on `main` and passes on the branch, and the existing
Gemini tests (whose fixtures carry no thoughts field, so the fold adds zero) stay green.
