# agent-failure-modes

A citable corpus of verified failure modes in LLM agent infrastructure,
maintained by Ehsan Barkhordar. Every entry is a real defect in a widely used
open source agent framework, reproduced in a clean Docker container against the
repo's HEAD before it was reported, then traced to its root cause and the
invariant it violates. Most entries link to an upstream fix.

The pattern that emerges: agent systems rarely fail in the model. They fail in
the plumbing around it, at message-conversion boundaries, in token accounting,
retrieval score math, text decomposition contracts, and template rendering.
Small correctness bugs in that plumbing silently corrupt agent behavior in ways
that look like "the model being dumb."

## Taxonomy

| Class | What breaks | n | Entries |
|---|---|---|---|
| Message-conversion boundaries | Content silently dropped or malformed when translating between provider message formats | 5 | [001](entries/001-hermes-image-only-dropped.md), [002](entries/002-swarms-empty-text-block.md), [003](entries/003-camel-sharegpt-toolcall-args.md), [023](entries/023-camel-cohere-parallel-toolcalls.md), [030](entries/030-sagemaker-loop-variable-rebind.md) |
| Text decomposition contracts | Two line/character decompositions disagree; positions desync; a regex cleanup pass deletes text the parser would have called code | 5 | [007](entries/007-serena-splitlines-desync.md), [008](entries/008-serena-glob-question.md), [020](entries/020-serena-delete-last-line-no-newline.md), [022](entries/022-serena-read-file-unmigrated-splitter.md), [033](entries/033-hindsight-separator-strip-before-fence-detection.md) |
| Streaming & usage accounting | Streamed deltas and token counts wrong, doubled, or incomplete | 3 | [005](entries/005-sambanova-stale-delta.md), [006](entries/006-anthropic-cache-tokens-lost.md), [031](entries/031-anthropic-streaming-cumulative-usage-summed.md) |
| Template & token rendering | Tokens valid in one context crash in another, raw templates shown as rendered values, or template syntax mis-parsed by a naive splitter | 3 | [009](entries/009-changedetection-restock-token.md), [015](entries/015-changedetection-jinja-url-raw.md), [025](entries/025-changedetection-import-jinja-split.md) |
| Initialization & control flow | Variables read where never bound; guard branches left unreachable | 2 | [011](entries/011-cohere-unbound-documents.md), [017](entries/017-gpt-researcher-optional-str-sentinel.md) |
| Cache-key & hashing collisions | Distinct inputs serialize to one key; caches return the wrong entry, or dedup discards a distinct record | 2 | [013](entries/013-lightrag-argshash-collision.md), [024](entries/024-ldr-benchmark-hash-omits-identity.md) |
| Indexing, ordering & counting contracts | Producer and consumers disagree on the index base or the ordering; a fixed index reads the wrong end; an enumeration yields an order-dependent subset of the pairs it owes | 3 | [016](entries/016-changedetection-browser-step-offbyone.md), [029](entries/029-changedetection-previous-price-reversed-index.md), [032](entries/032-hindsight-cooccurrence-swap-rebinds-outer-iterate.md) |
| Concurrency & context propagation | Work offloaded to a thread pool loses the request's ambient context; results silently drop | 2 | [018](entries/018-ldr-flask-context-parallel-search.md), [021](entries/021-ldr-progressive-explorer-context.md) |
| Retrieval & scoring math | Similarity scores misinterpreted, rankings inverted | 1 | [004](entries/004-ldr-faiss-metric-inversion.md) |
| Path & name handling | Names escape their intended directory | 1 | [010](entries/010-serena-memory-path-escape.md) |
| Resource liveness & reconnection | A cached handle to a cullable resource goes stale; the recovery path fails to recover | 1 | [012](entries/012-jupyter-mcp-kernel-liveness.md) |
| Schema traversal completeness | A recursive walker skips a valid branch of the tree | 1 | [014](entries/014-pydantic-ai-schema-composition-recursion.md) |
| Error handling & success reporting | A swallowed failure is reported as success; a status flag gates the recovery path | 1 | [019](entries/019-ldr-indexed-without-embeddings.md) |
| Concurrency & atomic claims | A read of shared state is mistaken for a claim on it; concurrent workers act on the same rows | 1 | [026](entries/026-ldr-download-queue-no-atomic-claim.md) |
| Tool side-effect scope | A tool reaches past what its caller asked for; a capability flag is read as consent to use the capability | 1 | [027](entries/027-jupyter-mcp-use-notebook-ui-focus-steal.md) |
| Configuration wiring & documented contracts | A documented setting reaches no code that reads it; a sentinel's documented meaning is never implemented | 1 | [028](entries/028-jupyter-mcp-execution-timeout-unwired.md) |

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
