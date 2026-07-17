# A silent dedup skip is invisible data loss, and the fix that ends it must also fix the prose that described it

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `benchmarks/web_api/benchmark_service.py::_persist_unsaved_results` (the `if query_hash in seen_hashes: continue` branch)
- **Class:** observability & stale documentation
- **Fix:** [PR #5136](https://github.com/LearningCircuit/local-deep-research/pull/5136) (merged; follow-up to [#5080](https://github.com/LearningCircuit/local-deep-research/pull/5080), issue [#4860](https://github.com/LearningCircuit/local-deep-research/issues/4860), owner-requested)

## Root cause

Two things were left behind after [#5080](https://github.com/LearningCircuit/local-deep-research/pull/5080) corrected the dedup key (see [entry 024](024-ldr-benchmark-hash-omits-identity.md)). First, the dedup skip at the `continue` still dropped its row without emitting anything: after the hash fix a same-hash collision can only be the *same* result entry staged twice, so a skip is now a benign no-op, but there was no log line to distinguish a benign skip from a real loss the next time the key semantics move. Second, the `_persist_unsaved_results` docstring still read that "a dataset can legitimately repeat a question" as the justification for the skip, and the matching comment in `test_benchmark_result_dedup.py` carried the same pre-fix framing. Both statements were made *false* by #5080, which is exactly what put an `example_id` back into the identity.

## Invariant violated

Two of them, one per half of the change.

A branch that discards a record must say so. A `continue` that drops a staged result with no log entry is indistinguishable, in the logs, from the record never having existed, so the one signal that would catch a regression in the dedup key is absent precisely when it is needed. Making the drop observable is not decoration; it is the difference between a future data-loss bug that surfaces in a warning and one that surfaces in a silent accuracy discrepancy.

A comment describes an invariant, and when a fix changes the invariant the comment becomes a lie that outlives it. A docstring that still justifies the old behavior after the behavior changed is worse than no docstring, because the next reader trusts it and reconstructs the pre-fix mental model. A correctness fix that falsifies a comment is not finished until the comment is fixed in the same change.

## Trigger

Any correctness fix that changes what a dedup or cache key means: the branch that acted on the old meaning keeps running under the new one, silently, and every comment that explained the old meaning is now wrong. The follow-up is easy to skip because the code "works" — the tests pass and the numbers look right — while the observability gap and the false prose sit dormant until the next person touches the key.

## Repro

Docker against the merged branch: the duplicate-skip path now emits a `WARNING` on the `local_deep_research` namespace when a staged `query_hash` repeats, asserted by the two dedup tests capturing the Loguru record and checking `record["level"].name == "WARNING"`. The verification also surfaced a real test-isolation bug the maintainer caught in review — the fixture enabled the package log namespace and teardown only removed the temporary sink, leaking the enabled state into the shared logging fixture — fixed with a `finally`-guarded `logger.disable("local_deep_research")` so it holds even if a test raises.
