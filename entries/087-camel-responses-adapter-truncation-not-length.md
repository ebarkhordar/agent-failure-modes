# A Responses→ChatCompletions adapter maps only the success terminal states, so a token-truncated call reports finish_reason='stop' instead of 'length'

- **Repo:** camel-ai/camel
- **Surface:** `camel/models/openai_responses_adapter.py`,
  `response_to_chat_completion` (non-streaming) and the streaming
  `response.completed` handler
- **Class:** message-conversion boundaries
- **Fix:** [PR #4216](https://github.com/camel-ai/camel/pull/4216) (in review;
  issue [#4215](https://github.com/camel-ai/camel/issues/4215))

## Root cause

The adapter's job is to make an OpenAI Responses-API call look like a Chat
Completions result, and `finish_reason` is part of that contract. The mapping reads
the response's output items to decide between `'tool_calls'` and `'stop'`, but it
never inspects `response.status` or `response.incomplete_details`. A Responses call
truncated at `max_output_tokens` comes back with `status == 'incomplete'` and
`incomplete_details.reason == 'max_output_tokens'`; because the mapping only
enumerates the success terminal states, that case falls through to `'stop'`, so a
truncated completion is reported as one that finished normally.

Streaming is worse: there is no `response.incomplete` / `response.failed` branch at
all, so a truncated stream hits the generic safety fallback, which both emits
`finish_reason='stop'` and never sets `state.usage` — the truncation signal *and* the
token usage are lost on that path. The two code paths disagree with each other as
well as with the contract: the same truncation reads differently depending on whether
the caller streamed.

## Invariant violated

An adapter that claims equivalence between two response formats must map every
*terminal* state of the source onto the target's vocabulary, not only the success
ones. `finish_reason='length'` is the single normalized signal that a completion was
cut at the token cap — camel documents it as the truncation contract and its
`anthropic_model` maps `max_tokens` to it — so collapsing `status=='incomplete'` to
`'stop'` tells the caller the model stopped on its own when it was actually
truncated, and any continue/retry loop that keys on `'length'` never fires. A mapping
enumerated over only `completed` is incomplete at exactly the state the caller most
needs to distinguish. And when a format is converted on two paths (streaming and
non-streaming), a terminal-state branch present on one path must exist on the other,
or the same source state yields two different `finish_reason`s — and here the missing
streaming branch also drops the usage record the non-streaming path preserves.

## Trigger

Any camel call routed through the OpenAI Responses adapter that hits
`max_output_tokens` (`status='incomplete'`). Non-streaming reports
`finish_reason='stop'` instead of `'length'`; streaming reports `'stop'` and loses
`usage`.

## Repro

The mapping is a pure function reachable from `openai_model.py` on both the
non-streaming and streaming paths; the repro mocks a Responses object with
`status='incomplete'` and `incomplete_details.reason='max_output_tokens'`, no API key
required. On master the adapter returns `finish_reason='stop'` (and, streaming, empty
usage); the fix adds the `status`/`incomplete_details` mapping to the non-streaming
path and an `incomplete`/`failed` branch that preserves usage to the streaming path,
each covered by its own test. The added suite has 7 tests passing on the branch; 4
fail against the master adapter.
