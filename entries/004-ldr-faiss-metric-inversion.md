# FAISS relevance mapping inverts ranking under cosine/inner-product metrics

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `CollectionSearchEngine.search()` relevance_score mapping
- **Class:** retrieval & scoring math
- **Fix:** [PR #5062](https://github.com/LearningCircuit/local-deep-research/pull/5062) (merged; issue [#4963](https://github.com/LearningCircuit/local-deep-research/issues/4963), owner-filed)

## Root cause

`relevance_score = 1/(1+score)` assumes `score` is a distance (smaller =
nearer). `FAISS.similarity_search_with_score` returns whatever the index
metric produces: with the default `cosine` (an `IndexFlatIP`), the score is an
inner product in `[-1, 1]` where **larger = nearer**. The mapping therefore
inverted the entire ranking (best match → 0.5, unrelated → 1.0) and divided
by zero on an anti-correlated pair (score = -1).

## Invariant violated

A score-to-relevance mapping must branch on the metric that produced the
score. There is no metric-agnostic formula; the index (`index.metric_type`)
is the ground truth.

## Trigger

Default configuration (cosine metric) — i.e., every user of collection
search. The bug made results *worse than random* (anti-ranked), which is the
kind of failure that reads as "the RAG isn't finding anything."

## Repro

Clean Docker, faiss-cpu: 3 tests fail on the old mapping (including
`ZeroDivisionError` on the anti-correlated vector), pass on the fix; the L2
test passes on both, confirming the branch is metric-correct.
