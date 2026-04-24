# What I Learned Trying to Reproduce Anthropic's Contextual Retrieval on a Free Local Model

Last September Anthropic put out a post about a technique called Contextual Retrieval. One small change to your ingest pipeline, they said, could cut retrieval failures by 35 to 49 percent. The change: before embedding a chunk, ask an LLM to generate a one or two sentence blurb that situates the chunk inside its parent document, and prepend that blurb to the chunk text. That's it. The recipe fits on a napkin. The numbers are the kind you stop and read twice.

I spent the last few days trying to reproduce this on my own RAG engine (AegisRAG, a side project), running fully local with qwen2.5:3b on an M4 Pro laptop. My recall@5 went from 0.856 to 0.894. So, +4.5%.

Not 35. Not 49. About a fifth of what the paper promised.

But the interesting part isn't how much smaller my number is. It's what I had to build to make sure the number meant anything at all.

The engine is private, but the reasoning trail and every eval report are here: [github.com/vigneshvicky07/aegisrag-notes](https://github.com/vigneshvicky07/aegisrag-notes). The numbers in this post are all reproducible from the files in that repo.

## The setup

AegisRAG is a pretty standard hybrid retrieval stack. Qdrant for dense vectors, OpenSearch for BM25, Reciprocal Rank Fusion on top. Parent-child chunking so I can search on small chunks but return the surrounding context. Streaming SSE API, tenant isolation, PII redaction at ingest time. The corpus I evaluated on is Tanenbaum's *Computer Networks*, roughly 900 pages, about 1000 child chunks once indexed.

The golden set is 52 queries I wrote myself after re-reading the book. One thing that made all the rest of this possible: I wrote the labels as **selectors** instead of chunk IDs. Each gold label is a small rule like `text_contains: ["SYN", "three-way"]` or `section_contains: "TCP"`. Chunk IDs get regenerated every time you re-ingest. Selectors don't. That matters a lot when the whole point of the experiment is comparing a "before" index to an "after" index with different chunk boundaries and different embeddings. If I'd used chunk IDs I would have had to relabel 52 queries every time. I did not want to do that.

## The first surprise: the recipe secretly assumes you have prompt caching

If you read Anthropic's post literally, the recipe says: wrap the entire document in `<document>` tags and send it with every single chunk. For my 900-page book, that means ~25k tokens of prefill per chunk, times 1000 chunks. A lot of tokens.

On the Claude API this is fine, because Anthropic's prompt caching means you only pay for the prefix once per 5-minute window. Every chunk after the first costs about 10% of a full-price call. The whole economics only works because caching exists.

Ollama doesn't have prompt caching. Every single call re-processes the full document from scratch. My first run projected out to something like 45 minutes, and when I looked at what was actually taking the time, it was prefill. The model spent most of its life re-reading the book, not generating the blurbs.

Which is the part the paper kind of glosses over. "Send the full doc with every chunk" is a recipe that assumes infrastructure most people don't have. Without prompt caching, the full-doc premise just falls apart.

## The fix: use the parent chunk instead of the full document

Once I understood what was actually slow, the fix took about fifteen minutes to reason through. My engine already does parent-child chunking. Every child chunk already knows its enclosing parent, which is around 1500 tokens. The parent is, by definition, the smallest piece of context that still carries the running topic of the section. Which is mostly what the situating blurb is trying to capture anyway.

So on the Ollama path, I pass the parent chunk instead of the full book. The Anthropic path is unchanged: it still sends the whole document and relies on `cache_control: ephemeral` to amortize the cost. Two lines of code decide which context each provider prefers:

```python
if provider == "anthropic":
    context = doc or parent   # cacheable full-doc prefix
elif provider == "ollama":
    context = parent or doc   # small prefill, no caching
```

Ingest time dropped from a projected 45 minutes to about 6. Same model, same corpus, same prompts. Just a smaller context block per call.

I also restructured the pipeline to interleave contextualize → embed → upsert per batch, instead of doing all the contextualization first and then all the indexing. Not for speed. It's so I could actually see `... 128/1000 chunks indexed` ticking by during a long run, instead of staring at a frozen terminal for half an hour wondering if Ollama had crashed.

## The numbers

Same corpus, same golden set, same retrieval code. Two different ingest regimes.

| metric | baseline | contextual | Δ |
| --- | --- | --- | --- |
| recall@5 | 0.856 | 0.894 | **+4.5%** |
| MRR@10 | 0.865 | 0.904 | +4.4% |
| recall@20 | 0.894 | 0.894 | 0% |
| retrieval latency p50 | 1.08s | 1.36s | +26% |

And the end-to-end latency against the live `/query` endpoint, for the whole streaming path (52 queries, all cache misses):

| metric | p50 | p95 |
| --- | --- | --- |
| first SSE event | 1.02s | 2.20s |
| first LLM token | 1.71s | 6.83s |
| answer complete | 2.77s | 10.59s |

So, +4.5% recall@5. A nice bump. Much smaller than 35-49%.

## Why didn't I get the paper's number?

A few things, and I think they stack:

The first is kind of embarrassing: my golden set is probably too friendly to BM25. When you write selectors by skimming a book you know well, you reach for phrases that appear in the text verbatim. "Nagle algorithm". "sliding window". BM25 nails those already. Where contextual retrieval actually earns its keep is on queries where the chunk doesn't contain the user's words. Someone asks about "the three-way handshake" and the only chunk that knows the answer says "LISTEN → SYN-RCVD → ESTABLISHED". Those are the queries where the blurb pulls its weight. I don't have many of those. The paper's benchmarks (CodeSearchNet, FiQA) are full of them.

The second is model size. qwen2.5:3b is a solid 3-billion-parameter model, but it's not Claude Haiku. A lot of the blurbs it generates are fine. Some of them are generic ("This chunk discusses networking concepts.") and those add roughly no retrieval signal at all.

The third is the parent-context tradeoff I made. I gave up whole-book context in exchange for 7x faster ingest. The parent tells the model what section we're in. The full doc would tell it what section, what chapter, how the topic relates to the rest of the book. You can imagine the difference: "This is about Nagle's algorithm" vs "This is about Nagle's algorithm as a small-packet optimization introduced in the TCP flow control chapter." The second one is better retrieval context. I don't get it on the Ollama path.

And the fourth is just that n=52. Small sets are noisy. My +4.5% could easily be anywhere between +2% and +7% on a different run. I haven't run the eval three times and reported the median yet. I probably should.

## The thing that actually mattered

The +4.5% is a data point. But the part I actually care about is the scaffolding the experiment forced me to build:

- Selector-based gold labels, so re-ingesting didn't mean relabeling.
- A `--reset` flag on the ingest CLI so baseline and contextual runs didn't contaminate each other.
- Per-batch progress output so a long ingest is visibly alive.
- A real end-to-end latency harness that separates time-to-first-event from time-to-first-token from time-to-done. Retrieval-only latency is a misleading number, because users don't see retrieval, they see TTFT.
- An ADR for every non-trivial decision. I'm up to eight now. In three months when I come back to this I won't have to reconstruct why I chose parent-only context on Ollama; the file exists.

None of those things is interesting to tweet about. All of them are what separates this from posting "my recall went up 40%!" without saying on what corpus, with what queries, against what baseline.

Reproducing a paper on your own data almost always goes like this. The technique is easy. The measurement scaffolding is hard. And if you don't build the scaffolding first, you can't actually tell whether your technique worked or whether your test set just happened to be friendly to it.

## What AegisRAG is, and isn't

Since I keep mentioning it: AegisRAG is a side project. It's a RAG engine I've been building to understand what a production-grade stack actually needs. It runs on a laptop. It's not battle-tested at scale. It does have hybrid retrieval, contextual retrieval on both Anthropic and Ollama paths, parent-child chunking, tenant isolation, PII redaction, streaming SSE with proper latency instrumentation, a golden-set eval harness, and nine ADRs explaining the decisions. It does not have the kind of throughput numbers a real production system would claim, and I've tried to keep the repo honest about what I actually measured vs. what I'm projecting.

If you're building your own RAG and some of this is useful, the ADRs in [docs/adr/](https://github.com/vigneshvicky07/aegisrag-notes/tree/main/docs/adr) of the public notes repo are probably more useful than any code. They're where I actually did the thinking.
