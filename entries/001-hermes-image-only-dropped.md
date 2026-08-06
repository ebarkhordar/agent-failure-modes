# Image-only user messages silently dropped at the Anthropic conversion boundary

- **Repo:** NousResearch/hermes-agent
- **Surface:** `agent/anthropic_adapter.py::_convert_user_message`
- **Class:** message-conversion boundaries
- **Fix:** fixed upstream independently on 2026-08-03 by
  [`a3257cbf4`](https://github.com/NousResearch/hermes-agent/commit/a3257cbf46b648c78836fdc455961298dcd8a64a),
  which filters blank text blocks per block rather than collapsing the whole list, and
  [`633bd354f`](https://github.com/NousResearch/hermes-agent/commit/633bd354f99a9f9bef13e093841e0b5ec9a2fa9a),
  which folds that into a shared helper. Our
  [PR #57907](https://github.com/NousResearch/hermes-agent/pull/57907) was withdrawn the same
  day and was not rejected: a maintainer had confirmed the premise on it, and its two
  regression tests pass against current main with nothing from the branch applied.

## Root cause

The converter replaced the entire content list with an `"(empty message)"`
placeholder whenever no non-blank TEXT block existed. For an image-only user
turn the text-block list is empty, and `all([])` is vacuously `True`, so the
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
converted request contains the placeholder text and no image. The PR's
regression test failed on main while the defect was live. It passes on main as
of 2026-08-03, which is how the independent fix above was confirmed to close
this rather than merely to overlap it.

## Note

`all(...)` over a possibly-empty list inside a "does anything real exist"
guard is the recurring shape here. Check the empty case explicitly.

Found and fixed by my collaborator [@AmirF194](https://github.com/AmirF194).
