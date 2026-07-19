# One export path skips a decode every sibling path applies, so a typed column leaks its raw storage strings

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/arrow_dataset.py` (`Dataset.to_pandas`, `Dataset.to_polars`)
- **Class:** round-trip & export fidelity
- **Fix:** [PR #8344](https://github.com/huggingface/datasets/pull/8344) (in review;
  issue [#8343](https://github.com/huggingface/datasets/issues/8343))

## Root cause

A `Json()` feature column is stored in Arrow as a string of serialized JSON. Every
in-memory export decodes it back to a Python object before returning: `to_dict`,
`to_list`, `to_json`, and `with_format("pandas")` all run
`get_json_field_paths_from_feature` followed by `json_decode_field`, so a caller
gets `{'a': 1}` (dict). `to_pandas` and `to_polars` convert the Arrow table
straight to a frame and never call the decode, so those two paths alone return
`'{"a":1}'` (str). Nested `List(Json())` columns have the same gap.

The column's declared feature type promises decoded objects, and four of the six
export paths deliver them. The value is not corrupted, it is simply left in its
on-disk storage form on the two paths that forgot the decode step, so downstream
code that indexes into the column with `row["col"]["a"]` gets a `TypeError` on a
string subscript, or worse silently treats the JSON text as an opaque value.

## Invariant violated

A column's feature type is a contract that every export path owes equally: a
`Json()` column must materialize as decoded Python objects wherever it surfaces,
because the caller selected the export function for its container type, not for its
fidelity. When a decode or normalization step lives in some members of a family of
sibling export functions but not others, the ones that skip it do not fail loudly,
they hand back a value of the wrong type that happens to print, so the gap is
invisible until a consumer assumes the type the feature promised. The correct fix
mirrors the step the sibling paths already run, in both the single and batched
branches, rather than adding a new special case.

## Trigger

Any dataset with a `Json()` (or `List(Json())`) column exported through
`to_pandas()` or `to_polars()`. No configuration or edge input is needed; the
common case of "load a dataset with a JSON column and hand it to pandas" hits it.

## Repro

On an unshallowed clone at HEAD `41adfd0f9` (`datasets` 5.0.1.dev0):
`ds = Dataset.from_dict({'col': [{'a': 1}]}, features=Features({'col': Json()}))`.
`ds.to_dict()['col'][0]` and `ds.with_format('pandas')[:]['col'][0]` both return
`{'a': 1}` (dict), while `ds.to_pandas()['col'][0]` returns `'{"a":1}'` (str). The
`to_pandas` divergence was executed directly; `to_polars` was not run in this
session (polars was not installed) but shares the identical raw
`query_table` to `pl.from_arrow` path with no decode, so it is reported as the same
defect by inspection, not by a separate execution. The fix applies the decode after
the Arrow-to-frame conversion; the added test asserts the exported cell is a dict.
