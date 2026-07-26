# Distributed Systems

The system design section (`06-system-design`) covers the architecture layer: which boxes to draw, where to put the cache, how to shard. This section covers the layer underneath — the theory and correctness machinery that makes those boxes actually work when the network partitions, the clock skews, and the retry fires twice. It is the difference between "I would use a leader with replicas" and being able to explain what happens in the 800 ms after the leader dies.

Senior and staff loops probe this layer directly. A mid-level candidate is asked to design a rate limiter; a staff candidate is asked why their rate limiter double-counts during a Redis failover, whether their lock is safe when a GC pause outlives the lease, and what "exactly-once" actually means in their pipeline. Interviewers at this level are rarely testing recall of Raft's message names — they are testing whether you have internalized that **the network is asynchronous, clocks lie, and any process can pause at any point for any length of time**, and whether your designs survive those facts.

The material here assumes you have read the system design section. Where that section says "replicas lag, so reads are eventually consistent," this section explains what the replica lag actually is, what anomalies it permits, which consistency model bounds them, and what you pay to close the gap.

## What this section covers

| File | Description |
| --- | --- |
| README.md | This overview: how this layer relates to system design, and the reading order. |
| 01-time-ordering-and-consensus.md | Physical vs logical clocks, Lamport and vector clocks, happens-before, linearizability vs serializability vs sequential consistency, Raft in detail, Paxos briefly, leader election, quorums, split-brain, fencing tokens. |
| 02-replication-and-partitioning.md | Single-leader, multi-leader, and leaderless replication; conflict resolution (LWW, CRDTs); consistent hashing and rebalancing; partitioning strategies and hot keys; read repair and hinted handoff; Dynamo-style tunable consistency (R + W > N). |
| 03-correctness-in-practice.md | Idempotency keys and dedup windows, exactly-once as myth vs achievable end-to-end patterns, distributed locks (Redlock caveats, ZooKeeper/etcd leases), 2PC vs sagas, retries done right (backoff + jitter, retry budgets, circuit breakers), failure detection, gray failures, tail latency. |
| 04-interview-questions.md | 15+ graded Q&A weighted toward senior/staff, including incident-style scenarios (double charges, distributed rate limiting, split-brain postmortems). |

## How to think about this material

Three habits separate candidates who know the vocabulary from candidates who can reason:

1. **Always ask "what if it happens twice, and what if it never happens?"** Every message can be duplicated (retries) and every message can be lost (timeouts). A design that assumes exactly-one delivery of anything is broken by construction. Most of `03-correctness-in-practice.md` is the toolkit for surviving this.
2. **Distrust wall clocks and in-flight assumptions.** A node that held a lock, checked a lease, or was elected leader can be paused (GC, VM migration, page fault storm) for longer than any timeout, then resume convinced it still holds the role. Fencing tokens and epochs exist because "I checked a moment ago" proves nothing. This is the core of `01-time-ordering-and-consensus.md`.
3. **Name the consistency model, not the adjective.** "Strongly consistent" and "eventually consistent" are marketing words. Linearizable, sequentially consistent, causal, read-your-writes — these are models with precise anomaly sets, and picking the weakest model that still makes the product correct is exactly the trade-off skill interviewers want to see.

## Recommended reading order

1. `01-time-ordering-and-consensus.md` — the foundations. Everything else refers back to happens-before, quorums, and fencing. Read it first even if consensus feels academic; leader election and split-brain come up in nearly every senior loop.
2. `02-replication-and-partitioning.md` — the mechanics of the storage systems you name-drop in design rounds. If you say "Dynamo-style" or "consistent hashing" out loud, you should be able to survive two follow-up questions on each.
3. `03-correctness-in-practice.md` — the most directly job-relevant file. Idempotency, retries, and distributed locks are day-one production concerns and the single most common staff-level probe ("your service double-charged a customer — go").
4. `04-interview-questions.md` — self-test after the first three. Answer out loud before reading the model answer; at this level, precision of language is a large part of the score.

As in the rest of this repo, concepts are generic but examples map onto a working stack: AWS, MySQL, Redis, Kafka, Go/PHP. The theory here is decades old and stable — Lamport's happens-before paper is from 1978 — but the systems that embody it (etcd, DynamoDB, Kafka's KRaft, CockroachDB) are exactly the ones you will be asked about in 2026.
