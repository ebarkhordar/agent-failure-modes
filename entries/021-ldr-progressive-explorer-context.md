# The same Flask-context loss recurs in an unaudited sibling search path

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `advanced_search_system/candidate_exploration/progressive_explorer.py::ProgressiveExplorer._parallel_search`
- **Class:** concurrency & context propagation
- **Report:** [issue #5079](https://github.com/LearningCircuit/local-deep-research/issues/5079) (closed as completed; sibling of [#4904](https://github.com/LearningCircuit/local-deep-research/issues/4904)/[#5076](https://github.com/LearningCircuit/local-deep-research/pull/5076))
- **Fix:** [PR #5123](https://github.com/LearningCircuit/local-deep-research/pull/5123) (merged
  2026-07-20, after its prerequisite
  [#5137](https://github.com/LearningCircuit/local-deep-research/pull/5137) landed; see
  entry [056](056-ldr-thread-context-teardown-mutates-parent-session.md))

## Root cause

`_parallel_search` fans out per-query searches with
`run_parallel_searches(queries, context_aware_search, max_workers=...)` and no
`context_factory`, so the default (context propagation disabled) applies.
Worker threads start with a fresh, empty Flask context; a search engine reading
`current_app` or `g` inside a worker raises `RuntimeError: Working outside of
application context`, which the surrounding `search_query` `except Exception`
swallows and returns `[]`; the whole progressive-exploration step silently
yields zero results.

This is the identical defect [#5076](https://github.com/LearningCircuit/local-deep-research/pull/5076)
fixed in `focused_iteration_strategy._execute_parallel_searches`. The catch:
`FocusedIterationStrategy` drives **both** call sites on the same request with
the same engine: `explorer.explore` reaches `_parallel_search`, and the
strategy separately calls `_execute_parallel_searches`. So #5076 fixed one path
and left its sibling with the same context-propagation bug live.

## Invariant violated

When a shared concurrency primitive is retrofitted to carry ambient request
context, every call site of that primitive must be audited, not only the one
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

That measurement is true and it was incomplete, which is worth stating plainly
because the harness is what hid the rest. The repro drove a bare
`Flask(__name__)`, which registers zero teardown handlers. At the HEAD it ran
against, the very `context_factory=thread_context` it green-lit also rolled back
the submitting thread's open database session, the defect recorded in entry
[056](056-ldr-thread-context-teardown-mutates-parent-session.md). Nothing was
mocked away here: the teardown that would have failed was never created. A
hand-built app is minimal by construction, so it contains exactly what makes the
test pass and is therefore selected for the part of production that agrees with
the change. It fails toward green. Building the fixture from the app factory the
production code uses, and printing what that factory registers, is what separates
"the fix works" from "the fix works on the path I built".

## What the review caught

The maintainer did not dispute the diagnosis, and did catch something this fix
did not know it was doing. Reviewing the one-kwarg change, he wrote that
"the existing `thread_context()` helper has a lifecycle issue that this new call
site exposes to the progressive path": the factory shallow-copies `g`, pushes and
pops a temporary app context on the submitting thread, and the copied
`g.db_session` is the same SQLAlchemy session the parent still holds, so
assembling a worker's context rolls the parent's session back. He asked for the
PR to be held, because "merging it first would expose the parent-session rollback
issue", and for the helper fix to land first as its own change with focused
regression coverage. That became #5137, and this PR was rebased onto it.

The generalizable part: a diff that adds no assignment can still be a writer of
shared state, because adding a *call* inherits every side effect the callee
performs. This change assigned nothing; it passed one keyword argument, and that
argument's value was a function whose invocation fires an application teardown.
A blast radius is a property of what the code reaches, not of how many lines it
spans.

## Lesson

Entry [018](018-ldr-flask-context-parallel-search.md) closed with the rule
"grep for the working comparator (`context_factory=`) and diff the two call
sites." This is the same repo demonstrating why the first half of that rule is
right and the second half is too small: the sibling was left unfixed, so the
identical failure recurred one call site over. Auditing every caller of the
primitive is not optional polish, it is the fix's completeness condition.

Two corrections to how that audit is run, both learned here rather than in 018.

The instrument is a parse, not a grep. This fix enumerated the call sites of
`run_parallel_searches` by walking the `advanced_search_system` package with
`ast`, which returns the complete set; grep answers whether a string occurs, and
a call written across two lines or reached through an alias is a member of the
set it will not show you. There are exactly three call sites, and after this
change all three pass `context_factory=thread_context`.

The completeness condition is wider than "every caller of the primitive", and the
same parse proved it. It also found `candidate_exploration/parallel_explorer.py`
driving a raw `ThreadPoolExecutor` with no context propagation, which is the same
class of bug and is not a caller of `run_parallel_searches` at all, so no audit
scoped to that primitive's callers can ever reach it. It was left out of this PR
deliberately, as a different file and a different entry point, and flagged to the
maintainer in the PR body. The lesson is that "audit the callers" scopes the
search by the fix you already have, while the defect is defined by the behavior
(fanning work out to threads that need ambient context). Those two sets are not
the same, and the second is the one that matters.
