# Renumber-fold folds zero-padded numeric prefixes, so the recovery is not byte-exact

- **Repo:** headroomlabs-ai/headroom
- **Surface:** `headroom/transforms/cross_turn_dedup.py::_LINENO_RE`
- **Class:** lossy fold in a prefer-false-negatives module
- **Fix:** [PR #2369](https://github.com/headroomlabs-ai/headroom/pull/2369) (merged)

## Root cause

On an HTTP tool-output re-read, `cross_turn_dedup` folds a contiguous span that
already appeared in an earlier block into a compact pointer. When the leading
line numbers shifted by a constant it carries the offset as a `delta`, and the
original bytes are meant to recover as `int(number) + delta`. The module
documents this renumber path as lossless for UNPADDED numbers only.

`_LINENO_RE = ^(\d+)(:|\t)(.*)$` never enforced that "unpadded" restriction:
`\d+` also matches a leading-zero prefix. A timestamped log row such as
`08:00:01 ...` is read as line number `8` rather than as data, so a re-read
that is a later window of the same hourly log folds under a uniform delta.
Recovery then renders `str(int("08") + 1)` as `"9"`, not `"09"`, so the round
trip is not byte-exact. This is a false-positive fold in a module whose stated
posture is to prefer false negatives (`CONTRIBUTING.md:129`,
`cross_turn_dedup.py:45-50`).

## Invariant violated

A fold that a lossless-recovery path emits must reproduce the original bytes
exactly. A numeric-prefix heuristic that feeds an arithmetic renumber may only
match prefixes for which `str(int(x) + delta)` reconstructs `x` byte-for-byte;
a zero-padded run breaks that identity the moment `delta` is nonzero.

## Trigger

Any re-read of tool output whose lines begin with a zero-padded numeric field
(a timestamp, a fixed-width counter) where the second read is shifted by a
constant from the first, so the fold takes a nonzero delta. Timestamped log
tails are the common shape.

## Repro

Restrict `_LINENO_RE` to `[1-9]\d*` so a leading-zero run stays non-numbered
and can fold only on an EXACT match (delta 0), never under a lossy renumber.
Real `grep -n` / `sed -n` / `rg -n` numbers never carry a leading zero, so the
intended renumber-fold feature is unchanged. A delta-aware reconstruction test
over a zero-padded input round-trips byte-exact on the fix and diverges on
`main`.

## Note

Same family as the fidelity-contract entries: two paths (the numeric heuristic
and the arithmetic recovery) agreed on what a "number" is only for the unpadded
subset, and the regex silently admitted a wider set than the recovery could
invert. The character class is the load-bearing guard, so the fix comments it
as such.
