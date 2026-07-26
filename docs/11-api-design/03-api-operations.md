# API Operations

Designing the happy-path contract is half the job. This file covers the operational half — what happens on retry, under abuse, across auth boundaries, and over years of evolution. In interviews, these topics are where senior candidates separate from mid-level ones: anyone can sketch `POST /orders`; fewer can explain what happens when the client sends it twice.

## Idempotency keys

The problem: POST is not idempotent, networks are unreliable, and the worst-case failure is invisible to the client — the request succeeded but the response was lost. The client's only safe move is to retry, and without protection the retry double-charges the card. (This is the client-facing twin of the queue-consumer idempotency discussed in the system design section.)

The industry-standard mechanism, popularized by Stripe:

1. The client generates a unique key per **logical operation** (a UUID minted when the user hits "Pay" — not per HTTP attempt) and sends it as a header: `Idempotency-Key: 6f0d...`.
2. Server-side, atomically claim the key before doing the work — an `INSERT` into a keys table with a unique constraint (or Redis `SET NX`), storing the key, a hash of the request body, and status `in_progress`.
3. Do the work; store the response (status code + body) against the key; mark it `completed`.
4. On a retry: key exists and is `completed` → **replay the stored response** without re-executing. Key exists and is `in_progress` → return `409 Conflict` (or wait briefly); the original request is still running, and racing it is exactly what the mechanism exists to prevent. Key exists but the request hash differs → `422`; the client is reusing a key for a different operation, which is a client bug worth failing loudly on.

```sql
CREATE TABLE idempotency_keys (
  key           VARCHAR(64) PRIMARY KEY,
  request_hash  BINARY(32) NOT NULL,
  status        ENUM('in_progress','completed') NOT NULL,
  response_code SMALLINT NULL,
  response_body JSON NULL,
  created_at    TIMESTAMP NOT NULL
);
```

Details that show depth:

- The claim must be in the **same transaction** as (or an atomic step ahead of) the business write — otherwise a crash between them leaves a claimed key with no effect, or an effect with no key, and the retry either loses the operation or duplicates it.
- Keys expire (24h is typical) to bound the table; scope keys per API key/tenant so clients cannot collide with each other.
- Recovery from crashes mid-operation needs a policy: an `in_progress` row older than a threshold is either resumed by a reconciler or failed so the stored response becomes an error the client can act on — leaving it in limbo forever blocks the client's key.
- The IETF is standardizing the header (`draft-ietf-httpapi-idempotency-key-header`); the semantics above are the de facto standard regardless.

## Rate limiting

Protects the platform from abuse, bugs (a partner's retry loop), and noisy neighbors; also enforces the tiers you sell. Know the algorithms and their trade-offs:

| Algorithm | Mechanism | Pros | Cons |
| --- | --- | --- | --- |
| Token bucket | Bucket of capacity B refills at R tokens/sec; each request consumes one | Allows bursts up to B while enforcing average rate R; two numbers express policy well | Slightly more state (tokens + last refill time) |
| Leaky bucket | Queue drained at fixed rate | Smooths output to a constant rate | Adds queueing latency; bursts wait |
| Fixed window | Counter per `(client, window)` | Trivial: `INCR` + `EXPIRE` | Up to 2x burst at window boundaries |
| Sliding window log | Timestamp per request, count in last N sec | Exact | Memory per request; expensive at scale |
| Sliding window counter | Weighted blend of current + previous window | Near-exact, cheap | Approximation (fine in practice) |

**Token bucket is the default answer** — bursts are legitimate client behavior (a page load fires ten calls), and the bucket forgives bursts while holding the average. Implementation at fleet scale: the counters live in Redis, updated by an atomic Lua script so multiple app/gateway instances share state without races:

```lua
-- KEYS[1]=bucket key, ARGV: rate, capacity, now_ms
local tokens = tonumber(redis.call('HGET', KEYS[1], 't') or ARGV[2])
local last   = tonumber(redis.call('HGET', KEYS[1], 'ts') or ARGV[3])
tokens = math.min(ARGV[2], tokens + (ARGV[3] - last) / 1000 * ARGV[1])
local allowed = tokens >= 1
if allowed then tokens = tokens - 1 end
redis.call('HSET', KEYS[1], 't', tokens, 'ts', ARGV[3])
redis.call('PEXPIRE', KEYS[1], 60000)
return allowed and 1 or 0
```

Decide the failure mode explicitly: if Redis is down, fail **open** (allow traffic — availability over enforcement) for most APIs, fail closed only where the limit is a security control. Layer limits: per-IP at the edge/CDN, per-API-key at the gateway, per-tenant and per-endpoint in the app. Distinguish **rate limits** (requests per second/minute — protects infrastructure) from **quotas** (requests per month — enforces the pricing plan); they are different mechanisms with different windows and different error UX, and conflating them produces support tickets.

Communicate limits in the contract — the emerging IETF standard (`draft-ietf-httpapi-ratelimit-headers`) and common practice:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 12
RateLimit-Limit: 1000
RateLimit-Remaining: 0
RateLimit-Reset: 12
```

Well-behaved clients read `Retry-After` and back off with jitter; your docs should say so, because the alternative is every rate-limited client retrying in lockstep — a thundering herd you created.

### Retry guidance you owe your clients

The flip side of rate limiting and idempotency is telling clients how to behave; put this in the docs and bake it into your SDKs so the default client is a good citizen:

- Retry on 429, 503, 5xx, and connection failures; never retry 4xx (the request will not become valid by repetition).
- Retry non-idempotent operations **only** with an idempotency key.
- Exponential backoff with full jitter, a bounded attempt count, and respect for `Retry-After` when present:

```text
sleep = random(0, min(cap, base * 2^attempt))   # full jitter: desynchronizes the herd
```

- A total time budget, so a retrying client eventually surfaces the failure instead of hiding a 10-minute outage inside one "slow" call.
- Pass the idempotency key through unchanged on every retry of the same operation — regenerating it per attempt silently defeats the entire mechanism.

## Authentication for APIs

Match the mechanism to the caller:

- **API keys** — a long random secret in a header (`Authorization: Bearer sk_live_...`). Simple to issue, use, and revoke; no expiry or token exchange dance. Right for server-to-server calls by known partners and for developer-facing products where onboarding friction matters (Stripe runs on API keys). Weaknesses: a bearer secret that never expires ends up in git and in logs — mitigate with prefixed keys (`sk_live_` makes secret scanners effective), storing only a hash server-side, per-key scopes, rotation support (two active keys during rollover), and last-used tracking.
- **OAuth2 client credentials** — the machine-to-machine grant: the client exchanges `client_id`/`client_secret` at the token endpoint for a short-lived JWT access token, then presents that. What it buys over raw keys: **short-lived credentials** (a leaked token expires in minutes), standardized **scopes**, central issuance/audit at the authorization server, and — because the JWT is signed — **local validation**: each service verifies the signature against the issuer's public keys (JWKS) without a per-request auth-service call. This is the norm for internal service-to-service auth and B2B platform APIs. Costs: an authorization server to run, token caching and refresh logic in every client, and revocation-before-expiry is genuinely hard (keep TTLs short instead).

```http
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=svc_orders&client_secret=...&scope=payments:read

HTTP/1.1 200 OK
{"access_token": "eyJhbGciOi...", "token_type": "Bearer", "expires_in": 900}
```
- **mTLS** — both sides present certificates; the client's identity is its cert. The strongest option: credentials never cross the wire as bearer secrets, and stolen tokens are useless without the private key. Right for zero-trust service meshes (Istio/linkerd issue and rotate workload certs automatically, which removes the historical pain) and high-stakes B2B like open banking. Rarely right for a public developer API — certificate lifecycle is real onboarding friction.
- The layered reality to describe: user-facing traffic uses OIDC/session auth at the edge; the gateway authenticates it and services communicate over mTLS (mesh-issued) plus a propagated, signed identity token; third-party server integrations use API keys or client-credentials JWTs with scopes. Always: authenticate at the edge, but **authorize at each service** — "the gateway checked" is not an authorization model.

## API gateways

The single front door for external traffic (Kong, AWS API Gateway, Envoy-based gateways, nginx): the cross-cutting concerns implemented once instead of in every service.

| Concern | What the gateway does |
| --- | --- |
| TLS termination | Terminates public TLS; re-encrypts or hands to the mesh internally |
| Authentication | Validates API keys/JWTs, rejects at the edge, injects verified identity headers downstream |
| Rate limiting | Per-key/per-IP token buckets before traffic touches a service |
| Routing | Path/host/header-based routing, canary and weighted traffic splits |
| Transformation | Header injection, light request/response reshaping, protocol translation (gRPC-JSON) |
| Caching | Response caching for cacheable GETs at the edge |
| Protection | WAF integration, payload size limits, IP allow/deny lists |

For interview purposes, the two failure modes to name: the gateway becoming a **business-logic dumping ground** (transformation Lua scripts nobody can test — keep it to cross-cutting policy), and the gateway as a single point of failure (it is on every request path; it must be horizontally scaled and boring). Distinguish from a BFF, which is per-frontend aggregation and shaping — the gateway is policy, the BFF is presentation. (The system design section covers this pairing in more depth.)

## Backward compatibility and deprecation

The prime directive of API work: **never break a client you have not warned.** That requires knowing precisely what "breaking" means.

Safe (non-breaking) changes — additive only:

- Adding a new endpoint, a new **optional** request field, or a new response field (clients must ignore unknown fields — state this in your docs).
- Adding a new enum value **only if** the enum is documented as open and clients were told to handle unknowns.
- Relaxing validation (accepting more than before).

Breaking changes — require a version or a migration:

- Removing or renaming anything (field, endpoint, enum value, header).
- Changing a type, format, or semantics (`amount` from cents to dollars is the career-limiting classic).
- Making an optional field required; tightening validation; changing error `type` URIs or status codes clients branch on; changing default sort order or pagination behavior.

Beware **Hyrum's Law**: with enough consumers, every observable behavior will be depended on — someone parses your error `detail` string, someone relies on field order. You cannot avoid this entirely; you contain it by making the contract explicit and testing against it.

A deprecation policy worth describing in an interview:

1. Announce with a migration guide and a date; set a real notice window (6-12 months for a public API).
2. Mark it in-band: `Deprecation: true` and `Sunset: <http-date>` response headers (RFC 8594), plus OpenAPI `deprecated: true` — dashboards and SDKs surface these automatically.
3. **Measure usage** — per-key metrics on the deprecated surface tell you exactly who to email; deprecation without measurement is a countdown to an outage you scheduled.
4. Apply brownouts near the end: deliberate short failures (e.g., 5 minutes of 410 per day) flush out consumers who ignored a year of emails, on your schedule instead of at final shutdown.
5. Sunset: `410 Gone` with a problem-details body linking the migration guide.

## Contract testing

Integration environments are slow, flaky, and combinatorially explosive; unit tests with mocked clients drift from reality. Contract testing fills the gap:

- **Consumer-driven contracts (Pact)**: each consumer records the exact interactions it depends on into a contract; the provider's CI replays every consumer's contract against the real service. The payoff is precision: the provider knows exactly which fields are load-bearing for whom, so "can I remove this field?" becomes a broker query instead of an all-hands email. Best fit: internal microservices, where you can enumerate consumers and make contract verification a deploy gate ("can-i-deploy").

```json
{
  "consumer": "checkout-web",
  "provider": "orders-api",
  "interactions": [{
    "description": "get an existing order",
    "providerState": "order ord_42 exists",
    "request": { "method": "GET", "path": "/orders/ord_42" },
    "response": {
      "status": 200,
      "body": { "id": "ord_42", "status": "shipped" }
    }
  }]
}
```

Note what the contract does *not* say: it lists only the fields checkout-web reads, so the provider is free to add fields, and removing any other field breaks no contract. Matching is by shape, not exact payload.
- **Spec-based testing**: validate provider responses against the OpenAPI spec (schema validation middleware in test and even production), fuzz against the spec (Schemathesis), and diff the spec in CI for breaking changes (`oasdiff`). Best fit: public APIs, where consumers are anonymous and the spec *is* the contract.
- The senior framing: contract tests do not replace integration tests; they replace the **combinatorial explosion** of end-to-end environment tests for the specific question "did I break my consumers?", and they move that answer from staging to CI.

## Observability for APIs

You cannot operate a contract you cannot see. The standard kit:

- **RED metrics per endpoint**: Rate, Errors (split 4xx vs 5xx — client errors are a signal about your docs and SDKs, server errors page someone), Duration as **percentiles, never averages** (p50/p95/p99 — the p99 is what your biggest customer's batch job experiences). Label by route template (`/orders/{id}`, not the raw path — raw paths explode cardinality), method, status class, and API version.

```text
http_request_duration_seconds_bucket{route="/orders/{id}", method="GET", status_class="2xx", le="0.1"}
http_requests_total{route="/orders", method="POST", status_class="5xx"}
```

- **SLOs on those metrics**: "99.9% of requests succeed and p99 < 500 ms over 30 days," with alerting on error-budget burn rate rather than on individual spikes — a burn-rate alert ("we are consuming 30 days of budget in 6 hours") catches both fast outages and slow bleeds, where a static threshold catches only one.
- **Correlation**: generate a request ID at the edge, return it in every response (`X-Request-Id` — and put it in error bodies, as in the RFC 9457 shape), propagate it via W3C `traceparent` through every downstream hop, and log it everywhere. Distributed tracing (OpenTelemetry) makes "why was this call slow" a lookup instead of an archaeology dig.
- **Structured access logs** with route, status, latency, API key/tenant ID, user agent, request ID — this is also the raw material for per-consumer usage metrics, which feed rate limiting tiers, deprecation targeting, and billing.
- **Per-consumer views**: an API product needs "which keys are hitting the deprecated endpoint," "whose error rate spiked after our deploy," and a status page with honest per-endpoint health.

## SDKs and developer experience

For an API-as-product, the SDK is where most of your contract meets most of your users, and it deserves a mention in any staff-level answer:

- **Generate, don't hand-write**: SDKs generated from the OpenAPI spec (or protobuf) in each supported language stay in lockstep with the contract; hand-written SDKs drift and become a second contract to maintain. The generated layer gets a thin hand-written ergonomic wrapper where the language idioms demand it.
- **Bake in the operational contract**: the SDK is where retries-with-jitter, `Retry-After` handling, idempotency-key generation, request IDs, and pagination iterators live — shipping them in the SDK means the median integration is well-behaved without reading the retry docs.
- **Version SDKs semantically and independently** of the API: an SDK major bump is a client-code break; an API version is a wire break. Conflating them confuses everyone.
- The rest of the DX surface — accurate reference docs generated from the same spec, a changelog clients can subscribe to, sandbox environments with test keys (`sk_test_`), and copy-pasteable curl examples for every endpoint — is not fluff; it is what determines whether integrations are built correctly, which in turn determines your support load and your error-rate dashboards.

!!! note "How this shows up in interviews"
    Operations questions are rarely asked as "explain token bucket" in senior loops — they arrive as follow-ups to your design: "what happens if the client times out and retries this POST?" (idempotency keys), "a partner starts hammering this endpoint" (rate limiting layers + headers), "how do you remove this field?" (deprecation policy + usage measurement + contract tests). Answering from the contract outward — what does the client see, what do the docs promise — is the habit that reads as senior.
