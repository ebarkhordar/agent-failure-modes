# A "lossless" renumber fold whose line-number regex also matches a zero-padded timestamp

- **Repo:** headroomlabs-ai/headroom
- **Surface:** `headroom/transforms/cross_turn_dedup.py` (`_LINENO_RE`, the renumber-fold path)
- **Class:** text decomposition contracts
- **Fix:** [PR #2369](https://github.com/headroomlabs-ai/headroom/pull/2369) (merged
  2026-07-21; self-discovered, no issue)

## Root cause

When a tool re-reads output that already appeared, `cross_turn_dedup` folds the
repeated span into a compact pointer, and when the line numbers shifted by a
constant it stores that constant as a `delta`, recovering each original number as
`int(number) + delta`. The module documents this renumber path as lossless for
unpadded numbers only. But its gate, `_LINENO_RE = ^(\d+)(:|\t)(.*)$`, does not
encode that restriction: `\d+` also matches a leading-zero run. A timestamped log
row such as `08:00:01 ...` is read as line number `08`, not as data, so a later
window of the same hourly log folds under a uniform delta. Recovery then renders
`str(int("08") + 1)` as `"9"`, not `"09"`: the round-trip is no longer byte-exact.
This is a false-positive fold in a module whose stated posture is to prefer false
negatives.

## Invariant violated

A transform that advertises a lossless path must have its recognizer enforce the
exact precondition that makes the path lossless, not a looser superset of it.
Here the precondition is "the token is an unpadded integer that survives
`str(int(x) + delta)`"; the regex enforces only "a run of digits", which admits
zero-padded values whose textual form the recovery cannot reproduce. The digit
class in `[1-9]\d*` versus `\d+` is load-bearing: it is the line that decides
whether a leading-zero string is treated as recoverable data or left verbatim.
When a compaction claims byte-exact recovery, the claim is only as strong as the
pattern that decides what is eligible to be compacted, and a pattern that is
wider than the claim silently converts data it cannot round-trip into a lossy
fold. Real `grep -n` / `sed -n` / `rg -n` line numbers never carry a leading
zero, so tightening the class preserves the intended feature and closes the gap.

## Trigger

A re-read whose lines begin with zero-padded numeric prefixes (timestamps like
`08:00:01`, ordinals like `007:`) that differ from an earlier block by a constant
offset: the span folds under a delta and reconstructs with the padding stripped
or the value shifted, so the recovered bytes differ from the original.

## Repro

Clean `python:3.12-slim` container, source tree on `PYTHONPATH`, against the
branch. Three named scenarios, one test each: a padded shifted re-read is left
verbatim (`spans_folded == 0`, fails on `main`, which folds it lossily); an
unpadded `grep -n` read renumbered by `+5` still folds and reconstructs
byte-exact (the feature guard); the same padded rows re-displayed verbatim still
fold with delta 0 (the surgical-scope guard). Reverting `_LINENO_RE` to `\d+` to
simulate `main` makes the first regression test fail, as required. `ruff`,
`ruff format --check`, and `mypy` are clean. The existing `_reconstruct` test
helper asserted delta was absent, so it never exercised the numbered path this bug
lives on; a delta-aware helper was added alongside the tests.
