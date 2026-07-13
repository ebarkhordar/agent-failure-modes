# Memory names could escape the memory directory

- **Repo:** oraios/serena
- **Surface:** `get_memory_file_path`
- **Class:** path & name handling
- **Fix:** [PR #1665](https://github.com/oraios/serena/pull/1665) (merged)

## Root cause

Memory names were joined onto the memory directory without rejecting absolute
paths or `..` traversal, so a crafted name resolved outside the intended
directory.

## Invariant violated

A name is not a path. Any user- or model-supplied identifier that becomes a
filesystem path must be validated to stay inside its root — especially in
agent systems, where the "user" supplying the name may itself be a model
following injected instructions.

## Trigger

A memory name that is absolute or contains traversal segments.

## Repro

Regression tests in the PR: absolute and escaping names rejected, normal
names unaffected.
