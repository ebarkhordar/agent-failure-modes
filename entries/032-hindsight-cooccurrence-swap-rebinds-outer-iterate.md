# Canonicalising a pair by swapping the loop variables edits the loop, not just the pair

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight-api-slim/hindsight_api/engine/entity_resolver.py::EntityResolver._link_units_to_entities_batch_impl` (co-occurrence edge accumulation for the entity graph)
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #2750](https://github.com/vectorize-io/hindsight/pull/2750) (merged)

## Root cause

For each memory unit, the resolver enumerates the distinct pairs of the entities that
appear in it and records one co-occurrence edge per pair. The pair must be stored in a
canonical order, because `entity_cooccurrences` has a primary key and a check constraint
that require `entity_id_1 < entity_id_2`. The enumeration canonicalised by swapping:

```python
for i, entity_id_1 in enumerate(entity_list):
    for entity_id_2 in entity_list[i + 1 :]:
        if entity_id_1 == entity_id_2:
            continue
        if entity_id_1 > entity_id_2:
            entity_id_1, entity_id_2 = entity_id_2, entity_id_1
        key = (entity_id_1, entity_id_2)
```

`entity_id_1` is the outer loop's iterate. The swap rebinds it, and Python does not
restore it on the next inner iteration: only the *outer* `for` reassigns it, and the outer
`for` is not what is advancing. So the first out-of-order pair permanently replaces the
element the outer loop was working on, and every later inner iteration builds its pair off
the wrong element. The remaining pairs for that unit are wrong: some real edges are never
emitted, and the ones that are emitted may be duplicates of an earlier key.

The loss is order-dependent, and the order comes from `list(entity_ids)` over a set, so
whether an edge survives depends on set iteration order for that run. Measured over random
inputs at HEAD, edges were dropped on 50% of runs at 3 entities per unit and 98.5% at 6.
Edges were only ever missing, never fabricated, which is exactly why it is quiet: the graph
is always a subset of the truth, and a subset of a co-occurrence graph looks like a sparse
graph, not a broken one.

## Invariant violated

Enumerating the distinct pairs of a list of n elements must yield all C(n,2) of them, once
each, whatever the input order and whatever ordering the consumer requires of each pair.

The general rule is the one from [030](030-sagemaker-loop-variable-rebind.md), seen
from the other side. There, assigning to a loop variable failed to change the container and
the loop accomplished nothing. Here, assigning to a loop variable changes something the
author never intended to touch: the loop's own state. The two failures look like opposites
and share a cause, which is that a `for` target is an ordinary local name. It is not a slot
in the container (030) and it is not a fresh binding per pass that the loop will re-establish
(this entry). It is one name, bound once per iteration of *its own* loop, and anything else
that writes to it is writing to the loop.

What makes the swap idiom specifically dangerous is that it is correct almost everywhere it
appears. `a, b = b, a` to normalise a pair is a standard, readable line, and it is fine when
`a` and `b` are locals. It becomes a bug only when one of the names happens to be an iterate,
and nothing in the line records which names those are. The property is not in the statement;
it is in the two lines above it. So the rule to carry is that the swap idiom must be read
against the enclosing loop headers, or avoided entirely: order into fresh names
(`a, b = (x, y) if x < y else (y, x)`) and the question cannot arise.

The comment above the swap was accurate and did not help. It explained *why* canonical
order was needed (the PK and the check constraint) and said nothing about which names were
safe to write, which is the thing a reader had to know. A comment that justifies a line
tends to be read as a warrant for the line.

## Trigger

Any retain where a memory unit mentions three or more entities and the set happens to
present them out of canonical order. No configuration, no scale, no concurrency. The
downstream effect is a thinner entity graph than the ingested memories imply, which
weakens entity disambiguation.

The chain from missing edges to user-visible duplicate entities is read from the code, not
observed: no live memory bank was measured. The measured claim is only the dropped edges.

## Repro

Clean `python:3.11-slim`, against unmodified `main` at `37fa0ad`, driving the real
`_link_units_to_entities_batch_impl` with stub DB ops and no Postgres. 400 random runs at
each of n=3..6 entities per unit; the pair set produced is compared against
`itertools.combinations`. Unpatched: wrong on 50.0% / 84.0% / 95.8% / 98.5% of runs at
n=3/4/5/6, missing-only. With the fix: 0 of 400 wrong at every n.

## Fix

The merged fix lifts the enumeration into a small pure generator,
`_canonical_cooccurrence_pairs(entity_list)`, which orders each pair into fresh locals and
yields it, leaving both iterates untouched:

```python
yield (entity_id_1, entity_id_2) if entity_id_1 < entity_id_2 else (entity_id_2, entity_id_1)
```

Extracting it is what makes the regression test deterministic. A test through the DB layer
would depend on set iteration order and fail perhaps 96% of the time rather than always,
which reads as a flake and gets reverted. The pure helper is tested with explicit input
orders (`["C", "A", "B"]` must contain `("B", "C")`) and against
`itertools.combinations`, so it fails the same way every run.
