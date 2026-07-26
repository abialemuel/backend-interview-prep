## Caching and Microservices

Two areas where backend systems win or lose in production. Caching is where most outages hide; microservices are where most architecture questions end up. Both reward thinking in trade-offs rather than recipes.

---

# Caching

A cache is a smaller, faster store that sits in front of a larger, slower, more durable store. Its only purpose is to satisfy some reads without going to the source of truth. Every cache is a bet that **locality** (the same data is read again) or **skew** (a small set of keys dominates traffic) holds for your workload. If neither is true, a cache hurts more than it helps — it adds invalidation complexity and code paths without paying back.

## Why cache

Three reasons, and you should be able to name which one motivates any given cache:

1. **Latency.** A Redis lookup is microseconds; a warmed MySQL point query is 1-10 ms; a cross-region call is 100+ ms. The cache turns a 10 ms request into a 100 μs one.
2. **Load.** Caching absorbs reads so your database does not have to. At scale, the cache becomes the read path; the database is the write path and the source of truth. Losing the cache is often equivalent to losing the read path entirely.
3. **Cost.** Cache hardware (memory) is more expensive per GB than disk, but far cheaper per request served. Lower instance counts, lower DB instance class, lower cross-AZ bandwidth.

## Cache patterns (write/read interplay)

The four patterns differ in **when the cache is updated** and **who can read stale data**.

### Cache-aside (lazy loading)

The application is responsible for the cache. On a read: check the cache; on a miss, read the DB, write the value back to the cache, return. On a write: write to the DB, then **delete** the cache key (not update — see consistency below).

```
read  GET key --miss--> SELECT ... --> SET key value --> return
write UPDATE db --> DELETE key
```

Pros: cache only contains data that someone actually asked for (cold data does not pollute it); resilient to cache failure (the app falls through to the DB). Cons: first read after a write is a cache miss; the cache can be stale if the DB write succeeds and the cache delete fails; code complexity lives in the app.

### Read-through

The cache is the read interface. The application calls `cache.get(key)`; on a miss the cache itself loads from the DB transparently. Pros: the application code is unchanged on a miss; consistent semantics across services. Cons: the cache must know how to load (needs a cache provider/library); a cold cache must warm up before serving traffic. Often implemented with libraries like Laravel's `Cache::remember`, or Redis with a Lua loader.

### Write-through

On a write, the application writes to the cache synchronously, and the cache writes through to the DB (or the application writes cache+DB in one logical step). Pros: cache is never stale; reads always hit. Cons: write latency equals cache-write latency + DB-write latency (a write that took 5 ms now takes 15 ms); a cache failure implies a write failure — durability is now coupled to availability.

```
write SET cache value --> UPDATE db --> ack
        (cache is unavailable -> write fails)
```

### Write-behind / write-back

The application writes only to the cache and returns. The cache asynchronously flushes to the DB later (per-key, batched, time-based). Pros: write latency is cache-only (microseconds), even for a slow DB; absorbs write spikes. Cons: **durability is at risk** — a cache crash between the write and the flush loses data. Use only when writes are copious, individually low-value, and statistically re-sendable (counters, metrics, telemetry), or when combined with persistence/replication. The trade-off is explicit: throughput and latency vs. durability.

## Cache invalidation

Cache invalidation is famously one of the two hard problems in computer science (the other being naming, plus off-by-one errors). The core difficulty: the cache is a **second copy of the truth** that must be kept coherent with the first. Invalidation strategies:

- **TTL (time-to-live).** Each entry has an expiration. After TTL, the entry is treated as missing. Simple, eventually consistent bounded by TTL. Good default. The trade-off: a TTL long enough to hit well is too long to be fresh.
- **Event-driven / explicit invalidation.** On a write to the DB, publish an event or directly delete the affected cache keys. Correct immediately, but requires knowing which keys a logical write affects — non-trivial when one DB row feeds many cache keys (a user profile feeds `user:123`, `user:123:summary`, `post:456:author`). Tagging/grouping cache keys (e.g., Redis tags, a reverse index) is a known but heavy approach.
- **Versioned keys.** Embed a version or epoch in the key (`user:123:v17`). On a write, bump the version in a small lookup; new reads miss the old key and load the new one. Old keys age out by TTL. Side-steps the invalidation race entirely; cost is one extra lookup per cache key for the version.

### The invalidation race, and why "delete, don't update"

Updating the cache on DB write looks safe but is not. Whenever the cache and DB are updated through two different writes, an ordering mistake produces stale data forever. Consider:

```
DB write succeeds, cache update fails:        cache stale until TTL
DB write succeeds, cache update fails AFTER:    cache forever stale
```

Worse, with concurrent updates:

```
Thread A: UPDATE db SET x = 1 (commit)
Thread B: UPDATE db SET x = 2 (commit)
Thread B: SET cache x = 2
Thread A: SET cache x = 1         <-- cache now has STALE x = 1 forever
```

The standard mitigation is **delete-on-write, never update-on-write**. A delete is idempotent — ordering does not matter; the next read repopulates from the DB, which is the source of truth. Even delete is still subject to a race:

```
Cache miss. Read 1: SELECT x  (returns 5)
                    DB write x = 6
                    DELETE cache key           <-- bad ordering below
Read 2: missed too, SELECT x (returns 6)
Read 2: SET cache x = 6
Read 1: SET cache x = 5         <-- cache now STALE x = 5 (DB has 6)
```

This race is rare but real. Mitigations: delayed double-delete (delete, sleep ~100 ms, delete again); bounded TTL as a backstop; versioned keys that sidestep the race entirely.

## Cache failure modes (the part that matters in production)

These five names appear in interviews often. Learn which is which and the fix that fits each.

### Cache stampede / thundering herd

A hot key expires. A burst of concurrent requests all miss at once, all hit the DB simultaneously to repopulate, and the DB gets an order-of-magnitude spike it was not sized for. Each request repopulates the cache independently — wasteful and dangerous.

Fixes:
- **Mutex / lock.** The first request to miss acquires a per-key lock; other requests wait or serve the stale value while the lock holder repopulates. Implemented with Redis `SETNX` or a process-local mutex combined with a shared lock.
- **Probabilistic early expiration (XFetch).** Each request that hits a near-expiry key randomly decides (with low probability, inversely proportional to TTL remaining) to refresh early. The refresh load is smeared across the last ~10% of TTL rather than spiking at expiry. Cheap and very effective — used at Netflix, Redis's `redis-cell`, many Go libraries.
- **Request coalescing.** All concurrent requests for the same key share a single in-flight load; the result is broadcast. Implemented at the library level (e.g., Laravel's lock-able cache, Go `singleflight`).
- **Lock-aside + serve-stale.** On a miss, only one request repopulates; others serve the previous value if it is still present (you keep the old value behind a separate key for this).

### Cache penetration

Clients ask for keys that **never existed** (e.g., bad IDs, probing attacks). Every request misses the cache, hits the DB, returns nothing, and the cache has nothing to store — so the next identical request also misses and hits the DB. The cache does not help.

Fixes:
- **Cache negative results.** Store `null` (or a sentinel) for a short TTL — say 30 seconds to a minute — so repeated lookups of the same bad key hit the cache.
- **Bloom filter.** Maintain a Bloom filter of all valid keys. Incoming requests check the filter first; if it says "not present," respond 404 without touching the DB. The filter has false positives but no false negatives, so legitimate requests always pass through.

### Cache breakdown / hot key expiry

A single hot key expires and a burst of traffic immediately hits the DB. Mathematically identical to a stampede but on a single key, and the term usually implies the key was hot *before* expiry as well.

Fixes:
- **Mutex, as above** — only one request repopulates.
- **No expiry + async refresh.** The hot key never expires by TTL; instead a background job refreshes it on a schedule. The read path always serves from cache; the worst case is slight staleness bounded by the refresh interval. Suitable for a small set of well-known hot keys (config, top stories, leaderboard).
- **Logical expiration.** Store a logical expiry time inside the value; serve the value past logical expiry while one request triggers an async refresh — the "stale-while-revalidate" idea applied at the cache layer.

### Cache avalanche

Many keys expire at the same time (e.g., a deploy that warms the cache with all identical TTLs, or a batch warmup that all set TTL=3600 from the same wall clock). All of them miss simultaneously, overwhelming the DB.

Fixes:
- **Randomize TTLs.** Add a jitter of ±10-20% to every TTL so expiries spread out.
- **Warm-up.** Pre-populate the cache before exposing the system to traffic (deploy-time warmup script; on cache restart, scheduled warmup job).
- **Circuit breaker / shed on the DB.** If the DB starts failing under the surge, short-circuit; serve stale or error rather than piling on.

### Distinguishing them

They sound similar but differ in **why the misses happen**:
- Stampede = many concurrent misses on the same expiring key.
- Breakdown = a single hot key expired.
- Avalanche = many keys expired at the same wall-clock time.
- Penetration = the key never existed in the first place.

The fixes differ accordingly — you cannot fix avalanche with a bloom filter, and you cannot fix penetration by jittering TTLs.

## Cache consistency with the DB

Keeping cache and DB in sync is where most subtle bugs live.

### Dual-write pitfalls

Writing to both cache and DB directly is dangerous:
- **Atomicity.** A cache write can succeed while the DB write fails (or vice versa) — now the two disagree, with no transaction spanning them.
- **Ordering.** Updates to two different stores can interleave and leave the cache stale (as above).
- **Race on read-back.** Concurrent reads and writes can race and produce a stale cache value that lives until TTL.

### The transactional outbox

Instead of writing to the cache directly, write the DB update **and** a cache invalidation message **in the same database transaction**, into an outbox table. A separate process (or CDC stream from binlog) reads the outbox and emits invalidations to the cache. The DB is the source of truth; the outbox ensures the invalidation message is durably published exactly-once (well, at-least-once; consumers must be idempotent). This is the same outbox pattern used for event publishing in microservices (see below) — reusable.

```
BEGIN;
UPDATE products SET price = ? WHERE id = ?;
INSERT INTO outbox(event_type, payload) VALUES ('invalidate', 'product:123');
COMMIT;
```

A worker or Debezium-style CDC consumes the outbox and `DEL product:123`. The invalidation cannot be lost even if the cache is briefly down, because it is durably recorded.

## Multi-level caching

Real systems stack caches because each layer is cheap and fast for a different reason:

```
Browser           -- cache-control / ETag (per-user)
CDN edge          -- shared by region, low latency to user
Reverse proxy     -- nginx varnish cache for hot GETs
In-process cache  -- sync.Map / Laravel array cache, nanoseconds
Redis             -- shared cluster, microseconds
DB buffer pool     -- MySQL InnoDB, ~100 ns within hot pages
```

Each layer has a different invalidation scheme and consistency semantics. Read paths must be designed with a notion of **staleness budget** — how stale a value is allowed to be for this read type (an admin viewing their own write: 0 ms; a public ranking: 5 minutes is fine). When a read can tolerate more staleness, push it further down the stack.

## Consistency vs staleness trade-off

There is no globally consistent answer; it is per-use-case. Push the cache closer to the write for hot content that must be fresh (e.g., write-through for an account balance); push it further away for content where 5-30 minutes of staleness is acceptable (e.g., popular posts). Specify the staleness budget explicitly and design invalidation to honor it. Blindly caching everywhere with a single TTL is a recipe for outage.

## Write-through vs cache-aside latency

Short version: cache-aside is faster on the **write path** (one DB write vs cache+DB) but the first read after a write pays a miss. Write-through is faster on the **read path** (always a hit) but every write pays the cache round trip + the DB write. Choose based on your read/write ratio and write latency tolerance. For most CRUD web apps (heavy read, infrequent write), cache-aside is the default; for account balances or strongly consistent counters, write-through.

## Hot keys and big keys

- **Hot keys** are keys that receive disproportionate traffic. A single Redis shard has a hard throughput ceiling (CPU, single-threaded); a hot key can saturate one node while others sit idle. Mitigations: shard the hot key (e.g., `cache:leaderboard:0..9` — pick a suffix at random to spread writes), replicate reads across read replicas, push hot keys into in-process caches, or move them off Redis onto a specialized store.
- **Big keys** are oversized values (multi-MB JSON blobs, hash with millions of fields). They block the single-threaded Redis event loop on read/write, balloon network egress, and on eviction incur heavy memory churn. Mitigations: split into smaller keys, use top-N truncation, store pointers (e.g., the S3 path of a large blob) instead of the blob itself.

## CDN and browser caching

- **Browser caching** uses `Cache-Control` (max-age, public/private, no-cache, immutable) and `ETag`/`If-None-Match` for revalidation. Indicate `immutable` for fingerprinted assets that never change; that lets the browser skip the revalidation round trip entirely.
- **CDN caching** follows the **only-cacheable fallacy**: a CDN only helps if a response *can* be cached. Many APIs return personalized data and should not be cached at the edge. Cache static assets aggressively; cache public read APIs with care and short TTLs; never cache writes or personalized reads without explicit cache keys.

---

# Microservices

## Monolith vs microservices

A monolith is one deployable unit that contains all the code. Microservices split that code into many independently deployable units communicating over the network. Both are valid; the trade-offs are severe.

**Microservices buy you:**
- Independent deploys, independent scaling, independent tech stacks per service.
- Clearer ownership and team boundaries (Conway's law in reverse).
- Fault isolation — a bad deploy in one service does not necessarily take down others.
- Easier horizontal scaling of a specific subsystem.

**Microservices cost you:**
- Network as a failure mode — every call can fail, time out, or stall. You now need circuit breakers, retries, bulkheads, timeouts everywhere.
- Distributed transactions are hard; you move from local ACID to sagas.
- Eventual consistency between services is now visible to users (often need UI redesign around pending states).
- Operational complexity explodes — you need service discovery, distributed tracing, centralized logging, more deploy pipelines, multi-service CI/CD.
- Latency adds up; one user request that fans out to 20 services has 20 network round trips even in the happy path.
- Cross-cutting changes become hard — a schema migration that touches five services requires coordinated deploys.

**When NOT to use microservices:** when the team is small (the operational overhead exceeds the scaling benefit), when the system is not actually growing, when an organizational reorg would solve the perceived problem better than a technical split, when latency is the primary non-functional requirement and a single in-process call would do.

**The common advice: start monolith, extract when boundaries are stable and pressure is real.** You only learn correct service boundaries after living with the system for a while. Premature splitting locks in a guess. Use the **strangler fig pattern** to carve services out incrementally (see below).

## Service boundaries: DDD bounded contexts

Domain-Driven Design's **bounded context** is the unit of decomposition. Each context has its own ubiquitous language, its own model of the same reality. "Customer" means different things to Billing (identity + payment method) and to Shipping (address + delivery preferences) — so do not force them to share one `Customer` model. A microservice should own one bounded context.

Signs you have it wrong: every change requires coordination across multiple services (boundary is too split); teams keep asking for data they do not own (boundary is in the wrong place); a single service keeps growing without limit (you extracted too little).

Anti-pattern: a **distributed monolith** — services are deployed separately but coupled on data and schema such that any change requires coordinated deploys. You got the costs of microservices without the benefits.

## Inter-service communication: sync vs async

**Synchronous (REST, gRPC, GraphQL federation):**
- Simple mental model; immediate response.
- Couples the caller's latency to the callee's availability.
- A call chain N services deep multiplies failure probability exponentially.
- Easier to start with; hard to fix when the chain gets deep.

**Asynchronous (message broker — Kafka, SQS, EventBridge, NATS):**
- Caller publishes an event and returns immediately; consumer processes when it can.
- Decouples availability: consumer can be down without failing the producer.
- Enables fan-out: one event serves many consumers without the producer knowing who.
- Natural for things that are not on the critical path (notifications, audit, search indexing).
- Adds a whole new infrastructure (broker), and the harder problem of dealing with at-least-once delivery (idempotency) and out-of-order events.

In practice, user-facing critical-path logic is often synchronous (you cannot respond 200 to "place order" until you have accepted the order), while side-effects are asynchronous (confirmation email, inventory decrement, search re-indexing). The interesting designs push the **synchronous boundary as early as possible** — accept the order, capture intent, and trigger the rest asynchronously. Broker selection (RabbitMQ vs SQS vs Kafka vs NATS), delivery guarantees, and Kafka internals are covered in [`../10-messaging-and-event-streaming/`](../10-messaging-and-event-streaming/README.md).

## Distributed transactions: the saga pattern

A distributed transaction spans multiple services. **Two-phase commit (2PC)** is the classic answer: a coordinator asks every participant to prepare; if all prepare, the coordinator tells them to commit. It is slow, blocking, fragile to coordinator failure, and very hard to operate at internet scale. In practice it is avoided in favor of **sagas**.

A saga is a sequence of local transactions, each in one service. Each step has a **compensating action** that semantically undoes it. If step 4 fails, run the compensations of steps 1-3 in reverse. There is no global isolation — intermediate states are visible to readers, so you design for it.

**Orchestration:** a central orchestrator calls each step and decides compensations. Pro: explicit flow, easy to reason about; con: orchestrator becomes a smart hub, can become a god service, can become a single point of failure.

**Choreography:** each service reacts to events and emits the next event. Pro: no central coordinator, scales naturally; con: the flow is implicit — to understand "what happens on order" you must trace N event subscriptions; hard to add a step.

Most production systems use **orchestration for the main business workflow** (order, payment, shipping — Temporal, AWS Step Functions, Cadence) and **choreography for side-effects** (publish `OrderPlaced`, let whoever cares react). Both need idempotency. A fuller treatment of sagas alongside the other event-driven patterns lives in [`../10-messaging-and-event-streaming/03-event-driven-patterns.md`](../10-messaging-and-event-streaming/03-event-driven-patterns.md); the 2PC-vs-saga theory is in [`../13-distributed-systems/03-correctness-in-practice.md`](../13-distributed-systems/03-correctness-in-practice.md).

## Idempotency and retries

A client retrying a timed-out request is the rule, not the exception. If a request is **not idempotent**, you get double charges, double sends, duplicate rows. Idempotency is the property that calling the same thing twice has the same effect as calling it once.

Make retry safe:
- **Idempotency keys** (Stripe-style). Client generates `Idempotency-Key` per logical operation; server stores `(key, response)` and replays it on retries within a window.
- **Unique constraints / dedup tables.** A unique `(user_id, action, slot)` constraint lets a duplicate insert simply fail. Use `INSERT ... ON CONFLICT DO NOTHING` (Postgres) or `INSERT IGNORE` (MySQL) for at-least-once writes.
- **Version stamps / conditional writes.** `UPDATE ... WHERE version = ?` makes retries safe — only the first succeeds, subsequent ones affect zero rows.
- **Idempotency for effects outside the system of record** (sending email, charging a card) is harder and requires the key approach.

A queue delivers **at-least-once** (sometimes "exactly-once" within a narrow scope, but assume at-least-once end-to-end and design your consumers accordingly). Network failures produce the same duplicates. **Idempotent consumers are mandatory.** For the API-surface version of this (idempotency keys as a contract with external clients) see [`../11-api-design/03-api-operations.md`](../11-api-design/03-api-operations.md); for why exactly-once is a spectrum rather than a switch, see [`../13-distributed-systems/03-correctness-in-practice.md`](../13-distributed-systems/03-correctness-in-practice.md).

## The outbox pattern

Problem: service A wants to (a) write to its DB and (b) publish an event, atomically. Writing the DB then calling Kafka is not atomic — Kafka can be down, or the call can fail, or you crash between the two. The event never reaches anyone who needs it.

Solution: write the DB row **and** a row in an `outbox` table **in the same transaction**. A separate process (or binlog CDC like Debezium, or a worker polling the outbox) reads the outbox and publishes to the broker. The event cannot be lost as long as the DB transaction commits; it is delivered at-least-once (consumers must be idempotent).

```
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = ?;
INSERT INTO outbox(event_id, topic, payload)
  VALUES (?, 'PaymentProcessed', ?);
COMMIT;
-- a separate poller/CDC stream reads outbox rows
-- and produces to Kafka, then marks them as published
```

Side benefits: you get an event log for free, you can replay, and you avoid dual-write consistency bugs entirely. Industry standard. Implementation details (CDC with Debezium, outbox table hygiene, ordering across the relay) are in [`../10-messaging-and-event-streaming/03-event-driven-patterns.md`](../10-messaging-and-event-streaming/03-event-driven-patterns.md).

## API gateway and BFF

- **API gateway** is the single front door for external clients, sitting at the edge: TLS termination, authentication, rate limiting, request routing, response composition, response caching, transformation. Examples: Kong, AWS API Gateway, NGINX, Envoy.
- **Backend-for-frontend (BFF)** is a per-frontend backend: one for the web app, one for iOS, one for Android. Each BFF aggregates downstream services into a response shape tailored to that frontend, avoiding bloated one-size-fits-all APIs and over-fetching. The BFF is not business logic — it is aggregation and shaping. Eliminates the curse of mobile clients making 15 round trips to render one screen.

Trade-off: a BFF is more servers to maintain, but it pays for itself the moment a mobile team can ship without coordinating with the web team. Some shops run BFFs as Lua/glue/GraphQL in the API gateway itself. Designing the contracts these gateways expose — resource modeling, pagination, versioning, error shapes — is its own interview round at API-first companies; see [`../11-api-design/`](../11-api-design/README.md). The authentication and authorization the gateway enforces (OAuth2/OIDC, service-to-service identity) is covered in [`../12-security-and-auth/`](../12-security-and-auth/README.md).

## Service discovery

In a microservices world, services come and go (deploys, autoscaling, crashes). The address of `orders-service` is not a fixed IP. Service discovery answers "who do I call?" Two approaches:

- **Client-side discovery.** A registry (Consul, etcd, ZooKeeper, Eureka) is queried by the client, which load-balances across returned instances. More client complexity.
- **Server-side discovery.** A load balancer (often internal DNS or service mesh) hides instance churn from the client. The client just calls `orders-service`. AWS Internal ALB, Kubernetes `Service`, Envoy with xDS are examples.
- **Service mesh** (Istio, Linkerd, Consul Connect) adds sidecar proxies that handle discovery, mTLS, retry, and tracing without application code changes.

In practice, most 2026 deployments get server-side discovery for free from Kubernetes (`Service` + kube-dns), and a mesh only when mTLS or traffic-shaping requirements justify its operational weight — see [`../09-containers-and-kubernetes/03-kubernetes-advanced.md`](../09-containers-and-kubernetes/03-kubernetes-advanced.md).

## Resilience patterns

These are the patterns every cross-service call needs in production:

- **Timeouts.** Every outbound call must have a timeout. Default deadlines, propagated via context (gRPC deadline, Go `context.Context`, Request-Timeout header). No timeout -> a slow downstream can hold all your worker threads.
- **Retries with exponential backoff and jitter.** Retry, but slowly and randomly. Exponential backoff keeps retry rate manageable under load; **jitter** (random delay added) prevents all clients from retrying in lockstep ("thundering herd" of retries). Add a retry budget so a persistent failure does not amplify load.
- **Circuit breaker.** Stop calling a service that is failing — open the circuit, return immediately (or fall back to a degraded response or cached value). After a cooldown, allow a probe request; close the circuit if it succeeds. Prevents cascading failure when a downstream dies.
- **Bulkheads.** Isolate resources per dependency: separate thread/connection pools per downstream. A slow `payments` service cannot exhaust the connections `orders` needs to call `inventory`. Concept borrowed from shipbuilding — flood one compartment, the rest stay dry.
- **Graceful degradation / fallbacks.** When a dependency is unavailable, return a sensible alternative (cached data, partial response, default). Often more useful than a hard error to the user.
- **Load shedding.** Under overload, prefer rejecting outright to going slow for everyone. Respond 503 with `Retry-After` to a fraction of traffic; protect the rest.

These are usually combined: timeout < total request deadline; retry with backoff+jitter up to a budget; circuit breaker around groups of calls; bulkheads in the connection pool; shedding at the gateway.

## Observability

Three pillars:

- **Metrics** — aggregated numeric time series (QPS, latency p50/p95/p99, error rate, saturation). Use them for alerting and dashboards. RED (Rate, Errors, Duration) for services; USE (Utilization, Saturation, Errors) for resources.
- **Logs** — structured, discrete events. Indexed for search; rarely a primary alerting tool due to volume.
- **Distributed tracing** — **essential** in microservices. A single request fans out across many services; without a trace, a p99 spike has no answer. A trace ID is generated at the edge, propagated via headers (W3C `traceparent`), and spans are emitted per call. Tools: OpenTelemetry, Jaeger, Honeycomb, Datadog APM. If you can identify where a 200 ms slowdown came from via a flame graph, you stop guessing.

The rule: logs tell you what happened, metrics tell you something is wrong, traces tell you where.

## Data ownership per service

**Each service owns its own data. No shared database.** A service exposes its data only through its API. Another service that needs the data calls the API (sync) or subscribes to its events (async). Sharing a database is the #1 way to accidentally couple two "services" into one distributed monolith: schema migrations have to coordinate, tables grow columns for unrelated reasons, and ownership erodes.

If another service needs a synchronous view of your data, common approaches:
- Sync via API call (with cache).
- Project an internal event the consuming service keeps its own read model of (CQRS).
- Materialized views for read-only reporting.

## Event sourcing and CQRS (brief)

- **Event sourcing** stores the log of changes (events) as the source of truth, instead of a mutable current state. Current state is a projection folded from the events. Benefit: full audit history, replayable, easy to derive new read models. Cost: complexity, large event logs, eventual consistency of projections.
- **CQRS (Command Query Responsibility Segregation)** separates the write model (commands) from one or more read models (queries). Writes go to the canonical store; reads are served from precomputed projections optimized for the question. Justified when the read shape diverges dramatically from the write shape (e.g., leaderboards, search indices).

Both are powerful but introduce real complexity. Use them when the problem demands it; do not adopt them by default. Full treatment — event notification vs event-carried state transfer vs event sourcing, projections, schema evolution — is in [`../10-messaging-and-event-streaming/03-event-driven-patterns.md`](../10-messaging-and-event-streaming/03-event-driven-patterns.md).

## Strangler fig pattern

When migrating a monolith to microservices, do not rewrite all at once. Identify a slice, route that traffic out of the monolith and to a new service (an API gateway can intercept the relevant paths), shut down the now-dead code in the monolith, and repeat. Over months the monolith shrinks; services grow organically where the value is clear. Named for the strangler fig tree, which grows around a host tree until the host is no longer needed.

```
Client -> API gateway -> monolith      (start)
Client -> API gateway -> new service   (after first extraction)
Client -> API gateway -> multiple new services    (after many extractions, monolith trivial)
```

This avoids the "big rewrite" trap — large rewrites fail more often than they succeed.

---

## Putting it together: a realistic request flow

```
Mobile client
   |
   +-> API gateway (auth, rate limit, TLS, routing)
         |
         +-> BFF (per-app aggregation)
               |
               +-> sync: orders-service (Go, owns orders DB)
               +-> sync: inventory-service (Laravel, owns inventory DB)
               +-> async (outbox): publish 'OrderPlaced'
                                  |
                                  +-> payment-service (consumer, idempotent)
                                  +-> notification-service (email/SMS, queued)
                                  +-> search-indexer (denormalized read model)
   +-> cache (Redis) caches read-only compositions
```

Every solid arrow implies: a timeout, a retry policy with jitter, a circuit breaker, a trace span. Every queue implies an idempotent consumer. Every database transaction that must also publish an event uses an outbox. The interviewer who asks "what could go wrong here?" is asking whether you have internalized these defaults — they should be reflexive.