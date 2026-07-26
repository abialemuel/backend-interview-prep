## Correctness in Practice

Theory says messages duplicate, clocks lie, and processes pause. This file is the engineering response: the patterns that keep a real payment, order, or notification system correct while all of that happens. This is the most heavily probed material in senior/staff loops, because it maps one-to-one onto production incidents.

### Idempotency: the foundation

An operation is **idempotent** when performing it twice has the same effect as performing it once. In a system with retries — which is every system, because a timeout is indistinguishable from a slow success — idempotency is not a nice-to-have; it is the property that makes retries safe. Some operations are naturally idempotent (`SET x = 5`, `DELETE row 42`); the ones that matter (`charge $50`, `INSERT order`, `send email`) are not, and must be made so.

**Idempotency keys** are the standard mechanism (Stripe popularized the API shape):

1. The **client** generates a unique key per *logical operation* — not per attempt. A UUID minted when the user presses "Pay," reused across every retry of that press.
2. The server, atomically with the operation, records the key. The clean implementation is a table with a unique constraint:

```sql
INSERT INTO idempotency_keys (key, request_hash, response, status, created_at)
VALUES (:key, :hash, NULL, 'in_progress', NOW());
-- unique violation => key seen before:
--   status = 'completed'    -> return the stored response (replay)
--   status = 'in_progress'  -> 409 / block until the first attempt finishes
```

3. On completion, store the response against the key; on retry, **replay the stored response** instead of re-executing.

Details that separate a working implementation from a bug factory:

- **Atomicity**: the key insert and the business write must share a transaction, or the key row must be claimed as a lock *before* side effects. Recording the key after doing the work reintroduces the exact window it was meant to close.
- **Request fingerprinting**: store a hash of the request body and reject a retry whose body differs. Same key + different payload is a client bug; silently replaying the old response would hide it.
- **Concurrent retries**: two in-flight requests with the same key must not both execute. The unique constraint handles this (first insert wins; the second gets a violation and waits or 409s); a check-then-insert does not.
- **Scope**: the key is scoped per operation type and per client — `charge:cust_123:key` — so key collision across endpoints cannot replay the wrong response.

**Dedup windows.** Keys cannot live forever; pick retention matched to the retry horizon:

- 24 hours covers client-driven retries (Stripe's choice).
- Queue-driven flows may need longer — as long as a message can possibly be redelivered, *including DLQ replays weeks later*.
- State the consequence out loud: after expiry, a duplicate becomes possible again, so downstream systems need a second net — a unique *business* constraint like `(account_id, transfer_reference)` that rejects duplicates forever, not just within the window.

Beyond keys, the other idempotency tools:

- **Natural idempotency** via unique business constraints: `INSERT ... ON DUPLICATE KEY UPDATE` keyed on an order number.
- **Conditional writes**: `UPDATE ... WHERE version = :v` (optimistic concurrency), DynamoDB condition expressions, S3 conditional PUTs.
- **Idempotent consumers**: track processed message IDs in the same transaction as the state change (next section).

### Exactly-once: the myth and the achievable version

"Exactly-once delivery" over a network is impossible in the strict sense: an ack can be lost after processing, and the sender cannot distinguish "processed, ack lost" from "never processed" — so it must either retry (at-least-once) or not (at-most-once). Anyone selling exactly-once *delivery* is selling something else. What **is** achievable is **effectively-once processing**: at-least-once delivery plus deduplication or idempotency at the effect, so duplicates arrive but do not double-apply.

The end-to-end pattern, in Kafka vocabulary but portable to SQS/RabbitMQ:

1. **Producer side — the outbox.** Write the business row and the outgoing event in one local transaction; a relay (Debezium CDC, or a poller) publishes from the outbox table. No dual-write; the event is exactly as durable as the data. Delivery downstream is at-least-once.
2. **Broker side.** Kafka's idempotent producer dedups broker-side retries (producer ID + sequence number), and transactions make consume-transform-produce atomic across topics for `read_committed` consumers. Together these give exactly-once *within* Kafka-to-Kafka pipelines — Kafka Streams EOS is real and production-proven. The guarantee ends the moment the pipeline touches an external system.
3. **Consumer side — where it is actually won.** Make the effect idempotent, one of three ways:
    - Store the processed offset or message ID **in the same transaction** as the state change; on crash-restart, the state and the offset agree.
    - Dedup on a message ID with a unique constraint.
    - Make the operation naturally idempotent (an upsert, a conditional write).

The one-line interview answer: *"Exactly-once delivery is impossible; exactly-once processing is at-least-once delivery plus idempotent effects — outbox on the producer side, transactional or deduped consumption on the consumer side."*

### Distributed locks: efficiency vs correctness

First question to ask (Kleppmann's framing, and the right opener in an interview): is the lock for **efficiency** (avoid duplicate work; a rare double-run is wasteful but harmless) or **correctness** (a double-run corrupts data or money)? The answer determines the entire architecture.

**Efficiency locks — single Redis is fine:**

```text
acquire:  SET lock:job42 <random-token> NX PX 30000
release:  Lua script — GET, compare token, DEL only if it matches
          (never a bare DEL: you might delete the *next* holder's lock
           after your own already expired)
```

Accept that the lock can be lost on Redis failover (async replication may not have carried the key to the promoted replica) or outlived by a paused client. The blast radius is a duplicate job — which deeper idempotency should already make harmless.

**Redlock** — acquire the lock on a majority of 5 independent Redis nodes within a time budget — attempts correctness-grade locking on Redis. Know the debate:

- **Kleppmann's critique**: Redlock's safety rests on timing assumptions. A client GC pause, clock jump, or network delay *after* acquisition lets the lease expire while the client still believes it holds the lock — and Redlock issues **no fencing token**, so the resource cannot reject the stale holder.
- **antirez's response** defends the timing model as realistic for practical deployments.
- The pragmatic 2026 position: Redlock adds the operational cost of five Redis nodes without delivering the one thing a correctness lock needs — fencing. Use single-Redis for efficiency; use a consensus-backed lease for correctness.

**ZooKeeper/etcd leases — locks built on consensus:**

- **etcd**: grant a lease with a TTL (client keep-alives it), then atomically create the lock key bound to the lease; if the client dies, the lease expires and the key vanishes. The key's **revision** is a monotonic fencing token.
- **ZooKeeper**: ephemeral sequential znodes; the lowest sequence holds the lock; each waiter watches only its predecessor (no thundering herd). Session expiry releases the lock. The **zxid** serves as the fencing token.

And the punchline from `01-time-ordering-and-consensus.md` applies in full: even a consensus-backed lock cannot save a client that pauses *between checking the lock and writing*. **Correctness lives at the resource**: the store must check the fencing token — a conditional write, a version check — and reject stale holders. A distributed lock without downstream fencing is an efficiency lock with better marketing.

### Distributed transactions: 2PC vs sagas

**Two-phase commit** gives real atomicity across participants:

1. **Prepare**: the coordinator asks each participant to durably promise it can commit; participants vote yes and hold locks.
2. **Commit/abort**: the coordinator writes its decision durably, then instructs everyone.

Its problems are structural, not incidental:

- It is **blocking**: a participant that voted yes must hold its locks until the coordinator answers. A crashed coordinator leaves participants **in doubt** — locks held, indefinitely. (3PC "fixes" this only under network assumptions real networks violate.)
- It couples availability: the flow is down if *any* participant or the coordinator is down.
- It needs XA-style support end-to-end, which most modern infrastructure (HTTP APIs, queues) does not offer.

Where 2PC legitimately lives in 2026: *inside* systems, coordinated by consensus — Spanner and CockroachDB run 2PC where the coordinator's decision is itself replicated in a Paxos/Raft group, removing the single-coordinator-crash problem. Across independently operated microservices, 2PC is almost always the wrong answer, and interviewers expect the *why* (blocking + coordinator SPOF + operational coupling), not just "it's old."

**Sagas** are the microservice answer: a sequence of local transactions, each with a **compensating action** that semantically undoes it (refund, release reservation, cancel shipment). If step k fails, run compensations for steps k-1 back to 1.

| | Orchestration | Choreography |
| --- | --- | --- |
| Flow lives in | An explicit orchestrator / workflow engine | Event subscriptions across services |
| Visibility | Execution history is queryable | The workflow exists only in everyone's head |
| Coupling | Services couple to the orchestrator | Services couple to event schemas |
| 2026 tooling | Temporal, AWS Step Functions (durable execution: state, retries, timeouts first-class) | Kafka/EventBridge topics |
| Use for | Core money flows | Side effects and notifications |

Saga fine print that earns staff-level credit:

- **No isolation**: intermediate states are visible, and other transactions can act on data a compensation later reverts. Countermeasures: *semantic locks* (mark records `PENDING`), *commutative updates*, and *pivot ordering* — place the non-compensatable step (charging the card, sending the email) **last**, after every step that could force a rollback.
- **Compensations can fail** and must be retried to completion — so every step *and every compensation* must be idempotent. The patterns in this file compose; they are not alternatives.
- Some actions cannot be compensated, only apologized for. Design the order so those are pivots.

### Retries done right

Retries are the most-deployed and most-misused resilience tool: done naively they convert one failure into a self-inflicted DDoS, tripling load on a dependency at exactly its weakest moment.

- **Retry only what can succeed**: timeouts, 503s, connection resets — not 400/401/422, not deterministic 500s. And only operations that are idempotent or made so with keys.
- **Exponential backoff with full jitter**:

```go
delay := min(cap, base*(1<<attempt))     // exponential, capped
sleep(rand.Float64() * delay)            // FULL jitter: random in [0, delay)
```

  Full jitter empirically wins (the AWS Architecture Blog analysis) because it decorrelates clients; backoff without jitter re-synchronizes every client into waves that arrive together.

- **Retry budgets, not just counts**: cap retries as a *ratio* of primary traffic (e.g., ≤10% — the Finagle/Envoy `retry_budget` model). Per-request "3 retries" at every layer of a 4-deep call chain multiplies to 3⁴ = 81x amplification at the bottom. Retry at **one** layer — usually the one closest to the user that can make an informed decision — and budget it.
- **Deadline propagation**: pass the caller's remaining deadline downstream (gRPC does this natively); never retry past the point where the end client has already given up — that work consumes capacity and helps no one.
- **Circuit breakers** stop the beating entirely:

```text
CLOSED ──(error rate/volume over threshold)──▶ OPEN (fail fast, no calls)
   ▲                                             │ cooldown
   └──(probe succeeds)── HALF-OPEN ◀─────────────┘ (let a few probes through)
```

  Breakers protect the *caller's* resources (threads, connection pools) and give the callee air to recover. Pair each breaker with a per-endpoint **fallback** — stale cache, default value, queue-for-later — and put breaker state on a dashboard; an open breaker is an incident signal, not an implementation detail.

- **Load shedding and backpressure** are the server's side of the contract: reject early (429/503 with `Retry-After`) when queues grow, rather than melting slowly for everyone. Instrument retry rates — a silent retry storm looks like "traffic mysteriously tripled."

### Failure detection

You cannot *detect* failure in an asynchronous system — only **suspect** it. A timeout cannot distinguish a dead node from a slow network from a GC-paused process (which is why fencing exists). Practical detectors, in increasing sophistication:

- **Heartbeats + fixed timeout**: simple, binary, and wrong in both directions — the timeout is a guess.
- **Phi-accrual detection** (Cassandra, Akka): maintain a suspicion score from the observed *distribution* of heartbeat inter-arrival times; the threshold adapts to actual network behavior instead of a hardcoded 500 ms.
- **Gossip/SWIM-style membership** (Consul/Serf, Cassandra): nodes probe random peers; on a miss, they ask *other* nodes to probe indirectly before declaring suspicion — distinguishing "the target is down" from "my link to the target is down" — and disseminate membership epidemically. The Lifeguard extensions further cut false positives caused by the *prober* being slow.

Design rule: any action triggered by a failure detector — failover, rebalance, lock release — must be safe when the detector is wrong, because it will be. That means fencing on takeover, and hysteresis/cooldowns on automated remediation so a flapping detector cannot amplify a partial failure into a full one.

### Gray failures

**Gray failures** dominate senior on-call reality: the node is not down, it is *degraded* — 5% packet loss, one bad disk making every twentieth read take two seconds, a NIC negotiated down to 100 Mbps, one slow shard. The defining property (from the Microsoft Azure paper) is **differential observability**: the system's own detectors see health while clients see failure. `/health` returns 200 because the process is up, so the orchestrator keeps routing users into the degradation.

Countermeasures:

- **Deep health checks** that exercise the real dependency path — used judiciously; deep checks that fan out can themselves cascade an outage.
- **Client-side truth**: outlier ejection (Envoy ejects upstreams whose error/latency profile deviates from the pool) treats the *caller's* experience as the health signal.
- **Peer-relative comparison**: judge a node against its siblings, not against absolute thresholds — a node 10x slower than its peers is sick even if it is "within SLA."
- **Conservative automation**: rate-limited remediation with hysteresis, because a gray failure is precisely the thing that fools the detector driving the automation.

### Tail latency

**Tail latency** is the aggregate face of gray failure. The math that matters: if a single server's p99 is 1 s, a request that fans out to 100 servers waits on that p99 for 1 − 0.99¹⁰⁰ ≈ **63%** of requests. At scale, *the tail is the common case*. From Dean & Barroso's "The Tail at Scale":

- **Hedged requests**: send to one replica; if no answer by ~p95, duplicate to another and take the first response. A few percent of extra load buys a large tail cut. Hedge only idempotent reads, and cancel the loser.
- **Tied requests**: enqueue on two servers, each aware of the other; whichever *starts* first cancels its twin — better than hedging when queueing delay dominates.
- **Micro-causes**: GC pauses, queueing, background compaction and flushes, noisy neighbors, TCP retransmits. **Micro-cures**: request prioritization, breaking large work into small interleavable units, selective replication of hot data.
- **Measurement discipline**: SLOs bind user-visible percentiles, never averages — and never average percentiles across hosts; p99 must be computed over the whole population.

### Verifying any of this actually works

Staff-level credibility includes knowing that these properties rot silently and must be tested:

- **Fault injection / chaos engineering**: kill the worker between charge and ack; partition the primary from the failover controller; pause a lock holder with `SIGSTOP` past its TTL. Every pattern in this file corresponds to a fault you can inject in a staging environment, and the ones you have never injected probably do not work.
- **Jepsen-style testing**: generate concurrent operations against the system while injecting partitions and clock skew, then check the observed history against the claimed consistency model (the Knossos/Elle checkers). You will not run Jepsen against your CRUD app, but you should know that this is how the claims of databases you depend on get falsified — and that Jepsen reports are the best-written distributed systems education available.
- **Property-based invariant checks in CI**: "replaying this event stream twice yields the same balances," "any prefix of the saga plus its compensations restores the invariant." Idempotency is a property; test it as one.
- **Reconciliation in production**: a daily job diffing your ledger against the PSP, your order count against confirmation events, your cache against the source of truth. Reconciliation is the admission that the machinery above is probabilistic — and the mechanism that turns a silent correctness bug into a ticket instead of a customer complaint.

### The composed answer

These patterns are one system, not a menu. The staff-level composition for "service A calls service B, money moves":

1. The client mints an **idempotency key**; A records it transactionally and **replays** the stored response on retry.
2. A writes state + event via the **outbox**; the relay delivers **at-least-once**; consumers **dedup in-transaction**.
3. The cross-service flow is a **saga** — orchestrated, non-compensatable step last — with every step and compensation idempotent.
4. Every remote call carries a **deadline**, retries with **full jitter under a retry budget**, behind a **circuit breaker** with a per-endpoint fallback.
5. Anything taking exclusive ownership (leader, cron, migration) holds a **lease** and writes with a **fencing token** the store enforces.
6. Detection assumes **gray failure**: peer-relative metrics, outlier ejection, hedged reads for the tail — and every automated remediation is safe against a lying failure detector.

If you can narrate that composition over a concrete incident — see `04-interview-questions.md` for the drills — you are giving the answer the loop is designed to find.
