# Redis Persistence & Clustering

This file covers how Redis survives restarts (RDB, AOF, hybrid) and how it
scales out (Sentinel vs Cluster, replication) plus the operational concerns
that come up most in interviews: cache stampede, hot keys, big keys,
client-side caching, ACLs, TLS, monitoring.

---

## 1. Persistence

Redis is primarily an in-memory store, but offers two complementary
persistence strategies: **RDB** (point-in-time snapshots) and **AOF**
(append-only log of write commands). They can be combined.

### 1.1 RDB (Redis Database) snapshots

RDB produces a compact binary snapshot of the dataset at a point in time,
stored as `dump.rdb`.

- **Mechanism:** Redis forks a child process. The child writes the snapshot to
  a temp file and atomically renames it over the previous `dump.rdb`. Because
  the child shares copy-on-write memory pages with the parent, snapshotting
  does not block command execution on the parent except briefly for the fork.
- **Trigger:** time-based `save` rules, manual `BGSAVE`, `SHUTDOWN`, or
  replication-induced full sync.
  ```conf
  save 3600 1     # at least 1 change within 3600s → snapshot
  save 300 100
  save 60 10000
  save ""         # disable RDB
  ```
- **Pros:** compact file, fast startup (load one binary file), good for
  backups and disaster recovery.
- **Cons:** snapshots are minutes apart — on a crash you lose all writes
  since the last snapshot; `fork()` can briefly pause large instances (memory
  doubling concerns under copy-on-write).

```bash
BGSAVE             # background snapshot
SAVE               # blocking snapshot (do not use in production)
LASTSAVE           # unix time of last successful snapshot
DEBUG SLEEP 0      # (no-op placeholder — avoid DEBUG in prod)
```

### 1.2 AOF (Append-Only File)

AOF logs every write command (after it executes), so replaying the log
reconstructs the dataset.

- **`appendfsync` policy** controls durability vs performance:
  - `always` — `fsync` after every command. Safest, very slow (disk-bound).
  - `everysec` — `fsync` once per second. **Common default.** Worst-case loss
    is up to ~1 second of writes. Background thread does the fsync, so the main
    thread is not blocked.
  - `no` — never `fsync` explicitly; rely on the OS. Fastest; can lose
    everything in the OS page cache on a crash.
- **AOF rewrite:** the log grows forever; Redis periodically rewrites it in
  the background, producing a minimal AOF that produces the same in-memory
  state. Triggered by `auto-aof-rewrite-percentage` and
  `auto-aof-rewrite-min-size`, or manually with `BGREWRITEAOF`. Like RDB, this
  uses a forked child.
- **Multi-part AOF (7.0+):** the AOF is no longer one giant file. It lives in
  a directory (`appenddirname`, default `appendonlydir/`) as a **base file**
  (RDB or AOF format snapshot) plus **incremental files**, tied together by a
  **manifest**. A rewrite just produces a new base + fresh increment and
  updates the manifest — no more double-buffering all writes in memory during
  the rewrite, which was a real spike source in 6.x.
- **Pros:** much lower data loss window (≤1 s with `everysec`); replay
  semantics are clear.
- **Cons:** larger files than RDB, slower replay at restart (commands executed
  one at a time, mitigated by the rewrite), continuous write-amplification on
  disk.

```conf
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

```bash
BGREWRITEAOF
CONFIG SET appendonly yes
```

### 1.3 Hybrid RDB+AOF (Redis 4.0+)

When both RDB and AOF are enabled, the AOF rewrite produces a file that
**starts with an RDB-formatted snapshot** followed by incremental AOF
commands. This combines fast restart (RDB preamble) with low data loss (AOF
tail).

```conf
aof-use-rdb-preamble yes      # default in 7.x
appendonly yes
appendfsync everysec
save 3600 1
```

### Which to choose?

| Workload                              | Recommendation                              |
|---------------------------------------|---------------------------------------------|
| Pure cache (rebuildable)              | RDB only, or nothing (`save ""`).           |
| Session store / durable cache         | AOF `everysec`.                             |
| Strict durability                     | AOF `always` (rare, slow) or AOF + replica. |
| General-purpose with low RPO          | Hybrid: RDB + AOF `everysec`.               |
| Backup/DR off-instance                | RDB to disk + periodic archive.             |

AOF `everysec` is the sweet spot for almost all real-world deployments:
negligible perf cost and at most ~1 s of data loss.

---

## 2. Replication

Redis replication is **asynchronous**: the primary streams commands to
replicas without waiting for acks (with `WAIT` you can opt into a synchronous
wait for N replicas to have acked a write — but this is an application-level
request, not the default).

- A replica connects to a primary with `REPLICAOF host port` (formerly
  `SLAVEOF`). On connection:
  1. Primary does a `BGSAVE` and streams the resulting RDB to the replica
     (**full resync**).
  2. During the RDB transfer, the primary buffers new write commands in the
     **replication backlog** (a circular buffer sized by
     `repl-backlog-size`, default 1 MB; increase for high write rate / slow
     replicas).
  3. Replica loads the RDB, then receives the backlog via the **command
     stream**, catching up to live state.
- After initial sync, the primary continues sending write commands. If the
  replica disconnects and reconnects within the backlog window, a **partial
  resync** happens — no full RDB needed. Otherwise, full resync again.
- Replicas are read-only by default (`replica-read-only yes`), allowing
  reads to scale horizontally but writes always go through the primary.
- Replication is **single-leader**: a replica can itself be a primary of
  another replica (chained replication), but the topology is still a tree.

```bash
REPLICAOF 10.0.0.5 6379
INFO replication           # role, master_host, master_link_status, offset
```

### Caveats

- Async replication means on a primary crash, the most recent few writes may
  not have reached any replica → **data loss**. This is what Sentinel/Cluster
  failover must trade off.
- Replicas do not count toward durability unless your application uses
  `WAIT N <timeout-ms>`.

---

## 3. Redis Sentinel (HA for single-primary)

Sentinel is a set of separate processes that provide **high availability** for
a single primary + its replicas. It does **not shard data**.

Responsibilities:

1. **Monitoring** — Sentinels ping primary, replicas, and each other.
2. **Automatic failover** — if the primary is down, Sentinels promote a
   replica to primary and reconfigure other replicas to follow it.
3. **Notification** — can call scripts / notify clients via pub/sub
   (`+sdown`, `+odown`, `+switch-master`, etc.).
4. **Configuration provider** — clients ask Sentinel "who is the current
   primary for `<master-name>`?" rather than hardcoding the primary's address.
   On failover, clients pick up the new primary by querying Sentinel again.

### Quorum and split brain

- `quorum` is the number of Sentinels that must agree a primary is down before
  triggering failover. Typically set to `N/2 + 1` (e.g., 2 of 3).
- Two-stage detection: **subjective down (SDOWN)** by one Sentinel → **objective
  down (ODOWN)** once quorum is reached.
- Only one Sentinel runs the failover at a time (elected via Raft-like leader
  election among Sentinels).
- Recommended topology: at least **3 Sentinel nodes** (or 5 for larger) and
  at least one replica, spread across failure domains. Avoid 2-node Sentinel
  clusters (no majority possible if one is lost).

### Typical config

```conf
sentinel monitor mymaster 10.0.0.5 6379 2     # quorum 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 30000
sentinel parallel-syncs mymaster 1
```

### Client usage

Clients connect to Sentinel on port 26379, request the current primary:

```bash
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
redis-cli -p 26379 SENTINEL masters
```

After failover the client library (most do this automatically) reconnects to
the new primary returned by Sentinel.

Use Sentinel when you want HA on a single dataset that fits on one node.
Use Cluster when you need to **shard** across multiple nodes.

---

## 4. Redis Cluster

Redis Cluster shards data across many primary nodes, each owning a subset of
the keyspace, with optional replicas per primary.

### Hash slots

- The keyspace is partitioned into **16384 hash slots** (not 65536 — a
  deliberate size to keep gossip messages small while still allowing thousands
  of nodes).
- The slot for a key is `CRC16(key) mod 16384`. CRC16 is used because it is
  fast and good enough for this purpose.
- Each primary node owns a contiguous or arbitrary range of slots; the union
  covers all 16384.
- **Key tags:** if a key contains `{...}`, the substring inside the braces is
  used for hashing instead of the full key. `user:{1001}:cart` and
  `user:{1001}:profile` both hash to the slot of `1001` and live on the same
  node — enabling multi-key operations on them.

```bash
CLUSTER KEYSLOT user:{1001}:cart          # → some slot number
CLUSTER KEYSLOT user:{1001}:profile       # → same slot
```

### Multi-key operations

- Operations involving multiple keys are only allowed when **all keys hash to
  the same slot**. Cross-slot `MGET`, `MULTI` transactions, Lua scripts that
  touch multiple keys, etc., will return a `CROSSSLOT` error.
- Use key tags (`{tag}`) to force related keys onto the same slot.
- `SELECT` is **not supported** in cluster mode — only DB 0. (Valkey 9.0
  lifted this — it supports multiple databases in cluster mode; Redis has not.)
- Classic Pub/Sub in a cluster broadcasts every message to **every node**,
  which does not scale. **Sharded Pub/Sub** (7.0+, `SPUBLISH`/`SSUBSCRIBE`)
  routes a channel to the node owning its slot, so pub/sub throughput scales
  with the cluster like keys do.

### Topology and gossip

- Cluster nodes gossip via a binary protocol on the cluster bus port
  (`port + 10000` by default, e.g., 16379 for a 6379 data port). They exchange
  ping/pong packets containing information about a subset of nodes, which
  spreads cluster state across the cluster.
- Each shard should have at least one replica. Recommended minimum: **3
  primaries + 3 replicas = 6 nodes** for a production cluster (no single
  failure domain owns a shard without replica).
- Clients are **cluster-aware**: most modern libraries (Lettuce, StackExchange,
  Jedis, go-redis, redis-py cluster) maintain a slot→node map and route
  commands to the right node, plus `MOVED`/`ASK` redirection handling.

### Failover

- Nodes detect each other via gossip; a primary is marked `PFAIL` (possible
  failure) then `FAIL` once a majority of masters agree.
- A replica of the failed shard then elects itself (similar to Raft leader
  election) and takes over its slots. Clients receive `MOVED` redirections on
  the next request and update their maps.
- During failover the affected slot range is briefly unavailable.

### Common operations

```bash
# Create a 6-node cluster (3 primaries, 1 replica each)
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
    127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
    --cluster-replicas 1

CLUSTER INFO                              # cluster_state: ok
CLUSTER NODES                             # full topology
CLUSTER SLOTS                             # slot → node mapping
CLUSTER COUNTKEYSINSLOT 12345             # keys in a slot
CLUSTER KEYSLOT somekey                   # slot for a key
CLUSTER GETKEYSINSLOT 12345 10            # sample keys in slot
CLUSTER FAILOVER                           # on a replica: trigger failover
```

### Cluster vs Sentinel

| Property            | Sentinel                       | Cluster                              |
|---------------------|--------------------------------|--------------------------------------|
| Sharding            | No (single dataset)            | Yes, 16384 hash slots                 |
| HA                  | Yes, automatic failover         | Yes, automatic failover per shard     |
| Multi-key           | No restrictions                 | Only same-slot keys (use `{tag}`)     |
| Databases           | 16 selectable                   | Only DB 0                             |
| Client complexity   | Sentinel-aware client           | Cluster-aware client (more state)     |
| Recommended nodes   | 3 Sentinels + 1 primary + ≥1 replica | 3 primaries + 3 replicas minimum |
| Use case            | HA for a small dataset that fits on one node | Horizontal scale-out            |

### Resharding note (2026)

Moving slots between nodes in OSS Redis Cluster is a key-by-key `MIGRATE`
orchestrated by `redis-cli --cluster reshard` — slow for big slots, and
multi-key ops on a migrating slot bounce between `ASK` redirections. Valkey
9.0 added **atomic slot migration** (slot ownership flips atomically after a
background copy), one of the first significant post-fork divergences worth
naming in an interview.

---

## 4b. Geo-distribution: Active-Active (CRDT) in Redis Enterprise

Everything above is single-writer per dataset/shard. For multi-region
deployments where each region must accept **local writes** with local latency,
Redis Enterprise (and its cloud offering) provides **Active-Active databases**
built on **CRDTs** (conflict-free replicated data types):

- Each region runs a full copy; writes are accepted locally and replicated
  asynchronously to peers.
- Conflicts are resolved by data-type-specific merge semantics, not
  last-writer-wins on the whole key: counters merge as the sum of per-region
  increments, sets as observed-remove sets, etc. Where semantics can't merge
  (e.g., two `SET`s of the same string), LWW applies.
- The result is strong *eventual* consistency: all regions converge, but a
  read may not see another region's most recent write — so it suits sessions,
  counters, caches, presence; not global uniqueness or inventory invariants.

This is an interview topic in two shapes: "how would you run Redis across two
regions?" (answer: OSS Redis gives you only single-writer + cross-region
replicas or client-side routing; true multi-writer needs CRDT-based
Active-Active or an app-level design) and "how do CRDTs resolve conflicts?"
(per-type merge functions with provable convergence). OSS Redis/Valkey have
no CRDT mode — be precise about that boundary.

---

## 5. Operational concerns

### 5.1 Cache stampede / thundering herd

A cache stampede happens when a hot key expires and many requests simultaneously
miss the cache, all hit the backend (DB / expensive API), and re-populate the
cache redundantly. Variants: "cache breakdown" (single hot key expires),
"cache avalanche" (many keys expire at once, often due to a uniform TTL set
during a bulk load).

Defenses:

1. **Mutex / distributed lock** — first miss acquires `SET NX EX` and computes;
   other requests wait briefly and re-read the cache, or return a stale value:
   ```bash
   SET cache:lock:hotkey 1 NX EX 5     # winner computes; losers retry after sleep
   ```
2. **Request coalescing / single-flight** — at the application layer, collapse
   concurrent identical requests into one backend call (Go's `singleflight`,
   library-level coalescing).
3. **Probabilistic early expiration (XFetch algorithm)** — each reader has a
   small probability of treating the cached value as already expired and
   refreshing it *before* the real TTL hits, smoothing the expiry spike:
   ```
   early = TTL * (random() % beta) * (-log(-log(random()))) + base
   ```
   With small `beta` (~5%), most clients still serve cache hits, only a
   random few start refreshing early.
4. **Randomized TTLs** — for bulk-loaded keys, set TTL = base ± jitter so they
   don't expire simultaneously (defends against avalanche).
5. **Stale-while-revalidate** — keep the old value, return it immediately, and
   refresh in the background.

### 5.2 Hot keys

A hot key is one that receives a disproportionate fraction of traffic, on a
single shard. Symptoms: one node CPU-bound, others idle.

Finding them:
- `--hotkeys` option on `redis-cli --memkeys` / `redis-cli --hotkeys` (requires
  `LFU` eviction policy to have meaningful sampling).
- `INFO commandstats` to see command frequency.
- `MONITOR` in dev only — never in production, it's a firehose.
- `LATENCY DOCTOR` for hints on what's slow.
- Cloud providers often expose hot-key dashboards (ElastiCache, MemoryDB,
  Upstash, etc.).
- Application-level sampling / metrics.

Handling them:
- **Copy the value to N random key variants** and have clients pick one at
  random — distribute reads across N keys on different slots.
- **Client-side caching** of the hot key to drop read rate on Redis.
- **Read replicas** of the hot shard if the cluster lets you read from
  replicas (`READONLY` mode in cluster).
- **Application-side caching / memoization** for the most extreme cases.

### 5.3 Big keys

A big key is one whose value (or set of values) is unusually large — e.g., a
hash with 1M fields, a list with 5M entries, a 100 MB string.

Why they hurt:
- Single `DEL` blocks the main thread (use `UNLINK` for async deletion).
- Single `HGETALL` / `LRANGE 0 -1` / `SMEMBERS` returns huge payloads, blocking
  the network and the client.
- Eviction picks them up rarely (sampling), so they sit and consume memory.
- Migration and resharding (`MIGRATE`) of big keys is slow and can time out.

Finding them:
```bash
MEMORY USAGE mykey SAMPLES 0     # full estimate in bytes
redis-cli --bigkeys              # scans DBs for biggest keys of each type
redis-cli --memkeys              # similar, focused on memory usage
MEMORY STATS
SCAN 0 COUNT 1000                # walk keyspace without KEYS
OBJECT ENCODING mykey
DEBUG OBJECT mykey               # serializedlength, type, refcount
```

Avoiding them:
- **Don't let a single collection grow unbounded.** Use `LTRIM`, `ZREMRANGEBYRANK`,
  `XTRIM`, or sharding by `{tag}` to split a logical collection across keys.
- **Bloom filters / HyperLogLog** instead of Sets for membership/cardinality
  at scale.
- **Asynchronous deletion** of large values: `UNLINK`, `FLUSHDB ASYNC`,
  `lazyfree-lazy-eviction yes`, `lazyfree-lazy-expire yes`,
  `lazyfree-lazy-server-del yes` in `redis.conf`.

### 5.4 Client-side caching (tracking, since Redis 6.0)

Redis 6 introduced **server-assisted client-side caching**. The server tracks
which clients cached which keys and sends **invalidation messages** when a key
changes, so clients can keep a local cache without going stale.

Two modes:

- **Default / "push" mode** — the client uses `CLIENT TRACKING ON`. The server
  remembers (per client, per key) what it has read. When a key is modified, the
  server pushes an invalidation message to that client. Lower bandwidth, but
  the server keeps state.
- **Broadcasting mode** — `CLIENT TRACKING ON BCAST PREFIX user:`. The client
  subscribes to *prefixes*; the server notifies all clients that match a
  prefix on any write to a key under that prefix. No per-key tracking, more
  invalidations, but no per-key state on the server.

```bash
CLIENT TRACKING ON                    # default push mode
CLIENT TRACKING ON BCAST PREFIX user:
CLIENT TRACKING OFF
CLIENT NO-EVICT ON                    # optional: don't get disconnected when evicting
```

Invalidation messages arrive as **RESP3 push messages** (clients connect with
`HELLO 3`); on legacy RESP2 they are delivered via a special pub/sub channel
(`__redis__:invalidate`) with a `REDIRECT`ed connection.

Use cases: read-heavy hot keys where round-trips to Redis dominate; cache the
result locally and trust Redis to invalidate. Combined with connection
pooling this can take read rate down by orders of magnitude. The client
library must support tracking (most major ones do now).

### 5.5 Connection pooling

Each Redis command is one request/response on a TCP connection. Opening a new
connection per request is extremely wasteful (TCP handshake + auth + SELECT).
Use a pool that keeps connections alive and reuses them.

- Typical pool size: 10–50 connections per application process; scale up only
  if you saturate them.
- Long-lived idle connections can be killed by `timeout` on the server; set a
  pool-level ping/keepalive.
- Use pipelining over pooled connections to amortize RTT further.
- In cluster mode, libraries maintain a pool *per node*, not per cluster.

### 5.6 Useful introspection commands

```bash
INFO                        # server, clients, memory, persistence, stats, replication, cpu, keyspace
INFO memory                 # used_memory, used_memory_rss, mem_fragmentation_ratio, maxmemory
INFO replication            # role, connected_slaves, master_link_status
INFO commandstats           # per-command call count and total CPU
CLIENT LIST                 # all connected clients with their info
CLIENT KILL ID <id>
CLUSTER NODES               # one line per node with id, ip:port, flags, slots, master link
CLUSTER INFO                # cluster_state, slot coverage, failover state
CONFIG GET maxmemory
CONFIG SET maxmemory-policy allkeys-lru
SLOWLOG GET 10              # last 10 slow commands (above slowlog-log-slower-than μs)
SLOWLOG RESET
LATENCY LATEST              # recent latency spikes per event
LATENCY DOCTOR              # human-readable analysis
MEMORY DOCTOR               # hints on memory issues
DEBUG SLEEP 1               # (dev only — blocks server for 1s)
```

### 5.7 Monitoring essentials

- **INFO** polled every N seconds — covers most of what you need for
  long-term trending.
- **SLOWLOG** — every command that exceeds `slowlog-log-slower-than`
  (default 10 ms) is recorded with arguments and duration.
- **LATENCY** framework — Redis samples notable latency events (fork, expire,
  aof-fsync-always, etc.) so you can correlate spikes with causes.
- **`MONITOR`** — live stream of all commands. Use only in dev / on isolated
  replicas; in prod it degrades performance and leaks data.
- **latency monitoring via replication lag**: `INFO replication` on a replica
  shows `master_link_status:up` and offset delta — large delta = the replica
  is falling behind.
- External: Prometheus exporter (`redis_exporter`), Grafana dashboards, APM
  spans on Redis calls.

### 5.8 TLS (since 6.0)

Redis supports TLS on the main port and the cluster bus. Configure with
`tls-port`, `tls-cert-file`, `tls-key-file`, `tls-ca-cert-file`, and the
usual cipher controls. Both client↔server and server↔replica traffic can be
encrypted.

```conf
port 0                 # disable plaintext
tls-port 6379
tls-cert-file /etc/redis/redis.crt
tls-key-file  /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes    # require client cert
```

```bash
redis-cli --tls --cert redis.crt --key redis.key --cacert ca.crt
```

Cluster bus and Sentinel traffic also support TLS (Sentinel since 6.0).

### 5.9 ACL (since 6.0)

Redis 6 introduced **Access Control Lists** — named users with permissions at
the command (and key-pattern) level.

```bash
ACL WHOAMI                                 # current user
ACL LIST                                   # all users and their rules
ACL SETUSER readonly on >somepass ~* +@read -@dangerous
ACL SETUSER alice on >alicepass ~user:* +get +hget +hgetall
ACL GETUSER alice
ACL DELUSER alice
```

Rule syntax highlights:
- `on` / `off` — enable/disable.
- `>pass` / `<pass` — add/remove a password. `nopass` to allow no-password.
- `+command` / `-command` — allow/deny a single command.
- `+@category` / `-@category` — by category (`read`, `write`, `admin`,
  `dangerous`, `slow`, `fast`, …).
- `~pattern` — allowed key patterns; `allkeys` for any; `resetkeys` to clear.
- `&pattern` — allowed Pub/Sub channel patterns.
- `reset` — wipe the user's rules.

ACLs are persisted via `ACL SAVE` to `users.acl` (configured with
`aclfile`). They replace the old single-password model and are essential when
multiple applications share one Redis.

### 5.10 Cache penetration / breakdown / avalanche

Three related failure modes worth distinguishing:

- **Cache penetration** — queries for keys that *don't exist in the DB
  either* (malicious or buggy). Every miss hits the DB and is never cached.
  Defenses:
  - Cache **empty/null results with a short TTL** so repeated misses are
    served from cache.
  - **Bloom filter** in front of the DB: if the key is "definitely not in DB",
    return immediately without touching the DB.
  - Rate-limit / block clearly invalid queries.

- **Cache breakdown** — a single **hot** key expires and many concurrent
  requests flood the DB at once. Defenses: distributed mutex (see §5.1),
  probabilistic early expiration, never-expiring cache + background refresh.

- **Cache avalanche** — **many** keys expire simultaneously (often because
  they were loaded with the same TTL), overwhelming the DB. Defenses:
  randomized TTLs with jitter, staggered refresh, warm-up the cache gradually,
  or use a sentinel "refresh lock" pattern.

---

## Quick decision cheat sheet

| Question                              | Answer                                              |
|---------------------------------------|-----------------------------------------------------|
| Pure cache, can rebuild on crash?     | RDB disabled or low-frequency snapshots.            |
| Want low RPO?                         | AOF `everysec` (+ RDB periodic for backups).        |
| Strict durability?                    | AOF `always` + replicas + replicas with `WAIT`.      |
| HA but data fits on one node?         | Sentinel, 3 Sentinels, 1 primary + ≥1 replica.       |
| Need to scale beyond one node?        | Redis Cluster, 3 primaries + 3 replicas minimum.    |
| Multi-key ops across many keys?       | Use `{tag}` to keep them on one slot; else, redesign. |
| Hot key overloading one shard?        | Randomize to N key variants, client-side cache, read replicas. |
| Big collection growing forever?       | Shard by `{tag}` or use HLL/Bloom; `UNLINK` to delete. |
| Stale cache risk on hot keys?         | Server-assisted client-side caching (tracking).      |
| Multi-region with local writes?       | Redis Enterprise Active-Active (CRDT); OSS has no equivalent. |
| Pub/Sub at cluster scale?             | Sharded Pub/Sub (`SPUBLISH`/`SSUBSCRIBE`, 7.0+).      |