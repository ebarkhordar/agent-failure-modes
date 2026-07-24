# A `[0]` collapses the per-group learning-rate list to a scalar, then a broadcast re-expands it, so every param group warms up to group 0's LR

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/runtime/lr_schedules.py`, `WarmupLR.__init__` (the
  `warmup_max_lr` fallback) feeding `_format_param`
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #8171](https://github.com/deepspeedai/DeepSpeed/pull/8171)
  (in review; self-discovered, no issue)

## Root cause

When `warmup_max_lr` is left unspecified, `WarmupLR` is meant to inherit the
optimizer's per-group learning rates. The fallback builds the full per-group list
and then immediately indexes it:

```python
warmup_max_lr = [group['lr'] for group in self.optimizer.param_groups][0]
```

The trailing `[0]` reduces the list of group LRs to a single scalar, group 0's LR.
`_format_param` then normalizes a scalar back to per-group form by broadcasting,
`[value] * len(param_groups)`, so the scalar is re-expanded to a uniform list where
every entry is group 0's LR. On an optimizer with multiple parameter groups at
distinct base LRs, groups 1..N warm up to group 0's target and their own configured
LRs are silently discarded. The comprehension gathers exactly the per-group values
the scheduler needs, and the `[0]` throws all but the first away one character
later.

## Invariant violated

A per-group schedule owes each group its own base value: the mapping from group to
warmup target must be a bijection over the groups, not a constant. Reducing a list
that carries one value per group down to a single element destroys that mapping, and
a later broadcast cannot recover it, it can only paper over the shape mismatch by
inventing a uniform list, which reads as correct (right length, right type) while
being wrong in content for every group past the first. When a value is legitimately
per-group, no step in its plumbing may collapse it to a scalar "representative";
the moment it does, downstream code that re-expands to per-group form fills the lost
positions with a duplicate rather than the dropped originals. The sibling
`WarmupCosineLR` had the identical multi-group collapse and was fixed the same way,
so this is one scheduler that fell out of a per-group contract its siblings already
keep.

## Trigger

Any `WarmupLR` built with `warmup_max_lr` unspecified on an optimizer whose param
groups have different LRs (a common setup: separate LR for a backbone vs a head, or
for decay vs no-decay groups). Every group warms to group 0's LR. A single-group
optimizer, or one where all groups share an LR, is unaffected; `WarmupDecayLR`
defaults `warmup_max_lr=0.001`, so this fallback path does not run there.

## Repro

Docker-verified this session in a CPU-only container against HEAD (real
`import deepspeed`, module resolved from the checkout). With two param groups at LR
0.1 and 0.2 and `warmup_max_lr` omitted, the computed max LRs are `[0.1, 0.1]`
before the fix (group 1 collapsed to group 0) and `[0.1, 0.2]` after. The fix drops
the trailing `[0]` so `_format_param` receives the full per-group list and each
group keeps its own base LR. The added regression test
(`test_warmup_lr_inherits_per_group_lr_when_max_unspecified`, mirroring the existing
per-group cosine test) fails on `master` (`assert [0.1, 0.1] == [0.1, 0.2]`) and
passes on the branch; the repo's formatting hooks (yapf, flake8, codespell) are
clean.
