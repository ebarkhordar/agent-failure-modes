# Two LR schedulers divide by a user-supplied step size with no guard, so a zero config crashes with a bare ZeroDivisionError instead of a named configuration error

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/runtime/lr_schedules.py`, `LRRangeTest._continuous_interval` (the `/ self.step_size` at line 347) and `OneCycle._initialize_cycle` (`step_ratio = cycle_first_step_size / self.total_size` at line 486)
- **Class:** error handling & success reporting
- **Fix:** [PR #8166](https://github.com/deepspeedai/DeepSpeed/pull/8166), merged
  2026-07-29. The guard that shipped is wider than the one first proposed; the
  review round that widened it is the most reusable part of this case and has its
  own section below.

## Root cause

Both schedulers take a step-size value straight from user config and use it as a
divisor without checking it is positive:

- `LRRangeTest` divides the step index by `self.step_size` in
  `_continuous_interval` / `_staircase_interval`. With `lr_range_test_step_size=0`
  the constructor accepts the config, and the first `step()` raises a bare
  `ZeroDivisionError` from deep inside the LR math.
- `OneCycle` computes `self.step_ratio = cycle_first_step_size / self.total_size`,
  where `total_size = cycle_first_step_size + cycle_second_step_size`. With both
  halves `0`, `total_size` is `0` and the constructor itself raises
  `ZeroDivisionError`.
- `OneCycle` divides a *second* time, one frame further out, and that division has
  its own zero. `_get_scale_factor` evaluates `x / self.step_ratio`, so a
  `step_ratio` of `0.0` is as fatal as a `total_size` of `0`. `step_ratio` is
  `cycle_first_step_size / total_size`, which is `0.0` whenever the first half is
  `0` and the second half is positive. That config leaves `total_size` positive,
  so it walks past the obvious guard and fails later, at the first `get_lr()`
  before any `step()` and again at every cycle boundary.

The neighboring `WarmupLR` / `WarmupCosineLR` constructors already validate their
`warmup_num_steps` and raise a clear `ValueError`
([#8126](https://github.com/deepspeedai/DeepSpeed/pull/8126),
[#8142](https://github.com/deepspeedai/DeepSpeed/pull/8142), #8151). These two
schedulers were simply skipped when that guard pattern was added, so a zero step
size is the one degenerate config in this file that still fails as an arithmetic
error rather than a validation error.

## Invariant violated

A configuration value that will be used as a divisor must be validated where the
config is accepted, and the failure for a degenerate value must name the offending
parameter, not surface as a `ZeroDivisionError` three frames into the schedule
computation. A representable-but-nonsensical config (`step_size=0`) is a user error
about a specific field; the scheduler owes a message that says which field and why,
so the misconfiguration is diagnosable at construction. The general rule: when a
family of sibling constructors adopts an input-validation guard, the guard is part
of the contract for the whole family; a member that keeps dividing by an
unvalidated input is not merely missing a nicety, it converts a clear config error
into an opaque crash at an unrelated call site (LRRangeTest crashes only on the
first `step()`, far from where the bad value was supplied).

## The guard that shipped is not the guard first proposed

The first version of this fix validated `total_size`, the sum. A review bot on the
PR pointed out that the sum is not what the code divides by, and it was right:
`cycle_first_step_size=0` with a positive `cycle_second_step_size` keeps
`total_size` positive, passes a sum-only check, and then divides by a `step_ratio`
of `0.0` one frame later. A guard written against the arithmetic that happens to
be on the line you are reading does not cover the arithmetic two lines down. The
merged version validates each half on its own terms:

```python
if cycle_first_step_size <= 0:
    raise ValueError(f"cycle_first_step_size must be positive, got {cycle_first_step_size}")
if cycle_second_step_size < 0:
    raise ValueError(f"cycle_second_step_size must be non-negative, got {cycle_second_step_size}")
```

Note that the two halves get different predicates, and that asymmetry is the point.
A zero *second* half is a legal configuration, not a degenerate one: it gives
`step_ratio == 1.0`, `x` stays below `1.0` in `_get_scale_factor`, and no division
by zero is reachable. So the widened guard had to be widened asymmetrically, and
the PR pins that with its own test asserting the zero-second-half schedule
constructs and climbs.

**The generalisable rule: when a guard protects a division, guard the operand the
division actually uses, and check whether that operand is derived.** `total_size`
is a sum, `step_ratio` is a quotient of it, and a validity condition on a sum is
strictly weaker than the same condition on each addend. Every degenerate input that
survives a sum check is one where the addends cancel out or one addend is zero, and
those are exactly the inputs a downstream ratio will collapse on. The failure is
biased toward looking fixed: the sum-only guard makes the loudest case (both halves
zero, crash at construction) go away, and leaves the quieter one (first half zero,
crash at first `get_lr()`) untouched, so a regression test written for the case that
motivated the patch passes and the hole stays open.

The secondary lesson is procedural: the hole was found by a reviewer, not by us,
on a patch whose reproduction and differential were otherwise complete. A test
suite written from the same mental model as the fix inherits the model's blind
spot, and no amount of running it harder will surface a case the model never
contained.

## Trigger

`LRRangeTest(optimizer, lr_range_test_step_size=0)` followed by `.step()`;
`OneCycle(optimizer, cycle_first_step_size=0, cycle_second_step_size=0)` at
construction; and `OneCycle(optimizer, cycle_first_step_size=0,
cycle_second_step_size=100)`, which constructs fine and raises at the first
`get_lr()`. Any valid config is unaffected: the guards fire only on a non-positive
first half or a negative second half. A zero second half stays legal.

## Repro

Reproduced in a clean CPU-only `python:3.11-slim` container with CPU-only torch,
the DeepSpeed logger stubbed, and the module loaded from HEAD `886790b5`:
`LRRangeTest(opt, lr_range_test_step_size=0).step()` raises `ZeroDivisionError` at
`lr_schedules.py:347`; `OneCycle(opt, cycle_first_step_size=0,
cycle_second_step_size=0)` raises `ZeroDivisionError` at construction,
`lr_schedules.py:486`. As a control, the already-guarded `WarmupCosineLR`
`total == warmup` case (fixed in #8142) does not crash, confirming the two
unguarded schedulers are the remaining gap. The added CPU-only regression tests sit
beside the existing scheduler-validation tests and assert a `ValueError` naming the
bad parameter; they crash or silently accept the misconfig on `master` and pass on
the branch.

Scope of what was executed: the two crashes above and the regression differential
were run in that container. The zero-first-half shape was found in review and
confirmed by reading `_get_scale_factor` against `_initialize_cycle`, then pinned
by the parametrised test that shipped (`(0, 100)` is one of its cases), so it is
covered by the merged suite rather than by a separate container run of our own.

Merged upstream 2026-07-29. One caveat on what upstream CI proves here: the
`modal-torch-latest` workflow selects a fixed set of test targets and has never
executed `tests/unit/runtime/test_lr_schedulers.py`, so the passing checks on the
PR are not evidence about these tests. The differential above is.
