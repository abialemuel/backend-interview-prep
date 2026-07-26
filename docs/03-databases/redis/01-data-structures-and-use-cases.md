# Redis Data Structures & Use Cases

This file covers the core Redis data types, their internal encodings, and the
typical interview use cases for each. It also covers the Redis 8 additions
(vector sets, JSON, probabilistic structures), expiration, eviction, the
single-threaded execution model, pipelining, transactions, Lua, and Functions.

All examples use `redis-cli` against a single Redis 8.x / Valkey instance.

---

## 1. Strings

Strings are the most fundamental Redis type. They are **binary safe**, can hold
up to 512 MB, and can represent text, integers, serialized JSON, raw bytes,
bitmaps, and more.

### Internal encoding

Redis does not use C strings internally. It uses **SDS** (Simple Dynamic
Strings) which carry an `len` field and a `free`/capacity field, enabling O(1
length, safe binary handling, and efficient appending.

There are three encodings for a String value (`OBJECT ENCODING <key>`):

- `int` — when the value is a string-representable integer up to `LONG_MAX`.
- `embstr` — short strings (≤ 44 bytes in Redis 7.x). The `redisObject` header
  and the SDS buffer are allocated in a single contiguous block, which is cache
  friendly. Read-only: any modification promotes it to `raw`.
- `raw` — strings longer than 44 bytes (or embstr strings that were modified).
  The header and SDS live in separate allocations.

SDS uses **pre-allocation**: when growing a string Redis allocates extra
unused space to make repeated appends amortized O(1) (up to 1 MB Strings get
`2x` headroom; beyond that a fixed 1 MB extra).

### Commands and examples

```bash
SET user:1 '{"name":"ada","age":36}'      # store serialized JSON
GET user:1
SETEX session:abc 60 "tok"                # set with TTL in seconds
SET lock:order NX EX 10                   # distributed lock primitive
INCR counter:page:home                    # 64-bit atomic counter
INCRBY hits 5
APPEND key "!"                            # append to current value
STRLEN key
GETRANGE key 0 4
```

### Use cases

- **General cache** — store JSON, HTML fragments, protobuf blobs, etc.
- **Atomic counters** — page views, like counts, inventory decrements with
  `INCR`/`DECR`. These are atomic because command execution is single-threaded.
- **Rate limiting (fixed window)** — `INCR` + `EXPIRE`:
  ```bash
  INCR ratelimit:user:42:1690000000   # key includes window start (epoch seconds)
  EXPIRE ratelimit:user:42:1690000000 60
  ```
  If the return is `1`, this is the first request in the window; set TTL. If the
  return exceeds the limit, reject. (See Sorted Sets for sliding window.)
- **Distributed locks** — `SET <lock> <token> NX EX <seconds>`. Release with a
  Lua CAS script comparing the token (see Lua section).

---

## 2. Lists

Lists are **ordered sequences of strings** addressed by insertion side
(`LPUSH`/`RPUSH`) and index.

### Internal encoding

- `listpack` (since Redis 7.0) for small lists — a compact contiguous encoding
  that replaced `ziplist`. Each element header carries its own length so the
  structure can be scanned without unrolling.
- `quicklist` — a doubly-linked list of `listpack` nodes, used when the list
  grows past the thresholds (`list-max-listpack-size`, default 8 KiB per node;
  `list-compress-depth` controls how many inner nodes stay uncompressed).

A `quicklist` is what you get for any non-trivial list: it keeps memory tight
via per-node listpacks while still giving O(1) push/pop on both ends.

### Commands

```bash
LPUSH queue:tasks t1 t2 t3
RPUSH queue:tasks t4
LPOP queue:tasks               # FIFO when combined with RPUSH
RPOP queue:tasks               # LIFO-ish (used as a stack)
LRANGE queue:tasks 0 -1
LLEN queue:tasks
LTRIM queue:tasks -100 -1      # keep only last 100 (capped collection)
BLPOP queue:tasks 30           # blocking pop with 30s timeout
```

### Use cases

- **Work queues** — `LPUSH`/`RPOP` (or `RPUSH`/`BLPOP`) makes a FIFO list used
  by many worker-pool designs.
- **Activity feeds / timelines** — fan-out-on-write: push event IDs to a
  per-user list, cap with `LTRIM` so feeds don't grow unbounded.
- **Capped collections** — `LTRIM key 0 999` after each push keeps only the most
  recent N entries (recent log lines, recent messages, etc.).
- **Reliable-ish queues** — `BRPOPLPUSH`/`LMOVE` moves an element to a
  processing list so a crash mid-job leaves it recoverable.

---

## 3. Hashes

Hashes map string fields to string values — the natural representation of an
object.

### Internal encoding

- `listpack` for small hashes (default threshold: 128 entries *and* 64 bytes per
  item, `hash-max-listpack-entries`, `hash-max-listpack-value`).
- `hashtable` for larger hashes — a separate-chaining hash table that rehashes
  incrementally to avoid latency spikes (the `dict` rehashing is done a few
  buckets at a time, mixed with normal operations).

### Commands

```bash
HSET user:1 name ada age 36 role admin
HGET user:1 name
HGETALL user:1
HINCRBY user:1 age 1
HDEL user:1 role
HSCAN user:1 0
```

### Use cases

- **Object storage** — instead of `SET user:1 '<json>'` and rewriting the whole
  object on every change, a hash lets you update one field with `HSET` (no full
  copy needed, no serialization round-trip).
- **Partial updates** — `HINCRBY` for a single numeric field, `HDEL` for one
  field: also avoids read-modify-write race conditions across fields.
- **Counters per dimension** — `HINCRBY stats:2024-07-09 page_views 1`, where
  each field is a metric for a day.

Interview tip: **Strings vs Hashes for objects**.
- Strings (`SET user:1 '<json>'`): simple, atomic GET/SET of the whole object,
  cheap reads, but every update rewrites the whole value.
- Hashes (`HSET user:1 ...`): field-level updates, lower write amplification.
- Memory: hashes are typically more memory-efficient when you have many small
  fields, because of listpack encoding.

**Hash field expiration (7.4+)** — a stale-knowledge check in 2026 interviews.
For most of Redis history TTL was per *key* only, and "hashes don't support
per-field TTL" was the standard trade-off. Since Redis 7.4 (and Valkey 9.0)
fields can carry their own TTLs:

```bash
HSET user:1:sessions dev1 tokenA dev2 tokenB
HEXPIRE user:1:sessions 3600 FIELDS 1 dev1     # TTL on one field
HTTL user:1:sessions FIELDS 2 dev1 dev2
HPERSIST user:1:sessions FIELDS 1 dev1
```

Use cases: per-device session tokens inside one user hash, per-entry caches
grouped under one key. Note it costs extra memory per TTL'd field — don't
apply it to millions of fields blindly.

---

## 4. Sets

Sets are **unordered, unique** collections of strings.

### Internal encoding

- `intset` — when *all* elements are integers *and* the count is below
  `set-max-intset-entries` (default 512). An intset is a sorted array of
  integers (16/32/64 bit, upgraded in place as needed). Compact and fast for
  integer-only sets.
- `listpack` (since 7.2) — small sets with non-integer members
  (`set-max-listpack-entries`, default 128) get the compact contiguous
  encoding too, instead of jumping straight to a hashtable.
- `hashtable` — otherwise, similar to the hash-table encoding used by Hashes
  (just with NULL values).

### Commands

```bash
SADD tags:post:1 redis cache db
SADD tags:post:2 redis nosql
SISMEMBER tags:post:1 redis
SMEMBERS tags:post:1
SCARD tags:post:1
SINTER tags:post:1 tags:post:2     # intersection
SUNION tags:post:1 tags:post:2
SDIFF tags:post:1 tags:post:2
SRANDMEMBER tags:post:1 2         # random sample (with replacement if negative)
SPOP tags:post:1 2                # random removal
```

### Use cases

- **Tags / labels** — boolean membership with `SISMEMBER`, intersection to find
  posts with overlapping tags.
- **Unique visitors in a finite window** — `SADD uv:day:20240101 <uid>`,
  `SCARD` for the count. For long ranges switch to HyperLogLog when exactness
  isn't required.
- **Set algebra** — `SINTER` for "items in both groups", `SUNION` for merge,
  `SDIFF` for "in A not in B".
- **Random sampling** — `SRANDMEMBER` / `SPOP` for A/B tests, lotteries,
  random winner/match-making picks.

---

## 5. Sorted Sets (ZSET)

A Sorted Set (ZSET) is a set of unique members each carrying a `double` score.
Elements are sorted by `(score, member)`.

### Internal encoding

- `listpack` for small ZSETs (default 128 entries *and* 64-byte values,
  `zset-max-listpack-entries`, `zset-max-listpack-value`).
- **skiplist + hashtable** for larger ZSETs. This is the canonical Redis
  internal almost always referenced in interviews:
  - The **skiplist** keeps members ordered by `(score, member)`. It is a
    probabilistic balanced tree — average O(log N) lookup/insert/range query,
    simpler than red-black trees, and worse constant factors but still tight.
  - The **hashtable** (a real dict, on the side) maps `member -> score`, giving
    O(1) `ZSCORE`, `ZRANK`'s initial lookup, etc.
  - The two structures share the same `zskiplistNode` objects, so they are not
    a memory duplication.*

### Commands

```bash
ZADD leaderboard 100 alice 220 bob 95 carol
ZRANGE leaderboard 0 -1 WITHSCORES           # ascending
ZREVRANGE leaderboard 0 2 WITHSCORES          # top 3 desc
ZRANGEBYSCORE leaderboard 100 200             # score range
ZRANK leaderboard alice                      # 0-based
ZSCORE leaderboard bob
ZINCRBY leaderboard 15 alice
ZREMRANGEBYSCORE leaderboard 0 99
ZREVRANGEBYLEX ...
```

Redis 6.2+ introduces new generic range commands: `ZRANGE` with `BYSCORE`
and `BYLEX` and `REV` flags, superseding the older `ZRANGEBYSCORE` /
`ZREVRANGEBYSCORE` / `ZRANGEBYLEX` family.

### Use cases

- **Leaderboards** — `ZADD`/`ZINCRBY` to bump score, `ZREVRANGE 0 9` for top 10.
- **Priority queues** — score is priority/earliest-deadline, `ZPOPMIN` pops the
  highest-priority item (`BZPOPMIN` for blocking version).
- **Time-based indexes** — store `score = epoch_ms` (or event time). `ZRANGEBYSCORE`
  becomes a time-range query: "all events between these two timestamps".
- **Sliding-window rate limiting** — one ZSET per user, score = request time:
  ```bash
  ZADD rl:42 1690000000123 <reqid>
  ZREMRANGEBYSCORE rl:42 0 1689999940123   # drop older than 60s ago
  ZCARD rl:42                             # if > N: reject
  ```
  Set a TTL on the set as a safety net, and wrap the drop-old + add + count
  sequence in a Lua script so it is atomic.

---

## 6. Bitmaps

Not a separate type — Bitmaps are **String values operated on as bit arrays**.
Up to 2^32 bits = 512 MB per bitmap.

### Commands

```bash
SETBIT user:active:20240701 42 1        # user 42 was active on July 1
GETBIT user:active:20240701 42
BITCOUNT user:active:20240701           # how many users active that day
BITOP AND days_combined user:active:20240701 user:active:20240702
BITPOS user:active:20240701 1           # find first set bit
BITFIELD user:stats U8:0 INCRBY 1       # manipulate counter bitfields
```

### Use cases

- **User active days** — `SETBIT user:42 169 1` means "user 42 was active on day
  169". `BITCOUNT user:42` gives lifetime active days. `BITOP` unions/intersects
  days between users.
- **Online user counting** — `BITCOUNT` over a daily bitmap.
- **Real-time features** — fast bitop-based set algebra when the *universe* of
  IDs is bounded and integer (fog of war, feature flags, online presence).
- **Approximate Bloom-filter-style membership** — using multiple `SETBIT`
  positions from N hash functions gives a hand-rolled Bloom filter. It works,
  but since Redis 8 a real Bloom filter (`BF.ADD`/`BF.EXISTS`) ships in core
  (formerly the RedisBloom module) — prefer that.

---

## 7. HyperLogLog

HyperLogLog (HLL) is a **probabilistic data structure** for cardinality
estimation. Standard error ≈ **0.81%**, fixed memory footprint ≈ **12 KB**
per HLL, regardless of how many unique values you add.

### Commands

```bash
PFADD page:uv:homepage user:1 user:2 user:1
PFCOUNT page:uv:homepage
PFMERGE page:uv:jul_total page:uv:homepage page:uv:search
```

### Use cases

- **Unique visitor counting** at scale where a Set would be too expensive.
  10M uniques per day for a year is ~120 KB of HLL versus ~hundreds of MB of
  Sets. Trades exact count for ~1% error.
- **Daily → monthly merge** with `PFMERGE`.

`PFADD` accepts any number of arguments, but each element is hashed to the same
fixed 14-bit register count; you cannot configure precision beyond the
~0.81% standard error.

---

## 8. Streams

Introduced in Redis 5.0. A Stream is an **append-only, immutable, persistent
log** of entries. Each entry has a server-generated ID `<ms>-<seq>` and an
arbitrary set of field/value pairs.

### Concepts

- **Producer:** `XADD` appends entries; IDs auto-increase.
- **Single consumer:** `XREAD` with `BLOCK` and a cursor (`$` for "tail", or
  a specific ID) reads new entries.
- **Consumer group:** named group of consumers that share consumption of the
  stream. Each group tracks its own pending entries list (PEL). Consumers
  acknowledge with `XACK`; unacked entries stay pending and can be claimed by
  another consumer via `XCLAIM` after their idle time exceeds a threshold.
- **XPENDING** shows pending entries per group; **XCLAIM** transfers them;
  **XAUTOCLAIM** automates this for stalled workers.

### Commands

```bash
XADD events '*' user 42 action click target banner       # * = auto ID
XLEN events
XRANGE events - +
XREAD COUNT 10 BLOCK 5000 STREAMS events $               # tail-read
XGROUP CREATE events grp1 $ MKSTREAM
XREADGROUP GROUP grp1 consumer-1 COUNT 10 STREAMS events >
XPENDING events grp1
XCLAIM events grp1 consumer-2 0 1690000000123-0          # min-idle, ID
XACK events grp1 1690000000123-0
XTRIM events MAXLEN 10000                                 # cap stream length
XADD events MAXLEN ~ 10000 '*' user 7 action view         # approximate cap
```

### Use cases

- **Event log / audit log** — append-only, bounded with `XTRIM` (or `MAXLEN ~ `
  on `XADD` for approximate, faster trimming).
- **Message queue with consumer groups** — multiple consumers sharing the work,
  at-least-once delivery via the PEL + `XCLAIM`.
- **Fan-out / replay** — multiple groups reading from the same stream
  independently.

### Streams vs Pub/Sub

| Property            | Pub/Sub                          | Streams                                  |
|---------------------|----------------------------------|------------------------------------------|
| Persistence         | None (fire and forget)           | Yes, entries persisted until trimmed      |
| Delivery            | Push, to currently subscribed clients | Pull (`XREAD`) or push (`XREAD BLOCK`)  |
| Consumer groups     | No                               | Yes, with per-group PEL and acks         |
| Replay history     | No                               | Yes, by ID/range                          |
| Backpressure        | None (subscribers can be dropped)| Producer stores; readers can lag          |
| Cross-process order | None                             | Total order on the stream                 |

Use Pub/Sub for *ephemeral notifications* (e.g., cache invalidation hints,
config reload signals); use Streams for *durable event logs and queues*.

### Streams vs Kafka

- Streams are **in-memory** (optionally persisted), scale per Redis instance
  (max memory bound). Kafka is disk-based, partitioned across brokers,
  designed for huge throughput and retention.
- Streams give you consumer groups, offset tracking, at-least-once semantics,
  and acks — the same vocabulary Kafka uses — but on a single Redis process or
  a Redis Cluster shard.
- For high-throughput, multi-broker streaming, Kafka; for in-app message queue
  or replayable event log at moderate rate, Streams are far simpler.
- Streams are a single partition per key; scaling requires sharding by key
  prefix across multiple streams/cluster nodes.

---

## 9. Geospatial

The GEO commands (`GEOADD`, `GEORADIUSBYMEMBER`, `GEOSEARCH`, etc.) are
**backed by a Sorted Set**: longitude/latitude are encoded into a 52-bit
geohash integer used as the ZSET score.

### Commands

```bash
GEOADD places -122.4194 37.7749 SF -122.0808 37.4275 PA -118.2437 34.0522 LA
GEODIST places SF LA km
GEOSEARCH places FROMLONLAT -122.4 37.8 BYRADIUS 100 km ASC COUNT 20
GEOSEARCH places FROMMEMBER SF BYRADIUS 100 km ASC COUNT 20 WITHCOORD WITHDIST
```

`GEORADIUS`/`GEORADIUSBYMEMBER` are deprecated since Redis 6.2 in favor of
`GEOSEARCH`/`GEOSEARCHSTORE`.

### Use cases

- **Nearby search** — drivers, stores, users near a point.
- **Radius queries** — "show me 20 closest coffee shops sorted ascending".

---

## 10. Redis 8 additions: vector sets, JSON, probabilistic types

Redis 8 (GA May 2025) folded the former Redis Stack modules into the core
open-source distribution and added one brand-new type. Interviewers now expect
you to know these exist in plain Redis, not as add-ons. (Valkey does **not**
ship these — it covers the classic types above; some are available to Valkey
as modules.)

### Vector sets — Redis as a vector store

The first genuinely new core data type in years, designed for AI workloads:
store high-dimensional embeddings and query by similarity. Modeled on sorted
sets — members carry a *vector* instead of a score — with HNSW-based
approximate nearest neighbor search and optional quantization to cut memory.

```bash
VADD docs VALUES 3 0.1 0.9 0.3 doc:1          # add embedding for doc:1
VADD docs VALUES 3 0.2 0.8 0.4 doc:2
VSIM docs VALUES 3 0.15 0.85 0.35 COUNT 5     # 5 nearest members
VSIM docs ELE doc:1 COUNT 5                   # nearest to an existing member
VDIM docs                                      # dimensionality
```

Use cases: **semantic caching** (cache LLM answers keyed by embedding, serve
"close enough" queries from cache), recommendation/similar-items lookups, and
RAG retrieval when your corpus fits in memory next to the cache you already
run. Trade-off vs dedicated vector databases (or PostgreSQL + pgvector — see
[PostgreSQL advanced features](../postgresql/03-advanced-features.md)):
Redis gives in-memory latency and zero extra infrastructure, but memory-bound
capacity and a younger ecosystem; pgvector keeps vectors transactionally next
to your relational data on disk.

### JSON

A real document type (`ReJSON` heritage): store JSON, read/update at a path
without rewriting the whole document.

```bash
JSON.SET user:1 $ '{"name":"ada","age":36,"tags":["admin"]}'
JSON.GET user:1 $.name
JSON.NUMINCRBY user:1 $.age 1
JSON.ARRAPPEND user:1 $.tags '"builder"'
```

Compared to Hashes: JSON supports nesting and arrays and pairs with the Redis
Query Engine for secondary indexing/search over documents; Hashes are flat
field→string maps but cheaper. Compared to storing JSON in a String: no
read-modify-write cycle for partial updates.

### Probabilistic structures

Beyond HyperLogLog (which was always in core), Redis 8 bundles:

- **Bloom filter** (`BF.ADD`, `BF.EXISTS`) — "definitely not present / probably
  present"; the standard cache-penetration defense.
- **Cuckoo filter** (`CF.*`) — like Bloom but supports deletion.
- **Count-min sketch** (`CMS.*`) — approximate per-item frequency counts.
- **Top-K** (`TOPK.*`) — heavy hitters, e.g. "top 10 queried keys today".
- **t-digest** (`TDIGEST.*`) — approximate percentiles (p99 latency) over streams.

All trade exactness for O(1)-ish memory — same philosophy as HLL. Time series
(`TS.*`) is also included in Redis 8 core.

---

## Pub/Sub

Pub/Sub is a *separate* channel mechanism disjoint from keyspace data:
publishers `PUBLISH channel message`, subscribers `SUBSCRIBE channel`. Messages
are **not persisted anywhere**; if no subscriber is connected, the message is
lost. Each subscriber gets its own copy (fan-out).

```bash
# Terminal A
SUBSCRIBE chat

# Terminal B
PUBLISH chat "hello"
```

`PSUBSCRIBE pattern.*` subscribes by glob pattern.

Key facts to remember:
- No acknowledgment, no replay, no consumer groups, no backpressure.
- Subscriber with a slow client can be force-disconnected by Redis once its
  output buffer exceeds `client-output-buffer-limit pubsub`.
- Use cases: cache invalidation hints, presence/co-presence, config-broadcast,
  simple chat backends where durability is not required.

---

## Key expiration

TTL keys are stored in an **expires dictionary** sampled by Redis.

- **Lazy expiration:** when a key is accessed, Redis checks the stored expire
  timestamp and removes the key if expired. Cheap, but stale keys sit in
  memory until they're touched.
- **Active expiration cycle:** a background job runs ~10 times/sec per DB.
  Each cycle it samples a small set of keys with TTLs set, deletes expired
  ones, and adaptively increases sampling if many expirations are found. The
  cycle is short (~25 ms by default) to avoid latency spikes.
- TTL is specified in **seconds** (`EXPIRE`, `EXPIREAT`) or **milliseconds**
  (`PEXPIRE`, `PEXPIREAT`, and the `PX`/`PXAT` flags on `SET`). Query with
  `TTL` (seconds) or `PTTL` (ms). `-1` = no TTL, `-2` = key doesn't exist.
- `PERSIST <key>` removes the TTL, making the key live forever.

```bash
SET k v
EXPIRE k 30
TTL k          # seconds remaining
PTTL k         # ms
PERSIST k
```

---

## Eviction policies

When `maxmemory` is reached, Redis uses the configured `maxmemory-policy` to
decide what to free:

| Policy              | Description                                                            |
|---------------------|------------------------------------------------------------------------|
| `noeviction`        | Return OOM errors for writes; default for non-cache workloads.         |
| `allkeys-lru`       | Evict least-recently-used key of **all** keys.                         |
| `allkeys-lfu`       | Evict least-frequently-used key of all keys (LFU, probabilistic).      |
| `volatile-lru`      | LRU only among keys with a TTL set.                                    |
| `volatile-lfu`      | LFU only among keys with a TTL set.                                    |
| `volatile-ttl`      | Evict the key with the soonest TTL among TTL'd keys.                   |
| `volatile-random`   | Random TTL'd key.                                                      |
| `allkeys-random`    | Random key (rarely useful).                                            |

LRU/LFU are **approximate** — Redis samples a small number of keys
(`maxmemory-samples`, default 5) and evicts the best candidate from the sample.
Higher sample size → closer to true LRU/LFU, more CPU per eviction.

### When to choose which

- **Pure cache** with no other persistent data: `allkeys-lru` (or `allkeys-lfu`
  if access has clear frequency skew).
- **Session store** where every key has a TTL: `volatile-lru`.
- **Distributed lock store** or any data store where losing arbitrary keys
  corrupts application state: `noeviction`. Monitor memory and sharding
  instead.
- **TTL-based cache mixed with long-lived keys**: `volatile-*` so long-lived
  keys are never touched.

```bash
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru
CONFIG SET maxmemory-samples 10
```

---

## Memory management

- **`maxmemory`** caps the process resident memory used by the dataset. When
  reached, eviction fires. Note this excludes some overhead (replication
  buffers, the AOF buffer, large client output buffers) — reserve headroom.
- **`INFO memory`** shows `used_memory`, `used_memory_rss`, `mem_fragmentation_ratio`
  (`rss / used`). A ratio ~1.0 is healthy; >1.5 indicates fragmentation (often
  fixed by active defrag, `activedefrag yes`); <1.0 means the OS is sharing
  pages (common after a fork).
- **Big keys bloating memory** — use `MEMORY USAGE <key> SAMPLES 0` to estimate
  total memory consumed by a key (with `SAMPLES 0` to measure all elements).
- **Lazy free** (`UNLINK`, `FLUSHALL ASYNC`, `DEL` over `lazyfree-*` policies)
  frees memory in a background thread, preventing main-thread stalls on
  deletions of large keys.

---

## Single-threaded model

Redis executes **command execution** in a single main thread. This is the
classic reason Redis is easy to reason about: no locks between commands,
no race conditions in application logic.

- I/O: socket reads/writes are *not* the main thread's bottleneck thanks to
  **I/O threading** (`io-threads`, `io-threads-do-reads`), introduced in **6.0**.
  Read/parse and write/socket output can be parallelized across worker threads,
  while the **actual command still runs on the main thread**. So Redis is
  single-threaded in the data-access sense, but multithreaded for I/O.
- Persistence snapshots are taken by a **`fork()`** child process.
- Async deletion, AOF fsync, lazy-free, and BIO threads handle the rest.

### Why is this single-threaded model fast?

1. **In-memory data** — no disk I/O on the hot path, microsecond operations.
2. **No context switching** between threads for command execution.
3. **No lock overhead** — locking on every data structure would dominate CPU
   at the in-memory speed; avoiding it is a big win.
4. **Pipelining / batching** — multiple commands sent without waiting for
   each reply collapse the RTT into a single round-trip.

The model degrades when commands are *slow* on huge data: `KEYS *`, `LRANGE
key 0 -1` on a giant list, `SMEMBERS` on a giant set, `SORT` on a large
input, `DEBUG SLEEP`, or unjustified Lua scripts. They block *all* other
clients — the famous "do not use KEYS in production" rule.

---

## Pipelining

Pipelining means the client sends multiple commands in one TCP write without
waiting for replies; Redis queues them and replies in order. The win is
**RTT reduction**: 1000 round trips at 1 ms RTT = 1000 ms; pipelined into
one round trip = ~1 ms plus command time.

```bash
printf 'SET k1 v1\nSET k2 v2\nINCR c\n' | redis-cli --pipe
```

Most client libraries expose pipelining/transactions via a single
multi-command API (StackExchange `CreateBatch`, Lettuce `RedisPipeline`, etc.).

Not to be confused with transactions (MULTI/EXEC): pipelining is purely a
transport-level optimization and does not affect atomicity.

---

## Transactions: MULTI / EXEC / DISCARD / WATCH

A Redis transaction is a **batch of commands executed sequentially and
without interruption** by other clients. It is *not* ACID.

```bash
MULTI
SET foo bar
INCR counter
EXEC              # atomically run; output a list of replies
DISCARD           # abort, queue cleared
```

- Commands queued in `MULTI` are run on the main thread back-to-back, on the
  same connection — so no other client interleaves between them. That is the
  atomicity guarantee.
- If a queued command is malformed (syntax error), the whole `EXEC` is aborted.
- If a command is well-formed but fails at runtime (e.g., `INCR` on a list),
  the **other commands in the transaction still execute**, and there is **no
  rollback**. Redis has no rollback semantics.

### WATCH — optimistic locking

`WATCH key [key ...]` places a watch on the key(s); if any is modified by
another client before `EXEC`, the whole transaction aborts (returns nil).

```bash
WATCH balance
val = GET balance
MULTI
SET balance (val - 10)
EXEC                  # nil if balance was modified between WATCH and EXEC → retry
```

This is the canonical optimistic-locking pattern for check-then-set in Redis.
It enables safe operations that would otherwise race.

### Transactions vs Lua

Both provide atomic execution on the main thread. Lua scripts can branch and
read intermediate results; MULTI cannot (you cannot base queued commands on
prior results). Lua scripts can be cached with `EVALSHA` and reused.

---

## Lua scripting

A Lua script runs **atomically**: from the moment it starts to the moment it
finishes, no other command executes. Because Redis is single-threaded for
command execution, this is naturally atomic without any mutexes.

```lua
-- Release a distributed lock (token CAS)
-- KEYS[1] = lock key, ARGV[1] = token value
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
```

Run with:

```bash
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:order mytoken
```

Cache the script and run by hash to avoid resending the body:

```bash
SCRIPT LOAD "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end"
# returns SHA1, then:
EVALSHA <sha1> 1 lock:order mytoken
```

Caveats:
- Scripts block everything — keep them short and avoid O(N) loops.
- In cluster mode, all keys accessed in a single script must hash to the same
  slot (use `{tag}` to guarantee).
- Since Redis 7.0, scripts replicate by **effects**: the writes the script
  performs are sent to replicas/AOF, not the script body. (Older "the script
  re-executes on the replica" behavior is gone — worth knowing, since
  non-deterministic scripts were the reason.)
- The script cache is not persisted: after a restart or failover, `EVALSHA`
  can return `NOSCRIPT` and clients must fall back to `EVAL`. This is exactly
  the operational gap Functions close.

---

## Redis Functions (7.0+)

Functions (introduced in Redis 7.0) are a **named, persisted** alternative to
`EVAL`. They live in a library registered on the server with `FUNCTION LOAD`,
survive restarts, can be copied to replicas automatically (vs scripts which
are stateless), and can be invoked by name.

```bash
FUNCTION LOAD '#!lua name=mylib
 redis.register_function("ping", function() return "pong" end)
'
FCALL ping 0
FUNCTION LIST
FUNCTION DELETE mylib
```

Conceptually Functions replace plain `EVAL` strings in any non-trivial
deployment: you version your server-side code the same way you version your
schema. They run with the same atomicity guarantees as scripts.

---

## Quick reference: internal encodings cheat sheet

| Type        | Small encoding                       | Large encoding                  |
|-------------|--------------------------------------|---------------------------------|
| String      | `int`, `embstr` (≤44B)               | `raw`                            |
| List        | `listpack`                           | `quicklist` (listpack nodes)     |
| Hash        | `listpack`                           | `hashtable`                      |
| Set         | `intset` (all ints ≤512), `listpack` | `hashtable`                      |
| ZSET        | `listpack`                           | `skiplist + hashtable`           |
| Stream      | `listpack`                           | `radix tree of listpacks`         |
| Vector set  | flat array (small / quantized)       | HNSW graph                       |

Verify with `OBJECT ENCODING <key>`.