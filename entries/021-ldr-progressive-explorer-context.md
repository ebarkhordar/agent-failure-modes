# The same Flask-context loss recurs in an unaudited sibling search path

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `advanced_search_system/candidate_exploration/progressive_explorer.py::ProgressiveExplorer._parallel_search`
- **Class:** concurrency & context propagation
- **Report:** [issue #5079](https://github.com/LearningCircuit/local-deep-research/issues/5079) (reported; sibling of [#4904](https://github.com/LearningCircuit/local-deep-research/issues/4904)/[#5076](https://github.com/LearningCircuit/local-deep-research/pull/5076))

## Root cause

`_parallel_search` fans out per-query searches with
`run_parallel_searches(queries, context_aware_search, max_workers=...)` and no
`context_factory`, so the default (context propagation disabled) applies.
Worker threads start with a fresh, empty Flask context; a search engine reading
`current_app` or `g` inside a worker raises `RuntimeError: Working outside of
application context`, which the surrounding `search_query` `except Exception`
swallows and returns `[]` — the whole progressive-exploration step silently
yields zero results.

This is the identical defect [#5076](https://github.com/LearningCircuit/local-deep-research/pull/5076)
fixed in `focused_iteration_strategy._execute_parallel_searches`. The catch:
`FocusedIterationStrategy` drives **both** call sites on the same request with
the same engine — `explorer.explore` reaches `_parallel_search`, and the
strategy separately calls `_execute_parallel_searches`. So #5076 fixed one path
and left its sibling with the same context-propagation bug live.

## Invariant violated

When a shared concurrency primitive is retrofitted to carry ambient request
context, every call site of that primitive must be audited — not only the one
that surfaced the failure. A fix that propagates context at one fan-out point
does not cover a sibling fan-out point reachable on the same request.

## Trigger

Any progressive-exploration research run inside a real Flask request where a
search engine reads `current_app`/`g` in a worker thread. As in the sibling
bug, isolated tests that never enter a request context pass, so it survives to
a user-visible "0 results."

## Repro

Reproduced in a clean container against `main` HEAD `790604e`, source selected
by `PYTHONPATH` with `__file__` provenance checked. Driving
`_parallel_search` inside `app.test_request_context` with a search that reads
`current_app` returns 0 results across three queries; the identical
`run_parallel_searches` call with `context_factory=thread_context` returns three
results. The swallowed `RuntimeError: Working outside of application context.`
is confirmed under the exception handler.

## Lesson

Entry [018](018-ldr-flask-context-parallel-search.md) closed with the rule
"grep for the working comparator (`context_factory=`) and diff the two call
sites." This is the same repo demonstrating why: the sibling that the grep
would have surfaced was left unfixed, so the identical failure recurs one call
site over. Grepping the primitive's every caller is not optional polish — it is
the fix's completeness condition.
