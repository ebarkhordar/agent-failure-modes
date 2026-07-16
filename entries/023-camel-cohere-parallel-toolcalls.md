# Parallel tool calls fanned onto the alternatives axis, so all but the first are dropped

- **Repo:** camel-ai/camel
- **Surface:** `camel/models/cohere_model.py::CohereModel._to_openai_response`
- **Class:** message-conversion boundary
- **Fix:** [PR #4187](https://github.com/camel-ai/camel/pull/4187) (in review; issue [#4185](https://github.com/camel-ai/camel/issues/4185))

## Root cause

`_to_openai_response` normalizes a Cohere v2 response into the OpenAI shape by
looping `for tool_call in tool_calls` and appending one `choice` per tool call.
N parallel tool calls from one model turn therefore become N choices, each
carrying exactly one call. Downstream, `camel/agents/chat_agent.py:3959` builds
`tool_call_requests` only from `response.choices[0].message.tool_calls`, which
is what the OpenAI contract entitles it to do. Calls 2..N are silently
discarded: no error, no warning, and the agent simply never runs those tools.

Two more defects sit at the same site. `index` is left `None` on each choice,
violating the OpenAI integer-index shape; and because every choice repeats
`response.message.tool_plan` as its content, `_handle_batch_response` (which
loops all choices for output messages but only `choices[0]` for tool calls)
emits N identical assistant messages.

## Invariant violated

An adapter that renders one provider's response into another provider's shape
must preserve that shape's cardinality, not just its field names. In the OpenAI
shape, `choices` is the axis of *alternative* generations, and
`choices[0].message.tool_calls` is the list of every call in that one turn.
These are different axes. Encoding a within-turn list onto the alternatives axis
is a category error, and it fails silently rather than loudly, because every
consumer downstream is contractually correct to read `choices[0]` and ignore the
rest. The sibling adapter `camel/models/mistral_model.py:152-166` builds a single
choice holding all calls; the Cohere adapter is the outlier.

## Trigger

Any Cohere turn returning more than one tool call. Cohere v2 documents parallel
tool calling ("the model can determine that more than one tool call is
required"), and `AssistantMessageResponse.tool_calls` is a `List[ToolCallV2]`,
so N > 1 is documented provider behavior rather than a degenerate input. A
single call maps to a single choice and behaves correctly, which is exactly why
the existing tests miss it.

## Repro

Clean `python:3.12-slim` container against master HEAD `0b9d986`, camel
`0.2.91a5` installed editable from source, cohere `5.21.1` (camel pins
`cohere>=5.11.0,<6`). Provenance asserted:
`camel.models.cohere_model.__file__` resolves to `/build/camel/models/cohere_model.py`.
The input is built from the real runtime type `cohere.v2.types.V2ChatResponse`,
which is what `cohere.ClientV2.chat` returns (camel's `ChatResponse` annotation
is `TYPE_CHECKING`-only and does not exist at runtime in the pinned range).

Two tool calls in one turn produce `choices=2`, `index=[None, None]`, and a
dispatch list of `[('get_weather', {'city': 'Toronto'})]`: the Berlin call is
gone, and two duplicate assistant messages are emitted. Control: one tool call
gives one choice and one dispatch, so the common path is unaffected and a fix is
regression-free there. Three calls give three choices.

**Not verified:** no live Cohere API call was made (no key on this runner), so
the multi-call response is constructed from the SDK's own `V2ChatResponse` type
rather than captured off the wire. The `MistralModel` comparison is cited as
code, not executed as a behavioral differential. Both gaps are stated in the
issue itself.

## Lesson

A conversion bug that only appears at N > 1 is invisible to every test written
against N = 1, and N = 1 is the shape every quickstart demonstrates. When an
adapter's output shape has an axis whose length the tests never vary, that axis
is where the untested mapping lives.
