# A single null row widens a fixed-dim int array to float64 on the python read path, corrupting every value past 2^53

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/features/features.py`,
  `ArrayExtensionArray.to_numpy` (the fixed-first-dim branch) and the `to_pylist`
  path that consumes it
- **Class:** round-trip & export fidelity
- **Fix:** [PR #8363](https://github.com/huggingface/datasets/pull/8363) (merged 2026-07-24;
  issue [#8362](https://github.com/huggingface/datasets/issues/8362))

## Root cause

`ArrayExtensionArray.to_numpy` has two branches. The fixed-first-dim branch handles
null rows with a single blanket cast:

```python
numpy_arr = np.insert(numpy_arr.astype(np.float64), null_indices, np.nan, axis=0)
```

To make room for a `np.nan` placeholder in the null slots it casts the *entire*
column to `float64`, so every non-null integer value is re-typed to float, and any
integer past float64's exact-integer range (2^53) is permanently altered. A
`Array2D(shape=(1,2), dtype='int64')` column whose rows are `[[10,20]]`,
`[[30,40]]`, `None` comes back from `to_dict()` as `[[10.0,20.0]]`, `[[30.0,40.0]]`,
`[[nan,nan]]`: the ints are now floats and the null row is `[[nan,nan]]` rather than
`None`, even on the python read path where `None` is representable.

The dynamic-first-dim branch of the *same method* already does it correctly: it
builds an object array of per-row integer ndarrays and inserts `np.nan` only into
the null slot, preserving the dtype and precision of every present row.
[PR #5751](https://github.com/huggingface/datasets/pull/5751) added that correct
object-array handling to the dynamic branch and left the fixed-dim branch on the old
lossy cast. The result is an internal contradiction: `ds[0]['a']` (single-row python
format) returns the integer `[[10,20]]` while `to_dict()['a'][0]` on the same column
returns floats: the output type depends on whether some *other*, unrelated row is
null.

## Invariant violated

A null in one row is a per-row fact; it must not change the dtype or the values of
the other, non-null rows. Widening an integer column to `float64` so a NaN sentinel
fits corrupts exactly the values that make `int64` worth keeping (those above 2^53)
and changes the returned type on a read path (python/pylist) where the null could
simply be `None`. Null-representation and value-preservation are separable concerns:
insert the sentinel into the null positions without re-typing the positions that hold
real data. When two branches of one method face the same null case and one preserves
dtype while the other flattens it, the preserving branch is the proof that the
flattening branch is a bug, not a representation tradeoff. (A dense numeric format
that genuinely cannot hold `None`, numpy/tensor output, is a separate, legitimate
`float64`+`nan` representation and is left unchanged; only the python read path,
which can hold both ints and `None`, is corrected.)

## Trigger

A fixed-shape `ArrayXD` (`Array2D`/`Array3D`/... with a fully specified `shape`) and
`dtype` in an integer type, read through any python-format path (`to_dict`,
`to_pylist`, `to_pandas`) with at least one `None` row present. A column with no null
rows round-trips its integers correctly, so the corruption appears only once a null
enters the column.

## Repro

Docker at HEAD `521a590` (datasets 5.0.1.dev0, pyarrow 25.0.0, numpy 2.4.6). On
`{'a': [[[10,20]],[[30,40]], None]}` typed `Array2D(shape=(1,2), dtype='int64')`,
`to_dict()['a']` returns floats with `[[nan,nan]]` for the null row while
`ds[0]['a']` returns the integer array: output depends on an unrelated row's
nullness. The smoking gun: a row holding `9007199254740993` (2^53+1) round-trips as
`9007199254740992.0`, the exact integer silently changed, purely because another row
is `None`; the no-null control preserves it. The fix builds the python list for a
fixed-shape `ArrayXD` with nulls from `self.storage.to_pylist()` (ints and `None`
preserved) instead of the float-casting `to_numpy`; the dense numpy format is
unchanged and its existing test still passes, and a new regression test fails on
`main`, passes on the branch.
