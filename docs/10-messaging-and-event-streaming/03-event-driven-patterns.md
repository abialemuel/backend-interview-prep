# Event-Driven Patterns

This is where messaging meets system design. The broker gives you a durable, ordered pipe; these patterns are what you build with it — and they are the substance of most senior/staff design interviews. The recurring themes: **what exactly goes in an event**, **how two systems stay consistent without distributed transactions**, and **how a business process spanning services completes or unwinds cleanly**.

## Three kinds of "event-driven" — know which one you mean

Martin Fowler's observation holds: "event-driven" describes at least three patterns with very different costs, and conflating them is a junior tell.

### Event notification

The event says *that* something happened, not much more: `{"event": "order_placed", "order_id": 42}`. Interested consumers call back to the source service's API for details. Pros: tiny events, no schema coupling on the full entity, the source stays authoritative. Cons: the callback re-couples the systems — the source must be up and must handle the read load, and by the time the consumer calls, the state may have changed (you get *current* state, not state *as of the event*). Good default when consumers are few and freshness of the callback read is acceptable.

### Event-carried state transfer

The event carries the data consumers need: `{"event": "order_placed", "order_id": 42, "customer_id": 7, "items": [...], "total": 129.90}`. Consumers maintain their own local copy and never call back. Pros: consumers are fully decoupled and available even when the source is down; no read amplification on the source; consumers see state *as of the event*. Cons: fat events, data duplicated across services, and **eventual consistency becomes explicit** — every consumer's copy lags slightly. This is the workhorse pattern for service autonomy (e.g., the shipping service keeps its own slice of customer addresses fed by `customer_updated` events).

### Event sourcing

The events **are** the source of truth. Instead of storing current state and emitting events as a side effect, you store the append-only sequence of events (`OrderPlaced`, `ItemAdded`, `PaymentCaptured`) and derive current state by replaying them. Current state is a cache; the log is the truth.

What you gain: a perfect audit trail (regulators love it; so do debugging sessions — "how did this account reach this balance?"), temporal queries (state as of any past moment), and the ability to build brand-new read models from history. What you pay, and must say out loud: **replay cost** (mitigated by periodic snapshots), **schema evolution forever** (you must be able to read events written five years ago — upcasters, versioned event types), no ad-hoc queries against the event store itself (you need projections), and a real learning curve. The senior qualifier: event sourcing is a *per-aggregate* decision, not an architecture-wide one — source the ledger and the order lifecycle; keep the product catalog as plain CRUD.

### The three at a glance

| | Event notification | Event-carried state transfer | Event sourcing |
| --- | --- | --- | --- |
| Event contains | ID + type, little else | The data consumers need | The full fact; events *are* the store |
| Consumer gets details by | Calling back to the source | Its own local copy | Replaying/projecting the log |
| Coupling | Runtime (callback) | Schema (payload) | Deepest — events are the model |
| Source of truth | Source service's DB | Source service's DB | The event log itself |
| Cost center | Read load + availability of source | Duplication + staleness of copies | Evolution, replay, projections |
| Reach for it when | Few consumers, fresh reads OK | Consumer autonomy, source offloading | Audit/temporality is a requirement |

## CQRS

Command Query Responsibility Segregation: separate the write model (commands, invariants, normalized) from the read model (queries, denormalized projections), often as separate stores updated asynchronously from events. It pairs naturally with event sourcing (events feed the projections) but neither requires the other — "MySQL for writes, Elasticsearch fed by CDC for search" is CQRS.

Use it when read and write shapes genuinely diverge (an order is written as an aggregate but read as a search result, a dashboard aggregate, and a customer history list) or when read/write scaling needs differ by orders of magnitude. The cost: the read model is **eventually consistent**, and the UX must tolerate read-your-own-write gaps (or you patch them: read-your-writes via the write model, client-side echo, or version-pinned reads). Anti-pattern to name: CQRS on a plain CRUD service — you take on dual models and projection lag for no benefit.

## The outbox pattern

The most load-bearing pattern in this file, and a near-guaranteed interview topic. The problem: a service must **update its database and publish an event**, atomically. The naive orderings both fail:

```
write DB, then publish  --> crash between them: DB updated, event lost
publish, then write DB  --> crash between them: event out, DB never updated
2PC across DB + broker  --> brokers don't support it; don't propose it
```

The fix: make the event part of the database transaction. Write the business change *and* the event — into an `outbox` table — in **one local transaction**. A separate relay then delivers outbox rows to the broker.

```sql
BEGIN;
UPDATE orders SET status = 'placed' WHERE id = 42;
INSERT INTO outbox (id, aggregate_id, type, payload, created_at)
VALUES (UUID(), 42, 'order_placed', JSON_OBJECT(...), NOW(6));
COMMIT;   -- state change and event now succeed or fail together
```

Two relay styles: a **poller** (query unpublished rows, publish, mark done — simple, adds polling latency and DB load) or **CDC/log tailing** (Debezium reads the outbox inserts from the DB's replication log — lower latency, no polling, more infrastructure; see CDC below). Either way the guarantee is **at-least-once**: the relay can crash after publishing but before marking done, so duplicates happen and consumers must be idempotent — which they had to be anyway (file 01). Events are also published in commit order per aggregate; key the topic by `aggregate_id` to preserve it. The inverse problem (consume a message and write the DB atomically) has the mirror solution: an **inbox** table — insert the message ID and the business write in one transaction, which is exactly the idempotent-consumer recipe.

## Sagas: transactions across services

A saga replaces the impossible distributed transaction with a sequence of **local transactions**, each publishing an event/command that triggers the next, plus **compensating actions** that semantically undo completed steps when a later one fails. Order flow: reserve inventory → charge payment → create shipment; if shipment creation fails, refund the payment and release the inventory. Compensations are business actions (refund), not rollbacks (the charge *happened* — it is in the ledger), and some steps are **pivot points** that cannot be compensated (goods shipped), so ordering matters: put risky, compensatable steps before irreversible ones.

### Choreography vs orchestration

**Choreography**: no central coordinator; each service reacts to events and emits its own. Order service emits `order_placed`; inventory hears it, reserves, emits `inventory_reserved`; payment hears that, charges, emits `payment_captured`; and so on. Pros: maximal decoupling, no single point of coordination, adding a step can be just a new subscriber. Cons: the workflow exists nowhere — it is implicit in N services' subscriptions, so answering "where is order 42 stuck?" means archaeology; failure/compensation paths multiply into an event spaghetti as steps grow; cyclic dependencies creep in.

**Orchestration**: a dedicated orchestrator (the order-saga service, or a workflow engine — Temporal, AWS Step Functions) holds the state machine, commands each participant, and drives compensation on failure. Pros: the flow is explicit, observable, and testable in one place; timeouts and retries are first-class; "where is order 42?" is one query. Cons: the orchestrator couples to every participant, can accrete business logic that belongs in services, and is itself a component to scale and keep available.

### A worked example: order placement saga (orchestrated)

```
                      +--------------------+
   order_placed ----> |  Order Saga        | --cmd--> Inventory: reserve(42)
                      |  (state machine    | <--evt-- inventory_reserved
                      |   per order_id)    | --cmd--> Payment: charge(42)
                      |                    | <--evt-- payment_failed   (!)
                      |                    | --cmd--> Inventory: release(42)
                      +--------------------+ --evt--> order_cancelled
```

| Step | Local transaction | Compensation | Notes |
| --- | --- | --- | --- |
| 1. Reserve inventory | `reservations` insert | Release reservation | Compensatable; goes first |
| 2. Charge payment | Gateway call + ledger entry | Refund | Compensatable but visible to the customer |
| 3. Create shipment | Shipment record + WMS call | — (pivot) | Irreversible; sequenced last |

Every command and event carries the saga/correlation ID; every participant is an idempotent consumer (the orchestrator *will* re-send commands after its own recovery); and each saga step has a timeout with a defined outcome ("no `inventory_reserved` within 30s → treat as failed, compensate"). Timeouts are the part candidates forget: without them, a lost message parks the order in limbo forever.

The practiced heuristic: **choreography for short flows (2-3 steps) with simple failure handling; orchestration once the flow has branching, timeouts, or more than ~3 steps** — the mainstream position having shifted toward orchestration-by-default for anything money-shaped, precisely because workflow engines made explicit state cheap. Either way, sagas give you eventual consistency with visible intermediate states; the design must name those states (`payment_pending`) and what users see during them.

## Living with eventual consistency

Every pattern in this file trades immediate consistency for decoupling, and the design is not done until you have said what happens *in the gap*. Interviewers increasingly probe this, because it is where event-driven systems meet users and auditors.

- **Name the intermediate states.** A saga in flight is not "inconsistent" — it is in a defined state (`payment_pending`, `awaiting_stock`). Model those states explicitly, expose them in APIs, and design the UI for them ("Your order is being confirmed") rather than pretending the flow is instantaneous.
- **Read-your-own-writes.** A user who just updated their profile and sees the old value via a lagging projection perceives a bug. Options, cheapest first: echo the write back client-side (optimistic UI); route the writing user's reads to the write model for a short window; version-pinned reads (the write returns a version, reads wait until the projection reaches it).
- **Bound the staleness.** "Eventually" is not an SLO. Put a number on projection/consumer lag per read model (search can be minutes behind; the balance shown before a withdrawal cannot) and alert on it — this is the consumer-lag discipline from the Kafka file wearing product clothes.
- **Reconcile.** Assume some invariant will eventually be violated — a lost compensation, a bug that dropped events, a replay done wrong. Periodic reconciliation jobs that diff independent sources of truth (gateway records vs order states; sum of ledger events vs cached balance) are how mature event-driven systems *detect* what the architecture could not prevent. Design detection, not just prevention.

## Change data capture (CDC) and Debezium

CDC turns the database's replication log (MySQL binlog, Postgres WAL) into an event stream: every committed insert/update/delete becomes a message, in commit order, with no application code changes and no chance of "forgot to publish." **Debezium** is the standard open-source implementation — connectors (typically run on Kafka Connect) that tail the log and emit change events per table, with an initial snapshot phase for existing data, typically into compacted, key-by-primary-key topics.

Where it shines: feeding search indexes and caches, replicating data to warehouses in near-real-time, strangler-fig migrations off legacy systems (the legacy DB's changes stream out from under it), and as the relay for the outbox pattern (Debezium has first-class outbox routing support). The caveat that separates senior answers: **raw CDC events are the database's schema, not a domain contract.** `orders` row-changed events couple every consumer to your table layout, and one `ALTER TABLE` becomes a breaking change for the company. The remedies: the outbox pattern (you control the event payload explicitly) or a transformation layer that maps table changes into proper domain events. Also know: CDC gives at-least-once (idempotent consumers again), ordering per key, and connector lag is one more thing to monitor.

## Stream processing basics

Consuming events one at a time covers most needs; **stream processing** is for continuous computation *over* streams — aggregations, joins, windowing. The concepts to hold (frameworks change, these don't):

- **Stateless vs stateful.** Filtering/mapping is trivially scalable. Counting orders per customer, joining clicks to impressions, sessionization — these need **state**, and managed state (local stores + changelogs/checkpoints for recovery) is the entire hard part of the field.
- **Windows.** Tumbling (fixed, non-overlapping), hopping/sliding (overlapping), session (gap-based). Any "per minute / per hour / per session" requirement is a windowing requirement.
- **Event time vs processing time, and watermarks.** Events arrive late and out of order; correct aggregations use the event's timestamp, with watermarks/grace periods deciding how long to wait for stragglers before finalizing a window. "What do you do with an event that arrives after its window closed?" is a classic probe (answer: allowed lateness + updating results, or route to a late-data path).
- **The stream-table duality.** A changelog stream compacts into a table; a table's changes are a stream. This is why Kafka Streams has KStream/KTable and why compacted topics (file 02) matter.

Tool selection in one breath: **Kafka Streams** is a Java library — no separate cluster, exactly-once v2, perfect for Kafka-to-Kafka transformations owned by one service; **Apache Flink** is a full engine — larger state, richer windowing/CEP, multiple sources/sinks, SQL, at the cost of operating a cluster (or paying for managed). Simple aggregation on a Kafka-centric JVM stack → Streams; heavy, multi-source, or SQL-driven pipelines → Flink. If neither fits the team, a plain consumer with a state store is not wrong — it is just hand-rolled stream processing, and saying so shows you see the spectrum.

## Designing event schemas and versioning

Events are **public API published into the past** — consumers you do not know about will read them months later, including from replays. Design accordingly.

**Anatomy of a good event:**

```json
{
  "event_id": "9c1f...-...",            // UUID — the idempotency key
  "event_type": "order.placed",
  "event_version": 2,
  "occurred_at": "2026-07-26T10:15:00.000Z",
  "aggregate_id": "order-42",           // partition key; ordering scope
  "correlation_id": "req-7f3a...",      // traces the whole flow
  "causation_id": "evt-...",            // the event that caused this one
  "payload": { "order_id": 42, "customer_id": 7, "total": 129.90, "currency": "AED" }
}
```

Ground rules:

- **Past tense, business language.** `order.placed`, not `create_order` (that's a command) and not `orders_table_updated` (that's CDC leakage). Events state facts.
- **Right-size the payload** on the notification ↔ state-transfer axis deliberately: enough that mainstream consumers don't need a callback; not your entire internal model. Never include data you'd have to redact from history later (PII in immutable logs is a compliance trap — consider referencing PII by ID, or crypto-shredding: encrypt per-user, delete the key to "erase").
- **Additive evolution by default.** Adding optional fields with defaults is backward-compatible; renaming, removing, or changing types/semantics is breaking. Enforce this mechanically with a schema registry compatibility mode (file 02), not by code review vigilance.
- **Breaking changes get a new version**, and you have two deployment shapes: publish both versions during a migration window (double-publish, deprecate v1 once consumers confirm migration), or publish v2 only and run **upcasters** — transformers that lift old events to the current shape at read time. Upcasting is mandatory in event-sourced systems, where v1 events live in the log forever.
- **Consumers: tolerant reader.** Ignore unknown fields, don't fail on additive change, validate only what you use.

## Anti-patterns to name-check

Recognizing these on sight is cheap senior signal, because each one is an event-driven system that quietly reintroduced the coupling it was built to remove:

- **Commands in event clothing.** `email_service.please_send_welcome_email` published as an "event" is a command with extra steps — the producer knows exactly who must act and what they must do. Events state facts (`user.registered`); if only one specific service can meaningfully react and the producer depends on it happening, be honest and send a command (or make a call). The distinction matters because events promise the producer doesn't care who listens — violating that promise silently makes every "event" a hidden synchronous dependency.
- **The distributed monolith.** Services that exchange events but cannot function, deploy, or be understood independently — every change fans out across three repos and a schema. Usually caused by anemic notification events forcing constant callbacks, or by slicing services along technical layers instead of business capabilities.
- **Event chains as workflow.** A 9-step business process implemented as 9 choreographed hops works until step 6 fails at 2 a.m. If a process has a name in the business ("order fulfillment"), it deserves a named home in the architecture — that is the saga-orchestration argument in one line.
- **The shared event bus as integration database.** Every team publishing everything onto one bus with no ownership, no registry, no compatibility rules — the enterprise service bus failure mode reborn on Kafka. Events need owners and contracts like any API (see below).

!!! note "The one-sentence summary an interviewer remembers"
    Local transaction + outbox to get events out reliably; at-least-once everywhere with idempotent consumers keyed by event ID; sagas (orchestrated, once non-trivial) for cross-service flows; CDC for data movement but domain events for contracts; and schema compatibility enforced by machinery, not discipline.
