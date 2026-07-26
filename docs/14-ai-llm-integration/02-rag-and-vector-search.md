# RAG and Vector Search

Retrieval-augmented generation (RAG) is the most common "add AI to this system" answer, and it is fundamentally a **search infrastructure** problem wearing an AI hat: an ingestion pipeline, an index, a query path, and a relevance problem. The LLM is the last 10%. Interviewers use RAG prompts to test whether you can design a data pipeline and reason about index trade-offs — the same skills as designing full-text search, with a new index type.

## Embeddings

An **embedding** is a fixed-length vector of floats (typically 256-3072 dimensions) produced by an embedding model, positioned so that semantically similar text lands close together. "How do I reset my password" and "forgot my login credentials" share almost no keywords but sit near each other in embedding space — that is the entire value proposition over keyword search.

Practical facts:

- **Similarity metric:** cosine similarity (or dot product on normalized vectors) is the default. The score is only meaningful *relative* to other candidates from the same model — never compare scores across models, and never mix vectors from different models (or model versions) in one index.
- **Embedding calls are cheap and fast** relative to generation — fractions of a cent per thousand chunks, tens of milliseconds. Cost lives in storage and index memory, not the embedding API.
- **Model choice is a real decision:** OpenAI text-embedding-3, Cohere Embed, Voyage, and strong open-weight models (runnable in-house for data-locality reasons) differ on multilingual quality, dimension count (bigger = better recall, more RAM), and price. Changing the model later means **re-embedding the entire corpus** — a full backfill, so treat it like a schema migration: version the model in metadata, dual-write during transition, cut over reads.
- Storage math for capacity estimates: 1536 dims × 4 bytes = ~6KB per vector; 10M chunks ≈ 60GB of raw vectors before index overhead — this is why HNSW-in-RAM sizing (below) matters.

## Chunking strategies

Documents are too large to embed whole (embedding models have their own token limits, and a whole-document vector averages away the specifics), so you split them into **chunks** — the unit of retrieval. Chunking quality is the highest-leverage, least glamorous part of RAG; bad chunking cannot be fixed downstream.

| Strategy | How | Trade-off |
| --- | --- | --- |
| Fixed-size (with overlap) | Every N tokens (e.g., 512), 10-20% overlap | Trivial, predictable; cuts sentences and tables mid-thought. The default baseline. |
| Recursive / structure-aware | Split on document structure (headings, paragraphs, code blocks) down to size limit | Respects semantic boundaries; needs format-aware parsing (Markdown, HTML, PDF layout). Usually the right answer. |
| Semantic | Split where embedding similarity between consecutive sentences drops | Best boundary quality; costs an embedding pass at ingestion, more moving parts. |
| Parent-child (small-to-big) | Embed small chunks for precise matching, return the larger parent section to the LLM | Decouples "what matches well" (small) from "what the LLM needs" (context-rich). Common senior answer. |

Sizing intuition: small chunks (128-256 tokens) match precisely but lose surrounding context; large chunks (1024+) carry context but dilute the embedding and waste prompt budget. 300-800 tokens with overlap is the usual starting range. Two refinements worth naming: **prepend document metadata** (title, section path, product name) to each chunk before embedding — a paragraph saying "it supports up to 500GB" is unsearchable without knowing what "it" is; and **contextual chunking** (using a cheap LLM at ingestion to prepend a one-line summary situating the chunk in its document) measurably improves retrieval at added ingestion cost.

## Vector databases and index types

You need a store that answers k-nearest-neighbor (kNN) queries. Exact kNN is a full scan — O(n) per query — fine to ~100K vectors, hopeless at millions. Everything at scale uses **approximate nearest neighbor (ANN)** indexes that trade a little recall for orders-of-magnitude speed.

### Index types

| Index | Structure | Strengths | Costs |
| --- | --- | --- | --- |
| **HNSW** (hierarchical navigable small world) | Multi-layer proximity graph; search greedily descends layers | High recall at low latency (ms at millions of vectors); good incremental inserts | Entire graph lives in RAM (roughly 1.5-2x raw vector size); slow to build; deletes degrade the graph until rebuild/repair |
| **IVF** (inverted file, e.g. IVFFlat) | k-means partitions vectors into clusters ("lists"); query probes the nearest few clusters | Much lower memory; fast to build; index can be disk-friendly | Lower recall at same latency — a true neighbor in an unprobed cluster is missed; needs training on representative data; recall/latency tuned via `nprobe` |
| **PQ / quantization** (composable with both) | Compress vectors (product quantization, or scalar/binary quantization) | 4-32x memory reduction; enables billion-scale | Lossy — recall drops; often paired with a re-scoring pass on full-precision vectors |

Default guidance: **HNSW if the index fits in RAM** (it usually does up to tens of millions of vectors) — it has the best recall/latency curve and no training step. IVF (+ quantization) when memory is the binding constraint or the corpus is mostly static. Both expose recall-vs-latency dials (`ef_search` for HNSW, `nprobe` for IVF) — mention that recall is *measurable* against exact kNN on a sample, so this is a tunable SLO, not a leap of faith.

One more trap interviewers like: **metadata filtering** ("only this tenant's documents, only docs updated this year"). Post-filtering after ANN can return fewer than k results (or none); good stores do filtered search in-index (pgvector via WHERE + index strategies, dedicated stores via filter-aware HNSW). Multi-tenancy pushes you toward per-tenant partitioning or stores with first-class filtered search.

### pgvector vs dedicated stores

| Option | When it wins | Watch out |
| --- | --- | --- |
| **pgvector** (Postgres extension) | You already run Postgres; up to low tens of millions of vectors; you want vectors *next to* relational data — one query joins similarity search with business filters, one transaction keeps them consistent, one system to operate | Shares resources with OLTP; HNSW build/memory pressure on the primary; scaling beyond one node is on you |
| **Dedicated vector DB** (Qdrant, Pinecone, Weaviate, Milvus) | 100M+ vectors, high QPS, need sharding/replication of the index itself, heavy filtered search | Another stateful system to operate (or a vendor bill); data now dual-written between source-of-truth DB and vector store — you own that consistency (outbox/CDC, same as any derived store) |
| **Existing search engine** (OpenSearch/Elasticsearch with kNN) | You already run it for BM25 — get hybrid search in one system | Operationally heavy if you *don't* already run it |

The interview-safe default: **start with pgvector**, treat the vector index as a derived read model fed by CDC/outbox from the source of truth, and migrate to a dedicated store only when scale or filtered-search performance forces it. That sentence demonstrates both pragmatism and the "derived data" mental model.

What that looks like concretely in Postgres:

```sql
CREATE TABLE doc_chunks (
    id          bigserial PRIMARY KEY,
    doc_id      bigint NOT NULL,
    chunk_seq   int    NOT NULL,
    tenant_id   bigint NOT NULL,
    content     text   NOT NULL,
    embedding   vector(1536) NOT NULL,   -- pgvector type
    model_ver   text   NOT NULL,          -- embedding model version
    updated_at  timestamptz NOT NULL DEFAULT now(),
    UNIQUE (doc_id, chunk_seq, model_ver)
);

CREATE INDEX ON doc_chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Query: top-8 chunks for this tenant, by cosine distance
SELECT id, doc_id, content, 1 - (embedding <=> $1) AS similarity
FROM doc_chunks
WHERE tenant_id = $2 AND model_ver = 'te3-large-v1'
ORDER BY embedding <=> $1
LIMIT 8;
```

Two operational notes worth dropping in an interview: the HNSW index build is memory- and CPU-hungry — build with a raised `maintenance_work_mem`, ideally on a replica or during off-peak; and `ef_search` (`SET hnsw.ef_search = 100`) is the per-session recall/latency dial. The `UNIQUE (doc_id, chunk_seq, model_ver)` constraint is what makes ingestion idempotent — re-processing an event upserts instead of duplicating.

## Hybrid search and reranking

Vector search alone fails on exact identifiers — SKUs, error codes, names, "iPhone 15 Pro Max 256GB" — precisely where keyword search excels; keyword search fails on paraphrase, where vectors excel. Production systems therefore run **hybrid search**:

1. Run **BM25** (keyword) and **vector** search in parallel, take top-N from each.
2. Merge with **reciprocal rank fusion (RRF)** — score each doc by Σ 1/(k + rank) across the lists. RRF is the standard because it needs no score normalization between incomparable scoring systems.
3. Optionally **rerank**: feed the top ~50-100 fused candidates through a cross-encoder reranker (Cohere Rerank, Voyage rerank, or an open cross-encoder) that scores each (query, document) pair jointly. Rerankers are far more accurate than embedding similarity (which compresses each side independently) but too slow to run over the whole corpus — hence retrieve-wide-then-rerank: cheap recall stage, expensive precision stage. This is the same two-stage pattern as candidate-generation + ranking in recommender systems.

RRF in five lines, because interviewers sometimes ask you to sketch it:

```python
def rrf(result_lists: list[list[str]], k: int = 60) -> list[str]:
    scores = defaultdict(float)
    for results in result_lists:                    # one list per retriever
        for rank, doc_id in enumerate(results):
            scores[doc_id] += 1.0 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

Pipeline shape: `BM25 top-100 ∪ vector top-100 → RRF → rerank top-50 → take top-5-10 into the prompt`. Reranking adds ~50-200ms and a per-query fee; it is usually the single biggest relevance win after fixing chunking.

### Query-side improvements

Retrieval quality is a function of the query as much as the index, and the query the user typed is often not the query you should embed:

- **Conversational query rewriting.** In a chat, "how much does it cost?" is unsearchable without resolving "it" from history. A cheap-model call rewrites the turn into a standalone query ("pricing for the Pro plan") before retrieval. This is near-mandatory for multi-turn RAG and the most commonly forgotten piece in interview answers.
- **Query expansion / multi-query.** Generate 2-3 paraphrases, retrieve for all, fuse with RRF — buys recall for ambiguous queries at the cost of parallel index hits.
- **HyDE (hypothetical document embeddings).** Have the model draft a hypothetical *answer* and embed that instead of the question — answers live closer to documents in embedding space than questions do. A known trick worth naming, not a default.
- **Intent routing.** Not every message needs retrieval ("thanks!", "make it shorter"). A classifier that skips retrieval when it isn't needed saves latency and avoids stuffing irrelevant chunks that degrade the answer.

## RAG architecture end-to-end

Two pipelines, deliberately decoupled:

```text
INGESTION (async, queue-driven)
  source docs → parse/clean → chunk → enrich (metadata, contextual summary)
      → embed (batch) → upsert vectors + metadata → index
  triggered by CDC/outbox events on the source of truth; idempotent upserts
  keyed by (doc_id, chunk_seq, embedding_model_version)

QUERY (sync, latency-sensitive)
  user query → [rewrite/expand query, embed]           ~20-50ms
      → hybrid retrieve (BM25 + ANN) → RRF             ~10-50ms
      → rerank top-N                                   ~50-200ms
      → assemble prompt (system + top chunks + query)
      → LLM generates, streaming, with citations       ~1-10s
```

A capacity estimate for a mid-size deployment, in the spirit of the standard framework:

```text
Corpus: 200K documents, avg 4K tokens each → ~800M tokens
Chunking at 500 tokens, 15% overlap → ~1.9M chunks
Vectors: 1.9M × 1536 dims × 4B ≈ 12GB raw; HNSW in RAM ≈ 20-24GB → one node
Ingestion embedding cost: ~800M tokens at ~$0.1/M ≈ $80 one-time (re-embeds on model change)
Query load: 50 QPS peak → trivial for HNSW (ms-range); reranker at 50 QPS × top-50 pairs
  is the real throughput constraint — check the reranker's rate limits, not the index's
```

Backend concerns that earn points:

- **Freshness and invalidation.** The vector index is a cache of derived data. Doc updated → re-chunk → re-embed → replace that doc's chunks (delete-then-insert by doc_id, or versioned upsert). Deletions must propagate or you serve stale/leaked content. Event-driven via outbox beats nightly rebuilds for anything user-facing.
- **Grounding and citations.** Instruct the model to answer *only* from the provided chunks and cite chunk IDs; return the citations to the client. "If the answer is not in the context, say you don't know" is the cheapest hallucination mitigation there is.
- **Failure modes down the pipeline:** empty retrieval (below threshold → fall back to "no answer found" rather than letting the model freestyle), context overflow (too many/large chunks → cap by token budget), and the classic **garbage-in**: retrieval returning plausible-but-wrong chunks produces confident wrong answers — which is why evaluation is its own section.

## Evaluating RAG

Split the evaluation, because failures split the same way — retrieval failures and generation failures need different fixes:

**Retrieval metrics** (needs a golden set of query → relevant-chunk labels; a few hundred pairs, often bootstrapped by an LLM and human-reviewed):

- **Recall@k** — fraction of queries where a relevant chunk appears in the top k. The primary metric: if the answer isn't retrieved, nothing downstream can save you.
- **MRR / NDCG** — rank-position quality; matters because chunks earlier in the prompt get more attention.
- Measure per change: chunk size, hybrid weights, reranker on/off — this is your A/B harness for the retrieval stage.

**Generation metrics** (typically scored by an LLM-as-judge with spot-check human audits):

- **Groundedness / faithfulness** — is every claim in the answer supported by the retrieved chunks? The anti-hallucination metric.
- **Answer relevance** — does it actually address the question?
- **Context precision** — how much of the retrieved context was actually useful (are you paying for noise)?

A golden-set entry is small and boring, which is the point — it is test data:

```json
{
  "query": "can I get a refund after 3 months?",
  "relevant_chunk_ids": ["policies/refunds.md#window", "policies/refunds.md#exceptions"],
  "expected_answer_contains": ["90 days"],
  "must_not_contain": ["full refund guaranteed"]
}
```

Run the suite in CI on every prompt/model/chunking change (regression testing for LLM systems is expanded in file 03). In production, log query, retrieved chunk IDs, scores, and answer — thumbs-down events plus logged retrievals are your future eval set.

A debugging heuristic worth stating: when a RAG answer is wrong, **look at the retrieved chunks before touching the prompt**. If the right chunk was never retrieved, it is a retrieval problem (chunking, hybrid weights, query rewriting); if it was retrieved and ignored or contradicted, it is a generation problem (prompt, grounding instruction, model tier). Teams that skip this triage tune prompts for weeks to fix an indexing bug.

## Multi-tenancy and access control

Retrieval is a data-access path and must enforce the same authorization as any other read API — a point interviewers increasingly probe because it is where real-world RAG deployments leak:

- **Filter at the index query, server-side.** `tenant_id` (and document ACLs where they exist) go into the vector/BM25 query as filters derived from the authenticated session — never trust a tenant ID that transited the model or the client.
- **Permissions change faster than indexes.** If document ACLs are volatile (shared drives, per-user folders), either re-check permissions on the retrieved candidates against the source of truth at query time (post-filter with over-fetch), or accept an SLA on permission-revocation propagation and document it.
- **Derived stores inherit deletion obligations.** A GDPR delete, a legal hold, an offboarded customer — all must propagate to chunks, vectors, and any semantic cache. Keying everything by `doc_id`/`tenant_id` is what makes that a `DELETE WHERE` instead of an archaeology project.

## RAG vs fine-tuning vs long context

The closing question of most RAG interviews. The clean framing: **RAG adds knowledge, fine-tuning adds behavior, long context is a simplicity play at small scale.**

| Approach | Best for | Freshness | Cost profile | Weaknesses |
| --- | --- | --- | --- | --- |
| **RAG** | Factual knowledge: docs, catalogs, tickets, policies; anything that changes; anything needing citations or per-tenant access control | Minutes (index update) | Infra + per-query retrieval; no training | Pipeline complexity; quality bounded by retrieval |
| **Fine-tuning** | Behavior: output format/style, domain vocabulary, reliably following a complex task spec, distilling a big model's behavior into a cheap one | Frozen at training time | Training runs + eval + redo on every base-model upgrade | Terrible at *knowledge* injection — facts don't stick reliably and can't be deleted (a compliance problem); no citations |
| **Long context** | Small, bounded corpora (< a few hundred K tokens): one contract, one codebase slice, a day's logs | Per-request (send the latest) | Pay to re-send the corpus every call — prompt caching makes repeat queries against the *same* corpus cheap | Doesn't scale past the window; per-query cost and latency grow with corpus; retrieval quality degrades in giant prompts ("lost in the middle") |

They compose: the common production stack is RAG for knowledge + a fine-tuned small model for the cheap pipeline steps (classification, query rewriting) + long context with prompt caching for "chat with this document" features. Saying "these are complements, not competitors — pick per requirement" is exactly the trade-off-first answer this whole guide keeps pushing.
