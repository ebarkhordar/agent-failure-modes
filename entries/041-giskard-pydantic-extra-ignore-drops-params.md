# A validation model used as a parameter filter drops sampling settings in silence

- **Repo:** Giskard-AI/giskard-oss
- **Surface:** `giskard/llm/translators/anthropic.py::AnthropicChatTranslator.to_anthropic`
- **Class:** message-conversion boundary
- **Report:** [issue #2613](https://github.com/Giskard-AI/giskard-oss/issues/2613)
  (deterministic repro, fix suggested)

## Root cause

`AnthropicProvider.complete(..., **params)` discards every completion param it
does not declare. No error and no warning: the request reaches the API without
them.

`to_anthropic()` funnels params through `AnthropicChatConfigParams`, a pydantic
model whose declared fields are `model, messages, max_tokens, tools, system,
temperature, timeout, output_config`. With pydantic's default `extra="ignore"`,
anything else is dropped without a sound. `top_p`, `top_k` and `stop_sequences`
all vanish.

The sibling OpenAI translator handles the identical situation by telling the
caller (`openai_chat.py`):

```python
unknown = set(params) - KNOWN_COMPLETION_PARAMS
if unknown:
    logger.warning("%s provider: ignoring unknown completion params: %s", PROVIDER, sorted(unknown))
```

Both provider `__init__` methods also warn on unknown kwargs. The convention in
this repo is already "unsupported input is ignored, but the caller is told". The
Anthropic completion path is the one place that stays silent.

## Invariant violated

A translator that cannot honor a caller's parameter must say so. Dropping is a
legitimate policy; dropping silently is not, because the caller has no way to
distinguish "applied" from "ignored" and the two produce different results.

What makes this one worth recording is that nobody decided it. `extra="ignore"` is
a pydantic **default**, so the silence is not a line of code that a reviewer could
have objected to. It is the absence of one. Validation libraries are built to
reject or coerce; pointed at a dict of caller params they quietly become a
whitelist, and the discarding happens inside the library, driven by a config value
no one in this repo wrote. There is no drop site to read. Grepping for where the
params go finds nothing, which reads like "nothing happens to them", which is the
opposite of the truth.

The corollary is a cheap and general audit. A repo with two implementations of one
boundary has already written down its own convention twice, and the disagreement
between them is free to find. This is not a missing convention; it is one sibling
that fell out of an existing one. Diffing the two translators is a five-minute
check that no amount of reading either one in isolation would have produced,
because each is internally coherent.

## Trigger

Any Anthropic completion in Giskard that passes a param outside the eight declared
fields. Giskard exists to evaluate LLM behavior, so sampling settings are part of
the experiment: a user who sets `top_p` or `stop_sequences` gets an eval run that
silently used different settings than they configured, and nothing in the logs
says so. The failure is invisible at exactly the moment it changes results.

## Repro

Clean `python:3.12-slim` container against `175670e`, run against the repo source
with `__file__` printed to confirm provenance:

```python
from collections.abc import Sequence
from pydantic import TypeAdapter
from giskard.llm.translators.anthropic import AnthropicChatTranslator
from giskard.llm.types import ChatMessage

msgs = TypeAdapter(Sequence[ChatMessage]).validate_python([{"role": "user", "content": "hi"}])
out = AnthropicChatTranslator.to_anthropic(
    "claude-sonnet-4-5", msgs,
    top_p=0.5, top_k=40, stop_sequences=["STOP"],
)
print(sorted(out.keys()))
```

```
PROVENANCE: /src/libs/giskard-llm/src/giskard/llm/translators/anthropic.py
keys actually sent to the Anthropic SDK: ['max_tokens', 'messages', 'model']
  top_p           survived? False
  top_k           survived? False
  stop_sequences  survived? False
```

**Not verified:** no live Anthropic call was made. The claim is about what
`to_anthropic` hands to the SDK, which is where the params are lost, not about how
the API responds.

## Suggested fix

Mirror the OpenAI translator: compare incoming params against the known set and
`logger.warning` the unknown ones before dropping them. That keeps the current
whitelist scope and removes only the silence. Whether the Anthropic path should
instead forward `top_p`, `top_k` and `stop_sequences` to the SDK is a larger call
that belongs to the maintainers, and the issue asks them to pick the direction
rather than assuming it.
