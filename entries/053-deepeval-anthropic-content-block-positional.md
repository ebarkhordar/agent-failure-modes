# Reading the first content block as text, when the first block is not always text

- **Repo:** confident-ai/deepeval
- **Surface:** `deepeval/anthropic/extractors.py::extract_messages_api_output_parameters`
- **Class:** message-conversion boundaries
- **Fix:** [PR #2914](https://github.com/confident-ai/deepeval/pull/2914) (in review;
  issue [#2913](https://github.com/confident-ai/deepeval/issues/2913))

## Root cause

The extractor reads a Messages-API response as `str(message_response.content[0].text)`.
Anthropic returns `content` as a list of typed blocks, and the first is not always
a `TextBlock`. A forced tool call (`tool_choice={"type": "tool"}` or
`disable_parallel_tool_use=True`) returns `content == [ToolUseBlock]`, and
extended thinking returns a `ThinkingBlock` first. Neither has a `.text`, so the
positional access raises `AttributeError`. That exception is swallowed by the
enclosing `safe_extract_output_parameters`'s bare `except`, which returns an empty
`OutputParameters()`, so the span records `output='NA'`, no token counts, and
`tools_called=None`, and the child tool spans are skipped. The `ToolUseBlock`
handling already present later in the same function is unreachable for exactly
the responses that need it most, a tool-use-first turn, because the
`content[0].text` line above it raises before control gets there. It still runs
when a `TextBlock` happens to come first.

An earlier revision was not positionally safe either, but it crashed on a smaller
set. It branched on whether the response contained any `ToolUseBlock` at all,
falling back to `content[0].to_json()` when it did, so a tool-use-first turn
survived while thinking-first with no tools already failed. The schema-alignment
commit `c088f1c0d` ("chore: adhere to the current schema") dropped that arm and
left the unconditional `.text` access, widening the crash to tool-use-first too.

## Invariant violated

When a payload is a list of typed members whose order the producer does not
promise, you select a member by its type, not by its position. Indexing
`content[0]` and assuming a text block encodes an ordering the API never
guarantees, so a legal reordering (tool-use-first, thinking-first) turns a valid
response into an exception. The failure is worse than a crash because a bare
`except` upstream converts it into a plausible-looking empty result: the eval
does not error, it scores the literal string `'NA'` and drops the tool calls,
which is a silently wrong measurement. A branch written to handle a block type is
only reachable if nothing above it assumes that type is absent.

## Trigger

Any `messages.create(...)` whose first content block is a `ToolUseBlock` (forced
or single tool call) or a `ThinkingBlock` (extended thinking). The output, token
counts, and `tools_called` are all lost for that span.

## Repro

Clean `python:3.11-slim` container, `--network=none`, keyless, against `main`
(`58c9ef78`); the installed extractor module was sha256-identical to the reviewed
source. Responses were built by parsing real Anthropic SDK objects
(`Message.model_validate`), so the content items are the concrete block types the
extractor branches on. New tests exercise tool-use-first and thinking-first: both
lose output, tokens, and the tool on `main` (`AttributeError` swallowed to empty)
and preserve them on the branch, where the output falls back to the tool calls
when there is no text block; text-first and text-plus-tool are unchanged controls
that pass on both. Scope: the extractor return values were verified directly; the
downstream `'NA'` output and skipped tool spans follow from
`model_integrations/utils.py` reading `output or "NA"` and gating tool spans on
`tools_called`, which was read but not driven end to end, and no live API call was
made.
