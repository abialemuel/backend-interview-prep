# Transactions and Isolation

This file covers ACID, the four SQL isolation levels and the anomalies each prevents, and InnoDB-specific behavior that interviewers expect you to know (next-key locking, MVCC, locking reads, deadlocks). It also covers the redo/undo/binlog subsystem and the two-phase commit that ties them together.

## 1. ACID

- **Atomicity** — a transaction is all-or-nothing. Either every statement commits or none do. InnoDB implements this with the **undo log**: old row versions are recorded before changes; a `ROLLBACK` (or crash before commit) undoes them.
- **Consistency** — the database moves from one valid state to another. Consistency is partly the application's responsibility (constraints enforce it: `NOT NULL`, foreign keys, `CHECK`, uniqueness), and partly a consequence of A, I, D plus constraints.
- **Isolation** — concurrent transactions appear to run one at a time, with respect to the chosen isolation level. InnoDB implements this with **locks** plus **MVCC**.
- **Durability** — once committed, a transaction survives a crash. InnoDB uses the **redo log** (write-ahead logging) and `innodb_flush_log_at_trx_commit=1` (default after MySQL 8.x) to guarantee this.

## 2. Transaction lifecycle

```sql
-- autocommit is ON by default. With it ON, every statement is its own transaction, auto-committed.
SET autocommit = 0;   -- explicitly start a "transaction"

-- Preferred explicit begin
START TRANSACTION;    -- or BEGIN (alias except inside stored routines, where BEGIN is a block marker)

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

SAVEPOINT before_audit;
INSERT INTO audit_log(...) VALUES (...);

-- If something goes wrong mid-audit:
ROLLBACK TO SAVEPOINT before_audit;   -- undoes back to the savepoint, transaction still open

COMMIT;   -- or ROLLBACK; to undo everything since BEGIN
```

`autocommit=1` is the default; with it on, wrapping several statements in `START TRANSACTION ... COMMIT` is the everyday pattern. Incidentally, DDL statements (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`) **implicitly commit** before and after themselves — they cannot be rolled back. This is a famous gotcha: you cannot run `BEGIN; INSERT...; ALTER; ROLLBACK;` and expect the `INSERT` to roll back.

## 3. InnoDB is the transactional engine

InnoDB is the only fully transactional engine bundled in modern MySQL. **MyISAM is legacy, non-transactional, table-locked, and lacks crash recovery by WAL** — mention it only for contrast. Default engine since 5.5; only data dictionary support since 8.0 (no MyISAM system tables).

## 4. The four isolation levels and the anomalies they prevent

The SQL standard defines four levels. InnoDB supports all four. Set per-session or globally:

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET GLOBAL  TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT @@transaction_isolation;   -- MySQL 8.0+ (was tx_isolation, now removed)
```

| Level | Dirty read | Non-repeatable read | Phantom read | InnoDB default? |
|-------|-----------|---------------------|--------------|------------------|
| READ UNCOMMITTED | Possible | Possible | Possible | No |
| READ COMMITTED | Impossible | Possible | Possible | No |
| REPEATABLE READ | Impossible | Impossible | Possible (per SQL std) | **Yes (default)** |
| SERIALIZABLE | Impossible | Impossible | Impossible | No |

### An anomaly walkthrough

Setup:

```sql
CREATE TABLE accounts (id INT PRIMARY KEY, balance INT) ENGINE=InnoDB;
INSERT INTO accounts VALUES (1, 100), (2, 100);
```

**Dirty read** — T1 reads an uncommitted change from T2:
- T2: `SET ... READ COMMITTED, but T2 ignores it for a moment; T2: UPDATE accounts SET balance=50 WHERE id=1;` (not committed)
- T1 (under READ UNCOMMITTED): `SELECT balance FROM accounts WHERE id=1;` → 50
- T2: `ROLLBACK` — T1 saw a value that never existed.

**Non-repeatable read** — T1 reads the same row twice and gets different values because T2 committed a change between:
- T1: `SELECT balance FROM accounts WHERE id=1;` → 100
- T2: `UPDATE ... SET balance=50; COMMIT;`
- T1: `SELECT balance FROM accounts WHERE id=1;` → 50  (under READ COMMITTED)
- Under REPEATABLE READ, T1's second `SELECT` still returns 100.

**Phantom read** — T1 runs a range query twice; T2 inserts a row that matches the range between the two:
- T1: `SELECT * FROM accounts WHERE balance >= 100;` → returns 2 rows
- T2: `INSERT INTO accounts VALUES (3, 500); COMMIT;`
- T1: `SELECT * FROM accounts WHERE balance >= 100;` → returns 3 rows (under standard REPEATABLE READ, which is what the SQL standard specifies)

### InnoDB REPEATABLE READ prevents phantoms (InnoDB-specific — important)

Per the SQL standard, REPEATABLE READ allows phantom reads. **InnoDB's REPEATABLE READ does not** — it uses **next-key locking** (a combination of record locks and gap locks) for locking reads and writes that touch a range, preventing other transactions from inserting into the locked gaps. Combined with MVCC for `SELECT` (plain, non-locking reads), a snapshot is reused for all reads in the transaction, so phantoms are blocked.

Concretely, under InnoDB RR:

```sql
-- T1
START TRANSACTION;
SELECT * FROM accounts WHERE balance >= 100 FOR UPDATE;  -- locks range [100, +inf)
-- T2
INSERT INTO accounts VALUES (3, 500);   -- BLOCKED (waits on the gap lock)
```

So **InnoDB RR ≈ SERIALIZABLE** in practice, with two caveats:
- Plain `SELECT` uses MVCC snapshots, never blocking writes — readers and writers do not see each other's uncommitted changes but do not block either. SERIALIZABLE upgrades plain `SELECT` to `SELECT ... LOCK IN SHARE MODE` (effectively acquiring shared next-key locks).
- InnoDB RR uses gap locks under locking reads/writes; SERIALIZABLE additionally locks with shared next-key on every read.

**SERIALIZABLE in InnoDB** is essentially RR + every `SELECT` becomes a locking read. Heavy and rarely used in production; most shops stay at the default RR.

## 5. MVCC (Multi-Version Concurrency Control)

InnoDB's MVCC provides each transaction with a **snapshot** of the database, established at first read (under RR) or per-statement (under RC). Key data structures:

- **Undo log**: for every change, the previous version of the row is written to the undo log. Row headers carry a pointer (`ROLL_PTR`) to the most recent undo record. Versions thus form a chain backwards in time.
- **Read view**: when a transaction starts a statement (RC) or the transaction itself (RR), InnoDB creates a "read view" — a snapshot of which active transactions are currently visible. A row version is visible to a transaction only if it was committed before the transaction's read view, by the visibility rules over the version's transaction ID (`TRX_ID`).
- Readers do not acquire row locks for plain `SELECT` and therefore do not block writers. Writers, in turn, see only the latest committed version (or their own uncommitted changes within the same transaction).
- Old undo records are purged by a background thread once no active transaction still needs them. Long-running transactions delay purge → undo log bloat → known performance pitfall. Avoid long-running, read-only transactions inside a heavy-write workload.

### Visibility sketch

Assume transactions get monotonic IDs. A row version V created by transaction `T_v` is visible to a read view R if:

```
(V.trx_id < R.min_active_trx_id)                          -- committed before R started, OR
(V.trx_id == R.creator_trx_id)                            -- read-your-own-writes, OR
(V.trx_id not in R.active_trx_set  AND  V.trx_id < R.max_trx_id)
```
Otherwise the version is invisible → InnoDB walks the undo chain to find a visible predecessor. If none exists in the undo log, "snapshot too old" style failure is not surfaced in MySQL but the read returns the latest visible version available.

## 6. Locking reads

Plain `SELECT` is consistent (snapshot) read under RR/RC. Locking reads bypass the snapshot and pull in current data with locks:

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;   -- X lock (write intent)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;     -- S lock (was LOCK IN SHARE MODE, deprecated alias in 8.0)
```

- `FOR UPDATE` acquires **exclusive next-key locks** on rows/index entries matching the predicate. Blocks other `FOR UPDATE` and `FOR SHARE` and `UPDATE` on the same rows.
- `FOR SHARE` acquires **shared next-key locks**. Multiple `FOR SHARE` can coexist, but blocks `FOR UPDATE`/`UPDATE`/`DELETE`.

Locking reads are essential for the classic pattern "read balance, check, then update":

```sql
START TRANSACTION;
SELECT balance INTO @b FROM accounts WHERE id = 1 FOR UPDATE;
IF @b >= 100 THEN
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
END IF;
COMMIT;
```

Without `FOR UPDATE`, two concurrent transactions could each read 100 and both allow the withdrawal → a race.

`NOWAIT` and `SKIP LOCKED` (MySQL 8.0+) let a locking read fail fast or skip locked rows:

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;       -- error 3572 if locked
SELECT * FROM jobs WHERE status = 'PENDING' FOR UPDATE SKIP LOCKED LIMIT 5;  -- work queue pattern
```

`SKIP LOCKED` is the canonical way to build a job queue without contention overhead.

## 7. Record, gap, and next-key locks

- **Record lock** — locks a single index record (the row at that index entry).
- **Gap lock** — locks the *gap* between index records, preventing inserts into the gap. Does not lock any actual row.
- **Next-key lock** — a record lock on index record R plus the gap before R. Equivalent to locking `[prev, R]`. This is what InnoDB uses at REPEATABLE READ for range predicates.
- **Insert intention lock** — a special gap lock that lets multiple inserts into the same gap proceed if their actual keys do not collide.

Gaps exist *on the index*. If your `UPDATE`/`SELECT FOR UPDATE` predicate is not indexed, InnoDB locks every row of the table → effectively a table lock. **Always index the columns used by locking reads and updates.**

Locking behavior differs by isolation level:
- **READ COMMITTED**: no gap locks (except for foreign-key constraint checks and unique checks); only record locks. This reduces locking contention, but allows phantoms.
- **REPEATABLE READ**: full gap/next-key locking as described above.

## 8. Deadlocks

A deadlock is a cycle in the wait-for graph of lock holders. Classic example: two transactions updating the same two rows in opposite order.

```sql
-- T1
START TRANSACTION;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;   -- holds lock on 1
-- T2
START TRANSACTION;
UPDATE accounts SET balance = balance - 10 WHERE id = 2;   -- holds lock on 2

-- T1
UPDATE accounts SET balance = balance + 10 WHERE id = 2;   -- waits for T2's lock on 2
-- T2
UPDATE accounts SET balance = balance + 10 WHERE id = 1;   -- waits for T1's lock on 1 → CYCLE
```

**InnoDB has automatic deadlock detection** (`innodb_deadlock_detect=ON`, default). The engine builds the wait-for graph on each lock acquisition and rolls back the smaller-cost transaction (the one with fewer undo log records) with error 1213 (`ER_LOCK_DEADLOCK`). The application must retry.

Tuning:
- For high-contention workloads, deadlock detection has O(n^2) lock-graph cost; you can disable it (`innodb_deadlock_detect=OFF`) and rely on `innodb_lock_wait_timeout` — rare and only with MySQL support guidance.
- You can monitor deadlocks via `SHOW ENGINE INNODB STATUS` (section `LATEST DETECTED DEADLOCK`) and the `information_schema.INNODB_METRICS` deadlock counter.

### How to design to avoid deadlocks

1. **Consistent lock ordering.** Always touch rows/table-in-franchise in the same canonical order across all transactions. With fixed order, no cycles can form.
2. **Keep transactions short.** Long transactions hold locks longer and broaden the cycle surface area.
3. **Lock via index access.** Without an index the engine locks whole tables → instant global contention and deadlocks.
4. **Use lower isolation when safe.** READ COMMITTED uses no gap locks, reducing lock footprint (accept phantoms).
5. **Batch with `SKIP LOCKED`** for work queues.
6. **Add retry** around deadlocked transactions (`ER_LOCK_DEADLOCK`, `ER_LOCK_WAIT_TIMEOUT`).

## 9. Optimistic vs pessimistic locking

**Pessimistic** — assume conflicts will happen, take locks upfront:

```sql
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- prepare to write
-- logic, then UPDATE ... COMMIT;
```

Useful under high contention and a real cost to retries (e.g. financial). Downside: lock contention, deadlocks, reduced concurrency.

**Optimistic** — assume conflicts are rare, detect on commit:

- Read row + its `version` (or `updated_at`, or checksum).
- Compute the change in the application.
- At commit, write only if the version is unchanged:
```sql
UPDATE accounts SET balance = balance - 100, version = version + 1
WHERE id = 1 AND version = @seen_version;
-- if affected_rows == 0 → conflict, retry the whole transaction.
```

No row locks taken across the think time, so throughput scales with conflict rate. Use when conflicts are rare (e.g. updating user profile, large form submission). With high contention, optimistic degrades to retry storms; pessimistic wins.

Either pattern can be expressed in InnoDB — the engine is enough; no extra plugin is needed. Interviewers often expect you to recognize the optimistic-style CAS update as just an indexed `UPDATE` with a version column, not a special isolation feature.

## 10. Distributed transactions / XA

MySQL supports XA transactions across multiple resource managers (e.g. two databases):

```sql
XA START 'xid';
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
XA END 'xid';
XA PREPARE 'xid';
XA COMMIT 'xid';
```

XA is a two-phase commit (prepare → ready, then commit/rollback). It works but is **widely avoided** in MySQL deployments because:

- It holds locks across both phases; if a coordinator fails between prepare and commit, resources stay locked until recovery.
- Performance is low.
- Many ORMs and connection poolers handle it poorly.

In modern architectures, distributed coordination across MySQL is typically achieved via outbox patterns + asynchronous, idempotent events rather than XA, or by keeping the cross-system atomicity at the application layer (sagas). XA is a "know it exists" topic, not a "we'll deploy XA" topic for most shops.

## 11. The redo log

InnoDB modifies pages in the buffer pool in memory. Before flushing a modified page to disk, it writes a redo log record describing the change (write-ahead logging, WAL). The redo log is **sequential**, append-only, fixed-size (a circular file), and is fsynced at commit (when `innodb_flush_log_at_trx_commit=1`).

Crash recovery replays redo log forward from the last checkpoint to reconstruct in-memory state, then applies undo log to roll back uncommitted transactions.

Key variables:
- `innodb_flush_log_at_trx_commit=1` (default) — fsync the redo log at every commit. Required for ACID durability.
- `=0` or `=2` — relax durability for throughput, at the cost of potential committed-transaction loss on OS crash (not on MySQL crash).
- `innodb_redo_log_capacity` (MySQL 8.0.30+, replacing `innodb_log_file_size` + `innodb_log_files_in_group` in 8.4) — total redo log size; influences checkpoint frequency and recovery time.

## 12. The undo log

Stores pre-image of modified rows so that:
- `ROLLBACK` can undo the changes.
- MVCC can construct older versions for snapshot reads.
- Crash recovery can roll back transactions that were not committed at crash time.

A long-running transaction keeps the undo log segment with the oldest needed version alive → "purge lag". Long-running OLAP-style `SELECT` on a write-heavy OLTP workload is the classic cause of undo-tablespace bloat and reduced write throughput.

## 13. The binlog

The **binary log** is a server-layer, ordered log of all committed changes to the database. It is independent of the redo log (which lives at the engine layer). It is used for:

1. **Replication** — replicas read the binlog and apply it.
2. **Point-in-time recovery** — replay the binlog from a backup forward to a chosen timestamp.

Three formats:

| Format | What it stores | Trade-offs |
|--------|----------------|------------|
| `STATEMENT` | The SQL text of each statement | Small logs; but non-deterministic SQL (`NOW()`, `UUID()`, `AUTO_RANDOM`, `LIMIT` without `ORDER BY`) yields different results on replicas |
| `ROW` (recommended) | Per-row before/after image | Deterministic, safe; larger logs for bulk updates; default in MySQL 8.x for good reason |
| `MIXED` | Statement, with fallback to ROW when the statement is unsafe | Compromise; confusing to reason about |

```sql
SET GLOBAL binlog_format = 'ROW';     -- recommended
```

`binlog_row_image=MINIMAL` reduces row-image size by logging only changed columns + PK.

## 14. Two-phase commit between redo log and binlog

The binlog and InnoDB's redo log are coordinated by an internal **two-phase commit (2PC)** so that either both reflect a transaction, or neither does — guaranteeing crash-consistency between the engine and the replication stream:

1. InnoDB prepares the transaction (writes prepare record to redo log, fsyncs).
2. The server writes the transaction's binlog events (and rotates binlog if needed), fsyncs the binlog.
3. InnoDB commits: writes a `COMMIT` marker to the redo log.

On crash recovery, InnoDB scans the redo log: for each prepared-but-uncommitted transaction, it checks whether the binlog contains that transaction. If yes → commit. If no → rollback. This guarantees replicas never see a transaction that the primary did not durably commit (`sync_binlog=1` is recommended to make the binlog itself durable).

`sync_binlog=1` fsyncs the binlog at every commit; required for crash-safe replication. Combined with `innodb_flush_log_at_trx_commit=1`, you have full ACID + replication safety at the cost of one fsync per commit. Group commit (`binlog_transaction_dependency_history_size`, `binlog_transaction_compression`) mitigates this under load.

## 15. Putting it together: write path of a committed UPDATE

1. Buffer-pool lookup for the page; if not present, read from disk (`data file`).
2. Generate undo record (old row image) into undo log buffer.
3. Modify the page in the buffer pool; update the page's `LSN`.
4. Write redo log record(s) into the redo log buffer.
5. `COMMIT`:
   - Redo log buffer fsynced (WAL durability) at step 2PC-prepare.
   - Binlog written and fsynced.
   - Redo log commit record written.
6. Later, the modified page is asynchronously flushed from the buffer pool to the data file (page cleaner threads) — these flushes are always preceded by corresponding redo log records being on disk.

A write never blocks readers; readers consume the pre-image via the undo log if needed. Mutually exclusive writers serialize on the row lock.

## 16. Quick reference

| Concept | InnoDB mechanism |
|---------|------------------|
| Atomicity | Undo log + `ROLLBACK` |
| Isolation | Locks + MVCC snapshots |
| Durability | Redo log + WAL + `innodb_flush_log_at_trx_commit=1` |
| Default isolation | REPEATABLE READ, snapshot via MVCC |
| Phantom prevention at RR | Next-key locking (gap + record) on locking reads/writes |
| Reader-writer interaction | Readers do not block writers; writers block writers on the same row |
| Deadlock handling | Automatic detection, victim rollback with `ER_LOCK_DEADLOCK` |
| Range locks require index | Yes — unindexed predicate → table lock |
| Replication + crash safety | Two-phase commit between redo log and binlog, `sync_binlog=1` |
| Binlog format recommended | ROW |
| Avoid | Long-running OLAP transactions, XA, SERIALIZABLE under high concurrency |