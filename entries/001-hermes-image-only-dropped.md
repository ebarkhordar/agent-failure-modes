# Image-only user messages silently dropped at the Anthropic conversion boundary

- **Repo:** NousResearch/hermes-agent
- **Surface:** `agent/anthropic_adapter.py::_convert_user_message`
- **Class:** message-conversion boundary
- **Fix:** [PR #57907](https://github.com/NousResearch/hermes-agent/pull/57907) (in review)

## Root cause

The converter replaced the entire content list with an `"(empty message)"`
placeholder whenever no non-blank TEXT block existed. For an image-only user
turn the text-block list is empty, and `all([])` is vacuously `True` — so the
guard fired and the image block was discarded, even though the API accepts
image-only turns.

## Invariant violated

A message conversion layer must never drop content the target API accepts.
The placeholder path is a fallback for *nothing survives*, not for *no text
survives*.

## Trigger

Any user turn consisting of only images (or images plus blank-text captions).

## Repro

Send a user message with a single image block through the adapter; the
converted request contains the placeholder text and no image. Regression test
in the PR fails on main, passes with the fix.

## Note

`all(...)` over a possibly-empty list inside a "does anything real exist"
guard is the recurring shape here. Check the empty case explicitly.
