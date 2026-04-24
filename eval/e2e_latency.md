# End-to-end latency

- Golden set: `eval/golden_set.jsonl`  (52 queries)
- API: `http://localhost:8000`
- Tenant: `demo`  Concurrency: 1
- OK: 52  Cache hits: 0  Misses: 52  Errors: 0

All times in seconds.

## Summary (TTLT — total time to complete answer)

| Bucket | n | mean | p50 | p95 |
|---|---:|---:|---:|---:|
| all | 52 | 3.841 | 2.765 | 10.586 |
| cache miss | 52 | 3.841 | 2.765 | 10.586 |
| cache hit | 0 | — | — | — |

## Time to first token (TTFT — LLM starts streaming)

| Bucket | n | mean | p50 | p95 |
|---|---:|---:|---:|---:|
| all | 52 | 2.585 | 1.713 | 6.833 |
| cache miss | 52 | 2.585 | 1.713 | 6.833 |
| cache hit | 0 | — | — | — |

## Time to first event (TTFE — first SSE byte from server)

| Bucket | n | mean | p50 | p95 |
|---|---:|---:|---:|---:|
| all | 52 | 1.368 | 1.016 | 2.198 |
