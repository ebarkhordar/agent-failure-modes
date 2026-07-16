# Dedup key omits the identity field, so distinct examples collapse to one row

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `benchmarks/web_api/benchmark_service.py::generate_query_hash`
- **Class:** cache-key & hashing collisions
- **Fix:** [PR #5080](https://github.com/LearningCircuit/local-deep-research/pull/5080) (in review; issue [#4860](https://github.com/LearningCircuit/local-deep-research/issues/4860), owner-filed)

## Root cause

`generate_query_hash` keys a benchmark result on
`f"{question.strip()}|{dataset_type.lower()}"`. `example_id` is not in the
input. Two distinct examples in one run that happen to share question text
therefore hash identically, and the second is dropped by the app-level dedup at
`benchmark_service.py:940` (`if query_hash in seen_hashes: continue`). The run
executes both examples and persists one. Because `overall_accuracy` is computed
in memory over what ran, the reported accuracy disagrees with the row count that
backs it, and nothing reports the loss.

## Invariant violated

A dedup key must be derived from the identity of the record, not from its
payload. `example_id` is the identity; question text is payload, and payload is
free to repeat. When the key omits the identity field, dedup quietly stops
meaning "do not store this twice" and starts meaning "do not store this at all",
which is data loss wearing the costume of an optimization.

## The layer that never fires

`database/models/benchmark.py:187-188` carries
`UniqueConstraint('benchmark_run_id', 'query_hash', name='uix_run_query')`, and
reading the code it looks like the enforcement point. It is not. The in-memory
`seen_hashes` check drops the row first, so the constraint never sees a second
insert and never raises.

The generalization is worth more than the bug: when two dedup layers guard the
same key, the outer one silences the inner. The inner layer is the one that
would have surfaced the problem (a constraint violation is loud), and the outer
one is the one that silently discards. So the layer that looks like the
safeguard is the layer that is doing nothing, and a fix aimed at it changes no
behavior at all. Find the outermost layer holding the key before deciding where
the fix goes. Here that makes the correct fix hash-only, which also happens to
need no Alembic migration; changing the constraint alone would have been both a
migration and a no-op.

## Trigger

Any benchmark run whose dataset contains two examples with identical question
text. Datasets built from templated or paraphrased questions, or assembled
across splits, hit this without anything looking unusual.

## Repro

Docker (image `ldr-repro4860`) against HEAD `3bc3373`. Two examples `ex_A` and
`ex_B` with distinct ids and the same question text hash to the same
`query_hash` (`1371ab08...`); `_persist_unsaved_results` stages only the first
and persists `ex_A`, dropping `ex_B` at the `continue` on line 940, before the
`uix_run_query` constraint is ever reached. Control with distinct question text
persists 2 rows. On the fix branch the new test passes and 1446 benchmark tests
pass; on `main` the same test fails with `assert 1 == 2`.
