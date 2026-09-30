# Core Service System Design — Rakuten Viki Loop

This file walks a streaming-backend design you can adapt to whatever prompt the Viki design round opens with ("design our playback service", "design the catalog", "design subtitles delivery"). The structure matters more than the exact domain: clarify requirements, estimate, sketch the read and write paths, then defend the failure modes — that is the senior loop.

Deeper material for every topic referenced here lives in [06-system-design](../06-system-design/README.md), [03-databases](../03-databases/mysql/README.md), [10-messaging-and-event-streaming](../10-messaging-and-event-streaming/README.md), and [13-distributed-systems](../13-distributed-systems/README.md).

## 1. Clarify the problem

Sample prompt: *"Design the core backend services for a video streaming platform like Viki — catalog, playback, and entitlements."*

Requirements worth stating out loud:

- **Scale**: tens of millions of registered users; read-heavy by orders of magnitude (1 write per catalog edit, millions of reads).
- **Global audience, geo-restricted content**: rights differ per territory; a title available in SG may be blocked in EU.
- **Two business models**: SVOD (Viki Pass — subscription gating) and AVOD (free with ads). Entitlement checks and ad-serving both touch the hot path.
- **Latency budget**: playback start is the metric users feel. Manifest + first-segment delivery should be seconds; catalog browse single-digit hundreds of ms.
- **Community subtitles**: subtitle tracks created/updated continuously by volunteers in 200+ languages; delivery must be cheap and versioned.
- **Consistency semantics**: entitlement decisions must be *correct enough but fast* — an expired subscriber watching one extra episode is a business cost; blocking paying users is worse. Say this trade-off explicitly; interviewers listen for it.

## 2. Rough capacity estimate

- 70M registered users, assume 5M DAU, peak concurrency during a popular release ~1–2% of DAU streaming ≈ 50–100k concurrent viewers.
- Playback API QPS: each viewer fetches manifest + a handful of API calls per session → a few thousand QPS at peak, bursty at episode drops.
- Catalog reads: browses dwarf playback; cache hit ratio is the whole game (target >95%).
- Subtitle files: small (KBs), immutable per version → ideal CDN/edge material.
- Storage: metadata in relational DB (tens of millions of rows — trivial); video assets in object storage behind a CDN; telemetry is the firehose (player beacons) → event pipeline, not the OLTP path.

## 3. Service sketch

```
            ┌────────────┐        ┌──────────────┐
client ───▶ │  API GW /  │──────▶ │ Catalog svc  │──▶ MySQL (read replicas)
            │  Edge LB   │        └──────────────┘
            │            │        ┌──────────────┐
            │            │──────▶ │ Entitlement  │──▶ Redis (session/entitlement cache)
            │            │        └──────────────┘
            │            │        ┌──────────────┐
            │            │──────▶ │ Playback svc │──▶ signed URLs → CDN (video + subtitles)
            └────────────┘        └──────────────┘
                  │                       ▲
                  │ events (Kafka/SQS)    │
                  ▼                       │
            ┌────────────┐         ingestion pipeline
            │ Telemetry  │◀──── player beacons (QoS)
            └────────────┘
```

- **Catalog service** — relational schema (series/episodes/availability), read replicas, aggressive cache invalidation on publish. Geo-rights: a rights table keyed by (title, territory, window); resolve at read time, cache the *resolved* answer per territory.
- **Entitlement service** — the correctness-critical one. AuthN via session/OIDC token ([12-security-and-auth](../12-security-and-auth/README.md)); entitlement decision cached in Redis with a short TTL so revocation propagates in TTL-time. Webhooks from app stores / payment providers for subscription events must be idempotent (provider retries are a guarantee, not a possibility).
- **Playback service** — takes (user, title, device) → resolves entitlement + geo → requests/produces a signed manifest or signed CDN URLs (HLS/DASH; DRM license flow handled by dedicated infra, but know the shape: Widevine/FairPlay/PlayReady, license token bound to session). Never expose raw object storage.
- **Subtitles** — community pipeline: contributors submit → moderation/approval workflow (state machine) → versioned subtitle objects → CDN. Reads are immutable-per-version, so caching is trivial; the interesting part is the *write* path (conflicting edits, review queues, notification events) — a good place to show event-driven design with idempotent consumers ([10-messaging](../10-messaging-and-event-streaming/README.md)).
- **Telemetry** — player beacons (rebuffer %, bitrate switches, start time) through a queue into a stream/analytics store. Mention it even if not asked: it shows ops maturity and feeds QoS-driven decisions.

## 4. Cross-cutting decisions to volunteer

- **Caching layers**: edge/CDN for static and subtitle assets; Redis for hot catalog and entitlement; in-process cache with short TTL for the hottest keys. Know invalidation trade-offs cold ([06-system-design/02-caching-and-microservices.md](../06-system-design/02-caching-and-microservices.md)).
- **Rate limiting and abuse**: per-user and per-IP limits at the gateway; token bucket in Redis ([06-system-design](../06-system-design/01-scalability-and-load-balancing.md), [11-api-design/03-api-operations.md](../11-api-design/03-api-operations.md)).
- **Idempotency**: every mutating API (purchases, subscription changes) takes an idempotency key; providers and clients retry.
- **Ordering and exactly-once**: subscription events must not be lost or double-applied — at-least-once delivery + idempotent consumers + dedup windows ([13-distributed-systems/03-correctness-in-practice.md](../13-distributed-systems/03-correctness-in-practice.md)). This is the exact muscle you built at Careem; say so.
- **Multi-region / failover**: active-active reads, single-writer DB per data type, CDN as first line of resilience; degrade gracefully (free tier if entitlement service is down? no — fail closed on *paid* playback, fail open on catalog browse).
- **Stateless services** for horizontal autoscaling — your Telkom monitoring redesign is the natural story here.

## 5. Senior follow-ups to expect

1. *Entitlement cache TTL vs user experience* — reconcile eventual revocation with abuse: TTL of seconds-to-minutes plus a revocation push for immediate cases; acknowledge the residual window and why it is acceptable.
2. *Double-charge scenario* — payment webhook retried: idempotency key from provider, dedup table, answer like the [13-distributed-systems interview questions](../13-distributed-systems/04-interview-questions.md) double-charge pattern.
3. *Hot title drop* — thundering herd on cache miss for one key: request coalescing, pre-warming, jittered TTLs.
4. *CDN provider outage* — multi-CDN strategy, traffic steering, what breaks (signed URLs are per-CDN), DNS-level failover cost.
5. *Subtitles in 200 languages* — storage model (per-track versions), delivery cost (CDN), and the moderation workflow as an event-driven state machine.
6. *What would you measure first after launch?* — playback start time, rebuffer ratio, entitlement error rate; tie answers to the telemetry pipeline.

## Walkthrough habit

For every diagram you draw: name the consistency model of each store, name the failure mode of each hop, and name one metric per component. Senior candidates lose offers on depth, not breadth — the repo's [system design deep dives](../06-system-design/README.md) and [distributed systems](../13-distributed-systems/README.md) are the reference material for that depth.
