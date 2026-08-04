# output_tokens set from the visible-only count while total keeps the billed total, so input + output != total when the model thinks

- **Repo:** Giskard-AI/giskard-oss
- **Surface:** `libs/giskard-llm/src/giskard/llm/translators/google_chat.py` (`GoogleChatTranslator.from_google`)
- **Class:** streaming & usage accounting
- **Fix:** [PR #2653](https://github.com/Giskard-AI/giskard-oss/pull/2653) (open,
  unreviewed, and conflicting on `README.md` as of 2026-08-04, so the defect is still
  live on `main`). A maintainer opened it on 2026-07-29 carrying this change plus the
  lint formatting the fork CI requires, and we closed our own
  [PR #2622](https://github.com/Giskard-AI/giskard-oss/pull/2622) unmerged in its favour
  once we had compared the two branches: `google_chat.py` and
  `test_google_chat_return.py` are identical between them. Self-discovered, no issue.

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
record `input_tokens + output_tokens == total_tokens` must hold. Each token class the
provider bills for belongs on the side that produced it: reasoning tokens are generated
output, tool-use-prompt tokens are prompt-side input. Dropping either from its side
while leaving it in `total_tokens` reports a self-inconsistent record that
under-attributes cost. When several provider translators normalize into
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
35)` and satisfy the identity. The fix folds `thoughts_token_count` into
`output_tokens` and `tool_use_prompt_token_count` into `input_tokens`, so the Google
path matches its siblings; the existing 27 google-translator tests pass on HEAD and
after the fix.

Two corrections to this entry, both from re-reading the shipped diff on 2026-08-04. It
said "the fix folds the reasoning (and tool-use-prompt) tokens into `output_tokens`",
and called both classes "output the caller pays for". Only the reasoning tokens go to
the output side. `tool_use_prompt_token_count` is prompt-side and the diff has always
put it in `input_tokens`, as our own PR body said; the entry contradicted the code it
was describing from the day it was written. The identity claim and the repro numbers
are unaffected, since both classes are missing from the mapped fields either way.

The differential above ran against the first form of the fix. A `gemini-code-assist`
bot review then pointed out that reading `thoughts_token_count` and
`tool_use_prompt_token_count` as plain attributes raises `AttributeError` on
google-genai versions predating those fields, so the shipped version reads all five
through `getattr(um, "...", 0) or 0`. That is a compatibility hardening, not a change
to the accounting.
