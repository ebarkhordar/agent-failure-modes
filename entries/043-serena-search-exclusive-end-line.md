# A match ending at a line break marks the following line as matched

- **Repo:** oraios/serena
- **Surface:** `src/serena/util/text_utils.py::search_text`
- **Class:** indexing, ordering & counting contract
- **Fix:** [PR #1708](https://github.com/oraios/serena/pull/1708) (merged)

## Root cause

An exclusive index was consumed as if it pointed at content. `search_text`
resolved the end of a regex match with
`end_line_num = TextUtils.get_line_from_index(content, end_pos)`, where
`end_pos` is `match.end()`. `match.end()` is *exclusive*: it is the position
one past the last matched character. `get_line_from_index` has no notion of
that convention; it maps an index to the line that index points at. So when a
match ends with a line break, `end_pos` is the first index of the *next* line,
and the helper faithfully returns that next line.

The number is then consumed *inclusively*: any line up to and including
`end_line_num` falls through to `LineType.MATCH`. `context_end` is derived from
the same inflated number, so `context_lines_after` shifts by one as well. On
`content = "alpha\nbeta\ngamma\n"`, the pattern `alpha\n` marked lines 0 and 1
as matched while the matched text occupies only line 0; `alpha\nbeta\n` marked
0, 1 and 2. Matches ending mid-line were always correct, which is why the bug
survived: the common case hides it.

## Invariant violated

The lines a search result labels `MATCH` are exactly the lines the matched text
occupies. More generally: an exclusive end index and a position that points at
content are different coordinate systems, and converting between them is the
caller's obligation. A shared index-to-line helper cannot know which convention
its caller holds.

## Trigger

Any `search_for_pattern` whose regex ends in a line break. `multiline=True` is
the tool default, so `re.DOTALL | re.MULTILINE` is on and such patterns are
reachable through ordinary use.

## Repro

Pure stdlib, deterministic, no language server needed. On `main`,
`search_text(r"alpha\n", content="alpha\nbeta\ngamma\n")` returns
`matched_lines == [0, 1]` and `num_matched_lines == 2`; the fix returns `[0]`
and `1`. The control `search_text(r"alpha", ...)` returns `[0]` either way.

The fix is two lines at the call site, leaving `TextUtils` untouched: when the
match spans more than one line and `end_pos` sits at column 0, the match ended
on the preceding line's newline, so `end_line_num` is decremented. The `col ==
0` test is deliberate and is not interchangeable with inspecting `end_pos - 1`:
on CRLF content, `end_pos - 1` lands inside the `\r\n` pair, which
`get_line_col_from_index` maps to the following line by design, reintroducing
the same off-by-one.

## Note

The defect predates the area's own refactor. The previous implementation,
`content[:end_pos].count("\n")`, carried the identical off-by-one, so the
commit that swapped it for `get_line_from_index` preserved the behavior rather
than introducing it: a semantics-preserving refactor of the *splitting
mechanism* that never adjudicated the *exclusive-end math*. A refactor passing
through a line is not a review of it.

User-visible effect is display fidelity rather than data loss:
`to_display_string()` renders the phantom line with the `>` matched-line
prefix, so the agent reading the tool output is shown a real, unmatched line as
part of the match.
