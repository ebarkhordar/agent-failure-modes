# A group-relative index is rebased onto the running per-item counter, so every item after the first in its group self-references

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight-api-slim/hindsight_api/engine/retain/fact_extraction.py`,
  `_convert_causal_relations` (`:2705`, parameter `fact_start_idx`) against its two call
  sites, which passed the loop's running `global_fact_idx` (`:2424/:2436/:2447`, same shape
  at `:2586/:2639/:2652`)
- **Class:** indexing, ordering & counting contracts
- **Report:** reproduced publicly on
  [issue #2934](https://github.com/vectorize-io/hindsight/issues/2934#issuecomment-5065435654),
  fixed upstream by
  [PR #2935](https://github.com/vectorize-io/hindsight/pull/2935) (merged 2026-07-24,
  "offset causal targets from the extraction-group start"). Triage only, no PR from us.

## Root cause

An LLM extraction pass returns facts in groups, and a fact may point at another fact in
its own group through `caused_by`. `target_fact_index` is defined group-relative, and the
codebase says so in three places: the field's docstring at `:176` ("Index of the related
fact in the facts array"), the extraction prompt at `:968` ("target_index must be < this
fact's index"), and both validators, which check the value against the fact's position
*within the LLM array* (`:1621` and `:2367`, `if target_idx < 0 or target_idx >= i`).

Converting to global indices therefore needs the group's starting offset. The function
was written for exactly that, named for it, and documented for it ("Adjusts
target_fact_index from content-relative to global indices"):

```python
def _convert_causal_relations(relations_from_llm, fact_start_idx: int) -> ...:
    ...
    target_fact_index=fact_start_idx + rel.target_fact_index,
```

Both call sites passed `global_fact_idx`, the counter incremented once per fact inside the
loop. At the group's first fact the running counter still equals the group start, so the
arithmetic is correct. At every later fact it has advanced, and the base is the *source
fact's own* global index rather than its group's. The emitted target is then at or past
the source, so a `caused_by` relation resolves to the fact that declares it, or past the
end. `link_utils.py:876` drops it with "Invalid target_fact_index", which is the silent
loss the reporter saw.

The two identifiers are numerically equal at the one place a small test would look, the
first fact of the first group, and diverge by exactly the fact's offset within its group
everywhere else.

## Invariant violated

An index is meaningless without its base, so the base is part of the value's type, and
translating between bases is the only place the two can be reconciled. When a function
exists to perform that translation, its parameter *is* the declaration of which base the
caller owes it, and passing a different quantity of the same Python type satisfies the
signature while inverting the meaning.

What makes this shape durable is that both quantities are plausible. `global_fact_idx` and
the group start are both non-negative ints naming a position in the same array, both are
in scope at the call site, and both are correct for the first item of every group. No type
checker, no assertion on range, and no validator downstream of the conversion can separate
them, because the wrong answer is a legal index into a real array. The only thing that
distinguishes them is the name, which is why the fix here is a rename plus a capture
before the loop (`extraction_group_start_idx = global_fact_idx`, hoisted above the
increment) rather than a change to any arithmetic.

The generalizable rule: when a loop maintains a running index and also needs the index the
current *batch* began at, capture the batch start explicitly before the loop body advances
anything, and never let the two be spelled the same way at a call site. A "start" and a
"current" that are equal on the first iteration are indistinguishable in every test that
uses one group of one item, and a fixture built to make the happy path pass will be exactly
that fixture.

The failure also degrades in the direction that hides it. A self-referential or
out-of-range target is *validated and dropped* rather than stored, so the corrupt link
never reaches the data; the symptom is missing causal edges, which reads as the extractor
having found nothing rather than as an arithmetic error, and there is no error to search
for.

## Trigger

Any fact that is not the first in its extraction group and carries a `caused_by` relation.
The first fact in each group is unaffected. So a single-fact extraction, or the first fact
of any batch, behaves correctly, and the loss scales with how many facts a group holds.

## Repro

Direct execution of the real `_convert_causal_relations` at HEAD `0db0d3ec` (no LLM, no
network, `lineno` printed from the loaded function to prove provenance), for a group
starting at global index 4 whose second fact (global 5) targets local index 0:

```
running real _convert_causal_relations from fact_extraction.py, lineno 2705
group_start=4, current_fact_global=5, target_fact_index(local)=0
  emitted global target_fact_index : 5 (== source fact 5 -> self-reference)
  correct global target_fact_index : 4 (group_start 4 + local 0)
```

Verified 2026-07-24 at HEAD `0db0d3ec`. Fixed upstream: at HEAD `ed120a25` both call sites
capture `extraction_group_start_idx = global_fact_idx` before the loop advances (`:2428`,
`:2623`) and the parameter carries that name (`:2706`), so the base is now the group's.
