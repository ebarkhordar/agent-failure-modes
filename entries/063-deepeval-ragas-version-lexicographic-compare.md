# Comparing version strings with `<` orders them lexicographically, so 0.10.0 sorts below 0.2.1

- **Repo:** confident-ai/deepeval
- **Surface:** `deepeval/metrics/ragas.py` (`import_ragas`, the installed-vs-required version gate)
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #2915](https://github.com/confident-ai/deepeval/pull/2915) (in review;
  issue [#2905](https://github.com/confident-ai/deepeval/issues/2905))

## Root cause

`import_ragas()` checks that an adequate `ragas` is installed by comparing the two
version strings directly: `if installed_version < required_version: raise
ImportError(...)`, with `required_version = "0.2.1"`. `str.__lt__` compares
character by character, so `"0.10.0" < "0.2.1"` evaluates to `True`: at the third
character `'1'` sorts before `'2'`, and the comparison stops there without ever
reaching the numeric meaning of the components. Every `ragas` release whose minor
component reached two digits (0.10.x and up) is therefore rejected with "ragas
version 0.2.1 or higher is required, but 0.10.0 is installed", even though it far
exceeds the requirement.

Lexicographic order and version order agree only while every component stays a
single digit, which is exactly the regime a young dependency lives in, so the gate
passes its entire early life and turns into a false rejection at the first
two-digit minor. The failure is not a warning or a downgraded feature: the import
raises, so the RAGAS metrics become unusable on any modern `ragas`.

## Invariant violated

A version is a tuple of integers with its own ordering, not a string, and the two
orderings diverge the moment any component crosses from one digit to two. Any
"is the installed version at least X" check must parse both operands into a
semantic version (`packaging.version.Version`) and compare those, because a string
comparison answers a different question that only coincides with the intended one
for single-digit components. A comparison predicate that looks right on today's
inputs but rests on the wrong ordering is a latent gate that flips the day the
dependency ships a 0.10 or a 1.0.

## Trigger

Any environment with `ragas` 0.10.0 or newer installed. Importing any deepeval
RAGAS metric runs `import_ragas`, which raises `ImportError` claiming the
satisfied requirement is unmet.

## Repro

Clean `python:3.12-slim` container at HEAD `58c9ef7`, deepeval installed editable
with `ragas` 0.10.x present: calling `import_ragas()` raises the "0.2.1 or higher
is required, but 0.10.0 is installed" `ImportError`. The fix parses both operands
with `packaging.version.Version` so the comparison follows semantic order; the
added test pins that a two-digit-minor release passes the gate. It fails before the
fix and passes after.
