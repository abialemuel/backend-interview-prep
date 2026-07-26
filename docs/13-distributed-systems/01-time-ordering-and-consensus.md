## Time, Ordering, and Consensus

This file covers the hardest-to-fake part of distributed systems knowledge: why you cannot trust clocks, how to order events without them, what the consistency models actually promise, and how a cluster of unreliable machines agrees on anything at all.

### Why physical clocks cannot be trusted

Every machine has two clocks, and neither gives you what distributed coordination needs — a shared notion of "before":

- The **time-of-day clock** (wall clock) is synced via NTP and can jump — forward or *backward* — when corrected. It is meaningful across machines only to the accuracy of the sync.
- The **monotonic clock** only moves forward and is good for measuring local durations (timeouts, latency), but its absolute value is meaningless and incomparable across machines.

What goes wrong in practice:

- NTP sync over the public internet is typically accurate to tens of milliseconds, and can be off by far more under congestion, asymmetric routes, or a misconfigured server.
- Clocks **skew**: quartz oscillators drift (order of parts-per-million — seconds per week), at different rates per machine, between syncs.
- Processes lose time wholesale: a VM paused for live migration, a laptop lid closed, a container frozen by the OOM killer's cgroup throttling.
- Leap seconds have caused real outages; the industry answer (smearing) means two "correct" clocks intentionally disagree during the smear.

The canonical production disaster: last-write-wins conflict resolution keyed on wall-clock timestamps silently drops writes from the node whose clock runs behind. Cassandra's LWW has eaten real data this way. Code that computes `expiry = now() + 30s` on one machine and checks `now() < expiry` on another is comparing two unrelated instruments.

The exception that proves the rule: Google Spanner's **TrueTime** exposes a bounded uncertainty interval (`[earliest, latest]`, typically single-digit milliseconds, backed by GPS and atomic clocks in every datacenter) and *waits out the uncertainty* before committing — that wait is how Spanner achieves external consistency. AWS has followed with microsecond-accurate time sync on supported EC2 instances, and clock-bound APIs are increasingly available. But unless your design explicitly budgets for uncertainty intervals the way Spanner does, treat cross-machine timestamp comparison as unreliable.

Rule of thumb: wall clocks are fine for TTLs, metrics, logs, and humans. They are not fine for ordering, uniqueness, or mutual exclusion between machines.

### Happens-before: ordering without clocks

Lamport's 1978 insight: you do not need physical time to order events; you need **causality**. Event `a` **happens-before** event `b` (written `a → b`) if:

1. `a` and `b` are in the same process and `a` comes first, or
2. `a` is the send of a message and `b` is its receipt, or
3. transitivity: `a → c` and `c → b`.

If neither `a → b` nor `b → a`, the events are **concurrent** — and no amount of timestamp precision changes that. They are causally unrelated; any observed order is legitimate, and a system that must pick one is making a *decision*, not discovering a fact.

### Lamport clocks and vector clocks

**Lamport clocks** assign each event a single counter:

- Increment the local counter on every local event.
- On send, attach the counter to the message.
- On receive, set `counter = max(local, received) + 1`.

This guarantees `a → b ⇒ L(a) < L(b)`: the numbering is consistent with causality, and ties broken by process ID give a total order. But the converse fails — `L(a) < L(b)` does **not** imply `a → b`. Lamport clocks cannot tell you whether two events were concurrent, which is exactly what conflict detection needs.

**Vector clocks** fix that. Each of N processes keeps a vector of N counters:

- Increment your own slot on each local event.
- On send, attach the whole vector.
- On receive, take the element-wise max of the two vectors, then increment your own slot.

Comparison recovers causality exactly:

```text
Lamport:  a → b  ⇒  L(a) < L(b)          (one direction only; cannot detect concurrency)
Vector:   a → b  ⇔  V(a) < V(b)          (V(a) < V(b): every element <=, at least one <)
          neither dominates              ⇔  a and b are concurrent
```

Concurrency detection is how Dynamo-style stores surface conflicting **siblings** that need resolution, rather than silently picking a winner. The cost is O(N) metadata per value, which is why real systems use bounded variants (dotted version vectors in Riak) or give up and use LWW — see `02-replication-and-partitioning.md` for what that costs.

**Hybrid logical clocks (HLC)** are the modern compromise you should be able to name: a timestamp that tracks wall time when clocks are healthy but carries a logical component so it never violates happens-before, even when the wall clock jumps. CockroachDB and MongoDB use HLCs; they give you "timestamps that are both human-meaningful and causally safe" without TrueTime hardware.

### The consistency model zoo, precisely

These three get conflated constantly; keeping them straight is a reliable senior-level signal.

| Model | Domain | Guarantee |
| --- | --- | --- |
| **Linearizability** | Single-object reads/writes | Every operation appears to take effect atomically at some instant between invocation and response, consistent with real time. Once any client reads the new value, every later read sees it. |
| **Serializability** | Multi-object transactions | The outcome equals *some* serial execution of the transactions. Says nothing about real time — a serializable database may legally order yesterday's transaction "after" today's. |
| **Sequential consistency** | Operations across processes | All processes observe the same total order, and that order respects each process's program order — but not real-time order across processes. |

Points worth making explicitly in an interview:

- **Linearizability is "recency."** It is what people mean by "strong consistency" for a register, a lock, or a leader flag. It requires coordination — quorums or a single serialization point — and it is exactly what CAP says you must give up to stay available under partition.
- **Serializability is "isolation."** It is a property of transaction systems. A single-node MySQL running `SERIALIZABLE` is serializable; the question of linearizability doesn't even arise until you replicate.
- **Strict serializability** = serializability + linearizability: the serial order agrees with real time. Spanner targets it (CockroachDB and FoundationDB come close, with documented caveats); it is the strongest practical model and the most expensive.
- **Causal consistency** — causally related writes are seen in order; concurrent writes in any order — is the strongest model achievable while remaining available under partition. That makes it the theoretical sweet spot, even though few off-the-shelf stores expose it directly.
- **Eventual consistency** is a liveness promise only: stop writing and replicas converge, eventually. It permits *any* anomaly in the meantime and bounds nothing without an explicit freshness SLA.

The **session guarantees** are the pragmatic middle ground, and most product bugs blamed on "eventual consistency" are really a missing session guarantee:

| Guarantee | Promise | Typical implementation |
| --- | --- | --- |
| Read-your-writes | A session sees its own writes | Sticky routing, or read tokens carrying last-seen LSN/offset |
| Monotonic reads | A session never sees time go backward | Sticky replica per session |
| Monotonic writes | A session's writes apply in order | Ordered per-session write path |
| Writes-follow-reads | Writes are ordered after the reads they depended on | Causal tokens |

These are fixable with sticky sessions or a session token, without paying for linearizability. "Cart needs read-your-writes; checkout needs a linearizable stock decrement" beats "make it strongly consistent" every time.

!!! note "Interview framing"
    When asked "should this be strongly consistent?", translate: which anomaly would the user actually observe, which model excludes that anomaly, and what is the cheapest mechanism providing that model?

### Consensus: the problem

Consensus: multiple nodes propose values; all non-faulty nodes must **agree** on a single value that was actually **proposed**, and must eventually **terminate**. Leader election, atomic broadcast, distributed locks, and group membership are all consensus in disguise — solve one and you can build the others.

The **FLP impossibility** result (1985) proves no deterministic algorithm can guarantee consensus in a fully asynchronous system with even one faulty process. Practical algorithms escape it by adding timeouts (partial synchrony): they are always **safe** (never two decisions) and **live whenever the network behaves for long enough**. That framing — safety unconditional, liveness conditional — is the right one-sentence summary of both Paxos and Raft.

### Raft, properly

Raft (2014) is the consensus algorithm you should be able to explain end-to-end; it runs inside etcd (and therefore Kubernetes), Consul, CockroachDB, TiKV, Redpanda, and Kafka's KRaft controller quorum. It decomposes consensus into leader election, log replication, and safety rules.

**Roles and terms.** Every node is a **follower**, **candidate**, or **leader**:

```text
                 times out, starts election          wins majority
   follower  ────────────────────────────▶ candidate ─────────────▶ leader
      ▲                                        │                      │
      │        sees higher term (from anyone)  │   sees higher term   │
      └────────────────────────────────────────┴──────────────────────┘
```

Time divides into numbered **terms**; each term has at most one leader. Terms act as a logical clock: any node that sees a higher term than its own immediately reverts to follower and adopts it. Stale leaders are neutralized by term numbers, not by wall clocks.

**Leader election:**

1. Followers expect periodic heartbeats (empty `AppendEntries`) from the leader. If none arrives within a randomized **election timeout** (e.g., 150–300 ms), the follower increments its term, becomes a candidate, votes for itself, and sends `RequestVote` to all peers.
2. A node grants its vote if it has not voted in this term **and** the candidate's log is at least as up-to-date as its own (compare the last entry's term, then log length). One vote per term is what makes two leaders in one term impossible.
3. A majority of votes makes a leader, which starts heartbeating. A split vote (simultaneous candidates) times out and re-runs with fresh randomized timeouts — the randomization is the entire liveness trick.

**Log replication:**

1. Clients send commands to the leader, which appends them to its log and replicates via `AppendEntries`. Each entry is identified by `(term, index)`.
2. `AppendEntries` carries the previous entry's `(term, index)` as a consistency check; a follower whose log does not match rejects, and the leader backs up until the logs agree, then overwrites the follower's divergent suffix. The leader's log is the truth.
3. Once a majority holds an entry, the leader marks it **committed**, applies it to its state machine, and replies to the client. Followers learn the commit index from subsequent messages.

**The safety subtleties interviewers use to separate levels:**

- **Election restriction**: the up-to-date-log voting rule guarantees every elected leader already holds all committed entries — Raft never needs to ship missing committed data to a new leader.
- **A leader only directly commits entries from its own term**; earlier-term entries commit implicitly once a current-term entry on top of them reaches a majority. This is Figure 8 in the paper — the scenario where a "majority-replicated" old-term entry can still be lost — and citing it correctly is a strong signal.
- **Reads are not automatically linearizable.** A deposed leader that has not yet noticed can serve stale reads. Fixes, in increasing cheapness: commit a no-op read entry through the log; **ReadIndex** (leader confirms leadership with one heartbeat round, then serves from local state); **leader leases** (serve reads while a clock-bound lease holds — note this quietly reintroduces a clock assumption). etcd exposes exactly this choice: linearizable reads by default, `serializable` (possibly stale, local) reads as an opt-out.

**Ops-relevant details:**

- Membership changes go through the log too — single-server changes or joint consensus. Never swap the config out-of-band.
- Logs are compacted via snapshots; a slow follower catches up from a snapshot, not the full log.
- A 5-node cluster tolerates 2 failures. Even-sized clusters add no fault tolerance over the next odd size down (4 nodes still needs 3 for a quorum) — they only add a vote that can tie.

### Paxos, briefly

Paxos (Lamport, published 1998) solves single-value consensus in two phases:

1. **Prepare**: a proposer picks a proposal number `n` and asks acceptors to promise not to accept anything numbered lower; each acceptor replies with any value it has already accepted.
2. **Accept**: the proposer must adopt the highest-numbered value it heard back (this rule is the entire safety core — a new proposal can never overwrite a possibly-chosen value); it then asks acceptors to accept `(n, value)`. A value accepted by a majority is **chosen**.

**Multi-Paxos** amortizes phase 1 by electing a stable leader and running only accept rounds per log entry — at which point it converges on essentially Raft's shape, with the leader machinery famously left as an exercise. That underspecification is why Raft ("designed for understandability") displaced it in new systems; Paxos variants persist inside Google (Chubby, Spanner) and in research such as **Flexible Paxos** — phase-1 and phase-2 quorums need only intersect *each other*, enabling, e.g., fast small-quorum steady-state writes at the cost of expensive elections.

For interviews: know the two-phase shape, the "adopt the highest accepted value" rule, and why Multi-Paxos ≈ Raft.

### Quorums and split-brain

A **quorum** is any set of nodes large enough that two quorums must intersect; majority quorums (`⌈(N+1)/2⌉`) are the standard because they tolerate the most failures. Intersection is the entire point — any decision (a vote, a committed write) is visible to any later quorum through at least one common node.

**Split-brain** is what quorums prevent: a partition producing two nodes that both believe they are leader, both accepting writes, diverging irreconcilably. With majority quorums, at most one side of any partition can elect or commit; the minority side stalls — that is CAP's C-over-A choice made concrete.

Classic split-brain factories to name in interviews:

- **Two-node HA pairs** with a keepalive link: when the link dies, both promote. Fix: a third tiebreaker/witness node, or a true quorum.
- **Failover driven by a single health-checker**: the checker is partitioned from a perfectly healthy primary, promotes a replica, and now two primaries accept writes. GitHub's 2018 incident (43 seconds of cross-DC partition, MySQL writes accepted on both coasts, hours of reconciliation) is the canonical public example.
- **Redis Sentinel with `min-replicas-to-write` unset**: an isolated primary keeps accepting writes that are discarded when it rejoins as a replica.

### Fencing tokens: the last line of defense

Quorums stop two nodes from *being elected* leader in the same term. They do not stop a **paused** old leader from *acting* on stale belief:

1. A process acquires a lock/lease or leadership.
2. It suffers a stop-the-world GC pause, VM migration, or long I/O stall — longer than the lease TTL.
3. The lease expires; a new holder is elected and proceeds.
4. The old process resumes and fires its write, having "checked" the lock before pausing. Check-then-act over a network is never atomic.

The fix is a **fencing token**: a strictly monotonically increasing number issued with every lock grant or election — Raft's term, ZooKeeper's zxid, etcd's lease/revision, Kafka's producer and leader epochs. Every protected resource checks the token and rejects anything older than the highest it has seen:

```sql
-- The resource enforces fencing, not the client:
UPDATE jobs
SET state = 'done', fence = :token
WHERE id = :id AND fence < :token;
-- 0 rows affected => stale holder; the write is dropped and should be logged loudly
```

Two implications worth stating out loud:

1. **The resource must participate.** A fencing token nobody checks is decoration. If the downstream store cannot do a conditional write (compare-and-set, conditional PUT — even S3 has supported `If-Match`/`If-None-Match` conditional writes since 2024), fencing cannot protect it.
2. **This is why "lock for efficiency" vs "lock for correctness" matters.** If the lock merely avoids duplicate work, a rare double-execution is fine and a simple Redis lock suffices. If the lock guards correctness, you need fencing end-to-end — the lock service alone can never be sufficient, because the client between the lock and the resource can always pause. This framing defuses most of the Redlock debate (see `03-correctness-in-practice.md`).

### Leader election in practice

You rarely implement Raft; you lease leadership from a system that already runs it:

- **etcd**: create a lease with a TTL and keep-alive it; use a transaction (`compare create-revision = 0`) to atomically claim a key bound to the lease. Leadership is possession of the key; the key's revision doubles as a fencing token. The `concurrency` package wraps this as an election recipe.
- **ZooKeeper**: ephemeral sequential znodes under an election path; the lowest sequence number is the leader; each node watches only its immediate predecessor (avoiding herd effects). Session expiry deletes the ephemeral node, triggering the successor.
- **Kubernetes**: the `Lease` object in `coordination.k8s.io` — the standard pattern for "run this controller in exactly one replica," backed by etcd underneath.
- **Kafka**: since KRaft replaced ZooKeeper (mandatory from Kafka 4.0, 2025), a Raft quorum of controller nodes elects the cluster controller and replicates all metadata as a log — a nice case study in replacing an external coordination service with embedded consensus.

Whatever the mechanism, the failure mode is identical: the elected leader can always be paused past its lease. Lease-based leadership is an **optimization for the common case**; fencing at the resource is the correctness mechanism. Say both sentences and you have answered the question at staff level.

### Putting it together

A checklist for any design that involves coordination:

1. Where do I compare timestamps across machines? (Each instance is a bug until proven otherwise.)
2. What orders my events — a single leader's log, a Lamport/vector clock, a Kafka partition offset? Is that order causal, total, or real-time?
3. Which consistency model does each read path actually need, and what is the cheapest mechanism that provides it?
4. Who elects the leader, what is the quorum, and what happens on the minority side of a partition?
5. When the old leader wakes up from a pause, what rejects its writes?

If you can walk a design through these five questions unprompted, you are operating at the level this section targets.
