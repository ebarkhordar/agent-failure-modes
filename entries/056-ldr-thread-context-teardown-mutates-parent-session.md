# Building a worker's context pops an app context on the parent thread, firing a teardown that rolls back the parent's session

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `utilities/thread_context.py::thread_context`, against
  `web/app_factory.py::cleanup_db_session`
- **Class:** concurrency & context propagation
- **Fix:** [PR #5137](https://github.com/LearningCircuit/local-deep-research/pull/5137)
  (merged; prerequisite for
  [#5123](https://github.com/LearningCircuit/local-deep-research/pull/5123))

## Root cause

`thread_context()` builds the app context that carries request state into search
worker threads. To populate the new context's `g`, it shallow-copies the current
`g`, then pushes and pops a temporary app context on the submitting thread and
sets each key inside the `with` block:

```python
with context:
    for key, value in global_data.items():
        setattr(g, key, value)
```

Popping an app context fires `teardown_appcontext`. The app registers exactly one
such handler, `cleanup_db_session`, which does `session = g.pop("db_session", None)`
and, if present, `session.rollback(); session.close()`. The copied `g.db_session`
is the *same* SQLAlchemy session object the submitting thread still holds, so this
teardown rolls back and closes the parent thread's live session, discarding its
uncommitted work. As a second consequence the worker never receives `db_session`
either, because teardown pops it from the copied `g` before the context is
returned. A helper whose only job was to *copy* request state into a child mutated
the parent's state as a side effect of how it did the copy.

## Invariant violated

Creating a child execution context must not mutate the parent's context. The trap
is that entering-and-leaving a framework context is not a neutral read: leaving it
runs registered teardown, and teardown is defined to *consume* the resources it
finds, so borrowing the parent's context machinery to assemble a child's state
runs the parent's cleanup against objects the child merely aliased. A shared
mutable resource carried by reference (here one DB session) has exactly one owner;
any code that triggers that resource's lifecycle hooks is a writer of it, even
when the code looks like it only reads. Populate the child's `g` directly, without
pushing the parent's context, and skip the teardown-owned keys so the child
acquires its own session rather than aliasing the parent's.

This entry is the concrete case behind a general rule: the writers of a shared
mutable field include everything a change newly *calls*, not just the lines it
edits, because invoking a helper inherits every lifecycle hook that helper fires.
A one-line "copy the context" can carry the entire blast radius of an app
teardown.

## Trigger

Any caller that builds a `thread_context()` while its own thread holds an open,
uncommitted `db_session`: the parent's pending work is rolled back the moment the
context is assembled, before any worker runs.

## Repro

Clean container against the branch, driving the real `create_app()` so the real
`cleanup_db_session` teardown is registered (not a bare `Flask(__name__)`, which
registers zero teardown handlers and would define the defect away). The claim
"creating `thread_context()` does not alter the submitting thread's active
`g.db_session`" was discharged by enumeration, not by a single green path: the
teardown handlers on the real app were counted
(`len(app.teardown_appcontext_funcs) == 1`, `cleanup_db_session`), and the writers
of `g.db_session` across the package were enumerated by parsing the AST, with
every other writer confirmed to live in a request or auth path the factory never
invokes on this route. The fix removes the only reachable trigger. The
discriminating tests mock no writer (real app, real teardown, real session), so
nothing is defined away, and each of the eight guarantees from the review thread
is one test: creation does not commit, roll back, or close the parent session; the
parent's pending work survives worker exit; safe values reach the worker; and the
teardown-owned `db_session` is not smuggled across threads.
