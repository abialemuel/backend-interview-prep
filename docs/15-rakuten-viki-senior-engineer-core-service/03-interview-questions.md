# Rakuten Viki Interview Questions — Coding, Design, Behavioral

Graded question bank for the remaining loop. The coding section is weighted heaviest because the next live round after Codility will almost certainly be algorithmic live coding.

## Coding rounds

**What to expect.** A Codility-style live session: one to two problems in 45–60 minutes, in your best language (Go), with the interviewer watching for problem decomposition, edge cases, and testing discipline. Publicly reported Rakuten Viki questions on NodeFlair include implementing a **round-robin load balancer** and a **weighted round-robin load balancer** — scheduler/distribution-flavored problems. Expect medium LeetCode difficulty, occasionally design-a-class style.

### Q1 (reported pattern): Implement a round-robin load balancer

Given servers and a stream of requests, each request goes to the next server in cycle. Model answer in Go: an atomic counter over a fixed server list; `next := atomic.AddUint64(&i, 1) % len(servers)`. Discuss: O(1) per dispatch, no shared mutable state beyond the counter, behavior when a server is added/removed mid-flight (rebuild list, counter mod changes — mention consistent hashing as the upgrade path, see [consistent hashing](../13-distributed-systems/02-replication-and-partitioning.md)).

### Q2 (reported pattern): Implement a weighted round-robin load balancer

Same surface, each server has a weight. Two clean answers:

- **Smooth weighted round-robin** (nginx algorithm): each server has `currentWeight += weight` per dispatch; pick max; subtract `totalWeight` from the picked one. Produces interleaved, fair sequences. This is the answer that impresses.
- Expanded list (repeat each server `weight` times) + round-robin — simpler, correct, but memory-heavy for large weights; say both and the trade-off.

### Q3: Rate limiter — design a token bucket for 1,000 requests/min per user, across instances

Redis + Lua for atomic decrement-and-refill, or per-instance approximation with background sync. Follow-ups: burst handling, Redis failure policy (fail open vs closed), distributed double-counting on failover (see [13-distributed-systems/04-interview-questions.md](../13-distributed-systems/04-interview-questions.md) Q15).

### Q4: LRU cache with TTL

Go: map + doubly linked list, O(1) get/put; TTL adds a background sweeper or lazy expiration on read. Interviewers at media companies love this one — caching is the daily bread of a playback backend.

### Q5: Merge intervals / availability windows

Catalog rights are availability windows (title X licensed in SG from date A to date B). Classic interval-merge/overlap problems map directly; practice the family: merge, insert, count overlaps.

### Q6: Process tasks with dependencies and cooldown

Task scheduler with cooldown (LeetCode 621 family) — matches the scheduler flavor of reported questions. Explain the math (`(maxCount-1)*(cooldown+1) + occurrencesOfMax`) and the heap-based simulation.

### Q7: Concurrent-safe structures in Go

Write a worker pool with graceful shutdown (`context.Context`, `sync.WaitGroup`, channels); discuss when to use mutex vs channel, goroutine leaks (`context` cancellation, closing channels from sender side). This is standard Viki-adjacent because Core Service backends are long-running Go (or JVM) services.

### Practice list (2–3 days)

- Weighted/round-robin schedulers, rate limiter, LRU/LFU with TTL
- Interval problems (merge/insert/room scheduling)
- Top-K / heaps (trending titles), sliding window
- String processing (subtitle/text transforms — parsing, matching)
- Concurrency: worker pool, pipeline with backpressure ([Go deep dive](../01-languages/go/README.md))

## System design round

Short-form graded answers; the full walkthrough lives in [02-system-design.md](02-system-design.md).

### Q1: Design playback resolution: user clicks play → video starts

Walk: API gateway → playback service (session token auth) → entitlement check (cached, short TTL) → geo-rights check → produce signed URLs/manifest for CDN → client fetches segments. Mention DRM adjacency (Widevine/FairPlay license flow), playback start time as the north-star metric, and fail-closed for paid content.

### Q2: Design the entitlement check hot path

Redis-backed cache keyed by (user, title-class), TTL in seconds-to-minutes, revocation push for immediate cases, fallback to origin DB with circuit breaker. Correctness argument: worst case a revoked user watches minutes longer — business cost, acceptable; never block a paying user.

### Q3: Subscription webhooks — at-least-once delivery from app stores. Design the consumer.

Idempotency key from provider event ID, dedup window table, outbox pattern for downstream effects, ordered processing per user (partition key), DLQ + alerting for poison events. This is the Careem CERT story transposed — use it.

### Q4: Catalog is cached at three layers. A title is corrected. How does the fix propagate?

Layer-by-layer: CDN (purge by key/soft TTL), Redis (explicit invalidation message via pub/sub or versioned keys), in-process (short TTL bounds staleness). Discuss why versioned keys (`catalog:{id}:v{n}`) beat delete-on-write under races.

### Q5: Popular episode drops in 1 hour. What breaks?

Thundering herd on the single hot catalog/entitlement key (request coalescing, pre-warm before drop), origin DB read spike (cache warm >95% target), CDN egress surge (fine — that is its job), player beacon flood (queue + backpressure). Answer with the load ladder from [06-system-design/01-scalability-and-load-balancing.md](../06-system-design/01-scalability-and-load-balancing.md).

### Q6: Design subtitle delivery in 200+ languages

Subtitle track = versioned immutable object → CDN edge, content-hashed URLs. Write path: community submission → moderation state machine (event-driven, idempotent consumers) → publish event invalidates nothing (new version = new URL). Mention endangered-language scale is tiny but governance matters — shows product empathy.

## Behavioral round

STAR answers, anchored to your real stories; see [08-behavioral](../08-behavioral/README.md).

- **Why Viki / why this role** — use the drafted answer in [README](README.md); connect Core Service platform work to Telkom AI Proxy and Careem CERT.
- **Why Singapore / relocation readiness** — concrete start date, prior international-team experience, long-term commitment.
- **A production failure you owned** — CERT relay silent-event-loss elimination: detection (metrics gap), fix (error classification, retries, ordering gate), result (zero silent drops).
- **Conflict or influence without authority** — backend standards work at Careem; frame as persuasion with evidence, not mandate.
- **Efficiency win** — Telkom monitoring rebuild: stateless autoscaled Go, ~80% memory reduction; frame as ownership plus measurable result.
- **Mentorship** — how you level up engineers; ties to senior-level expectations in the loop.

## Questions to ask the interviewers

1. What does the Core Service team own end-to-end, and what is the current biggest technical pain?
2. How do services communicate internally — sync APIs, events, or both?
3. What does on-call look like, and how are postmortems handled?
4. How does the Singapore office work with Tokyo/San Mateo/Seoul — timezone overlap and decision-making?
5. What would the first 90 days of this role look like?

## Rehearsal plan

1. Code Q1–Q7 timed, in Go, talking aloud — one per day.
2. Whiteboard 02-system-design.md twice: once playback-centric, once entitlement-centric.
3. Behavioral: record yourself on the five answers; cut filler.
4. Re-read [11-api-design](../11-api-design/README.md) and [13-distributed-systems](../13-distributed-systems/README.md) the night before each technical round.
