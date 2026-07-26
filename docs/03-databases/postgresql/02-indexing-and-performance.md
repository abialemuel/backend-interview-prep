# Indexing and Performance

PostgreSQL's headline advantage over most relational databases is the breadth of its index types and the sophistication of its cost-based planner. This file covers when each index type wins, how to read `EXPLAIN (ANALYZE, BUFFERS)` like the planner does, how statistics drive plan choice, and the two infrastructure topics every Postgres-at-scale interview touches: connection pooling and prepared statements.

## 1. Index types: pick the right tool

| Type | Structure | Best for | Watch out for |
|------|-----------|----------|---------------|
| **B-tree** (default) | Balanced tree, ordered | Equality, ranges, `ORDER BY`, `LIKE 'prefix%'`, uniqueness | The right default 95% of the time |
| **Hash** | Hash buckets | Equality only, slightly smaller/faster than B-tree for `=` | No ranges, no ordering, no uniqueness; WAL-logged and crash-safe since PG 10 (before that, unusable) |
| **GIN** | Inverted index: one entry per element → posting list of rows | JSONB containment (`@>`), array membership, full-text search (`tsvector`), trigram `LIKE '%x%'` | Slow to update (mitigated by `fastupdate` pending list); big; no ordering |
| **GiST** | Generalized balanced tree over "does this subtree possibly match" | Geometric/range types (`&&` overlap), nearest-neighbor `ORDER BY <->`, exclusion constraints | Lossy — may need recheck; usually slower than GIN for FTS lookups but cheaper to update |
| **SP-GiST** | Space-partitioned tree (quadtree, radix) | Non-balanced partitionable data: IP ranges (`inet`), points, text prefixes | Niche; know it exists |
| **BRIN** | Per-block-range min/max summaries | Huge append-only tables where a column correlates with physical order (`created_at` on an insert-only log) | Tiny (MBs for TB tables) but useless if data is not physically correlated |

Rules of thumb worth saying out loud in an interview:

- **B-tree** unless you can name the reason it isn't.
- **GIN** for "is X inside this column" questions — JSONB, arrays, full-text.
- **BRIN** when the table is append-only, huge, and queried by time range: a BRIN index on `created_at` can be 1000x smaller than a B-tree and nearly as effective, because it only stores min/max per 128-page block range.
- **GiST** when you need distance ordering (`ORDER BY location <-> point`) or exclusion constraints (`EXCLUDE USING gist (room WITH =, during WITH &&)` — the canonical "no double-booking" constraint).

```sql
-- GIN on JSONB:
CREATE INDEX idx_events_payload ON events USING gin (payload jsonb_path_ops);
SELECT * FROM events WHERE payload @> '{"type": "refund"}';

-- BRIN on an append-only log:
CREATE INDEX idx_log_created ON api_log USING brin (created_at);

-- Trigram GIN makes %substring% LIKE indexable (impossible with B-tree):
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING gin (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%smith%';   -- uses the index
```

!!! note
    PostgreSQL 18 added **skip scan** for multicolumn B-tree indexes: an index on `(tenant_id, created_at)` can now serve a query filtering only `created_at` by iterating the distinct `tenant_id` values — provided the leading column is low-cardinality. Before 18 (and still, for high-cardinality leading columns), the MySQL-style leading-column rule applies unchanged: design composite indexes as *equality columns first, range column last*.

## 2. Partial and expression indexes

Two Postgres features MySQL only partially matches, and reliable interview differentiators.

**Partial index** — index only the rows matching a predicate:

```sql
-- 99% of orders are 'done'; queries only ever chase the live ones.
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status IN ('pending', 'processing');
```

The index is tiny, stays hot in cache, and skips write amplification for rows that don't match. The query's `WHERE` clause must *imply* the index predicate for the planner to use it. Classic uses: soft-delete (`WHERE deleted_at IS NULL`), work queues, "only index non-null FKs". A **partial unique index** enforces conditional uniqueness — e.g. one active subscription per user:

```sql
CREATE UNIQUE INDEX uq_active_sub ON subscriptions (user_id) WHERE status = 'active';
```

**Expression index** — index the result of an expression, used when the query filters on `f(col)`:

```sql
CREATE INDEX idx_users_email_lower ON users (lower(email));
SELECT * FROM users WHERE lower(email) = lower($1);   -- index used
```

The query must use the *same expression*. Expression indexes also get their own statistics, which sometimes fixes bad row estimates on derived values. (Alternatively use the `citext` type for case-insensitive columns.)

**Covering indexes**: `CREATE INDEX ... (a, b) INCLUDE (c)` adds payload columns to the leaf level without making them part of the key — enabling index-only scans for queries that return `c`. Remember from file 01: an index-only scan still checks the **visibility map**, so a badly vacuumed table turns "index-only" into index + heap fetches (`EXPLAIN ANALYZE` shows `Heap Fetches: N` — high numbers mean vacuum is behind).

## 3. Reading EXPLAIN (ANALYZE, BUFFERS)

`EXPLAIN` shows the plan and estimates. `EXPLAIN ANALYZE` runs the query and shows actuals. Always add `BUFFERS` (PG 18 includes it in `ANALYZE` by default) — cache behavior is usually the story.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, u.email
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending' AND o.created_at > now() - interval '1 day';
```

```
Nested Loop  (cost=0.85..1245.30 rows=180 width=48)
             (actual time=0.04..3.20 rows=142 loops=1)
  Buffers: shared hit=890 read=12
  -> Index Scan using idx_orders_pending on orders o
       (cost=0.42..320.10 rows=180 width=16)
       (actual time=0.02..0.90 rows=142 loops=1)
       Index Cond: (created_at > (now() - '1 day'::interval))
  -> Index Scan using users_pkey on users u
       (cost=0.43..5.10 rows=1 width=40)
       (actual time=0.01..0.01 rows=1 loops=142)
       Index Cond: (id = o.user_id)
Planning Time: 0.25 ms
Execution Time: 3.45 ms
```

How to read it, in the order that finds problems fastest:

1. **Compare estimated `rows` to `actual rows` at every node.** A large mismatch (10x+) is the root cause of most bad plans — the planner chose a strategy for a data shape that doesn't exist. Fix the estimate (statistics, §5) before touching indexes.
2. **`actual time` × `loops`** is a node's real total cost. An inner index scan at 0.01 ms looks free until `loops=2,000,000`.
3. **`Buffers`**: `shared hit` = pages from cache, `read` = from disk. A "fast" query doing 50k reads is a cold-cache incident waiting to happen.
4. **Scan types**: `Seq Scan` (fine for small tables or low-selectivity predicates — not automatically a bug), `Index Scan` (index → heap per row), `Index Only Scan` (heap skipped where the VM allows; check `Heap Fetches`), `Bitmap Index Scan → Bitmap Heap Scan` (collect matching TIDs, sort by page, then read heap sequentially — the planner's choice for medium selectivity, and how multiple indexes get combined with `BitmapAnd`/`BitmapOr`).
5. **Join types**: `Nested Loop` (best when the outer side is small and the inner side is indexed), `Hash Join` (build a hash table from the smaller side; the default for big unsorted inputs), `Merge Join` (both sides sorted; wins for large pre-sorted inputs). A nested loop against a *mis-estimated* large outer side is the classic catastrophic plan.
6. **Memory spills**: `Sort Method: external merge  Disk: 84MB` or a hash join spilling batches means `work_mem` was too small *for that node* (each node of each query can use up to `work_mem` — hence the caution raising it globally).

## 4. The planner

Postgres has a pure **cost-based optimizer** — there are no index hints in core (the `pg_hint_plan` extension exists; many teams ban it). The planner enumerates plans (exhaustively up to `geqo_threshold` tables, genetic search beyond) and prices them with cost parameters:

- `seq_page_cost = 1.0`, `random_page_cost = 4.0` — **on SSDs/cloud storage, lower `random_page_cost` to ~1.1**; the 4.0 default models spinning disks and biases the planner toward seq scans.
- `effective_cache_size` — planner's guess of total cache (shared_buffers + OS cache); set to ~50–75% of RAM. It affects plan choice only, not allocation.
- `work_mem` — per-sort/per-hash budget, as above.

When the plan is wrong, the debugging ladder is: check row estimates → fix statistics → adjust cost parameters → rewrite the query — hints are not on the ladder. `SET enable_seqscan = off` is a *diagnostic* (to see what the alternative plan would cost), not a fix.

## 5. Statistics

The planner's row estimates come from per-column statistics collected by `ANALYZE` (autovacuum runs it automatically): NULL fraction, `n_distinct`, most-common values (MCVs) with frequencies, and a histogram — based on a sample whose size is `default_statistics_target` (100) × 300 rows.

When estimates go wrong:

```sql
-- More detailed stats for a skewed hot column:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;

-- Correlated columns: planner assumes independence.
-- WHERE city = 'Dubai' AND country = 'AE' -> estimate multiplies selectivities,
-- wildly underestimating. Fix with extended statistics (PG 10+):
CREATE STATISTICS st_orders_city_country (dependencies, ndistinct, mcv)
  ON city, country FROM addresses;
ANALYZE addresses;
```

Other estimate-killers: filtering on an expression (`WHERE a + b > 10` — no stats; use an expression index or generated column), stale stats after bulk loads (run `ANALYZE` manually), and joins across skewed distributions. PostgreSQL 18 finally **preserves planner statistics across `pg_upgrade`**, closing the old trap of a major-version upgrade going live with empty stats and collapsing under bad plans until `ANALYZE` finished.

## 6. Connection pooling and PgBouncer

Because each connection is a process (file 01, §1), and because cloud Postgres caps connections (often a few thousand at most), any service with more than a handful of instances needs pooling. Two layers, usually combined:

1. **Application-side pool** (HikariCP, `pgx/pgxpool`, etc.) — reuses connections within one process.
2. **PgBouncer** — a lightweight proxy multiplexing many client connections onto few server connections.

PgBouncer's pooling modes are the interview question:

| Mode | Server connection is held for | Compatible with |
|------|-------------------------------|-----------------|
| `session` | The client's whole session | Everything, but saves the least |
| `transaction` | One transaction, then returned to pool | **The production default.** Breaks session state: `SET` (non-local), advisory session locks, `LISTEN`, and named prepared statements* |
| `statement` | One statement | Only autocommit workloads; forbids multi-statement transactions |

*PgBouncer 1.21+ (2023) added `max_prepared_statements`, which tracks and replays named prepared statements across server connections — removing the single biggest historical pain of transaction mode. Still, anything session-scoped (`SET search_path`, `LISTEN/NOTIFY`, advisory locks held across transactions, temp tables) is unsafe under transaction pooling; use `SET LOCAL` inside the transaction instead.

Sizing intuition worth quoting: optimal *active* server connections ≈ `(cores × 2) + effective_spindles` — for a 16-core box that's ~40, not 4000. The pool's job is to queue the excess. Alternatives in the same space: Odyssey, AWS RDS Proxy, Supavisor; Postgres 18's async I/O doesn't change the math — connections are still processes.

## 7. Prepared statements

A prepared statement is parsed and analyzed once, then executed repeatedly with parameters:

```sql
PREPARE get_user (bigint) AS SELECT * FROM users WHERE id = $1;
EXECUTE get_user(42);
```

Drivers do this via the extended query protocol (parse/bind/execute) rather than SQL `PREPARE`. Benefits: skip repeated parse/plan cost, and **parameterization prevents SQL injection** structurally.

The planner subtlety interviewers like: for the first 5 executions Postgres builds a **custom plan** per parameter value; from the 6th, if the average custom-plan cost isn't beating the **generic plan** (planned once, parameter-value-agnostic), it switches to the generic plan. On skewed data this can flip a query from index scan to seq scan at execution #6 — the classic "query is fast in psql, slow in production, and only sometimes" mystery. Diagnose with `plan_cache_mode = force_custom_plan` (or per-statement in PG's `EXPLAIN (GENERIC_PLAN)` in 16+).

Prepared statements are per-connection state, hence the PgBouncer interaction above.

## 8. The slow-query workflow

1. **Find candidates** with `pg_stat_statements` (see `03-advanced-features.md`) — rank by `total_exec_time`, look at `mean_exec_time` and `rows`; and/or set `log_min_duration_statement = 100ms`. `auto_explain` logs the actual plan of anything slow — invaluable for queries you cannot reproduce interactively:

```ini
# postgresql.conf
session_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on          # actual rows/times (adds timing overhead)
auto_explain.log_buffers = on
```

   System-level I/O context comes from `pg_stat_io` (PG 16+) — per-backend-type reads, writes, and evictions, which tells you whether slow queries are victims of a cache-thrashing neighbor rather than culprits.
2. `EXPLAIN (ANALYZE, BUFFERS)` the top offenders on production-like data.
3. Check row-estimate accuracy first; fix stats if off.
4. Apply the smallest fix: the right index type (partial? expression? GIN?), a covering `INCLUDE`, a query rewrite, then schema changes.
5. Confirm the index is actually used and retire dead ones:

```sql
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes ORDER BY idx_scan ASC LIMIT 15;
```

!!! warning
    Always create and drop indexes on live tables with `CREATE INDEX CONCURRENTLY` / `DROP INDEX CONCURRENTLY`. Plain `CREATE INDEX` blocks all writes to the table for the build. `CONCURRENTLY` runs without blocking writes but cannot run inside a transaction and, if it fails, leaves an `INVALID` index you must drop and retry.

## 9. Quick reference

- B-tree by default; GIN for containment (JSONB/array/FTS/trigram); BRIN for huge, physically-ordered, append-only; GiST for distance and exclusion constraints.
- Partial indexes for hot subsets and conditional uniqueness; expression indexes for `f(col)` predicates.
- In `EXPLAIN ANALYZE`: estimates-vs-actuals first, then `loops`, then `Buffers`, then spills.
- Bad plans are usually bad estimates — fix statistics (`SET STATISTICS`, `CREATE STATISTICS`) before reaching for workarounds; core has no hints.
- `random_page_cost ≈ 1.1` on SSDs; `work_mem` is per-node, not per-query.
- PgBouncer in transaction mode is the production default; know what session state it breaks.
- Prepared statements: generic-plan switch after 5 executions explains many "intermittently slow" incidents.
- `CREATE INDEX CONCURRENTLY`, always, on live tables.
