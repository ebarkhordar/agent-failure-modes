# Two line decompositions disagree: search reports the wrong line

- **Repo:** oraios/serena
- **Surface:** the defect lived in `src/serena/util/text_utils.py::search_text`; the merged
  fix is mostly in `src/solidlsp/ls_utils.py` (the new `TextStepper`), with `text_utils.py`
  taking only two call-site swaps
- **Class:** text decomposition contract
- **Fix:** upstream [PR #1691](https://github.com/oraios/serena/pull/1691), maintainer-authored (merged 2026-07-15). [PR #1684](https://github.com/oraios/serena/pull/1684), which reported and first fixed this and was written by my collaborator [@AmirF194](https://github.com/AmirF194), was closed as superseded by it.

## Root cause

The function built its line list with `str.splitlines()` but computed line
numbers with `content.count("\n")` and indexed back via `lines[line_num]`.
`splitlines()` also breaks on `\r`, `\v`, `\f`, `\x1c`-`\x1e`, `\x85`,
`U+2028/9`; counting `"\n"` does not. One such character before a match and the
two decompositions disagree, so the number names one line and the content
printed beside it belongs to another.

Which half was wrong matters, and an earlier version of this entry got it
backwards. The reported *number* came from `count("\n")`, which is the same
convention the editing helpers use, so the number was correct and consistent
with them. What broke was the *content*, fetched from the `splitlines()` list at
that number. So the failure is a search result that names the right line and
displays the wrong text, not a search-then-edit that lands on the wrong line.

## Invariant violated

The index used to fetch a line and the number reported for it must come from
the same decomposition, and that decomposition must match the one used by
every consumer of the number.

A number and a payload emitted together read as one observation, and a reader
has no way to tell that they were computed by different code. When they disagree
the honest half is indistinguishable from the broken half, which is why the first
diagnosis blamed the number: the wrong-looking output was the content, and the
content is what a human notices.

## Trigger

Form feeds, vertical tabs, or Unicode line separators in a searched file. Lone
`\r` belongs on that list only in principle: `search_text` reads files through
Python's universal-newline translation, so a bare CR never survives the read, and
the maintainer amended the changelog to say exactly that ("only `"\n"` appears in
files read by `FileUtils.read_file`, Serena's default reading mechanism"). Rare
either way, and the harm is a misleading search result rather than a bad edit.

## Repro

Pure stdlib, deterministic:
`'alpha\nbeta\x0cgamma\ndelta\nTARGET_here\n'`. Pre-fix, `search_text` reports
the match at line 3 and prints its content as `delta`. `TARGET_here` sits at
index 3 in the `\n` decomposition (which is what line 3 means to the editing
helpers) and at index 4 in the `splitlines()` list, and the code reports the
former while displaying the latter.

## Follow-up lesson (from review)

Two corrections came out of that thread, and this entry used to record only the
smaller one.

The small one: the first fix (`content.split("\n")`) handled the exotic trigger
but regressed the common one, since CRLF files kept a trailing `\r` per line that
`splitlines()` had been consuming. When replacing a normalizing stdlib function,
enumerate every semantic difference and test one fixture per difference *plus the
most common class member*, not just the char that motivated the bug. A maintainer
raised it 17 minutes after the PR opened, as a question rather than a diagnosis.

The large one is the reason the PR was superseded, and it is a verdict about
shape rather than about a missing fixture. Two successive helper revisions were
judged wrong: "The global helper doesn't make sense in this form. It is applied
only in `search_text` ... It wouldn't make sense in most other contexts,
however." The maintainers then took the work over, on the stated ground that
"all the splitting and counting need to be handled in a single place in a
consistent way." What merged does not align search to the `\n` convention at all;
it replaces that convention with LSP line semantics across `TextUtils`. So the
correct fix was not a better patch at the call site, it was a single owner for the
decomposition, and a contributor patching one call site at a time cannot reach it.
When a maintainer says the abstraction is in the wrong place, that is a design
review, and iterating on the patch answers a question nobody asked.
