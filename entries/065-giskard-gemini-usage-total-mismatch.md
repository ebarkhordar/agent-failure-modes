# output_tokens set from the visible-only count while total keeps the billed total, so input + output != total when the model thinks

- **Repo:** Giskard-AI/giskard-oss
- **Surface:** `libs/giskard-llm/src/giskard/llm/translators/google_chat.py` (`GoogleChatTranslator.from_google`)
- **Class:** streaming & usage accounting
- **Fix:** [PR #2622](https://github.com/Giskard-AI/giskard-oss/pull/2622) (in review;
  self-discovered, no issue)

## Root cause

`from_google` maps google-genai's `usage_metadata` onto giskard's usage type by
setting `input_tokens = prompt_token_count`, `output_tokens =
candidates_token_count`, and `total_tokens = total_token_count`. Those three come
from different definitions. Google's own field documentation defines
`total_token_count` as the sum of `prompt_token_count`, `candidates_token_count`,
`tool_use_prompt_token_count`, and `thoughts_token_count`, where
`thoughts_token_count` is the model's generated reasoning output. So whenever a
Gemini model produces thinking tokens (or tool-use prompt tokens),
`candidates_token_count` counts only the visible answer while `total_token_count`
counts everything billed, and giskard's own `input_tokens + output_tokens` no
longer equals its `total_tokens`.

The Anthropic, OpenAI chat, and OpenAI responses translators in the same package
all fold every generated-and-billed output segment into `output_tokens`, so they
keep the identity. The Google translator is the sole violator, and the discrepancy
scales exactly with how much the model reasoned, so it is zero on trivial prompts
and large on the reasoning-heavy calls a user most wants accounted.

## Invariant violated

`output_tokens` must count every token the model generated and was billed for as
output, not only the tokens that reached the visible response, and across a usage
record `input_tokens + output_tokens == total_tokens` must hold. Reasoning and
tool-use-prompt tokens are output the caller pays for; dropping them from
`output_tokens` while leaving them in `total_tokens` reports a self-inconsistent
record that under-attributes cost. When several provider translators normalize into
one shared usage shape, each must satisfy that shape's arithmetic identity on its
own, because a downstream cost or budget consumer reads the identity, not the
provider-specific field names it came from.

## Trigger

Any Gemini call that emits `thoughts_token_count` (thinking enabled) or
`tool_use_prompt_token_count`. The translated usage then has `input_tokens +
output_tokens < total_tokens` by exactly the reasoning/tool-prompt token count.

## Repro

`python:3.13-slim` container at HEAD `175670e`, giskard-llm and giskard-core on the
path, `google-genai`/`anthropic`/`openai` installed. One scenario (prompt 10,
visible 5, reasoning 20, billed total 35) driven through all four translators:
`google/chat` returns `(input=10, output=5, total=35)`, so `input + output = 15 !=
35`; `anthropic/chat`, `openai/chat`, and `openai/response` all return `(10, 25,
35)` and satisfy the identity. The fix folds the reasoning (and tool-use-prompt)
tokens into `output_tokens` so the Google path matches its siblings; the existing
27 google-translator tests pass on HEAD and after the fix.
