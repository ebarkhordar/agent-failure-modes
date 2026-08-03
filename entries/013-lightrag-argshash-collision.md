# Delimiter-less argument join collides distinct configs onto one cache key

- **Repo:** HKUDS/LightRAG
- **Surface:** `lightrag/utils.py::compute_args_hash` (LLM response-cache key)
- **Class:** cache keys & invalidation
- **Report:** [issue #3392](https://github.com/HKUDS/LightRAG/issues/3392) (closed as completed; fixed upstream in [PR #3435](https://github.com/HKUDS/LightRAG/pull/3435), merged 2026-07-21, via length-prefixed encoding of the arguments)

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
genuinely distinct configs hash differently. The report also frames the one-time
cache invalidation any fix causes on upgrade honestly.

## What shipped, and why the obvious fix was rejected

The issue proposed joining on a control character (`\x1e`) in the shared
primitive. Upstream declined that and chose length-prefixing, and the reasoning is
the more useful half of this entry. The merged docstring states it: length
prefixing is used "instead of a sentinel character because query text and
free-form fields can contain arbitrary characters, so any fixed delimiter could
still be constructed to collide." A regression test pins exactly that,
`test_inputs_containing_record_separator_do_not_collide`.

That is the correction worth carrying. A delimiter restores injectivity only if it
cannot appear inside a field, and for arbitrary user text no character satisfies
that. Choosing an exotic one makes collisions harder to hit by accident while
leaving them constructible on purpose, which is the wrong property for a cache key
that decides whose response another user receives. A length prefix needs no
forbidden character, because it tells the reader where the field ends before the
content is read.

The merged fix is also narrower than "fix every call site at once". It keeps the
legacy plain join for `len(args) <= 1`, so document, entity and relation IDs
already persisted through `compute_mdhash_id` stay byte-identical, and it
length-prefixes only the multi-argument calls where a collision is possible. A
key-derivation change is a migration: the injectivity argument applies to the whole
function, and what ships has to weigh it against every ID already stored under the
old scheme.

The fix was written by another outside contributor, not by us. What this lane
contributed was the report and the colliding pairs.
