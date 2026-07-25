# Three dataset writers export nullable int columns through a bare to_pandas(), so pyarrow's default silently casts them to float

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/io/json.py` (`_batch_json`), `src/datasets/io/csv.py`
  (`_batch_csv`), and `src/datasets/io/sql.py` — each calls `batch.to_pandas()` with
  no arguments
- **Class:** round-trip & export fidelity
- **Fix:** [PR #8366](https://github.com/huggingface/datasets/pull/8366) (in review;
  issue [#8365](https://github.com/huggingface/datasets/issues/8365))

## Root cause

`to_json`, `to_csv`, and `to_sql` all convert each Arrow batch to pandas before
writing, and all three call `batch.to_pandas()` with no options. pyarrow's default
`integer_object_nulls=False` cannot keep a nullable integer column as an integer
dtype — a NaN has no `int64` representation — so it casts any `Value('int64')` column
that contains a null to `float64`. Every writer inherits that default. The
consequence is written to disk: an integer becomes a float in the output (JSON and
CSV can hold arbitrary-precision integers, so this is a pure fidelity loss, not a
format limitation), and any value past float64's exact-integer range (2^53) is
value-corrupted on the way out. A `to_json` → `load_dataset("json")` round trip
silently changes an `int64` column to `float64`.

The three writers are the same shape as the `to_pandas`/`to_polars` decode gap
(entry 062) and the `ArrayXD` fixed-dim widening (entry 086): a single conversion
helper's lossy default reached because the call site didn't override it.

## Invariant violated

When several export paths funnel a typed column through one conversion helper, the
option that governs type fidelity must be pinned at the helper, not left to a library
default whose lossy branch is invisible until a null appears. A nullable integer
column is still an integer column; exporting it to a format that can represent
integers (JSON, CSV, SQL) must not widen it to float because one row is null. The
default `integer_object_nulls=False` optimizes for a dense numeric in-memory frame,
which is the wrong objective for a serializer whose target can hold both integers and
nulls — the null belongs in the output as `null`, and the present values belong as
integers. The tell is that the corruption is conditional on a null being present:
correctness that depends on the data not exercising a code path is not correctness.

## Trigger

`Dataset.to_json`, `.to_csv`, or `.to_sql` on a dataset with an integer feature
column that contains at least one null. A column with no nulls exports as integers;
one null flips the whole column to float.

## Repro

Docker at HEAD `030a3e5` against an editable install (datasets 5.0.1.dev0, pyarrow
25.0.0). `Dataset.from_dict({'a': [9007199254740993, None]})` with an `int64`
feature, exported via `to_json`, writes `{"a": 9007199254740992.0}` — a float where
the source is `int64`, and off by one because the value exceeds 2^53. The same value
survives exactly when `to_pandas(integer_object_nulls=True)` is used. The fix routes
the writers through a conversion that preserves nullable integers (the
`integer_object_nulls` object-dtype path vs a nullable/arrow-backed dtype is the live
design choice flagged for the maintainer, since object dtype carries a throughput
cost); regression tests cover the json/csv/sql round trips.
