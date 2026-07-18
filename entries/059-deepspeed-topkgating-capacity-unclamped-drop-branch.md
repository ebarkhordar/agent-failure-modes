# A bound added to one branch of a fix, never carried to its siblings

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/moe/sharded_moe.py` — `topkgating` (`drop_policy='probs'`) and `top1gating` (`drop_tokens=True`)
- **Class:** initialization & control flow
- **Fix:** [PR #8155](https://github.com/deepspeedai/DeepSpeed/pull/8155) (in review)

## Root cause

MoE gating selects, per expert, the `capacity` highest-scoring tokens with
`torch.topk(x, k=capacity, dim=0)` over the token dimension. `torch.topk` requires
`k <= x.size(0)`, i.e. `capacity <= num_tokens`. But capacity is derived as
`ceil(num_tokens / num_experts * capacity_factor * k)`, which exceeds `num_tokens`
whenever `capacity_factor * k > num_experts`. When that holds, the `topk` call
raises `RuntimeError: selected index k out of range`.

PR [#5353](https://github.com/deepspeedai/DeepSpeed/pull/5353) ("Ensure capacity
does not exceed number of tokens") already recognised this and added the clamp
`capacity = min(capacity, num_tokens)` — but only to `top1gating`'s **no-drop**
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
precondition governs sibling branches — here three `torch.topk(dim=0)` sites born of
the same capacity formula — a clamp added to one of them is silently partial. The
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

Any gating call whose config makes `capacity_factor * k > num_experts` — e.g.
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
- With the clamp, both return; the dispatch buffer's capacity dimension equals
  `num_tokens`, and no token that should route is dropped (reducing capacity to
  `num_tokens` cannot remove a slot a token needed, since there are never more
  routed tokens than tokens).
- Two regression tests added under `tests/unit/moe/test_moe.py`, one per branch;
  both fail on the unpatched source and pass with the fix, and the pre-existing
  gating tests still pass.

The two `torch.topk(..., dim=0)` call sites were enumerated by parsing the module
with `ast` rather than grep: exactly the two above. `torch.topk(gates, k=k, dim=1)`
elsewhere is over the *expert* dimension with `k <= num_experts` and is not
involved.

**Not verified:** whether a real training configuration in the wild sets
`capacity_factor * k > num_experts` was not measured. The argument for the fix is
that the crash path is reachable from a legal config and that #5353 already
established `capacity > num_tokens` as a condition worth guarding.

## Fix

Apply the same `capacity <= num_tokens` clamp #5353 introduced, to the two drop
branches it missed. The clamp is a no-op whenever capacity already fits, so it
changes no behavior on any configuration that did not already crash.
