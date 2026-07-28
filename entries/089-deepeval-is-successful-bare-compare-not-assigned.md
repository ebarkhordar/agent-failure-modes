# A recompute-the-verdict method evaluates `score >= threshold` but never assigns it, so it returns the stale success flag

- **Repo:** confident-ai/deepeval
- **Surface:** `deepeval/metrics/turn_relevancy/turn_relevancy.py` and
  `deepeval/metrics/topic_adherence/topic_adherence.py`, each `is_successful`
- **Class:** error handling & success reporting
- **Fix:** fixed upstream on `main` by
  [PR #2951](https://github.com/confident-ai/deepeval/pull/2951) (merged 2026-07-27 by
  penguine-ip), which deleted both per-metric `is_successful` overrides so the two metrics
  inherit `BaseConversationalMetric.is_successful` (`base_metric.py:158-168`), and that one
  assigns `self.success` on every branch before returning it. Our
  [PR #2956](https://github.com/confident-ai/deepeval/pull/2956), which fixed the two bare
  compares in place, was closed as superseded (self-discovered, no issue). The defect was
  not disputed; the repair landed as part of a wider refactor.

## Root cause

`is_successful()` is the method contracted to recompute a metric's pass/fail verdict
from its current `score` and `threshold`. In these two metrics the body is a bare
expression statement:

```python
def is_successful(self) -> bool:
    ...
    self.score >= self.threshold   # value computed, then discarded
    return self.success
```

The comparison is evaluated and thrown away (an expression statement is a no-op), so
the method returns whatever `self.success` already held. That is whatever a prior
`measure()` last wrote, or the class default `None` if `measure()` has not run or was
run on a different threshold. The intended line is `self.success = self.score >=
self.threshold`. AST-enumerating all 59 `is_successful` definitions in the package
shows exactly these 2 omit the assignment; the other 57 (`exact_match`,
`tool_permission`, `role_violation`, `faithfulness`, `contextual_recall`, ...) all
use the canonical assign-then-return form. Git blame shows the bare compare was
introduced defective in a 2025-12-11 refactor, never a deliberately removed
assignment.

## Invariant violated

A method whose named job is "recompute X from current state" must *store* what it
computes into the field it reports, not merely evaluate the expression. A boolean
comparison written as a statement has no effect; the method then silently reports the
last value the field happened to hold. The failure is masked whenever the visible
call order is `measure()` then `is_successful()`, because `measure()` sets `success`
correctly. But the method exists precisely to be called *independently*: after the
threshold is mutated, on a metric object reused or copied across test cases, or
before `measure()`. In each of those it returns a stale or `None` verdict. When a
large family of siblings shares one idiom (57 of 59) and a few omit its essential
assignment, the omission is a defect against the established contract, and the
sole-pattern differential is the evidence, not a matter of local style.

## Trigger

Calling `is_successful()` on `TurnRelevancyMetric` or `TopicAdherenceMetric` in any
order other than "immediately after a successful `measure()` at the same threshold":
e.g. re-checking after changing `threshold`, reusing a metric instance, or querying
the verdict before measurement. It returns `self.success` (default `None`) instead of
the recomputed boolean.

## Repro

Docker at HEAD `6cf2e02` (python:3.11-slim, `pip install .`, no API key). With
`error=None`, `score=0.9`, `threshold=0.5`, `ExactMatchMetric.is_successful()`
returns `True` (correct sibling) while `TurnRelevancyMetric.is_successful()` and
`TopicAdherenceMetric.is_successful()` both return `None`. The fix adds `self.success
= ` to the two bare compares (one line per file); a per-metric unit test sets
`score`/`threshold` and calls `is_successful()` standalone, failing on `main` and
passing on the branch.

The repair that landed upstream is the stronger one and worth naming, because it is the
general answer to this class: instead of adding the missing assignment at each override,
#2951 deleted the overrides so both metrics reach the base implementation that was already
correct. A duplicated method that a subclass reimplements only to restate is a place this
defect can reappear; removing the duplication removes the site.
