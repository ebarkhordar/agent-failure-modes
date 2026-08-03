# A bare `yield` in the batched branch makes the whole method a generator function, so the documented default call returns an empty generator

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/iterable_dataset.py`, `IterableDataset.to_dict` (bare `yield` in
  the `batched=True` branch, `return` in the other) and `IterableDataset.to_polars`
- **Class:** deferred returns
- **Fix:** [PR #8382](https://github.com/huggingface/datasets/pull/8382) (in review; issue
  [#8381](https://github.com/huggingface/datasets/issues/8381))

## Root cause

Both methods are written as if `batched` selects between streaming and returning:

```python
if batched:
    ...
    yield ...          # bare yield, the lazy branch
else:
    return Dataset(table, fingerprint="unset").to_dict()
```

Generator-ness in Python is decided at compile time, for the whole function, by the
presence of a `yield` anywhere in its body. The `else` branch is not a different
kind of function; it is the same generator function, and its `return` value does not
reach the caller. It becomes the `value` attribute of the `StopIteration` that ends
the generator, which nothing here reads.

So `ids.to_dict()` returns a generator on every call, in both modes. Iterating it
yields nothing, because the loop that would have yielded lives behind `if batched`.
The declared return type is `Union[dict, Iterator[dict]]` and the docstring's only
example is the bare `ds.to_dict()` call, so the documented default is the broken one.

The two affected methods are the residue of a partial repair. `to_pandas` had the
identical shape and was converted in
[#8068](https://github.com/huggingface/datasets/pull/8068) to return a generator
expression, which keeps the lazy branch lazy while leaving the function an ordinary
one. `to_dict` and `to_polars` were not swept along, and they are the only two of
the eight `to_*` methods on the class still affected.

## Invariant violated

**A `yield` anywhere in a function body is a property of the entire function, so a
method cannot return a value on one branch and stream on another.** The branch
structure suggests two behaviours and the language provides one. The `return` is not
dead code in the sense a linter reports, it runs, computes the correct dict, and has
its result discarded by the protocol.

**A silent failure needs no exception when the wrong type is plausible.** A generator
is iterable, so `for row in ds.to_dict()` runs cleanly and produces nothing, and code
that treats an empty result as an empty dataset never learns otherwise. The error
only becomes loud when someone subscripts it, which is why this survived in a
documented public method.

**A fix applied to one member of a family is a claim about the family.** #8068 solved
this exact problem for `to_pandas` and the shape of the solution transfers verbatim,
which is what makes the two untreated siblings hard to see: the repository contains a
worked answer, so the pattern reads as handled.

## Trigger

Any call to `IterableDataset.to_dict()` or `IterableDataset.to_polars()`, including
the no-argument form both docstrings demonstrate. It does not require streaming, a
particular backend, or a large dataset.

## Repro

Clean `python:3.11-slim` container against a pinned checkout, `PYTHONPATH` pointed at
the tree and `datasets.__file__` printed on each run for provenance (`5.0.1.dev0`,
pyarrow 25.0.0, polars 1.43.2). `IterableDataset.to_dict()` returns an object of type
`generator`; `list(...)` on it is `[]`; subscripting raises `TypeError: 'generator'
object is not subscriptable`; and the real `{'a': [1, 2, 3], 'b': ['x', 'y', 'z']}`
is recoverable from `StopIteration.value`, which is where the `return` deposited it.
`to_polars()` measures the same way and yields a real `pl.DataFrame` once fixed.
`Dataset.to_dict()` and the sibling `IterableDataset.to_list()` are correct on the
same run and serve as the controls.

The completeness claim is answered by parsing rather than by the test: `ast` over the
class, cross-checked against `inspect.isgeneratorfunction` on the live class,
classifies every `to_*` method by whether it contains a `yield` outside a nested
function or comprehension. Before the fix exactly two of the eight qualify; after it,
none.

Verified 2026-08-03.
