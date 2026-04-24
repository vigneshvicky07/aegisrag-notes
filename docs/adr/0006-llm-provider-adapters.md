# ADR-0006: LLM provider adapters, no framework lock-in

**Status:** accepted
**Date:** 2026-04-19

## Context

LLM vendors change pricing, models, and APIs on a quarterly cadence.
A carrier's LLM choice is also driven by procurement, data residency,
and contract terms — often not a technical decision. Meanwhile, RAG
frameworks like LangChain and LlamaIndex wrap LLMs but change their
abstraction layers frequently.

## Decision

Define a tiny `LLMProvider` protocol with exactly the methods we need
(`stream`, `count_tokens`) and ship three reference implementations:
Anthropic, OpenAI, vLLM. No framework dependency on the critical path.

## Consequences

+ Each adapter is ~100 LOC. Reviewable end-to-end.
+ A new provider is a drop-in file. No framework upgrade churn.
+ The vLLM adapter lets carriers run fully on-prem with Llama 3.1 or
  similar open-weight models.
- We reimplement provider-specific optimizations (Anthropic prompt
  caching, OpenAI logprobs if needed) ourselves.
- We do not get the ecosystem benefits of LangChain (tools, agents,
  loaders) for free.

## Alternatives considered

- **LangChain as the core.** Strong ecosystem, rapid API churn. Rejected
  for the critical path; users can still wrap AegisRAG in a LangChain
  `Retriever`.
- **LiteLLM as the adapter layer.** Good option; adds a dependency we
  don't strictly need. Keeping the door open via
  `LLM_PROVIDER=litellm` is plausible future work.

## References

- Anthropic Prompt Caching: https://docs.claude.com/en/docs/build-with-claude/prompt-caching
- vLLM (Kwon et al., SOSP 2023): https://arxiv.org/abs/2309.06180
