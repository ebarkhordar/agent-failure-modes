# Arithmetic on a wire field that is optional, so an omitted count becomes None + int

- **Repo:** strands-agents/harness-sdk
- **Surface:** `strands/models/ollama.py::OllamaModel.format_chunk` (the metadata chunk)
- **Class:** streaming & usage accounting
- **Fix:** [PR #3341](https://github.com/strands-agents/harness-sdk/pull/3341) (in review;
  self-discovered, no issue)

## Root cause

On the Ollama streaming path, the metadata chunk builds usage by doing arithmetic
straight on the counts: `eval_count + prompt_eval_count` for `totalTokens`, and
`int(total_duration / 1e6)` for `latencyMs`. Those three fields are optional on
the wire. Ollama tags them `json:",omitempty"` in its Go `Metrics` struct, so a
zero or absent value is dropped from the response JSON, and ollama-python declares
them `Optional[int] = None`, so a dropped field deserializes to `None`.
`format_chunk` then evaluates `int + None` and `None / float` and raises
`TypeError`, which aborts the stream. `strands` itself declares
`inputTokens: Required[int]` on its `Usage` type, so even a `None` that slipped
past would break the contract the metadata chunk exists to satisfy. The
non-streaming path is unaffected: it reads one fully populated `message.usage`.

## Invariant violated

A field the protocol marks optional can arrive absent, and absent deserializes to
`None`, so every consumer that does arithmetic on it must coerce first. The bug is
not that the count is sometimes zero, it is that "zero" and "omitted" are the same
value on the wire and only one of them is a number. The sibling Anthropic adapter
already guards the identical hazard with `usage.get(...) or 0`; a defensive idiom
applied to one provider's usage extraction has to be applied to every provider's,
because the optional-field hazard is a property of the wire format, not of the
adapter. Reading an `Optional[int]` as if it were an `int` is a latent crash that
only fires on the inputs a minimal happy-path test never builds.

## Trigger

Any Ollama streaming response whose `prompt_eval_count`, `eval_count`, or
`total_duration` is omitted (its `omitempty` zero case), which raises `TypeError`
and aborts the stream.

## Repro

Clean `python:3.12-slim` container at the branch HEAD, driving the real
`OllamaModel.format_chunk` with real `ollama.ChatResponse` objects (no mocks),
each count omitted in turn: before the fix, omitting any of the three raises
`TypeError`; after, each returns integer usage with the omitted count read as `0`.
The added `test_format_chunk_metadata_omitted_counts` is parametrized over the
three fields and builds a real `ChatResponse` rather than a `Mock`, because a
`Mock` supplies integers and hides the `Optional[int]` contract that causes the
bug; it fails on `main` and passes on the branch. `ruff`, `ruff format --check`,
and `mypy ./src` are clean; the model's test file passes (43). Not verified: a
live Ollama server was not driven, so this shows that an omitted-count response
crashes today and is handled after the fix, not how often a given server omits
these fields.
