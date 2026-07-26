# Replication and Schema Design

This file covers two related disciplines: how MySQL is replicated for scale and HA, and how schema choices set you up (or not) for that scale. The first half is replication mechanics, topologies, and GTID. The second half is schema design — normalization, data types, primary keys, foreign keys, charset, partitioning, sharding, and online schema changes.

---

# Part A: Replication

## 1. The replication model

MySQL replication is **asynchronous, statement-or-row based, one-directional** at its core. The model has three components:

1. **Primary (master)** writes committed changes to its **binary log (binlog)**.
2. **Replica (slave)** has an **IO thread** that connects to the primary and streams binlog events into a local **relay log**.
3. The replica has a **SQL thread** (single-threaded in legacy replication; multi-threaded via MTS — Multi-Threaded Slave — using `slave_parallel_workers`) that applies relay log events to the replica's data, updating the replica's own binlog if `log_slave_updates=ON` (needed for chain topologies and for binlog-based failover).

```sql
-- Primary side (must already have binlog + a replication user)
CREATE USER 'repl'@'%' IDENTIFIED BY 'pw';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- Replica side
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='primary.example.com',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='pw',
  SOURCE_LOG_FILE='binlog.000123',
  SOURCE_LOG_POS=4,
  SOURCE_AUTO_POSITION=0;   -- set 1 with GTID
START REPLICA;     -- was START SLAVE (8.0.22+)
```

Terminology note: MySQL 8.0.22+ renamed `MASTER`/`SLAVE` to `SOURCE`/`REPLICA`. The old statements (`CHANGE MASTER TO`, `START SLAVE`, `SHOW SLAVE STATUS`) were deprecated in 8.0 and **removed in 8.4** — only the `SOURCE`/`REPLICA` forms exist now. Using the old syntax in an interview dates your knowledge to 8.0.

## 2. Asynchronous vs semi-sync vs synchronous

- **Asynchronous (default)** — the primary commits as soon as it writes its binlog; the replica may lag arbitrarily. Primary failure can lose transactions the replica had not yet received (data loss = lag). Fastest for writes; weakest consistency.
- **Semi-synchronous** — the primary blocks the commit until *at least one* replica has acknowledged receipt of the binlog event (`rpl_semi_sync_source_wait_for_replica_count`, default 1 — the old `..._master_wait_for_slave_...` plugin names are gone in 8.4). If the timeout expires it falls back to async. Two variants:
  - `AFTER_SYNC` (default in 8.0+): wait before the primary commits → no loss of committed transactions, but may have duplicates on failover if the replica received but the primary did not commit.
  - `AFTER_COMMIT`: wait after the primary commits → no duplicate, but commits the primary made may be lost on failover.
  Semi-sync significantly reduces RPO while still adding one round-trip latency per commit.
- **Synchronous / Group Replication (MGR)** — built into MySQL since 8.0. Uses a Paxos-style group communication protocol to replicate writes to a quorum of members before acknowledging. In **single-primary mode** one member accepts writes; in **multi-primary mode** all members can. Provides strong consistency within the group. **InnoDB Cluster** wraps MGR with MySQL Router (client routing + failover) and MySQL Shell admin tooling — the recommended HA stack.

The general trade-off: stronger consistency = more latency per write and tighter network requirements (lower RTT between members).

### Semi-sync vs Group Replication — the interview comparison

A common senior/staff question: "you need HA with no data loss — semi-sync or MGR?"

| Dimension | Semi-sync | Group Replication |
|-----------|-----------|-------------------|
| RPO | ~0 while a replica acks in time, but **degrades to async on timeout** — the guarantee is soft | 0 — a write is not committed until a quorum certifies it |
| Failover | External tooling decides (Orchestrator etc.); risk of promoting the wrong replica | Built-in membership + automatic primary election |
| Write latency | +1 network round trip | +group certification round (Paxos-style), typically similar or slightly higher |
| Conflicts | None (single writer) | Single-primary: none. Multi-primary: certification aborts conflicting transactions — the app must retry |
| Complexity | Low — plain replication + a plugin | Higher — group membership, flow control, strict requirements (every table needs a PK, `ROW` binlog, GTID) |
| Sweet spot | Bolting near-zero RPO onto an existing async topology | Green-field HA, especially as InnoDB Cluster with Router |

The one-line answer: semi-sync is a *durability patch* on classic replication whose guarantee silently weakens under stress; MGR is a *consensus-based membership system* with a hard guarantee, at the cost of operational complexity and stricter schema requirements.

### Provisioning replicas: the clone plugin (8.0.17+)

Before the clone plugin, building a new replica meant restoring a backup (XtraBackup, mysqldump) and hand-wiring coordinates. The **clone plugin** physically copies InnoDB data files from a running donor over the network and leaves the recipient with the donor's GTID set, ready to `START REPLICA`:

```sql
INSTALL PLUGIN clone SONAME 'mysql_clone.so';   -- on both donor and recipient
-- On the recipient:
SET GLOBAL clone_valid_donor_list = 'primary.example.com:3306';
CLONE INSTANCE FROM 'clone_user'@'primary.example.com':3306 IDENTIFIED BY 'pw';
-- Instance restarts with the donor's data + GTID_EXECUTED; then just point it at the source.
```

This is how InnoDB Cluster auto-provisions new members, and it is the modern answer to "how do you add a replica to a busy primary" (the donor stays online; monitor via `performance_schema.clone_status` / `clone_progress`).

## 3. Replication topologies

- **Primary → replica (1:1)** — simplest. Replica for read scale + HA.
- **Primary → N replicas** — read scale. Each replica adds primary-side bandwidth.
- **Circular / ring (A→B→C→A)** — each node is primary of the next; risk: a single node failure breaks the ring; cycle detection needed if `log_slave_updates` is on (it usually is). Rare in MySQL 8.x because Group Replication / InnoDB Cluster is cleaner for multi-writer needs.
- **Chain (A→B→C)** — B is both replica of A and primary of C. Useful to offload A's replication fan-out to B. Requires `log_slave_updates=ON` on B.
- **Primary → replica (delayed)** — replica runs `CHANGE REPLICATION SOURCE TO ... SOURCE_DELAY = 3600` to apply logs 1 hour late. Used as a poor-man's point-in-time recovery for accidental `DROP`.

Avoid dual-primary active/active with async replication on overlapping key ranges — writes to the same row in both primaries cause conflict resolution failures and split-brain. Active/passive is safe; multi-writer needs MGR or application-level sharding.

## 4. Replication lag

Lag is the time between a commit on the primary and the same change being applied on the replica. Sources:

- Single-threaded SQL thread on the replica (legacy; MTS helps).
- Large transactions — a single huge `DELETE` blocks the SQL thread for its duration.
- Long-running queries on the replica competing for CPU/IO with the SQL thread.
- Lock waits on the replica.
- Insufficient replica hardware (replication apply is single-threaded in dimension; write throughput matters).
- Network bandwidth.

### Monitoring lag

`SHOW REPLICA STATUS\G` exposes:

- `Seconds_Behind_Source` (the field's name since 8.0.22; older docs say `Seconds_Behind_Master`) — the **caveat**: it is computed as `current_time - timestamp_of_event_being_applied`, based on the primary's clock at event creation. If the IO thread is stalled but SQL thread is caught up, it shows 0 while actually stale. If the replica clock is skewed, the value is meaningless. If replication is stopped, it shows `NULL`.
- `SQL_Remaining_Delay`, `Replica_IO_Running`, `Replica_SQL_Running`, `Last_Error`, `Retrieved_Gtid_Set`, `Executed_Gtid_Set`.

Better monitoring via `performance_schema`:

```sql
SELECT CHANNEL_NAME, SERVICE_STATE, LAST_ERROR_NUMBER, LAST_ERROR_MESSAGE
FROM performance_schema.replication_connection_status;

SELECT THREAD_ID, SERVICE_STATE, LAST_ERROR_NUMBER
FROM performance_schema.replication_applier_status;

-- Per-worker lag in 8.0+:
SELECT * FROM performance_schema.replication_applier_status_by_worker;
```

`pt-heartbeat` (Percona Toolkit) is the production-grade approach: a heartbeat row updated on the primary at a known cadence; on the replica, compare the row's timestamp to the replica's clock to derive real lag independent of clock skew.

### Reads from a lagging replica

A read-from-replica pattern implies **eventual consistency** for reads. Application patterns to handle this:

- Read-your-writes: route reads from the user that just wrote to the primary for a short window (e.g. session-sticky routing or a write-token with TTL).
- Critical reads always go to primary.
- Cross-shard consistency: use GTID-based read-after-write checks (wait until the replica has applied the GTID of the user's write before serving the read — `WAIT_FOR_EXECUTED_GTID_SET`).

## 5. GTID (Global Transaction Identifier)

Each committed transaction on the primary is assigned a unique identifier of the form `<server_uuid>:<sequence>`, e.g. `3E11FA47-71CA-11E1-9E33-C80AA9429562:47`. The replica tracks which GTIDs it has executed (`Executed_Gtid_Set`); replication requests transactions it is missing from the primary.

Benefits over file/position replication:

- **Safe failover**: a new primary advertises its `Executed_Gtid_Set`; replicas know exactly which transactions they already have and can re-point without data loss or duplicates.
- **Easier topology changes**: `CHANGE REPLICATION SOURCE TO ... SOURCE_AUTO_POSITION=1` is portable across reconfigurations.
- **Detecting divergence**: comparing GTID sets across servers tells you whether they have identical data history.

Requirements: `gtid_mode=ON`, `enforce_gtid_consistency=ON`, all servers in the topology with GTID on, and no non-deterministic statements that violate GTID consistency (e.g. `CREATE TABLE ... SELECT`, transactions mixing transactional and non-transactional tables).

```sql
-- Switching on GTID requires a careful multi-step sequence:
SET @@GLOBAL.enforce_gtid_consistency = WARN;   -- observe warnings for a while
SET @@GLOBAL.enforce_gtid_consistency = ON;
SET @@GLOBAL.gtid_mode = OFF_PERMISSIVE;
SET @@GLOBAL.gtid_mode = ON_PERMISSIVE;
SET @@GLOBAL.gtid_mode = ON;
CHANGE REPLICATION SOURCE TO SOURCE_AUTO_POSITION=1;  -- on each replica
```

## 6. High availability concepts

- **Failover** — promote a replica to primary when the primary fails. Manual, via MHA/Orchestrator/ClusterControl, or automatic via InnoDB Cluster's MySQL Router + group membership.
- **Orchestrator** (open source) — topology discovery, refactoring (move a replica under a different primary), and automated failover with safety rules.
- **ClusterControl / MHA** — similar orchestration + HA, with their own conventions.
- **InnoDB Cluster** — the first-party solution: Group Replication + MySQL Router + MySQL Shell. Recommended for green-field deployments.
- **RPO** (Recovery Point Objective) — acceptable data loss on failover, in time. Async replication's RPO ≈ replication lag at the moment of failure; semi-sync RPO ≈ 0 (for the receipt-acknowledged replicas); MGR with quorum RPO = 0.
- **RTO** (Recovery Time Objective) — acceptable downtime. Automated failover with Orchestrator/Cluster → seconds; manual → minutes to hours.

In a failover, the promoted replica must have the most-up-to-date GTID set among the candidates; otherwise the failover loses or diverges from other replicas. GTID makes this check explicit.

---

# Part B: Schema design

## 7. Normalization

Normal forms reduce redundancy and update anomalies. Quick refresher:

- **1NF** — every column holds atomic values; no repeating groups or arrays in a cell. (MySQL `JSON` arguably violates 1NF if you query it like a column; use judiciously.)
- **2NF** — 1NF plus no non-prime attribute depends on a *proper subset* of a composite candidate key. Practically: a table with a composite key must not have attributes that depend on only part of the key.
  - Bad: `OrderLine(order_id, product_id, product_name)` where `product_name` depends only on `product_id` → move `product_name` to `Product(product_id, product_name)`.
- **3NF** — 2NF plus no transitive dependency: non-prime attributes depend only on the key, the whole key, and nothing but the key.
  - Bad: `Employee(emp_id, dept_id, dept_name)` where `dept_name` transitively depends on `emp_id` via `dept_id` → split into `Employee(emp_id, dept_id)` and `Department(dept_id, dept_name)`.
- **BCNF** — stricter 3NF: every non-trivial functional dependency's determinant is a candidate key. Catches edge cases 3NF misses (rarely matters in practice for typical schemas).

### When to denormalize

Normalize first for correctness and update ergonomics. Then denormalize **deliberately** for read performance when:

- A hot query joins many tables and a covering/precomputed form is much cheaper.
- You can maintain the denormalized column via triggers or async jobs, and consistency lag is acceptable.
- Reporting/aggregation needs repeated heavy joins — materialize via a periodic ETL into a star-schema-ish table.

Denormalization always adds write cost and consistency risk; do it on top of a sound normalized core, not instead of one.

## 8. Data types

General principle: choose the **smallest type that fits the domain**. Smaller columns mean smaller rows, smaller indexes, more rows per page, better cache hit rates, less memory.

- **Integer types**: `TINYINT` (1B), `SMALLINT` (2B), `MEDIUMINT` (3B), `INT`/`INTEGER` (4B), `BIGINT` (8B). The `(11)` display width was removed in MySQL 8.0.17 and is meaningless for storage. Use `BIGINT UNSIGNED` for surrogate PKs that may grow large.
- `BOOLEAN` is `TINYINT(1)`.
- **`VARCHAR(n)` vs `CHAR(n)`**:
  - `VARCHAR(n)` stores 1- or 2-byte length prefix + actual bytes. Saves space if values vary. Row format dynamic (InnoDB default since 5.7.9) stores `VARCHAR` overflow off-page.
  - `CHAR(n)` is fixed length, padded with spaces. Useful when values are uniformly sized (e.g. `CHAR(2)` for ISO country codes, `CHAR(36)` for UUID strings — though see UUID discussion below).
  - Long `VARCHAR` vs `TEXT`: `TEXT` is stored off-page by default; `VARCHAR` may or may not depending on row size. Indexing a `TEXT` requires a prefix length. Use `VARCHAR` when there is a known reasonable max, `TEXT` for unbounded content.
- **`DECIMAL(M, D)`** for money — exact base-10 arithmetic. `(M,D)` = total digits and decimal digits. `DECIMAL(12,2)` covers most currencies. Floats/doubles are binary floating-point and accumulate rounding error; never use for money.
- **Date/time types**:

| Type | Range | Storage | Notes |
|------|-------|---------|-------|
| `DATE` | `1000-01-01` to `9999-12-31` | 3 B | date only |
| `TIME` | `-838:59:59` to `838:59:59` | 3 B + optional fractional | can represent durations |
| `DATETIME` | `1000-01-01 00:00:00` to `9999-12-31 23:59:59` | 5–8 B | stores wall clock as entered; **no timezone conversion** |
| `TIMESTAMP` | `1970-01-01 00:00:01` UTC to `2038-01-19 03:14:07` UTC | 4–7 B | stored as UTC, converted to session `time_zone` on read/write; **2038 problem**; `TIMESTAMP` auto-initialize / auto-update on row change is convenient |

  - `TIMESTAMP` is good for "when this row was last touched" with auto-update; 4 B and the 2038 limit make it risky for far-future events (e.g. loan maturities in 2040+). Use `DATETIME` for those, storing UTC and converting in the app.
  - Avoid fractional seconds beyond what you need — each digit costs a byte.
- **`JSON`**: a typed, validated JSON document with binary storage, plus functions (`JSON_EXTRACT`, `JSON_ARRAYAGG`, `->`, `->>`). Good for semi-structured, schema-flexible attributes (e.g. user preferences, event payloads) where you query a few top-level keys. Bad when:
  - The same key is always present and queried → model it as a real column with an index.
  - You need transactional updates of subfields at scale → JSON partial updates help but are still harder to optimize than columns.
  - You rely on `JSON` for primary data → that's a document-store use case, not relational.
  Generate a **multi-valued index** to index array elements: `CREATE INDEX idx_tags ON items ((CAST(tags AS CHAR(20) ARRAY)));`
- **`BLOB`** for binary data, but prefer object storage for any sizeable blob (file paths in the row, bytes in S3/GCS).
- Avoid `ENUM` for anything that may change — adding a value is a schema change. Use a lookup table with a foreign key.
- **`VECTOR(n)`** (MySQL 9.0+): a fixed-length array of single-precision floats for storing ML embeddings, with functions like `STRING_TO_VECTOR` and `DISTANCE` (in HeatWave, full similarity search). Worth knowing it exists for "can MySQL store embeddings?" questions — but as of 9.7 LTS, community MySQL has no ANN index, so serious vector search still lives in PostgreSQL + pgvector, a dedicated vector store, or Redis (see [Redis as a vector store](../redis/01-data-structures-and-use-cases.md)).

## 9. Primary key choice

The clustered-index design makes PK choice consequential.

### Auto-increment `INT` / `BIGINT`

- Sequential → inserts append to the right edge of the B+Tree; minimal page splits.
- Compact (4 or 8 bytes) → secondary indexes are smaller (they store the PK in each leaf).
- Easy to reason about; no client-side collision.
- Disadvantages: predictable (enumeration attack — use a separate public id), single point of allocation (the auto-increment counter), reveals scale.

### UUID

- Globally unique without coordination — great for distributed inserts, offline clients.
- But the random UUID v4 has terrible locality: every insert lands at a random position in the B+Tree → page splits, fragmentation, larger index, more cache churn.
- Stored as `CHAR(36)` (37 bytes) wastes space; store as `BINARY(16)` (16 bytes) using `UUID_TO_BIN(uuid, swap=1)` to reorder the bytes so that time-ordering becomes monotonic.
- **UUID v7** (time-ordered, RFC 9562): the first 48 bits are a Unix timestamp in ms, the rest random. Inserts are roughly monotonic → comparable locality to auto-increment while keeping global uniqueness. Neither 8.4 LTS nor the 9.x line ships a native UUIDv7 generator (PostgreSQL 18 does — a nice contrast point); generate v7 in the application, or keep using `UUID_TO_BIN(UUID(), TRUE)` reordering.

```sql
CREATE TABLE events (
  id BINARY(16) PRIMARY KEY,
  payload JSON NOT NULL,
  created_at DATETIME(3) NOT NULL
);

-- Inserting with monotonic-friendly encoding:
INSERT INTO events (id, payload, created_at)
VALUES (UUID_TO_BIN(UUID(), TRUE), JSON_OBJECT('k','v'), NOW(3));
```

For most OLTP work, a `BIGINT UNSIGNED AUTO_INCREMENT` PK plus a separate non-enumerable public ID (a UUID or hash, indexed) is a pragmatic choice.

## 10. Foreign keys

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB;
```

InnoDB enforces referential integrity. Pros: catches orphaned rows, drives tooling (ORM, schema diagrams), documents relationships.

Cons:
- Each insert/delete/check on a child table performs a lookup on the parent's PK index — extra locks under some isolation levels and some contention at scale.
- Cascading actions can lock many rows unexpectedly (`ON DELETE CASCADE` of a popular parent).
- Concurrent writes to parent and child can deadlock more easily.
- They make bulk data load and schema migrations harder (constraint checks slow both).

High-scale shops sometimes disable FKs (keep the columns, drop the constraints) and enforce referential integrity in the application or via jobs. This is a trade-off: simpler operations, more risk of orphaned rows. Don't disable them lightly on green-field projects — the cost is real bugs.

## 11. Charset and collation

- **`utf8mb4`** is the only correct choice for text in MySQL 8.0+. The legacy `utf8` alias is `utf8mb3` — only 3-byte UTF-8, which **cannot store emoji or 4-byte CJK characters** (would silently error or replace with `?`). `utf8mb4` is the default since 8.0; pre-8.0 you had to set it explicitly.
- **Collation** determines equality and ordering. MySQL 8.0 default is `utf8mb4_0900_ai_ci` (`_ai` accent-insensitive, `_ci` case-insensitive). Use `_0900_as_cs` if you need case-sensitive or accent-sensitive comparisons. The collation must match across join columns or the join cannot use indexes — a frequent and silent performance bug.

```sql
CREATE TABLE users (
  email VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  ...
);
```

Index prefix lengths under `utf8mb4` are larger (4 bytes/char), affecting `VARCHAR` index sizing.

## 12. Partitioning

Partitioning splits a single logical table into physical sub-tables by a partitioning expression. InnoDB supports RANGE, LIST, HASH, KEY, and RANGE/LIST COLUMNS partitioning.

```sql
CREATE TABLE events (
  id BIGINT AUTO_INCREMENT,
  created_at DATETIME NOT NULL,
  payload JSON,
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (TO_DAYS(created_at)) (
  PARTITION p2024q1 VALUES LESS THAN (TO_DAYS('2024-04-01')),
  PARTITION p2024q2 VALUES LESS THAN (TO_DAYS('2024-07-01')),
  PARTITION p2024q3 VALUES LESS THAN (TO_DAYS('2024-10-01')),
  PARTITION pmax   VALUES LESS THAN MAXVALUE
);
```

What partitioning buys you:

- **Partition pruning**: queries that filter on the partitioning key scan only relevant partitions.
- **Drop old data cheaply**: `ALTER TABLE events DROP PARTITION p2024q1` is instant vs a giant `DELETE`.
- **Per-partition files**: smaller per-file working sets.

What partitioning does **not** buy you:

- Faster point lookups — those are still index lookups. An index is global to the table; each partition has its own local indexes. A lookup by a non-partitioning-key column searches every partition's index unless pruning applies.
- Concurrency or write throughput beyond what a single table gives you.
- Scale-out — partitioning is one-machine, not sharding.

Rules of thumb: partition by time when you have a retention pattern; do not partition just because the table is big if your queries are PK lookups. Keep partition count in the low hundreds at most.

## 13. Sharding

Sharding is the answer when a single MySQL instance no longer fits the write load or working set. Sharding strategies:

- **Range sharding** — assign key ranges to shards (e.g. user_id 0-1M on shard A, 1M-2M on shard B). Easy to reason about; risks hotspots if new inserts concentrate on the latest range.
- **Hash sharding** — `shard = hash(key) % N`. Even distribution, but re-sharding (changing N) is expensive: every key moves.
- **Consistent hashing** — distribute keys on a hash ring; adding a shard moves only ~1/N of keys. Standard in distributed caches (Memcached, Dynamo-style stores); for MySQL you typically see this implemented in the sharding middleware (Vitess, ShardingSphere) or the application.
- **Directory-based** — a lookup service maps each key to its shard. Most flexible; introduces a metadata service and an extra hop.

Cross-shard concerns you must plan for:

- Joins across shards are not supported; do them in the application or via denormalization.
- Distributed transactions across shards are awkward (see XA caveats). Prefer saga / outbox patterns.
- Global uniqueness for IDs — UUID or a Snowflake-style generator.
- Cross-shard counts/sorts — aggregate in the application.
- Rebalancing — pre-split shards, use range-based sharding with consistent hashing, or use a middleware that supports live resharding (Vitess).

Sharding is a system-design topic; the schema design lesson is *design your schema and IDs so sharding is possible later* (surrogate integer PKs or UUIDs, no implicit assumption of locality).

## 14. Online schema changes

Historically `ALTER TABLE` blocked writes (or rewrote the whole table) on InnoDB. Modern options:

- **`INSTANT` DDL** (MySQL 8.0.12+): adds a metadata-only change — no copy, no lock, no replication lag. Examples: add a column at the *end* of a table (8.0 only; 8.4 supports adding columns anywhere instantly), drop a column (8.0.29+, instant), rename a column, set a column default, change an `ENUM` by appending a value. Visible in `EXPLAIN` of the `ALTER` and `information_schema.INNODB_ALTER_TABLE`. Not all changes are instant; check the manual's "Online DDL Operations" matrix.
- **`INPLACE` DDL**: performed in-place with metadata changes and possibly row-level rebuild; may allow concurrent DML but still requires a brief `EXCLUSIVE` lock at start/end.
- **`COPY`** algorithm: the legacy path; builds a new table, copies all rows, swaps names. Blocks writes for the entire copy on a hot table. Avoid.

```sql
-- Fast, instant, metadata-only:
ALTER TABLE users ADD COLUMN nickname VARCHAR(64), ALGORITHM=INSTANT;

-- Prefer INSTANT, fall back to INPLACE, never COPY:
ALTER TABLE users ADD COLUMN nickname VARCHAR(64), ALGORITHM=INSTANT, LOCK=NONE;
```

For changes that cannot be done instantly or in-place (e.g. changing a column type, restructuring), use an external online-schema-change tool:

- **`pt-online-schema-change`** (Percona Toolkit) — creates a shadow table, adds triggers to keep it in sync, copies rows in chunks, swaps at the end. Well-tested, trigger-based.
- **`gh-ost`** (GitHub) — same idea but streams binlog events to keep the shadow table in sync instead of triggers; avoids trigger overhead and lock contention with the application. Preferred at large scale.

Both have caveats around foreign keys, unique-key changes, and paused-state behavior; test against a non-prod replica first.

Always verify the algorithm MySQL picked:

```sql
-- After ALTER:
SHOW CREATE TABLE users\G
-- Or observe via performance_schema.events_stages_current during the ALTER.
```

## 15. Schema design checklist

- Smallest type that fits.
- `BIGINT UNSIGNED` auto-increment PK, or monotonic-friendly UUID (v7 or `UUID_TO_BIN(..., TRUE)`).
- `utf8mb4` + `utf8mb4_0900_ai_ci` everywhere; matching collations on join columns.
- `DECIMAL` for money.
- Use `TIMESTAMP` for row-touched columns with auto-update; `DATETIME` (UTC) for far-future dates.
- Foreign keys on green-field; consider dropping constraints only at known scale pain.
- Normalize first, denormalize deliberately with a maintenance story.
- Index hot read paths with covering composite indexes.
- Partition by time when there is a retention pattern.
- Keep schema shardable (surrogate keys, no cross-table implicit joins).
- Plan migrations as `ALGORITHM=INSTANT` first, then `gh-ost`/`pt-osc`, never `COPY` on a hot table.

**PostgreSQL contrast** for replication questions: PostgreSQL replicates the physical WAL by default (byte-level, whole cluster) with optional logical replication per table, while MySQL replication has always been logical (binlog events), which is why per-schema filtering, heterogeneous topologies, and tools like gh-ost/Debezium grew up around MySQL first. See [PostgreSQL advanced features](../postgresql/03-advanced-features.md) for the WAL/logical-replication side.