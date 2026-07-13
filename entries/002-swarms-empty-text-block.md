# Empty text block sent to Anthropic on image-only vision calls

- **Repo:** kyegomez/swarms
- **Surface:** `swarms/utils/litellm_wrapper.py::anthropic_vision_processing` (and the OpenAI variant)
- **Class:** message-conversion boundary
- **Fix:** [PR #1712](https://github.com/kyegomez/swarms/pull/1712)

## Root cause

The vision path always appended `{"type": "text", "text": task}` next to the
image block, even when `task` was `''` or `None` (an image-only
`run(task, img=...)`). Anthropic rejects empty text content blocks. The repo
already guarded exactly this for *system* blocks; the vision user-text block
was the ungated member of the same family.

## Invariant violated

No content block sent to a provider may be empty when the provider documents
non-empty as a requirement — and a guard applied to one block family must be
applied to every block family with the same constraint.

## Trigger

`LiteLLM(...).run('', img=...)` or `_prepare_messages(task=None, img=...)` —
the public, documented image-only path.

## Repro

Structural assertion on the built message: 7 image-only cases fail on master,
9 pass on the fix branch (offline, no live API call; the rejection premise is
Anthropic's documented requirement plus the repo's own existing guard).
