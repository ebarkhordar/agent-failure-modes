# Usage-only final stream chunk re-emits the previous content delta

- **Repo:** run-llama/llama_index
- **Surface:** SambaNova `stream_chat` chunk handling
- **Class:** streaming & usage accounting
- **Fix:** [PR #22337](https://github.com/run-llama/llama_index/pull/22337) (in review)

## Root cause

The final chunk of a SambaNova stream carries only usage data and no content
delta. The handler reused the previous iteration's `delta` variable instead
of emitting an empty one, so the last content fragment was delivered twice.

## Invariant violated

Concatenating streamed deltas must equal the complete response, exactly once.
Any loop variable that survives across stream iterations is a stale-state
hazard on chunks that don't reassign it.

## Trigger

Every streamed SambaNova response that ends with a usage chunk — duplicated
tail text on essentially all streams.

## Repro

Mocked event stream in the PR's test: fails on main (tail duplicated), passes
with the fix.
