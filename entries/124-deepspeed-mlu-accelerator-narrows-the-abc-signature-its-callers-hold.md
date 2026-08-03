# One accelerator backend narrows the abstract signature its callers hold, so a keyword every call site passes is a TypeError on that device alone

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `accelerator/mlu_accelerator.py` (`pin_memory` at `:226`, `initial_seed` at
  `:81`, `create_graph` at `:186`), against `accelerator/abstract_accelerator.py`
  (`DeepSpeedAccelerator`, `pin_memory` at `:264`)
- **Class:** interface conformance across implementations
- **Fix:** [PR #8208](https://github.com/deepspeedai/DeepSpeed/pull/8208) (in review)

## Root cause

`DeepSpeedAccelerator` is an ABC with nine concrete backends. Callers hold the abstract
type, reaching it through `get_accelerator()`, so every call site is written against
the ABC's signatures and any backend may receive any of them.

`MLU_Accelerator` declares `pin_memory(self, tensor)` where the ABC declares
`pin_memory(self, tensor, align_bytes=1)`. Three in-tree call sites on the
ZeRO-Infinity / NVMe offload swap path pass `align_bytes=0` by keyword
(`runtime/swap_tensor/utils.py:188`, `partitioned_param_swapper.py:131` and `:398`), so
on this device they raise `TypeError` rather than differing quietly.

The other two divergences share the shape. `initial_seed` still requires the `seed`
argument that [#5569](https://github.com/deepspeedai/DeepSpeed/pull/5569) removed from
the ABC and from the six backends that existed then, because MLU landed three months
later in [#6472](https://github.com/deepspeedai/DeepSpeed/pull/6472) as a copy of the
pre-#5569 NPU code and reintroduced it. `create_graph` constructs
`torch.mlu.MLUGraph()` and never returns it, while `graph_process()`
(`runtime/utils.py:101`) passes the return value straight into `capture_to_graph()`
and then `replay_graph()`.

## Invariant violated

**A concrete implementation may widen the signature it inherits but never narrow it,
because the caller holds the abstract type and cannot know which one it has.** Adding a
parameter with a default, or giving a required parameter a default, keeps every
existing call valid. Dropping one of the abstract type's defaulted parameters, or
requiring an argument it does not declare, breaks calls that are correct against the
contract. This is the substitutability rule, and it is the whole reason the ABC exists.

**A late arrival copied from an earlier sibling inherits that sibling's past, not its
present.** The `initial_seed` divergence is a decision that was made once, applied to
every backend in the tree, and then undone by a copy of a pre-decision file three
months later. Nothing in the copy is wrong on its own terms; it is wrong only relative
to a change it predates, and a review of the new backend in isolation cannot see it.

**A missing return and a deliberate abstention are the same line of code and opposite
intentions, so they have to be told apart by their surroundings.** Five backends return
`None` from `create_graph` and pair it with a `noop_context()` `capture_to_graph`, which
is a coherent "this device does not capture graphs". MLU pairs its missing return with a
real `torch.mlu.graph(...)` capture and a real `graph.replay()`, which is the shape the
capturing backends use. The pairing, not the return, carries the intent.

## Trigger

Any DeepSpeed run on Cambricon MLU that touches the parameter swap path, which is
ZeRO-Infinity and NVMe offload, hits the `pin_memory` `TypeError` immediately. The
`create_graph` fault fires on any graph-capture path. `initial_seed` has no live in-tree
caller (its only reference, `runtime/pipe/module.py:200`, is commented out), so it is a
break in the public accelerator API rather than an in-tree crash.

## Invariant discharge

The claim that MLU is the sole violator quantifies over all nine backends, so it is
answered statically rather than by the test run. `ast` over every file in `accelerator/`
extracts each `ClassDef`'s methods with full argument lists and diffs them against the
ABC. Repo-wide there are exactly four signature disagreements. Two are the MLU faults
above. The other two are the cpu and mps `create_op_builder` positional rename
(`class_name` to `op_name`), which is harmless because the parameter is passed
positionally at every call site.

That distinction is why the shipped test is not an arity check. Every backend gives
`device`/`device_name` a default the ABC declares required, and that widening is correct.
Only requiring more than the ABC, or dropping one of its defaulted parameters, fails.

## Repro

Clean `python:3.12-slim` container, `--network none`, loading the real repository files
with `__file__` and sha256 provenance printed. `torch` is stubbed, which is legitimate
here because the fault is entirely at the Python signature level: pre-fix, `pin_memory`
raises `TypeError: unexpected keyword 'align_bytes'`, `initial_seed()` raises `TypeError:
missing positional 'seed'`, and `create_graph()` returns `None` so `replay_graph(None)`
raises `AttributeError: 'NoneType' object has no attribute 'replay'`. All three pass
post-fix. Two CPU-only tests added to the repo's own accelerator suite take it from
2 failed / 25 passed to 27 passed, with the eight other backends passing untouched
throughout.

**Not verified:** there is no Cambricon hardware on the machine that ran this and none in
the project's CI, so nothing executed on an MLU device. The three contract faults are
exercised directly; the downstream consequences described above, the swap path faulting
and `graph_process` mis-driving, are read from the call sites rather than observed.

Verified 2026-08-03 against `df84f6d8`.
