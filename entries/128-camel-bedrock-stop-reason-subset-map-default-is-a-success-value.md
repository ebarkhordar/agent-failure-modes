# Four of nine source values are listed and the default is itself a success value, so a guardrail block reports as a normal completion

- **Repo:** camel-ai/camel
- **Surface:** `camel/models/aws_bedrock_converse_model.py`, `_map_stop_reason` (`:621` at
  master `97b6e2dd`), called from `_convert_converse_to_openai_response` (`:695`,
  non-streaming) and `_converse_stream_to_openai_chunks` (`:837`, streaming), against
  `CohereModel._map_finish_reason` in `camel/models/cohere_model.py` (`:133`)
- **Class:** message-conversion boundaries
- **Report:** [issue #4261](https://github.com/camel-ai/camel/issues/4261) (open)
- **Fix:** [PR #4262](https://github.com/camel-ai/camel/pull/4262) (open, in review; no
  maintainer has commented as of 2026-08-06)

## Root cause

`_map_stop_reason` is one dict literal and one `.get`:

```python
return {
    "end_turn": "stop",
    "max_tokens": "length",
    "stop_sequence": "stop",
    "tool_use": "tool_calls",
}.get(stop_reason or "", "stop")
```

Bedrock's Converse API returns nine `stopReason` values, not four. That set is not
declared anywhere in camel. It lives in botocore's own service model
(`bedrock-runtime/2023-09-30/service-2.json.gz`), which at botocore 1.43.65 reports
`end_turn`, `tool_use`, `max_tokens`, `stop_sequence`, `guardrail_intervened`,
`content_filtered`, `malformed_model_output`, `malformed_tool_use` and
`model_context_window_exceeded`.

Five of the nine reach the `.get` default, and three of those five have a distinct
meaning in the target vocabulary that is lost: `content_filtered` and
`guardrail_intervened` are `content_filter`, and `model_context_window_exceeded` is
`length`. Both call sites route through this one helper, so streaming and non-streaming
report the same wrong value rather than disagreeing with each other.

## Invariant violated

**A lookup table's default has to be distinguishable from a mapped result.** `"stop"` is
not a sentinel here, it is the most common correct output of the same table. An input the
table has never heard of therefore produces an answer that is well formed, plausible, and
identical to the answer for a turn that really did finish normally. Nothing separates
"the model stopped" from "this normalization layer does not know this value": no
exception, no log line, no null.

**A table's completeness is a question about a set that lives outside the repository.**
camel names four values; nothing in camel names nine. No type, test or lint in the
project can observe the five that are missing, because the authority for the source
vocabulary is a data file shipped inside a dependency and versioned by that dependency. A
reviewer comparing the table against itself sees a coherent table. The instrument that
finds the gap is to read the enum out of the dependency at a pinned version and drive
every member through the real conversion, which is why the check belongs in the diff as a
differential over the whole enum rather than as a test per value someone happened to
think of.

**A subset mapping over an external enum loses the rare terminal states first, and the
rare terminal states are the ones callers branch on.** The values a table author writes
down are the ones they have seen: normal completion, tool call, token cap. What gets left
out is the exceptional tail, here content filtering, guardrail intervention and
context-window overflow. The failure is therefore not spread evenly over the enum. It
concentrates on exactly the events a caller keys on to detect a safety block or to
continue a truncated generation, and reports each of them as the happy path.

The same repository already holds the opposite answer to the same question.
`CohereModel._map_finish_reason` (`:133`) raises `ValueError` on a reason it does not
know and `RuntimeError` on the two that signal generation failure, over a module-level
constant map. Two providers, one codebase, two contradictory policies for an unknown
source value, and nothing makes the disagreement visible, because each helper reads as a
reasonable mapping on its own.

## Trigger

Any `AWSBedrockConverseModel` call, streaming or not, whose turn ends in a guardrail
intervention, a content filter, or a context-window overflow. `finish_reason` comes back
`"stop"`. Code branching on `"content_filter"` cannot see the safety block, and code
branching on `"length"` to continue a truncated generation never fires.

## Repro

Clean `python:3.11-slim`, camel installed from source at master `97b6e2dd`, botocore
1.43.65, with `camel.models.aws_bedrock_converse_model.__file__` resolving into the tree
under test. A Converse response is a plain dict, so both conversion methods are driven
directly with no AWS credentials and no network call.

All nine enum members, read from the installed service model rather than typed by hand,
through both real conversion methods:

```
                                 before             after
end_turn                         'stop'             'stop'
tool_use                         'tool_calls'       'tool_calls'
max_tokens                       'length'           'length'
stop_sequence                    'stop'             'stop'
guardrail_intervened             'stop'             'content_filter'
content_filtered                 'stop'             'content_filter'
malformed_model_output           'stop'             'stop'   (out of scope)
malformed_tool_use               'stop'             'stop'   (out of scope)
model_context_window_exceeded    'stop'             'length'
```

Six parametrized cases (three values across two paths) fail on master and pass on the
branch. A seventh pins `malformed_model_output` and `malformed_tool_use` on the default
and passes on both, which is what makes leaving them there a decision rather than an
oversight: neither has a defensible equivalent in the OpenAI vocabulary, so the PR names
both and leaves the call to the maintainer.

**Scope of what was measured.** No Bedrock endpoint was called. This is a statement about
the `finish_reason` camel computes from a `stopReason` string, not an observation of when
Bedrock emits each value. Whether the two out-of-scope values should map somewhere is not
settled here.

Verified 2026-08-05 at master `97b6e2dd`. Reported and fixed on a branch, not merged.
