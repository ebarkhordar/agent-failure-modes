# One serializer in a family omits three fields its siblings write, so a save/load round-trip nulls them

- **Repo:** confident-ai/deepeval
- **Surface:** `deepeval/dataset/dataset.py` (`save_as`, the single-turn JSONL record dict)
- **Class:** round-trip & export fidelity
- **Fix:** [PR #2924](https://github.com/confident-ai/deepeval/pull/2924) (in review;
  self-discovered, no issue)

## Root cause

`save_as` serializes goldens through several record builders that are supposed to
be interchangeable: the JSON path, the multi-turn JSONL path, and the single-turn
JSONL path. The single-turn JSONL builder assembles its per-record dict from a
subset of the golden's fields and omits `name`, `comments`, and `source_file`. The
JSON path writes all three, the multi-turn JSONL record writes all three, and the
JSONL loader (`add_goldens_from_jsonl_file`) reads all three. So a golden saved as
single-turn JSONL and loaded back comes out with those three fields silently set to
`None`, while the same golden round-tripped through JSON or multi-turn JSONL keeps
them.

Nothing errors. The record is written, the file is valid, and the reload succeeds:
the loss is only visible if you compare the reloaded golden field by field against
the original, because the drop happens in the writer and the reader simply finds no
key to populate.

## Invariant violated

When a loader reads a set of fields, every serializer that feeds that loader must
write the same set, or a save/load cycle is lossy for whichever fields a serializer
skips. Round-trip fidelity is a property of the writer/reader pair, so it has to be
checked against the reader's full field list, not against what one writer happens to
emit. A family of sibling serializers is only safe when they are field-complete
against each other; the single-turn path here was the one member selected for a
narrower payload, and because the reader tolerates missing keys, the narrowing
degraded to silent data loss instead of an error.

## Trigger

Any golden carrying a `name`, `comments`, or `source_file`, saved with
`save_as('jsonl')` as a single-turn golden and later reloaded with
`add_goldens_from_jsonl_file`. The three fields come back `None`.

## Repro

Non-shallow clone at HEAD `58c9ef7`, deepeval installed editable (module
`__file__` provenance printed). A `Golden(name='golden-A', comments='reviewer note
here', source_file='/data/qa.txt')` saved and reloaded through the JSON path keeps
all three fields; through the single-turn JSONL path all three come back `None`,
and the written JSONL line contains no `name`, `comments`, or `source_file` key.
The fix adds the three fields to the single-turn record dict so it matches the
loader and the sibling writers; the added test round-trips a golden and asserts the
fields survive.
