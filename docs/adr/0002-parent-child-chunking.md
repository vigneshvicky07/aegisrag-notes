# ADR-0002: Parent-child hierarchical chunking

**Status:** accepted
**Date:** 2026-04-18

## Context

Flat 512-token chunking is the standard RAG baseline. It is simple, fast to
implement, and measurably worse than hierarchical chunking on long, structured
documents like insurance policies. The tradeoff space is:

* small chunks embed crisply but lack context for the LLM,
* large chunks ground the LLM but dilute embedding relevance.

## Decision

We chunk at two granularities:

* **Parent:** section-delimited (by headings) or a fixed 2,048-token window
  with 256-token overlap. Parents are what the LLM reads.
* **Child:** 256-token sliding windows with 64-token overlap within a parent.
  Children are what the embedder indexes.

At query time, vector search returns child chunks; before reranking, we
expand each child to its parent; the LLM sees parents.

## Consequences

+ Recall@10 improved ~12 points and RAGAS faithfulness ~8 points vs flat 512.
+ Child embeddings stay semantically focused.
- Storage overhead: children + parent metadata. Net ~1.2× the flat scheme.
- Chunker complexity is higher; more edge cases (nested tables, short docs).

## Alternatives considered

- **Flat 512-token chunks.** Baseline; worse on all RAGAS metrics.
- **Sentence-level chunks with hierarchical merging** (LlamaIndex
  `SentenceWindowNodeParser`). Similar in spirit; more moving parts.
- **Recursive character splitter** (LangChain default). Works, but loses
  heading structure.

## References

- LlamaIndex: "Advanced RAG 01 — Small-to-Big Retrieval."
- Internal golden-set benchmarks (see `eval/reports/chunking-ablation.json`).
