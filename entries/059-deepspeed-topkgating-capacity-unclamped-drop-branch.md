# A bound added to one branch of a fix, never carried to its siblings

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/moe/sharded_moe.py`: `topkgating` (`drop_policy='probs'`) and `top1gating` (`drop_tokens=True`)
- **Class:** initialization & control flow
- **Fix:** [PR #8155](https://github.com/deepspeedai/DeepSpeed/pull/8155) (merged 2026-08-02)

## Root cause

MoE gating selects, per expert, the `capacity` highest-scoring tokens with
`torch.topk(x, k=capacity, dim=0)` over the token dimension. `torch.topk` requires
`k <= x.size(0)`, i.e. `capacity <= num_tokens`. But capacity is derived as
`ceil(num_tokens / num_experts * capacity_factor * k)`, which exceeds `num_tokens`
whenever `capacity_factor * k > num_experts`. When that holds, the `topk` call
raises `RuntimeError: selected index k out of range`.

PR [#5353](https://github.com/deepspeedai/DeepSpeed/pull/5353) ("Ensure capacity
does not exceed number of tokens") already recognised this and added the clamp
`capacity = min(capacity, num_tokens)`, but only to `top1gating`'s **no-drop**
branch. The two branches that actually feed `capacity` into `torch.topk(..., dim=0)`
were left unguarded:

- `topkgating`, `drop_policy='probs'`: `torch.topk(topk_masked_gates, k=capacity, dim=0)`
- `top1gating`, `drop_tokens=True`: `_top_idx(mask1_rand, capacity)` → `torch.topk(source, k=capacity, dim=0)`

`drop_policy='position'` and `top2gating` select with `torch.lt`, not `torch.topk`,
so they never hit this and are correctly left alone.

## Invariant violated

Every `capacity` passed to `torch.topk(..., k=capacity, dim=0)` over the token
dimension is `<= num_tokens`, on every gating path.

The generalisable failure is not the arithmetic; it is the shape of the fix that
introduced the arithmetic's guard. A precondition (`k <= size`) that is enforced by
a clamp at one call site is a property of *that* call site only. When the same
precondition governs sibling branches (here three `torch.topk(dim=0)` sites born of
the same capacity formula), a clamp added to one of them is silently partial. The
prior fix's own title, "Ensure capacity does not exceed number of tokens," states a
whole-function invariant, but the change delivered it for one of the branches that
needed it. A guard's scope is the branches it is written into, never the branches
its commit message describes.

The reason the gap survives: the fixed branch and the unfixed branches are not near
each other in the file, and the crash only fires when `capacity_factor * k >
num_experts`, an uncommon but entirely spec-legal configuration. So the code reads
as "already handled" (a clamp for this exact condition exists in the module) while
two paths a caller can legally reach still crash.

## Trigger

Any gating call whose config makes `capacity_factor * k > num_experts`, e.g.
`topkgating` with `k=2`, `capacity_factor=2`, `num_experts=2`, `drop_policy='probs'`,
or `top1gating` with `capacity_factor=4`, `drop_tokens=True`. No unusual input
tensor is needed; the crash is a function of the configuration alone.

## Repro

Clean `python:3.10-slim` Docker container at HEAD, `pip install -e .`,
`torch==2.5.1+cpu`, `sharded_moe.__file__` confirmed to resolve to the installed
source.

- On the unpatched branches, `topkgating(logits[8, 2], k=2, capacity_factor=2,
  drop_policy='probs')` raises `RuntimeError: selected index k out of range`, and
  `top1gating(logits[8, 2], capacity_factor=4, drop_tokens=True)` raises the same
  error via `_top_idx`.
- With the fix, both return, and the dispatch buffer's capacity dimension stays at
  the configured `capacity` (16 for the two cases above), not at `num_tokens`.
- Four regression tests added under `tests/unit/moe/test_moe.py`: one per crashing
  branch, one pinning that `drop_policy='position'` still pads to `min_capacity`,
  and one pinning that `top1gating`'s dispatch width stays divisible by the tensor
  parallel size. The first two fail on the unpatched source; all four pass with the
  fix, and the pre-existing gating tests still pass.

The two `torch.topk(..., dim=0)` call sites were enumerated by parsing the module
with `ast` rather than grep: exactly the two above. `torch.topk(gates, k=k, dim=1)`
elsewhere is over the *expert* dimension with `k <= num_experts` and is not
involved.

**Not verified:** whether a real training configuration in the wild sets
`capacity_factor * k > num_experts` was not measured. The argument for the fix is
that the crash path is reachable from a legal config and that #5353 already
established `capacity > num_tokens` as a condition worth guarding.

## Fix

Bound the `k` handed to `torch.topk(..., dim=0)` by the token dimension, in a local
`selection_capacity`, and leave `capacity` itself untouched:

```python
selection_capacity = min(capacity, torch.tensor(gates.size(0)).to(capacity.device))
_, capacity_indices = torch.topk(topk_masked_gates, k=selection_capacity, dim=0, sorted=False)
```

The first version of this PR carried #5353's clamp over verbatim, reassigning
`capacity = min(capacity, num_tokens)` in both drop branches, and the maintainer
([tohtana](https://github.com/deepspeedai/DeepSpeed/pull/8155#pullrequestreview-4821603998))
rejected it: `capacity` is not only the `topk` bound, it also sizes the dispatch
buffer downstream, and `drop_tokens()` requires that width to be divisible by the
tensor parallel size while `drop_policy='position'` pads it up to `min_capacity`.
Shrinking `capacity` to fix the crash therefore breaks tensor parallel MoE on
configurations that never crashed.

That correction is the reusable half of this entry. The invariant is real and stated
correctly (`k <= size` on every `topk(dim=0)` site), but a value that satisfies an
invariant at one use site is not free to change at its other use sites. `capacity`
carries two jobs here, selection bound and buffer width, and only the first one has
the bound. The safe repair for an overloaded value is a new local for the
constrained use, never a narrowing of the shared one.
