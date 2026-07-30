# A converter chain parses markdown in the dialect of its innermost library, so a list the caller wrote is absorbed into the paragraph above it

- **Repo:** sooperset/mcp-atlassian
- **Surface:** `src/mcp_atlassian/preprocessing/confluence.py:85`
  (`ConfluencePreprocessor.markdown_to_confluence_storage`), which builds its HTML with
  md2conf's `markdown_to_html`, which is Python-Markdown. Reached by
  `confluence/comments.py:143` and by page create/update at `confluence/pages.py:711`,
  `822`, `938`.
- **Class:** template & token rendering
- **Report:** triaged publicly on
  [issue #1493](https://github.com/sooperset/mcp-atlassian/issues/1493#issuecomment-4986406321)
  (open, filed by another user whose Confluence pages are the observation). No PR from us:
  the remedy is a choice between a preprocessing pass in this repo and a change upstream in
  md2conf or Python-Markdown, which is the maintainer's call, and the offer to send the
  preprocessing patch is open in the thread.

## Root cause

An MCP server sits between a model that writes markdown and Confluence, which stores
markup. The conversion runs entirely offline, before any API call:

```python
from mcp_atlassian.preprocessing.confluence import ConfluencePreprocessor
pre = ConfluencePreprocessor(base_url="https://confluence.example.com")
pre.markdown_to_confluence_storage("**Header:**\n- item one\n- item two\n\nSome other paragraph.\n")
```

```
<p><strong>Header:</strong> - item one - item two</p><p>Some other paragraph.</p>
```

The list is gone before md2conf's storage converter ever runs. Python-Markdown does not
implement CommonMark's rule that a list may interrupt a paragraph, so the two bullet lines
are lazy continuation lines of the paragraph they follow:

```
markdown_to_html("**Header:**\n- item one\n- item two\n")
-> '<p><strong>Header:</strong>\n- item one\n- item two</p>'
```

Insert a blank line before the first bullet and the same call returns
`<ul><li><p>item one</p></li><li><p>item two</p></li></ul>`.

The reported trigger is not the real one. Varying each element of the reporter's minimal
case separately:

- The bold is not involved. `Header:\n- item one` flattens identically.
- A heading is fine. `# Header\n- item one` renders as a list, because an ATX heading is
  not a paragraph and there is nothing for the bullets to continue.
- Ordered lists flatten too. `**Header:**\n1. item one` gives
  `<p><strong>Header:</strong> 1. item one</p>`, so a fix scoped to `-`, `*` and `+` would
  leave that case broken.
- It is not Server/DC specific. `add_comment` converts above the Cloud/DC branch, and the
  Cloud v2 path posts `representation="storage"` as well.

## Invariant violated

**"Markdown" is not one language, and in a converter chain the dialect that decides the
parse is whichever library sits innermost.** Every layer above it accepts a string it calls
markdown and hands it down unexamined, so the effective grammar is a transitive dependency's
choice. Nobody at the top of the chain can see which grammar they are writing for: the model
emits CommonMark because that is what almost every surface it was trained on renders, and
the tool's own docstring says markdown without naming a dialect. The failure is silent
because both dialects accept the input; they just disagree about what it means, and one of
the two answers loses a structure the other found.

**A reported trigger is a conjunction until you take it apart, and fixing the conjunction
fixes the report rather than the defect.** The reporter's case carried four features (bold,
a colon, a bullet list, Confluence Server) and exactly one of them mattered: a paragraph line
directly above a list marker. A patch aimed at bold headers would have closed the issue and
left ordered lists, plain-text headers and the page-body path broken, which is the shape of
fix that gets reopened as a new bug three weeks later. Vary each element of a repro
independently before choosing where to cut.

The corollary for anything that renders model output: the character sequences a model emits
most often are exactly the ones a strict dialect difference will catch. "Label line, then
bullets" is not an exotic input, it is close to the default shape of an LLM answer, so a
grammar gap here is hit constantly rather than rarely.

## Trigger

Any paragraph line immediately followed by a list marker, with no blank line between them,
sent through Confluence comment creation or page create/update. No Confluence instance is
needed to observe it.

## Repro

Docker `python:3.12-slim`, editable install of the repo at `0587fdc`, md2conf 0.6.1,
Python-Markdown 3.10.2, module resolved by `__file__` to
`/src/src/mcp_atlassian/preprocessing/confluence.py`. The calls and outputs above were
executed there.

One dead end worth recording, because it is the obvious lever and it is not the cause:
md2conf enables the `sane_lists` extension, but Python-Markdown 3.10.2 flattens this input
with the extension both on and off.

```
markdown.Markdown(extensions=[]).convert(body)             -> <p><strong>Header:</strong>\n- item one\n- item two</p>
markdown.Markdown(extensions=["sane_lists"]).convert(body) -> <p><strong>Header:</strong>\n- item one\n- item two</p>
```

**Not verified, and this bounds the entry:** nothing was posted to a live Confluence
instance, Cloud or Server. The claim that the Cloud path is affected comes from reading the
call order in `confluence/comments.py` and `confluence/v2_adapter.py`, not from execution.

Verified 2026-07-15 at `0587fdc`. Reported, not fixed upstream at the time of writing.
