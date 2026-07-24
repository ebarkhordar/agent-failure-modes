# A ">= 1.3" version gate written as `major >= 1 and minor >= 3` fires on every pandas 2.0-2.2, dropping two supported CSV params

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/packaged_modules/csv/csv.py`, `CsvConfig.pd_read_csv_kwargs`
  (the pandas-version guard that deletes `_PANDAS_READ_CSV_NEW_1_3_0_PARAMETERS`)
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #8358](https://github.com/huggingface/datasets/pull/8358)
  (in review; self-discovered, no issue)

## Root cause

`pd_read_csv_kwargs` strips the two params pandas added in 1.3.0
(`encoding_errors`, `on_bad_lines`) when the installed pandas is older than 1.3. It
encodes "older than 1.3" by testing the major and minor numbers independently:

```python
if not (PANDAS_VERSION.major >= 1 and PANDAS_VERSION.minor >= 3):
    for p in _PANDAS_READ_CSV_NEW_1_3_0_PARAMETERS:  # ["encoding_errors", "on_bad_lines"]
        del pd_read_csv_kwargs[p]
```

`major >= 1 and minor >= 3` is not the predicate "version >= 1.3". It requires the
minor component to be at least 3 regardless of the major, so pandas 2.0, 2.1, and
2.2 (minor 0, 1, 2) fail it, the guard fires, and both params are deleted before the
kwargs are `**`-expanded into `pd.read_csv`. pandas then falls back to its defaults,
so a caller's `on_bad_lines="skip"` is silently discarded and a malformed row raises
`ParserError` instead of being skipped. The bug window is exactly `[2.0, 2.3)`:
pandas 1.3-1.5 and 2.3+ happen to satisfy the buggy expression by coincidence, so
the defect stayed invisible until a version whose minor dropped back below 3.

## Invariant violated

A version is ordered as a tuple, not as independent components: `(2, 0) > (1, 3)`
holds only because the major dominates, which a per-component `major >= 1 and
minor >= 3` conjunction throws away. Testing each component against the target
component answers a different question that coincides with real version order only
while the major stays pinned at the boundary's major. Any "is the version at least
X.Y" check must compare the release tuples (`PANDAS_VERSION.release >= (1, 3)`, or
here its negation `< (1, 3)`), because a component-wise conjunction re-breaks the
moment the major increments and the minor rolls back under the threshold, exactly
when a new major line ships. The tell is that the sibling guard two blocks below in
the same property already uses the tuple form (`PANDAS_VERSION.release >= (2, 2)`),
so the file contains the correct idiom and the bug is one predicate that fell out of
it.

## Trigger

pandas 2.0.x, 2.1.x, or 2.2.x installed, loading CSV with a non-default
`on_bad_lines` or `encoding_errors` (for example `load_dataset("csv", ...,
on_bad_lines="skip")`). The param is deleted, `pd.read_csv` uses its default of
raising on bad lines, and a malformed row aborts the load. pandas < 1.3 (params
genuinely absent) and 1.3-1.5 / 2.3+ (predicate accidentally true) are unaffected.

## Repro

Docker-verified this session against real pandas 2.2.3 (editable checkout):
`CsvConfig(on_bad_lines="skip").pd_read_csv_kwargs` omits `on_bad_lines`, and
`Csv(on_bad_lines="skip")._generate_tables(...)` on a malformed file raises the
pandas C-tokenizer `Error tokenizing data ... Expected 2 fields in line 3, saw 3`.
The fix replaces the predicate with `PANDAS_VERSION.release < (1, 3)`; after it, the
param is forwarded and the malformed row is skipped. Added regression tests
monkeypatch `PANDAS_VERSION` to 2.0.3 / 2.1.4 / 2.2.3 (both params survive) and to
1.1.5 / 1.2.5 (both still dropped), plus one end-to-end skip test on the installed
pandas; the CSV suites pass (41 passed, 1 skipped) and `ruff check` /
`ruff format --check` are clean.
