# Messaging & Event Streaming

Almost every senior backend loop ends up here. The moment a design involves two services, a background job, a spike of writes, or a payment that must happen exactly once, the interviewer is really asking one question: **do you understand asynchronous communication and its failure modes?** Queues and event streams are how backend systems decouple, absorb load, and survive partial failure — and they are also where the subtle bugs live: duplicate deliveries, reordered events, poisoned messages, consumers that silently fall hours behind.

The reason this topic carries so much signal is that it cannot be faked with vocabulary. Anyone can say "put a queue in front of it." A senior engineer can say what the queue *guarantees* (and what it does not), what happens when the consumer crashes mid-message, why "exactly-once" is really "effectively-once" and what that costs, and how to pick between RabbitMQ, SQS, Kafka, and NATS for a specific workload rather than by fashion. As with the rest of this guide, concentrate on **trade-offs**: what you gain, what you pay, and under what conditions the trade flips.

Content is current as of mid-2026: Kafka is in the 4.x era (4.2 is the stable release), KRaft is the only mode — ZooKeeper is gone entirely — and the KIP-848 consumer rebalance protocol is the default. If you last touched Kafka in the 2.x/3.x days, the deep-dive file calls out what changed.

## What this section covers

| File | Description |
| --- | --- |
| README.md | This overview: why messaging matters in interviews, scope, and reading order. |
| 01-messaging-fundamentals.md | The broker-agnostic core: queues vs pub/sub vs streams, delivery guarantees, ordering, idempotent consumers, DLQs, backpressure, poison messages, and a RabbitMQ vs SQS/SNS vs Kafka vs NATS comparison. |
| 02-kafka-deep-dive.md | Kafka specifically: brokers, partitions, replication, KRaft, producer configuration, consumer groups and rebalancing, offsets, exactly-once semantics, compaction, retention, schema registry, and the classic production pitfalls. |
| 03-event-driven-patterns.md | Architecture patterns built on top of messaging: event notification vs event-carried state transfer vs event sourcing, CQRS, the outbox pattern, sagas, CDC with Debezium, stream processing, and event schema design/versioning. |
| 04-interview-questions.md | Graded Q&A (junior → senior → staff) with model answers, including diagnostic scenarios and a full design exercise. |

## Recommended reading order

1. `01-messaging-fundamentals.md` — read this first regardless of which broker your target company uses. Delivery guarantees, idempotency, and ordering are the concepts interviewers probe hardest, and they are broker-independent. Every later file assumes them.
2. `02-kafka-deep-dive.md` — Kafka is the de facto interview default for event streaming; even SQS-heavy shops ask Kafka questions. If you have limited time, prioritize consumer groups, offset management, and the exactly-once section.
3. `03-event-driven-patterns.md` — this is where messaging meets system design. The outbox pattern and sagas appear in nearly every microservices design question; event sourcing and CQRS are common staff-level probes.
4. `04-interview-questions.md` — self-test after the first three. Answer out loud before reading the model answer; the framing and the trade-offs you volunteer matter more than the raw facts.

## How this shows up in interviews

Three distinct question shapes, each testing something different:

- **Conceptual probes.** "What's the difference between at-least-once and exactly-once delivery?" "Why does Kafka only guarantee ordering per partition?" These test whether you understand the mechanics or just the marketing terms. Covered in files 01 and 02.
- **Diagnostic scenarios.** "Consumer lag is growing — walk me through your diagnosis." "The same order was charged twice — how?" These test production experience. You want a structured, hypothesis-driven answer, not a list of guesses. Covered in files 02 and 04.
- **Design questions with a messaging core.** "Design order processing," "design a notification system," "migrate this monolith to events." The messaging layer is where these designs succeed or fail: what goes on the bus, what guarantees each hop needs, and how money-touching steps stay exactly-once. Covered in files 03 and 04, building on the system design section (`06-system-design/`).

A note on stack framing, consistent with the rest of this guide: examples map onto a working backend stack (AWS, Go/PHP services, MySQL as the source of truth) but the patterns — idempotent consumers, the outbox, consumer group rebalancing — are identical in any ecosystem. If your interview is with an SQS shop, a RabbitMQ shop, or a NATS shop, the fundamentals file translates directly; only the Kafka file is vendor-specific, and even there the concepts (partitioned logs, consumer offsets) reappear in Kinesis, Pulsar, and Redpanda under different names.
