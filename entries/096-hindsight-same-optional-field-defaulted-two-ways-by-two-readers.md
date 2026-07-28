# Two readers default the same absent field in opposite directions, so a scheduler selects rows the work it schedules then filters out

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight-api-slim/hindsight_api/engine/memory_engine.py`, the
  `trigger.tags_match` default in `_resolve_refresh_tag_filtering` (`:843` at the verified
  commit) against the separate default inside `compute_mental_model_is_stale`
  (`:11252-11254`)
- **Class:** configuration wiring & documented contracts
- **Report:** reproduced publicly on
  [issue #2808](https://github.com/vectorize-io/hindsight/issues/2808#issuecomment-5017611072)
  (closed as completed), fixed upstream by
  [PR #2804](https://github.com/vectorize-io/hindsight/pull/2804) (merged 2026-07-21).
  Triage only, no PR from us: the competing-PR gate failed (#2804 was already open and
  already modified the fault file at both sites), and the direction to resolve it in was a
  policy fork for the maintainers. See the closing note for how they resolved it.

## Root cause

A mental model is refreshed when a memory in its scope is newer than its last refresh.
Two functions compute that scope from the same optional field, `tags_match`, on the same
mental model's trigger JSON, and each supplies its own default when the field is absent:

- `_resolve_refresh_tag_filtering` (`:843`): `"all_strict" if model_tags else "any"`
- `compute_mental_model_is_stale` (`:11252-11254`): `tags_match = "any"`, with the inline
  comment "default: untagged MM is 'global', tagged MM matches any overlap"

`_parse_tags_match` turns those modes into different SQL operators, `@>` for `all_strict`
and `&&` for `any`. So for a tagged model with no explicit `tags_match`, the staleness
check selects memories by array overlap and the refresh it just scheduled filters the same
memories by contains-all. A memory tagged `["project:status"]` on a model tagged
`["projects", "mental-model"]` passes the first test and is dropped by the second: the
model is marked stale, refreshes, and produces empty content, forever, because nothing
about the next cycle differs.

Neither default is wrong on its own. Each is defensible where it sits, `all_strict`
documented as deliberate at `:832` ("security isolation"), `any` matching what every tool
the reflect agent actually calls uses (`reflect/tools.py:160`, `:242`,
`memory_engine.py:4058`, `:8948`). The defect is that both exist.

## Invariant violated

An optional field has exactly one default, and that default belongs to the field, not to
the reader. Written at the point of read, a default is a decision made repeatedly by
whoever happens to touch the code next, and nothing in the language or the tests requires
those decisions to agree. Resolve the absent value once, at the boundary where the record
is loaded or in one shared resolver both readers call, and the question cannot be answered
twice.

The pairing here is the dangerous case, worth stating generally: when a **selector** and
the **operation it selects for** each derive their scope from the same optional input, a
divergence between them is not merely inconsistent, it is a self-sustaining loop. The
looser reader admits an item, the stricter reader rejects it, and the selector's verdict is
never revised because its own criterion still holds. Work is scheduled forever and
accomplishes nothing, with no error, no exception, and no growing queue to notice. Compare
a divergence between two *readers of the same kind*, where the worst case is one query
disagreeing with another: visible, and diagnosable by comparing outputs.

Second, "security isolation" as the stated reason for one of the two defaults is a reason
to unify them urgently rather than a reason to trust the code. If a strict default exists
because a loose one would leak, then any sibling path defaulting the other way is the leak,
and it was written by someone who read the same field name and reached the opposite
conclusion in good faith. A default that carries a security argument should be
unavailable to redeclare.

## Trigger

A mental model with tags, whose trigger JSON does not set `tags_match` explicitly, in a
bank whose incoming memories do not carry the model's full tag set. It is marked stale on
every ingest and refreshes to empty content.

## Repro

Clean `python:3.11-slim`, `pip install -e hindsight-api-slim`, imports confirmed to come
from `/build/hindsight-api-slim/hindsight_api/engine/memory_engine.py`, at HEAD
`327aa05`, with `model_tags=["projects", "mental-model"]` and `trigger={}`:

```
REFRESH   tags_match = 'all_strict' -> ('@>', False)
STALENESS tags_match = 'any'        -> ('&&', True)
```

Both readers were driven on one in-process object, so the two lines are the same field,
the same commit, and the same input, differing only in which function read it.

**Scope of what was measured.** The two defaults and the operators they produce were
executed. The end-to-end consequence for the reporter's data (marked stale, refreshes
empty) was stated as inference from those operators and from the diff of the then-open
PR #2804, not run against a database, and the public comment said so.

Verified 2026-07-19 at HEAD `327aa05`. Fixed upstream: at HEAD `ed120a25`,
`compute_mental_model_is_stale` calls the shared `_resolve_refresh_tag_filtering(mm_tags,
trigger)` (`:12346`) and holds no default of its own, so the divergence is gone. The
maintainers resolved the fork toward the strict arm and added an escape hatch rather than
flipping the default: [PR #2804](https://github.com/vectorize-io/hindsight/pull/2804)
(merged 2026-07-21) unified the two readers keeping `all_strict` for tagged models, and
[PR #2858](https://github.com/vectorize-io/hindsight/pull/2858) (merged 2026-07-21) made
`tags_match` settable on every creation surface;
[PR #2813](https://github.com/vectorize-io/hindsight/pull/2813), which would have defaulted
to `any`, was closed unmerged. Issue #2808 is closed as completed.
