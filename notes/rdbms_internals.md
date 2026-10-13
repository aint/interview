# SQL Cheat Sheet

## Table of Contents
1. [Planner Mental Model](#planner-mental-model)
2. [Execution Plans: What to Read First](#execution-plans-what-to-read-first)
3. [Statistics & Cardinality Estimation](#statistics--cardinality-estimation)
4. [Indexing Strategy](#indexing-strategy-beyond-add-index)
5. [Transactions, MVCC, and Isolation](#transactions-mvcc-and-isolation)
6. [Write Path: WAL & Durability](#write-path-wal--durability)
7. [Locking, Contention, and 2PL](#locking-contention-and-2pl)
8. [Schema Evolution & Migrations](#schema-evolution--migrations)
9. [Reliability, Replication, and Failover](#reliability-replication-and-failover)
10. [Performance Tuning Workflow](#performance-tuning-workflow)
11. [Interview Drill Questions](#interview-drill-questions)

---

SQL performance is mostly an I/O budget: read fewer pages to answer the query, and write fewer pages to store the change (heap, indexes, WAL). Stats, indexes, join choice, and durability settings are all ways of spending or saving that I/O.

---

## Planner Mental Model

Think in 3 stages:

1. **Parse / rewrite**
   - **Parse** turns SQL text into a tree. Syntax errors and unknown names fail here. No data is read.
   - **Rewrite** produces an equivalent tree the planner would rather cost: views expanded, simple subqueries inlined, predicates pushed down, constants folded. A rewrite can enable an index (predicate pushed onto a base table) or hide one (column wrapped in a function).
2. **Plan**: build candidate plans — join order, access method (seq scan, index scan, index-only, bitmap), join algorithm (nested loop, hash, merge). Score each as estimated I/O + CPU from statistics. The cheapest estimate wins, including when that means ignoring an index a human would have picked.
3. **Execute**: operators pull rows from their children. The basic executor does not re-plan when actual row counts diverge; a bad cardinality low in the tree poisons every choice above it. Some engines adapt mid-query, but don't assume it.

Cost-based planners optimize for estimated I/O + CPU, not for "human intuition."

Key implications:
- Wrong row-count estimates -> wrong plan choice.
- "Fast on small data" plans often collapse at scale.
- Query shape, statistics, and indexes all influence the same planner decision tree.

---

## Execution Plans: What to Read First

Use `EXPLAIN`/`EXPLAIN ANALYZE` (or DB equivalent).

Read top-down for structure, then validate bottom-up with actuals:

- **Biggest red flag**: estimated rows vs actual rows mismatch (10x+ is often bad).
- **Access path**: sequential scan, index scan, index-only scan, bitmap scan.
- **Join algorithm**: nested loop, hash join, merge join.
- **Expensive operators**: sort, hash aggregate, hash join build side.
- **Timing split**: where most time is actually spent.

If estimates are wrong, tuning indexes alone may not help until stats/data distribution issues are fixed.

---

## Statistics & Cardinality Estimation

Optimizer quality depends on statistics quality. The index-vs-scan decision is a cardinality decision:

- **Few rows expected** → index lookup. A handful of random reads is cheaper than walking the heap.
- **Many rows expected** → sequential scan. Reading each page once, in order, beats random heap fetches for a large fraction of the table.
- **Histograms and most-common values (MCVs)** are how the planner sees skew. A uniform guess gives a rare status and a hot status the same selectivity, so one of those predicates gets the wrong access path.

Core concepts:
- **Cardinality**: expected row count after each filter/join.
- **Selectivity**: fraction of rows matching a predicate.
- **Histograms / MCVs / correlation**: planner's view of distribution.

Common failure cases:
- Highly skewed columns (hot values).
- Correlated predicates across multiple columns.
- Stale stats after large writes/deletes.
- Functions/casts on indexed columns that hide selectivity.

Practical levers:
- Refresh stats after bulk loads, large deletes, or when estimated vs actual rows diverge. `EXPLAIN` does not update stats.
  - **Postgres**: `ANALYZE tbl;` (optionally `ANALYZE tbl (col);`). Autovacuum also auto-analyzes once `n_mod_since_analyze` crosses its threshold; check `last_analyze` on `pg_stat_user_tables`. For a skewed column, `ALTER TABLE tbl ALTER COLUMN c SET STATISTICS n;` then `ANALYZE` (default target is 100).
  - **MySQL 8**: `ANALYZE TABLE tbl;` refreshes index stats (InnoDB also recalculates after enough rows change). Column skew needs an explicit histogram: `ANALYZE TABLE tbl UPDATE HISTOGRAM ON c WITH n BUCKETS;`. Histograms do not track writes until you rebuild them.
- Use multicolumn stats for correlated predicates (engine-specific: Postgres `CREATE STATISTICS`; a single-column histogram will not see that `city` and `country` move together).

---

## Indexing Strategy

An index speeds a query by cutting the pages read: the tree locates matching keys instead of scanning the heap. Every secondary index is paid for on the write path — extra pages stored, and extra I/O on insert, update, and delete to maintain it.

### Clustered vs non-clustered

- **Clustered index** (InnoDB primary key, SQL Server clustered index): the table *is* the index. Rows live in key order. There is one per table. A lookup by the clustered key reads the row directly.
- **Non-clustered / secondary index**: a separate structure. Postgres stores a heap and points each index entry at a tuple id. InnoDB secondary entries store the primary key, so a lookup is secondary index → PK → clustered index (two structures, often two random reads).
- **Covering / index-only scan** avoids that second read when every column the query needs is in the index (Postgres also needs the visibility map to show the page is all-visible).
- Postgres has no maintained clustered index. `CLUSTER` rewrites the heap into index order once; later writes do not preserve it.

### B-tree fragmentation

Leaf pages split when they fill. How keys arrive decides whether those pages stay full:

- **Ordered inserts** (monotonic id, timestamp) append at the right edge. Pages stay packed, but the rightmost leaf is a latch hotspot under concurrent inserts.
- **Random inserts** (UUID v4, hash keys) split pages all over the tree. Split leaves are about half empty, so the index uses more pages, caches worse, and each lookup does more I/O. On InnoDB this also randomizes the clustered table: the row itself is written to a random page.
- Half-empty pages stay that way until a rebuild (`REINDEX`, `OPTIMIZE TABLE`, `pg_repack`). A `fillfactor` below 100 leaves room for in-page updates so a hot row doesn't split on every change (and, in Postgres, so HOT updates can avoid touching indexes when indexed columns don't change).

### Selectivity

A secondary index pays off only when it returns few rows. Each hit is typically a random heap (or PK) read. If the predicate matches a large fraction of the table, those random reads touch most pages anyway, and a clustered/sequential scan — each page once, in order — is cheaper. The planner is right to ignore the index.

- A boolean or status column with a common value is this case. An index on `is_deleted` does nothing for `is_deleted = false` when almost every row is live.
- The rare value can still deserve an index. A **partial index** (`WHERE is_deleted`) stays small and selective; the common value keeps using the sequential scan.
- Low selectivity is also why "index every column" fails: write amplification and cache pressure, with the planner skipping the indexes on the queries you added them for.

### B-tree baseline
- Best for equality, range, prefix lookups, and ordered retrieval.
- Leftmost-prefix rule applies for multicolumn indexes.

### Composite index design
- Put high-selectivity and frequently filtered columns first.
- Align column order with common `WHERE` + `ORDER BY` patterns.
- Avoid over-wide composites that are rarely used as a whole.

### Covering indexes
- Include/select needed columns to avoid table lookups.
- Great for read latency, but increase write amplification and storage.

### Partial/filtered indexes
- Index only hot subset (e.g., `status = 'ACTIVE'`).
- Great when workload targets a stable subset of rows.

### Specialized indexes
- Hash (engine-specific) for exact lookup.
- GIN/GiST/BRIN or equivalents for JSON/full-text/geospatial/time-series cases.

### Index anti-patterns
- Indexing every column.
- Duplicate/redundant indexes.
- Indexes on low-cardinality columns with poor selectivity.
- Ignoring write cost (insert/update/delete gets slower with each extra index).

---

## Transactions, MVCC, and Isolation

### MVCC (Multi-Version Concurrency Control)

Readers and writers mostly avoid blocking each other by keeping **multiple row versions** instead of overwriting in place.

**How it works (Postgres-style):**
1. `UPDATE`/`DELETE` does not remove the old row — it marks it dead and inserts/writes a new version.
2. Each transaction sees a **snapshot** of committed data as of txn start (or statement, depending on isolation).
3. Old versions are reclaimed later by **vacuum** / purge — long-running txns delay cleanup → bloat.

**MVCC ≠ no locking:** MVCC handles **read/write** concurrency; **write/write** and explicit `SELECT FOR UPDATE` still use locks (see [2PL](#locking-contention-and-2pl)).

### Isolation levels (practical)

Each level defines which anomalies a transaction may observe. Stronger levels widen the snapshot or take more locks, so they block or abort more often. Pick from the anomalies, not from the level name — Postgres Repeatable Read and InnoDB Repeatable Read are not the same mechanism.

| Level | What you get | Anomalies still allowed |
|-------|--------------|-------------------------|
| **Read Committed** | New snapshot **per statement** (Postgres and InnoDB) | Non-repeatable read, phantom, write skew. Dirty reads are blocked. |
| **Repeatable Read** | One snapshot for the txn. **Postgres**: snapshot isolation, so phantoms are hidden too. **InnoDB**: snapshot at first consistent read, plus **next-key locks** on locking reads to block phantoms. | **Write skew** on both. ANSI Repeatable Read (row locks only, no gap locks) still allows phantoms. |
| **Serializable** | A schedule equivalent to some serial order. Postgres uses SSI and **aborts** one transaction; InnoDB takes shared next-key locks on plain reads. | None of the above. Cost is retries or lock waits. |

**Read Uncommitted** allows dirty reads. InnoDB implements it; Postgres accepts the syntax and runs Read Committed instead. Defaults: Postgres **Read Committed**, InnoDB **Repeatable Read**.

### Anomalies (name them in interviews)

- **Dirty read** — read uncommitted data from another txn. Stopped at Read Committed and above.
- **Non-repeatable read** — same row returns different committed values later in one txn. Stopped at Repeatable Read (statement snapshot refreshed only at Read Committed).
- **Phantom read** — same predicate returns a different row set because another txn inserted or deleted. Stopped by Postgres Repeatable Read (the snapshot never sees the new row) and by InnoDB next-key locks on locking reads. Still possible at Read Committed, and at ANSI Repeatable Read without gap locks.
- **Write skew** — two txns read overlapping state and write **disjoint** rows, so neither row lock conflicts, and an invariant breaks (two doctors both seeing the other on call, both going off). Survives snapshot isolation, including Postgres Repeatable Read. Needs Serializable, an explicit lock, or a constraint that makes the writes conflict.

### Senior-level tradeoff

Pick the **weakest isolation** that preserves business invariants. Use constraints (`UNIQUE`, `CHECK`, FK), `SELECT ... FOR UPDATE`, or optimistic retry where stronger isolation is too expensive.

Capacity angle: MVCC lets **high read QPS** coexist with writes on the same rows (old versions served to readers). Write TPS still capped by WAL and locks — see [data_store_selection.md](./data_store_selection.md#why-single-node-postgresmysql-write-tps-caps-10k40k).

---

## Write Path: WAL & Durability

The **Write-Ahead Log (WAL)** is how Postgres/MySQL satisfy the **D** in ACID: changes are appended to a sequential log **before** data files are considered durable.

### Write sequence (simplified)

```
1. BEGIN
2. Modify buffer pool pages (in memory)
3. Append WAL records for those changes
4. COMMIT → fsync WAL to disk (durability point)
5. Later: checkpoint flushes dirty pages to data files (async batch)
```

Client gets **ack after step 4**, not after data files are written. Crash recovery **replays WAL** to reconstruct committed work.

### Why WAL caps write TPS

| Factor | Effect |
|--------|--------|
| **`fsync` / group commit** | Each durable commit waits for log flush to storage; NVMe has finite fsync/s |
| **Sequential log I/O** | Fast append, but commit latency bounded by disk sync latency |
| **Replication sync** | `synchronous_commit` / semi-sync replica waits for remote flush → higher latency, lower TPS |
| **Checkpoint spikes** | Dirty page flush can add I/O jitter under sustained write load |

**Group commit:** DB batches multiple transactions into one `fsync` — trades a little latency for higher throughput.

**Tuning (know trade-offs, don't hand-wave in prod):**
- Postgres `synchronous_commit=off` — faster commits; risk last ~few hundred ms of commits on crash.
- Relaxed `full_page_writes` / durability settings — engine-specific; never default for money/orders.

---

## Locking, Contention, and 2PL

### Two-phase locking (2PL)

Conflicting writes need a serial order. **Strict 2PL** is how engines get one for the rows they lock:

1. **Growing phase** — acquire locks as statements touch rows (row, gap, or next-key, depending on engine and isolation). No lock is released in this phase. Acquisition is incremental: the engine does not know the full set up front, so it locks what each statement actually hits.
2. **Shrinking phase** — at **commit or rollback**, every lock drops together. Plain 2PL would allow releases earlier; databases don't, because a reader or writer could then depend on a change that still might roll back, and the serial order would no longer hold.

The serial order is the order in which transactions finish growing (each holds everything it needs). Locks held until commit are why one slow transaction blocks writers of those rows. Incremental acquisition plus hold-until-commit is also why deadlocks exist: each side already holds a lock the other wants, and neither can release early.

### `SELECT ... FOR SHARE` vs `SELECT ... FOR UPDATE`

Plain `SELECT` is a snapshot and takes no row lock (Postgres). These two are **locking reads**: they pin the current row until the transaction ends. The difference is who else may lock it.

| | `FOR SHARE` | `FOR UPDATE` |
|--|-------------|--------------|
| **Mode** | Shared. Other `FOR SHARE` on the same row is allowed. | Exclusive. Conflicts with `FOR SHARE`, `FOR UPDATE`, and DML. |
| **Blocks** | `UPDATE` / `DELETE` / `FOR UPDATE`. | Every other locker and every writer of those rows. |
| **Use when** | The row must not change or vanish, and this transaction is not the writer (read a parent while inserting children; FK-style checks). | This transaction will `UPDATE` or `DELETE` the row, or must be the only one deciding. |
| **Failure mode** | Two transactions `FOR SHARE`, then both `UPDATE`: the lock upgrade deadlocks and one is aborted. | The second locker waits. `NOWAIT` errors immediately; `SKIP LOCKED` skips rows already locked. |

MySQL spelling: `FOR SHARE` (8.0) and older `LOCK IN SHARE MODE` are the same shared lock. Under InnoDB Repeatable Read, locking reads are **next-key** locks (the row plus the gap after it) unless the search is a unique equality, which is what blocks phantoms.

Postgres has two weaker modes worth naming: `FOR KEY SHARE` (what FK checks take — blocks deletes and key changes, not other column updates) and `FOR NO KEY UPDATE` (a column update that leaves the key alone).

| Lock mode | Typical use |
|-----------|-------------|
| **Shared (S)** | `FOR SHARE` / `LOCK IN SHARE MODE` — other shared lockers allowed, writers not |
| **Exclusive (X)** | `FOR UPDATE`, `INSERT`/`UPDATE`/`DELETE` — no concurrent writer or shared locker |

**MVCC + 2PL together:** plain `SELECT` uses snapshot (no S lock on row in Postgres). `UPDATE` takes **X lock** on the row version / key; concurrent `UPDATE` on same row **blocks** until first txn ends.


### Deadlocks

Cycle of txns waiting on each other's locks → engine aborts **one victim** (retry in app). Not a "bug" — symptom of concurrent access order.

### DDL locks

`ALTER TABLE`, index build — **metadata / table-level** locks can block or be blocked by DML. Use online/concurrent DDL where supported; plan migrations off peak.

---

## Schema Evolution & Migrations

Zero-downtime mindset:

1. Additive change first (new nullable column/table/index).
2. Dual-write or backfill in batches.
3. Switch reads to new path.
4. Remove old path in a later deploy.

Rules of thumb:
- Avoid long blocking DDL in peak hours.
- Build large indexes concurrently/online if supported.
- Backfills must be chunked and resumable.

### Online schema change

Use a native metadata-only change when the engine has one. When `ALTER` rewrites the table, an online tool copies in the background and takes a brief lock only at cutover.

- **MySQL native**: `ALGORITHM=INSTANT` for some `ADD COLUMN` forms (dictionary change, no row rewrite). `INPLACE` can rebuild an index while allowing DML. Prefer `ALGORITHM=INSTANT, LOCK=NONE` so the statement errors instead of taking a stronger lock than you expected.
- **pt-online-schema-change** (Percona Toolkit): creates a shadow table, installs triggers so live writes hit both, copies rows in chunks, then `RENAME`s. Budget ~2× disk and extra write load from the triggers. Foreign keys and existing triggers need an explicit plan; cutover still takes a short metadata lock.
- **gh-ost** (GitHub): same shadow table, but it applies ongoing changes by tailing the **binary log** instead of triggers, and it throttles when replica lag grows. Prefer it when trigger overhead on the live table is the problem.
- **Postgres**: `CREATE INDEX CONCURRENTLY` builds without blocking reads and writes (it cannot run inside a transaction block). `ADD COLUMN ... DEFAULT <constant>` is metadata-only since v11. Type changes and most table rewrites are not; **pg_repack** rebuilds a table or index online, with a short lock at the swap.

---

## Reliability, Replication, and Failover

Core concepts:
- Primary/replica lag affects read-your-write behavior.
- Async replication risks data loss on failover window.
- Sync/quorum replication improves durability but adds latency.

### RPO and RTO

- **RPO (recovery point objective)** — how much **committed** data you can lose, expressed as time. With async replication, RPO equals replica lag at the moment of failure: failover promotes a replica missing the last seconds or minutes of commits. Sync or quorum commit pushes RPO near zero, because the client is not acked until the required replicas have the WAL.
- **RTO (recovery time objective)** — how long the service stays down. Automatic failover (Patroni, Orchestrator, managed failover) aims at seconds. Manual promotion, DNS, and pool drain stretch it to minutes.

Trade-off: a tighter RPO (sync commit, more replicas that must ack) raises commit latency and lowers write TPS. A tighter RTO (hot standby, automatic promotion) adds failover machinery and split-brain risk if the old primary is not fenced. Async replication plus fast promotion is the common pair — short RTO, nonzero RPO. A path that cannot lose an acknowledged commit (payments, inventory decrement) needs sync replication or a client that can reconcile, not a faster promote of a lagging replica.

Interview angle:
- State the RPO and RTO you are buying with the replication mode, not just "we have replicas."
- Decide where strong consistency is required vs where eventual is acceptable.
- Route read traffic with awareness of replica lag and staleness tolerance.

---

## Performance Tuning Workflow

Use this order to avoid random tuning:

1. **Measure**: identify top queries by total load and p95/p99 latency.
2. **Plan inspection**: find wrong estimates, bad scans, bad joins, spills.
3. **Fix root cause**: stats, query shape, or indexing strategy.
4. **Re-measure**: validate latency + resource impact.
5. **Guardrail**: add regression checks (dashboards, query budgets, tests).

Do not tune single query latency in isolation if it hurts global throughput/concurrency.

---

## Interview Drill Questions

- How does a cost-based optimizer choose between index scan and full scan?
- Why can the same query flip plans over time in production?
- When is a composite index better than multiple single-column indexes?
- Why do cardinality estimation errors cause catastrophic join choices?
- How do you debug a query that became slow after data growth without code changes?
- Explain MVCC: how can a reader see old data while a writer updates the same row?
- What's the difference between MVCC snapshot reads and `SELECT FOR UPDATE`?
- Walk through WAL on commit — what happens before the client gets ACK?
- Why does `fsync` cap Postgres write TPS even with spare CPU?
- What is 2PL growing/shrinking phase? When do row locks block?
- What isolation level would you pick for payment/state transitions and why?
- How do you roll out a breaking schema change with zero downtime?
- What causes deadlocks, and how do you reduce their frequency?
- How do replica lag and failover strategy affect API consistency guarantees?
