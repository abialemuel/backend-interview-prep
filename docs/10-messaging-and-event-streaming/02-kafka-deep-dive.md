# Kafka Deep Dive

Kafka is the interview default for event streaming, and its internals are unusually instructive: almost every hard messaging concept (partitioned ordering, offset management, rebalancing, exactly-once) has a concrete, inspectable implementation here. This file assumes the fundamentals file and is current for the Kafka 4.x era (4.2 stable as of early 2026): **KRaft is the only mode — ZooKeeper is gone** — and the KIP-848 consumer protocol is the default. If your mental model is Kafka 2.x, the deltas are called out inline.

## Architecture

### The log, topics, and partitions

Kafka's core abstraction is a **partitioned, replicated, append-only log**. A **topic** is a named stream; each topic is split into **partitions**; each partition is an ordered, immutable sequence of records, each identified by a monotonically increasing **offset**. Records are `(key, value, timestamp, headers)`; both key and value are opaque bytes to the broker — serialization is a client concern (see schema registry below).

```
topic "orders", 3 partitions:

P0: [0][1][2][3][4][5] --> appends here
P1: [0][1][2][3]       --> appends here
P2: [0][1][2][3][4]    --> appends here
```

Partitions are the unit of everything: **parallelism** (one partition is consumed by at most one consumer within a group), **ordering** (guaranteed only within a partition), and **placement** (a partition lives entirely on the brokers that host its replicas). Records with the same key hash to the same partition — this is how per-entity ordering is achieved: key by `order_id` and all of order #42's events are totally ordered. Records with a null key are spread via the "sticky" partitioner (batches stick to one partition, then switch — better batching than round-robin per record).

Two consequences worth volunteering in interviews: partition count is effectively **the ceiling on consumer parallelism** for a group, and changing partition count **breaks key→partition mapping** (old and new events for the same key land in different partitions), so overprovision partitions modestly upfront rather than resizing under load.

### Brokers, replication, and the ISR

A **broker** is one Kafka server; a cluster is a set of brokers. Each partition has a configurable number of **replicas** (production default: 3). One replica is the **leader** — all produce and (normally) fetch traffic goes through it; the others are **followers** that replicate the leader's log. The **ISR (in-sync replica set)** is the leader plus the followers that are fully caught up. Two settings define your durability contract:

- `acks=all` (producer): the leader acknowledges only after all ISR members have the record.
- `min.insync.replicas=2` (topic/broker): a write with `acks=all` fails if the ISR shrinks below 2.

With RF=3 and `min.insync.replicas=2`, an acknowledged write survives the loss of any single broker, and you can still write with one broker down. This trio — `replication.factor=3, min.insync.replicas=2, acks=all` — is the standard durable configuration and a very common interview checkpoint. The **high watermark** (highest offset replicated to all ISR members) bounds what consumers can read: only fully-replicated records are visible, so consumers never see data that a leader failover could roll back. `unclean.leader.election.enable=false` (the default) means a partition with no live ISR member goes offline rather than electing a stale replica — availability sacrificed for consistency, exactly the CAP conversation from the system design section.

### KRaft: consensus without ZooKeeper

Historically, cluster metadata (broker membership, topic configs, partition leadership) lived in a separate ZooKeeper ensemble — a second distributed system to operate, and a scalability bottleneck. **KRaft** replaces it: metadata is stored in an internal Kafka topic (`__cluster_metadata`) replicated via the Raft consensus protocol among a quorum of **controller** nodes (dedicated, or combined with broker roles in small clusters). One of the controllers is the active leader; failover is near-instant because standby controllers already have the full metadata log in memory.

What to know for interviews: Kafka 4.0 **removed** ZooKeeper support entirely (KRaft-only); benefits are one system instead of two, much faster controller failover, and support for far more partitions per cluster (metadata handling scales as a log, not as ZooKeeper znodes). If a question or doc mentions ZooKeeper, it is describing pre-4.0 Kafka.

### Why Kafka is fast

A common warm-up question with a crisp answer. Kafka's write path is an append to the end of a file — **sequential I/O**, which disks (even SSDs, and especially the OS's own scheduling) handle at orders of magnitude above random writes. It leans entirely on the **OS page cache** rather than an application-level cache: recent log segments are usually in memory, so consumers reading near the tail never touch disk. Reads use **zero-copy** (`sendfile`): data moves from page cache to the network socket without passing through the JVM, avoiding copies and GC pressure. **Batching and compression** amortize per-record overhead across the wire and on disk, and the binary protocol keeps brokers doing almost no per-message work — no routing, no per-message bookkeeping, just offset arithmetic. The design lesson worth stating: Kafka is fast because the broker is *dumb* — it moves bytes and leaves semantics to clients.

## Producers

A producer batches records per partition, compresses batches (`compression.type=lz4|zstd` — practically free throughput, always enable), and sends them to partition leaders. The knobs that matter:

| Setting | What it controls | Trade-off |
| --- | --- | --- |
| `acks` | `0` = fire-and-forget, `1` = leader persisted, `all` = full ISR | Latency/throughput vs durability. `all` is the default since 3.0 |
| `linger.ms` / `batch.size` | How long/large batches accumulate before sending | Small latency cost (e.g., `linger.ms=5-20`) buys large throughput via bigger batches and better compression |
| `enable.idempotence` | Broker de-duplicates producer retries | Default `true` since 3.0; effectively no cost — leave it on |
| `retries` / `delivery.timeout.ms` | Automatic retry on transient errors | With idempotence on, retries are safe (no dupes, no reordering) |
| `max.in.flight.requests.per.connection` | Pipelining depth | With idempotence, up to 5 stays ordered |
| `buffer.memory` | Producer-side buffer | When full, `send()` blocks — built-in backpressure |

**Idempotent producer mechanics** (a favorite probe): each producer gets a **producer ID (PID)**; each batch carries a per-partition **sequence number**; the broker persists the last sequences per PID and rejects duplicates and gaps. This turns producer retries from "at-least-once with reordering risk" into "exactly-once *into the partition*, in order" — for a single producer session, to a single partition. Cross-partition and cross-session atomicity requires transactions (below).

## Consumer groups and rebalancing

A **consumer group** is a set of consumers sharing a `group.id` that divide a topic's partitions among themselves: each partition is owned by exactly one consumer in the group. Groups scale horizontally up to the partition count — the 5th consumer on a 4-partition topic sits idle. Different groups are independent: each has its own offsets, so "orders" can be consumed by the fulfillment group *and* the analytics group at their own pace. This is how the log unifies queueing (within a group) and pub/sub (across groups).

**Rebalancing** is the reassignment of partitions when membership or subscription changes: a consumer joins, leaves, crashes, or misses its heartbeats; or partitions are added. The classic protocol was **stop-the-world**: every consumer stopped, gave up all partitions, and rejoined — seconds of paused consumption, and the source of Kafka's most notorious operational pain (rebalance storms, where a slow consumer misses heartbeats, triggers a rebalance, which slows things further...). Cooperative/incremental rebalancing improved this, and the **KIP-848 protocol (default in 4.x)** redesigns it: the group coordinator (a broker) computes assignments and reconciles them incrementally, consumer-side; unaffected consumers keep consuming throughout, and heavy client-side "group coordination" logic is gone. Practical levers you should still know:

- `max.poll.interval.ms` (default 5 min): the max gap between `poll()` calls. Exceed it (slow batch processing) and the consumer is evicted → rebalance. The classic fix for "mysterious constant rebalances" is smaller `max.poll.records` or faster/async processing.
- `session.timeout.ms` / heartbeats: liveness detection; heartbeats run on a background thread, so this catches crashes, while `max.poll.interval.ms` catches stuck processing.
- **Static membership** (`group.instance.id`): a restarting consumer with the same instance ID reclaims its partitions without a rebalance — the difference between a rolling deploy causing N rebalances or zero.

## Offset management

Consumers periodically **commit** their position per partition to the internal `__consumer_offsets` topic. The committed offset is "the next offset to read" — where this group resumes after restart or rebalance. The commit timing *is* your delivery guarantee:

- **Commit after processing** (manual commit) → at-least-once. A crash between processing and commit means redelivery. The default correct choice; pair with idempotent processing.
- **Commit before/independently of processing** (`enable.auto.commit=true`, which commits every 5s on the poll thread) → can be effectively at-most-once for the tail of a batch: a crash after auto-commit but before finishing the batch **loses those messages silently**. Auto-commit is fine for tolerant workloads; for anything correctness-sensitive, commit manually after the work is durable.

`auto.offset.reset` governs what happens with *no* committed offset (new group, or offsets expired after `offsets.retention.minutes`, default 7 days): `earliest` reprocesses history, `latest` skips to now. A surprisingly common production incident: a consumer group idle longer than offset retention comes back, finds no offsets, and with `latest` silently skips everything in between. Know that story.

**Lag** — the gap between the partition's latest offset and the group's committed offset — is the single most important Kafka health metric (see pitfalls).

## Exactly-once semantics and transactions

Kafka's EOS builds in three layers:

1. **Idempotent producer** (above): exactly-once append per partition, per producer session.
2. **Transactions**: a producer with a `transactional.id` can write to multiple partitions *and commit consumer offsets* atomically. A transaction coordinator (broker-side) tracks the transaction in a log; commit/abort markers are written into each partition. The `transactional.id` also provides **zombie fencing**: each new producer instance bumps an epoch, and writes from the old instance (a "zombie" that was presumed dead but is still running) are rejected.
3. **Read-process-write loops**: the canonical EOS pattern. Consume from topic A, process, produce to topic B, and commit the input offsets — all in one transaction. Downstream consumers set `isolation.level=read_committed` so they never see records from aborted transactions. Kafka Streams packages this as `processing.guarantee=exactly_once_v2`.

```java
producer.initTransactions();
while (true) {
    var records = consumer.poll(timeout);
    producer.beginTransaction();
    for (var r : records) producer.send(transform(r));
    producer.sendOffsetsToTransaction(currentOffsets(consumer), groupMetadata);
    producer.commitTransaction();   // outputs + input offsets: atomic
}
```

The boundary to always state: this is exactly-once **within Kafka** (topic → topic). The moment the "write" is an external system — a MySQL row, a payment API — Kafka transactions cannot cover it, and you need an idempotent sink or the outbox/CDC patterns from file 03. Costs of EOS: added latency (transaction commit round-trips, `read_committed` consumers wait for markers), coordinator overhead, and more moving parts. Use it where duplicates are genuinely expensive (billing pipelines, stream aggregations where double-counting corrupts state); default to at-least-once + idempotency elsewhere.

## Retention and log compaction

Kafka trims logs by policy, not by consumption:

- **Delete retention**: drop log segments older than `retention.ms` (default 7 days) or beyond `retention.bytes`. The stream is a sliding window of history.
- **Log compaction** (`cleanup.policy=compact`): retain **at least the latest record per key**, forever. Old values for a key are garbage-collected in the background; a record with a null value (a **tombstone**) deletes the key (after `delete.retention.ms`). A compacted topic is a durable, replayable **table of latest state**: replay it from offset 0 and you rebuild the full current state. This powers `__consumer_offsets` itself, CDC topics, and Kafka Streams changelogs (KTables). Nuance worth knowing: compaction guarantees the latest value *survives*, not that only one value exists — readers must tolerate seeing older values for a key before the latest one.
- **Tiered storage** (production-ready since 3.9): older segments offloaded to object storage (S3), so retention can be weeks or months without scaling broker disks — long replay windows became much cheaper, which matters for event sourcing designs.

## Schema registry, Avro, and Protobuf

Brokers see bytes; producers and consumers must agree on format. Raw JSON works until the first uncoordinated schema change silently breaks a consumer. The standard fix is a **schema registry** (Confluent's, Karapace, AWS Glue): producers register schemas (Avro, Protobuf, or JSON Schema); each message carries a small schema ID; consumers fetch schemas by ID and deserialize. The registry's real value is **enforced compatibility checking** at registration time:

- `BACKWARD` (default): new schema can read old data — consumers upgrade first; you may delete fields and add optional ones.
- `FORWARD`: old schema can read new data — producers upgrade first.
- `FULL`: both. The safe default for topics with many independent consumers.

Avro is compact and registry-native with rich evolution rules (defaults required for added fields); Protobuf brings the gRPC ecosystem and good unknown-field semantics; JSON Schema is the pragmatic choice when humans need to read payloads. The choice matters less than *having* enforced compatibility — schema evolution strategy continues in file 03.

## Kafka Connect

Connect is Kafka's standardized integration framework: **source connectors** pull data from external systems into topics (Debezium's CDC connectors are the flagship — see file 03), and **sink connectors** push topics out to S3, Elasticsearch, JDBC databases, and warehouses. It runs as a worker cluster; connectors are configuration, not code — you declare the source/sink and Connect handles partitioning the work into tasks, offset tracking, retries, DLQ routing (`errors.tolerance` + `errors.deadletterqueue.topic.name`), and rebalancing tasks across workers. Single-message transforms (SMTs) do light per-record munging in the pipeline. When to reach for it: moving data between Kafka and *systems* — writing a bespoke consumer to copy a topic into S3 is reinventing a wheel with worse failure handling. When not to: business logic; the moment transformation stops being mechanical, you want a real consumer or a stream processor, not a chain of SMTs.

## Sizing and partition count

The perennial "how many partitions?" question has a defensible back-of-envelope method, and interviewers reward showing it:

1. **Target throughput ÷ per-partition throughput.** Measure (or assume conservatively) what one partition sustains end-to-end — often 5-25 MB/s produce-side, but the real limit is usually your *consumer's* per-partition processing rate. If you need 100k msg/s and one consumer instance handles 5k msg/s, you need ≥ 20 partitions just to allow 20 consumers.
2. **Add headroom for growth**, because resizing a keyed topic breaks key→partition mapping (same key, different partition, ordering lost across the boundary). 2-3x expected peak is a common allowance.
3. **Don't go silly.** Thousands of partitions per topic inflate leader election work, rebalance scope, producer/consumer memory (per-partition buffers), and end-to-end latency (more, smaller batches). KRaft raised cluster-wide partition ceilings substantially, but per-topic restraint still pays. Tens to low hundreds of partitions covers the vast majority of real topics.

Retention sizing is simpler arithmetic — ingest rate × retention window × replication factor = disk — and is the number that motivates tiered storage once the window grows past days.

## Monitoring: the metrics that matter

If asked "how do you know your Kafka setup is healthy?", name these, in this order:

| Metric | Why it matters |
| --- | --- |
| Consumer lag (and its growth rate / time-to-drain) | The user-facing health of every async pipeline; the one to put an SLO on |
| Under-replicated / under-min-ISR partitions | Durability is currently degraded; writes may start failing (`acks=all`) |
| Rebalance rate per group | Frequent rebalances = processing pauses; a storm looks like sawtooth lag |
| Produce/fetch request latency (p99) | Broker-side saturation — disk, network, or page cache pressure |
| Failed produce rate / `NotEnoughReplicas` errors | The durability contract actively rejecting writes |
| DLQ depth (if you run DLQ topics) | Silent data loss accumulating |
| Disk usage vs retention math | Kafka full = cluster down; the most preventable Kafka outage |

The trap answer is "broker CPU" — Kafka is rarely CPU-bound; it is disk-, network-, and *consumer*-bound, in that order of likelihood.

## Kafka's neighbors: Kinesis, Pulsar, Redpanda

One paragraph each, because "why Kafka and not X?" is a fair follow-up. **Kinesis** is AWS's managed partitioned log — same mental model (shards ≈ partitions, sequence numbers ≈ offsets) with zero cluster ops, but lower per-shard throughput caps, 7-day (extendable to 365) retention, thinner ecosystem, and per-shard pricing that punishes bursty scaling; the default when you want "Kafka-shaped but serverless on AWS" and volumes are moderate. **Pulsar** separates compute (brokers) from storage (BookKeeper), giving fast broker scaling, built-in geo-replication and multi-tenancy, and both queue and stream semantics natively — at the price of operating two distributed systems and a smaller community. **Redpanda** is a Kafka-API-compatible reimplementation in C++ — single binary, no JVM, strong tail latencies; you keep the entire Kafka client/tooling ecosystem, trading the Apache project's ubiquity for operational simplicity. The honest summary: the partitioned-log *model* won; the products differ in who operates it and how.

## Common pitfalls

The two below are the classic diagnostic interview scenarios; rehearse them as structured narratives.

### Hot partitions

One partition receives disproportionate traffic, so one broker and **one consumer** carry the load while the rest idle — adding consumers does nothing, because per-partition ordering pins a partition to a single group member. Causes: a low-cardinality or skewed partition key (tenant ID where one tenant is 60% of traffic; `country` when most users share one; null-key floods), or naturally hot entities. Diagnosis: per-partition metrics — message rate and consumer lag by partition; a hot partition shows one partition's lag growing while siblings are at zero. Fixes: choose a higher-cardinality key; composite keys (`tenant_id + order_id`) when you only need ordering per finer-grained entity; salt the hot key (`key + hash(id) % 8`) and accept ordering only within each salted sub-key; or handle the one pathological tenant on a dedicated topic. Note what you *cannot* do: repartitioning existing data in place — a new topic and a migration is the honest answer.

### Consumer lag

Lag = produced offset − committed offset, per partition. Growing lag means consumers are falling behind, and everything downstream is increasingly stale. The diagnosis tree:

1. **Is production up or consumption down?** Compare produce rate vs consume rate over time. A traffic spike with healthy consumers just needs time (or more consumers); a consumption drop is a consumer problem.
2. **All partitions or some?** All → systemic (slow downstream dependency, undersized group, GC). Some → hot partition or one sick consumer instance.
3. **Is the group rebalancing repeatedly?** Check rebalance metrics/logs. A consumer exceeding `max.poll.interval.ms` gets evicted → rebalance → pause → more lag → slower processing → evicted again: the rebalance storm loop. Fix the processing time (smaller `max.poll.records`, async/parallel handling within the poll loop), not the symptom.
4. **What is the consumer actually spending time on?** Usually not Kafka: a slow database write, a synchronous external API call per message, N+1 lookups. Batch the downstream writes, parallelize per-partition-safe work, cache lookups.
5. **Structural ceiling?** If per-consumer throughput is already optimized and consumer count = partition count, the only moves are more partitions (new topic or repartition, with the key-mapping caveat above) or cheaper processing.

The senior move is naming the metric you would page on: not raw lag, but **lag growth rate** or estimated time-to-drain against the staleness SLO of the consumer's purpose.

!!! tip "Kafka 4.x cheat-sheet for interviews"
    ZooKeeper removed — KRaft only. KIP-848 consumer protocol default: broker-driven, incremental, near-zero-pause rebalances. `acks=all` and idempotent producers are defaults. Queues for Kafka (share groups, KIP-932) is arriving as a first-class feature — per-message acks and more consumers than partitions, i.e., SQS-style semantics on Kafka topics — worth mentioning as "landing now," not as something to design around yet.
