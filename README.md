# agent-failure-modes

A curated corpus of verified failure modes in LLM agent infrastructure.

I study how LLM-based agents break under realistic conditions. This repo is the
empirical side of that work: every entry is a real bug found in a widely used
open source agent framework, reproduced in a clean Docker container against the
repo's HEAD before it was reported, and traced to a root cause and the invariant
it violates. Most entries link to an upstream fix.

The pattern that emerges: agent systems rarely fail in the model. They fail in
the plumbing around it — message conversion boundaries, token accounting,
retrieval score math, text decomposition contracts, template rendering. Small
correctness bugs in that plumbing silently corrupt agent behavior in ways that
look like "the model being dumb."

## Taxonomy

| Class | What breaks | Entries |
|---|---|---|
| Message-conversion boundaries | Content silently dropped or malformed when translating between provider message formats | [001](entries/001-hermes-image-only-dropped.md), [002](entries/002-swarms-empty-text-block.md), [003](entries/003-camel-sharegpt-toolcall-args.md) |
| Retrieval & scoring math | Similarity scores misinterpreted, rankings inverted | [004](entries/004-ldr-faiss-metric-inversion.md) |
| Streaming & usage accounting | Streamed deltas and token counts wrong or incomplete | [005](entries/005-sambanova-stale-delta.md), [006](entries/006-anthropic-cache-tokens-lost.md) |
| Text decomposition contracts | Two line/character decompositions disagree; positions desync | [007](entries/007-serena-splitlines-desync.md), [008](entries/008-serena-glob-question.md) |
| Template & token rendering | Tokens valid in one context crash in another, or raw templates shown as rendered values | [009](entries/009-changedetection-restock-token.md), [015](entries/015-changedetection-jinja-url-raw.md) |
| Path & name handling | Names escape their intended directory | [010](entries/010-serena-memory-path-escape.md) |
| Initialization & control flow | Variables read where never bound; guard branches left unreachable | [011](entries/011-cohere-unbound-documents.md), [017](entries/017-gpt-researcher-optional-str-sentinel.md) |
| Resource liveness & reconnection | A cached handle to a cullable resource goes stale; the recovery path fails to recover | [012](entries/012-jupyter-mcp-kernel-liveness.md) |
| Cache-key & hashing collisions | Distinct inputs serialize to one key; caches return the wrong entry | [013](entries/013-lightrag-argshash-collision.md) |
| Schema traversal completeness | A recursive walker skips a valid branch of the tree | [014](entries/014-pydantic-ai-schema-composition-recursion.md) |
| Indexing & counting contracts | Producer and consumers disagree on the index base; values off by one | [016](entries/016-changedetection-browser-step-offbyone.md) |

## Entry format

Each entry records: the repo and surface, the root cause at file level, the
**invariant violated** (the general rule, which is the reusable part), the
trigger conditions, a minimal reproduction, and the fix.

## Method

Every claim here was executed: reproduced in a clean container against the
then-current HEAD of the target repo, with the repro included in the entry or
the linked PR. Nothing is speculative; where a root cause is inferred rather
than stepped through, the entry says so. Maintained with AI assistance; every
entry is verified before it lands.

## License

Text is CC BY 4.0. Code snippets in entries are under the license of the repo
they came from, linked per entry.
