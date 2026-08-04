# A permissive union member keeps the payload as a dict, so the typed options model that would have caught the unmapped key is never built

- **Repo:** mozilla-ai/any-llm
- **Surface:** `src/any_llm/providers/ollama/ollama.py`, `_convert_completion_params` and the
  two call sites that hand its result to `options=` (`:206` streaming, `:274` non-streaming),
  against `ChatRequest.options` in the `ollama` SDK's `_types.py`
- **Class:** configuration wiring & documented contracts
- **Report:** reproduced publicly on
  [issue #1206](https://github.com/mozilla-ai/any-llm/issues/1206#issuecomment-5100726561),
  correcting the reporter's stated mechanism and widening the scope. No PR from us: nine
  keys survive the same gap, and whether each should be mapped or dropped with a warning is
  a maintainer's call, so the measurement went out as triage first.
- **Fix:** partial. [PR #1213](https://github.com/mozilla-ai/any-llm/pull/1213) by
  ShiroKSH, merged 2026-08-04, closing #1206 as completed. It maps `max_tokens` and
  `max_completion_tokens` onto `num_predict` (an explicit `num_predict` keeps priority, and
  a mismatch is logged). It covers `max_completion_tokens` too, which is what this entry
  argued the repair had to include. The mechanism the entry is about is untouched:
  `options=` still receives a plain dict, the permissive union member still keeps it one,
  and the other seven keys (`logit_bias`, `logprobs`, `n`, `parallel_tool_calls`,
  `tool_choice`, `top_logprobs`, `user`) still reach the daemon unmapped and unwarned.

## Root cause

Ollama names the output-length knob `num_predict`. Every other provider any-llm targets calls
it `max_tokens`, and `CompletionParams` uses the common name. The Ollama provider never
translates between the two, so the request goes out with `max_tokens` still spelled that way,
carrying a key the daemon does not know, and the caller's token limit has no effect.

The reported mechanism was that `ollama.Options` filters the unknown key out. It does not,
because `Options` is never constructed. `ChatRequest.options` is annotated
`Mapping[str, Any] | Options | None`, and pydantic's smart union matches the plain dict
against the permissive `Mapping` member, keeping it a dict. Nothing validates it, nothing
drops it, and it reaches the wire intact. Captured against a stub HTTP server on
`127.0.0.1:11434`:

```
POST /api/chat
options: {"max_tokens": 512, "num_ctx": 32000, "temperature": 0.1, "top_p": 0.9, "tool_choice": "auto"}
```

`max_tokens` is present, `num_predict` is absent, and the daemon ignores option keys it does
not recognize, so nothing on the path raises.

That distinction is not pedantic. It decides what a regression test may assert.
`Options(max_tokens=512).model_dump()` exercises a construction any-llm never performs, so a
test written against it passes on today's unfixed code. Only an assertion on the dict handed
to `client.chat` pins the real defect.

`max_tokens` is also not alone. Diffing `CompletionParams`'s fields against `Options`'s fields
at that HEAD, nine keys survive `_convert_completion_params` and reach `options=`:

```
logit_bias, logprobs, max_completion_tokens, max_tokens, n,
parallel_tool_calls, tool_choice, top_logprobs, user
```

`max_completion_tokens` is the one worth folding into the same repair. It is a public
parameter of `completion()` and `acompletion()` (`api.py:51`) naming the same concept, nothing
normalizes it to `max_tokens`, and it reaches the wire identically. A patch that only defaults
`num_predict` from `max_tokens` leaves it a silent no-op.

## Invariant violated

When a union offers both a permissive member and a strict one, the permissive member is not a
fallback, it is the winner. Smart-union resolution takes the member it can satisfy without
coercion, and `Mapping[str, Any]` satisfies every dict. So the strict member's validation is
not "also applied", it is skipped entirely, and the annotation reads as though a schema is
being enforced when none is.

Two consequences follow, and the second is the expensive one. The first is that unknown keys
pass silently, which downstream becomes an ignored request field rather than an error. The
second is that the strict member becomes a decoy for whoever writes the test. It is the
obvious thing to instantiate, it is documented, and it is right there on the annotation, so a
test built around it looks like it covers the path. It covers a path production does not take.
When the defect is about what reaches a boundary, the assertion has to sit at the boundary, on
the object actually handed across, and not on a type that merely happens to be nameable.

The third rule is about naming. A translation layer that unifies vocabularies across providers
has exactly one job at each provider's edge, and its failure is silent by construction: the
source name is valid in the source vocabulary and merely unknown in the target's, so no layer
is in a position to complain. Enumerating the two field sets and diffing them takes a minute,
and it is the only way the residue shows up at all. Nine keys is what one such diff found
here, on a file whose one known gap was a single key.

## Trigger

Any `completion()` or `acompletion()` call through `OllamaProvider` that sets `max_tokens` or
`max_completion_tokens`, streaming or not. The response is not held to the requested limit and
the model runs to its own default. Nothing is logged, because the provider considers the
parameter forwarded and the daemon considers it unknown.

## Repro

`python:3.12-slim`, any-llm installed from the checkout at `8f3bb1e`, `ollama` SDK 0.6.2. A
stub HTTP server bound to `127.0.0.1:11434` accepts `POST /api/chat` and prints the decoded
request body, so no Ollama daemon and no model download are needed. A `completion()` call
carrying `max_tokens=512` prints the `options` dict shown above. The nine-key list is a set
difference between `CompletionParams`'s fields and `Options`'s fields at the same HEAD, and
`max_completion_tokens` was confirmed on the wire with the same stub.

**Scope of what was measured.** No Ollama daemon was run, so this is a statement about the
request any-llm emits, not an observation of what the daemon does with it beyond ignoring
option keys it does not recognize. Whether `tool_choice` and `parallel_tool_calls` have
meaningful Ollama equivalents was not checked; they may be better dropped with a warning than
mapped, which is part of why the repair is the maintainers' to choose.
