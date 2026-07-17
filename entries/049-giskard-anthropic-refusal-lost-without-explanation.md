# A first-class refusal detected only when an optional free-text field happens to be set

- **Repo:** Giskard-AI/giskard-oss
- **Surface:** `giskard/llm/translators/anthropic.py::AnthropicChatTranslator.from_anthropic`
- **Class:** message-conversion boundary
- **Fix:** [PR #2616](https://github.com/Giskard-AI/giskard-oss/pull/2616)
  (in review; issue [#2615](https://github.com/Giskard-AI/giskard-oss/issues/2615))

## Root cause

Giskard detects a model refusal with one rule, defined once and consumed by the
workflow layer: a completion is a refusal when `finish_reason == 'refusal'` or
`message.refusal is not None`. Three provider adapters feed that rule. OpenAI
satisfies it directly, because the API returns `message.refusal`. Google
satisfies it by mapping its `refusal` finish reason through `FINISH_REASON_MAP`
and carrying the reason in `finish_message`. Anthropic satisfies it only by
accident.

`from_anthropic` populates `message.refusal` only when the SDK's optional
`stop_details.explanation` free-text string is present. But on a refusal the
Anthropic SDK sets `stop_reason == 'refusal'` as a first-class value, with
`stop_details` typed `Optional[RefusalStopDetails] = None`, and inside it
`explanation: Optional[str] = None` while the structured reason lives in a
separate `category` field the translator ignores entirely. So a fully populated,
non-degenerate refusal that carries its reason in `category` and no free-text
`explanation` produces `message.refusal = None`. The Anthropic refusal reason
does not reach `finish_reason` either, so both halves of the detection rule read
false. The refusal is invisible to the one place that checks.

## Invariant violated

When several adapters feed one consumer, every adapter must satisfy the
consumer's contract by construction, not by luck. Here the contract is "a
provider-signalled refusal leaves the response in a state the detector
recognizes", and two of three providers meet it structurally while the third
meets it only when an unrelated optional field is populated.

The sharper rule is about which field carries the signal. A refusal is a
first-class status: the SDK exposes it as `stop_reason == 'refusal'`, a value
that is always present on a refusal. `explanation` is an optional human-readable
gloss on that status. Detecting the status by reading the gloss inverts the
dependency, so a mandatory signal becomes conditional on an optional one, and
detection degrades to chance. A signal that must always be observed has to be
read from a field that is always set.

## Trigger

Any Anthropic refusal whose `stop_details.explanation` is absent, which includes
the plain `stop_details = None` case (the SDK default) and the structured case
where the reason is in `category`. The refusal then falls through to
`output_model.model_validate_json(message.text or '')`, so a refused generation
surfaces as a pydantic `ValidationError` on empty or apology text instead of the
`ModelRefusalError` the workflow raises for OpenAI and Google. For a tool whose
job is to evaluate model behavior, a refusal is the signal, not an error to be
swallowed: it gets scored as a malformed result.

## Repro

Clean `python:3.12-slim` container against `175670e`, provenance by sha256
identity between the source tree and the installed module (stronger than
`__file__`), anthropic SDK 0.116.0. The refusal shapes are built as SDK fixtures,
the same methodology the repo's own tests use to construct messages:

```
OPENAI     message.refusal="I can't help with that."           detected=True
GOOGLE     finish_reason='refusal'                             detected=True
ANTHROPIC  stop_details=None                                   detected=False
ANTHROPIC  stop_details=RefusalStopDetails(category='bio')     detected=False
ANTHROPIC  stop_details.explanation='Policy decline.'          detected=True
```

The last row is the only Anthropic shape that passes, and it is the sole shape
the existing Anthropic test exercises, which is why the gap survived a green
suite.

**Not verified:** the live Anthropic API was not called. The claim is confined to
what `from_anthropic` does with each SDK-valid refusal shape, not to which shape
the API emits most often in production.
