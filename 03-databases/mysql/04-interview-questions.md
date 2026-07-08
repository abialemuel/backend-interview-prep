# MySQL Interview Questions

Model answers are written for an InnoDB / MySQL 8.4 / 8.0 context. Where the answer is engine-specific, that is called out.

## Easy

### Q1: Why does InnoDB use a B+Tree rather than a B-Tree or a hash index for its indexes?

A B+Tree stores all values in leaf nodes and keeps internal nodes for routing only, giving very high fan-out, so the tree stays 3–4 levels deep for tens of millions of rows and point lookups are a handful of page reads. The leaves are linked as a doubly-linked list in key order, which makes range scans (`>`, `BETWEEN`, `ORDER BY`) cheap — you find the first leaf and walk the list. A plain B-Tree stores values in every node, lowering fan-out and forcing re-traversal for ranges. A hash index gives O(1) point lookups but no ordering at all, so it cannot support range scans, ordered reads, or `LIKE 'prefix%'`. InnoDB therefore uses B+Tree for both the clustered index and secondary indexes (an adaptive hash index is built on top for hot point lookups, but it is not user-managed).

### Q2: What is the difference between a clustered index and a secondary index in InnoDB?

In InnoDB the clustered index *is* the table: its B+Tree leaf nodes store the full row data, keyed by the primary key. A secondary index is a separate B+Tree whose leaf nodes store the indexed column(s) plus the **primary key value** — not a physical row pointer and not the full row. To read a non-indexed column from a row found via a secondary index, InnoDB does a second B+Tree lookup in the clustered index using the PK stored in the secondary index leaf; this is the bookmark lookup. A covering index avoids this second lookup by including every column the query needs in the secondary index (the PK is always implicitly present), which `EXPLAIN` reports as `Using index`.

### Q3: What is a covering index, and how do you know the query is using one?

A covering index is a secondary index that contains every column a query needs (filter columns, returned columns, and the PK is always there implicitly). When that holds, InnoDB can answer the query entirely from the index B+Tree without descending the clustered index — no bookmark lookup, much less I/O. In `EXPLAIN` this appears as `Using index` in the `Extra` column. Example: given `KEY (country, created_at)`, `SELECT id, country, created_at FROM users WHERE country='DE'` is covered because `id` is the PK and is stored in every secondary index leaf, so `Extra: Using index` shows.

### Q4: How do you read the `type` column of EXPLAIN, from best to worst?

Best to worst: `const` (unique/PK equals a constant, one row), `eq_ref` (join where each outer row matches exactly one inner row via a unique/PK index), `ref` (non-unique index equality), `range` (index range scan via `>`, `BETWEEN`, `IN`, `LIKE 'x%'`), `index` (full scan of an entire index, cheaper than full table scan), `ALL` (full table scan, the red flag unless the table is tiny or most rows qualify). The most important `Extra` values are `Using index` (covering), `Using where` (post-filter in server layer), `Using filesort` (extra sort pass — usually fixable with the right index), `Using temporary` (temp table for `GROUP BY`/`DISTINCT`), and `Using index condition` (Index Condition Pushdown filters rows in the engine using index entries).

### Q5: Why does `WHERE DATE(created_at) = '2024-01-01'` not use the index on `created_at`?

Because the index is keyed on the raw column value, not on `DATE(created_at)`. Wrapping the column in a function makes the predicate non-sargable: the optimizer cannot turn `f(col) = const` into a range on `col`. The fix is to rewrite the predicate as a range on the bare column:

```sql
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';
```

If rewriting is impossible, MySQL 8.0+ supports a **functional index** on the expression: `ALTER TABLE users ADD KEY idx_d ((DATE(created_at)));`.

### Q6: What does the leading-column rule mean for a composite index?

An index on `(a, b, c)` is a B+Tree ordered by `a` then `b` then `c`. It can be used for queries that filter on a prefix: `(a)`, `(a,b)`, `(a,b,c)`. A query that filters only `b` or `b,c` cannot use it — the rows are not ordered by `b` at the top level, so the index cannot prune. A query filtering `a = 1 AND c = 3` uses only the `a` portion: once a non-leading column is missing the engine stops using further index columns for filtering (though they may still be usable for covering). The corollary is to put the equality-filtered column first in a composite index, and the range-filtered column last.

### Q7: What is ACID?

**Atomicity** — a transaction is all-or-nothing; InnoDB uses the undo log so `ROLLBACK` (or crash before commit) reverts every change. **Consistency** — the database moves from one valid state to another, enforced by A+I+D plus constraints (`NOT NULL`, FK, `CHECK`, unique). **Isolation** — concurrent transactions do not observe each other's uncommitted state at the chosen isolation level; implemented with locks and MVCC. **Durability** — once committed, the change survives a crash; InnoDB achieves this via write-ahead logging to the redo log with `innodb_flush_log_at_trx_commit=1` (fsync at commit).

### Q8: What is the difference between `DATETIME` and `TIMESTAMP`?

`DATETIME` stores the wall-clock value as entered (5–8 bytes, range year 1000–9999) and does no timezone conversion. `TIMESTAMP` stores a Unix epoch seconds value (4 bytes without fractional seconds, range 1970–2038) and converts to/from the session `time_zone` on read/write, plus it supports convenient `DEFAULT CURRENT_TIMESTAMP` / `ON UPDATE CURRENT_TIMESTAMP`. Use `TIMESTAMP` for row-touched timestamps; use `DATETIME` (storing UTC) for far-future dates because of `TIMESTAMP`'s 2038 limit. Both can take fractional seconds (cost: 1 byte per 1–2 digits).

### Q9: Why must MySQL text columns be `utf8mb4`?

The legacy `utf8` alias in MySQL is `utf8mb3` — only 3-byte UTF-8, which cannot encode 4-byte characters such as emoji and some CJK symbols; inserting one fails or replaces with `?`. `utf8mb4` is the full UTF-8 encoding and is the default since MySQL 8.0. All new text columns and tables should use `utf8mb4`. The default collation in 8.0+ is `utf8mb4_0900_ai_ci` (accent-insensitive, case-insensitive). Also note that join columns must share the same collation or the join cannot use indexes — a frequent silent performance bug.

## Medium

### Q10: Explain phantom reads, and why InnoDB's REPEATABLE READ prevents them when the SQL standard says it should not.

A phantom read happens when a transaction runs a range query twice and the second run returns extra rows because another transaction committed an insert into that range in between. Per the SQL standard, REPEATABLE READ allows phantoms (it only guarantees re-reads of *existing* rows are stable). InnoDB's REPEATABLE READ prevents phantoms by two mechanisms: MVCC gives plain `SELECT` a stable snapshot for the whole transaction (so inserts by others are not visible), and locking reads/writes use **next-key locks** (record lock plus a gap lock) on the index range touched, which block other transactions from inserting into the gap. So InnoDB's RR is effectively stronger than the standard's — close to SERIALIZABLE for the queries that take locks, without the full-locking cost of SERIALIZABLE.

### Q11: What is MVCC, and how does it let readers not block writers?

MVCC (Multi-Version Concurrency Control) keeps multiple versions of each row so that a transaction sees a consistent snapshot without taking locks on the rows it reads. InnoDB writes the previous version of a row to the **undo log** before modifying it; the row header carries a `ROLL_PTR` to the most recent undo record, forming a version chain. When a transaction starts a statement (READ COMMITTED) or the whole transaction (REPEATABLE READ), InnoDB builds a **read view** listing the active transaction IDs; a row version is visible only if it was committed before the read view's horizon. A reader walks the version chain to find the latest visible version without locking the current row, so a writer updating that row is not blocked. Old versions are purged once no active transaction needs them — which is why long-running transactions cause undo-log bloat.

### Q12: What is the difference between `SELECT ... FOR UPDATE` and `SELECT ... FOR SHARE`?

`FOR UPDATE` acquires exclusive next-key locks on the rows (and gaps) matching the predicate, blocking other transactions' `UPDATE`/`DELETE` and other `FOR UPDATE`/`FOR SHARE` on those rows until commit. `FOR SHARE` acquires shared next-key locks: multiple `FOR SHARE` readers can coexist, but they block writers and `FOR UPDATE` readers. Both bypass the MVCC snapshot to read current committed data. They are used to implement pessimistic locking — e.g. read a balance, decide, then update, all under one transaction so no other transaction can change the balance in between. MySQL 8.0+ adds `NOWAIT` (fail immediately if a row is locked) and `SKIP LOCKED` (skip locked rows), the latter being the canonical pattern for job-queue tables.

### Q13: How do deadlocks happen in InnoDB, and how do you avoid them?

A deadlock is a cycle in the wait-for graph of lock holders: T1 holds lock on row A and waits for row B, while T2 holds B and waits for A. InnoDB detects deadlocks automatically (it builds the wait-for graph on each lock acquisition) and rolls back the smaller-cost transaction with error `ER_LOCK_DEADLOCK` (1213); the application must retry. Avoidance strategies: (1) acquire locks in a consistent order across all transactions (canonical row ordering eliminates cycles); (2) keep transactions short to shrink the window; (3) always access rows via an index — an unindexed `UPDATE`/`SELECT FOR UPDATE` locks every row of the table and turns into a table lock; (4) consider READ COMMITTED when gap locks are unnecessary, since RC uses no gap locks and reduces lock surface; (5) use `SKIP LOCKED` for work-queue patterns.

### Q14: When would you choose optimistic locking over pessimistic locking?

Pessimistic locking (e.g. `SELECT ... FOR UPDATE` then `UPDATE ... COMMIT`) takes locks for the whole think time and is right when contention is high or the cost of a retry is high — classic example is moving money between accounts. Optimistic locking assumes conflicts are rare: read the row with its `version`, do the work, then on commit do `UPDATE ... SET version = version + 1 WHERE id = ? AND version = ?`. If `affected_rows == 0`, a concurrent writer changed the row and you retry the whole transaction. Optimistic wins under low contention (e.g. user profile edits) because it takes no locks and scales concurrency; it loses under high contention because retries cascade. The decision is workload-driven, not a feature of the isolation level.

### Q15: What are the redo log, undo log, and binlog each for?

The **redo log** is InnoDB's write-ahead log at the engine layer: every change to a buffer-pool page is recorded there and fsynced at commit (`innodb_flush_log_at_trx_commit=1`), giving durability — on crash, replaying redo reconstructs the in-memory state and then undo rolls back uncommitted transactions. The **undo log** stores pre-images of modified rows for `ROLLBACK`, MVCC snapshots, and crash-recovery rollback of uncommitted transactions; it is also what long-running transactions keep alive (causing purge lag). The **binlog** is a server-layer ordered log of all committed changes, used for replication (replicas apply it) and point-in-time recovery. ROW is the recommended `binlog_format` because it is deterministic; STATEMENT is unsafe for non-deterministic SQL like `NOW()` or `UUID()`.

### Q16: Explain the two-phase commit between the redo log and binlog.

To keep the engine (redo) and the server (binlog) crash-consistent, InnoDB and the binlog do an internal 2PC: (1) InnoDB writes a prepare record to the redo log and fsyncs; (2) the server writes the binlog events and fsyncs the binlog (`sync_binlog=1`); (3) InnoDB writes a commit marker to the redo log. On crash recovery, for each transaction in the "prepared" state InnoDB checks whether the binlog contains it: if yes, commit (the replica already saw it); if no, rollback. This guarantees a transaction is either durably committed on the primary and present in the binlog for replicas, or completely rolled back — no half-applied state that would diverge replicas from the primary. Combined with `sync_binlog=1` and `innodb_flush_log_at_trx_commit=1` you get full ACID plus crash-safe replication at the cost of one fsync per commit (mitigated by group commit).

### Q17: What is replication lag and what is wrong with `Seconds_Behind_Master`?

Replication lag is the time between a commit on the primary and the same change being applied on the replica. `Seconds_Behind_Master` is computed as `replica_clock - timestamp_of_event_currently_being_applied`, where the event's timestamp was set by the primary's clock. The caveats: (a) if the IO thread has stalled but the SQL thread is caught up, the value can read 0 while the replica is actually stale; (b) if the replica's clock is skewed versus the primary, the number is meaningless; (c) when replication is stopped the value is `NULL`. Production monitoring should use `performance_schema.replication_applier_status_by_worker` for per-worker state, and a heartbeat approach such as `pt-heartbeat` — a row updated on the primary at a known cadence whose replica-side timestamp gives true lag independent of clock skew.

### Q18: What is GTID and why is it better than file/position replication?

A GTID is a globally unique transaction identifier of the form `<server_uuid>:<sequence>` assigned to every committed transaction on the primary. A replica tracks the set of GTIDs it has executed (`Executed_Gtid_Set`) and asks the primary for any transactions it is missing. Compared to file/position replication, GTID makes failover safe: a candidate primary advertises its executed GTID set, so the orchestrator can pick the most-advanced replica and other replicas can re-attach without losing or duplicating transactions. Topology changes become `CHANGE REPLICATION SOURCE TO ... SOURCE_AUTO_POSITION=1` without manual log-file math. GTID also makes divergence detection explicit — comparing GTID sets across servers tells you whether they share the same data history. Requirements: `gtid_mode=ON`, `enforce_gtid_consistency=ON`, all servers in the topology enabled, and no GTID-inconsistent statements.

### Q19: What is the difference between asynchronous and semi-synchronous replication?

In **asynchronous** replication (the default) the primary commits as soon as its own binlog is written; the replica may be arbitrarily behind. Primary failure can lose transactions the replica had not yet received — RPO equals lag at the moment of failure. In **semi-synchronous** replication the primary blocks the commit until at least one replica (configurable via `rpl_semi_sync_master_wait_for_slave_count`) acknowledges receipt of the binlog event; if the timeout fires it degrades back to async. With `AFTER_SYNC` (default in 8.0+) the wait happens before the primary commits, so no committed transaction is lost on failover (though a duplicate may appear if the replica received but the primary did not commit). Semi-sync gives near-zero RPO at the cost of one round-trip per commit; async gives the lowest write latency but the weakest durability.

### Q20: What are normalization and denormalization, and when would you denormalize?

Normalization is structuring tables so each non-key attribute depends on the whole key and nothing but the key (1NF → 2NF → 3NF → BCNF), which removes redundancy and update anomalies. For example, `OrderLine(order_id, product_id, product_name)` violates 2NF because `product_name` depends only on `product_id`; you split out `Product(product_id, product_name)`. Denormalization is the deliberate reintroduction of redundancy for read performance. You denormalize when a hot query joins many tables and a precomputed or duplicated column avoids the join — for example, storing a `customer_country` snapshot on `orders` to avoid joining to `customers` on every read. The cost is update overhead (you must keep the duplicate in sync via triggers, jobs, or application discipline) and a consistency risk; do it on top of a sound normalized core, not instead of one.

### Q21: Why is a random UUID a poor primary key for InnoDB, and what is the fix?

InnoDB's clustered index is ordered by the primary key, so inserts land at the position determined by the key. A random UUID v4 puts every new row at a random position in the B+Tree, causing page splits, fragmentation, larger index files, and worse cache hit rates than a sequential key. Also, storing the UUID as `CHAR(36)` wastes 37 bytes per row in every secondary index (which stores the PK). Two fixes: (1) store the UUID compactly as `BINARY(16)` using `UUID_TO_BIN(uuid, TRUE)` which reorders bytes so the time-bits come first, making inserts roughly monotonic; (2) prefer **UUID v7** (time-ordered, RFC 9562) which has a millisecond timestamp in the high bits so inserts are naturally ordered while remaining globally unique. For most OLTP, a `BIGINT UNSIGNED AUTO_INCREMENT` PK plus a separate non-enumerable public UUID is a pragmatic compromise.

### Q22: What are the trade-offs of foreign keys?

Foreign keys enforce referential integrity: an insert into a child table checks the parent exists, a delete from a parent can be `RESTRICT`/`CASCADE`/`SET NULL`, and the schema is self-documenting. The costs: every child write performs an extra lookup on the parent's PK index (and may take shared locks on the parent row, increasing lock contention); `ON DELETE CASCADE` on a popular parent can lock or modify many rows in one statement; FKs make bulk loads and some migrations slower because of constraint checks; and they complicate cross-shard schemas where parent and child live on different shards. Many high-scale shops keep the FK columns but drop the constraints, enforcing integrity in the application or via background jobs. For green-field projects the safety is usually worth the cost; do not drop them lightly.

### Q23: What does `OFFSET`-based pagination cost, and what is the alternative?

`SELECT ... ORDER BY x LIMIT 20 OFFSET 10000` does not skip 10000 rows for free — the engine still produces and discards them, so the cost grows linearly with `OFFSET`. At large offsets this becomes a near-full-table scan. The alternative is **keyset (cursor) pagination**: remember the last row's sort key and use it as a filter:

```sql
SELECT id, created_at FROM orders
WHERE merchant_id = 7
  AND (created_at, id) < ('2024-05-01 12:00:00', 98765)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

With a composite index `(merchant_id, created_at, id)` this is a cheap indexed range scan returning exactly 20 rows at any depth. The tie-breaker on `id` is required so the comparison is strictly monotonic. Cursor pagination is also stable under concurrent inserts/deletes, unlike `OFFSET`.

## Hard

### Q24: Design the composite index for this query and justify the column order.

```sql
SELECT user_id, status, created_at
FROM orders
WHERE merchant_id = 7
  AND status = 'PAID'
  AND created_at >= '2024-01-01'
ORDER BY created_at DESC
LIMIT 50;
```

The right index is `(merchant_id, status, created_at)`. Order rules: equality columns first (`merchant_id = 7`, `status = 'PAID'`) so the engine can use the index for filtering; the range column last (`created_at >= ...`) because a range stops the use of further index columns for narrowing. Putting the range column last also means the index order matches the `ORDER BY created_at DESC`, so the engine can satisfy the sort by walking the index backwards (a "backward index scan") with no `filesort`. The query also happens to be covering because the returned columns (`user_id` is the PK and is implicitly in every secondary index, `status`, `created_at`) are all in the index, so `EXPLAIN` will show `Using index`. Without the index you would see `type=ALL` and `Using filesort`.

### Q25: A query that should use an index is doing a full table scan. List the likely causes.

In roughly decreasing frequency: (1) a function on the indexed column (`WHERE DATE(col)=...`, `WHERE LOWER(name)=...`) makes the predicate non-sargable — rewrite as a range or add a functional index; (2) implicit type conversion, e.g. comparing a `VARCHAR` column to a numeric literal, casts the column and kills the index — match the literal type; (3) leading-wildcard `LIKE '%x%'` is not ordered by prefix and cannot use the index — use `FULLTEXT` or external search; (4) violation of the leading-column rule for a composite index — the query filters on a non-leading column; (5) the optimizer's statistics are stale — run `ANALYZE TABLE`; (6) the predicate's selectivity is so low (e.g. `WHERE is_active = 1` with 99% true) that a full scan is genuinely cheaper and the optimizer is correct; (7) `OR` across two indexed columns where index merge is not chosen — rewrite as `UNION`; (8) collation mismatch between joined columns; (9) the table is small enough that scanning it really is cheaper. Verify each hypothesis with `EXPLAIN` after the change.

### Q26: When should you NOT add an index?

Do not add an index when the query is rare (a monthly report doesn't need an index — use an analytics replica or a pre-aggregated table); when the column's selectivity is very low and the index wouldn't be covering (a boolean flag with 99% one value); when the table is small enough that full scans dominate; when the workload is write-heavy and the index has no matching read path; or when the index would not actually be used — verify with `EXPLAIN` after adding, and drop it if `type=ALL` persists. Every secondary index costs write amplification (each `INSERT`/`UPDATE`/`DELETE` updates every index B+Tree), storage, and buffer-pool memory. Track usage via `performance_schema.table_io_waits_summary_by_index_usage` or `sys.schema_unused_indexes` and drop dead indexes.

### Q27: How would you implement a work queue in MySQL without contention overhead?

Use a status table with an indexed `status` column and the `SKIP LOCKED` locking-read option (MySQL 8.0+):

```sql
-- Worker claims up to 10 jobs atomically:
START TRANSACTION;
SELECT id FROM jobs
WHERE status = 'PENDING'
ORDER BY id
LIMIT 10
FOR UPDATE SKIP LOCKED;        -- skips rows locked by other workers
UPDATE jobs SET status = 'RUNNING', claimed_at = NOW(3)
WHERE id IN (/* the ids above */);
COMMIT;
```

`SKIP LOCKED` lets multiple workers claim disjoint sets without blocking each other, and `FOR UPDATE` ensures the claim is atomic. Index `status` (or better, a composite `(status, id)`) so the `WHERE` is index-accessed — without an index the locking read would lock the whole table, defeating the design. Workers process jobs and either mark them `DONE` or `FAILED`; a reaper job requeues `RUNNING` jobs whose `claimed_at` is older than a timeout. This avoids the heavyweight centralized queue while still being transactional.

### Q28: Explain next-key, gap, and record locks, and why an unindexed `UPDATE` can lock the whole table.

A **record lock** locks a single index record. A **gap lock** locks the gap *between* two index records (or before the first / after the last), preventing inserts into that gap — it does not lock any actual row. A **next-key lock** is a record lock on a record R plus the gap lock on the interval before R; InnoDB uses next-key locks at REPEATABLE READ for predicates that match a range, which is how it prevents phantoms. At READ COMMITTED, gap locks are disabled (except for foreign-key/unique checks), reducing lock surface. Because all of these locks are taken *on the index*, a `UPDATE`/`SELECT FOR UPDATE` whose predicate is not covered by an index has no index entries to lock — InnoDB walks every row of the table and locks each one, producing an effective table lock with high deadlock risk. The rule: always index the columns you filter on in locking reads and updates.

### Q29: How would you shard a MySQL-backed application, and what problems does sharding introduce?

Pick a sharding key that the application can route on without a global lookup (typically `user_id` or `tenant_id`), and a strategy: range sharding (key ranges per shard — easy but risks hotspots on the newest range), hash sharding (`shard = hash(key) % N` — even distribution but expensive to reshard), or consistent hashing (a hash ring so adding a shard moves only ~1/N of keys — what middleware like Vitess implements). Generate IDs with a globally-unique, time-ordered scheme (UUID v7 or Snowflake) so inserts do not collide and remain roughly monotonic per shard. Sharding introduces: no cross-shard joins (denormalize or aggregate in the application); awkward cross-shard transactions (XA is avoided, so use sagas/outbox patterns); global aggregations require fan-out + merge; rebalancing is an operational project; and the schema must be designed shardable from the start (no implicit assumption of locality, no auto-increment shared sequences). Plan for resharding before you need it.

### Q30: What is INSTANT DDL, when can you use it, and when do you need `gh-ost` or `pt-online-schema-change`?

INSTANT DDL (MySQL 8.0.12+, expanded in 8.0.29 and 8.4) is a metadata-only schema change: no row copy, no exclusive lock, no replication lag — it updates the data dictionary and returns immediately. Examples include adding a column (at the end of the table in 8.0; anywhere in 8.4), dropping a column (8.0.29+), adding or dropping a virtual column, renaming a column, setting or dropping a column default, and extending a `VARCHAR` length in place. You request it with `ALTER TABLE ... ALGORITHM=INSTANT, LOCK=NONE` and verify with `SHOW CREATE TABLE` (which notes instant columns) or `information_schema`. When the change cannot be INSTANT or INPLACE — e.g. changing a column type, changing row format, modifying a primary key, restructuring a large table — use `gh-ost` (GitHub's binlog-streaming online schema changer, preferred at scale because it avoids trigger overhead) or `pt-online-schema-change` (Percona's trigger-based tool). Both build a shadow table, keep it in sync, copy rows in chunks, and rename at the end. Always test on a non-prod replica first; the legacy `COPY` algorithm should never be used on a hot table because it blocks writes for the entire copy.