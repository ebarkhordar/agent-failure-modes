# A cleanup pass that runs before the parser has no idea what it is deleting

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight-api-slim/hindsight_api/engine/reflect/structured_doc.py::parse_markdown` (the mental-model document parsed and re-rendered on every delta refresh)
- **Class:** text decomposition contracts
- **Report:** [issue #2752](https://github.com/vectorize-io/hindsight/issues/2752)
- **Fix:** [PR #2755](https://github.com/vectorize-io/hindsight/pull/2755) (merged)

## Root cause

The mental model is stored as markdown and round-trips through
`parse_markdown` / `render_document` every time it is refreshed. LLMs like to emit
horizontal rules between sections, and the renderer never emits them, so the parser
deleted them first:

```python
raw_lines = (markdown or "").splitlines()
lines = _strip_separators(raw_lines)     # every `---`/`***`/`___` line -> ""
```

`_strip_separators` blanked every line matching `\s*([-*_])\1{2,}\s*` across the whole
list, and only afterwards did `_split_blocks` walk the lines tracking `in_fence` to find
fenced code. So the one pass that knew what was code ran second, and by the time it could
have objected, the content was already gone. A `---` inside a fenced block was replaced
with an empty line, and `_split_blocks` faithfully preserved that empty line as part of the
code block, because inside a fence it preserves blanks. The document parses cleanly,
renders cleanly, and is missing a line.

`---` inside a fence is not decoration. It is the YAML document separator, the front-matter
delimiter, and a legal line in most of the languages an agent writes into a mental model.
Losing it does not corrupt the markdown; it corrupts the payload, turning two YAML
documents into one invalid one while leaving a code block that still looks plausible.

The existing test covered exactly the case the author had in mind: a rule between two
sections is dropped and does not become a paragraph. It passed before and after the fix.
The bug lived in the case nobody wrote a test for, which was not an exotic edge but the
same character in the other context the file already knew about, since `_split_blocks`
had carried fence tracking all along.

## Invariant violated

`render_document(parse_markdown(md)) == md` for any document whose constructs the parser
claims to support, and text inside a fence is never interpreted.

The general rule is that a normalisation pass and a structure-aware pass cannot be ordered
arbitrarily, and the normalisation must not be first. "Strip the junk, then parse" is a
tempting shape because the strip is a one-line comprehension over a flat list and the parse
is a state machine, so the simple thing wants to go first. But stripping *is* an
interpretation: to say a line is a separator is to claim it is not content, and that claim
requires the very context the parser has and the comprehension does not. Any pass that
deletes by regex over a flat line list is asserting that the pattern means the same thing
everywhere in the document, and in a language with fences, quotes, or escapes, it never
does.

The second half of the rule is about where the fix goes. The obvious repair is to give
`_strip_separators` its own `_FENCE_RX` toggle, which works and leaves two fence state
machines in one file, drifting apart at whatever rate the language's fence rules evolve.
The better repair deletes the pass and moves its decision into the state machine that
already tracks the context, so there is exactly one place that knows what a fence is. Two
correct implementations of the same context are worse than one, because the bug they
eventually produce is a disagreement, and a disagreement has no obviously wrong side.

## Trigger

Any mental model containing a fenced code block with a `---`, `***`, or `___` line: a YAML
manifest, front matter, an RST heading underline, a text-art divider. No configuration and
no scale needed, and the delta refresh means the loss is written back and compounds.

The end-to-end chain through `memory_engine.py` is read from the code, not observed. What
was executed is the pure `parse_markdown` / `render_document` pair at HEAD.

## Repro

`uv` on `python:3.11-bookworm`, HEAD `c27fafb`, container source identity-checked against
the clone by sha256. Two new tests against the unfixed source: 2 failed / 4 passed, by
assertion rather than import error. Against the fixed source: 6 passed. The full
`tests/test_structured_doc.py` is 41 passed. A 26-case old-vs-new matrix isolates the blast
radius: 17 non-fence cases byte-identical, 9 fence cases differ, 0 unexpected.

## Fix

The merged fix removes `_strip_separators` and the `parse_markdown` call to it, and teaches
`_split_blocks`, which already tracked `in_fence`, to treat a rule line as blank only
when it is outside a fence:

```python
if line.strip() == "" or _SEPARATOR_RX.fullmatch(line):
```

Behaviour between sections is bit-identical, which the pre-existing test proves; that test
passing unchanged is what shows the fix is narrow rather than a rewrite of the parser.
