# Reversing a list changes what every fixed index into it means

- **Repo:** dgtlmoon/changedetection.io
- **Surface:** `changedetectionio/processors/restock_diff/__init__.py::Watch.extra_notification_token_values` (the `{{ restock.previous_price }}` notification token)
- **Class:** indexing, ordering & counting contracts
- **Report:** [issue #4260](https://github.com/dgtlmoon/changedetection.io/issues/4260) (deterministic repro, one-character fix offered)

## Root cause

`extra_notification_token_values` sorts the watch's history keys with `sorted(history,
key=int)`, which is ascending and therefore oldest-first. It then calls `.reverse()` to
make the list newest-first. It then reads `sorted_keys[-1]` to fill
`restock.previous_price`. On a newest-first list, `[-1]` is the oldest snapshot: the
first price the watch ever recorded.

The `.reverse()` is dead code. `[-1]` after a reverse is exactly `[0]` before it, so the
two operations cancel and what survives is a read of the wrong end of the history. At two
snapshots the result is correct by coincidence, because with only two entries the oldest
snapshot IS the previous check. The defect appears at three or more, which is why the
existing test, written against a two-price scenario, never caught it.

## Invariant violated

`restock.previous_price` must be the price at the check immediately before the current
one. `save_history_blob()` runs before `send_content_changed_notification()` in the same
linear block, so at notification time the newest key is the current check and the previous
check is the second-newest: a fixed offset from the newest end, never from the oldest.

The general rule is that an index is meaningful only relative to an ordering, so any
operation that changes the ordering silently changes the meaning of every fixed index that
reads it. `[-1]` does not mean "the latest". It means "whichever element the current order
happens to put last". Reversing a list and leaving the index alone retargets that index to
the opposite element while the code goes on reading as though it means what it meant
before.

Two things make this survive review. First, the `.reverse()` is what lends the read its
credibility: a reviewer sees the list explicitly put in newest-first order and reads the
following `[-1]` as "the newest end", when reversing is exactly what made that false. A
transform and an index that cancel each other look, line by line, like a transform and an
index that cooperate. Second, the coincidence at the boundary is what keeps the tests
quiet. When two candidate readings ("first-ever" and "previous") agree on the smallest
input that exercises the path, a test at that size cannot distinguish them, and it passes
while asserting nothing about which one the code implements. A test of an ordinal accessor
needs enough elements that every candidate answer is a different value, which for
"previous" means at least three.

## Trigger

Any restock watch with three or more snapshots whose notification body uses `{{
restock.previous_price }}`. It reports the first price the watch ever saw, forever, and
keeps reporting it as the previous price on every subsequent check.

## Repro

Reproduced in clean Docker (`test-changedetectionio:latest`, python 3.11.15) against
master HEAD `5eb2638`. Provenance: a fresh clone mounted over the image's install path
`/app`, since the image bakes the repo there and mounting elsewhere imports the baked copy
instead; `inspect.getsourcefile(Watch)` confirmed
`/app/changedetectionio/processors/restock_diff/__init__.py`.

Built with the repo's own unit-test construction pattern from
`tests/unit/test_watch_model.py` (a `Watch` with a mock datastore, plus
`save_history_blob`). Observed: n=2 gives 960.45 (correct, by coincidence); n=3 with
prices `[960.45, 1950.45, 2500.00]` gives 960.45 where 1950.45 is correct; n=4 still gives
960.45 where 2500.00 is correct. The candidate fix (`[-1]` to `[1]`) was tested in the
same image: all three cases correct, n=2 unchanged.

**Not verified:** `tests/test_restock_itemprop.py` was not executed, as it needs the
`live_server` fixture. No live browser or scrape path was exercised. The reproduction is
at the notification-token layer against the real `Watch` model, which is where the defect
lives.
