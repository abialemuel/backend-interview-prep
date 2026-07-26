# Advanced Features

These are the features that answer "why did you pick PostgreSQL?" in a system design interview: a document store, a search engine, an analytics layer, a queue, and a vector database — good-enough versions of each inside one transactional system. The senior-level skill is knowing where "good enough" ends and a dedicated system begins; each section below states that boundary.

## 1. JSONB

Postgres has two JSON types: `json` (stores the raw text, preserves key order/duplicates, reparsed on every access) and **`jsonb`** (parsed binary format, deduplicated keys, binary comparison and indexing). Use `jsonb` unless you must preserve the exact original text.

```sql
CREATE TABLE events (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  type    text NOT NULL,
  payload jsonb NOT NULL
);

-- Operators you should know cold:
SELECT payload->'user'->>'email'          FROM events;                    -- navigate; ->> returns text
SELECT * FROM events WHERE payload @> '{"country": "AE"}';               -- containment
SELECT * FROM events WHERE payload ? 'refund_id';                        -- key exists
SELECT * FROM events WHERE payload @? '$.items[*] ? (@.price > 100)';    -- SQL/JSON path (12+)
UPDATE events SET payload = payload || '{"reviewed": true}';             -- merge
UPDATE events SET payload = jsonb_set(payload, '{user,email}', '"x@y.z"');
```

Indexing: a **GIN** index makes containment and existence fast; `jsonb_path_ops` is a smaller, faster variant supporting only `@>`/`@?`. For one hot field, a **B-tree expression index** on `(payload->>'field')` beats GIN. PG 17 added the SQL-standard `JSON_TABLE()` — shred JSONB into relational rows inside a query — plus `JSON_EXISTS`/`JSON_QUERY`/`JSON_VALUE`.

Design guidance (the actual interview question): keep **columns for what you query, join on, or constrain; JSONB for what you store and pass through** — ingested webhooks, per-tenant settings, sparse attributes. JSONB fields have no statistics by default (estimate problems), no FKs, no type checking, every update rewrites the whole datum (TOAST, file 01 §6), and there is no partial-document locking. "Schema-less" quickly becomes "schema in the application, enforced nowhere". If everything is JSONB, you have rebuilt MongoDB minus its scaling model and plus MVCC bloat.

## 2. Full-text search

Built-in FTS pipeline: a **`tsvector`** (normalized lexemes with positions) matched against a **`tsquery`**, with language-aware stemming and stop words.

```sql
ALTER TABLE articles ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body,  '')), 'B')
  ) STORED;

CREATE INDEX idx_articles_search ON articles USING gin (search);

SELECT id, title, ts_rank(search, q) AS rank
FROM articles, websearch_to_tsquery('english', 'postgres "full text" -mysql') q
WHERE search @@ q
ORDER BY rank DESC LIMIT 20;
```

`websearch_to_tsquery` accepts Google-like syntax safely from user input. Ranking (`ts_rank`) is computed per matching row at query time — fine for thousands of matches, painful for millions. `ts_headline` generates highlighted snippets.

Boundary: Postgres FTS covers "search my app's records" very well — one system, transactional consistency, no sync pipeline. Reach for Elasticsearch/OpenSearch/Meilisearch when you need relevance tuning (BM25, per-field boosting beyond A–D weights), typo tolerance/fuzzy matching, faceted aggregations at scale, or search QPS high enough to deserve its own cluster. Fuzzy *name* matching in Postgres is `pg_trgm` similarity, not FTS.

## 3. CTEs and window functions

**CTEs** (`WITH`) name subqueries; **recursive CTEs** traverse hierarchies (org charts, category trees, graph reachability):

```sql
WITH RECURSIVE subordinates AS (
  SELECT id, manager_id, name, 1 AS depth FROM employees WHERE id = 42
  UNION ALL
  SELECT e.id, e.manager_id, e.name, s.depth + 1
  FROM employees e JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

The version-history point: before PG 12, every CTE was an **optimization fence** (materialized, no predicate pushdown). Since 12, non-recursive CTEs referenced once are inlined; you can force either behavior with `AS MATERIALIZED` / `AS NOT MATERIALIZED`. A deliberate fence is occasionally useful to pin a plan.

**Window functions** compute over a partition without collapsing rows:

```sql
-- Top 3 orders per customer, and each order's share of customer total:
SELECT * FROM (
  SELECT customer_id, id, total,
         row_number() OVER w AS rn,
         total / sum(total) OVER (PARTITION BY customer_id) AS share
  FROM orders
  WINDOW w AS (PARTITION BY customer_id ORDER BY total DESC)
) t WHERE rn <= 3;
```

Know the families: ranking (`row_number`, `rank`, `dense_rank`, `ntile`), offsets (`lag`, `lead` — month-over-month deltas), aggregates over frames (`sum(...) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` — rolling averages). Top-N-per-group via `row_number` is the single most common practical use. MySQL has had these since 8.0, so they are table stakes — but interviewers still use them to separate "writes CRUD" from "writes SQL".

## 4. Declarative partitioning

Split one logical table into physical child tables by **RANGE**, **LIST**, or **HASH**:

```sql
CREATE TABLE events (
  id bigint GENERATED ALWAYS AS IDENTITY,
  created_at timestamptz NOT NULL,
  payload jsonb,
  PRIMARY KEY (id, created_at)          -- partition key must be in every unique constraint
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_07 PARTITION OF events
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
```

Why partition — in honest priority order:

1. **Instant data retention**: `DROP TABLE events_2025_07` (or `DETACH PARTITION ... CONCURRENTLY`) removes a month in milliseconds with zero bloat, versus a `DELETE` that writes millions of dead tuples and a vacuum hangover.
2. **Partition pruning**: queries filtering on the partition key touch only relevant partitions (verify with `EXPLAIN` — a query *without* the key scans every partition).
3. **Maintainability**: vacuum, `ANALYZE`, and indexes operate per-partition; smaller B-trees, saner autovacuum on TB-scale data.

Costs: the partition key must appear in primary/unique constraints (global unique indexes don't exist); too many partitions (>few thousand) inflate planning time; partition management needs automation (`pg_partman`). Don't partition tables that fit in memory — below ~100 GB it usually adds complexity for nothing. Partitioning is *not* sharding: all partitions live on one node; for write scale-out you need Citus or application-level sharding.

## 5. Replication: streaming vs logical

Two fundamentally different mechanisms, one interview question.

| | Streaming (physical) | Logical |
|---|---|---|
| Ships | WAL bytes — physical block changes | Decoded row changes (INSERT/UPDATE/DELETE) via a publication |
| Replica is | Byte-identical copy of the whole cluster, read-only | A normal writable database applying changes to subscribed tables |
| Granularity | Entire cluster | Per table / subset via `PUBLICATION` |
| Cross-version | No — same major version | Yes — the basis of near-zero-downtime major upgrades |
| Use for | HA, failover, read replicas | Selective sync, upgrades, CDC, consolidating/splitting databases |

**Streaming replication** is the HA workhorse: replicas replay WAL and serve read-only queries (hot standby). Modes: asynchronous (default; failover can lose the last moments of commits) vs **synchronous** (`synchronous_standby_names`, per-transaction override via `synchronous_commit` — `remote_apply` guarantees read-your-writes on the standby). Know the failure story: `pg_stat_replication` for lag, **replication slots** to stop the primary discarding WAL a replica still needs — and the trap that a slot for a *dead* replica retains WAL forever until the disk fills (`max_slot_wal_keep_size` caps this).

**Logical replication** (`CREATE PUBLICATION` / `CREATE SUBSCRIPTION`) decodes WAL into logical changes. Restrictions to name: DDL is not replicated (schema changes must be coordinated), sequences don't replicate, and `UPDATE`/`DELETE` need a replica identity (PK). The same decoding infrastructure powers **CDC** — Debezium reads a logical slot via `pgoutput` and streams changes to Kafka; this is the standard answer to "how do you keep Elasticsearch/caches in sync with Postgres" (transactional outbox or CDC, not dual writes). PG 16 allows logical replication *from a standby*; PG 17 added **failover slots** (logical slots survive physical failover) and `pg_createsubscriber` (bootstrap a logical subscriber from a physical replica — no slow initial copy).

## 6. Extensions

`CREATE EXTENSION` is Postgres's superpower — the reason its ecosystem outruns MySQL's. Four to know well:

**`pg_stat_statements`** — aggregated per-query-shape statistics; the first tool for any performance investigation and effectively mandatory in production (`shared_preload_libraries`).

```sql
SELECT queryid, calls, mean_exec_time, rows,
       shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0)::float AS hit_ratio,
       query
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

**`pgvector`** — vector similarity search for embeddings, the default RAG storage answer in 2026:

```sql
CREATE EXTENSION vector;
CREATE TABLE docs (id bigint PRIMARY KEY, content text, embedding vector(1536));
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops);   -- ANN index
SELECT id, content FROM docs ORDER BY embedding <=> $1 LIMIT 10; -- <=> cosine distance
```

Index choice: **HNSW** (graph-based; better recall/latency, slower builds, more memory) vs **IVFFlat** (clustering-based; faster builds, needs data present at creation, recall depends on `probes`). Both are *approximate* — the recall/speed trade-off is the interview point, plus the architectural one: pgvector keeps embeddings transactional and joinable with your relational data (filter by tenant + vector search in one query); dedicated vector DBs (Pinecone, etc.) win at billions of vectors or when vector search is the whole product. `halfvec` and binary quantization cut memory 2–8x when needed.

Also name-drop: **`pg_trgm`** (fuzzy/substring matching), **PostGIS** (the geospatial standard), **`pg_partman`** (partition automation), **TimescaleDB** (time-series). Managed services whitelist extensions — check before designing around one.

## 7. Row-level security

RLS attaches row-filtering policies to a table, enforced by the database itself for every query path:

```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- Per-request, inside the transaction (PgBouncer-safe):
SET LOCAL app.tenant_id = '42';
SELECT * FROM documents;   -- only tenant 42's rows, no WHERE clause needed
```

`USING` filters visible rows; `WITH CHECK` validates writes. Gotchas that separate real users from feature-list readers: **table owners and superusers bypass RLS** unless `FORCE ROW LEVEL SECURITY` — so the app must connect as a non-owner role; policies add a predicate to every query, so they must be index-friendly (`tenant_id` leading every index); and with PgBouncer transaction pooling the tenant setting must be `SET LOCAL`. RLS is the backbone of shared-table multi-tenancy (it is how Supabase exposes Postgres directly to clients) — defense in depth against the "forgot the WHERE tenant_id" class of bug, at some planner-complexity cost.

## 8. LISTEN / NOTIFY

Built-in pub/sub over a connection:

```sql
LISTEN order_events;
NOTIFY order_events, '{"order_id": 123}';           -- or pg_notify('order_events', $1)
```

Properties that define its niche: notifications from a transaction are delivered **only on commit** (transactional pub/sub for free); delivery is **at-most-once to currently-connected listeners** — no persistence, no replay, no delivery guarantee if the listener is down; payloads are small (8000 bytes); and a listener needs a real, dedicated connection (**not** compatible with PgBouncer transaction pooling).

The canonical pattern is *not* sending data via NOTIFY, but using it as a **wake-up signal over a queue table**: writers insert jobs and `NOTIFY`; workers `LISTEN` and, when woken, claim work with:

```sql
UPDATE jobs SET status = 'running', claimed_by = $1
WHERE id = (
  SELECT id FROM jobs WHERE status = 'pending'
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED     -- the crucial clause: no worker contention
  LIMIT 1
)
RETURNING *;
```

`SKIP LOCKED` + LISTEN/NOTIFY is a perfectly good job queue up to thousands of jobs/sec (this is what pgmq, Oban, Solid Queue, and Graphile Worker build on) — with exactly-once *processing* semantics because claiming is transactional. Move to Kafka/SQS/RabbitMQ when you need replay, fan-out to many consumer groups, cross-service decoupling, or throughput that would turn the jobs table into a vacuum nightmare (file 01 §8: queue tables are the classic bloat workload).

## 9. Quick reference

- `jsonb` + GIN for flexible attributes; columns for anything you filter, join, or constrain. Whole-datum rewrite on every update.
- FTS: generated `tsvector` column + GIN + `websearch_to_tsquery`; Elasticsearch only for relevance tuning, fuzziness, facets at scale.
- CTEs inline since PG 12 (`MATERIALIZED` to force a fence); recursive CTEs for trees; window functions for top-N-per-group and rolling metrics.
- Partitioning buys instant retention (`DROP PARTITION`) and pruning; it is not sharding; partition key must be in unique constraints.
- Streaming replication = HA and read replicas (physical, whole-cluster); logical = selective, cross-version, CDC. Watch abandoned replication slots.
- `pg_stat_statements` always; `pgvector` (HNSW vs IVFFlat, approximate) for embeddings alongside relational data.
- RLS for multi-tenant isolation: non-owner role, `SET LOCAL`, index the policy predicate.
- LISTEN/NOTIFY = commit-time wake-up signal; pair with `FOR UPDATE SKIP LOCKED` for a Postgres-native job queue.
