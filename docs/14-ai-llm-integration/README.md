# AI & LLM Integration

Backend interviews in 2025-2026 increasingly include an AI twist: "now add semantic search to this catalog," "how would you bolt an assistant onto this support system," "what breaks when you put an LLM call in the request path?" These are not ML research questions. Nobody expects you to derive attention math or train a model. What interviewers are probing is whether you can treat an LLM as **just another remote dependency** — an expensive, slow, occasionally wrong, rate-limited third-party API — and apply the same systems thinking you would apply to any other external service: caching, queueing, retries, timeouts, fallbacks, cost control, and observability.

That framing is the whole section in one sentence. An LLM provider is a downstream service with unusual characteristics: latency measured in seconds instead of milliseconds, cost measured per token instead of per request, output that is non-deterministic and must be treated as untrusted input, and rate limits that are tighter than anything you have dealt with before. Every pattern you already know — cache-aside, backpressure, circuit breakers, the outbox, idempotent consumers — reappears here with new parameters. Candidates who realize this pass; candidates who treat "AI" as a magic box that needs a totally new playbook flounder.

## What this section covers

| File | Description |
| --- | --- |
| README.md | This overview: framing, scope, and recommended reading order. |
| 01-llm-fundamentals-for-backend.md | How LLM APIs actually work: tokens, context windows, streaming, sampling, structured output and tool calling, cost and latency characteristics, model selection, prompt and semantic caching, rate limits and retries. |
| 02-rag-and-vector-search.md | Embeddings, chunking, vector databases (pgvector vs dedicated stores), index types (HNSW, IVF), hybrid search and reranking, end-to-end RAG architecture, evaluation, and when to choose RAG vs fine-tuning vs long context. |
| 03-llm-systems-in-production.md | The production view: agents and orchestration, guardrails and prompt injection, evals and regression testing, observability and cost monitoring, queueing and backpressure, streaming to clients, failure modes and fallbacks, data privacy. |
| 04-interview-questions.md | Graded practice questions, including the three design exercises that come up most often: semantic search for a catalog, a support assistant with human escalation, and cost control at scale. |

## Scope: backend engineer, not ML researcher

Explicitly in scope:

- Calling LLM APIs correctly and economically (tokens, caching, batching, model tiers).
- Retrieval systems: embeddings, vector indexes, hybrid search, the RAG pipeline as a data pipeline.
- Productionizing: queues in front of slow calls, SSE to clients, tracing and cost attribution, fallbacks when a provider is down, treating model output as untrusted.
- Design-round answers for "add AI to this system" prompts.

Explicitly out of scope: training and fine-tuning mechanics beyond the build-vs-buy trade-off, GPU infrastructure, model internals, and prompt-engineering artistry beyond what a backend engineer needs to ship a reliable feature.

## Recommended reading order

1. `01-llm-fundamentals-for-backend.md` — the API surface and its cost/latency physics. Everything else assumes you know what a token is, why output tokens dominate cost, and why a single call can take 30 seconds.
2. `02-rag-and-vector-search.md` — the most common "add AI" design question is a retrieval question. This file is as much about search infrastructure as about LLMs.
3. `03-llm-systems-in-production.md` — the systems patterns that distinguish a senior answer: backpressure, guardrails, evals, observability, fallbacks.
4. `04-interview-questions.md` — self-test after the first three. Answer out loud before reading the model answer; for the design prompts, use the standard framework from the system design section (requirements → estimation → high-level → deep dive → trade-offs).

A note on currency: the provider and model landscape moves fast — specific model names and prices in this section are accurate as of mid-2026 and will drift. The *structure* does not: there will always be a frontier tier, a mid tier, and a cheap-fast tier; input tokens will stay cheaper than output tokens; caching will stay an order of magnitude cheaper than recomputing. Learn the structure, quote the numbers as "order of magnitude, as of when I last checked."
