# PostgreSQL Interview Questions

Model answers assume PostgreSQL 17/18 unless stated; version-specific behavior is called out. Try each question from memory before reading the answer — the reference files in this section cover everything here in more depth.

Grading guide:

- **Junior** (Q1–Q8) — mechanics and definitions; expected of anyone who has run Postgres in production.
- **Senior** (Q9–Q15) — trade-offs, diagnosis workflows, and cross-system comparisons.
- **Staff** (Q16–Q20) — incident-shaped scenarios where the answer is a decision process, not a fact.

## Junior

### Q1: How does an `UPDATE` physically work in PostgreSQL, and why does it matter?

An `UPDATE` never modifies a row in place. Postgres writes a complete new tuple (possibly on a different page), stamps the old tuple's `xmax` with the updating transaction's ID, and updates indexes to point at the new version — the old version stays in the heap as a dead tuple until VACUUM removes it. You can watch it happen:

```sql
SELECT ctid, xmin, xmax FROM accounts WHERE id = 1;  -- (0,1)
UPDATE accounts SET balance = balance + 10 WHERE id = 1;
SELECT ctid, xmin, xmax FROM accounts WHERE id = 1;  -- (0,7): a new physical tuple
```

This is Postgres's MVCC design: old versions live in the table itself, unlike InnoDB which keeps them in a separate undo log. It matters because update-heavy tables accumulate dead tuples (bloat), vacuum becomes a first-class operational concern, and every index may need a new entry per update — unless the update qualifies as HOT (no indexed column changed and the new version fits on the same page), in which case indexes are untouched. This one mechanism explains most Postgres-specific operational behavior.

### Q2: What is VACUUM and why does PostgreSQL need it when MySQL doesn't have an equivalent command?

VACUUM reclaims dead tuples left behind by updates and deletes, updates the visibility map (which enables index-only scans), freezes old transaction IDs (preventing wraparound), and updates the free space map. Postgres needs it because dead row versions live in the table heap; InnoDB stores old versions in the undo log and purges them with an internal background thread, so MySQL users never see an equivalent command. Autovacuum runs it automatically when dead tuples pass a threshold (~20% of the table by default, which is too lazy for large hot tables and commonly tuned down per-table). Plain `VACUUM` makes space reusable but does not shrink the file; `VACUUM FULL` does, but takes an exclusive lock — production table rewrites use `pg_repack` instead.

### Q3: Name the main PostgreSQL index types and one use case for each.

**B-tree** (default): equality, ranges, `ORDER BY`, prefix `LIKE` — the right choice ~95% of the time. **GIN**: inverted index for "element inside a value" — JSONB containment (`@>`), array membership, full-text search, and trigram indexes that make `LIKE '%substr%'` indexable. **GiST**: distance ordering (`ORDER BY location <-> point`) and exclusion constraints such as "no overlapping bookings". **BRIN**: tiny block-range min/max summaries for huge append-only tables queried by a physically-correlated column like `created_at` — can be 1000x smaller than a B-tree. **Hash**: equality only, marginally better than B-tree for pure `=`; crash-safe since PG 10. **SP-GiST**: space-partitioned structures for things like IP ranges. The differentiating answer names GIN-for-JSONB and BRIN-for-logs without prompting.

### Q4: What is the difference between `json` and `jsonb`, and how do you index a `jsonb` column?

`json` stores the raw text — original formatting, key order, and duplicate keys preserved — and is reparsed on every access. `jsonb` stores a parsed binary representation: slightly slower to write, much faster to query, deduplicated keys, and — decisively — indexable. Default to `jsonb`. Index it with GIN: `CREATE INDEX ... USING gin (payload)` supports containment (`@>`), key existence (`?`), and JSON-path operators; the `jsonb_path_ops` operator class is smaller and faster if you only need `@>`. For a single hot field, a B-tree expression index on `(payload->>'field')` is smaller and supports ranges and ordering, which GIN does not.

### Q5: What is a partial index and when would you use one?

A partial index only indexes rows matching a predicate:

```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';
```

If 99% of orders are completed and queries only chase live ones, the index stays tiny and hot in cache, and writes to non-matching rows skip it entirely. The planner uses it only when the query's `WHERE` clause implies the index predicate. Common uses: work-queue tables, soft deletes (`WHERE deleted_at IS NULL`), and partial *unique* indexes for conditional uniqueness:

```sql
-- "One active subscription per user" — declarative, race-free:
CREATE UNIQUE INDEX uq_active_sub ON subscriptions (user_id)
WHERE status = 'active';
```

MySQL cannot express either with a plain index — this is a reliable Postgres differentiator to volunteer.

### Q6: What are the PostgreSQL isolation levels and which is the default?

The default is **READ COMMITTED**: each statement sees a fresh snapshot of data committed before the statement began — two identical selects in one transaction can see different data. **REPEATABLE READ** takes one snapshot for the whole transaction; unlike the SQL standard's minimum, Postgres's implementation also prevents phantom reads, and concurrent write-write conflicts fail with serialization error `40001` rather than blocking until one wins. **SERIALIZABLE** adds Serializable Snapshot Isolation: it detects dangerous read-write dependency cycles among concurrent transactions and aborts one, giving true serializability without pervasive locking. `READ UNCOMMITTED` exists syntactically but behaves as READ COMMITTED — dirty reads are impossible in Postgres. The practical contract: above READ COMMITTED, the application must be prepared to retry aborted transactions.

### Q7: What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`, and when is `ANALYZE` dangerous?

`EXPLAIN` shows the planner's chosen plan with *estimated* costs and row counts without running the query. `EXPLAIN ANALYZE` **actually executes the query** and reports actual times, actual row counts per node, and loop counts — which is what you need, because most plan problems are estimate-vs-actual mismatches. The danger follows directly from "actually executes": `EXPLAIN ANALYZE` on an `INSERT`/`UPDATE`/`DELETE` performs the write. The safe pattern:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) UPDATE orders SET status = 'expired' WHERE ...;
ROLLBACK;
```

(Remembering from Q1 that even a rolled-back update leaves dead tuples behind.) Also add `BUFFERS` habitually — cache hit/read counts are usually the story; PG 18 includes it by default with `ANALYZE`.

### Q8: What is a UUID primary key's cost in PostgreSQL, and what changed with UUIDv7?

Less than in MySQL, but not free. Postgres tables are heaps, so a random UUIDv4 PK does not scatter the *table* the way it fragments InnoDB's clustered index — but the PK's B-tree index still receives inserts at random positions: poor cache locality (every page is equally hot), more page splits, a bigger index, and full-page-write amplification in WAL after checkpoints. **UUIDv7** (RFC 9562) puts a millisecond timestamp in the high bits, so index inserts are append-mostly — restoring the locality of a sequence while keeping global uniqueness and non-coordination; PG 18 ships `uuidv7()` built in (earlier versions: generate in the application or an extension). Remaining costs vs `bigint`: 16 bytes vs 8 in every index entry and FK, and slightly slower comparisons. Reasonable default in 2026: `uuidv7()` PKs when you need client-side generation or cross-service uniqueness; `bigint GENERATED ALWAYS AS IDENTITY` otherwise, with a separate public UUID if IDs must be non-enumerable.

## Senior

### Q9: PostgreSQL vs MySQL — when and why would you choose each?

Choose **PostgreSQL** for: richer SQL and types (JSONB with GIN indexing, arrays, range types, exclusion constraints, partial/expression indexes, transactional DDL); extensibility (PostGIS, pgvector, pg_stat_statements, TimescaleDB); stricter standards compliance and a stronger planner for complex queries; true SERIALIZABLE via SSI; and the modern managed ecosystem (Aurora PostgreSQL, AlloyDB, Neon, Supabase) — which is why it is the default for new backends in 2026. Choose **MySQL/InnoDB** when: the workload is update-heavy on hot rows (undo-log MVCC avoids Postgres's dead-tuple bloat and vacuum tuning); you want a clustered primary key for PK-range locality; you need thousands of direct connections without a pooler (threads vs processes); or the team/tooling ecosystem is MySQL-native (Vitess for sharding is more battle-proven than Postgres equivalents).

The mechanism-level differences worth naming explicitly:

| Dimension | PostgreSQL | MySQL/InnoDB |
|-----------|------------|--------------|
| Old row versions | In the heap; VACUUM reclaims | In undo log; purge thread reclaims |
| Table storage | Heap; all indexes point to TID | Clustered by PK; secondary indexes point to PK |
| Connections | Process each; pooler required at scale | Thread each; tolerates thousands |
| Phantom prevention | Snapshots (RR) / SSI (SERIALIZABLE) | Next-key (gap) locks |
| DDL | Transactional | Atomic (8.0) but not transactional |
| Replication default | Physical WAL streaming | Logical binlog (row-based) |

Honest framing: both are excellent general-purpose OLTP databases; the decision is usually operational maturity and team experience, not features. The senior answer names these concrete mechanisms rather than vibes.

### Q10: Walk me through how you would diagnose a slow query in production.

First find it: `pg_stat_statements` ordered by `total_exec_time` for aggregate offenders, or `log_min_duration_statement` / `auto_explain` for outliers. Then run `EXPLAIN (ANALYZE, BUFFERS)` on production-like data and read it in this order: (1) compare estimated vs actual rows at each node — a 10x+ mismatch means the planner chose a strategy for data that doesn't exist, so fix statistics before anything else (`ANALYZE`, raise the column's statistics target, or `CREATE STATISTICS` for correlated columns); (2) find the node where actual time × loops concentrates — a 0.01 ms inner index scan with `loops=2M` is the cost; (3) read `Buffers` — high `read` counts mean the working set doesn't fit cache; (4) look for spills (`Sort Method: external merge`, hash batches) indicating `work_mem` too small for that node. Then apply the smallest fix: the right index (partial/expression/covering `INCLUDE`), a rewrite, or a schema change — always via `CREATE INDEX CONCURRENTLY` — and verify the new plan plus check `pg_stat_user_indexes` later to confirm it's used.

### Q11: Why does PostgreSQL need an external connection pooler like PgBouncer, and what breaks under transaction pooling?

Each Postgres connection is a forked OS process holding several MB and contending on shared memory; performance degrades well before the connection counts a thread-per-connection server like MySQL tolerates, and useful active concurrency is roughly `cores × 2` — so a fleet of app instances each holding a pool would exhaust the server with mostly-idle connections. PgBouncer multiplexes thousands of client connections onto a small server pool. In **transaction mode** (the production default), a server connection is borrowed per transaction, so anything session-scoped breaks: `SET` (use `SET LOCAL` inside the transaction), advisory session locks, temp tables, `LISTEN/NOTIFY`, and historically named prepared statements — though PgBouncer 1.21+ can track and replay those via `max_prepared_statements`. **Session mode** is fully compatible but saves little; **statement mode** forbids multi-statement transactions. A complete answer mentions that RLS-style `SET app.tenant_id` must be `SET LOCAL` under transaction pooling, or tenants will bleed across requests.

### Q12: What is an index-only scan and why might it still hit the heap?

If an index contains every column a query needs (naturally, or via `INCLUDE` payload columns), Postgres can answer from the index alone — an **index-only scan**. But index entries carry no visibility information (dead and live tuples both have entries), so for each index entry Postgres checks the **visibility map**: if the tuple's heap page is marked all-visible, no heap access is needed; otherwise it must fetch the heap page to check `xmin`/`xmax` anyway. `EXPLAIN ANALYZE` reports this as `Heap Fetches`:

```
Index Only Scan using idx_orders_user_created on orders
  (actual time=0.03..48.1 rows=52000 loops=1)
  Heap Fetches: 51344        <- vacuum is behind; nearly every row hit the heap anyway
```

A table with lagging vacuum has few all-visible pages, so a nominally index-only scan silently degrades to index-scan cost. That coupling — index-only scan performance depends on vacuum recency — is the interview point, and another reason autovacuum tuning is a read-performance topic, not just a storage one.

### Q13: Explain streaming vs logical replication and when you'd use each.

**Streaming (physical) replication** ships raw WAL to replicas that replay it into byte-identical, read-only copies of the entire cluster — the workhorse for HA failover and read replicas. It is asynchronous by default (failover can lose the last commits); `synchronous_standby_names` makes commits wait for standby acknowledgment, with per-transaction control via `synchronous_commit` (up to `remote_apply` for read-your-writes on the standby). **Logical replication** decodes WAL into row-level changes published per-table (`CREATE PUBLICATION`/`SUBSCRIPTION`); the subscriber is a normal writable database, can be a different major version, and can subscribe to a subset of tables. Use it for near-zero-downtime major upgrades, selective data distribution, consolidating databases, and CDC (Debezium reads a logical slot to feed Kafka — the standard "keep Elasticsearch in sync" answer). Limitations to name: no DDL replication, no sequences, `UPDATE`/`DELETE` require a replica identity. Both use **replication slots**, and the classic incident is a slot for a dead consumer retaining WAL until the primary's disk fills — cap it with `max_slot_wal_keep_size`. PG 17's failover slots let logical consumers survive physical failover.

### Q14: How would you build a job queue in PostgreSQL? When is that the wrong choice?

A `jobs` table with a partial index on pending rows, claimed atomically:

```sql
UPDATE jobs SET status = 'running', claimed_by = $1
WHERE id = (
  SELECT id FROM jobs WHERE status = 'pending'
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

`SKIP LOCKED` lets N workers claim disjoint rows without lock contention, and transactional claiming gives exactly-once processing (crash before commit returns the job to the pool). Add `LISTEN/NOTIFY` so workers sleep until a writer commits a `NOTIFY` instead of polling; note NOTIFY fires only on commit (transactional), delivers at-most-once to connected listeners, and requires a dedicated connection incompatible with PgBouncer transaction pooling. The killer feature is enqueueing a job *in the same transaction* as the business change — no dual-write problem. This pattern (used by Oban, Graphile Worker, pgmq, Solid Queue) is solid to thousands of jobs/sec. It's the wrong choice when you need replay/retention, fan-out to multiple consumer groups, cross-service decoupling, or very high throughput — a hot insert+delete queue table is the pathological vacuum workload, generating dead tuples exactly as fast as it processes jobs. Then: Kafka for streams, SQS/RabbitMQ for tasks.

### Q15: A query is fast in psql but intermittently slow from the application. What do you suspect?

The classic culprit is the **generic-plan switch for prepared statements**. Drivers use the extended protocol; for the first five executions Postgres plans with actual parameter values (custom plans), then switches to a cached generic plan if it doesn't estimate worse — and on skewed data the generic plan can be catastrophically wrong for some parameter values (e.g. `status = 'pending'` matching 0.1% of rows vs `'done'` matching 90%: the generic plan must assume average selectivity). Diagnose with `EXPLAIN (GENERIC_PLAN)` (PG 16+) or by setting `plan_cache_mode = force_custom_plan`. Other suspects worth listing: parameter-value skew generally (fast for most users, slow for the one with 10M rows), connection pool saturation (queue wait measured as query time), lock waits from a concurrent writer or DDL (check `pg_locks`/`pg_stat_activity`), and cold cache after failover. The senior habit: reproduce with the *same protocol* the app uses, not just interactive SQL — psql's simple query protocol plans with literal values every time, hiding the whole class of problem.

## Staff

### Q16: What is transaction ID wraparound, what actually causes wraparound incidents, and how do you prevent them?

XIDs are 32-bit and compared circularly, so from any point ~2 billion XIDs are "past" and ~2 billion "future". A tuple whose `xmin` fell ~2 billion transactions behind would appear to be from the future — committed data becoming invisible. VACUUM prevents this by **freezing** old tuples (marking them visible-to-all, exempt from XID comparison); `autovacuum_freeze_max_age` (200M) forces an aggressive anti-wraparound vacuum per table, and if the cluster's oldest unfrozen XID nears the limit, Postgres stops accepting write transactions — a full outage (Sentry's 2015 postmortem is the canonical reference). The staff-level insight is that wraparound is a *symptom*: the incident cause is whatever prevented freezing from keeping up — a weeks-old transaction or `idle in transaction` session pinning the horizon, an abandoned replication slot, autovacuum disabled "temporarily", vacuum starved by cost limits on a huge table, or an orphaned prepared transaction. Prevention: alert on `age(datfrozenxid)` (e.g. at 500M), alert on oldest transaction/slot age, set `idle_in_transaction_session_timeout`, and ensure vacuum on the largest tables completes faster than XID consumption. On billions-of-transactions-per-day systems, anti-wraparound vacuums of TB-scale tables must be capacity-planned like any other batch workload.

### Q17: Your multi-TB table is 60% bloat and vacuum "isn't working". Diagnose and fix — without downtime.

First, verify what "isn't working" means: is autovacuum not *triggering* (default 20% scale factor means 200M dead rows on a 1B-row table before it fires — tune `autovacuum_vacuum_scale_factor` down per-table), not *finishing* (cost-limit throttling; check `pg_stat_progress_vacuum`, raise `autovacuum_vacuum_cost_limit`), or finishing but **unable to remove tuples**? The last is the big one — check all three horizon-pinners:

```sql
SELECT pid, state, xact_start FROM pg_stat_activity
WHERE xact_start < now() - interval '1 hour';          -- ancient transactions

SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;                              -- stale slots

SELECT * FROM pg_prepared_xacts;                        -- orphaned 2PC transactions
```

Any of these pins the removal horizon cluster-wide, and no amount of vacuuming helps until it's cleared (vacuum's log line "N dead tuples cannot be removed yet" confirms it). Having fixed the cause, plain vacuum only makes the 60% reusable — it won't shrink the file. To reclaim space online, use **`pg_repack`** (rebuilds the table concurrently via triggers plus a brief lock at swap) — never `VACUUM FULL` on a live multi-TB table (ACCESS EXCLUSIVE for the whole rewrite); `REINDEX CONCURRENTLY` handles the index side. Then prevent recurrence: per-table autovacuum settings, `idle_in_transaction_session_timeout`, slot monitoring, and — if the bloat came from bulk deletes — convert the table to partitioning so retention becomes `DROP PARTITION`. PG 17's vacuum memory rework (radix-tree TID store, no 1 GB cap) materially reduces multi-pass vacuums on tables this size, which is a legitimate reason to prioritize the upgrade.

### Q18: Design multi-tenant isolation for a B2B SaaS on PostgreSQL. Compare the options.

Three models. **Database-per-tenant**: strongest isolation, per-tenant restore/tuning; operationally untenable past a few hundred tenants (migrations × N, connection pools × N, no cross-tenant queries). **Schema-per-tenant**: middle ground; still N× migrations, catalog bloat at thousands of schemas, and connection poolers struggle with per-schema `search_path` (a PgBouncer transaction-mode hazard). **Shared tables + `tenant_id`** — the default at scale — with **row-level security** as the enforcement layer:

```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id')::bigint);
-- App, per request, inside the transaction:
SET LOCAL app.tenant_id = '42';
```

RLS gives database-enforced defense in depth against the "forgot the WHERE tenant_id" bug class. Implementation details that show real experience: connect as a **non-owner** role (owners bypass RLS unless `FORCE ROW LEVEL SECURITY`); make every index lead with `tenant_id` so policy predicates are index-friendly; `SET LOCAL` (not `SET`) under transaction pooling; watch for planner limitations with policies on complex joins; and plan the whale-tenant escape hatch — LIST-partition by tenant or extract the largest tenants to dedicated instances when one tenant dominates. Mention Citus if the honest answer to projected scale is distributed Postgres sharded by `tenant_id`.

### Q19: You're adding semantic search over 50M documents. Does pgvector suffice, or do you need a dedicated vector database?

Frame it as an architecture decision, not a benchmark. **pgvector wins on integration**: embeddings live next to relational data, so "vector similarity AND tenant_id = X AND status = 'published'" is one indexed query with transactional consistency — no sync pipeline, no second system to page on. At 50M × 1536-dim float32, vectors alone are ~300 GB plus an HNSW index; workable on a large instance, better with `halfvec` (halves memory) or binary quantization with rescoring. Know the index trade-off: **HNSW** (better recall/latency, expensive builds, memory-resident graph) vs **IVFFlat** (cheaper builds, recall depends on probes, needs representative data at creation); both are approximate — recall vs speed is tunable, not free. Sharpest technical issue: **filtered ANN search** — a highly selective metadata filter combined with HNSW can degrade recall or force scanning far more of the graph (pgvector's iterative index scans mitigate this; partial indexes per hot filter also work). A dedicated vector DB (or dedicated search infra) earns its operational cost at billions of vectors, very high QPS where you'd rather scale search independently of OLTP, or when you need its extras natively. Staff-level move: start with pgvector inside the existing Postgres footprint, define the exit criteria (recall SLO, p99 latency, index build windows), and keep the embedding pipeline storage-agnostic so migration is a target change, not a rewrite.

### Q20: Zero-downtime schema migrations on a hot PostgreSQL table — what's dangerous and what's the playbook?

Every DDL takes some lock; the danger is `ACCESS EXCLUSIVE` held long, *or* briefly but queued behind a long-running query — everything behind it in the lock queue blocks, so **always set `lock_timeout`** (e.g. 2s) and retry, turning a potential outage into a bounced migration.

Safe by design: `ADD COLUMN` (nullable, or with a constant default — PG 11+ stores the default in the catalog, no rewrite), `CREATE INDEX CONCURRENTLY` / `DROP INDEX CONCURRENTLY` (never plain `CREATE INDEX` on a live table; handle the `INVALID`-index-on-failure case). Dangerous and needing decomposition:

- **Changing a column type** — full table rewrite under exclusive lock. Instead: add new column, dual-write, backfill in batches, swap.
- **Adding `NOT NULL`** — PG 12+:

```sql
SET lock_timeout = '2s';
ALTER TABLE orders ADD CONSTRAINT orders_user_nn
  CHECK (user_id IS NOT NULL) NOT VALID;          -- instant
ALTER TABLE orders VALIDATE CONSTRAINT orders_user_nn;  -- light lock, full scan
ALTER TABLE orders ALTER COLUMN user_id SET NOT NULL;   -- cheap: uses the constraint as proof
```

- **Adding a foreign key** — same pattern: `NOT VALID` first, `VALIDATE CONSTRAINT` separately.
- **Backfills** — batch with pauses; a single 100M-row `UPDATE` doubles the table's live+dead tuples and hands autovacuum a crisis (file 01 §8).

Postgres-specific advantage worth stating: **DDL is transactional**, so a multi-statement migration rolls back atomically — with the caveat that `CONCURRENTLY` operations cannot run inside a transaction. Tooling that encodes this playbook: `gh-ost`-equivalents aren't needed here — pgroll, Reshape, or strong-migrations-style linters, plus expand/contract as the application-level contract.
