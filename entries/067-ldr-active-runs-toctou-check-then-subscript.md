# A membership check and a separate subscript on a shared dict race a concurrent delete, so the poll raises KeyError and returns a spurious HTTP 500

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/benchmarks/web_api/benchmark_service.py` (`BenchmarkService.sync_pending_results`)
- **Class:** concurrency & atomic claims
- **Fix:** [PR #5146](https://github.com/LearningCircuit/local-deep-research/pull/5146) (merged;
  issue [#4859](https://github.com/LearningCircuit/local-deep-research/issues/4859))

## Root cause

`sync_pending_results` reads the shared `self.active_runs` dict with a
non-atomic membership check followed by a separate subscript:

```python
if benchmark_run_id not in self.active_runs:   # check
    return 0
run_data = self.active_runs[benchmark_run_id]   # then read
```

The benchmark worker thread deletes the entry at the end of
`_sync_results_to_database` (`del self.active_runs[benchmark_run_id]`), and that
delete runs outside `_results_sync_lock`. When it lands between the check and the
subscript, the subscript raises `KeyError`. Because the read sits before the
method's own `try`, the error escapes `sync_pending_results`; the caller
`get_benchmark_results` wraps it in a broad `try/except Exception`, so instead of
the clean `0` the next poll would have returned, the client sees a spurious `500`
on `GET /api/results/<id>`.

## Invariant violated

A membership check and the subscript that trusts it are two operations, not one:
between them any other thread may invalidate the check. A read of shared mutable
state that must not raise has to obtain the presence test and the value from a
single atomic lookup (`dict.get`), not from a `not in` guard followed by `[]`. A
"finished during the poll" outcome is normal and must degrade to "nothing left to
sync" (return `0`), never to an uncaught exception surfaced as a server error.

## Trigger

Polling `GET /api/results/<id>` for a benchmark run at the moment the worker
thread finishes it and executes its `del self.active_runs[id]`. The delete
falling inside the check-then-subscript window turns an ordinary end-of-run poll
into a `KeyError` and a 500.

## Repro

Clean container (`python:3.12-slim`, `pip install -e .`) against HEAD `fc7ea75b`,
with `benchmark_service.__file__` printed on both sides to confirm the mounted
source ran. A regression test replaces `active_runs` with a mapping whose
`__contains__` reports the key present while `__getitem__` raises, modelling the
delete firing inside the window: it fails on unmodified HEAD with `KeyError: 123`
and passes with the fix, which returns `0`. The fix reads once with
`run_data = self.active_runs.get(benchmark_run_id)` and returns `0` when it is
`None`, so the presence test and the value come from the same lookup. The second
check-then-subscript site in `_sync_results_to_database` runs on the worker
thread that is itself the sole live-run deleter, so it cannot race the delete and
was left unchanged. Not driven end to end: the propagation through
`get_benchmark_results` to an HTTP 500 is read from the route's `try/except`, not
exercised against a live server; the fix is verified at the
`sync_pending_results` level.
