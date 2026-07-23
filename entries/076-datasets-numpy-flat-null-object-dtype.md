# A None-handling clause keyed on `isinstance(x, float)` promotes a whole numeric column to object dtype, but only when the dtype happens to subclass Python's float

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/formatting/formatting.py`, `NumpyArrowExtractor._arrow_array_to_numpy`, the `any(...)` that decides object dtype (the `np.isnan` clause around line 194)
- **Class:** round-trip & export fidelity
- **Fix:** [PR #8352](https://github.com/huggingface/datasets/pull/8352) (in review;
  issue [#8351](https://github.com/huggingface/datasets/issues/8351))

## Root cause

`_arrow_array_to_numpy` decides whether a column must become a `dtype=object` array
by testing whether any element is itself an ndarray (a genuine per-row array column)
or a null. [PR #3195](https://github.com/huggingface/datasets/pull/3195) ("More
robust None handling") added an unguarded clause to that test:

```python
or (isinstance(x, float) and np.isnan(x))
```

The intent was to catch a null row inside an *array* column, where the null arrives
as a scalar `nan` sitting among per-row ndarrays. But the clause fires on any float
`nan` at all, including a flat, homogeneous numeric column that contains no ndarrays.
So `with_format("numpy")` on a plain numeric column with nulls is forced to object
dtype instead of a numeric array with `nan` in the null positions.

The subtlety that makes the bug intermittent is a type-hierarchy accident:
`np.float64` *is* a subclass of Python's built-in `float`, but `np.float32` and
`np.float16` are *not*. So the `isinstance(x, float)` guard is true only for
`float64` elements. Integer columns are caught too, because pyarrow promotes an
integer column to `float64` once it holds a null. `float32` slips past untouched.
The result is that the same logical column, "numbers with some missing values,"
comes back as `object` for `float64`/`int` and as a proper numeric array for
`float32`. The object array holds `np.float64(nan)`, not `None`, so there is not
even a None-versus-NaN preservation story to justify it.

## Invariant violated

`with_format("numpy")` (and the batch and column extractor paths that share this
function) on a flat homogeneous numeric column must yield a numeric ndarray, with
`nan` for nulls, never `dtype=object`. Object promotion is reserved for `array` /
`ArrayXD` columns whose null rows appear as scalar `nan` among per-row ndarrays. The
deeper rule is about the predicate, not the format: a membership test written as
`isinstance(x, float)` silently partitions the numpy scalar dtypes, because only
`float64` subclasses Python `float`. A duck-typing check that looks dtype-agnostic
is in fact a `float64`-only check, so any behavior gated on it is inconsistent
across the float widths of otherwise identical data. When a branch must fire on
"a NaN," test the array's structure (does it actually contain ndarrays?), not the
Python type of one sampled element.

## Trigger

`ds.with_format("numpy")` (or the batch/column extractors) on a `float64` or
integer column that contains at least one null. The returned array has
`dtype=object`, and any vectorized math, dtype-dependent code, or model input that
expected a numeric array breaks; a `float32` column of the same shape is unaffected,
so the failure is silent and width-dependent.

## Repro

Reproduced this session in a clean `python:3.11-slim` container against HEAD
`959825da1`, with `datasets.formatting.formatting.__file__` printed to confirm the
in-tree module. Driving `NumpyArrowExtractor().extract_column` over single-column
tables:

```
float64  [1.0, None, 3.0]  -> dtype=object    (wrong; expected float64)
float32  [1.0, None, 3.0]  -> dtype=float32   (correct)
int64    [1,   None, 3]    -> dtype=object    (wrong; expected float64)
```

On the fix branch the same three inputs return `float64`, `float32`, `float64`, all
numeric. No network, sub-second. The added extractor tests assert numeric dtype for a
flat numeric column with nulls and still force object dtype for a real array column
with a null row, failing on `main` and passing on the branch.
