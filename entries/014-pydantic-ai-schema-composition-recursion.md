# Schema transformer skips composition members when the node also has a type

- **Repo:** pydantic/pydantic-ai
- **Surface:** `pydantic_ai/_json_schema.py::JsonSchemaTransformer` (`_handle`)
- **Class:** schema traversal completeness
- **Report:** [issue #6439](https://github.com/pydantic/pydantic-ai/issues/6439) (upstream fix [PR #6440](https://github.com/pydantic/pydantic-ai/pull/6440))

## Root cause

Recursion into `allOf`/`anyOf`/`oneOf` members was gated behind
`elif type_ is None:`. A node that carries both a `type` and a composition
keyword (valid JSON Schema, and exactly what arrives from user or MCP tool
`inputSchema`s) had its composition members, and every subschema nested under
them, never walked. The provider-specific transform (stripping unsupported
keys, inlining) was therefore never applied to that whole branch of the tree.

## Invariant violated

A schema walker must visit every subschema exactly once. `allOf`/`anyOf`/
`oneOf` are independent applicators that can co-exist with `type`; gating their
traversal on the *absence* of `type` silently drops a valid, common branch.
Completeness of a recursive transform is a property of the traversal, not of
the individual node handler.

## Trigger

A tool or output schema whose node has both a `type` and a composition keyword,
e.g. `{"type": "object", "anyOf": [{"type": "string", "format": "date-time"}]}`.

## Repro

Offline, pure walker logic: running the transformer over the schema above
leaves `title`/`format` on the `anyOf` member on HEAD (member never visited),
and strips them once composition members are recursed even when a `type` is
present. The maintainers had just merged the `type is None` `allOf` recursion
([#6394](https://github.com/pydantic/pydantic-ai/pull/6394)), which left the
typed-sibling case as the remaining gap. Reported with the repro; their bot
generated PR #6440 covering exactly the reported case, so no competing PR was
opened.
