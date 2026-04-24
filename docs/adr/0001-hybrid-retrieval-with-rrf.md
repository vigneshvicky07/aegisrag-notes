# ADR-0001: Hybrid retrieval with Reciprocal Rank Fusion

**Status:** accepted
**Date:** 2026-04-18

## Context

Insurance queries alternate between semantic paraphrase ("does this cover a
slip and fall?") and exact-lexical lookup ("endorsement CG 21 47"). Dense
retrieval handles the first; BM25 handles the second. Using only one leaves
recall on the floor.

## Decision

We use both dense (Qdrant HNSW) and sparse (OpenSearch BM25) retrievers in
parallel, and fuse the rankings with Reciprocal Rank Fusion (Cormack et al.,
SIGIR 2009) with `k=60`. No score normalization; ranks only.

## Consequences

+ Recall@10 on our insurance golden set improved by ~9 points over dense-only.
+ RRF is rank-based, so changes to either retriever's score distribution don't
  destabilize fusion.
- Two systems to operate (Qdrant + OpenSearch) instead of one.
- At 100M+ scale, coordinating shard layout across both adds complexity.

## Alternatives considered

- **Convex-combination fusion** (weighted average of normalized scores):
  fragile to score-distribution drift, requires per-query calibration.
- **Cascade** (BM25 first, then rerank with dense): misses semantic matches
  that lack lexical overlap.
- **Single-index hybrid** (OpenSearch with k-NN plus BM25): viable, but ties
  us to OpenSearch internals and loses Qdrant's HNSW quality.

## References

- Cormack, Clarke, Büttcher (2009). "Reciprocal Rank Fusion outperforms
  Condorcet and individual rank learning methods." SIGIR '09.
- BEIR benchmark: Thakur et al., NeurIPS 2021 Datasets & Benchmarks.
