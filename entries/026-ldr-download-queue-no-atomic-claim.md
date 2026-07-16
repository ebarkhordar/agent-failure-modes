# A SELECT of PENDING rows is not a claim on them

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/research_library/routes/library_routes.py::download_bulk`
- **Class:** concurrency & atomic claims
- **Fix:** [PR #5081](https://github.com/LearningCircuit/local-deep-research/pull/5081) (in review; issue [#4691](https://github.com/LearningCircuit/local-deep-research/issues/4691), owner-filed)

## Root cause

`download_bulk` queries `LibraryDownloadQueue` for `PENDING` rows, materializes
them with `.all()`, and downloads each one. Between the read and the download
there is no write that takes ownership of a row. Two overlapping calls run the
same query, receive the same rows, and both download them. `DocumentStatus.PROCESSING`
exists in the model and is never set on this path, so no state anywhere records
that a row is spoken for.

There is a dedup, which is what makes the gap easy to miss: `download_resource`
checks for an already-`COMPLETED` Document. That check only describes the world
after one stream has finished. It says nothing about the window while a download
is in flight, which is exactly the window two concurrent streams occupy.

## Invariant violated

Reading a row in state `PENDING` tells you what was true at the moment of the
read; it does not reserve anything. Between that `SELECT` and any subsequent
write, another worker's identical `SELECT` returns the same rows, and both
proceed. A claim must be a single atomic operation that tests and sets in one
step (`UPDATE ... WHERE status = 'PENDING'` conditioned on the current status,
or `SELECT ... FOR UPDATE`), with only the worker whose write actually changed
the row allowed to continue. The rest must lose the race explicitly.

The corollary is the part that generalizes: a `PROCESSING` state that exists in
the schema but is never written is not a claim mechanism, it is documentation of
one. Its presence in the model is what makes the code read as if it claims rows,
which is precisely why the gap survives review. When auditing a queue, do not
look for the state; look for the write that sets it, and check that reaching the
work requires having won that write.

The same test applies to any dedup already in the path. A guard keyed on a
terminal state (`COMPLETED`, `FAILED`) can only exclude work that has already
finished, so it is not a mutual-exclusion mechanism no matter how much it reads
like one. Ask which interval a check covers, not merely whether a check exists.

## Trigger

Two overlapping `download_bulk` calls against one queue: a double-clicked
button, a client retry, two open tabs, or a retry fired while the first request
is still running.

## Repro

Clean Docker container against current `main` (SQLAlchemy 2.0.51), with
`local_deep_research.__file__` confirmed to resolve to the repo `src`.
`test_download_bulk_claims_pending_row_before_downloading` drives the real
`/api/download-bulk` endpoint against a real on-disk SQLite database and asserts
the row is `PROCESSING` by the time `download_resource` is called. It fails on
unfixed `main`, where the row is still `PENDING` at download time, and passes on
the branch. Existing suites stay green (609 passed across the library-routes and
download-service tests).

**Not verified:** the collision was not reproduced with two real OS threads
under load. The test asserts the claim-before-download invariant that closes the
window rather than timing an actual double download, so the window is executed
and observed while the collision itself is inferred from it. No Postgres backend
was exercised (LDR uses per-user SQLite; a conditional UPDATE claim is
backend-portable).
