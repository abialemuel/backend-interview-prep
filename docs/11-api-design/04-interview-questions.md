# API Design Interview Questions

Use these for self-testing after reading the first three files. Answer out loud before reading the model answer — in this round the precision of your wording (exact status codes, exact header names, concrete request/response shapes) carries more of the grade than in system design. Design exercises follow the structure from the README: resources, operations, contract details, failure handling, evolution.

The grading rubric across levels is consistent: junior questions test whether you know the conventions; senior questions test whether you can defend a trade-off and design for failure (retries, concurrency, abuse); staff questions test whether you think in contracts, consumers, and years — evolution policy, organizational consistency, and the judgment to break a rule deliberately.

## Junior

### Q1: Map CRUD operations for an `orders` resource onto HTTP correctly.

**Answer:** `GET /orders` lists (paginated, filterable); `GET /orders/{id}` fetches one (200, or 404 if absent); `POST /orders` creates (201 with a `Location: /orders/{id}` header and the created resource in the body); `PATCH /orders/{id}` partially updates (200 with the updated resource, or 204); `DELETE /orders/{id}` removes (204; a second DELETE can return 404 — DELETE is idempotent in effect, not in response). Use plural nouns, keep verbs out of URLs (`POST /orders`, never `/createOrder`), and reserve GET for side-effect-free reads because intermediaries and prefetchers will replay GETs on the assumption they are safe.

### Q2: What is the difference between 401 and 403? Between 400 and 422?

**Answer:** 401 means unauthenticated — no credentials or invalid credentials; the fix is to (re)authenticate, and the response should indicate how (`WWW-Authenticate`). 403 means authenticated but not permitted — retrying with the same identity will never succeed. (For resources whose existence is itself sensitive, returning 404 instead of 403 avoids confirming existence — the tenant-isolation pattern.) 400 vs 422: 400 is a malformed request — broken JSON, wrong types; 422 is well-formed but semantically invalid — email fails validation, amount is negative. Plenty of good APIs use 400 for both; the point is to pick a convention, document it, and be consistent, because clients branch on these codes.

### Q3: Why should a JSON API never return errors with HTTP 200?

**Answer:** The status code is the machine-readable contract layer that everything HTTP-aware relies on: retry middleware retries on 5xx, caches store 200s, load balancers health-check on status, and monitoring computes error rates from status classes. A 200 wrapping `{"success": false}` makes errors cacheable, makes retries impossible to automate, and makes your dashboards read healthy during an outage. It also forces every client to implement a second, bespoke error-detection layer. Status code for the class of outcome, body (RFC 9457 problem details) for the specifics.

### Q4: Why are floats wrong for money in an API, and what do you use instead?

**Answer:** IEEE 754 floats cannot represent most decimal fractions exactly — `0.1 + 0.2 != 0.3` — so serializing money as JSON numbers invites rounding drift, and different client languages will deserialize into different precisions. Use integer minor units (`"amount": 2000` meaning 20.00, with an explicit `"currency": "USD"` — Stripe's convention) or a string decimal (`"amount": "20.00"`). Currency must always be explicit because minor-unit scale varies (JPY has no minor unit, KWD has three). The same logic applies to any precision-sensitive value: 64-bit IDs go as strings too, because JavaScript numbers lose integer precision above 2^53.

### Q5: Where does a given piece of request data belong: path, query string, header, or body?

**Answer:** Path for identity — what resource this is (`/orders/ord_42`); a request for a different path is a request for a different thing, which is also why the path is the cache key. Query string for modifying a read — filters, sort, pagination, expansion (`?status=shipped&limit=50`); never secrets, since URLs land in access logs, browser history, and Referer headers. Headers for transport- and cross-cutting concerns that are not the payload: auth (`Authorization`), content negotiation, idempotency keys, request IDs, conditional-request preconditions. Body for the representation itself — the data being created or changed. The common smells: business data smuggled into headers (invisible in docs and tooling), auth tokens in query strings (logged everywhere), and verbs in the path (`/orders/create`) doing the method's job.

### Q6: What is an ETag and what are its two uses?

**Answer:** An ETag is a version fingerprint of a resource representation, returned as a response header. Use one: caching — the client re-requests with `If-None-Match: "v7"`, and if unchanged the server returns 304 with no body, saving bandwidth and time. Use two, the more important one: optimistic concurrency — the client sends its write with `If-Match: "v7"`, and if the resource has since moved to v8 the server returns 412 Precondition Failed, forcing the client to re-fetch and re-apply instead of blindly overwriting someone else's change. It is the HTTP-contract expression of `UPDATE ... WHERE version = ?`, and it prevents the lost-update problem without pessimistic locks.

## Senior

### Q7: Offset vs cursor pagination — explain the trade-off and when each is right.

**Answer:** Offset (`LIMIT 50 OFFSET 100`) is simple and supports random access ("page 7 of 41"), but has two real defects: under concurrent writes the pages shift, so clients see duplicates or silently miss rows — a correctness bug for feeds or any syncing consumer — and the database must scan and discard all skipped rows, so cost grows linearly with depth (`OFFSET 100000` is a slow query). Cursor (keyset) pagination returns an opaque token encoding the last row's sort key; the next page is an index seek — constant cost at any depth, stable under inserts:

```sql
-- page N, regardless of N:
SELECT * FROM orders
WHERE (created_at, id) < ('2026-05-10 12:00:00', 847)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

Costs: no jump-to-page, no cheap total count, and the sort keys must be uniquely tie-broken (append `id` to any timestamp sort — a bare timestamp sort with duplicate timestamps skips or repeats rows at page boundaries). The cursor must be opaque — base64, ideally signed — or its internals become contract. Default to cursor for anything unbounded or machine-consumed; offset is acceptable for small, slow-changing, human-browsed admin tables. Either way, paginate every collection from day one with an enforced max `limit` — retrofitting pagination onto a shipped endpoint is a breaking change, and an unbounded list works fine in dev right up until a tenant has 500k rows in production.

### Q8: A client's POST /payments call times out. Walk through what can have happened and how your API makes the retry safe.

**Answer:** A timeout is ambiguous three ways: the request never arrived, the server failed mid-processing, or the payment succeeded and the response was lost. The client cannot distinguish them, so the API must make retrying always-safe: idempotency keys. The mechanism:

1. The client sends `Idempotency-Key: <uuid>` minted per logical operation (per "Pay" click, not per HTTP attempt).
2. The server atomically claims the key — a unique-constraint insert, in the same transaction as (or atomically ahead of) the business write — storing the request hash and status `in_progress`.
3. On completion, the response (status + body) is stored against the key.
4. Retry with a completed key replays the stored response without re-executing; retry while `in_progress` gets 409 rather than a racing second execution; the same key with a different body gets 422, because that is a client bug worth failing loudly on.

Keys are scoped per API key and expire after ~24h. Without this, "retry on timeout" and "never double-charge" are irreconcilable client requirements — which is why every serious payments API makes the key mandatory rather than optional.

### Q9: Design the rate limiting for a public API: algorithm, enforcement point, and contract.

**Answer:** Algorithm: token bucket — capacity B allows legitimate bursts (a page load firing ten calls), refill rate R enforces the sustained average; the pair (B, R) expresses per-tier policy cleanly. Fixed windows allow 2x bursts at boundaries; sliding-window counters are a fine near-exact alternative. Enforcement: counters in Redis updated by an atomic Lua script (refill-by-elapsed-time, then decrement) so multiple gateway/app instances share state without races; layered — per-IP at the edge/CDN for abuse, per-API-key at the gateway for tiering, per-tenant-per-endpoint in-app for expensive routes. Decide the Redis-down behavior explicitly: fail open for most APIs (availability over enforcement), fail closed only where the limit is a security control. Contract: 429 with `Retry-After`, plus `RateLimit-Limit`/`RateLimit-Remaining`/`RateLimit-Reset` on every response so clients can self-throttle, and documentation telling clients to back off with jitter — otherwise all limited clients retry in lockstep and you have manufactured a thundering herd.

### Q10: When would you choose gRPC over REST, and what does gRPC cost you?

**Answer:** Choose gRPC for internal service-to-service traffic: the protobuf contract is compiler-enforced in every language (a whole class of serialization bugs disappears), the binary encoding is several times smaller and cheaper to encode than JSON (real CPU at high internal QPS), deadlines propagate through the call chain so a timed-out edge request cancels all downstream work instead of leaving servers computing responses nobody will read, and streaming (server, client, bidirectional) is first-class. Costs: browsers cannot speak it natively (gRPC-Web or a transcoding gateway — so public APIs usually stay REST), debugging needs tooling instead of curl, and HTTP/2's long-lived multiplexed connections defeat naive L4 load balancing — you need L7-aware balancing (Envoy sidecars) or client-side lookaside, or one backend gets all the traffic. The standard shape: REST (or gRPC-transcoded REST) at the edge, gRPC east-west.

### Q11: Your mobile team complains the REST API forces three round trips and over-fetching per screen. GraphQL?

**Answer:** First separate the problem from the fashionable solution. If there is one mobile client (or a small set you control), the cheaper fix is a BFF — a per-frontend aggregation endpoint returning exactly the screen's shape — or even purpose-built composite endpoints; you keep HTTP caching, predictable per-request cost, and boring operations. GraphQL earns its complexity when there are many diverse clients with genuinely different data needs over a rich object graph, and no BFF team can keep up. If adopted, go in knowing the operational bill: N+1 resolution requires DataLoader-style batching per request; arbitrary client queries require depth/complexity limits and ideally persisted queries (allowlisted query hashes — which for public traffic effectively compiles the flexibility away in production); authorization moves to field level; and HTTP/CDN caching is largely lost because everything is a POST with a unique body. The senior answer is a decision process, not a verdict: over-fetching is real, but the first-line fix is aggregation, not a new query language.

### Q12: Design a webhook delivery system for your platform. What does "done well" include?

**Answer:** Consumers register endpoint URLs per event type; delivery is a queue problem, not an HTTP call in the request path. Sign every delivery — HMAC-SHA256 over `timestamp.raw_body` with a per-endpoint secret:

```http
POST https://consumer.example.com/webhooks
Webhook-Id: evt_5m1xq
Webhook-Timestamp: 1782115200
Webhook-Signature: v1,MEYCIQ...   <- HMAC over "{timestamp}.{raw_body}"

{"type": "payment.succeeded", "event_id": "evt_5m1xq", "data": {"payment_intent": "pi_8f3k2"}}
```

Consumers verify against raw bytes before parsing and reject stale timestamps (~5 min) to prevent replay. Treat delivery as at-least-once: only 2xx counts as success; retry with exponential backoff and jitter over a long horizon (up to ~72h), from a queue with per-endpoint concurrency caps so one slow consumer cannot delay everyone; auto-disable persistently failing endpoints and notify owners; provide a redelivery API and delivery logs in the dashboard. Every event carries a unique `event_id` for consumer-side dedup, and consumers must not assume ordering — which is why thin events (type + resource IDs, consumer fetches current state) beat fat payloads: out-of-order becomes harmless and sensitive data stays out of third-party logs. Document the consumer contract too: respond 2xx fast, process async, dedupe on `event_id`, reconcile via `GET /events` as a backstop for missed deliveries.

### Q13: What exactly counts as a breaking change in a JSON API? Give the non-obvious ones.

**Answer:** Obvious: removing/renaming fields or endpoints, changing types or units, making optional fields required, changing status codes or error `type` identifiers clients branch on. Non-obvious, where incidents actually come from:

- Adding a value to an enum clients treat as closed — their `switch` statement throws in production.
- Changing default sort order or default page size — clients that never passed `sort` still depended on it.
- Tightening validation: requests that used to succeed now 422; a stricter regex is a break.
- Changing ID format or length — someone's DB column is `VARCHAR(16)`, someone's regex expects digits.
- Semantic changes with identical shape: `amount` from gross to net is invisible to every schema tool and catastrophic to every consumer.
- Lowering rate limits — a batch job tuned to the old limit starts failing nightly.
- Even fixing a bug clients have coded around — Hyrum's Law says every observable behavior acquires dependents.

Mitigations: document that unknown response fields and (where declared open) unknown enum values must be tolerated; put a breaking-change diff (oasdiff) in CI so shape breaks cannot merge silently; and measure per-key usage so "can anyone be affected?" is a query, not a guess. Semantic breaks are the residual risk no tool catches — those need review culture and changelogs.

## Staff

### Q14: How do you evolve an API for years without breaking clients? Give the full playbook.

**Answer:** Four layers, in order of preference:

1. **Make non-breaking evolution the default.** Additive-only changes (new endpoints, new optional fields), explicit tolerant-reader rules in the docs (ignore unknown fields; handle unknown enum values where enums are declared open), and a written definition of "breaking" so the debate happens once, not per PR.
2. **Enforce it.** The OpenAPI spec lives in the repo; CI diffs it on every PR and fails on breaking changes. Consumer-driven contract tests (Pact) for internal consumers make "who depends on this field?" a broker query and a deploy gate rather than a guess.
3. **Measure.** Per-API-key, per-endpoint, per-field-where-it-matters usage metrics — you cannot safely retire what you cannot see, and deprecation without measurement is an outage with a countdown.
4. **When a break is unavoidable, run the deprecation machine.** Version it (URL v2 for a coarse break, or Stripe-style date-pinned versions per key if you have the infrastructure); publish a migration guide; mark the old surface with `Deprecation` and `Sunset` headers (RFC 8594) and `deprecated: true` in the spec; email the measured users; run a long notice window (6-12 months public); apply brownouts near the end (scheduled short 410 windows that flush out inattentive consumers on your schedule, not at final shutdown); then sunset with 410 Gone pointing at the guide.

The staff-level point: versioning is the last resort, not the strategy — every live version is a permanent tax on development, testing, and support, and the goal of the whole playbook is to almost never need v2.

### Q15: Design the API for a payments service (a Stripe-like charge flow).

**Answer:** Requirements first: merchants charge customers' stored payment methods; money moves exactly once regardless of retries; charges have a lifecycle (authorization, capture, refund); consumers are third-party servers, so REST + API keys.

**Resources:** `/customers`, `/payment_methods`, `/payment_intents` (the charge attempt as a first-class resource with a state machine, not a fire-and-forget action), `/refunds` (own lifecycle and queryability, so top-level with a `payment_intent` reference, not a nested action), and `/events` + webhooks for the async reverse channel.

**Core flow** — state transitions as sub-resource POSTs, each idempotent, each returning 409 with a problem-details body if the current state forbids the transition:

```http
POST /v1/payment_intents
Authorization: Bearer sk_live_...
Idempotency-Key: 6f0dc1a2-...

{"amount": 2000, "currency": "usd", "customer": "cus_9k2", "capture_method": "manual"}

HTTP/1.1 201 Created
Location: /v1/payment_intents/pi_8f3k2
{
  "id": "pi_8f3k2",
  "status": "requires_confirmation",
  "amount": 2000,
  "currency": "usd",
  "links": { "confirm": "/v1/payment_intents/pi_8f3k2/confirm" }
}
```

Then `POST /payment_intents/{id}/confirm`, and later `/capture`. Contract details that carry the grade: money as integer minor units plus explicit currency; IDs prefixed and opaque (`pi_`, `re_`); a mandatory `Idempotency-Key` on every mutating call. Declines are not HTTP errors in the transport sense — the intent moves to `"status": "requires_payment_method"` with a `last_payment_error` object, because a decline is a domain outcome to render, not a retryable transport failure (402 with problem details is a defensible alternative; the key is distinguishing transport errors from domain outcomes).

**The async boundary:** card networks are asynchronous, so the contract states that statuses may change out-of-band; merchants receive signed webhooks (`payment_intent.succeeded`, thin payload, `event_id` for dedup) and must treat webhooks — backstopped by polling `GET /events` — not synchronous responses, as the source of truth for fulfillment.

**Platform surface:** cursor pagination on all lists (`/payment_intents?customer=cus_9k2&created_gte=...`), RFC 9457 errors with `request_id`, per-key rate limits with standard headers, scoped keys (`sk_live_` vs restricted read-only keys). The graded core: idempotency end-to-end, the state machine as resource design, and the sync/async boundary made explicit in the contract.

### Q16: Fifty teams each ship their own service APIs and they are drifting: naming, errors, pagination all differ. Fix it as the platform lead.

**Answer:** Treat consistency as an engineering system, not a style debate.

1. Write a short, opinionated API style guide — naming, pagination shape, error shape (problem details), auth, versioning policy — decided once by a small group, with an ADR trail. A 200-page guide nobody reads is failure; twenty enforced rules beat two hundred aspirational ones.
2. Encode it: Spectral lint rules over every OpenAPI spec in CI, so "field names are snake_case" and "every list operation has cursor params" are build failures, not review comments. Add breaking-change diffing (oasdiff) to the same pipeline.
3. Design-first workflow: specs reviewed before implementation, with an API review forum that is consultative and fast (an SLA on reviews), reserving mandatory review for public and cross-org surfaces — a rubber-stamp committee for every internal endpoint kills velocity and gets routed around.
4. Make the right thing the easy thing: shared middleware/libraries that emit the standard error shape, pagination, idempotency handling, and metrics for free in each supported stack; scaffolding templates that start compliant.
5. A catalog (Backstage or similar) so teams discover existing APIs instead of duplicating them, with ownership and deprecation status visible.
6. Grandfather existing APIs with a lint baseline and a ratchet — new endpoints must pass, old ones improve opportunistically. A big-bang migration of fifty existing APIs will not happen and demanding it discredits the whole program.

The staff signal is sequencing tooling before policing, and acknowledging the failure mode of governance theater — process that adds friction without changing what ships.

### Q17: Your public API's p99 latency is fine but one enterprise customer reports intermittent timeouts and duplicate orders. Diagnose via the API contract lens.

**Answer:** Duplicates plus timeouts is the signature of retry-without-idempotency. The likely path: their client sets an aggressive timeout, some of your requests exceed it (their p99.9, invisible in your global p99 — first lesson: per-consumer percentile dashboards keyed by API key, because a global p99 averages away exactly the customer who is paging you), their middleware retries `POST /orders`, and the original had succeeded.

Fixes on both sides of the contract. Server: support idempotency keys on order creation with store-and-replay responses; also accept a client `order_reference` with a unique constraint as a belt-and-braces domain-level guard; return 409 on in-flight duplicate keys. Client guidance, in docs and baked into SDKs: retry only on 5xx/429/connect failures, never on an ambiguous timeout without an idempotency key; exponential backoff with jitter; sensible timeout budgets.

Then investigate the latency tail itself with request-ID correlation and tracing: a slow tenant-specific query, a cold cache, or connection-pool exhaustion under their bursts — which also suggests checking whether their burst pattern is hitting internal queueing that surfaces as latency rather than as clean 429s. The staff-level habits on display: per-consumer observability, contract-level fixes over case-by-case firefighting, and updating docs and SDKs so the next customer cannot hit the same trap.

### Q18: Design the API surface for long-running operations (e.g., a report that takes 10 minutes to generate).

**Answer:** Never hold the connection. `POST /reports` validates and enqueues, returning `202 Accepted` with the operation as a resource:

```http
POST /reports
Idempotency-Key: 9c41e8b0-...
{"type": "settlement", "period": "2026-06"}

HTTP/1.1 202 Accepted
Location: /reports/rpt_7x1
{"id": "rpt_7x1", "status": "processing", "created_at": "2026-07-01T08:00:00Z"}

GET /reports/rpt_7x1          <- poll, paced by Retry-After
{"id": "rpt_7x1", "status": "succeeded",
 "result_url": "https://files.example.com/rpt_7x1?sig=...&expires=..."}
```

Clients poll until `status` is `succeeded` (with a `result_url`, typically a time-limited signed S3 URL so the 200 MB artifact does not flow through your API tier) or `failed` (with a problem-details error object embedded). Offer a webhook (`report.completed`) so integrated consumers do not have to poll at all; SSE is the browser-facing equivalent for progress streaming. Contract details that get graded: the POST takes an idempotency key (double-submit of an expensive job is a real cost bug); status enum documented as open; failed operations keep their record (queryable post-mortem, not a 404); `DELETE /reports/{id}` as cancellation semantics — returns 202/409 depending on whether cancellation is still possible; retention policy documented (results live 30 days). This "operation as a resource" pattern is the general answer for anything async — batch imports, video transcodes, ML jobs — and interviewers use it to test whether you reach for resources or for RPC-shaped hacks under pressure.

### Q19: When is it correct to break the REST rules you have described?

**Answer:** The rules encode defaults, and staff-level judgment is knowing their boundaries. Legitimate deviations: **action endpoints** over resource purity when the domain is verb-shaped — `POST /payments/{id}/capture` beats a tortured `PATCH {"status": "captured"}` precisely because it makes side effects explicit; **POST for reads** when queries exceed URL limits or contain sensitive parameters that must not land in access logs (`POST /orders/search`) — accepting the loss of HTTP caching consciously; **RPC-style internal APIs** where REST's uniform interface buys nothing between two services owned by one team; **bulk endpoints** (`POST /orders/batch`) that violate one-resource-per-request because 10,000 sequential POSTs is worse — with an explicit partial-failure contract (207-style per-item results, or all-or-nothing, but documented); **denormalized/composite responses** that duplicate data across endpoints because a mobile round trip costs more than purity. The meta-answer: every deviation is fine when it is (a) chosen for a stated reason, (b) consistent across your API, and (c) documented in the contract. What is not fine is drift — five teams each breaking a different rule a different way. Rules are for consistency, and consistency is for consumers; when a rule stops serving consumers, break it deliberately and write it down.

### Q20: Design the API contract for bulk import (up to 100k records per request) into a CRM.

**Answer:** Two decisions dominate: the ingestion shape and the partial-failure contract. Ingestion: a single giant JSON array is hostile at 100k records (memory spikes, all-or-nothing parsing, timeout risk), so accept a file — `POST /imports` with NDJSON (one record per line, streamable and resumable) either uploaded directly or, better at this size, via a pre-signed upload URL: `POST /imports` returns `{"id": "imp_3k1", "upload_url": ...}`, the client PUTs the file to storage, then `POST /imports/imp_3k1/start`. Processing is the long-running-operation pattern from Q18: `202`, poll `GET /imports/{id}` for `{"status": "processing", "processed": 61400, "failed": 220}` progress, webhook on completion. Partial failure is the graded part — all-or-nothing is wrong for imports (one bad row in 100k should not waste the other 99,999), so the contract is per-record results: valid rows commit, invalid rows are reported in a downloadable error artifact (`GET /imports/{id}/errors` — NDJSON of `{line, field, code, message}`) so the client can fix and resubmit only the failures. Semantics that need explicit documentation: per-record idempotency via a client-supplied `external_id` with upsert-or-reject-duplicates as a declared mode (`"on_conflict": "update" | "skip" | "fail"`); ordering not guaranteed; the whole import idempotent via `Idempotency-Key` so a retried `start` cannot double-run. Rate/size limits in the contract: max file size, max concurrent imports per tenant, and import jobs metered separately from the request-rate limit — a bulk endpoint that shares the interactive rate limit starves one or the other. The senior tell is refusing the naive `POST` with a 100k-element array and designing for the failure modes: partial validity, retry, and progress visibility.

!!! note "Using these in practice"
    The design exercises (Q15, Q18, Q20) are worth doing on a whiteboard with a timer: 25 minutes, actual request/response examples written out, then compare against the model answer for what you skipped. The most common gap in real interviews is not knowledge but specificity — candidates say "and then pagination" instead of writing the cursor field into the response. Writing the JSON forces the decisions the interviewer is grading.

    For the follow-up rounds these questions feed into — "now scale it," "now make it multi-region" — pair this section with the system design section; the API contract you define here is the interface those discussions assume as given.
