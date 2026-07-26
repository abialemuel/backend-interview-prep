# PostgreSQL Interview Prep

This section covers PostgreSQL deeply enough for a backend engineering interview. PostgreSQL has become the default relational database for new backends — many companies that used to run MySQL now start on Postgres (or actively migrate to it) because of its stronger SQL feature set, extensibility (JSONB, full-text search, `pgvector`), and the ecosystem around it (Aurora PostgreSQL, AlloyDB, Neon, Supabase, Timescale). Interviewers increasingly assume Postgres knowledge by default and probe the parts where it differs from MySQL: the MVCC/vacuum model, the process-per-connection architecture, and the richer index types.

Content is current as of **PostgreSQL 18** (released 2025) and **PostgreSQL 17** (2024) — version-specific behavior is noted inline. Anything unmarked applies to every version you will realistically meet in production (13+).

## Files in this section

| File | Description |
|------|-------------|
| `01-architecture-and-mvcc.md` | Process model, MVCC internals (xmin/xmax, snapshots), VACUUM and autovacuum, WAL and checkpoints, TOAST, table/index bloat, transaction ID wraparound. |
| `02-indexing-and-performance.md` | B-tree/GIN/GiST/BRIN/hash index types, partial and expression indexes, reading `EXPLAIN (ANALYZE, BUFFERS)`, the cost-based planner and statistics, connection pooling with PgBouncer, prepared statements. |
| `03-advanced-features.md` | JSONB, full-text search, CTEs and window functions, declarative partitioning, streaming vs logical replication, key extensions (`pg_stat_statements`, `pgvector`), row-level security, LISTEN/NOTIFY. |
| `04-interview-questions.md` | 18 graded interview questions (junior → senior → staff) with model answers, including "PostgreSQL vs MySQL — when and why" and production scenario questions. |

## Versions covered

- **PostgreSQL 18** (2025) — asynchronous I/O, B-tree skip scans, built-in `uuidv7()`, virtual generated columns, planner statistics preserved across `pg_upgrade`.
- **PostgreSQL 17** (2024) — reworked VACUUM memory management, SQL/JSON `JSON_TABLE`, failover-safe logical replication slots, incremental backup.
- **PostgreSQL 15/16** — still very common in production; `MERGE` (15), parallel and `pg_stat_io` improvements (16). Differences from 17/18 are noted inline.

## Recommended reading order

1. `01-architecture-and-mvcc.md` — the MVCC/vacuum model is *the* thing that makes Postgres different; almost every hard interview question traces back to it.
2. `02-indexing-and-performance.md` — builds on the storage model; index-only scans and the visibility map only make sense after you understand vacuum.
3. `03-advanced-features.md` — the features that usually decide "why Postgres over MySQL" in system design rounds.
4. `04-interview-questions.md` — self-test after the reference docs; answers cross-reference the above.

## How to use this section

- Read the reference docs top-to-bottom first; try to answer each interview question from memory before reading the model answer.
- Run the SQL examples against a local Postgres 17+ instance (`docker run -e POSTGRES_PASSWORD=pw postgres:18`) — `EXPLAIN ANALYZE` output is much easier to internalize hands-on.
- If you already read the MySQL section, pay special attention to the **differences**: heap tables vs clustered indexes, vacuum vs purge, `REPEATABLE READ` semantics, and replication models. Interviewers love "how does X differ between MySQL and Postgres" questions.

## Companion resources

- Official manual: <https://www.postgresql.org/docs/current/>
- `EXPLAIN` documentation: <https://www.postgresql.org/docs/current/using-explain.html>
- Routine vacuuming: <https://www.postgresql.org/docs/current/routine-vacuuming.html>
- PgBouncer: <https://www.pgbouncer.org/>
