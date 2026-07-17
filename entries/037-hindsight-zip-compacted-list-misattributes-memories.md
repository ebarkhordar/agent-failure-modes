# Zipping a compacted list back onto the original: one bad id re-labels every memory after it

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight-api-slim/hindsight_api/engine/reflect/tools.py::tool_expand`
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #2759](https://github.com/vectorize-io/hindsight/pull/2759) (in review)

## Root cause

`tool_expand` validates the requested `memory_ids` into a second, shorter list,
then zips the two back together:

```python
valid_uuids: list[uuid.UUID] = []
errors: dict[str, str] = {}
for mid in memory_ids:
    try:
        valid_uuids.append(uuid.UUID(mid))     # only the ids that PARSE land here
    except ValueError:
        errors[mid] = f"Invalid memory_id format: {mid}"
...
for mid, mem_uuid in zip(memory_ids, valid_uuids):   # tools.py:398
```

`valid_uuids` is compacted, so `valid_uuids[i]` corresponds to `memory_ids[i]`
only while every preceding id parsed. One invalid id shifts every later pair by
one, and `zip` then truncates to the shorter list, dropping the tail.

Two consequences follow, and the first is the serious one:

1. A memory comes back **stamped with a different memory's id**. Within a single
   response item, `item["memory_id"]` and `item["memory"]["id"]` disagree. Nothing
   raises and nothing logs. The reflect agent is handed B's text under A's id and
   cites it as A.
2. The last requested id gets **no entry at all**, not even an error entry, so the
   `errors` branch that exists precisely to report a bad id does not fire for it.

Reachability is not theoretical. `_execute_tool` (`agent.py:1420-1425`) reads
`args.get("memory_ids", [])`, checks only that it is non-empty, and passes the raw
model-supplied array through; `tools_schema.py:121-125` declares `memory_ids` as a
free-form array of string. One hallucinated or truncated id among valid ones is
enough. The `errors` dict and the `if mid in errors` branch already in the
function are the code's own acknowledgement that such input arrives.

## Invariant violated

`tool_expand` returns one entry per requested `memory_id`, in request order, and
each entry's `memory` payload is the row whose id equals that entry's own
`memory_id`.

Positional correspondence between two lists is a contract that holds only while
both are built in lockstep, and a conditional append is exactly what breaks
lockstep. The general form: any `zip(a, b)` where `b` was built by appending
inside an `if` or a `try` is an unchecked claim that nothing was ever skipped.

`zip` is the specific instrument that makes the break silent. It does not raise
on unequal lengths and it does not pad; it truncates. That single design choice
produces both symptoms above, always as a pair: mis-pairing from the first skip
onward, and a lost tail at the end. So the two bugs are not two bugs. They are
what a length mismatch looks like when the consumer is `zip`.

`zip(..., strict=True)` turns this into a `ValueError`, which is the honest
failure: a mis-attributed memory is a wrong answer, and a crash is not. But
strictness only converts the tail loss into an exception. It cannot detect the
mis-pairing when the skip happens to be at the end and the lists still differ in
length, and it says nothing at all if two ids are dropped and two extra appear.
The durable fix is to stop having two lists. Key each element to its own derived
value, or carry the pair, so an element can only ever affect itself.

The deeper reason this survives review: the loop reads correctly. `for mid,
mem_uuid in zip(memory_ids, valid_uuids)` says "for each id and its uuid", and
that is what the author meant and what a reader sees. The falsehood is not in the
loop; it is in the eleven lines above, where the second list quietly stopped being
parallel. Correspondence is a property of a construction, and it is asserted at
the point of use by a name.

## Trigger

Any `expand` call where at least one `memory_id` fails to parse as a UUID and at
least one valid id follows it. With every id valid the pairing is correct, which
is why the compaction is the trigger rather than the lookup.

## Repro

Clean `python:3.12-slim` container, package installed non-editable, HEAD
`9676fc1`. Provenance pinned by sha256 identity between the installed module and
the checkout on both sides of the differential (unfixed
`6f05a294f95600e6...`, fixed `82d80e5f771d4399...`).

```
--- CASE 1: ["not-a-uuid", A, B]
    count returned: 2                          <- 3 requested
    entry memory_id=not-a-uuid -> error='Invalid memory_id format: not-a-uuid'
    entry memory_id=aaaaaaaa-...-aaaaaaaaaaaa
          memory.id=bbbbbbbb-...-bbbbbbbbbbbb  <-- MIS-ATTRIBUTION
          text='MEMORY-B: dogs are loyal animals'
    MISSING (requested, no entry at all): ['bbbbbbbb-...-bbbbbbbbbbbb']

--- CASE 2: [A, "bad"]
    count returned: 1
    MISSING (requested, no entry at all): ['bad']

--- CASE 3 (control): [A, B] -> both correct
```

Two regression tests, one per named behavior
(`test_tool_expand_pairs_each_memory_id_with_its_own_memory` for case 1,
`test_tool_expand_reports_a_trailing_invalid_memory_id` for case 2), fail on main
(`assert 1 == 2`, `assert 2 == 3`) and pass on the branch; the full file is 16
passed. They use the file's existing fake-connection idiom, so they need no
database and no LLM. The connection is the only thing faked and it supplies rows;
the pairing under test runs for real inside `tool_expand`.

**Not verified:** not run against a real Postgres. The other 27 `zip()` call sites
in the package were not audited for the same shape. How often a model actually
emits a malformed id here was not measured, so the frequency in production is
unknown; the argument for the fix is that the code already has an error path for
that input and mis-pairs instead of taking it.

`git log -S 'zip(memory_ids'` on an unshallowed clone returns exactly one commit,
`4f28338` (#132), which introduced the tool. This is original behavior, not a
regression, and there is no prior attempt or revert to answer.

## Fix

Key each id to its own UUID and iterate `memory_ids` directly, so an invalid id
can only affect its own entry. Nine lines added, six removed, no signature or API
change. One incidental effect was measured rather than assumed: `valid_uuids` is
now built from the dict's values, so a repeated id is sent to `WHERE id = ANY($1)`
once instead of once per occurrence, while the returned results are unchanged (a
duplicate id still gets one entry per occurrence, checked before and after).
