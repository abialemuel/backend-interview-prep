## Scalability and Load Balancing

This file covers the foundational mental model a backend engineer needs for the system design interview: how systems scale, how load is distributed, how to estimate capacity, and the theoretical limits (CAP/PACELC, consistency models) within which every real architecture lives.

### Vertical vs horizontal scaling

- **Vertical scaling (scale up)** adds resources to a single machine: more CPU, more RAM, faster disks, a larger instance class (e.g., RDS db.r6g.16xlarge). It is simple, preserves a single coherent state, and requires no code changes. It has two hard problems: there is a physical ceiling (you eventually run out of bigger boxes to buy), and a single machine is a single point of failure — a kernel panic takes the whole system down.
- **Horizontal scaling (scale out)** adds more machines behind a load balancer. It is the only model that scales without bound in principle, and it is fault-tolerant by construction (the loss of one node does not kill the service). Its cost is operational: state must be partitioned or replicated, requests must be routable, and you have to reason about distributed consistency.

A useful rule of thumb: scale vertically until you cannot, and scale horizontally before you have to. Most systems do both — a vertically scaled primary that handles writes, and a horizontally scaled fleet of stateless workers + read replicas that handle reads.

### Stateful vs stateless services

A service is **stateless** when any request can be served by any instance — no instance holds state the next request depends on. Sessions live in a shared store (Redis, a JWT, an encrypted cookie) rather than in process memory. Statelessness is the precondition for horizontal scaling: as long as your application server owns state, you cannot freely add or remove instances, and a deploy or resize risks dropping in-flight user sessions.

Stateful services (databases, caches, sticky-session web tiers, Kafka brokers) cannot be scaled by simply adding machines — you have to partition (shard), replicate, and route consistently. The first refactor most monoliths make is extracting session state out of the application process and into a shared store, precisely so the app tier can become stateless.

### The load balancer layer

A load balancer distributes incoming requests across multiple upstreams. It hides individual node failures from clients, enables zero-downtime deploys, and concentrates TLS termination.

**Layer 4 vs Layer 7:**
- **L4** operates on TCP/UDP (IP + port). It is fast, opaque to application protocol, and cheap — typically used for database or internal service traffic (e.g., AWS NLB, HAProxy in TCP mode). It cannot route by URL, header, or cookie, and cannot inspect the body.
- **L7** operates on HTTP/gRPC and can route by path, host, header, or cookie; can rewrite, terminate TLS, inject headers, and do content-based traffic shaping (e.g., AWS ALB, nginx, Envoy). It is more expensive and a sieben larger attack surface, but it is the standard for public web tiers.

**Algorithms:**
- **Round-robin** — cycles through upstreams in order. Simple; assumes uniform capacity.
- **Weighted round-robin** — assigns more requests to bigger/more powerful upstreams. Useful in heterogeneous fleets.
- **Least connections** — picks the upstream with the fewest in-flight requests. Best when request cost is variable (e.g., some endpoints are heavy).
- **IP/cookie hash** — consistent hashing by client identity, used to gain session affinity ("sticky sessions"). Useful when the upstream is stateful or when you want to maximize cache locality.
- **Random** — uniformly picks an upstream; cheap and statistically fair, but offers no affinity.
- **Power of two choices (p2c)** — pick two random upstreams and choose the lesser-loaded; dramatically better than pure random with minimal overhead. Used in production L7 LBs (Envoy, Finagle).

**Health checks and graceful deploys:**
- Active health checks (the LB polls `/health` and removes failing upstreams).
- Passive health checks (the LB counts upstream errors and ejects bad nodes for a window — Envoy's outlier detection).
- During deploy, drain nodes by removing them from the LB pool, waiting for in-flight requests to finish (graceful shutdown), then stopping the process. The application should listen for `SIGTERM`, stop accepting new work, finish current work, and exit. This avoids 502s and dropped long-poll connections.

### DNS, CDN, and the global load balancer

At the very top, **DNS** maps a name to one or more IPs. It is the first load balancer: simple round-robin across A records, or geo-routing via a managed DNS provider (Route 53 latency-based routing, geolocation routing). DNS is cheap, globally distributed, and cache-driven — which means it is also slow to change (TTL-bound) and a coarse instrument.

A **CDN** (CloudFront, Fastly, Cloudflare) pulls static and cacheable content to the edge, close to users. It cuts latency for the user, offloads the origin, and absorbs traffic spikes. TTLs and cache keys are the levers; **stale-while-revalidate** and **surge control** allow serving slightly stale content while refreshing in the background.

A **global load balancer** (Route 53, Cloudflare load balancer, Google's Maglev) routes a single anycast IP to the nearest healthy region, enabling multi-region active-active or active-passive failover.

### The database scaling path

This is the canonical path most OLTP systems follow when they outgrow a single MySQL box:

1. **Single primary.** All writes go to one node, reads too. Simple. Bounded by that box's CPU, RAM, IOPS, and connection pool.
2. **Read replicas.** Add read-only replicas that stream from the primary via binlog (MySQL) or WAL (Postgres). Reads scale horizontally; writes still do not. Replication is asynchronous, so replicas lag — the read path becomes eventually consistent. Route reporting/admin reads to replicas; keep user-facing read-after-write on the primary when you must show the user their own write.
3. **Sharding / partitioning.** Split the dataset by a shard key (user_id, tenant_id, geography). Now writes scale too, at the cost of cross-shard queries becoming expensive or impossible. Choose the key carefully — it is nearly impossible to change later. Common strategies: hash partitioning (even distribution, no range queries), range partitioning (range queries OK, hot spots possible), directory-based (a lookup service maps keys to shards).
4. **Denormalization.** Stop joining. Duplicate data into read-optimized tables (counter caches, materialized views, summary tables) so reads become single-shard lookups. Pay with write amplification and harder consistency maintenance.
5. **NoSQL / specialized stores.** Move hot workloads to purpose-built stores: DynamoDB for key/value at extreme scale, Elasticsearch for search, a column store (ClickHouse, BigQuery) for analytics, a graph store for relationships. The relational database still usually owns the source of truth for transactional writes; specialized stores own read paths.

### Caching layers

Caches appear at every layer of the stack; each layer absorbs a different kind of load:

```
Browser cache  <-- HTTP Cache-Control / ETag
   |
CDN edge       <-- static assets, cacheable API responses
   |
App-level      <-- in-process (process-local) cache for hot lookups
   |
Redis/Memcached <-- shared cluster, cross-instance
   |
Database       <-- buffer pool, query cache
```

- In-process caches (Laravel's `Cache::remember` with the array driver, Go's `sync.Map` or `groupcache`) are fastest and free, but they are unshared and per-instance, so writes only invalidate locally.
- A shared Redis cluster is the workhorse for distributed cache. It is usually the second-most-important component after the database, and the source of the most subtle production bugs (see `02-caching-and-microservices.md`).

### Asynchronous processing

Not everything must be done synchronously. Anything that is not on the user's critical path — sending a welcome email, generating a thumbnail, fan-out to a feed, third-party webhooks — should be queued and processed out of band, by worker processes. This decouples capacity from request duration: a burst of signups does not have to mean a burst of SMTP negotiations in the request handler.

Queues also enable retry, backpressure, and load shedding. A failed job retries with exponential backoff; an overloaded system can shed load by queueing rather than rejecting. Common choices: SQS (managed, simple), Redis Streams or lists (low operational cost), Kafka (durable logs, high throughput, ordered partitions), EventBridge (AWS-native event routing). A worker that fails a job permanently should publish to a dead-letter queue for inspection.

### Availability vs reliability

These terms are often used loosely; in an interview it pays to be precise.

- **Availability** is the fraction of time the system is able to serve requests. It is expressed in "nines": 99% (two nines) ≈ 7.2 hours downtime per month; 99.9% ≈ 43 minutes; 99.99% ≈ 4.3 minutes; 99.999% ≈ 26 seconds. Each additional nine costs roughly an order of magnitude more engineering.
- **Reliability** is the probability that the system performs correctly over a period — including correctness, not just reachability. A system available but returning wrong data is not reliable.
- **Durability** is the probability your data survives, even if the system is temporarily unavailable (e.g., S3's 11 nines is durability, not availability).

### The CAP theorem

CAP says that during a **network partition** (the P), a distributed system must choose between **consistency** (C — every read sees the latest write or an error) and **availability** (A — every request gets a non-error response, possibly stale). Because partitions are unavoidable in real networks, you cannot have all three; you pick a side:

- **CP** — refuse requests during a partition (e.g., HBase, ZooKeeper, etcd, a quorum-replicated MySQL primary failover that blocks until consensus).
- **AP** — keep serving, accept staleness (e.g., DynamoDB, eventually consistent reads, multi-region active-active with async replication).

In practice, **the partition is rare and the steady state is "no partition"** — which CAP does not describe. **PACELC** extends it: under a partition (P), you choose A or C; **else (E) you choose latency (L) or consistency (C).** This is the model that actually explains production choices. MySQL with synchronous replication: PC/EL during normal operation (low latency eventually sacrificed to replication quorum). DynamoDB: PA/EL — fast and available, eventually consistent. Spanner: PC/EC — globally strongly consistent, at a latency cost.

### The fallacies of distributed computing

The famous list (L. Peter Deutsch, James Gosling):

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every one of these will bite you. Builds queues, retries, idempotency, encryption, and timeouts around all of them.

### Back-of-the-envelope capacity estimation

The goal is ballpark, not precision. Some memorized numbers:

**Time constants:**
- 1 day ≈ 86,400 seconds
- 1 year ≈ 31.5 million seconds (31536000 ≈ π * 10^7, a useful mnemonic)

**Dean's latency numbers (approximate, single operation):**
- L1 cache reference: 0.5 ns
- Branch mispredict: 5 ns
- L2 cache reference: 7 ns
- Mutex lock/unlock: 25 ns
- Main memory reference: 100 ns
- Compress 1 KB with Zippy: 3 μs
- Send 1 KB over 1 Gbps network: 10 μs
- Read 4 KB randomly from SSD: 100 μs (≈ 0.1 ms)
- Read 1 MB sequentially from memory: 250 μs
- Round trip within a data center: 0.5 ms (500 μs)
- Read 1 MB sequentially from SSD: 1 ms
- HDD seek: 10 ms
- Read 1 MB sequentially from HDD: 20 ms
- Packet CA->Netherlands->CA: 150 ms

Memorize the **orders of magnitude**, not the exact digits: RAM is ~1000x faster than SSD; SSD is ~1000x faster than HDD; within-DC round trip is ~0.5 ms; cross-continental is ~150 ms.

**Quick rule of thumb for daily active users to RPS:**
- DAU * actions/day / 86400 = average RPS.
- Peak is typically 5-10x average for consumer products.

### Worked estimation: URL shortener

- 100M new URLs/month, 10:1 read/write ratio -> 1B reads/month.
- New URLs: 100M / 30 / 86400 ≈ 38 writes/sec average, peak ~400 writes/sec.
- Reads: 1B / 30 / 86400 ≈ 385 reads/sec average, peak ~4k reads/sec.
- Storage: assume 7-char base62 short code + 500-byte average long URL + 100 bytes metadata ≈ 600 bytes/entry. 100M/month * 600 B ≈ 60 GB/month, ~720 GB/year, ~7 TB over 10 years. Trivially fits on a single MySQL instance with a hot replica.
- Bandwidth: write 40 * 600B ≈ 24 KB/s; read 4k * 600B ≈ 2.4 MB/s — also trivial.
- Conclusion: at this scale the architecture is a single primary plus a few read replicas, fronted by a Redis cache (hit rate 90%+ because short links are read-heavy and skew hot). The interesting problems are the ID generation strategy (auto-increment, Snowflake, hash of URL, base62 encoding of the numeric ID), and avoiding collisions while keeping the code short.

### Worked estimation: news feed at moderate scale

- 10M DAU, 100 feed refreshes/day each, fetching ~50 posts per refresh.
- 10M * 100 = 1B feed fetches/day -> ~11500 RPS average, ~100k RPS peak (10x).
- Reads dominate at ~100:1 over writes (people scroll more than they post).
- Storage: assume post is 1 KB average, user posts ~2x/day -> 20M new posts/day -> 20 GB/day of post bodies, ~7 TB/year, ~70 TB over a decade. Manageable in sharded MySQL with media offloaded to S3.
- Fan-out: when a user posts, the post must appear in the feeds of followers. **Fan-out on write** (precompute each follower's feed at post time) is fast for reads but expensive for users with many followers (a celebrity with 10M followers writes one post -> 10M feed writes). **Fan-out on read** (compute the feed at fetch time by blending the timelines of followees) is cheap at write time but expensive and slow at read time. Hybrid: fan-out on write for typical users, fan-out on read for celebrities, with a counter cache tracking followers so the system can choose the path. Cache the precomputed feeds in Redis (sorted sets scored by post timestamp).

### Consistency models

- **Strong consistency (linearizability)** — every read returns the value of the most recent write; writes appear in a single total order. Costly: requires quorum or synchronous replication, and limits where you can place replicas.
- **Sequential consistency** — operations appear in some total order consistent with each client's program order, but not necessarily real-time order. Weaker, easier to achieve.
- **Causal consistency** — preserves the order of causally related operations; concurrent operations can be observed in any order. Captured by vector clocks or version stamps.
- **Eventual consistency** — given no new writes, all replicas converge eventually. No bound on how long "eventually" is until you add a freshness SLA. Dynamo-style systems.
- **Read-your-writes / session consistency** — a session never sees an older version of data the same session wrote. Easy to implement with sticky sessions or a session-scoped token/epoch.

In a typical app (Laravel + MySQL + Redis), the write path is strong (a single MySQL primary serializes writes), the read path is eventually consistent (read replicas lag), and the cache is eventually consistent with the DB (you invalidate on write, but there is always a window). Knowing where each boundary sits and being able to argue why it is correct for the use case is the actual interview skill.

### Proxy vs reverse proxy

- A **forward proxy** sits in front of clients (e.g., a corporate egress proxy, Squid, Tor). The client knows it is using a proxy; the server does not see the original client IP. Used for policy, anonymity, and caching of outbound traffic.
- A **reverse proxy** sits in front of servers (e.g., nginx, HAProxy, AWS ALB, Cloudflare). The client thinks it is talking to the server; the proxy hides upstream topology. Used for TLS termination, load balancing, caching, rate limiting, and routing.

A CDN is a special case of a reverse proxy that also replicates content geographically.

### Putting it together

A typical web request end-to-end:

```
Browser/Client
   -- DNS --> Route 53 (geo/latency routing)
   -- HTTPS --> CloudFront (CDN cache for static + cacheable API)
   -- HTTPS --> ALB (L7 LB, TLS termination, health-checked upstreams)
   -- HTTP --> app fleet (stateless Laravel/Go, auto-scaling group behind ALB)
   -- read --> Redis (shared cache, hot keys; misses fall through)
   -- read/write --> MySQL primary (writes) + replicas (reads)
   -- async job --> SQS/Kafka --> worker fleet
```

The interview questions about this layer are usually: why stateless? why L7 not L4? where is the consistency boundary? what happens if Redis dies? what happens if the primary dies? You should be able to answer each for the diagram above without hesitation.