# llama.cpp sampler params nested under an OpenAI-SDK-only `extra_body` key, then posted verbatim over raw httpx, so the server never reads them

- **Repo:** strands-agents/harness-sdk
- **Surface:** `strands/models/llamacpp.py::LlamaCppModel._format_request` (builds
  `request["extra_body"]`), consumed by `stream()` which posts the dict verbatim
  (`self.client.post("/v1/chat/completions", json=request)`)
- **Class:** configuration wiring & documented contracts
- **Fix:** [PR #3423](https://github.com/strands-agents/harness-sdk/pull/3423)
  (merged; self-discovered, no issue)

## Root cause

`LlamaCppModel` supports the llama.cpp-specific sampling parameters (`top_k`,
`repeat_penalty`, `min_p`, `typical_p`, `tfs_z`, `top_a`, `mirostat` and its
`mirostat_tau`/`mirostat_eta`, `penalty_last_n`, `n_probs`, `min_keep`,
`ignore_eos`, `logit_bias`, `cache_prompt`, `slot_id`, `samplers`). `_format_request`
collects them into `request["extra_body"]`, a nested dict. `extra_body` is an
OpenAI **Python SDK client-side idiom**: the SDK flattens `extra_body` into the
top level of the JSON body just before sending, so on an OpenAI-SDK-backed provider
those keys reach the server at the top level where it reads them.

This provider has no OpenAI SDK anywhere. `stream()` posts the request dict verbatim
over raw `httpx` (`self.client.post("/v1/chat/completions", json=request)`), so
nothing ever flattens `extra_body`. The llama.cpp server receives a literal
`{"extra_body": {...}}` object, does not recognize that key, and silently ignores
every sampler nested inside it. OpenAI-standard params (`temperature`, `top_p`,
`stop`) are placed at the top level directly, so they work, which is exactly why the
failure is invisible: a request looks like it is honoring the caller's sampling
config while every llama.cpp-specific knob is dropped and generation falls back to
server defaults with no error. `grammar`/`json_schema` are handled on a separate path
and are unaffected.

## Invariant violated

`extra_body` is not a wire field, it is a client-library instruction to a specific
SDK to hoist those keys before transmission. A transport that does not run that SDK
inherits none of that behavior, so borrowing the idiom without the library that
implements it turns a supported parameter set into a dead letter: the params are
present in the payload but under a key no consumer reads. When you post a request
dict verbatim, every key must be one the receiving server actually reads at the
position you place it. The bug is not that the values are wrong, it is that they
land at an address that does not exist, and the standard params sharing the same
request happen to be placed correctly, which masks the drop for anyone not
exercising a llama.cpp-specific sampler.

## Trigger

Any call through `LlamaCppModel` that sets a llama.cpp-specific sampler (for
example `top_k`, `repeat_penalty`, `mirostat`, `min_p`, `typical_p`, `logit_bias`,
or `samplers`). The parameter is serialized under `extra_body`, the raw httpx POST
sends it as a nested object, and the server ignores it; generation silently uses its
defaults for that knob. Present since the provider landed (#585): it was raw-httpx
and used `extra_body` together from the first commit, never OpenAI-SDK-backed.

## Repro

Docker-verified this session against HEAD in a clean `python:3.12-slim` container
(`__file__` under site-packages confirmed the installed provider was under test).
Driving `_format_request` with `{temperature: 0.7, top_k: 40, repeat_penalty: 1.3,
mirostat: 2, min_p: 0.05}`: `temperature` lands at the top level, while `top_k`,
`repeat_penalty`, `mirostat`, and `min_p` are all present only under `extra_body`
and absent from the top level. Reading `stream()` confirmed the request is posted
with `json=request` and no `openai` import anywhere, so `extra_body` is never
flattened. The fix hoists the llama.cpp params to the top level of `request` where
the server reads them; the model's test file passes (38), and `ruff check` /
`ruff format --check` are clean. Not verified: a live llama.cpp server was not
driven, so this shows the params are dead-lettered under `extra_body` today and
reach the top level after the fix, not a measured change in generated output.
