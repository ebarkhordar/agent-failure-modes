# Optional[str] env fields keep the literal "none" because the sentinel branch is unreachable

- **Repo:** assafelovic/gpt-researcher
- **Surface:** `config/config.py::convert_env_value`
- **Class:** initialization & control flow
- **Report:** [issue #1899](https://github.com/assafelovic/gpt-researcher/issues/1899) (closed as completed; fixed upstream in maintainer [PR #1910](https://github.com/assafelovic/gpt-researcher/pull/1910), merged)

## Root cause

For a `Union[str, None]` field, `get_args()` returns `(str, NoneType)` and the
coercion loops the members in order. It hits `str` first, and the single-type
conversion for `str` returns the env value unconditionally (it never raises),
so the loop returns before reaching the `NoneType` member whose branch maps
`"none"`/`"null"`/`""` to Python `None`. That sentinel branch is therefore
dead: `Optional[str]` fields (`AGENT_ROLE`, `REPORT_SOURCE`,
`IMAGE_GENERATION_MODEL`) keep the truthy literal string `"none"` instead of
`None`, and it flows downstream as a real role/source value.

## Invariant violated

A guard branch the code contains must be reachable on the inputs it was written
for. When a `Union` is resolved by trying members in order and a permissive
member (one that accepts anything, like `str`) precedes a sentinel member (like
`NoneType`), the permissive member shadows the sentinel and its handling never
runs. Order the checks so the specific/sentinel case is tested before the
catch-all.

## Trigger

An `Optional[str]` config field whose env value is `"none"`, `"null"`, or empty,
e.g. `AGENT_ROLE=none`.

## Repro

Reproduced in a clean `python:3.12-slim` install at HEAD:
`convert_env_value("AGENT_ROLE", "none", Union[str, None])` returns `"none"`
(and `"null"`/`""` likewise) instead of `None`. The fix tests the `NoneType`
sentinel before looping the remaining members.

## Note

Found and reported by my collaborator
[@AmirF194](https://github.com/AmirF194).
