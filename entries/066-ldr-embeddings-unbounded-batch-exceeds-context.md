# An embeddings batch is sized by a fixed input count, never by the endpoint's token budget, so a large document exceeds context

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/embeddings/providers/implementations/openai.py` (`OpenAIEmbeddingsProvider.create_embeddings`)
- **Class:** configuration wiring & documented contracts
- **Fix:** [PR #5144](https://github.com/LearningCircuit/local-deep-research/pull/5144) (merged;
  issue [#4966](https://github.com/LearningCircuit/local-deep-research/issues/4966))

## Root cause

`create_embeddings` builds the LangChain `OpenAIEmbeddings` object without ever
setting `chunk_size`, so it keeps LangChain's default of 1000. `embed_documents`
groups up to `chunk_size` inputs into a single `/v1/embeddings` request and puts no
bound on the batch's cumulative token count: a document that splits into N chunks
(N <= 1000) is sent as one request of N inputs regardless of how many tokens those
inputs total. Against an OpenAI-compatible local server (LM Studio, vLLM,
llama.cpp) with a small context window, the combined chunk tokens cross `n_ctx` and
the endpoint returns `400 exceed_context_size` (the `n_prompt_tokens > n_ctx` error
in the issue).

`check_embedding_ctx_length` does not help: whether it is on or off, the whole
batch still goes in one request. The size limit that actually binds a local model
is a token budget per request, and nothing in this path expresses one, so the only
ceiling is a fixed count of inputs that has no relationship to the model's context.

## Invariant violated

A batch sent to an embeddings endpoint must be bounded by that endpoint's context
window, which is a token limit, not a fixed count of inputs. Sizing a request by
"how many items" instead of "how many tokens" leaves the real ceiling unwired, so
the request stays legal for small inputs and fails hard the moment the inputs are
long enough, with the failure depending on document content rather than on any
setting the operator can see. A cloud provider with a large context hides the
defect; a local endpoint with a small `n_ctx`, which is the whole point of this
provider, exposes it as a hard 400.

## Trigger

Indexing a document long enough to split into many or large chunks, against a local
OpenAI-compatible embeddings endpoint with a modest context window. The single
batched request exceeds `n_ctx` and returns `400 exceed_context_size`, aborting the
indexing.

## Repro

Clean container with `langchain-openai` 1.3.5 at HEAD `52896f9`: driving the
provider so that `embed_documents` receives more chunk text than a small `n_ctx`
can hold sends it all in one request, which the endpoint rejects with the
context-size 400; no per-request token budget is ever applied because `chunk_size`
is left at the default and no token-aware batcher exists. The fix sets a
configurable `chunk_size` (and a token-aware bound) so the request is split under
the endpoint's limit. Not verified against a specific live local server's exact
`n_ctx`; the repro shows the request is assembled with no token bound and the
default 1000-input batching path is taken.
