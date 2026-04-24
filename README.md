# aegisrag-notes

Public writeup, eval reports, and architecture decision records from
**AegisRAG** — a compliance-grade, hybrid-retrieval RAG engine I've been
building as a side project.

## What's in here

```
docs/
  blog/
    contextual-retrieval-local-model.md   — the Substack post, preserved here so it travels with the numbers
  adr/
    0001-hybrid-retrieval-with-rrf.md      — Qdrant dense + OpenSearch BM25 fused with RRF (k=60)
    0002-parent-child-chunking.md          — small-to-big retrieval with 1500/300-token parents/children
    0006-llm-provider-adapters.md          — switchable Anthropic / Ollama / OpenAI LLM backends
    0008-contextual-retrieval.md           — Anthropic-style contextual retrieval, adapted for Ollama
eval/
  golden_set.jsonl                         — 52 selector-based gold labels over Tanenbaum's Computer Networks
  report_baseline.md                       — baseline retrieval run (no contextual blurbs)
  report_contextual-ollama.md              — same corpus + golden set, contextual retrieval on via qwen2.5:3b
  comparison.md                            — baseline → contextual delta table
  e2e_latency.md                           — live /query SSE timings (TTFE / TTFT / TTLT)
```

## Headline numbers

Hybrid retrieval baseline vs. the same stack with Anthropic-style
**contextual retrieval** (Ollama path, `qwen2.5:3b` on M4 Pro, parent-chunk
context — see ADR-0008 for why parent instead of full doc):

| metric | baseline | contextual | Δ |
| --- | ---: | ---: | ---: |
| recall@5 | 0.856 | 0.894 | **+4.5%** |
| MRR@10 | 0.865 | 0.904 | +4.4% |
| recall@20 | 0.894 | 0.894 | +0% |
| retrieval latency p50 (s) | 1.08 | 1.36 | +26% |

End-to-end latency measured against the live `/query` SSE endpoint, 52
queries, concurrency=1, all cache misses:

| metric | p50 (s) | p95 (s) |
| --- | ---: | ---: |
| TTFE (first SSE event) | 1.02 | 2.20 |
| TTFT (first LLM token) | 1.71 | 6.83 |
| TTLT (done) | 2.77 | 10.59 |

All numbers are **single-run** on a laptop — variance is real. The blog
post explains why the gain is smaller than the 35-49% Anthropic reported
and what I plan to re-measure next.

## Why these numbers, not bigger ones

The engine isn't tuned to a benchmark. The golden set is 52
hand-authored queries against one book (~900 pages of Tanenbaum's
*Computer Networks*). The labels are **selectors** (`text_contains`,
`section_contains`, `page_range`) rather than chunk IDs, so they survive
re-ingestion — which is what made the before/after comparison tractable.

The selector format is visible in
[`eval/golden_set.jsonl`](eval/golden_set.jsonl). If you want to see
what a workable, boring golden set looks like before worrying about
sophisticated eval harnesses, that file is probably the most useful
thing in this repo.

## About AegisRAG

The engine combines hybrid retrieval (dense + BM25 + RRF), parent-child
chunking, optional contextual retrieval at ingest, streaming SSE
responses, tenant isolation, Presidio-based PII redaction, and an audit
log with replay. Switchable LLM providers (Anthropic, OpenAI, Ollama)
so the same stack runs against a paid API or fully local. Nine ADRs,
four of which are checked in here, document the reasoning behind each
decision.

It is a side project — not production-hardened, not scale-tested. The
interesting claims are the ones measured in `eval/`; everything else is
infrastructure-honest.


## Contact

vkrishnasamy5@gmail.com. Happy to talk about anything in here.
