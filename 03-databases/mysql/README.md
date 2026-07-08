# MySQL Interview Prep

This section covers MySQL concepts deeply enough for a backend engineering interview, with InnoDB-specific behavior called out where it matters. Content is current as of **MySQL 8.4 LTS** and **MySQL 8.0** (still widely deployed). Both versions use InnoDB as the default storage engine, support `INSTANT` DDL, histogram statistics, `utf8mb4` as the default charset, CTEs, window functions, and GTID-based replication.

## Files in this section

| File | Description |
|------|-------------|
| `01-indexing-and-query-optimization.md` | B+Tree indexes, clustered vs secondary, EXPLAIN, composite index ordering, covering indexes, pagination, slow query workflow. |
| `02-transactions-and-isolation.md` | ACID, isolation levels, InnoDB next-key locking, MVCC, locking reads, deadlocks, redo/undo/binlog and two-phase commit. |
| `03-replication-and-schema-design.md` | Async/semi-sync replication, GTID, high availability, normalization, data types, UUID vs auto-increment, partitioning, sharding, online schema changes. |
| `04-interview-questions.md` | 30+ interview questions grouped by difficulty, with thorough model answers. |

## Versions covered

- **MySQL 8.4 LTS** (released 2024, long-term support until ~2032) — current default.
- **MySQL 8.0** (released 2018, EOL April 2026) — still common in production; differences from 8.4 are noted inline.
- InnoDB is the default and only transactional engine of note. MyISAM is legacy and non-transactional; mentioned only for contrast.

## Recommended reading order

1. `01-indexing-and-query-optimization.md` — indexing is the single most common interview topic and the foundation for performance work.
2. `02-transactions-and-isolation.md` — builds on the row/locking model implied by indexes.
3. `03-replication-and-schema-design.md` — broadens to distributed systems and schema trade-offs.
4. `04-interview-questions.md` — use to self-test after reading the reference docs; answers cross-reference the above.

## How to use this section

- Read the reference docs top-to-bottom first.
- Try to answer each interview question from memory before reading the model answer.
- Where SQL examples are given, run them against a local MySQL 8.0+ instance — behavior is much easier to internalize hands-on.
- Topics marked **InnoDB-specific** differ on other engines (and historically on MyISAM); interviewers often probe whether you know they are engine-specific.

## Companion resources

- Official manual: <https://dev.mysql.com/doc/refman/8.4/en/>
- `EXPLAIN` output reference: <https://dev.mysql.com/doc/refman/8.4/en/explain-output.html>
- InnoDB locking reference: <https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html>