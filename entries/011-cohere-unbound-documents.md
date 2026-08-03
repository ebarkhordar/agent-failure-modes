# UnboundLocalError on the Cohere RAG path: variable read where never bound

- **Repo:** run-llama/llama_index
- **Surface:** `get_cohere_chat_request`
- **Class:** initialization & control flow
- **Fix:** [PR #22333](https://github.com/run-llama/llama_index/pull/22333), withdrawn
  2026-08-03. We closed it ourselves to keep the open-PR list to a size one person can
  shepherd; no maintainer reviewed or refused it. The defect stands on main and the branch
  still applies cleanly, both checked 2026-08-03.

## Root cause

A guard rejects documents supplied through two channels at once:

```python
additional_kwargs = messages[-1].additional_kwargs

# cohere SDK will fail loudly if both connectors and documents are provided
if additional_kwargs.get("documents", []) and documents and len(documents) > 0:
    raise ValueError(...)

messages, documents = remove_documents_from_messages(messages)
```

The guard reads `documents`, and the only line that binds it runs five lines
later. There is no `documents` parameter, so the name is unbound at the guard
and the call raises `UnboundLocalError`. Parsing the function with `ast` gives
the whole story: one `Store` of `documents`, at line 238, and a `Load` at line
233 that precedes it.

## Invariant violated

**A guard must run after the code that produces the values it guards.** This one
was written to compare two sources of documents against each other, and it was
placed above the extraction that yields the second source, so it can never have
worked.

**`and` short-circuits, so an unbound operand is a crash only on the path where
the operands before it are truthy.** `documents` is evaluated only when
`additional_kwargs.get("documents", [])` is non-empty. That is the RAG path, the
exact case the guard exists to catch, so the defect is invisible on every
ordinary call and fires precisely when the check is supposed to do its job. A
use-before-assignment sitting behind a short circuit is not reachable by the
common path that would have caught it in review or in a smoke test.

## Trigger

A chat request whose last message carries a non-empty
`additional_kwargs["documents"]`, which is the ordinary way to attach documents
to a Cohere RAG call.

## Repro

Offline and deterministic, with the Cohere client mocked so no key or network is
involved: build a request from one `ChatMessage` carrying
`additional_kwargs={"documents": [...]}`. Main raises `UnboundLocalError`; with
the binding moved above the guard it returns the request, and a second test
pins that documents supplied through both channels still raise the guard's
intended `ValueError` rather than the crash.

## Correction

Until 2026-08-03 this entry described the root cause as a variable "only
assigned inside a conditional branch but read unconditionally afterward", and
its trigger as "an ordinary non-RAG call". Both were wrong, and the trigger was
backwards: the binding is unconditional and merely late, and short-circuit
evaluation confines the crash to the RAG path. The entry had never matched its
own PR, whose fix moves one line and whose test docstring states the real
mechanism.
