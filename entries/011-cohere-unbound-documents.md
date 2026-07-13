# UnboundLocalError on the Cohere RAG path: variable read where never bound

- **Repo:** run-llama/llama_index
- **Surface:** `get_cohere_chat_request`
- **Class:** initialization & control flow
- **Fix:** [PR #22333](https://github.com/run-llama/llama_index/pull/22333)

## Root cause

`documents` was only assigned inside a conditional branch but read
unconditionally afterward; on the path where the branch didn't run, the
function crashed with `UnboundLocalError` — taking down the RAG request
entirely.

## Invariant violated

Every variable read on a code path must be bound on that path. Conditional
assignment + unconditional read is the textbook shape; linters catch it when
the read is syntactically visible, but builder-function refactors reintroduce
it constantly.

## Trigger

A Cohere chat request built without the documents-populating branch running —
an ordinary non-RAG call through the RAG-capable builder.

## Repro

Offline, deterministic: construct the request on the crashing path; test
fails on main with `UnboundLocalError`, passes with the fix.
