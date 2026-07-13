# A document is marked indexed after its embeddings failed to store

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `research_library.services.library_rag_service.index_document`
- **Class:** error handling & success reporting
- **Fix:** [PR #5063](https://github.com/LearningCircuit/local-deep-research/pull/5063) (issue [#4855](https://github.com/LearningCircuit/local-deep-research/issues/4855))

## Root cause

`LocalEmbeddingManager._store_chunks_to_db` catches any exception and returns an
empty list. On success it appends exactly one id per chunk (a reused id for a
pre-existing chunk, a fresh id for a new one), so `len(embedding_ids) ==
len(chunks)`; an empty return for a non-empty chunk set means nothing was
persisted. `index_document` ignored that signal: with `embedding_ids == []` it
ran the FAISS merge (which adds zero vectors), then still set
`DocumentCollection.indexed=True` and returned `status="success"`. The document
was flagged indexed with no vectors behind it, so it was silently unsearchable.

## Invariant violated

A document flagged `indexed=True` must have had its embeddings persisted (it must
be searchable). A swallowed store failure has to leave the row `indexed=False`,
because the background reconciler retries only `indexed.is_(False)` rows: marking
a failed store as indexed removes it from the one path that would have recovered
it.

## Trigger

Any chunk-store failure inside `_store_chunks_to_db` (a DB error, or a missing
user context) on a document that did produce chunks. The FAISS merge over zero
new vectors logs "all chunks already exist, skipping" and does not raise, so the
success path is reached with an empty embedding set and no error surfaces.

## Repro

Unit test drives `index_document` with the chunk splitter yielding a real chunk
set while `_store_chunks_to_db` returns `[]` (its exception path). On main the
call returns `status="success"` and sets `indexed=True`; after the fix it returns
`status="error"` and leaves the document un-indexed.

## Lesson

A function that swallows exceptions and returns an empty result turns a failure
into a silent success at the caller unless the caller treats "empty" as the
failure signal it is. When a downstream flag (`indexed=True`) gates a recovery
path (the reconciler only retries un-indexed rows), the flag must never be set on
the error path: doing so converts a retryable transient failure into a permanent
one.
