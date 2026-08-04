# A MIME bundle is read `text/plain`-first, so every rich display object returns its bare object repr and the real content is discarded

- **Repo:** datalayer/jupyter-mcp-server
- **Surface:** `jupyter_mcp_server/utils.py`, the `display_data` / `execute_result` branch of `extract_output` (around lines 289-297)
- **Class:** message-conversion boundaries
- **Fix:** the richest-readable-text selection landed upstream in
  jupyter-kernel-client as `get_mimebundle_text`
  ([PR #44](https://github.com/datalayer/jupyter-kernel-client/pull/44), merged,
  released in 0.12.0), so every consumer of the client shares one implementation;
  [jupyter-mcp-server PR #305](https://github.com/datalayer/jupyter-mcp-server/pull/305)
  (merged 2026-07-23; issue [#304](https://github.com/datalayer/jupyter-mcp-server/issues/304))
  reworks `extract_output` to delegate to it

## Root cause

`extract_output` collapses a cell output's MIME bundle to a single text string for
the tool response. For a `display_data` / `execute_result` output it checks
`text/plain` first and returns it whenever present:

```python
if "text/plain" in data:
    return strip_ansi_codes(str(data["text/plain"]))
elif "text/html" in data:
    return "[HTML Output]"
else:
    return f"[{output_type} Data: keys={list(data.keys())}]"
```

But for every `IPython.display.*` object the kernel emits a bundle whose
`text/plain` is only the bare object repr, while the real content lives in a richer
key. A `Markdown("# hi")` output arrives as
`{"text/plain": "<IPython.core.display.Markdown object>", "text/markdown": "# hi"}`,
and the `text/plain`-first check returns the repr string and drops the markdown.
`Latex`, `JSON`, and `HTML` display objects behave identically: the readable
representation is present in `text/latex` / `application/json` / `text/html` and is
never consulted. An HTML-only bundle returns the literal placeholder
`"[HTML Output]"`, and anything else falls to `"[<type> Data: keys=...]"`. In every
case the content the user asked to see is in the bundle and is thrown away.

## Invariant violated

When several representations of one value are offered together, a reducer that must
pick one must pick the richest readable one, not the first key it happens to test.
`text/plain` is the correct choice for an ordinary result whose repr *is* the
content, but it is the wrong default for a display object, where `text/plain` is
defined to be only the type's repr and the payload lives in a sibling key. A MIME
bundle is an ordered offer of fidelity, and reading it positionally ("check
`text/plain` first") discards that ordering. The reusable rule: when collapsing a
multi-representation payload to one, rank the representations by information content
and prefer the richest present, falling back to the bare repr only when nothing
richer exists, never let key-lookup order stand in for a fidelity ranking.

## Trigger

Any cell whose output is an `IPython.display` object with both a repr `text/plain`
and a richer text key: `Markdown`, `Latex`, `JSON`, `HTML`, and similar. Because
`extract_output` (via `safe_extract_outputs`) feeds `execute_cell`, `execute_code`,
and `read_cell` in both server modes and the local backend, the loss shows up on
every read path, not just one tool.

## Repro

Reproduced this session in a clean `python:3.11-slim` container with the package
installed from source, `jupyter_mcp_server.utils.__file__` printed to confirm the
in-tree module. On HEAD `9d03c5e` (the parent of the fix):

```
{"text/plain": "<IPython.core.display.Markdown object>", "text/markdown": "# hi"}
  -> "<IPython.core.display.Markdown object>"
{"text/plain": "<IPython.core.display.Latex object>", "text/latex": "$E=mc^2$"}
  -> "<IPython.core.display.Latex object>"
```

On the fix branch the same two bundles return `"# hi"` and `"$E=mc^2$"`. The fix
ranks `text/markdown`, `text/latex`, `application/json`, then `text/plain`, and falls
back to the bare repr only when no richer text is present. The added pure-function
fidelity test asserts the
rich content is returned for each display type and that an ordinary
`text/plain`-only result is unchanged; it fails on the parent and passes on the
branch, and the repo's build/test CI is green on the branch.

## Corrections

This entry claimed the fix prefers "the JSON/HTML equivalents". Half of that is wrong
and has been since the entry was written. `RICH_TEXT_MIMETYPES` is
`("text/markdown", "text/latex", "application/json", "text/plain")`, and the helper's
own docstring says `text/html` is deliberately not consulted: markup is not readable
text, and a `pandas` DataFrame emits both an ASCII `text/plain` table and an HTML one,
where the plain table is what a text consumer should see. An HTML-only bundle still
returns `"[HTML Output]"` after the fix, pinned by a test named
`test_html_only_placeholder_unchanged`. So the root cause above is right that HTML
content is discarded, and the fix left that case alone on purpose.

The architecture was the reviewer's, not ours. The first version of this PR put the
selection in `jupyter_mcp_server/utils.py`, and `echarles` asked whether it belonged in
jupyter-kernel-client instead. We argued against it: choosing what representation a
model sees looked like a consumer-side decision. He held the line, on the grounds that
any other consumer of the client benefits from the util, and released
jupyter-kernel-client 0.12.0 with the helper about half an hour after that PR merged.
Two days later the sharing stopped anyway: `937eb514b` (#319, 2026-07-25) migrated the
repo off jupyter-kernel-client onto `code-sandboxes`, and `get_mimebundle_text`,
`_coerce_bundle_text` and `RICH_TEXT_MIMETYPES` are now local definitions in
`jupyter_mcp_server/utils.py` again, with the dependency gone from `pyproject.toml`. A
helper is only shared for as long as the dependency carrying it survives. The
behaviour is identical either way, and the upstream helper is still released and still
used by whoever else depends on that client.
