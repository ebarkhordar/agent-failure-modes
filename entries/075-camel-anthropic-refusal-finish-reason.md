# A safety refusal leaks off-contract on one path and silently coerces to "stop" on its twin, because the same mapping is hand-inlined in two places

- **Repo:** camel-ai/camel
- **Surface:** `camel/models/anthropic_model.py`, the two `stop_reason` to `finish_reason` mappings, non-streaming in `_convert_anthropic_to_openai_response` (the `else` branch around line 712) and streaming in `_convert_anthropic_stream_to_openai_chunk` (the `message_delta` branch around line 940)
- **Class:** message-conversion boundaries
- **Fix:** [PR #4210](https://github.com/camel-ai/camel/pull/4210) (in review; issue
  [#4209](https://github.com/camel-ai/camel/issues/4209))

## Root cause

`AnthropicModel` normalizes Anthropic's `stop_reason` onto the OpenAI
`finish_reason` vocabulary in two separate hand-written `if/elif` chains, and
neither chain had a case for `refusal`. Anthropic 0.89.0 declares
`StopReason = Literal['end_turn', 'max_tokens', 'stop_sequence', 'tool_use',
'pause_turn', 'refusal']`; `refusal` is the value the API returns when it declines
a generation on safety grounds. The two paths fail it differently, and that is the
interesting part:

- Non-streaming: the `else` branch is `finish_reason = stop_reason`, so the raw
  string `'refusal'` passes straight through into a field whose OpenAI contract is
  `Literal['stop', 'length', 'tool_calls', 'content_filter', 'function_call']`. The
  emitted value is off-contract, but at least it is visibly wrong.
- Streaming: the `message_delta` branch has no `refusal` case at all, so
  `finish_reason` is never assigned and stays `None`; the later `message_stop`
  event then computes `None if finish_reason_sent else "stop"` and reports `"stop"`.
  A safety refusal is delivered to the caller as an ordinary, successful
  completion.

Two sibling adapters in the same codebase already centralize this translation:
`aws_bedrock_converse_model._map_stop_reason` and `cohere_model._map_finish_reason`
(the latter from the merged [#4202](https://github.com/camel-ai/camel/pull/4202)).
The Anthropic adapter is the outlier that kept the mapping inline, and inlining it
twice let the two copies drift into two different wrong answers for the same input.

## Invariant violated

When one mapping is copied inline into two code paths, the paths do not stay in
agreement; each is edited on its own occasion, and a value added to neither is
handled by neither, in two different ways. A missing case is not uniformly bad: a
pass-through `else` leaks an off-contract value that a strict consumer can still
reject, while a chain with no `else` leaves the target unset and lets a downstream
default (`message_stop` to `"stop"`) fabricate a plausible, in-contract, wrong
value. The second is the more dangerous failure, because the output is now a legal
member of the target vocabulary and nothing downstream has any signal that it is
incorrect. The reusable rule: a vocabulary translation that must hold across
formats belongs in one function that enumerates the source vocabulary exhaustively,
so that a newly added source value fails loudly in one place rather than silently
in each inline copy, each in its own way.

## Trigger

Any Anthropic completion whose `stop_reason` is `refusal`, a model safety decline.
On the non-streaming path the caller sees `finish_reason='refusal'`, a value the
OpenAI schema forbids; on the streaming path the caller sees `finish_reason='stop'`,
indistinguishable from a normal finish. `pause_turn`, the other non-OpenAI value in
the enum, signals a paused (non-terminal) turn with no OpenAI equivalent and is left
out of scope by the fix; only `refusal` maps to `content_filter`.

## Repro

Reproduced against HEAD with `camel` installed from source and `anthropic` 0.89.0
pinned, `import camel.models.anthropic_model` resolving to the in-tree file by
`__file__`. The exact `StopReason` enum was read off the installed SDK type this
session rather than assumed. A mock Anthropic response and a mock stream with
`stop_reason='refusal'` were driven through the two real conversion methods; no
live API key is needed because both are pure conversion functions. Before the fix
the non-streaming path returns `finish_reason='refusal'` and the streaming path
returns `'stop'`; after mapping `refusal` to `content_filter` in both, each returns
`'content_filter'`. The added regression tests assert membership in the OpenAI value
set for both paths, failing on `main` and passing on the branch; the existing
Anthropic tests (whose fixtures use `end_turn` / `tool_use`) stay green.
