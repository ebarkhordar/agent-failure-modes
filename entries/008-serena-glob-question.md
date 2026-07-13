# Glob `?` consumed the wrong number of characters in glob-to-regex translation

- **Repo:** oraios/serena
- **Surface:** `glob_to_regex`
- **Class:** text decomposition contract
- **Fix:** [PR #1682](https://github.com/oraios/serena/pull/1682) (merged)

## Root cause

The glob-to-regex translation mishandled the `?` wildcard so a pattern's `?`
did not match exactly one character as glob semantics require.

## Invariant violated

`?` in a glob matches exactly one character — a spec-compliance contract.
When you implement a mini-language translator, its semantics are defined by
the source language's spec, not by what the regex happens to do.

## Trigger

Any file-matching pattern using `?` — silently wrong match sets for agent
file-search tools.

## Repro

Regression test in the PR: fails on the old translation, passes on the fix.
Reachability traced from the user-facing tool to the fault before filing.
