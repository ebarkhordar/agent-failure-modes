# A length guard written with `is not` is correct up to 256 and raises on every correct batch above it

- **Repo:** confident-ai/deepeval
- **Surface:** the `batch_predict` result-length guard in six benchmarks:
  `deepeval/benchmarks/logi_qa/logi_qa.py:220`, `mmlu/mmlu.py:241`,
  `math_qa/math_qa.py:223`, `hellaswag/hellaswag.py:236`,
  `truthful_qa/truthful_qa.py:261`, `big_bench_hard/big_bench_hard.py:260`
- **Class:** indexing, ordering & counting contracts
- **Report:** reproduced publicly on
  [issue #2982](https://github.com/confident-ai/deepeval/issues/2982#issuecomment-5117736600)
  (open, filed by another contributor). No PR from us: the reporter claimed the fix and
  opened [PR #2983](https://github.com/confident-ai/deepeval/pull/2983), which is open.

## Root cause

Each benchmark checks that a custom model returned one generation per prompt before
scoring:

```python
if len(predictions) is not len(goldens):
    raise ValueError(
        "Custom `batch_generate` method did not return the same number of generations as the number of prompts."
    )
```

`is not` compares object identity. CPython caches the integers from -5 to 256 and
returns the same object for every reference to one of them, so while both counts are at
most 256 the two `len()` calls hand back the same cached object, identity holds, and the
guard behaves exactly like `!=`. Above 256 each call constructs a fresh `int`, identity
fails on values that are equal, and the guard raises on a batch where the model did
precisely what it was asked.

The failure is not in the model, the schema, or the prompt path: the container output
below shows `len(predictions) == len(goldens) == 257` immediately before the
`ValueError` that says they differ.

Two of the eight benchmarks that define `batch_predict` already write it correctly:
`squad.py:228` uses `len(predictions) != len(goldens)` and `drop.py:276` uses
`len(predictions) != effective_batch_size`. As in entry
[041](041-giskard-pydantic-extra-ignore-drops-params.md), the repo contains its own
counter-example, so the disagreement between two implementations of one check was free
to find and needed no judgement about which form is right.

## Invariant violated

**Identity is not equality, and for immutable values the two coincide only inside an
interning window the language does not promise.** `is` asks which object, `==` asks
which value; where a runtime happens to hand out one shared object per small integer,
the wrong question returns the right answer. Nothing in the language reference
guarantees that window, and CPython's is an implementation detail free to move between
versions.

The reusable part is what that does to the defect's shape. **A wrong operator that is
accidentally correct on small inputs converts a logic bug into a scale threshold**, and
a scale threshold is invisible to the entire practice that would normally catch a logic
bug. Unit fixtures are small by construction, review reads the line as an equality
test because that is what it means in English, and the guard fires correctly hundreds of
times during development. Coverage is not the missing property: every one of these
lines can be exercised, and passing tells you nothing, because the branch taken is
correct on the value you passed.

The second half is the direction of the failure. This guard fails toward raising, which
reads as strict rather than broken, and the exception text names the model as the party
at fault. A user whose eval run dies on a large batch sees a message accusing their
`batch_generate` of misbehaving, and the workaround they will find by bisecting
(lowering `batch_size`) is the one action that also hides the cause forever. **A guard
that reports the wrong party is worse than a missing guard**: it spends the report on a
false accusation, and the user's successful mitigation removes their reason to look
again.

## Trigger

`batch_size >= 257`, not dataset size. All six call sites chunk before calling
`batch_predict` (`logi_qa.py:84`: `goldens_batch = goldens[i : i + batch_size]`), so
`len(goldens)` inside `batch_predict` is bounded by `batch_size` and never by the number
of goldens. 10000 goldens at `batch_size=100` never trip it; 300 goldens at
`batch_size=300` trip on the first batch. The reported condition, "more than 256
goldens", is not the trigger, and `n_problems_per_task` is irrelevant to it.

Reaching the guard at all requires a custom `DeepEvalBaseLLM` with `batch_generate` and
`batch_size` set, which is the intended path for local and self-hosted models.

## Repro

Clean `python:3.11-slim` container on CPython 3.11.15, `pip install .` from the tree at
HEAD `0d100e37`, driving the real `LogiQA.batch_predict` with a model whose
`batch_generate` returns exactly one prediction per prompt:

```
n= 256  len(predictions)==len(goldens)==256  -> OK, 256 results
n= 257  len(predictions)==len(goldens)==257  -> ValueError: Custom `batch_generate` method did not return the same number of generations as the number of prompts.
n= 300  len(predictions)==len(goldens)==300  -> ValueError: (same)
```

The six sites are the complete set, established by parsing rather than by searching for
the string: an `ast` walk of all 914 Python files for `Is`/`IsNot` comparisons where
neither operand is a singleton literal returns 145 hits, of which these six are the only
`len(...) is not len(...)` occurrences. The other 139 were read individually and are
correct uses of `is`: sentinel checks (`_SKIP`, `_MISSING`, `_STOP`,
`PydanticUndefined`), type and enum identity (`origin is Union`,
`kind is RunnerStatusType.TIE`), and object identity in tests.

**Not verified:** no benchmark was run end to end against a real dataset. The claim
covers `batch_predict` and its guard, not the surrounding scoring loop.

Verified 2026-07-29 at HEAD `0d100e37`. Re-checked 2026-07-30: `main` is still
`0d100e37` and all six lines are unchanged. Reported, not fixed upstream at the time of
writing.
