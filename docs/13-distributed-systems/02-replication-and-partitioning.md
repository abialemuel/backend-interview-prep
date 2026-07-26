## Replication and Partitioning

Replication answers "how many copies, and who may write?"; partitioning answers "which node owns which keys?". Every distributed datastore you will name in an interview — MySQL with replicas, DynamoDB, Cassandra, Redis Cluster, Kafka — is a specific combination of answers to those two questions. This file gives you the mechanics behind the names, at the depth a senior/staff follow-up expects.

### The three replication topologies

| Topology | Who accepts writes | Conflicts | Canonical systems |
| --- | --- | --- | --- |
| **Single-leader** | One leader; followers replicate its log | None (leader serializes) | MySQL/Postgres replication, Redis, Kafka partitions, MongoDB replica sets |
| **Multi-leader** | One leader per site/region; leaders replicate to each other | Yes — concurrent writes to the same key in different sites | Bidirectional MySQL replication, DynamoDB global tables, CouchDB, offline-first apps |
| **Leaderless** | Any replica, via quorum reads/writes | Yes — surfaced as siblings or resolved by LWW | Dynamo, Cassandra, ScyllaDB, Riak |

### Single-leader: the default, and where its bodies are buried

Single-leader should be your default in interviews: writes are serialized in one place, so there are no write conflicts, and the replication stream (binlog/WAL/oplog) gives followers a well-defined order. The costs: the leader is a write bottleneck, and failover is a hard problem wearing a simple name.

Failover decomposes into three steps, each with its own failure mode:

1. **Detect** the leader's death — a timeout, which is a guess. Too short and you fail over on a GC pause (flapping); too long and you extend the outage.
2. **Elect** a successor — a consensus problem (see `01-time-ordering-and-consensus.md`). A single health-checker doing the promotion is a split-brain factory.
3. **Repoint** clients and, critically, **fence the old leader** so it cannot keep accepting writes if it was merely slow, not dead.

The replication mode determines what failover can lose:

- **Asynchronous**: the leader acks commits before followers have them. Fast, and the new leader may be missing the old leader's last acknowledged writes — which are silently discarded when the old leader rejoins. Acknowledged data loss is a *chosen* RPO, not an accident; say so.
- **Synchronous**: every commit waits for a follower. No loss on single-node failure, but write latency includes a network round trip, and a dead sync follower stalls all writes.
- **Semi-synchronous / quorum-acked**: the practical compromise — one sync follower (MySQL semi-sync) or a majority quorum (Group Replication, Postgres `synchronous_standby_names = ANY 1 (...)`, Raft-based storage). Survives any single failure at a bounded latency cost.

**Replication lag** is the other daily cost: followers are behind by milliseconds normally and minutes under load, so the read path is eventually consistent. The anomalies and their session-guarantee fixes (read-your-writes, monotonic reads) are covered in `01`; the operational habit is to expose lag as a first-class metric and route lag-sensitive reads to the leader.

Lag has characteristic pathologies worth recognizing on sight:

- **A large transaction** (bulk `UPDATE`, schema migration) serializes through single-threaded or per-schema replication apply and stalls everything behind it — the reason online-schema-change tools (gh-ost, pt-osc) exist.
- **Replica read storms**: a lagging replica serves staler data, which triggers application retries, which load the replica further — lag begets lag. Cap staleness: pull a replica out of rotation past a lag threshold rather than letting it poison reads.
- **Failover into lag**: promoting the *least-lagged* replica is a controller responsibility; promoting a random one maximizes data loss.

### Multi-leader: geography and offline

Multi-leader exists for two reasons: **geography** (each region accepts writes locally at sub-10 ms and replicates asynchronously cross-region) and **offline clients** (your phone's calendar is a leader that syncs later). The price is that two regions can accept conflicting writes to the same key and both commit locally — conflict resolution becomes mandatory, not optional.

Things to say before the interviewer says them:

- Retrofitting multi-leader onto a schema that assumed serialized writes — uniqueness constraints, auto-increments, foreign keys, triggers — is a minefield. Auto-increment collides immediately (fix: interleaved ranges or UUIDs/ULIDs); uniqueness simply cannot be enforced asynchronously.
- Replication topology matters: all-to-all is standard; ring/star topologies create ordering hazards where one node's outage stalls or reorders others' updates.
- DynamoDB added a **strong-consistency mode for global tables** (multi-region strongly consistent writes at cross-region latency cost, GA 2025) — the industry keeps re-learning that many "multi-leader" use cases actually wanted a single serialization point they could pay latency for.

**Conflict avoidance is the cheapest conflict resolution.** Before designing merges, route around them:

- **Home-region ownership**: each key (user, tenant, document) has a home region that serializes its writes; other regions forward. Writes for a given key never conflict; you pay cross-region latency only for the minority of non-local writes, and you need a (rare, careful) home-migration procedure.
- **Data ownership by field**: split the record so each region writes disjoint fields — merge becomes trivial union.
- **Immutable events instead of mutable state**: two regions appending to a log never conflict; the conflict moves into the (deterministic, testable) fold that builds state from the log.

Say "avoid, then resolve" before diving into CRDTs and you signal that you have operated one of these systems, not just read about them.

### Leaderless: Dynamo-style quorums

Leaderless drops the leader entirely: the client (or a coordinator node) writes to all N replicas and considers the write successful after W acks; reads query and succeed after R responses. Availability is excellent — there is no failover because there is nothing to fail over — and the cost is that staleness and conflicts are *normal operation*, handled by the machinery below.

**Tunable consistency — R + W > N:**

- **R + W > N** means every read quorum intersects every write quorum, so at least one replica in any read saw the latest write; version numbers pick the newest.
- Common settings for N=3: `W=2, R=2` (balanced), `W=3, R=1` (fast reads, fragile writes), `W=1, R=1` (fast and loose — R + W ≤ N means reads can miss writes entirely).
- Cassandra spells these `ONE / QUORUM / ALL / LOCAL_QUORUM`. `LOCAL_QUORUM` — quorum within the local datacenter — is the multi-region workhorse, because cross-DC quorums are latency poison.

!!! warning "Quorum ≠ linearizable"
    R + W > N alone does **not** give linearizability. Sloppy quorums break the intersection guarantee; two writes with the same resolved version can interleave; and a read racing a partially-completed write can be non-monotonic *across clients* — one client sees the new value, a later read by another client sees the old one — unless read repair is synchronous and writes are per-replica atomic. Present Dynamo-style quorums as "very fresh with high probability," not "consistent."

**Sloppy quorums and hinted handoff.** The write path never blocks on the home replicas: if one is down, a **sloppy quorum** accepts the write on a fallback node outside the key's home set, tagged with a **hinted handoff** — "this belongs to node A; deliver it when A returns." This trades read-guarantee strength for write availability: W acks no longer imply the read quorum will intersect, so a read during the outage can miss the write even with R + W > N. Riak defaults sloppy quorums on; Cassandra's hinted handoff is the same idea attached to its own quorum rules.

**Anti-entropy: how replicas converge.**

- **Read repair**: when a read gathers R responses and finds stale replicas, it writes the newest value back to them (synchronously or async). Hot keys self-heal; cold keys never do.
- **Merkle-tree repair**: a background process compares replicas' key ranges via hash trees — compare roots, descend only into mismatched subtrees — and syncs the differences cheaply. Cassandra's `nodetool repair`; DynamoDB runs the equivalent internally. This catches the cold keys read repair misses; forgetting to run it is a classic operational bug (tombstones resurrect deleted data past `gc_grace_seconds`).

### Conflict resolution

When concurrent writes to the same key both commit (multi-leader or leaderless), someone must decide what the value is.

**Last-write-wins (LWW)** tags each write with a timestamp and keeps the highest:

- Simple, convergent, zero metadata growth — and it **silently discards committed writes**.
- Because the timestamps are wall clocks, "last" is decided by whichever node's clock runs fastest (see `01` on skew). Cassandra is LWW to its core (per-cell timestamps).
- Acceptable when each key has a single natural writer (sensor readings, session blobs keyed by user). Dangerous whenever concurrent writers are legitimate: counters, carts, collaborative anything.

**Version vectors / siblings**: detect concurrency (vector clocks per key) and keep both versions; the next reader receives siblings and must merge. Honest — no data silently lost — but it pushes merge complexity to every client. The canonical anecdote: early Dynamo shopping carts merged siblings with a set union, so deleted items resurrected. The *merge function* is the real design decision.

**CRDTs (conflict-free replicated data types)** make the merge automatic by construction: types whose merge is commutative, associative, and idempotent, so replicas converge regardless of delivery order or duplication ("strong eventual consistency"). Know these concretely:

| CRDT | What it is | Merge | Use |
| --- | --- | --- | --- |
| G-Counter | Per-replica increment counters | Element-wise max | Distributed counters |
| PN-Counter | Two G-Counters (inc, dec) | Element-wise max of both | Counters with decrement |
| LWW-Register | Value + timestamp | Keep higher timestamp | Simple fields (same LWW caveat) |
| OR-Set | Elements tagged with unique add-IDs | Union; remove deletes only *observed* tags | Sets where add-wins is right (carts, favorites) |
| Sequence CRDTs (RGA, Yjs, Automerge) | Ordered lists/text | Positional identifiers | Collaborative editing |

```text
G-Counter, 3 replicas — each increments only its own slot, value = sum:
  A: [3, 1, 0]   B: [2, 4, 0]   merge = element-wise max = [3, 4, 0]  =>  7
Idempotent (re-merge changes nothing), commutative, associative => convergent.
```

By 2026, sequence CRDTs are mainstream infrastructure — Figma-style multiplayer, Yjs/Automerge in production — and "CRDTs for the collaborative surface, single-leader DB for the money" is a defensible staff-level architecture sentence.

Limits to volunteer before being asked:

- CRDTs guarantee **convergence, not invariants**: a PN-Counter happily goes negative under concurrent decrements, so "never oversell inventory" still needs coordination — reservations, escrowed quota splits, or a single leader for that key.
- Metadata grows (tombstones, per-replica entries) and needs garbage collection, which itself needs coordination — the complexity comes back at the edges.

### Partitioning (sharding) strategies

Replication copies the whole keyspace; partitioning splits it. Three families:

1. **Range partitioning** — contiguous sorted key ranges per shard (HBase, CockroachDB/TiKV, DynamoDB sort keys within a partition, manual MySQL range shards).
   - Wins: efficient range scans; splits adapt to data size.
   - Risk: **hot ranges** — monotonically increasing keys (timestamps, auto-increments) hammer the last shard forever. Fix with a distributive prefix (tenant ID, hashed prefix), paying with scatter-gather on time-range queries.
2. **Hash partitioning** — shard by `hash(key)` (Cassandra's Murmur3 token ring, DynamoDB's partition key, Redis Cluster's CRC16 into 16384 slots).
   - Wins: uniform distribution by construction.
   - Loss: no cross-shard range scans. Compound keys recover locality: Cassandra hashes the partition key and sorts by clustering key *within* the partition — "hash across, range within" is the pattern to cite.
3. **Directory-based** — an explicit lookup service maps keys or tenants to shards (Vitess, most home-grown tenant sharding).
   - Wins: maximum flexibility; move any tenant anywhere; isolate noisy tenants.
   - Loss: the directory is a critical, cached, must-not-lag dependency, and someone has to build its tooling.

**Secondary indexes** do not partition along the primary key; there are two shapes:

- **Local indexes** (document-partitioned): each shard indexes its own rows; a query on the indexed field scatter-gathers across *all* shards. Fine at low shard counts, brutal at high ones.
- **Global indexes** (term-partitioned): the index itself is partitioned by the indexed value (DynamoDB GSIs). Reads hit one index partition; writes fan out to index partitions, and the index is asynchronously consistent with the base table — a read-after-write on a GSI can miss.

### Consistent hashing and rebalancing

Naive `hash(key) mod N` reshuffles almost every key when N changes — a cache-cluster resize becomes a total cache flush, a database resize becomes a full migration. **Consistent hashing** fixes this: place nodes and keys on a hash ring; each key belongs to the next node clockwise. Adding or removing a node moves only ~1/N of the keys — those between the new node and its predecessor.

```text
        0 ──────────── ring ──────────── 2^32
        │   A₁    B₁      C₁  A₂   B₂  C₂ ...      (virtual nodes)
 key k ─┘   hash(k) lands here ──▶ owned by next vnode clockwise
 replicas: next N *distinct physical* nodes clockwise (skip same machine/rack)
```

Two refinements make it production-grade:

- **Virtual nodes (vnodes)**: each physical node claims dozens-to-hundreds of ring positions. This smooths the load variance of random placement, spreads a dead node's load across many successors instead of one, and lets heterogeneous hardware take proportional vnode counts. Cassandra, Riak, and Dynamo all default to vnodes.
- **Replication on the ring**: a key's N replicas are the next N distinct physical nodes clockwise (skipping vnodes of the same machine, and typically the same rack/AZ) — the "preference list."

Alternatives worth name-dropping with one clause each:

- **Rendezvous (HRW) hashing** — for each key, pick the node with the highest `hash(key, node)`; no ring to maintain, minimal disruption, great inside cache client libraries.
- **Jump consistent hash** — O(1) and stateless, but supports only numbered buckets that grow at the end: good for fixed pools, useless for arbitrary node removal.
- **Maglev hashing** — Google's lookup-table variant optimizing even load and O(1) lookup for load balancers.

**Rebalancing** in practice takes one of three shapes:

1. **Fixed partitions, many more than nodes** (Kafka's model, Redis Cluster's 16384 slots): partitions never split; they just move between nodes. Choose the partition count generously up front — you cannot cheaply change it later.
2. **Dynamic splitting** (HBase, CockroachDB/TiKV): ranges split when they exceed a size/load threshold — adapts to skew automatically.
3. **Operator-driven resharding** (the MySQL/Vitess answer): dual-write or CDC-backfill the new topology, verify, cut over reads, then writes.

Never let rebalancing be fully automatic *and* fast: a flapping health check that triggers mass partition movement is a self-inflicted outage (the data movement saturates the network exactly when the cluster is already struggling). Rebalance with a throttle and a human-visible plan.

### Hot keys and skew

Partitioning distributes keys, not load. One celebrity, one viral post, one tenant running a load test — and a single partition melts while the rest of the cluster idles. This is the follow-up to every sharding answer, so have the ladder ready:

1. **Detect**: per-key/per-partition metrics — Redis `--hotkeys`, DynamoDB CloudWatch contributor insights, Kafka partition-lag skew. You cannot fix what you attribute to "the database being slow."
2. **Cache in front**: a hot key is by definition cacheable for some staleness budget; even a 1-second in-process cache flattens most read storms.
3. **Salt / split the key**: write `key#<rand 0..k>` across k sub-keys and aggregate on read (counters, rate limiters), or replicate a read-hot key k times and read a random copy. Costs read fan-out or aggregation.
4. **Isolate**: move the hot tenant or key to dedicated capacity — directory-based sharding shines here. DynamoDB's adaptive capacity does a version of this automatically, isolating a hot partition key onto its own partition.
5. **Redesign the key**: the durable fix. Monotonic keys get a distributive prefix; global counters become sharded counters; "one row everyone fights over" becomes an append log aggregated asynchronously.

### Worked example: an orders table at scale

A concrete pass through the decisions, DynamoDB-flavored but portable:

- **Access patterns first**: fetch an order by ID; list a customer's orders newest-first; and (for ops) find all orders in a time range. Partitioning is designed backward from these, never from the entity model.
- **Primary key**: partition key `customer_id`, sort key `order_ts#order_id`. Hash across customers, range within a customer — "list my orders" is a single-partition query, and no customer's history spans partitions.
- **The hot-key check**: is any single customer's write rate near a partition's throughput ceiling? A B2C marketplace: no. A B2B platform where one enterprise tenant is 30% of traffic: yes — that tenant gets a salted key (`customer_id#0..7`) or dedicated capacity via directory routing.
- **The time-range query** conflicts with the customer-partitioned layout — serve it from a global secondary index keyed by `order_date` bucket, accepting that the GSI lags the table and that a date bucket is itself a monotonic hot key, so salt it (`order_date#rand(0..15)`) and scatter-gather 16 index partitions on read.
- **Replication/consistency**: order writes need read-your-writes for the customer who just placed the order (session-sticky or strongly consistent read on the confirmation page) and eventual consistency is fine for everything else. Multi-region: active-passive for the money path, because "order placed exactly once" is a uniqueness invariant that does not merge.

The generic lesson: every partitioning answer is *access patterns → key design → skew check → secondary-path plan*, in that order.

### Choosing, quickly

A compact decision frame for interviews:

- Need transactions, constraints, and a familiar operational story ⇒ **single-leader** (MySQL/Postgres), replicas for reads, shard by tenant/user when writes outgrow one primary — and name the failover mechanism and the replication-lag consequences unprompted.
- Need multi-region active-active writes or offline clients ⇒ **multi-leader**, and immediately name the conflict strategy *per data type*: LWW where a single natural writer exists, CRDTs for counters/sets/collaborative state, app-level merge for the rest, and a single serialization point for uniqueness/money.
- Need always-on writes at massive scale with per-key access ⇒ **leaderless/Dynamo-style**, quorum-tuned per operation, with read repair + Merkle repair, sloppy quorums acknowledged as a durability trade, and a hot-key plan.
- Everywhere: consistent hashing with vnodes (or many fixed partitions) for placement, and a rebalancing story that moves data deliberately, not reactively.

The pattern the interviewer is listening for is not the topology name — it is that every choice arrives with its failure mode attached: async replication *loses acknowledged writes on failover*, LWW *drops concurrent writes*, sloppy quorums *weaken read guarantees*, hash sharding *kills range scans*, GSIs *lag their base table*. Attach the cost yourself, before the follow-up question does.
