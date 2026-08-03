# Deleting the last line crashes when the file has no trailing newline

- **Repo:** oraios/serena
- **Surface:** `src/solidlsp/ls_utils.py::TextUtils.delete_text_between_positions`
- **Class:** text decomposition contract
- **Fix:** [PR #1678](https://github.com/oraios/serena/pull/1678) (merged)

## Root cause

Two decompositions of the same file disagree at the last line. The read side, at
the time of this bug, numbered lines with `str.splitlines()`, which reports the
same N lines whether or not the file ends in `\n`. So the model reads N lines and
issues `delete_lines(k, N-1)`. (That read side has since changed: commit
`08bc0615`, "make read_file report the line numbers the editing tools use", moved
it to `TextUtils.split_lines`, which does distinguish the two file shapes. The
account here is of the code as it was when the defect existed.) The delete side addresses that as the position one line
past the last line (line `N`, column 0) and resolves it with
`get_index_from_line_col`, which walks the text counting newlines. With no
trailing newline there is no closing newline to count, so it hits EOF and
raises `InvalidTextLocationError`. `delete_text_between_positions` had no guard
for the past-EOF end position, so deleting the last line of a file with no
trailing newline (a very common file shape) crashed instead of deleting.

## Invariant violated

A position that one decomposition emits as valid (the one-past-final-line
sentinel `str.splitlines()` implies) must be resolvable by the decomposition
that consumes it. When two components address the same text by line, the "end
of the last line" position must mean the same thing to both, independent of
whether the file ends in a newline.

## Trigger

Any line-based `delete_lines`/`replace_lines` whose range extends through the
final line of a file that has no trailing newline. Files without a trailing
newline are extremely common, so the file shape is routine rather than exotic.
Reach is narrower than that suggests, and a maintainer said so on the PR: "most
our users never use the line-based editing tools since agent harnesses can do
that and thus the tools are disabled for them." A common file shape behind a
default-off tool is a common trigger on an uncommon path, and this entry
originally claimed only the first half.

## Repro

Pure stdlib, deterministic. On `main`,
`TextUtils.delete_text_between_positions("a\nb\nc", 2, 0, 3, 0)` raises
`InvalidTextLocationError`; the trailing-newline variant `"a\nb\nc\n"` deletes
cleanly. The fix clamps the unresolvable one-past-final-line end position to
end-of-file (the same guard `insert_text_at_position` already applies for that
position), so both variants leave the file as `"a\nb\n"`. A start position that is
genuinely out of range still raises.

The two variants agree on the resulting text, not on the whole return value: the
deleted text that comes back with it is `"c"` without the trailing newline and
`"c\n"` with it. An earlier version of this entry said both variants return
`("a\nb\n", "c")`, which overstates the guarantee, and the claim came from our own
PR body rather than from a measurement. The merged test pins only what is true,
`assert without_nl == with_nl == "a\nb\n"`, discarding the second element.

## Note

Found and fixed by my collaborator
[@AmirF194](https://github.com/AmirF194). Same family as
[007](007-serena-splitlines-desync.md): `str.splitlines()` and explicit
newline counting are not interchangeable line decompositions, and an agent that
searches or reads by one convention while editing by another lands off by a
line: here, off the end of the file.
