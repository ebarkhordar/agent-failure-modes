# A save path and its paired load path pick different codecs, so the round trip crashes on the values where the two codecs disagree

- **Repo:** confident-ai/deepeval
- **Surface:** `deepeval/dataset/dataset.py` (`EvaluationDataset.save_as("csv")` writing the `additional_metadata` column, and `add_goldens_from_csv_file` reading it back)
- **Class:** round-trip & export fidelity
- **Fix:** [PR #2942](https://github.com/confident-ai/deepeval/pull/2942) (in review; self-discovered, no issue)

## Root cause

The CSV save path encodes each golden's `additional_metadata` with `json.dumps`, so a
Python dict `{"ok": True, "missing": None}` is written to the cell as the JSON text
`{"ok": true, "missing": null}`. The load path decodes the same cell with
`ast.literal_eval`. `ast.literal_eval` parses only Python literals, and `true`, `false`
and `null` are not Python literals, so it raises `ValueError: malformed node or string`
and aborts the entire CSV load, not just the offending row.

The two halves of the round trip use different serializers for one column. They agree on
the overlap of their grammars: a double-quoted string and a bare number are both valid
JSON and valid Python literals, so metadata built only from strings and numbers survives
the mismatch and loads correctly. The divergence is exactly the three JSON scalars that
have a different spelling in Python — the booleans and null — so the failure is confined
to the values a caller is least likely to put in a hand-written fixture and most likely to
have in real metadata (a `passed: False`, an optional field left `None`).

## Invariant violated

A value written by one codec must be read by that codec's inverse. When a save function and
its paired load function are wired to different serializers, they do not form an
encode/decode pair at all; they form two independent grammars that happen to overlap. The
round trip then appears correct for every input in the intersection of the two grammars and
fails for every input in the symmetric difference. Because the intersection is the "obvious"
data (strings, ints) and the difference is the "edge" data (bools, null, and anything whose
textual form differs between the two encodings), the bug is structurally invisible to any
test whose fixture stays inside the intersection — which a minimal fixture always does,
because a minimal fixture is built from the simplest values that exercise the path.

The reusable rule: a serializer choice on the write side is a contract the read side owes
exactly, and "it parses" is not "it parses correctly" — `ast.literal_eval` on JSON text
succeeds silently for the common case and raises only on the tokens the two formats spell
differently, so a load path can look interoperable for months. The correct fix reads the
column with the inverse of what wrote it (`json.loads` to match `json.dumps`), keeping the
old `ast.literal_eval` only as a fallback so cells written by the previous encoding still
load.

## Trigger

Call `EvaluationDataset.save_as("csv")` on a dataset whose goldens carry an
`additional_metadata` value containing a boolean or `None` at any depth, then reload the
file with `add_goldens_from_csv_file`. The reload raises `ValueError` and no golden is
imported. Metadata made only of strings and numbers round-trips without error, which is why
the gap survived: the shipped fixtures carry no metadata column at all, so nothing asserted
the current behavior in either direction.

## Repro

Reproduced in a clean `python:3.12-slim` container against HEAD, with module provenance
confirmed by `__file__`. A golden whose `additional_metadata` mixes `True`, `False`, `None`
and an integer is saved with `save_as("csv")` and reloaded with
`add_goldens_from_csv_file`: on HEAD the reload raises `ValueError: malformed node or
string` from `ast.literal_eval`; with the `json.loads`-first fix the round trip returns the
metadata unchanged. The added regression test
(`test_save_as_round_trips_additional_metadata`) asserts equality across the round trip; it
fails on `main` and passes on the branch. The Confident AI push/pull paths are separate from
local file save/load and were not exercised.
