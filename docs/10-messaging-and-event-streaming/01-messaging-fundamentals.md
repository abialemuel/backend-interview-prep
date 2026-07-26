# Messaging Fundamentals

Everything in this file is broker-agnostic. Whether the interview is about Kafka, SQS, RabbitMQ, or NATS, the questions that carry the most signal — delivery guarantees, ordering, idempotency, backpressure — are answered the same way everywhere, because they are consequences of distributed systems physics, not vendor choices. Learn these once and every broker becomes a configuration detail.

## The three messaging models

The first clarifying question in any messaging discussion: **who consumes the message, and does it survive being consumed?**

### Point-to-point queues

One message, one consumer. Producers put messages on a queue; a pool of competing consumers pulls from it; each message is processed by exactly one of them and then deleted. The queue is a **work distribution** mechanism: it load-balances tasks across workers and buffers bursts.

```
producer --> [ queue ] --> worker 1
                      \--> worker 2   (each message goes to ONE worker)
                       \-> worker 3
```

This is the model behind background jobs: send-this-email, resize-this-image, charge-this-card. SQS, RabbitMQ queues, and Laravel/Sidekiq-style job queues all live here. Scaling is trivial — add workers — precisely because no worker cares what the others are doing.

### Publish/subscribe

One message, many consumers. Producers publish to a topic; every subscriber gets its own copy. Pub/sub is a **fan-out** mechanism: the producer does not know or care who is listening, which is the whole point — you can add a new consumer (analytics, audit, search indexing) without touching the producer.

```
producer --> [ topic ] --> notification service   (every subscriber
                      \--> analytics service       gets EVERY message)
                       \-> search indexer
```

Classic pub/sub (SNS, Redis Pub/Sub, NATS core) is **fire-and-forget**: if a subscriber is offline when the message is published, it never sees it. That property disqualifies plain pub/sub for anything that must not be lost — which is why the pattern in practice is SNS fanning out into per-consumer SQS queues, or a durable stream.

### Streams (append-only logs)

A stream is a durable, ordered, append-only log. Messages are not deleted on consumption; instead each consumer tracks its own **position (offset)** in the log, and the log is trimmed by a retention policy (time, size, or compaction), not by acknowledgment. This one design change unlocks everything that distinguishes Kafka/Kinesis/Redpanda/Redis Streams from queues:

- **Multiple independent consumers** at different positions — pub/sub semantics with durability.
- **Replay.** A new service, a bug fix, or a rebuilt cache can re-read history from any offset.
- **Ordering** within a partition, because the log itself is the order.
- **The log as data**, not just transport — the foundation for event sourcing and CDC (file 03).

The trade: consumers must manage offsets (a whole class of bugs — see the Kafka file), per-message semantics like delay or priority are awkward, and selective deletion is impossible by design.

| | Queue | Pub/Sub | Stream |
| --- | --- | --- | --- |
| Consumers per message | One | All subscribers | All consumer groups, at their own pace |
| Message lifetime | Until acked/deleted | Instant (gone if you missed it) | Retention window |
| Replay | No | No | Yes |
| Primary use | Task distribution | Fan-out notification | Event backbone, replayable history |
| Examples | SQS, RabbitMQ | SNS, Redis Pub/Sub, NATS core | Kafka, Kinesis, Redis Streams, NATS JetStream |

## Delivery guarantees

The most probed topic in this file. There are exactly three, and they are defined by **what happens across failures**, not by the happy path.

### At-most-once

Fire and forget. The producer sends and does not wait for an acknowledgment; the consumer acknowledges *before* processing (or there is no ack at all). If anything fails, the message is lost — but it is never duplicated. Cheapest and fastest. Acceptable only when a lost message is genuinely fine: metrics, logs, presence pings, real-time telemetry where the next data point supersedes this one anyway.

### At-least-once

The default of every serious system. The producer retries until the broker confirms; the consumer acknowledges only *after* processing completes. Nothing is lost — but **duplicates are guaranteed to occur eventually**, and both ends produce them:

- Producer side: the broker persisted the message but the ack was lost in transit; the producer retries; the broker now has two copies.
- Consumer side: the consumer processed the message but crashed before acking; the broker redelivers; the message is processed twice.

The consequence, and the sentence worth saying verbatim in an interview: **at-least-once delivery plus an idempotent consumer is the standard correctness recipe for asynchronous systems.** You accept duplicates in transport and neutralize them in processing.

### Exactly-once

The honest framing: exactly-once *delivery* over an unreliable network is impossible in general (it collapses into the Two Generals problem — you can never confirm the final ack). What systems actually provide is **exactly-once processing semantics** (or "effectively once"): each message's *effects* are applied once, achieved through some combination of:

1. **Idempotent producers** — broker-side deduplication of retried sends (Kafka's idempotent producer, SQS FIFO's deduplication ID).
2. **Transactions** — atomically committing the processing output together with the consumption position (Kafka read-process-write transactions).
3. **Idempotent consumers** — application-level dedup, which works with *any* broker and is where you should reach first.

Costs: latency (transaction coordination, fencing), throughput, and operational complexity. And critically, broker-level exactly-once ends at the broker's boundary — the moment a consumer calls an external API (a payment gateway, an email provider), you are back to needing application-level idempotency. Interviewers love this boundary; see the Kafka file and Q&A file.

| Guarantee | Loss | Duplicates | Cost | Use when |
| --- | --- | --- | --- | --- |
| At-most-once | Yes | No | Lowest | Loss is acceptable (metrics, telemetry) |
| At-least-once | No | Yes | Moderate | Default — pair with idempotent consumers |
| Exactly-once | No | No (in effect) | Highest | Money movement, stream aggregations |

## Idempotent consumers

Since at-least-once is the pragmatic default, idempotency is not optional — it is the load-bearing wall. Three ways to get it, in order of preference:

1. **Naturally idempotent operations.** `SET status = 'shipped'` is idempotent; `UPDATE balance = balance - 10` is not. Where you can, design writes as absolute assignments keyed by identity rather than relative mutations. Upserts (`INSERT ... ON DUPLICATE KEY UPDATE`) are your friend.
2. **Deduplication by idempotency key.** Every message carries a unique ID (event ID, or a business key like `order_id + state`). The consumer records processed IDs and skips repeats. The critical detail: **the dedup record and the business write must commit in the same transaction**, otherwise a crash between them recreates the duplicate.

```sql
BEGIN;
INSERT INTO processed_messages (message_id) VALUES ('evt_123'); -- unique key
-- if this violates the unique constraint, ROLLBACK and ack: already done
UPDATE orders SET status = 'paid' WHERE id = 42;
COMMIT;
```

3. **Version/state checks.** The message carries a version or expected prior state; the consumer applies it only if the entity is in that state (`WHERE status = 'pending'`). This handles both duplicates *and* out-of-order delivery, which pure ID dedup does not.

Practical notes: the dedup table needs a TTL/cleanup (dedup forever is dedup that fills your disk — scope it to the broker's redelivery horizon plus margin); and if the consumer's output is a call to an external system, pass the idempotency key downstream (Stripe-style `Idempotency-Key` headers) so the *whole chain* is idempotent.

## Ordering

Global ordering across a distributed system is a mirage — it requires a single serialization point, which caps throughput at one node. Every scalable broker therefore offers **partial ordering**:

- **Kafka/Kinesis:** order is guaranteed *within a partition/shard* only. You choose the partition key; all events for one entity (one order, one user) go to one partition and arrive in order relative to each other.
- **SQS Standard:** no ordering at all (best-effort). **SQS FIFO:** ordering within a *message group ID* — the same idea as a partition key, with throughput limits per group.
- **RabbitMQ:** ordered within a queue with a single consumer; competing consumers or requeues break it.

The interview-grade insight: **you almost never need global ordering — you need per-entity ordering.** Events for order #42 must be ordered relative to each other; their order relative to order #99's events is irrelevant. So: partition by entity ID, and design consumers to tolerate cross-entity interleaving. And remember what breaks ordering even within a partition: consumer-side parallelism (processing a batch concurrently), retries that skip ahead, and DLQ-then-continue (the failed message's successors are processed before it). If a later event genuinely cannot be applied before an earlier one, the version/state check above is the guard.

## Dead-letter queues and poison messages

A **poison message** is one that fails processing deterministically — malformed payload, a bug in the consumer, a referenced entity that does not exist. Without a plan, it produces the worst failure mode in messaging: the consumer receives it, throws, the broker redelivers, it throws again — an infinite retry loop that, on an ordered queue, **blocks every message behind it**. One bad message halts the pipeline.

The standard defense is a **dead-letter queue (DLQ)**: after N failed delivery attempts (SQS `maxReceiveCount`, RabbitMQ `x-dead-letter-exchange`, a Kafka consumer writing to a `topic.DLQ`), the message is moved aside and the pipeline continues. What separates senior answers from junior ones is what happens *after*:

- **Alert on DLQ depth.** A DLQ nobody watches is a data-loss mechanism with extra steps.
- **Preserve diagnostics.** Attach the exception, attempt count, and original topic/offset as metadata.
- **Have a redrive path.** After fixing the bug, replay DLQ messages to the source queue (SQS has native redrive). Redriven messages are duplicates by definition — idempotency again.
- **Distinguish transient from permanent failures in the consumer.** Retry timeouts and 5xx with backoff; send validation failures straight to the DLQ on attempt one. Retrying a deterministic failure five times is just five times the log noise.
- **Mind ordering.** DLQ-ing message 5 for order #42 and then processing message 6 may violate your invariants. If per-entity ordering is sacred, you may need to park *the entity* (pause its key) rather than skip the message.

## Backpressure

Backpressure is what happens when producers outpace consumers. The queue is a buffer, and a buffer only converts a *temporary* rate mismatch into latency; a *sustained* mismatch fills any buffer eventually. Your options, roughly in order of escalation:

1. **Buffer and absorb.** The default. Works for bursts; watch queue depth / consumer lag as a first-class SLO metric. The key derived metric is **time-to-drain** (depth ÷ net drain rate), not raw depth.
2. **Scale consumers.** Add workers (queues make this trivial) or partitions+consumers (streams; note Kafka parallelism is capped by partition count — file 02). Only helps if the bottleneck is the consumers, not a shared downstream (the database behind the consumers is the usual real limit).
3. **Slow the producer.** Real backpressure: bounded internal buffers that block the producing thread (Kafka's producer blocks when its buffer fills; Go channels do this naturally), or explicit rate limiting at the API edge.
4. **Shed load.** Drop or sample low-value messages, or reject new work at the edge with 429s. Deciding *in advance* what is droppable is an architecture decision, not an incident-time one.

!!! warning "Unbounded buffering is not resilience"
    "The queue will absorb it" is only true for bursts. Under sustained overload, an unbounded queue converts an obvious fast failure into a slow, opaque one: hours of latency, gigabytes of backlog, and a consumer fleet that can never catch up. Bound something — the buffer, the producer rate, or the acceptance of work.

## Mechanics interviewers probe

A grab-bag of smaller concepts that show up as follow-up questions. Each is one sentence of definition and one of trade-off — that is usually all the interviewer wants.

### Push vs pull consumers

Push (RabbitMQ delivering to consumers, SNS invoking Lambda): the broker controls the rate, giving low latency but requiring a prefetch/credit mechanism so a slow consumer is not buried. Pull (Kafka `poll()`, SQS `ReceiveMessage`): the consumer controls the rate — backpressure is inherent, batching is natural, and a slow consumer simply pulls less; the cost is polling latency (mitigated by long polling — SQS `WaitTimeSeconds=20` also cuts empty-receive cost dramatically). Log-based systems are pull-based almost by necessity: only the consumer knows its own position and capacity.

### Visibility timeout and ack deadlines

In SQS, receiving a message does not delete it — it becomes *invisible* for the visibility timeout while you process; delete-on-success, or it reappears for redelivery. Set the timeout too short and a healthy-but-slow consumer causes duplicate processing (the message reappears while still being worked); too long and a crashed consumer delays redelivery by the full window. The same concept appears as RabbitMQ unacked message + consumer timeout and Kafka's `max.poll.interval.ms`. Long-running work should extend the timeout as it goes (heartbeat pattern) rather than setting a giant one upfront.

### Request-reply over messaging

Sometimes you need an answer back over an async channel: the requester includes a `reply_to` queue/subject and a `correlation_id`; the responder sends the result there. RabbitMQ (direct reply-to) and NATS (built-in request-reply, arguably its killer feature) make this first-class. The trade-off to name: you have rebuilt synchronous RPC with more moving parts — justified when you want the broker's load-leveling, routing, or fan-out even for request-shaped traffic; otherwise just make an HTTP/gRPC call.

### Delayed and scheduled messages

"Retry this in 5 minutes," "send the reminder in 24 hours." SQS has native per-message delays (up to 15 minutes) and RabbitMQ has TTL + dead-letter tricks or the delayed-exchange plugin; Kafka has **nothing native** — the log is strictly append-and-read — so delay on Kafka means consumer-side scheduling, tiered retry topics (`retry-5m`, `retry-1h`) with pausing consumers, or an external scheduler. For genuinely long or per-message-precise schedules, a database-backed scheduler or a workflow engine (Temporal) is usually more honest than broker contortions.

### Message TTL and priorities

Per-message TTL ("this is worthless after 30 seconds" — presence updates, quotes) is native in RabbitMQ and implied by retention elsewhere; expired-in-queue messages should be dropped, not processed late. Priority queues (RabbitMQ supports them natively) look attractive and usually aren't: under low load everything is fast anyway, and under high load a full queue of high-priority messages starves the rest. The more robust pattern is **separate queues per class of traffic** with separately scaled consumers — isolation instead of reordering.

## Broker comparison: RabbitMQ vs SQS/SNS vs Kafka vs NATS

The senior-level framing is never "which is best" but "which fits this workload." The dominant axis: **smart broker / dumb consumer** (RabbitMQ — routing, per-message TTLs, priorities live in the broker) versus **dumb broker / smart consumer** (Kafka — the broker is a fast log; semantics live in clients).

| | RabbitMQ | SQS + SNS | Kafka | NATS (+ JetStream) |
| --- | --- | --- | --- | --- |
| Model | Smart broker: exchanges, bindings, routing keys | Managed queue (SQS) + fan-out (SNS) | Partitioned, replicated append-only log | Lightweight pub/sub core; JetStream adds persistence/streams |
| Ordering | Per queue, single consumer | None (Standard) / per group (FIFO) | Per partition | Per subject (JetStream) |
| Replay | No (consumed = gone) | No | Yes — core feature | Yes with JetStream |
| Throughput ceiling | Tens of thousands msg/s per node | High (managed, but FIFO caps per group) | Millions msg/s, horizontal via partitions | Very high, very low latency |
| Delivery | At-least-once (acks) | At-least-once; FIFO adds dedup window | At-least-once; exactly-once with idempotence + transactions | At-most-once core; at-least/exactly-once via JetStream |
| Routing sophistication | Best in class (topic/headers/fanout exchanges, per-message TTL, priorities, delays) | Basic (SNS filter policies) | None in broker — consumers filter | Subject hierarchy wildcards |
| Ops burden | Moderate (Erlang cluster, quorum queues) | Zero — fully managed | Highest self-hosted (KRaft simplified it); managed options (MSK, Confluent) | Very low — single small binary |
| Sweet spot | Complex task routing, RPC-ish work queues, per-message control | AWS-native services wanting zero ops | Event backbone, high-volume streams, replay, stream processing, CDC | Service-to-service messaging, IoT/edge, low-latency internal comms |

Decision heuristics worth saying out loud:

- **Background jobs for a product on AWS** → SQS (Standard unless you truly need FIFO). Zero ops beats every feature you will not use.
- **Complex routing** (this message to these three queues based on headers, with priorities and per-message delay) → RabbitMQ; emulating that on Kafka means building a broker inside your consumers.
- **Event backbone / high volume / multiple independent readers / replay / audit** → Kafka. If you need yesterday's events again, only a log can give them to you.
- **Lightweight internal messaging, request-reply between services, edge/IoT** → NATS; it is to messaging what Redis is to storage — small, fast, and simple.
- Mixed workloads are normal: SQS for jobs *and* Kafka for the event stream in the same architecture is the common end state, not a smell.

The next file goes deep on Kafka, because it is both the most-asked-about broker and the one with the most instructive internals.
