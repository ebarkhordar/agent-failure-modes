# Two LR schedulers divide by a user-supplied step size with no guard, so a zero config crashes with a bare ZeroDivisionError instead of a named configuration error

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/runtime/lr_schedules.py`, `LRRangeTest._continuous_interval` (the `/ self.step_size` at line 347) and `OneCycle._initialize_cycle` (`step_ratio = cycle_first_step_size / self.total_size` at line 486)
- **Class:** error handling & success reporting
- **Fix:** [PR #8166](https://github.com/deepspeedai/DeepSpeed/pull/8166) (in review)

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

## Trigger

`LRRangeTest(optimizer, lr_range_test_step_size=0)` followed by `.step()`, and
`OneCycle(optimizer, cycle_first_step_size=0, cycle_second_step_size=0)` at
construction. Any valid (positive) config is unaffected: the guards fire only for
values `<= 0`, which previously crashed or produced a meaningless schedule.

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
