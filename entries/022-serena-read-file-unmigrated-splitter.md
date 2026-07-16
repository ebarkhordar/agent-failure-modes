# The last unmigrated caller of a replaced splitter keeps the old bug alive

- **Repo:** oraios/serena
- **Surface:** `src/serena/tools/file_tools.py::ReadFileTool.apply`
- **Class:** text decomposition contract
- **Fix:** [PR #1699](https://github.com/oraios/serena/pull/1699) (merged 2026-07-15, maintainer-amended before merge)

## Root cause

`delete_lines` and `replace_lines` both document that the agent must read a
range with `read_file` first, so the indices `read_file` reports are the ones
the editing tools act on. They came from different splitters. `ReadFileTool.apply`
decomposed the file with Python's `splitlines`, which breaks on `\f`, `\v`,
`\x1c`, `\x1d`, `\x1e`, `\x85`, U+2028 and U+2029 in addition to the real line
terminators; the editing path (`code_editor.delete_lines` into `TextUtils` /
`TextStepper`) counts only the LSP-compliant breaks. For a file holding any of
those characters the two disagree and every line after it shifts. Given
`"line0\n\x0cline1\nline2\n"`, `read_file` reports index 2 as `line1`, while
`delete_lines(2, 2)` removes `line2`. The agent reads a line, asks to delete
that index, and a different line is deleted, with no error.

The interesting part is how it survived. Maintainer PR
[#1691](https://github.com/oraios/serena/pull/1691) had already fixed this class
repo-wide: it made the editing side LSP-compliant and added `TextUtils.split_lines`
precisely so that "usages of Python's `splitlines` [can] be replaced." `read_file`
was the one content-splitting `splitlines` left in the tool layer. The migration
that fixed the bug everywhere else is what created this one, because it moved the
editing side and left the reading side behind.

## Invariant violated

When a codebase introduces its own primitive to replace a stdlib one whose
semantics are subtly wrong, the migration is not complete until every producer
and every consumer of the affected values moves together. A partial migration
does not shrink the defect, it relocates it to the seam between migrated and
unmigrated code, and that seam is now the one place where two decompositions are
guaranteed to differ. The remaining caller is the most dangerous one when it is
the tool whose numbers the migrated code contractually consumes.

## Trigger

Any file containing a separator that `splitlines` honors and the LSP line model
does not. A form feed is the realistic case: it is a page-break convention that
appears inside real source files. Ordinary LF, CRLF and lone-CR files are
unaffected, which is why every existing test passed.

## Repro

Clean `python:3.11-slim` container against the submitted branch, module provenance
asserted (`serena.tools.file_tools.__file__` resolves to `/build/src/serena/tools/file_tools.py`).
`python -m pytest test/serena/test_file_tools.py -q`: 20 passed on the branch;
with only `file_tools.py` reverted to `main` and the same new test, 6 failed and
14 passed, failing exactly the `\f`, `\v`, `\x1c`, `\x85` and U+2028 cases. The
14 that pass on both sides are the no-change cases (LF, CRLF, lone CR, no
trailing newline, empty).

## Lesson

The swap was not mechanical, and the disagreement over how to handle that is the
second half of the rule. `split_lines` yields a trailing empty line for
newline-terminated files where `splitlines` does not, so replacing the call
naively changes `read_file`'s full-file output for every ordinary file in the
repo. The submitted PR therefore dropped that phantom trailing line, making the
result byte-identical for all files without the exotic separators: the
conservative migration, on the reasoning that a bug fix should differ from the
old behavior only where the bug is.

The maintainer removed that line before merging and took the broader change: on
`main`, `read_file` now returns `TextUtils.split_lines` output unmodified, so a
newline-terminated file reads back with its trailing newline. He owns
`read_file`'s output contract, and he preferred the primitive's honest semantics
over compatibility with what the buggy version happened to emit.

Both instincts are defensible and the split is the lesson. Pinning the common
case byte-identical is the right default for an outside contributor, because it
is the version whose blast radius you can prove and the one that asks least of
the reviewer. But "minimize the diff against existing behavior" is a heuristic
for earning a merge, not a statement about correctness: where the old behavior
was itself an artifact of the bug, preserving it is preserving the artifact, and
that call belongs to whoever owns the contract. Propose the conservative version,
show the delta you suppressed and why, and let the maintainer take the wider one
if the coherence is worth it. Entry [007](007-serena-splitlines-desync.md) is the
same root family one file over, and its own follow-up lesson was learned the same
way.
