# The empty-row filter calls .strip() on every value, so a ragged CSV row (DictReader's None or restkey list) crashes the load the filter exists to survive

- **Repo:** UKGovernmentBEIS/inspect_ai
- **Surface:** `src/inspect_ai/dataset/_sources/csv.py` (`csv_dataset_reader`, the empty-row predicate at line 79)
- **Class:** error handling & success reporting
- **Report:** [issue #4546](https://github.com/UKGovernmentBEIS/inspect_ai/issues/4546) (open; fix offered). Reported, not yet fixed upstream.

## Root cause

`csv_dataset_reader` builds a `csv.DictReader` with the default `restval=None`
and `restkey=None`. That default is a contract about ragged rows: a row with
fewer fields than the header gets `None` for each missing column, and a row with
more fields collects the surplus into a `list` stored under the key `None`. The
empty-row filter one line later assumes every value is a string:

```python
if data and any(value.strip() for value in data.values()):
```

`None.strip()` (short row) and `list.strip()` (long row) both raise
`AttributeError`, aborting the entire dataset load. The crash only surfaces when
the ragged row is *otherwise* empty: a single non-blank field makes `any()`
short-circuit before it reaches the `None` or the list, so a fully-populated row
never trips it. That is the exact shape the filter exists to discard, so the
predicate fails precisely on its own target set. A well-formed empty row (every
column present, every value `""`) is skipped correctly; only the ragged empty
row crashes.

## Invariant violated

A predicate whose entire purpose is to tolerate a class of inputs must be total
over that class: it cannot itself raise on a member of the set it was written to
skip. The narrower rule is a type contract on the reader. Nothing that iterates a
`csv.DictReader` row may assume `row.values()` are all `str`, because the reader's
own default `restval`/`restkey` inject `None` and `list` for ragged rows. Any
per-value string operation in that loop must be total over `None` and `list`, or
the reader must be constructed with a `restval` that keeps the values in the
string domain the loop assumes. A guard that decides whether to keep a row must
degrade a malformed-but-blank row to "skip", never to an exception that takes the
whole load down with it.

## Trigger

A CSV whose row has a different column count than the header and is otherwise
blank: `input,target,id` followed by `,` (two fields for a three-column header)
raises `NoneType has no attribute 'strip'`; `a,b` followed by `,,x` collects
`['x']` under key `None` and raises `'list' object has no attribute 'strip'`. A
ragged row with any non-blank field is unaffected, because `any()` returns on the
real field before reaching the `None` or list.

## Repro

Clean container (`python:3.11-slim`, `pip install "git+https://github.com/UKGovernmentBEIS/inspect_ai"`)
against `main` HEAD `54c7b65a`, with `csv.__file__` printed to confirm the
installed source ran. Two inputs, each written to a temp file and loaded with
`csv_dataset(path)`: the short-row case (`input,target,id` header, `,` row) fails
on unmodified HEAD with `AttributeError: 'NoneType' object has no attribute
'strip'` at `csv.py:79`; the long-row case (`a,b` header, `,,x` row) fails on the
same line with `'list' object has no attribute 'strip'`. A control row that is
ragged but carries a non-blank field loads without error, confirming the crash is
gated on the row being otherwise empty. Verified at the loader level; the fix
(treat `None` and the `restkey` list as empty rather than calling `.strip()` on
them, so the filter returns `False` for both) was described in the report but not
yet landed upstream, so no merged fix is linked.
