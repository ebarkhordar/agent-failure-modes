# Deleting the last line crashes when the file has no trailing newline

- **Repo:** oraios/serena
- **Surface:** `src/solidlsp/ls_utils.py::TextUtils.delete_text_between_positions`
- **Class:** text decomposition contract
- **Fix:** [PR #1678](https://github.com/oraios/serena/pull/1678) (in review)

## Root cause

Two decompositions of the same file disagree at the last line. The read side
numbers lines with `str.splitlines()`, which reports the same N lines whether
or not the file ends in `\n`. So the model reads N lines and issues
`delete_lines(k, N-1)`. The delete side addresses that as the position one line
past the last line — line `N`, column 0 — and resolves it with
`get_index_from_line_col`, which walks the text counting newlines. With no
trailing newline there is no closing newline to count, so it hits EOF and
raises `InvalidTextLocationError`. `delete_text_between_positions` had no guard
for the past-EOF end position, so deleting the last line of a file with no
trailing newline — a very common file shape — crashed instead of deleting.

## Invariant violated

A position that one decomposition emits as valid (the one-past-final-line
sentinel `str.splitlines()` implies) must be resolvable by the decomposition
that consumes it. When two components address the same text by line, the "end
of the last line" position must mean the same thing to both, independent of
whether the file ends in a newline.

## Trigger

Any line-based `delete_lines`/`replace_lines` whose range extends through the
final line of a file that has no trailing newline. Files without a trailing
newline are extremely common, so this is a routine last-line edit, not an edge
case.

## Repro

Pure stdlib, deterministic. On `main`,
`TextUtils.delete_text_between_positions("a\nb\nc", 2, 0, 3, 0)` raises
`InvalidTextLocationError`; the trailing-newline variant `"a\nb\nc\n"` deletes
cleanly. The fix clamps the unresolvable one-past-final-line end position to
end-of-file — the same guard `insert_text_at_position` already applies for that
position — so both variants return `("a\nb\n", "c")`. A start position that is
genuinely out of range still raises.

## Note

Found and fixed by my collaborator
[@AmirF194](https://github.com/AmirF194). Same family as
[007](007-serena-splitlines-desync.md): `str.splitlines()` and explicit
newline counting are not interchangeable line decompositions, and an agent that
searches or reads by one convention while editing by another lands off by a
line — here, off the end of the file.
