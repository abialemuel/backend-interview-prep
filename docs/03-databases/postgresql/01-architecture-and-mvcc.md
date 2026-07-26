# Architecture and MVCC

PostgreSQL's storage and concurrency model is fundamentally different from InnoDB's, and those differences drive nearly every operational behavior interviewers ask about: why vacuum exists, why long transactions are dangerous, why an `UPDATE`-heavy table bloats, and what transaction ID wraparound is. Understand this file and the rest of the section follows.

## 1. Process model

PostgreSQL is a **process-per-connection** server. The `postmaster` process listens for connections and `fork()`s one **backend process** per client connection. Backends share state through **shared memory**, primarily:

- **shared_buffers** — the page cache for table and index pages (typically set to ~25% of RAM; the OS page cache handles the rest, so Postgres deliberately double-buffers).
- Lock tables, the commit log (`pg_xact`), and the WAL buffers.

Alongside the backends, a set of background processes do the housekeeping:

| Process | Role |
|---------|------|
| WAL writer | Flushes WAL buffers to disk on a timer, offloading work from commit paths |
| Checkpointer | Periodically writes all dirty shared buffers to disk and marks a checkpoint in WAL |
| Background writer | Trickles dirty pages out between checkpoints so backends rarely have to evict dirty pages themselves |
| Autovacuum launcher + workers | Vacuum and analyze tables automatically (see §4) |
| WAL sender / receiver | Ship WAL to replicas (streaming replication) |
| Logical replication workers | Apply logical changes on subscribers |
| I/O workers (PG 18) | Perform asynchronous reads when `io_method = worker` |

The interview-relevant consequence: **each connection costs a real OS process** — a few MB of memory, scheduler overhead, and contention on shared structures. Postgres degrades noticeably past a few hundred *active* connections, which is why external connection pooling (PgBouncer, covered in `02-indexing-and-performance.md`) is near-mandatory at scale, whereas MySQL's thread-per-connection model tolerates thousands of connections more gracefully.

!!! note
    PostgreSQL 18 introduced **asynchronous I/O** (`io_method = worker` by default, `io_uring` on Linux). Sequential scans, bitmap heap scans and vacuum can now issue overlapping reads instead of blocking on each page, which materially speeds up cold-cache scans. Before 18, all reads were synchronous, relying on OS readahead.

## 2. Heap tables — no clustered index

Unlike InnoDB, a Postgres table is a **heap**: an unordered collection of 8 KB pages. Rows (tuples) live wherever there is free space. The primary key is just an ordinary unique B-tree index over the heap; it does **not** determine physical row order.

Every index entry — primary key or secondary — points to a tuple's physical address, the **TID** (`ctid`), a `(page, item)` pair. Consequences worth stating in an interview:

- **All indexes are equal citizens.** There is no "secondary index does a bookmark lookup through the PK" as in InnoDB. Every index lookup goes index → TID → heap page.
- **Insert order ≈ physical order** initially, but updates and space reuse scramble it over time. `CLUSTER` can rewrite a table in index order, but the ordering is not maintained afterwards.
- **Random-UUID primary keys hurt much less** than in InnoDB (no clustered tree to fragment), though they still bloat the PK index itself and ruin its cache locality. PG 18 ships a built-in `uuidv7()` (time-ordered, RFC 9562) which keeps index inserts append-mostly.

## 3. MVCC internals: xmin, xmax, snapshots

Postgres implements MVCC by **keeping old row versions in the table itself** — not in a separate undo log like InnoDB. Every tuple carries system columns:

- **`xmin`** — the transaction ID (XID) that created this tuple version.
- **`xmax`** — the XID that deleted it (or superseded it via `UPDATE`); `0` if live.
- **`ctid`** — its physical address; an updated tuple's old version points forward to the new one.

An `UPDATE` is physically a **DELETE + INSERT**: the old tuple gets its `xmax` stamped, and a complete new tuple is written (possibly on another page). A `DELETE` just stamps `xmax`. Nothing is removed in-place — the dead versions stay in the heap until vacuum reclaims them.

Visibility: each statement (READ COMMITTED) or transaction (REPEATABLE READ and above) takes a **snapshot** — the set of XIDs that were in-flight at that moment. A tuple is visible if its `xmin` is committed and before the snapshot, and its `xmax` is absent, aborted, or after the snapshot. Commit status of each XID lives in `pg_xact`; per-tuple **hint bits** cache the answer after the first check (which is why a big `SELECT` right after a bulk load can generate surprising write I/O — it is setting hint bits).

```sql
-- You can inspect the versioning directly:
SELECT ctid, xmin, xmax, * FROM accounts WHERE id = 1;
```

Trade-off versus InnoDB's undo-log design, in one sentence each:

- **Postgres:** readers never visit a separate undo structure and rollback is instant (just mark the XID aborted), but dead tuples pollute the heap and indexes until vacuum runs.
- **InnoDB:** the heap stays compact and purge is cheap, but readers of hot rows may walk long undo chains, and rollback of a big transaction is slow (it must undo each change).

Two optimizations reduce the update cost:

- **HOT (Heap-Only Tuple) updates** — if no indexed column changed *and* the new version fits on the same page, Postgres links the new tuple within the page and no index entries need updating. This is why `fillfactor < 100` on update-heavy tables (leaving per-page free space) matters, and why "don't index columns you don't query" is doubly true in Postgres: every extra index makes HOT less likely.
- **Index-only scans + the visibility map** — covered with vacuum below.

### Isolation levels

Postgres implements three distinct levels (`READ UNCOMMITTED` is accepted but behaves as READ COMMITTED):

| Level | Snapshot scope | Anomalies possible |
|-------|----------------|--------------------|
| `READ COMMITTED` (default) | Per statement | Non-repeatable reads, phantoms, lost updates without explicit locking |
| `REPEATABLE READ` | Per transaction | Serialization failures on write-write conflict (error `40001`); no phantoms — stricter than the SQL standard requires |
| `SERIALIZABLE` | Per transaction + SSI | None; predicate conflicts abort with `40001` |

`SERIALIZABLE` uses **Serializable Snapshot Isolation (SSI)**: optimistic detection of dangerous read/write dependency cycles, aborting one transaction rather than blocking. Contrast with InnoDB's SERIALIZABLE, which takes shared locks on every read. The application contract is the same at both `REPEATABLE READ` and `SERIALIZABLE`: **be prepared to retry on `40001`**.

## 4. VACUUM and autovacuum

Because dead tuples accumulate in the heap, something must reclaim them. **VACUUM** does four jobs:

1. **Removes dead tuples** whose `xmax` is older than every active snapshot (the "removal horizon"), making the space reusable *within the table* (plain `VACUUM` does not shrink the file; `VACUUM FULL` rewrites the table under an exclusive lock and does).
2. **Updates the visibility map (VM)** — one bit per page saying "all tuples on this page are visible to everyone." Index-only scans check this bit to skip heap fetches; pages not all-visible force a heap visit per row.
3. **Freezes old tuples** — replaces old `xmin` values with a permanent "frozen" marker so the 32-bit XID space can be reused (see §7).
4. **Updates the free space map (FSM)** so inserts can find reusable space.

`ANALYZE` (often run together) refreshes planner statistics.

**Autovacuum** runs this automatically. A table is vacuumed when dead tuples exceed roughly `autovacuum_vacuum_scale_factor (default 0.2) × rows + autovacuum_vacuum_threshold (50)` — i.e. ~20% churn. On big tables 20% is far too lazy: 20 million dead tuples on a 100M-row table before vacuum triggers. Standard production tuning:

```sql
-- Per-table override for a hot, update-heavy table:
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,   -- vacuum at ~2% dead
  autovacuum_vacuum_cost_delay   = 1       -- vacuum faster (less throttling)
);
```

```sql
-- Is autovacuum keeping up?
SELECT relname, n_dead_tup, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

Key operational facts interviewers probe:

- Autovacuum is **throttled** by a cost model (`autovacuum_vacuum_cost_limit` / `cost_delay`) so it does not saturate I/O; a common failure mode is vacuum that runs but *cannot keep up*, silently, until bloat is severe.
- **A long-running transaction (or an abandoned `idle in transaction` session, or a stale replication slot) holds back the removal horizon globally** — vacuum cannot remove any tuple that transaction might still see, on *any* table. This is the number-one cause of sudden bloat. Monitor `pg_stat_activity` for old `xact_start` and set `idle_in_transaction_session_timeout`.
- PostgreSQL 17 reworked vacuum's dead-tuple memory (a radix-tree TID store): vacuum is no longer capped at 1 GB for dead-TID tracking and uses ~20x less memory, so large tables need far fewer index-scan passes per vacuum.
- Since PG 13, **B-tree deduplication** and **bottom-up index deletion** (PG 14) keep indexes leaner between vacuums.

## 5. WAL: write-ahead log

All durability flows through the **WAL**. Before a dirty data page can be written, the log records describing the change must be flushed. At `COMMIT`, only the WAL is fsynced (`synchronous_commit = on`); the data pages are written later by the checkpointer/background writer.

- WAL lives in `pg_wal/` as 16 MB segments; positions are **LSNs** (log sequence numbers).
- A **checkpoint** flushes all dirty buffers and records "recovery can start here." Crash recovery = load last checkpoint, replay WAL forward. Longer `checkpoint_timeout` / bigger `max_wal_size` → fewer, smoother checkpoints but longer recovery.
- **`full_page_writes`**: the first change to a page after a checkpoint logs the entire 8 KB page, protecting against torn writes — this is why write amplification spikes right after each checkpoint.
- WAL is also the transport for **streaming replication**, **PITR** (base backup + `archive_command` + replay to a timestamp), and the source that **logical decoding** reads (see `03-advanced-features.md`).
- `synchronous_commit = off` trades a few hundred ms of potential data loss on crash (not corruption — the database stays consistent) for a large commit-latency win; a legitimate setting for non-critical writes, and settable per-transaction.
- PG 17 added **incremental backup** (`pg_basebackup --incremental`), using WAL summaries to copy only changed blocks.

## 6. TOAST: oversized values

A tuple must fit in one 8 KB page, so Postgres **TOASTs** large values (The Oversized-Attribute Storage Technique). When a row exceeds ~2 KB, large varlena columns (`text`, `bytea`, `jsonb`, …) are compressed and/or moved out-of-line to an associated **TOAST table**, sliced into ~2 KB chunks; the main tuple keeps an 18-byte pointer.

What matters practically:

- TOASTed columns are fetched **lazily** — `SELECT id FROM docs` never touches the TOAST table; `SELECT body FROM docs` does (extra random I/O per row).
- Compression is `pglz` by default; **`lz4` (PG 14+) is faster** at similar ratios: `ALTER TABLE docs ALTER COLUMN body SET COMPRESSION lz4;`.
- Updating a row with large TOASTed values that don't change is cheap — the new tuple version reuses the same TOAST pointer.
- Heavily updated JSONB documents still rewrite the whole datum on each change (no partial TOAST update), which is a real cost of "just put it in JSONB".

## 7. Transaction ID wraparound

XIDs are **32-bit** and compared circularly: from any XID, ~2 billion XIDs are "in the past" and ~2 billion "in the future". If a tuple's `xmin` fell more than ~2 billion transactions behind, it would suddenly look like it was created *in the future* — i.e. invisible. Committed data would effectively vanish.

**Freezing** prevents this: vacuum marks sufficiently old tuples as *frozen* (visible to everyone, regardless of XID arithmetic — implemented as a hint bit since 9.4). Every table tracks `relfrozenxid`, the oldest unfrozen XID it might contain. The safety machinery escalates:

1. When a table's oldest unfrozen XID exceeds `autovacuum_freeze_max_age` (default 200M), autovacuum launches an **aggressive (anti-wraparound) vacuum** for it — this one cannot be cancelled by ordinary DDL waiting and runs even if autovacuum is disabled.
2. If the oldest XID in the cluster gets within ~3M of wraparound (because vacuums keep failing or a prepared transaction / replication slot pins the horizon), Postgres **refuses new write transactions** until vacuum succeeds — a full outage.

Monitoring is one query:

```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
-- Alert well before age approaches 2^31; many shops alert at 500M–1B.
```

Real-world incidents (famously Sentry's 2015 outage, and Mailchimp's 2019 one) are the standard war-story reference. The staff-level answer: wraparound outages are almost always a *symptom* — something (long transaction, stale replication slot, `DISABLE`d autovacuum, vacuum starved by cost limits) prevented freezing from keeping up.

## 8. Bloat

**Bloat** = space occupied by dead tuples and empty space that plain vacuum has made reusable but not returned to the OS. Tables and indexes bloat independently; index bloat is often worse because B-tree pages that become sparsely populated are not merged.

Causes, roughly in order of frequency:

1. Autovacuum can't keep up with update/delete rate (scale factor too lazy, cost limit too low, too few workers).
2. Long-running transactions / idle-in-transaction sessions / stale replication slots pinning the removal horizon.
3. Mass `UPDATE`/`DELETE` patterns (e.g. rewriting a whole table in place — every row now exists twice).
4. Queue-like tables (constant insert+delete at the head) — the classic pathological workload.

Detection and repair:

```sql
-- Rough signal from stats:
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

- Index bloat: `REINDEX INDEX CONCURRENTLY idx_name;` (PG 12+) rebuilds without blocking writes.
- Table bloat: `VACUUM FULL` rewrites the table but takes an `ACCESS EXCLUSIVE` lock for the duration — rarely acceptable online. The production tool is **`pg_repack`**, which rebuilds the table concurrently using triggers and a short final lock swap.
- Prevention beats repair: tune per-table autovacuum, keep transactions short, and use partitioning so old data is removed by `DROP PARTITION` (instant, no bloat) instead of `DELETE` (creates dead tuples that vacuum must chase).

## 9. Locking essentials

MVCC means readers and writers never block each other, but writers still lock what they write:

- **Row locks** — `UPDATE`/`DELETE`/`SELECT ... FOR UPDATE` take exclusive row locks; `FOR SHARE`/`FOR NO KEY UPDATE` are the weaker variants. Postgres has **no gap locks** — nothing like InnoDB's next-key locking exists, because phantom prevention comes from snapshots (RR) or SSI (SERIALIZABLE), not from locking ranges. Row locks are stored *on the tuple itself* (via `xmax` and the multixact machinery), not in a lock table, so locking a million rows costs no lock memory — but does dirty a million tuples.
- **Table locks** — every statement takes some table-level lock; most conflict with nothing you care about. The dangerous one is `ACCESS EXCLUSIVE` (taken by `VACUUM FULL`, most `ALTER TABLE` forms, `TRUNCATE`, `DROP`). The classic incident is not the lock itself but the **lock queue**: an `ALTER TABLE` waits behind one long-running `SELECT`, and every subsequent query on that table — including plain reads — queues behind the `ALTER`. Always run DDL with a `lock_timeout`:

```sql
SET lock_timeout = '2s';
ALTER TABLE orders ADD COLUMN note text;   -- bounces instead of freezing the app
```

- **Deadlocks** — detected after `deadlock_timeout` (1s) by walking the wait-for graph; one transaction is aborted with error `40P01`. Prevention is the same as everywhere: lock rows in a consistent order, keep transactions short.
- **Advisory locks** — application-defined locks on arbitrary 64-bit keys (`pg_advisory_lock(id)`, `pg_try_advisory_xact_lock(id)`), unrelated to any table data. Standard uses: singleton cron jobs across app instances, per-entity mutexes without a lock table. Session-scoped advisory locks are incompatible with transaction-mode PgBouncer; use the `_xact_` variants.
- Diagnose blocking live with `pg_stat_activity` (`wait_event_type = 'Lock'`) joined through `pg_blocking_pids(pid)`.

## 10. Quick reference

- Postgres = process per connection + shared buffers; pool connections externally.
- Heap storage, no clustered index; every index points to a physical TID.
- `UPDATE` = insert new version + mark old dead; HOT avoids index churn when no indexed column changes.
- Vacuum removes dead tuples, maintains the visibility map (index-only scans), freezes XIDs (wraparound), updates the FSM.
- Long transactions and stale replication slots are the root cause behind most bloat and wraparound incidents.
- WAL is the single source of durability, replication, PITR, and logical decoding.
- TOAST moves >2 KB values out of line; large-value reads and JSONB rewrites have hidden costs.
- No gap locks; row locks live on tuples (unlimited count, no lock memory); DDL needs `lock_timeout` because of the lock queue; advisory locks for app-level mutexes.
- Know the version markers: PG 17 = vacuum memory rework, incremental backup; PG 18 = async I/O, `uuidv7()`.
