# Messaging & Event Streaming Interview Questions

Model answers for backend interviews, graded junior → senior → staff. Use
these as a self-test, not memorization material — answer out loud before
reading, and notice that the model answers volunteer trade-offs and failure
modes without being asked. That habit *is* the seniority signal.

---

## Junior

### Q1: What is the difference between a message queue and pub/sub?

**Answer:** In a queue, each message is consumed by exactly one consumer —
competing workers pull from the queue and the message is deleted once
acknowledged. It is a work-distribution mechanism: background jobs, task
processing. In pub/sub, each message is delivered to *every* subscriber of the
topic — it is a fan-out mechanism where the producer doesn't know who is
listening, so new consumers can be added without changing the producer.
Classic pub/sub (SNS, Redis Pub/Sub) is also fire-and-forget: an offline
subscriber misses the message. That is why durable fan-out in practice is
either SNS delivering into per-consumer SQS queues, or a log-based system like
Kafka, where consumer groups give you both semantics — queue behavior within a
group, pub/sub behavior across groups.

### Q2: Why put a queue between two services at all? What does it cost?

**Answer:** Three gains. Decoupling: the producer doesn't need the consumer to
be up, deployed, or fast right now. Load leveling: a burst of writes becomes a
backlog that drains at the consumer's sustainable rate instead of a spike that
takes the downstream out. Fan-out and extensibility: new consumers attach
without producer changes. The costs, which should be named in the same breath:
eventual consistency (the work hasn't happened when the producer returns — the
caller sees "accepted," not "done"); harder failure semantics (duplicates,
redelivery, ordering); harder debugging (a request is now a trail across async
hops, so you need correlation IDs); and operating the broker itself. A queue
is the right default for anything that doesn't need to happen within the
request — and the wrong tool when the caller needs the result synchronously.

### Q3: Explain at-most-once, at-least-once, and exactly-once delivery.

**Answer:** They describe behavior across failures. At-most-once: no retries,
or ack-before-processing — messages can be lost but never duplicated;
acceptable for metrics and telemetry. At-least-once: producer retries until
acked, consumer acks only after processing — nothing is lost, but duplicates
are inevitable (a lost ack causes a resend; a crash after processing but
before ack causes redelivery). Exactly-once: strictly, exactly-once *delivery*
is impossible over an unreliable network; what real systems provide is
exactly-once *processing* — effects applied once — via broker mechanisms
(idempotent producers, transactions) or application-level idempotency. The
practical takeaway: at-least-once plus an idempotent consumer is the standard
recipe; true exactly-once machinery is reserved for cases where duplicates are
expensive, like billing.

### Q4: What is a dead-letter queue and why do you need one?

**Answer:** A DLQ is where the broker moves a message after it fails
processing N times, instead of redelivering it forever. Without one, a poison
message — one that fails deterministically, e.g., a malformed payload — gets
redelivered in an infinite loop, burning consumer capacity, and on an ordered
queue it blocks every message behind it. The DLQ unblocks the pipeline while
preserving the message for inspection. The parts juniors forget: alert on DLQ
depth (an unwatched DLQ is silent data loss), attach diagnostics (error,
attempt count, source), and have a redrive path to replay messages after the
bug is fixed — remembering redriven messages are duplicates, so consumers must
be idempotent. Also, don't retry deterministic failures: a validation error
should go to the DLQ on attempt one; retries with backoff are for transient
errors like timeouts.

### Q5: What is a Kafka consumer group?

**Answer:** A set of consumers sharing a `group.id` that split a topic's
partitions among themselves — each partition is owned by exactly one consumer
in the group, which is how Kafka parallelizes consumption while preserving
per-partition ordering. Add consumers and partitions are rebalanced across
them, up to a hard ceiling: with 4 partitions, a 5th consumer idles. The group
tracks its progress as committed offsets per partition (stored in
`__consumer_offsets`), so it resumes where it left off after restarts.
Different groups are completely independent, each with its own offsets — the
same `orders` topic can feed the fulfillment group and the analytics group at
different speeds. Within a group you get queue semantics; across groups,
pub/sub.

---

## Senior

### Q6: How do you make a consumer idempotent, concretely?

**Answer:** In order of preference. First, make the operation naturally
idempotent: absolute writes (`SET status='shipped'`), upserts keyed by
business identity — applying them twice is harmless. Second, dedup by
idempotency key: every message carries a unique ID; the consumer inserts it
into a `processed_messages` table with a unique constraint **in the same
database transaction as the business write** — that atomicity is the crux,
because a dedup check that commits separately from the work recreates the
race. On a duplicate, the insert violates the constraint, you roll back and
ack. Third, state/version guards: apply the message only if the entity is in
the expected prior state (`UPDATE ... WHERE status='pending'`), which also
defends against out-of-order delivery, which pure ID dedup does not. Two
production details: the dedup table needs TTL cleanup scoped to the broker's
redelivery horizon, and if processing calls an external API, propagate the
idempotency key downstream (e.g., Stripe's `Idempotency-Key`) so the whole
chain is idempotent, not just your hop.

### Q7: Kafka only guarantees ordering per partition. How do you design around that?

**Answer:** By recognizing you almost never need global ordering — you need
per-entity ordering. Key messages by the entity whose event sequence matters
(`order_id`, `account_id`); all of that entity's events hash to one partition
and are totally ordered relative to each other, while different entities
interleave freely, which is fine. Then protect that guarantee from yourself:
ordering survives only if you don't break it consumer-side — processing a
batch concurrently across keys is fine, concurrently within a key is not;
DLQ-ing message 5 and continuing to message 6 for the same entity may violate
invariants. For consumers that can still receive logically stale events
(redelivery, replays), add version or state guards so an older event can't
overwrite a newer state. And know the operational trap: changing partition
count changes key→partition mapping, so in-order-ness across the resize
boundary is lost — plan partition counts up front rather than resizing a keyed
topic under load.

### Q8: Walk me through `acks`, `min.insync.replicas`, and replication. What's the standard durable config?

**Answer:** Each partition has a leader and followers; the ISR is the set of
replicas fully caught up with the leader. `acks` controls what the producer
waits for: `0` nothing (fire-and-forget), `1` the leader's write, `all` every
ISR member. `acks=all` alone is a trap: if the ISR has shrunk to just the
leader, `all` means one machine. `min.insync.replicas=2` closes it — writes
fail if fewer than two replicas are in sync. The standard durable config is
replication factor 3, `min.insync.replicas=2`, `acks=all`: an acknowledged
write survives any single broker loss, and the cluster still accepts writes
with one broker down. Two adjacent facts worth volunteering: consumers only
read up to the high watermark (fully-ISR-replicated offsets), so a leader
failover can't un-deliver data; and `unclean.leader.election.enable=false`
means a partition with no live ISR member goes offline rather than electing a
stale replica — choosing consistency over availability, and you should know
which your workload wants.

### Q9: Consumer lag on your main Kafka topic is growing steadily. Diagnose it.

**Answer:** Structured, hypothesis-driven, in this order. First, frame the
metric: lag = latest produced offset − committed offset, per partition; what
matters is growth rate and time-to-drain against the consumer's staleness SLO.
(1) Produce-side or consume-side? Compare produce rate vs consume rate over
time — a marketing-campaign traffic spike with healthy consumers is a capacity
question, not a bug. (2) All partitions or some? Uniform lag points to
something systemic — slow downstream dependency, undersized group. Lag
concentrated in one partition points to a hot key or one sick consumer
instance. (3) Is the group thrashing? Check rebalance frequency — a consumer
exceeding `max.poll.interval.ms` gets evicted, triggering a rebalance, pausing
consumption, growing lag, and slowing processing further: the rebalance storm.
Fix by reducing `max.poll.records` or making per-message work faster/async.
(4) Profile the consumer — the bottleneck is usually not Kafka but the
synchronous DB write or external API call per message; batch downstream
writes, parallelize across keys, cache lookups. (5) Only then, structure: if
consumers = partitions and per-message cost is optimized, you need more
partitions (with the key-remapping caveat) or fundamentally cheaper
processing. Interim mitigations while fixing: scale consumers up to partition
count, and if the data has a freshness horizon, consider whether skipping
ahead is acceptable — usually it is not, but asking shows judgment.

### Q10: What is the outbox pattern and what problem does it solve?

**Answer:** The dual-write problem: a service must update its database and
publish an event, and no ordering of the two naive writes is safe —
DB-then-publish loses the event on a crash between them; publish-then-DB emits
an event for a change that never happened. There is no 2PC with a message
broker, so the outbox makes the event part of the DB transaction: write the
business change and an event row into an `outbox` table in one local
transaction, then a relay delivers outbox rows to the broker — either a poller
or, better, CDC (Debezium tails the binlog and has native outbox routing). The
guarantee is at-least-once: the relay can crash after publishing but before
marking a row done, so duplicates occur and consumers must be idempotent —
which at-least-once transport required anyway. Events also leave in commit
order per aggregate; keep it by using the aggregate ID as the partition key.
The mirror pattern on the consume side is the inbox: message ID insert +
business write in one transaction, i.e., the idempotent consumer.

### Q11: Compare saga choreography and orchestration. When do you pick each?

**Answer:** A saga is a cross-service business process as a chain of local
transactions with compensating actions for rollback — refund the charge,
release the reservation — because you cannot hold a distributed transaction
across services. Choreography: no coordinator; each service reacts to events
and emits its own. Maximum decoupling and no central component, but the
workflow exists nowhere — it is implicit in N subscription tables, so "where
is order 42 stuck?" is archaeology, and compensation paths turn into event
spaghetti as steps grow. Orchestration: a coordinator (a saga service, or a
workflow engine like Temporal or Step Functions) owns the state machine,
commands participants, and drives compensations; the flow is explicit,
observable, and testable, at the cost of a central component that couples to
every participant and can accrete logic that belongs in services. Heuristic:
choreography for short flows — two or three steps with trivial failure
handling; orchestration once there are branches, timeouts, human steps, or
money — which in practice means orchestration for most order/payment flows.
Two saga fundamentals to volunteer: compensations are business actions, not
rollbacks (the charge happened; you refund it), and irreversible steps (goods
shipped) must be sequenced after compensatable ones.

### Q12: Producer-side: what does Kafka's idempotent producer actually do, and what doesn't it do?

**Answer:** It fixes retry duplicates at the partition level. Each producer
session gets a producer ID; each batch carries a per-partition sequence
number; the broker tracks the last sequences per producer and rejects
duplicates and out-of-order gaps. Result: a producer retry after a lost ack
cannot create a duplicate or reorder writes — exactly-once *append into a
single partition, for a single producer session*, and since it's default-on
(Kafka 3.0+) with negligible cost, there's no reason to disable it. What it
does not do: nothing across partitions or across producer restarts (a new
session gets a new PID — that needs transactions with a `transactional.id`,
which also fences zombie instances via epochs); nothing about consumer-side
duplicates (redelivery after a crash before commit is unaffected); and nothing
beyond Kafka (a consumer writing to MySQL or calling a payment API still needs
its own idempotency). It is one layer of the exactly-once stack, not the
stack.

### Q13: When would you choose SQS over Kafka, and vice versa? Be concrete.

**Answer:** SQS when the job is work distribution and the stack is AWS:
background jobs, email sends, image processing. It is fully managed — zero
brokers to run — scales without partition math, has native DLQ and redrive,
per-message delays, and pay-per-use pricing. Its limits define the boundary:
no replay (consumed is gone), no fan-out on its own (pair with SNS or
EventBridge), no ordering in Standard (FIFO adds per-group ordering with
per-group throughput caps), and 14-day max retention. Kafka when the messages
are an event stream, not a task list: multiple independent consumers reading
at their own pace, replay for new services and backfills, high sustained
volume, per-key ordering, log compaction for latest-state topics, and the
stream-processing/CDC ecosystem. Its costs: partition/consumer-group
operational complexity, and cluster ops if self-hosted (managed MSK/Confluent
shifts that to money). The senior answer refuses the false choice: most real
architectures run both — SQS for jobs, Kafka as the event backbone — and picks
per workload, not per fashion. If the honest requirement is "a queue for
jobs," Kafka is overkill; if it's "the history of what happened, consumable by
teams I haven't met," only a log does that.

### Q14: Walk through SQS visibility timeout and dead-letter queues. What breaks if the timeout is misconfigured?

**Answer:** SQS doesn't delete a message on delivery — it hides it from other
consumers for the visibility timeout (default 30s) and expects the consumer
to call `DeleteMessage` after successful processing. If the timeout expires
first, the message reappears and another consumer can pick it up. Two
opposite failure modes come from getting this wrong: a timeout shorter than
actual processing time causes the queue itself to create duplicates — the
message becomes visible again and gets processed a second time while the
first attempt is still legitimately running, no network failure involved,
just bad configuration; a timeout that's too long means a genuinely failed
message sits invisible and unretried for the whole window, inflating
recovery latency. The fix for variable-length work is to heartbeat —
extend the timeout mid-processing with `ChangeMessageVisibility` — rather
than just setting a large fixed value. The dead-letter queue is the backstop
for messages that keep failing: a `RedrivePolicy` with `maxReceiveCount`
moves a message to the DLQ after that many failed receive attempts instead of
redelivering it forever, which is the managed version of poison-message
handling. The part people forget: the DLQ is a normal queue that nothing
drains automatically, so without alerting on its depth, messages die there
silently until a customer notices. `StartMessageMoveTask` lets you redrive
DLQ messages back to the source queue once the underlying bug is fixed,
without hand-rolling that plumbing.

### Q15: A single domain event needs to reach three independent consumer teams on AWS, and one of them only cares about high-value orders. How do you design the fan-out?

**Answer:** SQS alone can't do this — one message is deleted once one
consumer acks it, so a lone queue can't serve three independent teams. The
standard shape is publish-once, fan-out-many: the producer publishes to an
SNS topic, and each team subscribes its own SQS queue to that topic. Each
subscriber now has a durable, independently-consumed, independently-scaled
copy of every message, with its own DLQ if it falls behind or fails — this is
SNS+SQS's answer to what Kafka's consumer groups get for free from a
partitioned log. For the team that only wants high-value orders, two options
depending on how the filter needs to work: an SNS filter policy on message
*attributes* if "high-value" is a flag the producer already sets at publish
time (simplest — SNS just won't deliver non-matching messages to that
subscription); or EventBridge if the filter needs to inspect the event's
actual content (nested fields, numeric thresholds like `amount > 10000`,
`anything-but` exclusions) rather than a pre-set attribute — EventBridge
rules match on the payload itself, which is the more common shape for
"route by business meaning" rather than "route by a flag someone remembered
to set." The trap to name unprompted: reaching for EventBridge everywhere
"to be flexible" adds a moving part that isn't free when a plain SNS filter
policy already covers the actual requirement.

---

## Staff

### Q16: Design order processing where the customer is charged exactly once, end to end.

**Answer:** Start by naming the truth: the payment gateway is an external
system, so no broker transaction can span it — "exactly-once" must be
engineered as at-least-once delivery + idempotency at every hop, with the
gateway's idempotency key as the keystone. Flow: (1) API receives the order;
in one MySQL transaction, insert the order in state `pending_payment` and an
outbox event — this kills the dual-write problem at the source. (2) CDC/poller
relays the event to a Kafka topic keyed by `order_id`, so each order's events
are ordered. (3) The payment consumer is inbox-idempotent (event ID + state
guard `WHERE status='pending_payment'`, committed atomically with its own
state change) and calls the gateway with a **deterministic idempotency key** —
`order_id` plus payment attempt number — so a retry after a timeout returns
the original charge instead of a second one. The subtle case to raise
unprompted: a gateway *timeout* is ambiguous — the charge may or may not have
happened — so the consumer must not fail-and-retry blindly, and must not mark
failure; it re-calls with the same idempotency key or queries the gateway, and
parks the order in `payment_unknown` for reconciliation if ambiguity persists.
(4) Outcome events (`payment_captured` / `payment_failed`) go out via the same
outbox discipline; downstream (fulfillment, notifications) are idempotent
consumers. (5) Failure handling: retries with backoff for transient gateway
errors, DLQ + alert for poison orders, and a reconciliation job diffing
gateway records against order states as the financial backstop — because at
staff level you assume some invariant will eventually be violated and design
the detection, not just the prevention. Kafka EOS transactions are
deliberately *not* the centerpiece: they cover Kafka-to-Kafka hops only; here
the correctness lives in the outbox, the inbox, and the gateway idempotency
key.

### Q17: Your CDC pipeline streams table changes to other teams, and a schema migration just broke three downstream consumers. What went wrong architecturally, and how do you fix it?

**Answer:** The architecture leaked a private interface: raw CDC events are
the *table's* shape, so every consumer was silently coupled to the internal
schema, and an `ALTER TABLE` became a company-wide breaking change. The
database schema was never a contract; it was treated as one. Fixes, in layers.
Short term: restore compatibility (re-add/alias the old columns or transform
in-flight) and get the consumers unblocked. Structurally: put a contract
boundary between the table and the consumers — either move producers to the
outbox pattern, where the service explicitly composes domain events
(`order.placed`, versioned, past-tense, business-shaped) rather than exposing
row diffs, or keep CDC but add a transformation layer that maps table changes
into stable public events, so internal refactors stay internal. Then make
compatibility mechanical, not cultural: a schema registry with `BACKWARD` or
`FULL` compatibility enforced at publish time, so a breaking change is
rejected in CI, not discovered in production by another team. Finally, the
organizational part that makes this a staff question: published events need
owners, versioning policy, deprecation windows, and consumer-driven contract
tests — the same governance as a public REST API, because that is what they
are. The tell of a mature answer is stating the general principle: CDC is
excellent plumbing for data movement, but *contracts* should be domain events
you design, not schemas you happen to have.

### Q18: When is event sourcing worth it, and what goes wrong in practice?

**Answer:** Worth it when the history *is* the business value: financial
ledgers and anything auditable ("prove how this balance arose"), domains with
temporal queries (state as of a date), heavy compliance, or systems that
benefit from rebuilding new read models from history. It composes with CQRS:
the event log is the write model; projections serve reads. What goes wrong, in
rough order of frequency: (1) applying it everywhere — it's a per-aggregate
decision; sourcing the CRUD-shaped catalog buys complexity for nothing; (2)
underestimating eternal schema evolution — v1 events live in the log forever,
so you commit to upcasters/versioning for the system's lifetime; (3) replay
cost without snapshots — an aggregate with a million events cannot be folded
on every load; snapshot periodically and replay the tail; (4) PII in an
immutable log colliding with deletion rights — mitigate by referencing PII by
ID or crypto-shredding (per-user keys; delete the key to erase); (5) treating
the event store as queryable — ad-hoc queries need projections, and
projections are eventually consistent, so the UX must handle read-your-write
gaps; (6) confusing it with "we have Kafka" — a topic with retention is not an
event store; you need per-aggregate streams, optimistic concurrency on append,
and permanent retention. My honest default: plain state + outbox events covers
90% of systems; event-source the specific aggregates where audit or
temporality is a first-class requirement, and be able to defend that line.

### Q19: The same Kafka event was processed twice, three weeks after the code shipped. Walk me through where duplicates come from and how you'd close the gap.

**Answer:** First, the priors: at-least-once systems *will* duplicate; the bug
is almost never "Kafka broke" but "we assumed it wouldn't." Sources, producer
to consumer: producer retry after a lost ack (closed by the idempotent
producer — but only per session and per partition; a producer restart or an
application-level retry that calls `send()` again with a new record is
invisible to it); double-publish from a broken outbox relay (crashed between
publish and mark-done — by design at-least-once); consumer redelivery
(processed, crashed before offset commit; or a rebalance revoked the partition
after processing but before commit — the classic three-weeks-later trigger,
since it needs an ill-timed rebalance); offset regression (offsets expired for
an idle group with `auto.offset.reset=earliest`, or a manual reset); and
replays/redrives, which are duplicates on purpose. Diagnosis: compare the two
processings' partition/offset — same offset twice means consumer-side
redelivery or offset regression (check rebalance and reset history); different
offsets with the same event ID means producer/relay-side duplication. The
durable fix is making duplicates harmless rather than impossible: idempotency
keyed on event ID with the dedup record committed atomically with the business
write, plus state guards for out-of-order redelivery. And the process point
that makes it staff-level: this should have been caught by design review —
"what happens if this handler runs twice?" is a question every async consumer
must answer before shipping, and a duplicate-injection test (replay staging
traffic) is cheap insurance.

### Q20: You're migrating a monolith to event-driven microservices. What's your messaging strategy, and what failure modes do you guard against from day one?

**Answer:** Strategy: strangler fig, events first. (1) Stand up the backbone
(managed Kafka unless there's a reason not to) and start publishing domain
events *from the monolith* via the outbox — the monolith keeps working, but
its facts become available; CDC on the monolith's DB can bootstrap this before
code changes are possible, with the explicit caveat that raw CDC shapes are
temporary scaffolding, not contracts. (2) First consumers are read-only —
analytics, search indexing, notifications — which build event muscle with no
correctness risk to the core. (3) Extract services along aggregate boundaries;
each new service owns its data, subscribes to event-carried state transfer for
what it needs from others, and publishes its own events via its own outbox.
(4) Cross-service workflows become sagas — orchestrated for anything
money-shaped. Day-one guardrails, because these are cheaper to install than
retrofit: schema registry with enforced compatibility before the second
consumer exists; event ID + idempotent-consumer convention as a paved road
(shared library), not a suggestion; correlation IDs and distributed tracing
across every async hop, or debugging becomes archaeology; DLQs with alerts and
a standard redrive runbook; lag monitoring with SLOs per consumer. Failure
modes I'd name explicitly: dual writes sneaking in wherever someone skips the
outbox "just this once"; the distributed monolith — services synchronously
calling each other over events-as-RPC, keeping the coupling while adding the
latency; entity-state events so anemic that every consumer calls back to the
source, re-coupling everything; and eventual consistency surprising product —
the UX for "placed but not yet confirmed" states is a product decision, and
surfacing it early is part of the architecture job.
