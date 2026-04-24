# ADR-0008: Contextual retrieval

**Status:** accepted
**Date:** 2026-04-23

## Context

Insurance (and most long-form technical) corpora exhibit severe
vocabulary mismatch between chunk text and user queries. A chunk describing
a TCP connection state transition might say "the server transitions from
LISTEN to SYN-RCVD"; the user types "explain the three-way handshake" and
retrieves nothing, because the chunk never uses the phrase "handshake".

Hybrid retrieval (ADR-0001) and query expansion help, but the root cause
is that each chunk, once sliced from its parent document, loses the
situational context that a human reader would carry in their head: which
chapter this is from, what the running topic is, what acronyms have been
defined. A chunk-level BM25 or dense index cannot recover that context
from the chunk text alone.

Anthropic published "Contextual Retrieval" in September 2024 showing that
prepending a 1-2 sentence LLM-generated "situating context" to each chunk
before embedding/indexing reduces retrieval failures by 35-49%.

## Decision

At ingest time, for each child chunk we call an LLM with a prompt of the
form:

    <document>{full doc}</document>
    Here is the chunk we want to situate within the whole document:
    <chunk>{chunk text}</chunk>
    Please give a short succinct context to situate this chunk within
    the overall document for the purposes of improving search retrieval
    of the chunk. Answer only with the succinct context and nothing else.

The returned blurb is prepended to the chunk text (`"<blurb>\n\n<chunk>"`)
and that *contextualized text* is what we embed and BM25-index. The raw
chunk text is preserved on the stored payload so citations, PII
redaction, and LLM-prompt context at query time all continue to use the
original wording.

The implementation is in `aegisrag/contextualizer.py`, wired into the
ingest pipeline in `aegisrag/ingest.py`. It is feature-flagged behind
`settings.enable_contextual_retrieval` and is on by default.

**Switchable provider:**

- **`anthropic`** (default): Claude Haiku with prompt caching on the
  `<document>` block. Cost scales as O(docs) not O(chunks) because every
  chunk of the same document reuses the cached prefix. Typical cost for
  a 900-page book: ~$5.
- **`ollama`**: fully local fallback. No caching; re-prefills per chunk.
  Typical runtime for a 900-page book on M4 Pro with qwen2.5:3b: ~30 min.
  Zero-cost, zero-egress, acceptable for air-gapped deployments.

Contextualization is **fail-soft**: any exception returns an empty
blurb, and the chunk is indexed without context. A failed contextualizer
call never fails the whole ingest batch.

## Consequences

+ Expected +15-30% recall@5 on our golden set (empirical validation
  pending; see `eval/report_*.md`).
+ Fits the existing compliance story: the blurb is machine-generated,
  deterministic at `temperature=0`, and auditable (the exact prompt and
  response can be logged per chunk if required).
+ Retrieval code path is unchanged — the improvement is entirely in what
  gets embedded / BM25-tokenized. Smallest possible blast radius.
+ The same LLM infrastructure serves both ingest-time contextualization
  and query-time generation; no new deps.
- Adds an LLM call per chunk at ingest time. For Haiku + prompt caching
  this is trivial; for Ollama it makes ingest ~5-10× slower.
- Re-ingesting a corpus with the provider/model changed produces different
  blurbs and therefore different embeddings; the `contextualized` boolean
  on each payload records which regime produced the entry.
- Fail-soft behavior means a partial outage of the contextualization
  provider produces a mix of contextualized and non-contextualized
  chunks in the same index, which is acceptable but worth knowing.

## Alternatives considered

- **Skip contextual retrieval, invest in reranking.** A stronger cross-
  encoder (Cohere Rerank 3) would help, but it acts on whatever the
  retriever surfaces; if the contextless chunk never makes it to top-k
  the reranker cannot save it. Contextual retrieval fixes the recall
  ceiling first; reranking fixes precision within it. Both are useful;
  contextual retrieval is the higher-leverage change and runs first.
- **HyDE (Hypothetical Document Embeddings).** Generates a fake answer
  at *query time* and embeds that. Moves LLM cost to the query path,
  which is latency-critical for us. Contextual retrieval moves it to
  ingest, which is batch. Compatible, not mutually exclusive.
- **Pre-generated synthetic questions** per chunk, added as extra
  retrieval keys. Similar spirit; covers the complementary failure mode
  (query-phrasing variance rather than chunk-phrasing sparsity). Noted
  as a future ADR if measurements justify stacking both.
- **Run contextualization at query time** on retrieved candidates. Adds
  significant query latency for a one-time-per-chunk cost. Rejected.

## Revisions

### 2026-04-24 — Parent-chunk context on the self-hosted path

First real contextual ingest on the Tanenbaum corpus (900 pages,
qwen2.5:3b on M4 Pro GPU) projected to ~45 minutes. Root cause: on the
Ollama path, every chunk re-prefills the entire `<document>` block because
no prompt cache exists. At ~25k tokens of prefill per chunk × ~1000
chunks, prefill — not decode — dominates the wall clock.

Anthropic's recipe implicitly depends on prompt caching — it's the only
reason sending the full doc each time is cheap. Take caching away and the
full-doc premise collapses.

**Refinement.** The self-hosted (Ollama) path now passes the child's
**parent chunk** (~1500 tokens, ~8k chars) as the situating-context block
rather than the full document. Parents are, by construction, the smallest
context that still carries the running narrative around the child. The
Anthropic path is unchanged — it still sends the full doc and relies on
`cache_control: ephemeral` for amortization.

Implementation:

- `contextualize(chunk, *, parent, doc)` takes both; each adapter picks
  its preferred context (anthropic → doc, ollama → parent) and falls
  back to the other if empty.
- `ingest.py` builds `parent_by_id` once per document and passes each
  child its enclosing parent's text.
- Ingest is now interleaved (contextualize batch → embed → upsert →
  progress echo → repeat) rather than "contextualize all, then upsert
  all", so operators see per-batch progress during long runs and memory
  stays bounded.
- New knob `settings.contextualization_concurrency` (default 4) replaces
  the hardcoded 2-or-4 split; pair with `OLLAMA_NUM_PARALLEL=4` on the
  Ollama server for a further ~2× speedup.
- `ingest.py` now exposes `--reset` to drop the Qdrant collection and
  OpenSearch index before ingest — needed between baseline/contextual
  eval passes since the two regimes produce incompatible embeddings.

Expected effect: ~5-10× faster Ollama contextualization at comparable
blurb quality (parent chunks already contain the section's running
topic, which is most of what the blurb needs to express). Quality to be
re-measured against the golden set; if parent-only context costs
>2 points of recall@5 we revisit.

## References

- Anthropic, "Contextual Retrieval," Sept 2024.
  https://www.anthropic.com/news/contextual-retrieval
- ADR-0001: Hybrid retrieval with RRF.
- ADR-0002: Parent-child chunking.
- ADR-0006: LLM provider adapters (the switchable-provider pattern is
  reused verbatim for contextualization).
