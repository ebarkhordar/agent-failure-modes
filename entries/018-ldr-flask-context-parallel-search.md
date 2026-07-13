# Parallel search fan-out loses Flask app context in worker threads

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `focused_iteration_strategy._execute_parallel_searches`
- **Class:** concurrency & context propagation
- **Fix:** [PR #5076](https://github.com/LearningCircuit/local-deep-research/pull/5076) (merged; issue [#4904](https://github.com/LearningCircuit/local-deep-research/issues/4904))

## Root cause

`_execute_parallel_searches` ran per-question searches in a `ThreadPoolExecutor`
but called `run_parallel_searches(queries, context_aware_search)` with no context
factory. Worker threads start with a fresh, empty Flask context, so any search
code touching `current_app` or `g` raised `RuntimeError: Working outside of
application context`. The sibling `source_based_strategy` already did it right,
passing `context_factory=thread_context`.

## Invariant violated

A worker thread that runs application logic on behalf of a request must carry
that request's ambient context. Offloading work to a thread pool does not
inherit thread-local state; the context has to be reconstructed inside each
worker explicitly.

## Trigger

Any focused-iteration research run driven inside a real Flask request where a
search path reads `current_app`/`g`. Passed in isolated tests that never entered
a request context, which is why it survived to a user report.

## Repro

Integration test drives the strategy inside a Flask request handler that touches
`current_app`: raises `RuntimeError` on main, passes once `context_factory=
thread_context` is threaded into `run_parallel_searches`.

## Lesson

When one code path fans out to threads correctly and a sibling does not, the gap
is the bug. Grep for the working comparator (`context_factory=`) and diff the two
call sites rather than reasoning about the concurrency from scratch.
