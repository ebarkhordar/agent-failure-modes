# One `choices` entry emitted per tool call, so a downstream reader of `choices[0]` silently drops every parallel call but the first

- **Repo:** camel-ai/camel
- **Surface:** `camel/models/cohere_model.py` (`CohereModel._to_openai_response`)
- **Class:** provider-adapter response mapping / parallel tool calls
- **Fix:** [PR #4187](https://github.com/camel-ai/camel/pull/4187) (merged),
  reported as [issue #4185](https://github.com/camel-ai/camel/issues/4185)

## Root cause

`_to_openai_response` converts a Cohere `ChatResponse` into an OpenAI-shaped
`ChatCompletion` by looping over `response.message.tool_calls` and appending one
`choice` per call. Command R/R+ supports parallel tool calling, so a turn that
requests N tools produces N `choices`, each carrying exactly one call, instead of
one `choices[0]` carrying all N.

Nothing downstream reads past the first choice. `camel/agents/chat_agent.py` takes
`response.choices[0].message.tool_calls`, so calls 2..N are discarded with no error
and no warning: the agent executes one tool, and the model's other requests for that
turn simply never happen. The sibling adapter `camel/models/mistral_model.py` builds a
single choice whose `message.tool_calls` is the full list, which is the shape the rest
of the codebase assumes. A secondary defect sat at the same site: each emitted choice
carried `index=None`, where the OpenAI schema requires an integer.

## Invariant violated

An OpenAI-shaped `ChatCompletion` for one generation has exactly one `choices[0]`,
and `choices[0].message.tool_calls` holds every tool call from that turn. `choices`
is the axis for alternative *completions* (n>1 sampling), not for the parts of a
single completion. An adapter that overloads it to enumerate tool calls produces a
structurally valid object that means something different from what it says, so every
consumer written against the schema reads it correctly and still gets the wrong
answer. When several provider adapters normalize into one shared response shape,
each must satisfy that shape's own semantics independently, because consumers read
the shape and never the provider it came from.

## Trigger

Any Cohere turn that returns more than one tool call. Single-call turns are
indistinguishable from correct behavior (one choice, one call), which is why this
survived: the common path is exactly right, and the failure needs parallel tool
calling on a secondary provider to appear at all.

## Repro

`python:3.12-slim` container at master `0b9d986`, `__file__` provenance asserted on
both sides. A mocked Cohere `ChatResponse` carrying two tool calls is passed through
`CohereModel._to_openai_response`. On master: `len(result.choices) == 2` (expected 1)
and `choices[0].index is None` (expected 0), 2 failed / 1 passed, where the passing
case is the single-call control. After the fix, mirroring the mistral adapter's list
comprehension and setting `index=0`: 3 passed. Pure function, no network. The five
pre-existing `test_cohere_model` failures need `COHERE_API_KEY` and are identical on
both sides.
