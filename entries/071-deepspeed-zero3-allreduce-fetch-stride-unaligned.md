# A flat buffer sized in aligned units but walked in unaligned ones, so each parameter's span overlaps its predecessor's padding

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/runtime/zero/partition_parameters.py::Init._all_gather_coalesced`,
  the `stage3_use_all_reduce_for_fetch_params` branch
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #8158](https://github.com/deepspeedai/DeepSpeed/pull/8158) (in review;
  self-discovered, no issue)

## Root cause

The branch reserves the flat buffer in aligned units and then advances through
it in unaligned ones:

```python
flat_buffer_size = sum(p.ds_numel_aligned for p in params)   # aligned
...
    start_param += param.ds_numel                            # unaligned
```

`ds_numel_aligned` is `ds_numel` rounded up to a multiple of the partition world
size, so the two agree only when `ds_numel % world_size == 0`. For any padded
parameter they differ by the padding, and every subsequent parameter's base
offset lands that many slots *inside* its predecessor's aligned span.

The overlapping slots are the predecessor's partition padding, and that padding
is not zero: `_partition_param` allocates the partition with `torch.empty`, and
on the one rank whose partition extends past the end of the parameter it copies
only `elems_to_copy` elements, leaving the tail holding whatever the allocator
returned. Because this particular fetch reconstructs the parameter with an
`all_reduce` under SUM, that uninitialized tail is *added into* the next
parameter's leading elements. The sibling `all_gather` path writes the same
padding, but each parameter narrows to its own `ds_numel` out of the result, so
the padding is never read there. The corruption is specific to the SUM
reconstruction.

Nothing crashes. Total stride is smaller than the allocated buffer, so there is
no out-of-bounds access; the parameter simply comes back with wrong leading
values.

## Invariant violated

Each parameter owns a disjoint contiguous span of `ds_numel_aligned` slots in
the flat tensor, which is exactly what `flat_buffer_size` already reserves for
it. Sizing and striding are two readings of one layout, and a layout is only
coherent if both use the same unit; when they diverge the allocation stays
generous enough to hide the disagreement, so the error surfaces as quiet data
corruption instead of a bounds failure.

Two things make this shape worth generalizing. First, an alignment quantity that
exists solely to reserve room is a standing invitation to this bug: code that
allocates remembers the alignment because the allocation is where alignment is
visible, and code that walks forgets it because iteration reads naturally in
element counts. Any place where a buffer is sized by one expression and traversed
by another deserves the question of whether those two expressions are the same
unit. Second, the reduction operator decides whether padding is inert or toxic.
The same overwritten-padding layout is harmless under a gather (each reader
narrows to its own slice) and corrupting under a SUM (every slot accumulates
every writer). Padding is only ignorable if some reader ignores it, and that is
a property of the combining operation, never of the padding.

Writer analysis rather than inspection: `ds_numel_aligned` has a single writer
in the package, `param.ds_numel_aligned = tensor_size` in `_partition_param`,
set in the same block that creates `ds_tensor`. It has two readers, the buffer
sizing above and the changed line. Since the sizing already reads the attribute
for every param in this same loop, the fix introduces no new requirement about
when the attribute must exist.

## Trigger

The `stage3_use_all_reduce_for_fetch_params` path, with at least one parameter
in the coalesced group whose element count is not divisible by the partition
world size, and at least one further parameter after it. Divisible sizes make
aligned and unaligned identical and the defect disappears, which is why ordinary
model shapes conceal it.

## Repro

CPU-only container, world_size 2 over gloo with `LOCAL_SIZE=2`, against HEAD
`4c275f9e` on an unshallowed clone, running
`tests/unit/runtime/zero/test_zero_allreduce_fetch_params.py` parametrized over
a padded case (numels 5 and 7) and an aligned case (numels 4 and 6). The padded
case fails on master and passes on the branch; the aligned case passes on both.
On master the padded case reports:

```
param p1 was not reconstructed exactly:
  expected [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0]
  got      [7778.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0]
```

7778 is the sentinel 7777 filling `p0`'s uninitialized padding plus `p1`'s real
first element of 1.0, which identifies the value as the SUM overlap rather than
generic garbage.

The sentinel deserves its own note, because it is the reason the first attempt
at this reproduction was wrong. The padding tail is uninitialized by
construction, but a *fresh* allocation usually reads back as zero (the OS hands
out zero pages), so the SUM adds zero and the bug looks absent. A first attempt
showed exactly that and appeared to disprove the defect. Uninitialized memory
only holds garbage once the caching allocator recycles a freed block, which is
the steady state of a real training loop and not of a short script. The test
therefore fills new allocations with a sentinel so the tail's contents are
deterministic rather than dependent on allocator history; it injects nothing
into the buffer under test. The general point: a repro of a
reads-uninitialized-memory defect must control the allocator's history, or its
green result is measuring the operating system's zero-page policy rather than
the code.
