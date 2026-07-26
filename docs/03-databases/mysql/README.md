# MySQL Interview Prep

This section covers MySQL concepts deeply enough for a backend engineering interview, with InnoDB-specific behavior called out where it matters. Content is current as of **MySQL 8.4 LTS** (still the most common production baseline) and the **9.x line** — quarterly innovation releases (9.0–9.6) consolidated into **MySQL 9.7 LTS** (April 2026). All of these use InnoDB as the default storage engine, support `INSTANT` DDL, histogram statistics, `utf8mb4` as the default charset, CTEs, window functions, and GTID-based replication.

## Files in this section

| File | Description |
|------|-------------|
| `01-indexing-and-query-optimization.md` | B+Tree indexes, clustered vs secondary, EXPLAIN, composite index ordering, covering indexes, invisible indexes, pagination, slow query workflow. |
| `02-transactions-and-isolation.md` | ACID, isolation levels, InnoDB next-key locking, MVCC, locking reads, deadlocks, redo/undo/binlog and two-phase commit. |
| `03-replication-and-schema-design.md` | Async/semi-sync/group replication, GTID, clone plugin, high availability, normalization, data types, UUID vs auto-increment, partitioning, sharding, online schema changes. |
| `04-interview-questions.md` | 35 interview questions graded junior/senior/staff, with thorough model answers. |

## Versions covered

- **MySQL 8.4 LTS** (released 2024, long-term support until ~2032) — the most widely deployed production baseline.
- **MySQL 9.7 LTS** (released April 2026, supported ~8 years) — the new LTS line, consolidating the 9.x innovation releases. Headline additions over 8.4: the `VECTOR` data type (9.0+), JavaScript stored programs (Enterprise Edition, 9.0+), and incremental optimizer/replication improvements. The supported upgrade path is 8.4 LTS → 9.7 LTS (you cannot skip an LTS series).
- **MySQL 8.0** (released 2018) reached **end of life in April 2026**. Treat it as legacy in interviews — differences are noted inline where 8.0 behavior still shows up in production.
- InnoDB is the default and only transactional engine of note. MyISAM is legacy and non-transactional; mentioned only for contrast.
- The **query cache** was removed entirely in MySQL 8.0 — if an interviewer (or an old blog post) presents it as a tuning lever, that is a trap. Caching lives in the application layer (Redis/Memcached) now; see `../redis/`.

## Recommended reading order

1. `01-indexing-and-query-optimization.md` — indexing is the single most common interview topic and the foundation for performance work.
2. `02-transactions-and-isolation.md` — builds on the row/locking model implied by indexes.
3. `03-replication-and-schema-design.md` — broadens to distributed systems and schema trade-offs.
4. `04-interview-questions.md` — use to self-test after reading the reference docs; answers cross-reference the above.

## How to use this section

- Read the reference docs top-to-bottom first.
- Try to answer each interview question from memory before reading the model answer.
- Where SQL examples are given, run them against a local MySQL 8.4+ instance — behavior is much easier to internalize hands-on.
- Topics marked **InnoDB-specific** differ on other engines (and historically on MyISAM); interviewers often probe whether you know they are engine-specific.
- "PostgreSQL vs MySQL" is a standard 2026 interview question. After this section, read the companion [PostgreSQL section](../postgresql/README.md) and be ready to compare MVCC implementations, index types, and replication models.

## Companion resources

- Official manual: <https://dev.mysql.com/doc/refman/8.4/en/>
- `EXPLAIN` output reference: <https://dev.mysql.com/doc/refman/8.4/en/explain-output.html>
- InnoDB locking reference: <https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html>
- LTS vs innovation release model: <https://dev.mysql.com/doc/refman/8.4/en/mysql-releases.html>
