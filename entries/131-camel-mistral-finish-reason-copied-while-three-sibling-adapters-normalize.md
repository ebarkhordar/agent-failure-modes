# Three of four sibling adapters grew a normalization helper, so the fourth's raw copy reads as the house style rather than the omission

- **Repo:** camel-ai/camel
- **Surface:** `camel/models/mistral_model.py`, the `finish_reason=` argument inside
  `MistralModel._to_openai_response` (`:162-164` at master `97b6e2dd`), against
  `CohereModel._map_finish_reason` (`camel/models/cohere_model.py:133`), the `stop_reason`
  branch in `AnthropicModel`, and `AWSBedrockConverseModel._map_stop_reason`
- **Class:** message-conversion boundaries
- **Report:** [issue #4266](https://github.com/camel-ai/camel/issues/4266) (open)
- **Fix:** [PR #4267](https://github.com/camel-ai/camel/pull/4267) (open, in review; only a
  review bot has commented as of 2026-08-07)

## Root cause

`_to_openai_response` builds the `ChatCompletion` camel hands back and passes the provider's
value straight into the OpenAI-shaped field:

```python
finish_reason=response.choices[0].finish_reason
if response.choices[0].finish_reason
else None,
```

`mistralai` 1.12.4, inside camel's own `mistralai>=1.1.0,<2` pin, types that field as
`Union[Literal["stop", "length", "model_length", "error", "tool_calls"], UnrecognizedStr]`.
OpenAI's `Choice.finish_reason` is `Literal['stop', 'length', 'tool_calls', 'content_filter',
'function_call']`. Three values are in both vocabularies and survive the copy. `model_length`
and `error` are not, and nothing rejects them, because the response is built with
`ChatCompletion.construct`, which is defined to skip validation.

The value does not stop at the adapter. `chat_agent.py:4015` rebuilds the list straight off
the returned object:

```python
finish_reasons = [
    str(choice.finish_reason) for choice in response.choices
]
```

and that reaches `info["finish_reasons"]` through `get_info_dict`, so the untranslated string
is visible to user code and not only to camel's internals. A `model_length` stop is a
truncated generation that nothing looking for `length` will find, and an `error` reason
arrives shaped like a generation that finished and happens to carry content.

## Invariant violated

**The presence of a normalization helper across a family of adapters is not evidence that the
family normalizes.** Camel already had three answers to this exact question when Mistral's
copy was written: Cohere has `_map_finish_reason`, Anthropic branches on `stop_reason`,
Bedrock has `_map_stop_reason`. An audit that asks "do the provider adapters normalize finish
reasons here?" reads those three, finds an established pattern, and certifies the whole
family. The question that has behaviour behind it is asked once per adapter, and the answer
for one of the four was no code at all.

**Absent code is the hardest omission to see, because there is no line to review.** A wrong
mapping is a hunk a reader can disagree with. A missing mapping is a field assignment whose
left and right sides are named the same thing, which is what correct interoperability code
also looks like. The reviewer's eye is drawn to the three files that have a helper, and the
one without a helper offers nothing to look at.

**"Has a mapping" and "maps correctly" are separate properties, and the corpus already holds
both failures in this one repo.** Entry [128](128-camel-bedrock-stop-reason-subset-map-default-is-a-success-value.md)
is the Bedrock adapter, which does have `_map_stop_reason`, listing four of the nine values
its API can return and defaulting the rest to `"stop"`. So of camel's four adapters, one
copies raw and one maps a subset into a default that is itself a valid success value. Counting
helpers finds neither. Only enumerating the source vocabulary against the target vocabulary,
per adapter, finds either. Entries [070](070-camel-cohere-finish-reason-vocabulary.md) and
[075](075-camel-anthropic-refusal-finish-reason.md) are the same boundary in the two adapters
that do normalize, which is what makes the family worth reading together rather than
separately.

## Trigger

Any `MistralModel` call whose generation stops for a reason outside the shared vocabulary:
`model_length` when the model runs into its own context limit, `error` when the generation
fails. No API key, no unusual configuration, and no error path in camel is involved. Both
values are ordinary outcomes of the provider camel is wrapping.

## Repro

Clean `python:3.11-slim` container, camel installed from source at master `97b6e2dd`,
`mistralai` 1.12.4, `openai` 2.53.0, with the module under test identified by sha256
(`c8d6e8be8f43d1f24294e68b54927304afc5480b3edfdd882c40df121e151c62`) between the source tree
and the installed package. No network and no API key: real `mistralai` response objects are
constructed directly and driven through the real `_to_openai_response`. All five values in the
SDK's enum:

```
stop          -> 'stop'
length        -> 'length'
model_length  -> 'model_length'
error         -> 'error'
tool_calls    -> 'tool_calls'
```

`ChatCompletion.model_validate` on the `model_length` output fails with `literal_error`;
the same call on a `'length'` output validates. That pair is the whole defect: the object
camel returns is one its own declared type rejects.

**What was not verified.** Mistral's API reference lists `finish_reason` without documenting
what each value means, so reading `model_length` as OpenAI's `length` comes from the value
name and the SDK type, not from a specification. That caveat is stated in the issue and the PR
rather than hidden, and the maintainers were invited to correct the reading. Whether `error`
should raise or map to a value is a policy choice, not a fact: the PR raises `RuntimeError`,
following the sibling `CohereModel._map_finish_reason`, and deliberately lets unknown values
pass through unchanged instead of raising, because `UnrecognizedStr` in the SDK type means a
value camel has not heard of is a later API addition rather than a fault.
