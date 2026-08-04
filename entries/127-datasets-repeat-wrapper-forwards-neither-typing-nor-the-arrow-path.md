# A wrapper that only changes how many times rows come out silently changes what they are, because the properties it does not define default to "capability absent"

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/iterable_dataset.py`, `RepeatExamplesIterable` (`:2120` at
  `b7cb10b0e`), against its immediate neighbours `SkipExamplesIterable` (`:2012`) and
  `TakeExamplesIterable` (`:2175`)
- **Class:** interface conformance across implementations
- **Report:** [issue #8394](https://github.com/huggingface/datasets/issues/8394) (open)
- **Fix:** [PR #8395](https://github.com/huggingface/datasets/pull/8395) (open and
  unreviewed as of 2026-08-04; its CI is held awaiting maintainer approval, which the
  status rollup reports as no checks rather than as pending)

## Root cause

`IterableDataset` composes as a chain of `_BaseExamplesIterable` objects, each wrapping
the one below it. Three properties on that base class describe what the chain can do:
`iter_arrow` (the Arrow fast path, `None` when unavailable), `is_typed`, and `features`.
A wrapper that does not define them inherits the base class defaults, which say the
capability is absent.

`RepeatExamplesIterable` defines none of the three. `.repeat()` therefore puts a
non-Arrow, untyped parent on top of a child that may be both, and the chain is read from
the top. Two consequences follow from one omission:

- The Arrow path is switched off. `iter_arrow` reads `None` at the parent, so iteration
  falls back to the per-example path even when the source is an `ArrowExamplesIterable`.
- The declared types are lost. With no `features` and `is_typed` false, the formatter
  re-infers types from Python objects, and Python integers and floats infer as the widest
  numpy dtype rather than the declared one.

The neighbours make the omission legible: `SkipExamplesIterable` and
`TakeExamplesIterable` sandwich this class in the file and both carry the three-property
block. Walking every `_BaseExamplesIterable` subclass at `b7cb10b0e`, 12 of 14 wrap
another iterable, 11 of those 12 forward all three, and `RepeatExamplesIterable` is the
one that forwards none.

## Invariant violated

**A wrapper that changes cardinality must forward everything about identity.** Repeating
rows is an operation on how many, not on what: the contract `.repeat()` advertises is
that the same rows come out again. `.take()` and `.skip()` make the same kind of promise
and keep it. Any property describing what the rows are, or how they can be read, belongs
to the wrapped iterable and has to be handed up untouched.

**An unimplemented property is not neutral, it is an answer.** This is what makes the
failure quiet. Nothing here raises, and no code says "repeat does not support Arrow"; the
class simply says nothing, and a base class default turns that silence into a claim of
`iter_arrow = None`, `is_typed = False`, `features = None`. Forgetting to answer and
answering "no" are indistinguishable to the caller, so the missing code path is invisible
in review precisely because there is no code to review. The same shape appears whenever a
capability is advertised by an overridable member: an abstract method that raises makes an
omission loud, and a default-valued property makes it silent.

**Type declarations that are re-derived rather than carried decay at every hop.** The
dtypes here are not lost in a conversion; they are recomputed from values that no longer
remember them. A pipeline that forwards its schema is exact for any number of hops, and a
pipeline that re-infers is correct only when inference happens to agree, which for
`int32`, `float32` and `uint8` it does not.

**A sibling set is a checkable contract, and drift within it is measurable.** Nothing
declares these three properties mandatory for wrappers. The evidence that they are is
statistical and sits in the file itself: 11 of 12 comparable classes carry them. This
class has fallen out of that set once before, in the same way, and was fixed the same
way. [PR #7581](https://github.com/huggingface/datasets/pull/7581), "Add missing property
on `RepeatExamplesIterable`", merged 2025-06-05 from an outside author, renamed its
`n_shards` to `num_shards` and corrected its `shard_data_sources` signature after the
base class moved. A class that has drifted once is worth enumerating rather than reading.

## Trigger

Any `IterableDataset.repeat()` over an Arrow-backed source. Under `numpy` format the
declared dtypes widen on the first example; under `arrow` format the throughput collapses
to the per-example path. Neither raises, and both look like a property of the data rather
than of the wrapper.

## Repro

Clean `python:3.12-slim`, `datasets` main at `b7cb10b0e` installed with `pip install -e .`
(provenance printed from `/src/src/datasets/__init__.py`), pyarrow 25.0.0, numpy 2.5.1.

```python
features = Features({"i32": Value("int32"), "f32": Value("float32"), "u8": Value("uint8")})
ds = Dataset.from_dict({"i32": [1, 2], "f32": [1.5, 2.5], "u8": [3, 4]}, features=features)
ds = ds.to_iterable_dataset().with_format("numpy")
```

```
plain      {'i32': int32, 'f32': float32, 'u8': uint8}
take(2)    {'i32': int32, 'f32': float32, 'u8': uint8}
skip(0)    {'i32': int32, 'f32': float32, 'u8': uint8}
repeat(2)  {'i32': int64, 'f32': float64, 'u8': int64}
```

`take` and `skip` are the differential: same chain, same source, one property block apart.
On 100k rows under `arrow` format, `take(100000)` runs in 0.500s and `repeat(1)` in
6.644s. After the fix the dtypes are restored, the chain dump shows `iter_arrow` present
on the parent, `repeat(1)` runs in 0.469s against `take`'s 0.465s, and the values are
unchanged at `[1, 2, 1, 2]`. The repository's own suite goes from 441 to 445 passing, with
the same four pre-existing torch-missing interleave failures on both sides; the four new
tests fail on unpatched HEAD and pass after.

**Scope of what was measured.** `numpy` format, scalar `Value` columns, single-example
iteration. The torch formatter and non-scalar feature types were not checked, and the
issue says so. The resume-after-interruption evidence is the repository's own
`assert_load_state_dict_resumes_iteration` and `assert_load_state_dict_resumes_arrow_iteration`
helpers passing on the new Arrow path, not an enumeration of every writer of the state
dict.

Verified 2026-08-04 at `b7cb10b0e`.
