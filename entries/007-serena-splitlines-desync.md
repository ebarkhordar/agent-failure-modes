# Two line decompositions disagree: search reports the wrong line

- **Repo:** oraios/serena
- **Surface:** `src/serena/util/text_utils.py::search_text`
- **Class:** text decomposition contract
- **Fix:** upstream [PR #1691](https://github.com/oraios/serena/pull/1691), maintainer-authored (merged 2026-07-15). [PR #1684](https://github.com/oraios/serena/pull/1684), which reported and first fixed this, was closed as superseded by it.

## Root cause

The function built its line list with `str.splitlines()` but computed line
numbers with `content.count("\n")` and indexed back via `lines[line_num]`.
`splitlines()` also breaks on `\r`, `\v`, `\f`, `\x1c`-`\x1e`, `\x85`,
`U+2028/9`; counting `"\n"` does not. One such character before a match and
the reported line content and number desync — and both diverge from the
`\n`-based convention the edit tools use, so a search-then-edit lands on the
wrong line.

## Invariant violated

The index used to fetch a line and the number reported for it must come from
the same decomposition, and that decomposition must match the one used by
every consumer of the number.

## Trigger

Form feeds, vertical tabs, lone CRs, or Unicode line separators in a searched
file. Rare, but agent workflows (search → edit by line number) turn it into
silent wrong-file edits.

## Repro

Pure stdlib, deterministic: `'alpha\nbeta\x0cgamma\ndelta\nTARGET_here\n'` —
search reports the match at line 3 with content `delta`; read/edit tools
place TARGET at line 4.

## Follow-up lesson (from review)

The first fix (`content.split("\n")`) handled the exotic trigger but
regressed the common one: CRLF files kept a trailing `\r` per line —
`splitlines()` had been consuming it. When replacing a normalizing stdlib
function, enumerate every semantic difference and test one fixture per
difference *plus the most common class member*, not just the char that
motivated the bug. The maintainer caught it in 17 minutes.
