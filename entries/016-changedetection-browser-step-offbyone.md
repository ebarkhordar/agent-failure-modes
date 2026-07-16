# Already-1-indexed browser-step number re-incremented before reporting

- **Repo:** dgtlmoon/changedetection.io
- **Surface:** browser-step error reporting (`worker.py`, `notification_service.py`)
- **Class:** indexing & counting contracts
- **Fix:** [PR #4258](https://github.com/dgtlmoon/changedetection.io/pull/4258) (merged; issue [#4200](https://github.com/dgtlmoon/changedetection.io/issues/4200))

## Root cause

The browser-step iterator increments its counter *before* each step runs, so a
failure raises an exception whose step number is already 1-based (the first
step reports 1). Two separate consumers, the worker and the notification
service, each added `+1` to that value before storing and reporting it. The
reported step came out one too high, and the frontend highlighted the wrong
list item: its two JS consumers treat the stored value as 1-based (comparing
against `i+1` and selecting `nth-child(value)`).

## Invariant violated

A value is offset into a target index base exactly once, at one place. When the
producer already emits a 1-based number and every consumer expects 1-based, an
added `+1` double-counts. Establish the index base at the boundary where the
number is produced and let it flow unchanged; do not re-derive the offset at
each consumer, where the base is easy to mistake and impossible to keep
consistent.

## Trigger

Any failing browser step; the reported number and the highlighted step are both
off by one.

## Repro

The reporter confirmed it empirically (a 2-step config where step 2 fails stored
`3`). The notification path has a browser-free unit test: it fails on HEAD and
passes once the two `+1`s are removed so the stored value equals the 1-based
`step_n`.
