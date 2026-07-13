# Delimiter-less argument join collides distinct configs onto one cache key

- **Repo:** HKUDS/LightRAG
- **Surface:** `lightrag/utils.py::compute_args_hash` (LLM response-cache key)
- **Class:** cache-key & hashing collisions
- **Report:** [issue #3392](https://github.com/HKUDS/LightRAG/issues/3392) (deterministic repro, fix offered)

## Root cause

`compute_args_hash(*args)` builds its key as
`"".join(str(arg) for arg in args)`, with no field delimiter, then hashes it. The
query LLM-cache passes free text and integer parameters positionally
(`mode, query, response_type, top_k, chunk_top_k, ...`), so an adjacent-field
boundary shift produces an identical string. `(top_k=1, chunk_top_k=20)` and
`(top_k=12, chunk_top_k=0)` both stringify through `...120...`, hash to the
same md5, and share one cached LLM response; the same holds for two free-text
fields whose split moves by one character.

## Invariant violated

A key-derivation function must be injective over its logical inputs: two
semantically distinct argument tuples must never serialize to the same bytes.
Concatenation without an unambiguous separator is not injective: it lets one
field's tail merge with the next field's head.

## Trigger

Any two calls whose stringified adjacent arguments coincide under a boundary
shift. `response_type` is a user-settable free string, so a colliding pair is
trivially constructible.

## Repro

Offline and deterministic: the two numeric-boundary tuples above hash
identically, as do the two string-boundary tuples, while control tuples of
genuinely distinct configs hash differently. The smallest robust fix joins
with a control-char delimiter in the shared primitive, fixing every call site
at once; the report also frames the one-time cache invalidation this causes on
upgrade honestly.
