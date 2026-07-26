# LLM Fundamentals for Backend Engineers

This file covers the LLM API surface the way you would study any third-party dependency before putting it on the critical path: the request/response model, the units of cost, the latency profile, and the knobs that control both. None of this requires ML background — it requires the same discipline you apply to a payments provider or a search cluster.

## How LLM APIs work

Every major provider (OpenAI, Anthropic, Google) exposes roughly the same shape: a stateless HTTPS endpoint that takes a list of messages and returns a completion.

```json
POST /v1/messages
{
  "model": "claude-sonnet-5",
  "max_tokens": 1024,
  "system": "You are a support assistant for an e-commerce platform.",
  "messages": [
    {"role": "user", "content": "Where is my order #12345?"}
  ]
}
```

Three properties matter more than anything else:

1. **The API is stateless.** There is no server-side conversation. Every request must carry the full history (system prompt + all prior turns). A 20-turn conversation means the 20th request re-sends everything — which is why input grows quadratically over a conversation's life and why prompt caching (below) exists.
2. **Billing is per token, split by direction.** You pay for input tokens (everything you send) and output tokens (everything generated), and output is typically 3-6x the input price. A **token** is a subword unit — roughly 3-4 English characters, ~0.75 words; code and non-English text tokenize less efficiently. Tokenizers differ per model family, so never reuse counts across providers; use the provider's count-tokens endpoint.
3. **The context window is a hard capacity limit.** Frontier models in 2026 offer 200K-1M tokens of context with 64K-128K max output. Exceeding it is a hard error (or forced truncation), so long-running conversations need summarization/compaction, and "just stuff everything in" has a real ceiling and a real bill.

The response carries the generated content plus two fields your code must always read:

- **`stop_reason`** — why generation ended. The values map directly to control flow:

| stop_reason | Meaning | Your handling |
| --- | --- | --- |
| `end_turn` | Model finished naturally | Normal path |
| `max_tokens` | Hit your output cap | Output is truncated mid-thought — retry with a higher cap, or stream |
| `tool_use` | Model wants to call a tool | Execute it, append the result, call again (the agent loop) |
| `refusal` | Safety systems declined | Surface gracefully; do not blind-retry the same prompt; consider fallback model |

  Code that reads the content without checking `stop_reason` ships truncated answers and silently dropped tool calls.

- **`usage`** — exact input/output/cached token counts for this call. This is your metering: multiply by price, emit as metrics, attribute to tenant/feature. Cost observability starts here (see `03-llm-systems-in-production.md`).

### Streaming

A full response can take 5-60+ seconds to generate. All providers therefore support streaming via **server-sent events (SSE)**: the connection stays open and tokens arrive as they are generated:

```text
event: content_block_delta
data: {"delta": {"type": "text_delta", "text": "Your order"}}

event: content_block_delta
data: {"delta": {"type": "text_delta", "text": " shipped yesterday"}}

event: message_stop
data: {"usage": {"input_tokens": 812, "output_tokens": 143}}
```

Two latency numbers matter and they are different:

- **TTFT (time to first token)** — typically 0.3-2s. This is what the user perceives if you stream.
- **Total generation time** — proportional to output length, typically 30-100+ tokens/second. A 2,000-token answer takes tens of seconds regardless of TTFT.

The rule: any user-facing LLM call should stream, and any non-streamed call needs a generous HTTP timeout (SDK defaults are ~10 minutes for a reason). Streaming implications for your own edge (proxy buffering, SSE fan-out) are covered in `03-llm-systems-in-production.md`.

### Sampling and temperature

Generation is probabilistic: at each step the model samples the next token from a probability distribution. **Temperature** scales that distribution — low values (0-0.3) make output nearly deterministic and repetitive-safe (good for extraction, classification, code), higher values (0.7-1.0) increase variety (good for creative generation). Two caveats worth stating in an interview: temperature 0 is *not* a determinism guarantee (GPU nondeterminism and provider-side changes still produce variance), and some 2025+ models have removed sampling parameters entirely in favor of prompt-level steering — so never design a system whose correctness depends on identical outputs for identical inputs.

## Prompt engineering basics (the 20% a backend engineer needs)

- **System prompt = configuration, user messages = data.** Put role, rules, tone, and output format in the system prompt; keep it byte-stable (no timestamps, no per-request IDs) because caching is prefix-based. Inject dynamic context late in the message list, not into the system prompt.
- **Few-shot examples beat instructions.** Two or three input→output examples of the exact format you want outperform paragraphs of description, especially for classification and extraction.
- **Be explicit about the failure path.** Tell the model what to do when it cannot answer ("if the order ID is not found in the provided context, say so — do not guess"). Unhandled ambiguity is where hallucinations come from.
- **Structure with delimiters.** Wrap injected documents/user content in XML-ish tags (`<context>...</context>`) so the model can distinguish instructions from data. This also helps (but does not solve — see file 03) prompt injection.

Treat prompts like code: version them, review changes, and run them against an eval set before deploying (file 03). A one-word prompt change is a production deploy.

## Structured output and tool calling

Free text is useless to a backend. Two mechanisms make LLM output machine-consumable:

**Structured output (JSON schema enforcement).** You pass a JSON schema; the provider constrains decoding so the response is guaranteed-valid JSON matching the schema. This replaced the old "please respond in JSON" + retry-on-parse-failure dance and should be your default for anything a program consumes:

```python
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=512,
    messages=[{"role": "user", "content": ticket_text}],
    output_config={"format": {"type": "json_schema", "schema": {
        "type": "object",
        "properties": {
            "category": {"type": "string", "enum": ["billing", "shipping", "technical", "other"]},
            "sentiment": {"type": "string", "enum": ["positive", "neutral", "negative"]},
            "needs_human": {"type": "boolean"}
        },
        "required": ["category", "sentiment", "needs_human"],
        "additionalProperties": False
    }}}
)
```

**Tool / function calling.** You declare tools (name, description, JSON schema of parameters); the model responds with a structured "call this tool with these arguments" block instead of text. *Your code* executes the tool and sends the result back; the model then continues. This is the primitive under every agent:

```text
user: "Where is order #12345?"
  → model returns tool_use: get_order_status(order_id="12345")
  → your code calls the orders service, appends the result as a tool_result message
  → model returns text: "Your order shipped yesterday via DHL..."
```

Systems points interviewers listen for: the loop is driven by **your** code (the model only ever *requests* calls); tool arguments are model output and must be validated like user input; tools with side effects (refunds, emails) need idempotency keys and often human approval; and each loop iteration is a full API round trip re-sending the whole history — so chatty tool loops get expensive fast.

## Cost and latency characteristics

Representative per-million-token prices as of mid-2026 (they change quarterly — quote the shape, not the digits):

| Tier | Example models | Input $/M | Output $/M | Typical use |
| --- | --- | --- | --- | --- |
| Frontier | GPT-5.6 Sol, Claude Opus 4.8 | ~$5 | ~$25-30 | Hard reasoning, agents, code generation |
| Mid | Claude Sonnet 5, GPT-5.6 Terra, Gemini 3.1 Pro | ~$2-3 | ~$12-15 | Most production workloads |
| Cheap/fast | GPT-5.6 Luna, Gemini 3 Flash, Claude Haiku 4.5 | ~$0.50-1 | ~$3-6 | Classification, routing, extraction, high volume |
| Budget open-weight | DeepSeek-V4 and hosted OSS models | ~$0.15 | ~$0.30 | Cost-dominant bulk work |

Structural facts to internalize:

- **Output tokens cost 3-6x input tokens** (generation is sequential and compute-bound; input processing is parallel). Cost control therefore starts with capping and shortening *output*: `max_tokens` limits, "be concise" instructions, enum outputs instead of prose.
- **Input volume dominates most real workloads anyway**, because you re-send history and stuff context (RAG chunks, tool results) on every call. A chat feature's cost curve is driven by conversation length; an agent's by loop count.
- **Cached input is ~10% of the standard rate** across major providers — the single biggest cost lever (below).
- **Batch APIs give ~50% off** for non-latency-sensitive work (nightly enrichment, backfills, eval runs) with results in minutes-to-24h. If the work doesn't need to be synchronous, it shouldn't pay the synchronous price.
- Latency: TTFT sub-second to ~2s; full responses seconds to minutes. Plan capacity like a slow downstream: a worker holding a connection open for 30s is a very different concurrency profile from a 20ms DB call.

### Worked example: what does a chat feature cost?

Interviewers love a back-of-envelope here, in the same spirit as QPS math. Suppose a support assistant on a mid-tier model ($3/M input, $15/M output):

```text
System prompt + tool definitions:        2,000 tokens (static → cacheable)
RAG context per turn:                    1,500 tokens
Average conversation: 8 turns, history grows ~600 tokens/turn

Input per conversation ≈ Σ over 8 turns of (2,000 + 1,500 + history)
                       ≈ 8×3,500 + (0+600+...+4,200) ≈ 28,000 + 16,800 ≈ 45K tokens
Output per conversation ≈ 8 × 250 = 2K tokens

Uncached:  45K × $3/M + 2K × $15/M  ≈ $0.135 + $0.03  ≈ $0.17/conversation
With prompt caching (~70% of input served from cache at ~10% price):
           (13.5K × $3 + 31.5K × $0.30 + 2K × $15)/M   ≈ $0.08/conversation
```

At 50K conversations/day that is the difference between ~$8.5K/day and ~$4K/day — from a caching change alone. The number itself matters less than showing you can decompose cost into (static prefix × cache rate) + (context × turns) + (output × turns) and identify which term dominates.

## Model selection: capability vs cost vs latency

Model choice is a routing decision, not a one-time decision. The senior-engineer answer is a **cascade**:

1. **Default down, escalate up.** Route everything to the cheapest model that passes your evals for that task; escalate to a bigger model on failure signals (low confidence, schema violation, user retry, "needs_human" flag). Classification and routing rarely need more than the cheap tier; multi-step agent reasoning usually needs mid or frontier.
2. **Split the pipeline by step.** A support assistant might use a cheap model to classify + retrieve, and a mid model to draft the answer. Don't pay frontier prices for the classification step because the *last* step needs quality.
3. **Latency is a tier property too.** Small models are also 2-5x faster. For autocomplete-style UX, the fast tier is chosen for TTFT as much as for cost.
4. **Pin versions and re-evaluate on upgrade.** Model upgrades are behavioral breaking changes. Pin the model ID, run your eval suite against the new version, then migrate — exactly like a major dependency bump.

## Caching strategies

Two very different caches, both worth naming explicitly in interviews:

**Prompt caching (provider-side prefix cache).** Providers cache the processed *prefix* of a prompt; a request whose prefix byte-matches a cached one pays ~10% of input price for the cached span (writes cost a small premium, ~1.25x). It is a **prefix match**: one changed byte early in the prompt invalidates everything after it. Design consequences:

- Order content by stability: static system prompt and tool definitions first, per-session context next, the volatile user turn last.
- Kill silent invalidators: `datetime.now()` in the system prompt, unsorted JSON serialization, per-request UUIDs, per-user tool sets.
- Multi-turn conversations and agent loops are the big win — every iteration re-sends history that is now ~90% cheaper. Verify with the usage fields (`cache_read_input_tokens` etc.); zero reads across identical requests means an invalidator is hiding somewhere.

A prompt structured for caching looks like this — stable spans first, cache breakpoint at the stability boundary, volatile content after:

```text
[tools: deterministic order, sorted keys]        ── static, cached
[system prompt: no timestamps, no user IDs]      ── static, cached
[session context: user profile, injected once]   ── per-session, cached after turn 1
────────── cache breakpoint ──────────
[conversation history]                            ── grows, incrementally cached
[current user message + retrieved chunks]         ── volatile, never cached
```

**Semantic caching (your-side response cache).** Classic cache-aside, but keyed by *meaning*: embed the incoming query, look up nearest neighbors in a vector store of previously answered queries, and serve the stored response if similarity exceeds a threshold:

```python
def answer(query: str, tenant: str) -> str:
    key_vec = embed(normalize(query))
    hit = vector_cache.nearest(key_vec, filter={"tenant": tenant}, k=1)
    if hit and hit.similarity >= 0.92 and not expired(hit):
        return hit.response                      # LLM call skipped entirely
    response = call_llm(query)
    vector_cache.upsert(key_vec, response, ttl=86400, tenant=tenant)
    return response
```

Great for high-repetition workloads (FAQ-shaped support traffic, search suggestions). Trade-offs the interviewer wants to hear: threshold tuning is precision/recall on cache correctness (too loose serves *wrong* answers, which is worse than a miss — measure false-hit rate on a labeled sample before trusting it); entries need TTL/invalidation when underlying facts change; don't use it for personalized or context-dependent answers unless user/tenant is part of the key. An exact-match Redis cache on a normalized query string is the cheap first step and often captures most of the win.

The two caches stack, and it helps to keep the layers straight:

| Layer | What is cached | Saves | Owned by |
| --- | --- | --- | --- |
| Semantic/exact response cache | The final answer | The entire LLM call | You |
| Prompt cache | Processed input prefix | ~90% of cached input cost | Provider |
| (RAG) embedding cache | Query embeddings | Embedding calls (small) | You |

## Rate limits and retries

Provider rate limits are multi-dimensional: **RPM** (requests/min), **TPM** (tokens/min, often split input/output), sometimes tokens/day — per organization, per model. Token-based limits mean one user pasting a 100K-token document can consume the same budget as a thousand small requests. On breach you get **429** with a `retry-after` header; under provider load you may also see **529/503 overloaded**, which is retryable but not your fault.

The standard hardening stack, most of which you already know from generic resilience patterns:

```go
// Client-side: bounded retry with exponential backoff + jitter.
// Retry 429 (honoring Retry-After) and 5xx/529; never retry 400/401/404.
delay := min(base*math.Pow(2, float64(attempt)) + rand.Float64()*jitter, maxDelay)
```

- **Respect `retry-after`; add jitter.** Synchronized retries from a fleet are a thundering herd against your own quota.
- **Client-side concurrency limiting.** A semaphore / token bucket sized below your TPM keeps you from ever hitting 429 in bursts — smoother than reacting to 429s. Track spend against the limit using the usage returned on every response.
- **Queue, don't drop.** For non-interactive work, put requests on a queue (SQS/Kafka) and let workers drain at the sustainable rate; 429s then become backpressure, not errors. Interactive traffic gets priority; batch traffic soaks the leftover quota.
- **Idempotency on retries.** An LLM call that timed out may have completed. For calls with side effects (via tools) or billed cost you attribute to tenants, use request IDs / idempotency keys so a retry doesn't double-execute or double-charge.
- **Budget alarms and kill switches.** Rate limits protect the provider; *spend* limits protect you. Per-feature and per-tenant token budgets with alerts — a retry loop against an expensive model is a five-figure incident by morning (more in file 03).

The theme to close on in an interview: nothing here is exotic. It is timeouts, backoff, bulkheads, cache-aside, and queues — tuned for a dependency that is 100x slower and 1000x more expensive per call than the ones you are used to.
