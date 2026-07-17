# One function, two return paths, two newline contracts

- **Repo:** oraios/serena
- **Surface:** `src/solidlsp/ls_utils.py::FileUtils.read_file` (the `charset_normalizer` fallback)
- **Class:** text decomposition contract
- **Fix:** [PR #1706](https://github.com/oraios/serena/pull/1706) (merged 2026-07-17)

## Root cause

`read_file` returns file contents by either of two paths, and they disagree about
newlines.

The primary path is `open(file_path, encoding=encoding).read()`. Python's default
`newline=None` means universal newlines apply, so `\r\n` and lone `\r` are
translated to `\n` before any caller sees them. The fallback path, added later so
that files which fail to decode with the configured encoding are still readable,
is `match.raw.decode(match.encoding)`. Decoding bytes performs no newline
translation at all.

So a file that decodes cleanly enters Serena LF-normalized, and an otherwise
identical file that trips the encoding fallback enters carrying its CR
characters. Nothing downstream expects the second case: the rest of the code
treats in-memory content as LF-normalized, and treats the `line_ending` setting
as the single point where line-ending translation happens, on write. Both writers
are built on that assumption (`code_editor.py:89` opens with
`newline=self.newline`, `file_tools.py:87` passes `newline=` to `write_text`).

The path is reachable through ordinary editing rather than through some exotic
call: `ls.py:164` fills `LSPFileBuffer._contents` from `read_file`,
`code_editor.py:265` hands that buffer back as the edited file's contents, and
`_save_edited_file` writes it out with `newline=self.newline`. Editing one symbol
in a non-UTF-8 file therefore pushes the whole file through a second translation.
Measured in a container before the fix, composing the fallback's output with the
real writer call shape: with `line_ending: lf` the file's 22 CRLF pairs came
through unchanged, so the setting did nothing, and with `line_ending: crlf` they
became `\r\r\n`.

## Invariant violated

Every return path of a function owes its callers the same contract. That is easy
to state and easy to miss here, because the contract in question is not written
anywhere in the function: it is a default argument of `open`.

Universal-newline translation is a property of how you opened the file, not of
the bytes and not of anything visible at the return statement. `open(...).read()`
normalizes; `bytes.decode(...)` does not. Both spell "read the file as text" in
one short expression, and the difference between them is a `newline=None` that
nobody typed. A second return path added years later for an unrelated reason (the
encoding fallback exists to handle undecodable files, not to have opinions about
newlines) inherits whichever behavior its chosen primitive happens to have, and
no reviewer of that patch was thinking about newlines at all.

The wider rule is about the "single point of translation" design that the
`line_ending` setting represents. Funneling all writes through one translator is
the right shape, and it silently carries a precondition: the data reaching the
translator must not already contain what it translates. Normalize-on-input is
what makes normalize-on-output well defined. When the input path has a hole, the
single translator does not fail loudly, it composes: applied to already-CRLF
text it produces `\r\r\n`, and configured for LF it becomes a no-op that looks
like a broken setting. Neither outcome points at the reader that let CR in.

So when auditing a pipeline that promises to handle X in exactly one place, do
not read that place. Read every entry point and ask whether X can already be
present on arrival.

## Trigger

A file that fails to decode with the project's configured `encoding`, which is
what sends it down the fallback, and which in practice means the mixed-encoding
legacy Windows and cp1252 codebases that motivated both the fallback and the
`line_ending` setting in the first place. Editing any symbol in such a file
rewrites the entire file's line endings.

## Repro

Clean `python:3.11-slim` container against HEAD `063f872a`, with the imported
module's sha256 checked against the repo file to confirm provenance. The fixture
is a realistic cp1252 file rather than a handful of bytes, because encoding
detection is unreliable on very short inputs, and the test asserts the fixture is
undecodable as UTF-8 so it cannot quietly stop exercising the fallback.

`python -m pytest test/solidlsp/util/test_ls_utils.py -q` gives `4 passed` on the
branch. With only `src/solidlsp/ls_utils.py` reverted to main and the new tests
kept: `2 failed, 2 passed`, the two failures being the fallback tests.
`test_read_file_primary_path_normalizes_crlf` passes on main by design, since it
pins the existing primary-path behavior that the fallback is being aligned to.

**Not verified:** behavior on Windows. `line_ending: native` resolves to
`os.linesep`, so it should double identically to the `crlf` case there, but the
test box is Linux (`os.linesep == '\n'`) and that was not run. The `\r\r\n` and
no-op measurements above were taken by composing the fallback's output with the
real writer call shape and the actual `LineEnding.*.newline_str` values, not by
driving a full edit-and-save through the LSP layer.

## Fix

Normalize to LF at the fallback's return, so both paths agree. One line at one
return path: no new helper, no change to encoding detection, and nothing touched
on the write side.
