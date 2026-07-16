# A space inside a template block splits the URL from its tags

- **Repo:** dgtlmoon/changedetection.io
- **Surface:** `changedetectionio/blueprint/imports/importer.py::import_url_list.run`
- **Class:** template & token rendering
- **Fix:** [PR #4259](https://github.com/dgtlmoon/changedetection.io/pull/4259) (in review; issue [#1516](https://github.com/dgtlmoon/changedetection.io/issues/1516))

## Root cause

The URL-import format is one watch per line, `<url> <tags>`, and the parser
implements that with `if " " in url: url, tags = url.split(" ", 1)`. The
project also supports jinja2 inside watch URLs, and a jinja2 block contains
spaces of its own. Importing

```
https://changedetection.io/test.txt?date={% now 'Europe/Berlin', '%d' %}
```

splits at the first space *inside the block*, yielding
`url='https://changedetection.io/test.txt?date={%'` and
`tags="now 'Europe/Berlin', '%d' %}"`. Both fields are corrupted.

The truncated URL is then rejected downstream rather than stored: `add_watch()`
calls `is_safe_valid_url()`, which deliberately jinja2-renders any URL
containing `{%`, and `?date={%` is a template syntax error. `add_watch()`
returns `None`, and the line is counted as skipped. So the user's import
silently drops exactly the watches whose URLs use the feature, and the same URL
that imports as nothing is accepted when added through the quick-add form.

## Invariant violated

A delimiter-based split is sound only while the delimiter cannot occur inside a
field. Once a field's grammar admits the delimiter, the split has to become
grammar-aware: track the template-block nesting and separate only on whitespace
outside `{% %}` and `{{ }}`. The failure mode here is structural rather than
local. The feature that broke this parser (jinja2 in URLs) was added somewhere
else entirely, and adding a template language to a field retroactively
invalidates every naive parser of any format that field appears in. Those
parsers do not fail at the time of the change, so nothing points back at it.

The asymmetry is the tell. One entry point renders the template and the other
splits on its whitespace, so support for the feature is real on one path and
absent on the other. When a field gains a grammar, the set of code paths that
must learn that grammar is every path the field enters by, not the one the
feature was demonstrated on.

## Trigger

Importing any URL containing a jinja2 block with a space in it, which is the
ordinary way such blocks are written.

## Scope

The sibling report [#1568](https://github.com/dgtlmoon/changedetection.io/issues/1568)
is a literal space in a plain URL, and it is deliberately out of scope: the
import format uses a bare space as the URL/tag separator (`test_import.py:24`
imports `https://example.com tag1, other tag`), so a raw space in a URL is
genuinely ambiguous and cannot be resolved by the same rule. The jinja2 case is
unambiguous because `{% %}` delimits it. Grammar-aware parsing fixes what the
grammar actually disambiguates and nothing more.

## Repro

The repo's own Docker image built from its Dockerfile at `5eb2638`, branch vs
unmodified master. The new test `test_import_url_with_jinja2_whitespace` fails
on unmodified master (`Imported 0 new UUIDs`, the watch is skipped) and passes
with the fix. `pytest tests/test_import.py`: 7 passed, so the existing
space-separated tag tests (`https://example.com tag1, other tag`) are unchanged.
`pytest tests/unit/`: passed. The fix masks the jinja2 blocks while preserving
length, so offsets still map back to the line, then splits on the first space
outside them; lines with no jinja2 block take exactly the previous path.

**Not verified:** the two `tests/test_jinja2.py` failures observed are
pre-existing and identical on unmodified master (a 500 from the preview page,
unrelated to the importer). The Playwright and Selenium CI legs were not run,
as the change does not touch them.
