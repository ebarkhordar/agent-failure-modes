# A streaming filter reads whole Arrow tables but checkpoints only the emitted row, so resume skips every row already read-ahead

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/iterable_dataset.py`, `IterableDataset.filter` (the
  `FormattedExamplesIterable` wrap that omits the `batch_size=1` rebatch its `_map`
  sibling applies)
- **Class:** round-trip & export fidelity
- **Fix:** [PR #8360](https://github.com/huggingface/datasets/pull/8360) (merged 2026-07-24;
  issue [#8359](https://github.com/huggingface/datasets/issues/8359))

## Root cause

A non-batched `filter()` (the default, `batched=False`) wraps its source in a
`FormattedExamplesIterable`. That iterable's Arrow branch pulls a *whole* Arrow
table from the underlying `ArrowExamplesIterable` and advances the source's
`shard_example_idx` to the end of the table — a read-ahead of up to a full
row-group — while it hands examples downstream one at a time. So after emitting
1500 rows the consumer's cursor is at 1500, but the source's checkpoint cursor has
already jumped to the end of the last table it read (say 2000).

The two positions are meant to be reconciled by a rewind checkpoint, but
`MappedExamplesIterable` records `previous_state` only in its *batched* branch; the
non-batched filter path never records it, so it stays `None`. On resume,
`load_state_dict` finds no rewind, and `ArrowExamplesIterable._iter_arrow` does
`shard_example_idx += len(pa_table); if shard_example_idx <= start: continue` — it
skips the entire table it had already read, discarding every row in it that was read
but not yet emitted. The loss on each resume is `table_size - (consumed mod
table_size)`; on a single-table shard it drops *all* remaining rows.

`_map` does not have the bug because it inserts a `RebatchedArrowExamplesIterable(batch_size=1)`
ahead of the formatter, capping read-ahead at one row so the emit cursor and the
source cursor can never diverge. The merged precedent [#8147](https://github.com/huggingface/datasets/pull/8147)
applied exactly that fix to the `_map` path and left `filter()` untouched.

## Invariant violated

A stateful iterator that reads ahead — pulls a block but yields it element by element
— must checkpoint the position it has *emitted*, not the position its underlying
reader has *advanced to*. Whenever the emit cursor lags the read cursor, the resume
offset has to rewind to the emit cursor, or the unemitted read-ahead is silently
lost. There are only two correct implementations and the fix is whichever one the
siblings use: bound the source's read-ahead to one element before each checkpoint
(`batch_size=1`), or record a rewind checkpoint capturing how far ahead the reader
ran. When one member of a family of transforms (`_map`) applies that bound and a
sibling (`filter`) omits it, the sibling drops data across a `state_dict` /
`load_state_dict` cycle — a round trip that is supposed to be lossless — and the loss
is invisible until a training run resumes from a mid-shard checkpoint.

## Trigger

`load_dataset(..., streaming=True).filter(...)` (or `to_iterable_dataset().filter(...)`)
with the default `batched=False`, consumed partway, then resumed via
`state_dict()` / `load_state_dict()` — the standard `StatefulDataLoader`
training-resume path. `batched=True` filter, `map`, `skip`, `take`,
`select_columns`, and `rename_column` all resume without loss.

## Repro

Docker at HEAD `521a590` (datasets 5.0.1.dev0, pyarrow 25). Consume 1500 of a
5000-row single-shard `filter(lambda e: True)`, capture `state_dict()`, build a
fresh filtered dataset, `load_state_dict`, then drain it: it starts at row 2000
(length 8000 on the 10000-row variant) instead of row 1500, so rows 1500-1999 are
gone. A 10-row shard fully consumed and resumed yields `[]`. The same harness run
against `skip`, `take`, `select_columns`, `rename_column`, non-batched `map`,
`with_format('numpy')`, and batched `filter` shows zero loss — only non-batched
`filter` loses. The fix mirrors #8147 (wrap the arrow source in
`RebatchedArrowExamplesIterable(batch_size=1)` before formatting); the added
regression test consumes N, checkpoints, resumes, and asserts no rows are lost.
