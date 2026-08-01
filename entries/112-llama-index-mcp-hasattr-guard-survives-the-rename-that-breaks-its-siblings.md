# A compatibility guard converts a dependency's field rename into an empty list, so the one break that needs porting is the one the port cannot see

- **Repo:** run-llama/llama_index
- **Surface:**
  `llama-index-integrations/tools/llama-index-tools-mcp/llama_index/tools/mcp/base.py:86`
  (`McpToolSpec.fetch_resources`), against the two import sites that raise first,
  `client.py:21` and `utils.py:4`; all three under `mcp>=2`
- **Class:** dependency & runtime contract drift
- **Report:** reproduced publicly on
  [issue #22515](https://github.com/run-llama/llama_index/issues/22515#issuecomment-5139459991)
  (open, filed by another user asking for `mcp` 2.x support). No PR from us: the reporter
  offered to write it, and how far to take the migration is a scoping call the maintainers
  own, since the `mcp<2` ceiling in `pyproject.toml` is deliberate and currently
  protective.

## Root cause

The MCP Python SDK's 2.0.0 release renames modules and fields. Three sites in this
integration break under it, and the third does not raise.

`client.py:21` imports `ProgressFnT` from `mcp.shared.session`. In 2.0.0 `mcp.shared` has
no `session` submodule and the symbol lives in `mcp.client.session`, so this is the first
failure of the package under 2.0.0, ahead of everything the report lists.

`utils.py:4` imports `FastMCP` and `Context` from `mcp.server.fastmcp`. That module does
not exist in 2.0.0 and `hasattr(mcp.server, "FastMCP")` is `False`. The nearest
replacement, `mcp.server.mcpserver.MCPServer`, takes no `host` or `port` in its
`__init__`, so this is a port rather than a re-verification. Because `utils.py` is
re-exported from the package `__init__`, it also blocks `import llama_index.tools.mcp`
for client-only users who never touch the server helper.

`base.py:86` is the silent one:

```python
dynamic_response.resourceTemplates if hasattr(dynamic_response, "resourceTemplates") else []
```

`ListResourceTemplatesResult.resourceTemplates` is `resource_templates` in 2.0.0. The
read is guarded, so it does not raise: `hasattr` is `False`, the expression yields `[]`,
and `McpToolSpec.fetch_resources()` returns the static resources only. Every resource
template disappears from the result with no exception, no log line and no partial output
to notice.

The guard is not a workaround somebody left behind. `git log -G 'resourceTemplates'
--follow` over the file returns a single commit, `69d27c22b` (#19307, 2025-07-03), and
reading its hunks shows the guard arrived *with* the change that added resource-template
support. It is defensive coding against an SDK too old to have the field, written by the
person adding the feature. The expression one line above reads
`static_response.resources` behind the same `hasattr` shape, so this is the file's idiom
for touching SDK result objects rather than a single cautious line.

## Invariant violated

**A guarded read of a field you do not own converts a rename into missing data, and
missing data is the one breakage a migration cannot be driven by.** A strict read fails at
the exact line that has to change, so the interpreter hands the porter a work list for
free: fix the traceback, run again, fix the next one, stop when it imports. A guarded read
contributes nothing to that list. It returns its default, the call succeeds, and the
feature it fed is absent from the output. So across a major-version bump, the guards act
as a filter, and what they select for is precisely the set of breaks that will survive the
port and ship.

**The guard cannot distinguish the case it was written for from the case that hurts,
because both present as the absence of an attribute.** `hasattr(x, "resourceTemplates")`
answers "does this object have that name", and both "this SDK predates resource templates"
and "this SDK renamed resource templates" answer `False`. The first is the intended
tolerance and degrading to `[]` is right. The second is data loss. Nothing in the
expression separates them, and no amount of care at the call site can, since the
information needed is the dependency's version and the guard never looks at it. A version
check would have raised the question at the right moment; the attribute check quietly
answers it forever.

**Neither break is reachable today, and that is what makes the pair worth recording rather
than fixing.** The `mcp<2` pin holds, so nothing is broken for installs, no test in the
project can reach any of this, and CI is green on all three sites. The day the pin is
lifted, two of them stop the build immediately and one does not. The asymmetry is fixed at
the moment the guard is written, years before the version bump that exercises it, which is
why the useful habit is not "audit before migrating" but "prefer the read that raises,
unless the fallback is a value the caller can act on".

## Trigger

`mcp>=2` resolved into the environment. Not reachable through the declared dependency set
today, since the integration's `<2` ceiling holds; reachable the moment it is raised,
which is what the issue asks for. The first two sites break every user of the package.
The third affects users of `McpToolSpec.fetch_resources()` on servers that publish
resource templates, and they see a shorter list rather than an error.

## Repro

Clean `python:3.12-slim`, against `main` `c864fcfa2c1d1f987ccdbcdab7b18e395c01ba86`, on a
sparse checkout of the integration only (`git rev-parse --is-shallow-repository` = false,
so the history gate above is reading real history). `llama-index-core` was installed
first, then the integration with `--no-deps`, so that the `mcp` specifier is the only
thing that differs between runs:

```
mcp 1.29.0 (what the current pin resolves to)
  import llama_index.tools.mcp: OK

mcp 2.0.0
  client.py:21  ModuleNotFoundError: No module named 'mcp.shared.session'

mcp 2.0.0, after redirecting that one import to mcp.client.session
  utils.py:4    ModuleNotFoundError: No module named 'mcp.server.fastmcp'
```

The silent site was measured the same way, evaluating the `base.py:86` expression against
a `ListResourceTemplatesResult` carrying one template, built from each SDK's own model:

```
mcp 2.0.0 : hasattr resourceTemplates False -> []
mcp 1.29.0: hasattr resourceTemplates True  -> ['doc']
```

**Not verified:** no MCP server was driven over the wire at 2.0.0; the result objects were
constructed from each SDK's own types. The package was not ported and its suite was not
run, so this covers these three sites and makes no claim about the total size of the
migration. `tests/server.py:9-10` and `examples/mcp_server.py:2` import
`mcp.server.fastmcp` as well and would need the same port before the suite could run at
all.

Verified at `main` `c864fcfa`, read 2026-07-28. Re-checked 2026-07-31: `main` is still
`c864fcfa`, the guard at `base.py:86` is unchanged and `pyproject.toml:44` still declares
`mcp>=1.24.0,<2`. Reported, not fixed upstream at the time of writing.

## Update, 2026-08-01

An outside contributor opened
[PR #22535](https://github.com/run-llama/llama_index/pull/22535), which is open and not
merged, so nothing has changed in the shipped package and the guard still returns an empty
list under `mcp>=2`. It supports 2.x alongside 1.x rather than bumping the floor, isolating
the differences in a `_compat` module, and it replaces the `hasattr` at `base.py:86` with a
read that tries both spellings of the field. Its comment on that hunk states the reason as
template resources otherwise being silently dropped, which is the same defect this entry
describes, reached independently.

The diff also names a fourth break this entry did not: `Tool.inputSchema` becomes
`input_schema`, read at `base.py:139`. That read is unguarded, so it raises rather than
degrading, and it is therefore the kind of break a port is driven by. The split is the
entry's own point restated by the fix. Of the two field renames in the same file, the
unguarded one announced itself and the guarded one had to be found by reading, and only the
guarded one had been silently returning nothing to users the whole time.
