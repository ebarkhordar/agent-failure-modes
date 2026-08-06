# A first-class refusal detected only when an optional free-text field happens to be set

- **Repo:** Giskard-AI/giskard-oss
- **Surface:** `giskard/llm/translators/anthropic.py::AnthropicChatTranslator.from_anthropic`
- **Class:** message-conversion boundaries
- **Fix:** merged upstream in
  [PR #2647](https://github.com/Giskard-AI/giskard-oss/pull/2647) on 2026-08-03, which
  also closed issue [#2615](https://github.com/Giskard-AI/giskard-oss/issues/2615) as
  completed. Our [PR #2616](https://github.com/Giskard-AI/giskard-oss/pull/2616) was
  closed unmerged on 2026-07-29 by a maintainer who folded this and the silent-param drop
  of entry [041](041-giskard-pydantic-extra-ignore-drops-params.md) into a single outside
  contribution, [PR #2623](https://github.com/Giskard-AI/giskard-oss/pull/2623). #2623 was
  in turn closed unmerged on 2026-07-31 in favour of #2647, a maintainer's copy of the same
  change with the formatting this repo's CI requires. The root cause below was not disputed
  at any point in that chain, and the merged tests assert exactly the two shapes it named as
  invisible.

## Root cause

Described here as it stood before #2647 merged; the repro table and the ref it pins
(`175670e`) are pre-fix.

Giskard detects a model refusal with one rule, defined once and consumed by the
workflow layer: a completion is a refusal when `finish_reason == 'refusal'` or
`message.is_refusal`, the latter being true whenever `message.refusal is not None`
(it has a second branch, for a `RefusalContent` block, which the Anthropic inbound
path cannot produce because its block converter emits only text and tool calls).
Three provider adapters feed that rule. OpenAI
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

## What shipped

#2647 replaces the `explanation`-only read with a fallback chain, so the two failing
rows above now detect:

```python
if raw.stop_reason == "refusal":
    details = raw.stop_details
    refusal_out = (
        (details.explanation or details.category) if details is not None else None
    ) or "refusal"
```

The trailing `or "refusal"` is the part that matters to the invariant: it makes the
field unconditionally non-`None` whenever the provider signalled a refusal, so
detection no longer depends on any optional field being populated. Both previously
invisible shapes are pinned by new tests, one for `stop_details = None` and one for
`category` without `explanation`.

`FINISH_REASON_MAP` was not touched, so `refusal` still maps to `finish_reason
== "stop"` and the merged test asserts that. That is not a contradiction of the
analysis above, it is the reason the fix had to go where it did: the
`finish_reason == 'refusal'` half of the detection rule is unreachable on this
path by design, which leaves `message.refusal` as the only carrier available.
