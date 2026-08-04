# A hand-rolled copy of a type's own `__add__` sums five of its seven fields, so recovered logs lose reasoning tokens and cost from the rollup

- **Repo:** UKGovernmentBEIS/inspect_ai
- **Surface:** `src/inspect_ai/log/_recover/_write.py`, the module-local `_add_usage` at
  `:290`, called from `_StatsAccumulator._add_stats` at `:275` and `:279`, against the
  canonical `ModelUsage.__add__` in `src/inspect_ai/model/_model_output.py`
- **Class:** round-trip & export fidelity
- **Fix:** [PR #4730](https://github.com/UKGovernmentBEIS/inspect_ai/pull/4730) (open and
  awaiting review as of 2026-08-04; no maintainer has commented)

## Root cause

`ModelUsage` carries seven fields. Its `__add__` sums all seven. The log recovery path
does not use it: `_write.py` defines its own `_add_usage`, whose docstring reads
`"""Sum two ModelUsage instances."""` and which sums five, omitting `reasoning_tokens`
and `total_cost`.

The result is a file that disagrees with itself. Recovering a crashed run writes an
`.eval` whose per-sample records carry real reasoning-token counts and real costs, and
whose top-level `stats.model_usage` rollup reports both as null. The rollup is the first
place most people look, and it is the only part of the file that is wrong.

This is drift, not a decision. `_add_usage` arrived in `5f270f3e1`
([#3652](https://github.com/UKGovernmentBEIS/inspect_ai/pull/3652)) already in its
five-field form, and both omitted fields were already on `ModelUsage` at that commit's
parent, so the copy was incomplete when it was written rather than correct and later
outgrown. Nothing has to change for a duplicate like this to become wrong; it only has
to be added to.

## Invariant violated

**A type that defines how to combine its own values owns that operation, and a second
implementation of it is a second definition of the type.** `ModelUsage.__add__` is not a
convenience; it is the answer to "what does it mean to add two usages". A local helper
that answers the same question differently makes the meaning depend on which code path
reached it.

**A duplicated operation does not stay duplicated, it decays, and the decay is silent
and one-directional.** Adding a field to `ModelUsage` updates `__add__` by construction,
because the operator is written against the type. The copy in another module is not
reached by that edit and no tool reports the omission, so every field added after the
copy is a field the copy drops. The gap can only widen.

**A docstring that states the contract is evidence about intent, not about behaviour.**
"Sum two ModelUsage instances" is exactly what the function was for and exactly what it
does not do, and its presence is what makes the five-field body read as complete.

## Trigger

Any recovered log from a crashed run, for any model reporting reasoning tokens or a
cost. The ordinary eval path aggregates the same per-sample data through
`ModelUsage.__add__` and is unaffected, so the two fields are correct everywhere except
in recovery, and the discrepancy appears only when comparing a recovered file against a
normal one.

## Invariant discharge

The sole-violator claim is static. `ast` over `src/inspect_ai` enumerates all 25
`ModelUsage(...)` constructions and identifies the paths that aggregate per-sample usage
into `EvalStats.model_usage`. There are exactly two: the normal eval path, which uses
`total_usage += usage` and therefore the operator, and this one.

## Repro

Clean `python:3.12-slim` container against a pinned checkout, running the repository's
own `tests/log/test_recover_write.py` with `reasoning_tokens` and `total_cost` added to
its sample fixture. The usage values travel through a real `.eval` ZIP serialize and
deserialize round trip via the repo's own helpers, and `_StatsAccumulator` is
instantiated by `write_recovered_eval_log` itself rather than hand-built, so the object
under test is the production one. Each of the two samples carries
`reasoning_tokens=3` and `total_cost=0.125`, so the rollup owes 6 and 0.25. Before: the
recovered `stats.model_usage` comes back with `reasoning_tokens: None` and
`total_cost: None` while those sample values sit in the same file. After: all seven
fields are populated and the rest of the file's tests still pass.

No mocks are involved in this test, so the question of whether a writer of the field was
mocked away is answered trivially, and the question behind it, whether a writer was
never instantiated at all, is answered by the accumulator being constructed by the real
entry point.

Verified 2026-08-03. A reader of the PR who is not a maintainer ran both claims
independently on 2026-08-04 and reported them holding, and made two suggestions that
went in: assert the recovered numbers as literals rather than recomputing them as
`usage + usage`, since a test that reuses the operator under discussion can agree with a
wrong implementation, and cover `role_usage` as well as `model_usage`, since
`_add_stats` calls the same helper on both. The test now asserts
`input 20 / output 10 / total 30 / cache_write 4 / cache_read 2 / reasoning 6 /
cost 0.25` as literals over both accumulators.
