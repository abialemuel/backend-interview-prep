# Redis Interview Questions

Model answers for backend interviews, current for Redis 8.x / Valkey. Grouped
by difficulty. Use these as a self-test, not as memorization material — the
goal is to be able to *explain* each idea out loud and follow up on probing
questions.

**How answers are graded.** A rough calibration interviewers use:

- **Junior** — knows the data types, TTLs, and the single-threaded model;
  can describe a cache-aside flow.
- **Senior** — internals and failure modes: encodings, persistence
  trade-offs, stampedes/hot keys/big keys, atomicity idioms (Lua, WATCH),
  Sentinel vs Cluster.
- **Staff** — system boundaries: when Redis is the wrong tool, multi-region
  strategy, durability guarantees under failover, licensing/fork landscape
  and its operational consequences.

Easy ≈ junior baseline, Medium ≈ senior, Hard ≈ senior/staff — but a strong
candidate upgrades any answer by volunteering the failure modes.

---

## Easy (junior baseline)

### Q1: Why is Redis so fast?

**Answer:** Redis keeps all data in memory, so reads and writes are
microsecond-level disk-free operations. Command execution is single-threaded
on the main thread, which means no lock contention, no context switching
overhead between commands, and no need for complex concurrent data
structures. It uses efficient, purpose-built encodings (SDS, listpack,
skiplist, intset) chosen automatically per value. Pipelining collapses many
round trips into one, and since 6.0 I/O threading parallelizes socket
read/write and protocol parsing across worker threads while the main thread
still owns command execution. Finally, fork-based RDB and async deletion let
expensive background work happen off the hot path. In short: in-memory +
single-threaded execution + tight data structures + pipelining.

### Q2: What does "Redis is single-threaded" actually mean?

**Answer:** Only **command execution** is single-threaded. One main thread
runs the entire command pipeline — reading from client buffers, looking up
keys, executing commands, writing replies. This eliminates the need for locks
on the data structures. Everything else is parallel: I/O threads (since 6.0,
`io-threads`) handle socket reads/writes and RESP parsing; a `fork()` child
does RDB snapshots and AOF rewrites; BIO background threads do `fsync`, lazy
free, and async eviction. So you get the simplicity of single-threaded
semantics with most of the throughput benefits of multi-threading for I/O.

### Q3: What is the difference between Strings and Hashes for storing objects?

**Answer:** With Strings you typically store the whole object as one JSON
value (`SET user:1 '{"name":"ada","age":36}'`); reads and writes are atomic
on the whole value but every update rewrites the entire JSON. With Hashes
(`HSET user:1 name ada age 36`) you store the object as fields and can update
one field with `HSET` or `HINCRBY` without rewriting the rest, which is
cheaper for partial updates and avoids read-modify-write races across fields.
Hashes are also usually more memory-efficient for small objects thanks to the
listpack encoding (up to 128 fields / 64 bytes each by default). The downside
of Hashes is that you can't atomically read the whole object as a single JSON
string. Note the classic "no per-field TTL" limitation is gone: since Redis
7.4 (and Valkey 9.0), `HEXPIRE`/`HTTL`/`HPERSIST` give individual hash fields
their own TTLs — citing the old limitation unprompted dates your knowledge.
Rule of thumb: Hashes for objects that get partial updates, Strings for
objects that are read and written as a whole (and the JSON type, in core
since Redis 8, when you need nesting plus path-level updates).

### Q4: How does Redis expire keys? What is the difference between lazy and active expiration?

**Answer:** Redis stores expiration timestamps in a separate `expires`
dictionary and uses two complementary mechanisms. **Lazy expiration** checks
the timestamp when a key is accessed and removes the key if it's already
expired; this is cheap but lets stale keys sit in memory if nobody touches
them. **Active expiration** runs a background cycle ~10 times per second per
database; it samples a small set of keys with TTLs, deletes the expired ones,
and adaptively increases sampling if the sample shows a high expiry ratio.
The cycle is short (≤ ~25 ms) so it doesn't cause latency spikes. Together
they ensure expired keys are eventually reclaimed even if never read again.
You can query the remaining time with `TTL` (seconds) or `PTTL` (ms), and
remove a TTL with `PERSIST`.

### Q5: What is pipelining and how is it different from a transaction?

**Answer:** Pipelining is a transport-level optimization: the client sends
many commands in a single TCP write without waiting for each reply, so the
RTT is paid once instead of N times. Redis executes the commands in order and
sends all replies back together. Pipelining says nothing about atomicity —
another client's commands can interleave between yours. A transaction
(`MULTI`/`EXEC`) is a server-side semantic: commands are queued and run
back-to-back on the main thread with no other client interleaving, giving
atomic execution. You can pipeline a transaction (most clients do), but the
two concepts are orthogonal. Pipelining = fewer round trips; MULTI/EXEC =
atomic batch.

### Q6: What is the difference between Redis and Memcached?

**Answer:** Both are in-memory key-value stores, but they diverge in scope.
Memcached is a pure, simple, multi-threaded key→value (string) cache with LRU
eviction and no persistence, no replication, no richer types — designed to be
dumb and fast. Redis is single-threaded for command execution, supports many
rich data types (Lists, Hashes, Sets, Sorted Sets, Streams, Bitmaps, HLL,
Geo), has persistence (RDB, AOF), replication, Sentinel, Cluster sharding,
pub/sub, Lua/Functions, client-side caching, ACLs, and TLS. Redis is a data
structure server that you can use as a cache, a database, a message queue, or
a session store. Memcached's multi-threaded design can saturate more cores on
a single box for *pure get/set* workloads, but Redis's feature set has made it
the default for most modern backends. On licensing: Memcached has always been
BSD; Redis left BSD in 2024 (SSPL/RSALv2), which spawned the BSD-licensed
Valkey fork, and since Redis 8 (May 2025) Redis is tri-licensed with AGPLv3 —
OSI open source again.

### Q7: How does `EXPIRE` interact with operations that modify a key?

**Answer:** When you modify a key that has a TTL (e.g., `INCR` on a counter
with a TTL, `HSET` on a hash with a TTL), Redis **preserves the existing
TTL** — it does not reset it, except for the explicit `PERSIST`, `EXPIRE` /
`PEXPIRE` (with or without the `GT`/`LT` flags), or the `EX`/`PX`/`EXAT`/`PXAT`
options on `SET`. This is important for the fixed-window rate limiter: you
`INCR` then `EXPIRE` only when the value is `1`, so the TTL is set once at
window start and survives subsequent `INCR`s. If you `SET key value` (without
`KEEPTTL`) on a key, the TTL is cleared.

---

## Medium (senior)

### Q8: What is the internal structure of a Sorted Set (ZSET)?

**Answer:** For small ZSETs (≤ 128 entries and ≤ 64 bytes per member by
default), Redis uses a `listpack`, a compact contiguous encoding. For larger
ones it uses **two structures together**: a **skiplist** that keeps members
ordered by `(score, member)` and a **hashtable** that maps `member → score`.
The skiplist is a probabilistic balanced tree — average O(log N) for insert,
delete, and range queries — and supports the ordered operations (`ZRANGE`,
`ZRANGEBYSCORE`, `ZRANK`). The hashtable gives O(1) `ZSCORE`, `ZMSCORE`, and
the initial member lookup for `ZRANK`. Crucially, the two structures share
the same `zskiplistNode` objects (the hashtable's entry points into the
skiplist node), so the memory overhead is the hashtable entries, not a full
duplication of all the data. Skiplists are chosen over red-black trees
because they are simpler to implement, easier to do range queries on (the
linked list at the bottom level walks the range directly), and have
comparable performance with worse constant factors but better concurrency
characteristics (less restructuring).

### Q9: When would you choose each eviction policy?

**Answer:** For a **pure cache** where every key is fair game, use
`allkeys-lru` (least recently used) if access is time-skewed, or
`allkeys-lfu` (least frequently used, since 4.0) if some keys are accessed
many times and others rarely. For a **session store** where every key has a
TTL and you want long-lived non-TTL keys (locks, configuration) to be
untouchable, use `volatile-lru` or `volatile-ttl` (evict the key with the
soonest expiry first). For a **data store** where losing an arbitrary key
would corrupt application state — distributed locks, queue metadata, a
primary index — use `noeviction` and instead plan for capacity, sharding,
and replicas. LRU/LFU in Redis are **approximate**: Redis samples
`maxmemory-samples` (default 5) keys and evicts the best candidate, so
raising the sample count gets you closer to true LRU/LFU at higher CPU cost
per eviction. Never rely on eviction as a correctness mechanism.

### Q10: How do you implement a distributed lock in Redis? What's wrong with `SETNX` + `EXPIRE`?

**Answer:** The modern idiom is `SET lock:resource <token> NX EX <seconds>`,
which atomically sets the key only if it doesn't exist and applies the TTL in
one command. The token is a unique value (a UUID or random string) generated
by the client. Releasing the lock must be a CAS: only delete the key if its
current value equals your token, otherwise you might delete someone else's
lock. Because `GET` + `DEL` is not atomic, you do this with a Lua script:
```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end
```
The old `SETNX` + `EXPIRE` pattern is broken because the two commands are not
atomic: if the client crashes between them, the lock is held forever with no
TTL. The TTL itself is a liveness bound — if the holder crashes or stalls, the
lock auto-releases, but another client may then acquire it while the original
is still doing work, so lock holders must fence operations with a token or
increasing fencing token. For multi-node setups, Redlock is the
often-discussed extension — see the next question.

### Q11: What is Redlock and what's the debate around it?

**Answer:** Redlock is an algorithm proposed by Salvatore Sanfilippo (Redis's
original author) for distributed locking across **multiple independent Redis
masters** (e.g., 5 nodes). A client acquires the lock by getting
`SET NX EX` to succeed on a majority (N/2 + 1) of nodes within a short total
time budget; if it does, it holds the lock for the remaining TTL. The
intuition is that losing one or two nodes doesn't break lock availability.
The **debate**: Martin Kleppmann wrote a widely cited critique arguing that
Redlock is unsafe under realistic conditions because (a) Redis has no
fencing-token mechanism, so a paused or GC-paused client can hold a lock past
its TTL and then perform a guarded action with a stale lock; (b) the
algorithm depends on clock assumptions across nodes that don't hold in NTP-
synced environments with jumps. Sanfilippo responded that the failure modes
Kleppmann describes require specific timing pathologies, and that for
*efficiency* (not correctness) Redlock is fine. The practical takeaway most
engineers use: **for mutual exclusion where correctness matters, use a
consensus system (Zookeeper, etcd, Consul) with fencing tokens**; for
*efficiency* locks where occasional double-execution is tolerable, `SET NX EX`
on a single Redis (or Redlock) is fine.

### Q12: How do you implement rate limiting in Redis?

**Answer:** Two common patterns. **Fixed window**: include the window start
in the key (e.g., `ratelimit:user:42:1690000000` for the window starting at
that epoch second), `INCR` it, and only `EXPIRE` (set TTL = window length)
when the result is `1`. If the value exceeds the limit, reject. This is
simple but has bursty behavior at window boundaries (a client can do 2× the
limit by sending N requests just before the window ends and N just after).
**Sliding window** with a Sorted Set: store each request as a ZSET member with
`score = request timestamp`; on each request, atomically (in Lua) `ZREMRANGEBYSCORE`
everything older than the window, `ZADD` the new request, `ZCARD` to check the
count, and reject if over the limit. This gives a true sliding window at the
cost of storing one entry per request (mitigated by short TTLs and the fact
that you can use approximate IDs). For very high scale, a **sliding window
counter** approximation (weighted sum of current and previous fixed-window
counts) gives near-sliding behavior with O(1) memory. Token bucket can be
implemented with a Hash storing `tokens` and `last_refill`, updated under a
Lua script.

### Q13: HyperLogLog vs Set for unique counting — when do you pick which?

**Answer:** A `Set` gives an **exact** count with `SCARD`, supports
`SISMEMBER` for membership tests, and lets you intersect/union sets with
`SINTER`/`SUNION`. The cost is memory proportional to the number of unique
elements: ~64 bytes per entry in the hashtable encoding, which adds up fast
(100M uniques ≈ several GB). `HyperLogLog` (`PFADD`, `PFCOUNT`, `PFMERGE`)
gives an **approximate** count with ~0.81% standard error using a fixed
**12 KB** per HLL, regardless of how many uniques you add — but it does not
support membership queries (you cannot ask "have I seen this user before?")
and only supports union, not intersection or difference. Choose a Set when
you need exactness, membership tests, or set algebra, and the cardinality is
small enough to fit in memory. Choose HyperLogLog when you only need an
approximate cardinality at scale (UV counts, unique IPs, search query
counts) and memory would be prohibitive. A common pattern: use a Set for the
last 24 hours (small, exact) and roll up to HyperLogLog for weekly/monthly
totals.

### Q14: What's the difference between Redis Pub/Sub and Streams?

**Answer:** Pub/Sub is a **fire-and-forget** push mechanism: publishers
`PUBLISH` to a channel, currently subscribed clients receive the message, and
**nothing is persisted** — if no subscriber is connected, or a subscriber is
slow and gets disconnected, the message is lost. There are no consumer
groups, no acknowledgments, no replay. Streams are an **append-only,
persistent log** (`XADD`) with consumer groups, per-group pending entries
lists (PEL), acknowledgments (`XACK`), and replay from any ID/range. Streams
support at-least-once delivery: a consumer that crashes leaves its messages
pending and another consumer can `XCLAIM` them after an idle timeout. Use
Pub/Sub for ephemeral signals (cache invalidation hints, config reload,
presence) where loss is acceptable; use Streams for durable event logs,
queues, and anything where you need replay, fan-out with independent
consumers, or guaranteed delivery.

### Q15: When would you use Redis Streams vs Kafka?

**Answer:** Use Streams when the workload fits on a single Redis instance or
a small Redis Cluster and you want a low-ops, low-latency in-memory log with
consumer groups and at-least-once semantics — internal app events, task
queues, audit logs, change-data-capture within a service, real-time fan-out.
Streams are a single partition per key; scaling means sharding by key prefix
across cluster nodes, which is more manual than Kafka's partitioning. Use
Kafka when you need **high throughput** (millions of events/sec across
brokers), **long retention** (days/weeks/months on disk), **multi-tenant
topics with many consumer groups** (each independent and replayable),
exactly-once via transactions, and a mature ecosystem (Connect, Streams,
schema registry). Kafka is operationally heavier (Zookeeper/KRaft, brokers,
partition rebalancing) but scales far beyond what a Redis process can hold.
A useful heuristic: if you're already running Redis for caching, Streams are
free; if you have a dedicated streaming platform need, Kafka.

### Q16: RDB vs AOF — what are the trade-offs?

**Answer:** **RDB** is a compact point-in-time binary snapshot taken via
`fork()`. It's great for backups, fast restart (load one file), and minimal
disk write amplification, but on a crash you lose all writes since the last
snapshot (potentially minutes of data). **AOF** logs every write command and
lets you choose an `fsync` policy: `always` (safest, very slow),
`everysec` (the common default — up to ~1 s of data loss, fsync in a
background thread so the main thread isn't blocked), or `no` (let the OS
flush — fastest, can lose everything in the page cache on a crash). AOF files
are larger and replay is slower, but the durability window is tiny. Since
Redis 4.0 you can run both: the AOF rewrite produces a file with an **RDB
preamble** followed by incremental AOF commands (`aof-use-rdb-preamble yes`),
combining fast restart with low RPO. For a pure cache, RDB (or nothing) is
fine; for a session store or anything with durability requirements, AOF
`everysec` (optionally hybrid) is the standard.

### Q17: Why is `appendfsync everysec` the common default?

**Answer:** `everysec` calls `fsync` on the AOF file once per second from a
background thread. That gives you a worst-case data-loss window of about one
second on a crash, which is acceptable for the vast majority of workloads
(session stores, caches with persistence, queues with at-least-once). It
keeps the main thread unblocked because the fsync happens off-thread, so
command latency stays microsecond-ish. `always` would fsync after every write
— safe but disk-latency-bound, killing throughput. `no` lets the OS decide
when to flush — fastest but you can lose the entire page cache on a power
loss. `everysec` is the sweet spot in the durability/performance trade-off,
which is why Redis ships it as the recommended default for AOF deployments.

### Q18: What is Redis Sentinel and how does it differ from Redis Cluster?

**Answer:** Sentinel is a **high-availability** system for a single primary
plus its replicas: a set of Sentinel processes monitors the primary, detects
failures (subjective down → objective down with quorum), promotes a replica
on failure, reconfigures other replicas, and acts as a configuration provider
so clients can ask "who is the current primary?". It does **not shard** data
— the whole dataset still lives on one primary, so capacity is bounded by
one node. Redis Cluster is a **sharding + HA** system: it partitions the
keyspace into 16384 hash slots distributed across multiple primaries, each
with optional replicas, and does per-shard failover via gossip. Cluster
gives you horizontal scale but imposes restrictions: only DB 0, multi-key
operations only within the same slot (use `{tag}` to keep related keys
together), and clients must be cluster-aware. Use Sentinel when your dataset
fits on one node and you just want HA; use Cluster when you need to scale
out across multiple nodes.

### Q19: How does Redis Cluster shard data? What are hash slots and key tags?

**Answer:** Redis Cluster partitions the keyspace into **16384 hash slots**.
The slot for a key is `CRC16(key) mod 16384`. Each primary node owns a subset
of slots; the union of all primaries' slots covers the full 16384. CRC16 was
chosen for speed; the slot count of 16384 (rather than 65536) was chosen to
keep gossip messages compact while still allowing thousands of nodes. A
**key tag** is the substring between `{` and `}` in a key, if present: when
computing the slot, Redis uses the tag instead of the whole key. So
`user:{1001}:cart` and `user:{1001}:profile` both hash to the slot of
`1001` and are guaranteed to live on the same node. This is what enables
multi-key operations (`MGET`, `MULTI`, Lua scripts that touch multiple keys)
in cluster mode — all involved keys must hash to the same slot, and key tags
are the way to force that. Without a tag, related keys will generally land
on different shards and `CROSSSLOT` errors will block any multi-key operation.

### Q20: What is the cross-slot limitation and how do you work around it?

**Answer:** In Redis Cluster, any command that operates on more than one key
(`MGET`, `MULTI`/`EXEC` transactions, Lua `EVAL`/`EVALSHA` scripts, `SINTERSTORE`,
`ZUNIONSTORE`, etc.) requires **all keys to hash to the same slot**. If they
don't, Redis returns a `CROSSSLOT` error. The workaround is to use **key
tags**: embed a shared tag inside `{...}` in every related key, e.g.,
`order:{1001}:header`, `order:{1001}:items`, `order:{1001}:totals`. Redis
hashes only the tag, so all three keys land on the same slot and the same
node, making multi-key operations legal. When a logical group of keys is too
big to tag (e.g., a Set that should union keys from many users), you have to
redesign: do the operation client-side by fetching each key individually, or
pre-aggregate into a single key. There's no way to do a cross-slot
transaction in cluster mode — this is one of the main reasons people choose
Sentinel over Cluster for workloads with lots of multi-key operations.

### Q21: What is a cache stampede and how do you prevent it?

**Answer:** A cache stampede (also called thundering herd) is what happens
when a hot key expires and many concurrent requests all miss the cache at
once, all hit the backend, and all recompute and re-populate the cache. The
backend gets a sudden spike of identical expensive work. Variants: "cache
breakdown" for a single hot key expiring, "cache avalanche" for many keys
expiring at once (often because they were loaded with the same TTL).
Defenses: (1) **mutex / distributed lock** — the first miss acquires
`SET lock:key NX EX 5` and computes; concurrent misses sleep briefly and
re-read the cache. (2) **request coalescing / single-flight** at the
application layer — collapse concurrent identical requests into one backend
call (Go's `singleflight` is the canonical example). (3) **probabilistic
early expiration (XFetch)** — each reader has a small probability of treating
the value as already expired and refreshing it *before* the real TTL hits,
smoothing the spike. (4) **randomized TTLs with jitter** — set TTL = base ±
random so bulk-loaded keys don't expire simultaneously. (5) **stale-while-
revalidate** — return the stale value immediately and refresh in the
background. For a single hot key, mutex or XFetch; for many keys expiring
together, jittered TTLs.

### Q22: What are hot keys and how do you handle them?

**Answer:** A hot key is one that receives a disproportionate fraction of
traffic, concentrating load on the single shard that owns it (in cluster
mode) or on the single instance (in Sentinel). Symptoms: one node CPU-bound
while others are idle, one shard's `INFO commandstats` dominated by
operations on one key. Find them with `redis-cli --hotkeys` (requires an LFU
eviction policy for meaningful sampling), `INFO commandstats`, cloud-provider
hot-key dashboards, or application-level metrics. Handle them by: (1)
**copying the value to N random key variants** so clients pick one at random
and distribute the read rate across N keys on different slots; (2)
**client-side caching** the hot value so most reads don't hit Redis at all;
(3) reading from **replicas** if your workload is read-heavy and the
client/library supports `READONLY` mode in cluster; (4)
**application-level memoization** for the most extreme cases. The right fix
depends on whether the hot key is read-heavy (replicate / cache / split
variants) or write-heavy (redesign the data model to spread writes across
keys).

### Q23: What are big keys and why are they dangerous?

**Answer:** A big key is a single key whose value is unusually large — a
hash with millions of fields, a list with millions of entries, a giant
string. They're dangerous because: (1) `DEL` on a big key blocks the main
thread (use `UNLINK` for async deletion); (2) `HGETALL`, `LRANGE 0 -1`,
`SMEMBERS` return huge payloads, blocking the client and the network and the
main thread; (3) eviction sampling rarely picks them, so they sit and
consume memory; (4) `MIGRATE` during resharding is slow on big keys and can
time out; (5) on replication they inflate the full-resync RDB. Find them
with `MEMORY USAGE <key> SAMPLES 0`, `redis-cli --bigkeys`, or `SCAN` +
`OBJECT ENCODING` / `DEBUG OBJECT`. Avoid them by: capping collections with
`LTRIM`, `ZREMRANGEBYRANK`, `XTRIM`; sharding a logical collection into many
keys using `{tag}`; using HyperLogLog or Bloom filters instead of Sets at
scale; enabling `lazyfree-lazy-eviction`, `lazyfree-lazy-expire`,
`lazyfree-lazy-server-del`, and using `UNLINK` instead of `DEL`.

### Q24: How does `WATCH` provide optimistic locking?

**Answer:** `WATCH key [key ...]` marks the given keys as watched; if any of
them is modified by any client (or expires, or is evicted) between the
`WATCH` and the subsequent `EXEC`, Redis aborts the transaction and returns
nil from `EXEC` instead of running the queued commands. The canonical
pattern is read-then-write: `WATCH balance`, `GET balance`, compute the new
value client-side, `MULTI`, `SET balance <new>`, `EXEC`. If another client
changed `balance` in between, `EXEC` returns nil and you retry the whole
sequence. This is **optimistic locking** — assume contention is rare, retry
when it happens — and it's the right Redis idiom for check-then-set
operations that don't fit a single atomic command (like `INCR` or a Lua
script). It's cheaper than pessimistic locking for low-contention cases
because it doesn't hold any locks; under high contention the retry rate
climbs and you should switch to a Lua script or a distributed lock.

### Q25: Why are Lua scripts atomic in Redis?

**Answer:** Because command execution is single-threaded on the main thread.
From the moment a Lua script starts executing (`EVAL` / `EVALSHA`) until it
returns, no other command from any client runs — the main thread is busy
inside the script. So everything the script does — multiple `redis.call`s,
conditionals, loops — happens atomically with respect to other clients,
without any explicit locking. This is why Lua is the right tool for
multi-step atomic operations like releasing a distributed lock with a CAS,
atomically incrementing-and-capping a counter, or atomically doing the
`ZREMRANGEBYSCORE` + `ZADD` + `ZCARD` sequence in a sliding-window rate
limiter. The downside is that a slow script blocks *every* client, so keep
scripts short and avoid O(N) loops. Cache scripts with `SCRIPT LOAD` and run
by SHA1 with `EVALSHA` to avoid resending the body. In cluster mode, all
keys a script touches must hash to the same slot (use `{tag}`). Redis
Functions (7.0+) are the named, persisted, library-organized successor to
`EVAL` — same atomicity guarantees, better operational story.

---

## Hard (senior/staff)

### Q26: How does server-assisted client-side caching work, and when should you use it?

**Answer:** Redis 6.0 introduced **server-assisted client-side caching** via
the `CLIENT TRACKING` command. The idea: clients keep a local cache of
values they've read, and Redis tells them when those values change so they
can invalidate. Two modes. In **default (push) mode**, the server remembers
which clients have read which keys; when a key is modified, it pushes an
invalidation message (`invalidate` message in RESP3, or a pub/sub message on
a special channel in RESP2) to each affected client. Lower bandwidth, but
the server keeps per-key, per-client tracking state. In **broadcasting
mode** (`BCAST PREFIX user:`), clients subscribe to *prefixes*; the server
notifies all matching clients on any write to a key under that prefix. No
per-key tracking state on the server, but more invalidation messages. Use it
when you have read-heavy hot keys where the round-trip to Redis dominates —
caching them locally and letting Redis invalidate cuts the read rate by
orders of magnitude. Don't use it for write-heavy keys (you'll just
invalidate constantly) or for keys that are already cheap to compute. Note
that the client library must support tracking (Lettuce, StackExchange.Redis,
redis-py, go-redis with the right options), and that invalidations arrive as
RESP3 push messages (`HELLO 3`), with a pub/sub fallback on RESP2.

### Q27: What is cache penetration vs cache breakdown vs cache avalanche, and how do you defend against each?

**Answer:** Three distinct failure modes. **Cache penetration** is queries
for keys that don't exist in the DB either — every miss goes all the way to
the DB and is never cached (often malicious, e.g., `user_id = -1`). Defenses:
cache empty/null results with a short TTL so repeated misses are served from
cache; put a **Bloom filter** in front of the DB so "definitely not present"
queries return immediately without hitting the DB; rate-limit clearly
invalid queries. **Cache breakdown** is a single **hot** key expiring and
many concurrent requests flooding the DB at once. Defenses: distributed
mutex (`SET NX EX`), probabilistic early expiration (XFetch), or a
never-expiring cache refreshed in the background. **Cache avalanche** is
**many** keys expiring at the same time (often because they were loaded with
the same TTL during a bulk load), overwhelming the DB. Defenses: randomized
TTLs with jitter (`TTL = base ± random`), staggered refresh, gradual cache
warm-up, or a global "refresh lock" pattern. The three are related but the
defenses differ: penetration is about *non-existent* keys (Bloom filter +
null caching), breakdown is about *one hot key* expiring (mutex / XFetch),
avalanche is about *many keys* expiring together (jittered TTLs + staggered
refresh).

### Q28: How does Redis Cluster failover work, and what data-loss window should you expect?

**Answer:** Nodes gossip about each other on the cluster bus. When a node
can't reach a primary, it marks it `PFAIL` (possible failure). Once a
majority of masters agree the primary is down, it's marked `FAIL`. One of
that shard's replicas then elects itself via a Raft-like leader election
among the replicas (the most up-to-date replica by replication offset wins
ties), takes over the failed primary's slots, and broadcasts the new
configuration. Clients hitting those slots get `MOVED` redirections to the
new primary and update their slot maps. The data-loss window is bounded by
**asynchronous replication**: the most recent writes the old primary
acknowledged to clients may not yet have reached the replica that takes
over, so those writes are lost. You can shrink this window with `WAIT N
<timeout>` on critical writes (the client blocks until N replicas have
acknowledged), but that trades latency for durability and isn't the default.
`cluster-node-timeout` (default 15 s) controls how long a node must be
unreachable before it's considered failed — shorter means faster failover
but more false positives on transient network issues.

### Q29: Walk through the full resync vs partial resync flow in Redis replication.

**Answer:** When a replica connects to a primary (or reconnects after a
drop), it sends its last replication offset and primary run ID. If the
primary's run ID matches *and* the offset is still inside the
**replication backlog** (a circular buffer sized by `repl-backlog-size`,
default 1 MB), the primary does a **partial resync**: it just streams the
backlog from that offset onward, no full RDB transfer. This is fast and
cheap. If either the run ID differs (the primary restarted, or the replica
was pointing at a different primary) or the offset is outside the backlog
window (the replica was disconnected too long, or the write rate exceeded
the backlog capacity), the primary does a **full resync**: it `BGSAVE`s an
RDB while buffering new writes in a client output buffer, streams the RDB to
the replica, then streams the buffered writes, and finally continues with
the live command stream. Full resync is expensive — it forks the primary and
transmits the entire dataset — so sizing the backlog large enough to absorb
expected disconnections is important for write-heavy workloads. Replicas can
be chained (replica of a replica) to offload the primary, but the topology is
still a single-leader tree.

### Q30: A primary fails in a Sentinel setup. Walk through the failover.

**Answer:** Each Sentinel pings the primary and other Sentinels every second.
If a Sentinel doesn't get a reply within `down-after-milliseconds`, it marks
the primary **subjectively down (SDOWN)**. It then asks other Sentinels
whether they agree; once `quorum` Sentinels agree, the primary is marked
**objectively down (ODOWN)**. The Sentinels then run a leader election
(Raft-like) to pick one Sentinel to orchestrate the failover. The elected
Sentinel selects the best replica (by replication offset, then priority
config `replica-priority`, then run ID), promotes it to primary (`REPLICAOF
NO ONE`), and instructs the other replicas to follow the new primary
(`REPLICAOF <new-primary>`). It also updates its own configuration and
publishes the change on the pub/sub channel so clients can learn the new
primary. Clients that use Sentinel-aware libraries automatically pick up the
new primary by re-querying Sentinel. Quorum should be set to `N/2 + 1` (e.g.,
2 of 3 Sentinels), and you should run at least 3 Sentinels across failure
domains to avoid split brain. `parallel-syncs` controls how many replicas
resync concurrently after failover (lower = less load on the new primary,
slower recovery).

### Q31: You're seeing periodic latency spikes every few minutes on a Redis instance with persistence enabled. How do you debug it?

**Answer:** Start with `LATENCY DOCTOR` and `LATENCY LATEST` — Redis samples
notable events (fork, aof-fsync-always, expire-cycle, unlink) and will tell
you which one is spiking. Check `INFO Persistence` for `rdb_bgsave_in_progress`
and `aof_rewrite_in_progress` — a periodic spike that aligns with RDB saves
or AOF rewrites points to `fork()` latency on a large dataset, especially
under memory pressure or with copy-on-write page duplication. Check
`latest_fork_usec` in `INFO stats`. Mitigations: enable `latency-monitor-
threshold`, use `activedefrag yes` if `mem_fragmentation_ratio` is high, size
`repl-backlog-size` and client output buffers to avoid stalls, switch AOF
fsync from `always` to `everysec` if applicable, consider `disable-thp`
(Transparent Huge Pages off — a famous source of fork latency on Linux), or
move persistence to a replica (`replica-priority 0` so it's never promoted,
but takes the snapshotting load off the primary). Also check `SLOWLOG GET
10` for slow commands — a `KEYS`, a giant `SORT`, or a `LRANGE 0 -1` on a
huge list will block everything. Finally check `INFO commandstats` for
commands with high total CPU — sometimes the spike is a slow command, not
persistence.

### Q32: Design a rate limiter that supports a sliding window with high precision and high throughput. What trade-offs do you make?

**Answer:** The **sliding-window log** with a Sorted Set gives true precision:
one ZSET per (user, resource), `score = request timestamp in ms`, `member =
unique request ID`. On each request, a Lua script atomically does
`ZREMRANGEBYSCORE key 0 (now - window_ms)`, `ZADD key now reqid`, `ZCARD key`,
and rejects if the count exceeds the limit. The script is atomic so concurrent
requests can't race. Precision is exact; throughput is limited by the ZSET
operations (O(log N) each) and by the fact that you store one entry per
request (memory grows with the request rate within a window). Trade-offs: (1)
**memory** — one ZSET entry per request can be expensive at very high QPS;
cap with periodic `ZREMRANGEBYRANK` or by relying on the `ZREMRANGEBYSCORE`
in the script. (2) **throughput** — every request hits Redis once with a Lua
script; if that's too much, drop to the **sliding-window counter**
approximation: keep two counters per user, current window and previous window,
estimate the current count as `prev * (1 - elapsed/window) + curr`, O(1)
memory and O(1) Redis ops but with ~boundary error. (3) **clock skew** — use
server time (`TIME` command inside the script) so all clients agree. (4)
**sharding** — key by user so the same user's requests always hit the same
cluster slot. For most real systems, the sorted-set sliding window is the
right default; the counter approximation is the scale-out escape hatch.

### Q33: You need to migrate a single-node Redis to a Cluster without downtime. How do you do it?

**Answer:** A common approach is **dual-write + cutover**. (1) Provision the
new cluster with the desired topology (e.g., 3 primaries + 3 replicas). (2)
Run an offline copy of the data into the cluster using a tool like
`redis-cli --cluster import` or `redis-shake` or a custom script that
`SCAN`s the source and `SET`s/`HMSET`s/etc. into the cluster, using key tags
where needed to keep multi-key operations valid. (3) Switch the application
to **dual-write** — every write goes to both the old single-node Redis and
the new cluster — while reads still go to the old node. (4) Once you've
verified the cluster is up to date (compare key counts, sample some keys,
run a read diff), switch reads to the cluster. (5) After a soak period, drop
the dual-write to the old node and decommission it. For keys that need key
tags in cluster mode but didn't have them in the single-node deployment,
you'll need a key-naming migration — either rename keys during the copy step
or change the application's key naming and dual-write under the new names.
The tricky parts are multi-key operations (you may need to redesign to use
key tags or split into multiple commands) and `SELECT` (cluster only
supports DB 0, so any DB selection in the old deployment has to be encoded
into the key namespace). Alternatively, a **proxy** (Twemproxy, Envoy's
Redis filter, or a Redis Cluster proxy) can let the application keep talking
to a single endpoint while you shard underneath.

### Q34: What changed with Redis licensing, what is Valkey, and how would you choose in 2026? *(staff)*

**Answer:** Timeline: Redis was BSD-3 until March 2024, when Redis Ltd.
relicensed 7.4+ under dual SSPL/RSALv2 — not OSI open source — primarily to
stop cloud providers reselling it. The industry response was **Valkey**, a
BSD-3 fork of Redis 7.2 under the Linux Foundation backed by AWS, Google,
Oracle and others; most cloud "Redis-compatible" services (ElastiCache,
Memorystore) moved to it. In May 2025, **Redis 8** added **AGPLv3** as a
third license option — OSI-approved, so Redis is open source again — and
folded the former Stack modules (JSON, query engine, time series,
probabilistic) plus the new vector set type into core. Choosing in 2026:
Redis 8 if you want the richer type surface (vector sets, JSON, search) and
AGPLv3 is acceptable to your legal team (AGPL's network-copyleft scares some
enterprises); Valkey if you want a permissive license, are on a cloud managed
service anyway, or want its performance work (much stronger I/O-thread
scaling, and in Valkey 9.0 atomic slot migration, cluster-mode multi-DB,
hash-field TTLs). Core commands and protocol remain compatible; the risk to
name is *future* drift as the two roadmaps diverge. A staff answer also notes:
for most workloads the deciding factors are managed-service availability and
legal posture, not features.

### Q35: Redis Functions vs Lua scripts — what problem do Functions solve? *(senior)*

**Answer:** Both run atomically on the main thread and can call any command;
the difference is operational, not semantic. `EVAL` scripts are anonymous
blobs the *application* owns: the server caches them by SHA1, but the cache
is volatile — after a restart or failover `EVALSHA` throws `NOSCRIPT` and
every client needs fallback logic to re-`SCRIPT LOAD`. Every service that
talks to Redis must carry its own copy of the script, so shared logic drifts
across codebases. **Functions** (7.0+) invert ownership: you `FUNCTION LOAD`
a named library into the server; it is **persisted** (RDB/AOF) and
**replicated** to replicas like data, survives restarts, and any client
invokes it by name with `FCALL`. That makes server-side logic a deployable,
versionable artifact — like a stored procedure — instead of an inline string.
Same caveats as scripts: they block the main thread, and in cluster mode all
touched keys must share a slot. Interview one-liner: "Functions are Lua
scripts with a name, a home, and a replication story."

### Q36: How would you use Redis as a vector store, and when would you not? *(staff)*

**Answer:** Redis 8's **vector sets** store embeddings as members of a
sorted-set-like structure (`VADD key VALUES <dim> ... member`) and answer
approximate nearest-neighbor queries (`VSIM`) over an HNSW graph, with
optional quantization to reduce memory; the Redis Query Engine also supports
vector similarity over JSON/hash documents with filters. The killer use case
is **semantic caching** for LLM apps: embed the incoming prompt, `VSIM`
against cached prompts, and serve the stored answer if similarity exceeds a
threshold — Redis is usually already in the stack and gives sub-millisecond
lookups. Also good: session-scoped RAG, recommendations, dedup of
near-identical content. When not: corpora that don't fit in RAM (memory is
the cost ceiling — disk-based stores or pgvector win on $/GB), workloads
needing vectors transactionally consistent with relational data
(PostgreSQL + pgvector keeps them in the same ACID store — see
[PostgreSQL advanced features](../postgresql/03-advanced-features.md)), or
heavy filtered search over billions of vectors (dedicated engines). Also
flag: vector sets are Redis-only — Valkey does not ship them — so using them
couples you to Redis proper.

### Q37: Design per-device session storage for a user. How does hash field expiration change the design? *(senior)*

**Answer:** Pre-7.4 you had two awkward options: one key per (user, device) —
`SETEX session:42:dev1 3600 token` — which gives clean TTLs but scatters a
user's sessions across keys (listing them needs `SCAN` or a bookkeeping set,
which itself goes stale); or one hash per user — `HSET sessions:42 dev1
token` — which makes "list all sessions" one `HGETALL` but forces you to
store expiry timestamps in the values and lazily purge them yourself, since
TTL was per key. With **hash field expiration** (7.4+) the hash design
becomes clean: `HSET sessions:42 dev1 token` then `HEXPIRE sessions:42 3600
FIELDS 1 dev1` — each device session expires independently, `HGETALL` lists
only live sessions, and revoking one device is `HDEL`. Trade-offs to
mention: per-field TTLs cost extra memory per field; the whole hash still
lives on one cluster slot (fine for one user, the natural shard unit); and
in cluster mode the pattern keeps a user's sessions co-located for atomic
multi-field operations. This question is often a probe for whether your
Redis knowledge is current.

### Q38: When is Redis the wrong tool — what can your primary database already do? *(staff)*

**Answer:** The staff-level instinct is to resist adding a second stateful
system until the primary database actually can't cope. Job queue at modest
scale: `SELECT ... FOR UPDATE SKIP LOCKED` in [MySQL](../mysql/04-interview-questions.md)
or [PostgreSQL](../postgresql/04-interview-questions.md) gives a transactional
queue with no extra infrastructure — and the jobs commit atomically with the
business data, which Redis can never offer. Ephemeral pub/sub: PostgreSQL
`LISTEN/NOTIFY` covers low-rate notification fan-out. Caching: measure first —
a well-indexed query served from the buffer pool is often fast enough that a
cache only adds an invalidation problem. Rate limiting at low QPS: a counter
table works. Redis earns its place when you need sub-millisecond latency at
high QPS, shared state across many stateless app instances, data structures
the RDBMS lacks (sorted-set leaderboards, HLL, streams with consumer groups),
or to shed read load that demonstrably saturates the primary. The follow-up
trap is consistency: once data lives in both Redis and the DB you own an
invalidation/dual-write problem, so every cached byte needs a staleness
budget. Answering "Redis for everything" or "Postgres for everything" both
fail; the signal is knowing where the boundary is and what crossing it costs.