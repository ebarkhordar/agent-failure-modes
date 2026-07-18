# A fast path that appends onto the leaked loop variable instead of its accumulator

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/iterable_dataset.py::HorizontallyConcatenatedMultiSourcesExamplesIterable._iter_arrow`
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #8342](https://github.com/huggingface/datasets/pull/8342) (in review; issue [#8341](https://github.com/huggingface/datasets/issues/8341))

## Root cause

`concatenate_datasets([...], axis=1)` on iterable datasets glues the sources'
columns together side by side. There are two implementations of that glue: the
plain-Python `__iter__`, which merges dicts (`new_example.update(example)`), and an
Arrow fast path `_iter_arrow` that pulled in whenever a consumer wants Arrow,
NumPy, or tensor batches. The fast path builds the combined table like this:

```python
for key, pa_table in pa_table_iterator:      # fetch loop; pa_table is the source table
    ...
new_pa_table = pa_table                        # base = first source (intended)
for name, col in zip(table.column_names, table.columns):
    new_pa_table = pa_table.append_column(name, col)   # iterable_dataset.py:1179
```

The accumulator is `new_pa_table`, but the append reads `pa_table` — the variable
that leaked out of the fetch loop above, still bound to the *last* source's table.
So every iteration rebuilds `new_pa_table` from the wrong base, and the earlier
sources' columns are never carried forward. For sources with columns `[a, b]` and
`[c]`, the result is the second source's table with `c` appended to itself
(`[c, c]`); `a` and `b` are gone. With resolved features the mismatch surfaces one
step later as a `CastError`.

The plain-Python path directly above is correct. Only the Arrow fast path diverges,
so the bug is invisible to anyone who never requests an Arrow/tensor format.

## Invariant violated

`concatenate_datasets(axis=1)` yields every source's columns, in source order, and
the Arrow fast path yields the same column set as the plain-Python path it exists
to accelerate.

The reusable rule is narrower and sharper than "a typo": a loop variable from one
loop that is still in scope in the next is a bound name that reads correctly and
means the wrong thing. `pa_table` and `new_pa_table` differ by a prefix, both are
`pa.Table`, and `x = pa_table.append_column(...)` is a grammatical, type-correct
statement — nothing at the append site can tell you the base should have been the
accumulator. The only witness is the fetch loop several lines up, where `pa_table`
was last assigned. Whenever an accumulator loop lives below a fetch loop that binds
a similarly named variable, the accumulator's identity is asserted only by which
name you type, and Python keeps the stale one alive to be typed by mistake.

The second, structural half: when an optimized path and a reference path both exist,
the optimized one is only correct if a test drives *it*, not the reference. Here the
existing `axis=1` tests never asked for an Arrow format, so they exercised the
correct path and stayed green over a fast path that dropped half its columns.

## Trigger

An iterable `concatenate_datasets([...], axis=1)` whose result is consumed as Arrow,
NumPy, or a tensor format (`.with_format("arrow"|"numpy"|"torch")`, or a downstream
rebatch) — anything that routes through `_iter_arrow`. Plain iteration without a
columnar format takes the correct path and shows nothing.

## Repro

Clean `python:3.11-slim` container, editable install, `datasets.__file__` confirmed
to resolve to the checkout rather than site-packages.

```python
ds1 = Dataset.from_dict({"a": [1, 2], "b": [3, 4]}).to_iterable_dataset()
ds2 = Dataset.from_dict({"c": [5, 6]}).to_iterable_dataset()
out = concatenate_datasets([ds1, ds2], axis=1).with_format("arrow")
tbl = next(iter(out))
assert tbl.column_names == ["a", "b", "c"]   # fails on main: a, b dropped
```

Added `test_concatenate_datasets_axis_1_arrow_format` asserting the column names and
values; it fails on `main` (the `CastError`/dropped columns) and passes on the
branch. The three existing `test_concatenate_datasets_axis_1*` tests pass on both
sides, confirming the plain-Python path is untouched and the divergence is isolated
to the Arrow fast path.

**Not verified:** only the axis=1 concatenation and `with_format` slice of
`tests/test_iterable_dataset.py` was run (11 passed, 25 skipped for optional
torch/tf/jax), not the full suite. The other `append_column` call sites in the
module were not audited for the same leaked-variable shape.

## Fix

Append onto the accumulator: `new_pa_table = new_pa_table.append_column(name, col)`.
One token. It restores the invariant that the Arrow fast path yields the same
columns, in the same order, as the plain-Python path.
