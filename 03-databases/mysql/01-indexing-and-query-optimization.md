# Indexing and Query Optimization

Indexes are the single most impactful lever for MySQL performance and the most common interview topic. This file covers the InnoDB index model end-to-end, how the optimizer chooses indexes, how to read `EXPLAIN`, and the practical pitfalls (functions in `WHERE`, leading wildcards, `OFFSET` pagination, etc.).

## 1. Why B+Tree for databases

InnoDB (and effectively all relational engines) uses a **B+Tree** variant for its secondary and primary key indexes. A B+Tree is a balanced multi-way tree with two properties that matter for database workloads:

1. **All data lives in leaf nodes.** Internal nodes contain only routing keys. This keeps internal nodes small → high fan-out → shallow trees. A typical InnoDB B+Tree is 3–4 levels deep for tens of millions of rows, meaning a point lookup takes 3–4 page reads (and the top levels are cached in the buffer pool).
2. **Leaf nodes are linked in a doubly-linked list** in key order. This makes **range scans** cheap: find the start leaf, then walk the linked list forward without re-traversing the tree.

Contrast with a plain **B-Tree**: a B-Tree stores values in *every* node, not just leaves. That reduces fan-out (internal nodes are bigger) and makes range scans more expensive (you must re-traverse the tree). So B+Tree optimizes for the dominant database access pattern: ordered scans over ranges.

Contrast with a **hash index**: O(1) point lookups, but *no* ordering → no range scans, no `ORDER BY` optimization, no `LIKE 'prefix%'`. InnoDB has an internal adaptive hash index on top of B+Tree for hot point lookups, but you do not manage it directly. Memory engine used to default to hash indexes; that is rare in production.

## 2. Clustered index vs secondary index (InnoDB-specific)

This is the most important InnoDB-specific concept to nail in an interview.

### Clustered index

InnoDB stores the **table data** as a B+Tree keyed by the primary key. The leaf nodes of this tree contain the **full row data**. This is called the **clustered index** (also "index-organized table"). If you do not declare a primary key, InnoDB picks the first unique non-null index, or synthesizes a hidden `ROWID`.

Implications:

- Rows are physically ordered by primary key *within a page* (not strictly across pages, but logically).
- Point lookups by PK are a single tree descent.
- Secondary indexes point to PK, not to the physical row location (see below).

### Secondary index

A secondary index (any index that is not the PK) is also a B+Tree, but its **leaf nodes store the indexed column(s) plus the primary key value** — not the full row, and not a physical row pointer.

To read a non-indexed column from a row found via a secondary index, InnoDB must do a second lookup: descend the clustered index using the PK stored in the secondary-index leaf. This is called a **bookmark lookup** (also "table lookup", "PK lookup"). Each bookmark lookup is one B+Tree descent → potentially random I/O.

```sql
CREATE TABLE users (
  id           BIGINT UNSIGNED PRIMARY KEY,
  email        VARCHAR(255) NOT NULL,
  country      CHAR(2)       NOT NULL,
  created_at   DATETIME(0)   NOT NULL,
  UNIQUE KEY uk_email (email),
  KEY idx_country_created (country, created_at)
) ENGINE=InnoDB;

-- Uses idx_country_created; finds PK, then does bookmark lookup for id? No — id is PK, in the index.
-- But to get email InnoDB must look up the clustered index.
EXPLAIN SELECT email, created_at FROM users WHERE country = 'DE';
-- type=ref, key=idx_country_created, Extra=Using index condition (need bookmark lookup for email)
```

### Covering index

If a query reads **only** columns that are fully contained in a secondary index (including the PK, which is always present), InnoDB does not need the bookmark lookup — it reads everything from the index B+Tree. This is called a **covering index** and shows up in `EXPLAIN` as `Using index` in the `Extra` column.

```sql
-- Covers: country, created_at returned, and PK id available in the index leaf.
EXPLAIN SELECT id, country, created_at FROM users WHERE country = 'DE';
-- Extra: Using index  → covering, no bookmark lookup, no table access.
```

A common optimization technique: extend a composite index so common queries are covered, e.g. `(country, created_at) → (country, created_at, email)` if a hot query returns email by country.

## 3. EXPLAIN: how to read it

`EXPLAIN` shows the optimizer's plan for a `SELECT` (MySQL 8.0+ also supports `EXPLAIN ANALYZE` for actual runtime stats with row counts per join).

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'PAID';
```
```
+----+-------+--------+-----------+---------+-------+------+-----------------------+
| id | table | type   | key       | key_len | ref   | rows | Extra                 |
+----+-------+--------+-----------+---------+-------+------+-----------------------+
|  1 | orders| ref    | idx_user  | 12      | const |  150 | Using index condition |
+----+-------+--------+-----------+---------+-------+------+-----------------------+
```

Key columns to read carefully:

- **`type`** — the access method. Rough quality, best → worst:
  - `const` — PK or unique index equals a constant. One row. Best possible.
  - `eq_ref` — join where each row from outer table matches exactly one row via unique/PK index. Best for joins.
  - `ref` — non-unique index equality; multiple rows possible.
  - `range` — index range scan (`>`, `BETWEEN`, `IN`, `LIKE 'x%'`).
  - `index` — scans an entire index (no filter, just cheaper than scanning the table).
  - `ALL` — full table scan. Usually a red flag unless the table is tiny or most rows qualify.
- **`key`** — the index actually chosen. If `NULL` the optimizer gave up on indexes.
- **`key_len`** — how many bytes of the index are used. Useful for verifying a composite index uses all expected columns.
- **`rows`** — estimated rows the engine will read to satisfy this step. Multiply across join steps for the rough cost.
- **`Extra`** — additional info. Important values:
  - `Using index` — covering index, no bookmark lookup. The presence of this is usually a win.
  - `Using where` — the storage engine returns rows and a post-filter in the server layer prunes them. Normal, but combined with `ALL` it indicates inefficiency.
  - `Using filesort` — the sort cannot use the index order; an extra sort pass is needed. Often fixable by adding the right ordering columns to an index.
  - `Using temporary` — a temp table is needed (e.g. for `GROUP BY`/`DISTINCT` without a covering index).
  - `Using index condition` — **Index Condition Pushdown (ICP)**: filter predicates on indexed columns are evaluated in the storage engine against the index entry, avoiding a bookmark lookup for rows the predicate rejects. Big win in 5.6+.

Use `EXPLAIN FORMAT=TREE` (8.0+) and `EXPLAIN ANALYZE` (8.0.18+) to see join order and actual per-node costs:

```sql
EXPLAIN ANALYZE
SELECT o.id, u.email
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.status = 'PAID' AND u.country = 'DE';
```

## 4. When an index is used vs ignored

The optimizer chooses an index when the estimated cost is lower than alternatives. Indexes get *ignored* in well-defined cases:

### a. Leading-column rule for composite indexes

An index `(a, b, c)` can be used for queries that filter on a prefix of its columns: `(a)`, `(a,b)`, `(a,b,c)`. A query filtering only `b` and `c` **cannot** use the index (the B+Tree is ordered by `a` first).

```sql
-- Index: idx_a_b_c (a, b, c)
SELECT * FROM t WHERE a = 1 AND b = 2 AND c = 3;  -- uses all 3 columns
SELECT * FROM t WHERE a = 1 AND b = 2;            -- uses a, b
SELECT * FROM t WHERE a = 1;                       -- uses a
SELECT * FROM t WHERE b = 2;                       -- cannot use idx_a_b_c
SELECT * FROM t WHERE a = 1 AND c = 3;            -- uses a only (c skipped because b missing)
```

### b. Range conditions stop further column use

Once a column is used with a range predicate (`<`, `>`, `BETWEEN`, `IN`, `LIKE 'x%'`), subsequent columns in the composite index cannot be used to *narrow further* (still good for covering, bad for further filtering).

```sql
-- idx last_first_dob (last_name, first_name, dob)
-- Equality on last_name, range on first_name → dob is NOT used as a filter via the index.
SELECT * FROM people
WHERE last_name = 'Smith' AND first_name > 'A' AND dob = '1990-01-01';
```

Rule of thumb: **equality columns first, range columns last** in a composite index.

### c. Functions on indexed columns prevent index use

A predicate like `WHERE f(col) = const` cannot use a normal index on `col` because the index is keyed on `col`, not `f(col)`. Wrap the column, and the index is dead:

```sql
-- idx users (created_at)
SELECT * FROM users WHERE DATE(created_at) = '2024-01-01';  -- cannot use index
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';  -- range, uses index
```

MySQL 8.0+ supports **functional indexes** — you can index an expression directly:

```sql
ALTER TABLE users ADD KEY idx_created_date ((DATE(created_at)));
```

But generally prefer rewriting the predicate to a sargable range.

### d. Implicit type conversion

Comparing a string column to a number (or vice versa) forces a type cast on the column, which acts like a function — the index is unusable.

```sql
-- phone_number is VARCHAR
SELECT * FROM users WHERE phone_number = 4155551212;     -- implicit cast: index NOT used
SELECT * FROM users WHERE phone_number = '4155551212';    -- good: index used
```

Same story for `CHAR` vs `VARCHAR` mismatches with different collations in joins — the collation has to match.

### e. Leading-wildcard LIKE

`LIKE 'smith%'` is sargable (uses the index as a range). `LIKE '%smith'` is not — the B+Tree is sorted by prefix, and a suffix has no ordering. For full-text search use a `FULLTEXT` index or an external search engine.

```sql
-- good
SELECT * FROM users WHERE last_name LIKE 'Smi%';
-- bad — index unusable, full scan
SELECT * FROM users WHERE last_name LIKE '%ith';
```

### f. OR across indexes

`WHERE a = 1 OR b = 2` where `a` and `b` each have an index can use **Index Merge** (see below), but historically it has been inefficient. Often better to rewrite as `UNION`:

```sql
SELECT * FROM t WHERE a = 1
UNION
SELECT * FROM t WHERE b = 2;
```

OR within the same composite index works fine if it is sargable on the leading column.

## 5. Composite index column ordering

Bring together the leading-column rule and range-ordering: **equality columns first, then the range column, then columns used for `ORDER BY`/`GROUP BY`/covering.**

```sql
-- Query
SELECT user_id, status, created_at
FROM orders
WHERE merchant_id = 7
  AND created_at >= '2024-01-01'
ORDER BY created_at DESC
LIMIT 50;

-- Best composite index:  (merchant_id, created_at)
--       ^ equality      ^ range + sort order matches
```

**Selectivity heuristic**: put the most selective (highest-cardinality, most-filtering) column first — *when* all predicates are equality. The "most selective first" heuristic has a caveat: it is irrelevant when a column is used for a range, because the range column must come after the equalities regardless of selectivity. And if the index is also used for ordering, the ordering columns must come after the equalities or the engine will not use the index for the sort.

## 6. Index merge

MySQL 5.0+ can combine multiple indexes for a single table, fetching row IDs from each and merging. Visible in `EXPLAIN` as `type=index_merge` and `Extra: Using union(...); Using where`. Index merge is a fallback, not a primary plan — it is often worse than a single well-designed composite index. Do not rely on it; design a composite index instead.

## 7. Index cardinality and selectivity

- **Cardinality** = number of distinct values in the column.
- **Selectivity** = cardinality / total rows. 1.0 is perfect (unique). Selectivity close to 0 (e.g. a boolean `is_active` with 99% true) means the index is rarely useful for filtering.

InnoDB maintains approximate cardinality statistics by sampling pages (`innodb_stats_sample_pages`, default 20). You can force a refresh:

```sql
ANALYZE TABLE users;
```

A low-cardinality index can still be useful as part of a composite index (e.g. `is_active` followed by `created_at`) or for a covering scan.

## 8. Over-indexing: when NOT to add an index

Every secondary index costs:

- **Write amplification**: each `INSERT`/`UPDATE`/`DELETE` that touches an indexed column must update every index B+Tree plus the clustered index.
- **Storage**: each index occupies roughly the size of `(indexed column + PK) * row count`, plus tree overhead.
- **Memory**: index pages take buffer-pool space away from data pages.

Do not add an index when:

- The query is rare (e.g. a monthly report). Use a separate analytics replica or aggregate table.
- The column has very low selectivity *and* the index would not be covering.
- The table is small (a few thousand rows) — sequential scans dominate.
- The table is write-heavy with no matching read workload.
- The index would not actually be used: check with `EXPLAIN` after adding; if `type=ALL` persists, drop it.

Drop unused indexes by querying `performance_schema.table_io_waits_summary_by_index_usage` or `sys.schema_unused_indexes`.

## 9. The optimizer and statistics

InnoDB's cost model uses:

- Per-table cardinality stats (sampled, refreshed by `ANALYZE TABLE`).
- Data dictionary stats on engine features.
- In **MySQL 8.0+**, **histogram statistics** for non-indexed columns. The optimizer can store value distributions on a column and use them to estimate `col = const` selectivity even without an index.

```sql
ANALYZE TABLE users UPDATE HISTOGRAM ON country WITH 100 BUCKETS;
ANALYZE TABLE users DROP HISTOGRAM ON country;
```

Histograms help the optimizer decide between two competing indexes for an inequality, e.g. a `country` with skewed distribution.

Statistics can be frozen with `ANALYZE TABLE ... PERSISTENT FOR ...` and disabled autoupdate per column if you have a tricky plan that regresses.

## 10. Slow query log and analysis workflow

1. **Enable the slow log**:

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.1;   -- seconds; 10 = too coarse for tuning
SET GLOBAL log_queries_not_using_indexes = ON;
```

2. **Collect** for an hour or a day under realistic load.
3. **Aggregate** with `mysqldumpslow` or, much better, `pt-query-digest`:

```bash
pt-query-digest /var/log/mysql/mysql-slow.log
```

This groups queries by fingerprint and ranks by cumulative time — focus on the top items.
4. Run `EXPLAIN` for each top query.
5. Identify the access method (`type`), whether the index is covering (`Using index`), whether there is a `filesort` or `temporary`, and the `rows` estimate.
6. Apply the smallest change first: a composite/covering index, a query rewrite (sargable predicates), then denormalization.
7. Re-measure.

The `performance_schema` and `sys` schema expose per-query stats without the file overhead:

```sql
SELECT digest_text, count_star, avg_timer_wait/1000000000 avg_ms,
       sum_rows_examined/count_star rows_per_call
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 10;
```

A high `rows_per_call` relative to rows returned is the classic sign of a missing index.

## 11. LIMIT and pagination

### Naive OFFSET

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
```

`OFFSET 10000` does **not** skip 10000 rows cheaply. The engine still reads and discards them — generating and discarding 10000 rows takes time scaling with `OFFSET`. At large offsets this becomes a full table scan in disguise.

### Keyset (cursor) pagination

Use the last row's sort key as the cursor:

```sql
-- first page
SELECT id, created_at, merchant_id
FROM orders
WHERE merchant_id = 7
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- subsequent pages, given (prev_created_at, prev_id):
SELECT id, created_at, merchant_id
FROM orders
WHERE merchant_id = 7
  AND (created_at, id) < ('2024-05-01 12:00:00', 98765)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

With an index on `(merchant_id, created_at, id)` this is a cheap indexed range scan returning exactly 20 rows, regardless of page depth. The index must support the sort key order; the tie-breaker `id` is essential so the comparison is strictly monotonic.

Bonus: cursor pagination is stable under inserts/deletes (no duplicate or skipped rows), unlike `OFFSET`.

### LIMIT optimization gotchas

- `LIMIT` without `ORDER BY` returns rows in **non-deterministic** order — especially bad when paginating.
- The optimizer can use `ORDER BY ... LIMIT n` to stop early on an index scan when the order matches the index; otherwise it produces the whole result and sorts.

## 12. Anti-patterns checklist

| Anti-pattern | Why bad | Fix |
|--------------|---------|-----|
| `SELECT *` | Reads every column → no covering index, more I/O, more network | Name the columns you need |
| Function on indexed column in `WHERE` | Index unusable | Rewrite to sargable range, or use functional index |
| `WHERE DATE(col) = '...'` | Unindexable | `WHERE col >= ... AND col < ...` |
| `LIKE '%substr%'` | Unindexable, full scan | `FULLTEXT` index or external search |
| `OR` across two indexed columns | Often needs index merge; sometimes full scan | Rewrite as `UNION ALL`/`UNION` |
| Implicit type cast | Index unusable | Match column type to literal type |
| `OFFSET` pagination | Linear scan cost per page | Keyset / cursor pagination |
| `LIMIT 1` over unindexed filter | Full scan | Add an index for that filter |
| N+1 from outer loop | Runs N queries, no index leverage per call | Do one join or batch the IN-list |
| `DISTINCT`/`GROUP BY` without covering index | `Using temporary; Using filesort` | Add index covering the grouping columns |

## 13. Worked example

```sql
CREATE TABLE orders (
  id           BIGINT UNSIGNED PRIMARY KEY,
  user_id      BIGINT UNSIGNED NOT NULL,
  status       VARCHAR(16)     NOT NULL,
  total        DECIMAL(12,2)   NOT NULL,
  created_at  DATETIME(0)      NOT NULL,
  KEY idx_user_status_created (user_id, status, created_at)
) ENGINE=InnoDB;

-- Reported slow query
SELECT id, status, total, created_at
FROM orders
WHERE user_id = 88
  AND status = 'PAID'
  AND created_at >= '2024-01-01'
ORDER BY created_at DESC
LIMIT 50;

EXPLAIN
SELECT id, status, total, created_at
FROM orders
WHERE user_id = 88
  AND status = 'PAID'
  AND created_at >= '2024-01-01'
ORDER BY created_at DESC
LIMIT 50;
-- Without idx_user_status_created: type=ALL, Extra=Using where; Using filesort.
-- With idx_user_status_created:
--   type=range, key=idx_user_status_created, key_len reflects user_id+status+created_at,
--   Extra: Using index condition; Backward index scan (DESC over a forward index).
-- Returns exactly 50 rows via index; no filesort, no full scan.
```

The covering index makes this query both filtered (range) and ordered (matching `created_at`) for free. If you needed the user's `email` too, you would do a join to `users` rather than widen `orders` and break the covering index.

## 14. Quick reference: optimizer pointers

- Always verify with `EXPLAIN` *before and after* adding an index. If the extra index is unused → drop it.
- Prefer sargable predicates over `f(col) = const`.
- Prefer `=`, `IN`, `BETWEEN`, `LIKE 'x%'` over `%x%`.
- For hot queries, design a covering composite index: `equality, equality, range, order-by-columns`.
- Match collations across joined columns.
- Use `EXPLAIN ANALYZE` to see actual row counts per node (` loops=… rows=… actual_rows=…`).
- Set `long_query_time` low enough during tuning windows — the slow log is the cheapest, highest-signal tool you have.