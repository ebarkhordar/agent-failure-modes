# A claim that tested one statement above the write it was guarding

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/research_library/services/download_service.py`, `library_download_queue` status transitions
- **Class:** check-then-act race, shared mutable state, claim/release protocol
- **Fix:** [PR #5081](https://github.com/LearningCircuit/local-deep-research/pull/5081) (merged), [issue #4691](https://github.com/LearningCircuit/local-deep-research/issues/4691)

## Root cause

`LibraryDownloadQueue.status` is the only thing standing between two download
streams and the same resource, and every path that touched it read the row,
decided, and then wrote, with the decision and the write in separate statements.
Two streams could both read `pending`, both conclude the row was free, and both
proceed: the second write does not fail, it simply overwrites, so the row ends up
claimed once and processed twice.

The first shape of the fix was an `if` guard above the write. It loses the same
race it was written to close: stream 2 can read `pending` at the check, stream 1
claims, and stream 2's now-stale decision still authorizes its blind write, which
un-claims a row that is actively being processed. A guard one statement above the
write is not a guard; it is a second read. What closes it is pushing the test into
the write itself, as a conditional `UPDATE ... WHERE id = ? AND status != 'processing'`,
so the check and the write cannot straddle another stream's claim.

## Invariant violated

A claim on a shared row is established by the same statement that tests it.

More generally: mutual exclusion cannot be assembled out of a read followed by a
write, no matter how little code sits between them, because the gap is not made of
code, it is made of scheduling. The only compare-and-set available here is the
database's own conditional update, and any protocol that reads first has already
lost.

## Trigger

Two download streams reaching the same queued resource concurrently. The ordinary
path, not a degenerate one: the queue exists precisely so that multiple streams can
work it.

## Repro

Focused claim, reset and fault tests covering: a pre-pass that would un-claim an
in-flight row, a second request arriving against a claimed row (now answered `409
Download already in progress` rather than downloading anyway), and an exception
after the claim releasing the row rather than stranding it.

## Note

The interesting part of this one was not the fix, it was what the fix's correctness
argument turned out to depend on. Guarding the claim site says nothing about whether
the claim is worth anything, because the claim's value is a property of every other
writer of that column, not of the claim itself. Three review rounds each landed on
the same shape: an additional writer of `LibraryDownloadQueue.status` that respected
no claim. `queue_research_downloads`, then `queue_all_undownloaded`, then
`download_source`, each resetting rows without asking whether anyone held them.

The category the writers list still misses is a caller that never participates at
all. Several routes reached the download path without ever claiming, so they were
not writers of the guard and appeared in no query about it, and they falsify a
universal from outside the protocol rather than from within it. This is also why the
tempting last step, adding `status == 'processing'` to the terminal write, was left
undone: for exactly those non-claiming callers the row is never `processing`, so the
strengthened guard would silently stop recording their terminal state, with no error
and no failing test. A guard that tightens an invariant for participants is a no-op
for non-participants, and shipping it would have been a regression wearing a fix's
clothes.

The merged scope is stated rather than implied: the guarantee covers `download-bulk`
and `download-source`. The direct PDF and text entry points, the scheduler, and
recovery of rows stranded in `processing` by a process that died holding a claim are
named as follow-ups instead of being quietly counted as covered.
