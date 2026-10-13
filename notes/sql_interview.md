# SQL Interview Prep

What backend interviews test: write correct queries, design a schema, explain why a query is slow, and reason about concurrency. Postgres-flavored where engines differ; the ideas transfer. Throughput numbers and when to leave SQL live in [data_store_selection.md](./data_store_selection.md).

## Table of Contents

1. [Relational Model](#relational-model)
2. [How a Query Is Evaluated](#how-a-query-is-evaluated)
3. [NULL](#null)
4. [Joins](#joins)
5. [Aggregation](#aggregation)
6. [Window Functions](#window-functions)
7. [Subqueries and CTEs](#subqueries-and-ctes)
8. [Writes](#writes)
9. [Schema and Constraints](#schema-and-constraints)
10. [Normalization](#normalization)
11. [Indexes](#indexes)
12. [The Planner](#the-planner)
13. [Reading a Query Plan](#reading-a-query-plan)
14. [Transactions, Isolation, and Locks](#transactions-isolation-and-locks)
15. [Scaling](#scaling)
16. [Postgres vs MySQL](#postgres-vs-mysql)
17. [Patterns](#patterns)
18. [Drill Questions](#drill-questions)
19. [Quick Reference](#quick-reference)

---

## Relational Model

SQL is declarative: you describe the result, the planner picks the access path.

---

## How a Query Is Evaluated

Logical order (what aliases and filters are allowed to see):

```
FROM / JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  DISTINCT  →  ORDER BY  →  LIMIT
```

Window functions run after `HAVING` and before `DISTINCT`.

Consequences:

- `WHERE` cannot use a `SELECT` alias. Repeat the expression, or wrap it in a subquery.
- `ORDER BY` can use select aliases and ordinals.
- `HAVING` filters **groups**. `WHERE` filters **rows** before they are grouped.
- `DISTINCT` is a post-projection dedupe. A `GROUP BY` you already need is usually the clearer tool.

```sql
SELECT dept_id, COUNT(*) AS n
FROM employees
WHERE active          -- rows
GROUP BY dept_id
HAVING COUNT(*) > 5   -- groups
ORDER BY n DESC
LIMIT 10;
```

`UNION` dedupes and sorts. `UNION ALL` keeps duplicates and is the one to write unless you need set semantics. `INTERSECT` / `EXCEPT` (MySQL: `EXCEPT` from 8.0.31) are set difference and intersection.

---

## NULL

`NULL` means unknown. Any comparison with `NULL` is unknown, so it fails a `WHERE` filter.

| Expression | Result |
| --- | --- |
| `col = NULL` | unknown — matches nothing |
| `col IS NULL` | the correct test |
| `NULL OR TRUE` | true |
| `NOT IN (1, NULL)` | unknown for every row — the whole predicate matches nothing |
| `COUNT(*)` | counts rows |
| `COUNT(col)` / `SUM(col)` / `AVG(col)` | skip nulls |
| `UNIQUE (col)` | multiple nulls are allowed (Postgres and MySQL) |

`COALESCE(a, b, 0)` returns the first non-null. `NULLIF(a, b)` returns null when `a = b` (useful to avoid divide-by-zero).

Prefer `NOT EXISTS` over `NOT IN (subquery)`. One null in the subquery makes `NOT IN` reject every outer row.

---

## Joins

| Join | Keeps |
| --- | --- |
| `INNER` | Pairs that match |
| `LEFT` | Every left row; unmatched right columns are null |
| `RIGHT` | Mirror of `LEFT`. Rewrite as `LEFT` so the preserved side is obvious. |
| `FULL` | Both sides; unmatched columns null. MySQL has no `FULL OUTER JOIN` — emulate with `LEFT` `UNION ALL` a `RIGHT` anti-join. |
| `CROSS` | Cartesian product. Intentional when generating combinations; accidental when a join condition is missing. |

A **self-join** is the same table twice (`employees e JOIN employees m ON e.manager_id = m.id`).

**Filter placement on an outer join:** predicates on the preserved side belong in `WHERE`. Predicates on the optional side belong in `ON`. A `WHERE` test that rejects nulls on the optional side (`WHERE o.status = 'paid'`) drops unmatched left rows, and the join behaves as an inner join.

```sql
SELECT u.id, o.id
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
 AND o.status = 'paid'   -- optional-side filter stays in ON
WHERE u.active;          -- preserved-side filter stays in WHERE
```

**Semi-join** (“has at least one”): `EXISTS`. **Anti-join** (“has none”): `NOT EXISTS`. Both stop at the first match and stay correct when the inner side contains nulls.

---

## Aggregation

Every selected expression is either aggregated or listed in `GROUP BY`. Postgres enforces this. MySQL does too when `ONLY_FULL_GROUP_BY` is on (the 8.x default).

`WHERE` then group then `HAVING`:

```sql
SELECT user_id, SUM(amount) AS total
FROM payments
WHERE status = 'captured'
GROUP BY user_id
HAVING SUM(amount) > 1000;
```

`GROUP BY` collapses rows. A later join sees one row per group. If you need the original line items and a total, use a window, not a group.

Conditional aggregates avoid a second pass:

```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'paid') AS paid_orders,          -- Postgres
  SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END) AS paid_mysql   -- both
FROM orders;
```

---

## Window Functions

A window computes a value **per row** using a peer set, without collapsing the row.

```sql
function(...) OVER (
  PARTITION BY ...
  ORDER BY ...
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

| Function | Ties |
| --- | --- |
| `ROW_NUMBER()` | Unique rank. Arbitrary among ties. Use for “exactly N per group” and deduping. |
| `RANK()` | Ties share a rank; the next rank skips (`1, 2, 2, 4`). |
| `DENSE_RANK()` | Ties share a rank; the next rank does not skip (`1, 2, 2, 3`). “Second distinct salary.” |
| `LAG` / `LEAD` | Previous / next row inside the partition. |
| `SUM` / `AVG` / `COUNT` | Running or partition totals. |
| `NTILE(n)` | Bucket rows into `n` groups. |
| `FIRST_VALUE` / `LAST_VALUE` | Careful: default frame ends at the current row, so `LAST_VALUE` is often the current row. Use an explicit frame through `UNBOUNDED FOLLOWING`. |

`ORDER BY` inside `OVER` changes the default frame to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Equal sort keys are peers, so a running `SUM` jumps by the whole tie group. Use `ROWS` when you want a physical running total.

Top-N per group:

```sql
SELECT *
FROM (
  SELECT e.*,
         ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC, id) AS rn
  FROM employees e
) t
WHERE rn <= 3;
```

The tie-breaker (`id`) makes the result stable.

---

## Subqueries and CTEs

| Form | When |
| --- | --- |
| **Scalar subquery** | One value in the select list. Must return at most one row. |
| **`IN` / `EXISTS`** | Membership. `EXISTS` is null-safe and usually the better anti-join. |
| **Derived table** | A subquery in `FROM`. Required to filter on a window result (`WHERE rn = 1`). |
| **CTE (`WITH`)** | Name a step so the query reads top-down. |
| **Correlated subquery** | Inner query references the outer row. It can run once per outer row. Rewrite as a join when the plan shows a nested loop over a large outer set. |

Postgres 12+ **inlines** CTEs by default (the planner can optimize across them). `WITH x AS MATERIALIZED (...)` forces a fence: compute once, reuse. Older “CTEs are always optimization fences” is stale.

**Recursive CTE** walks a tree or graph:

```sql
WITH RECURSIVE chain AS (
  SELECT id, manager_id, name, 1 AS depth
  FROM employees
  WHERE id = 42
  UNION ALL
  SELECT e.id, e.manager_id, e.name, c.depth + 1
  FROM employees e
  JOIN chain c ON e.manager_id = c.id
)
SELECT * FROM chain;
```

Always have a stop condition (`depth`, or a path array that rejects cycles).

---

## Writes

```sql
INSERT INTO users (email, name) VALUES ('a@b.co', 'A');

UPDATE users SET name = 'B' WHERE id = 1;

DELETE FROM users WHERE id = 1;
```

`UPDATE` and `DELETE` without `WHERE` affect every row. In a transaction you can `ROLLBACK` and check `COUNT` first.

**Upsert** — one statement, so two sessions cannot both insert:

```sql
-- Postgres
INSERT INTO users (email, name)
VALUES ('a@b.co', 'A')
ON CONFLICT (email) DO UPDATE
SET name = EXCLUDED.name;

-- MySQL
INSERT INTO users (email, name)
VALUES ('a@b.co', 'A')
ON DUPLICATE KEY UPDATE name = VALUES(name);
```

The conflict target must be a unique index or primary key.

Postgres can return the affected row in the same statement: `INSERT/UPDATE/DELETE ... RETURNING *`. Upstream MySQL has no `RETURNING` (MariaDB does).

`TRUNCATE` removes all rows at once, skips per-row triggers, and usually resets identity. In MySQL it implicitly commits. In Postgres it is transactional and will not run while foreign keys reference the table unless you `CASCADE`.

`MERGE` (Postgres 15+, MySQL 8+) is the standard upsert/sync statement when the decision depends on a match, not only on a unique violation.

---

## Schema and Constraints

Put rules in the database. Application checks race.

| Constraint | Enforces |
| --- | --- |
| `PRIMARY KEY` | Unique, not null, the replication/join identity |
| `UNIQUE` | No duplicate business keys. Multiple nulls still allowed. |
| `NOT NULL` | Column always present |
| `CHECK (balance >= 0)` | Row-local predicate |
| `FOREIGN KEY ... REFERENCES` | Child points at a real parent |
| `ON DELETE CASCADE / RESTRICT / SET NULL` | What happens to children when the parent goes away |

Postgres does **not** auto-index foreign-key columns. A missing index makes `DELETE` on the parent scan the child table. InnoDB creates that index for you.

**Types that change correctness:**

- Money and quantities: `numeric`, never `float` / `double`.
- Time: `timestamptz` (Postgres) stored in UTC. `timestamp` without zone drops the offset.
- Identifiers: `bigint identity`. Random UUID v4 as a clustered primary key scatters inserts across the B-tree. Time-ordered ids (UUID v7, ULID) or a `bigint` PK plus a unique UUID column avoid that.
- Free-form attributes: a `jsonb` column for the part that actually varies. Columns you filter, join, or constrain stay relational.

**Soft delete:** `deleted_at timestamptz`. A partial unique index keeps the business key unique among live rows:

```sql
CREATE UNIQUE INDEX users_email_live
ON users (email)
WHERE deleted_at IS NULL;   -- Postgres
```

**Views** store a query. **Materialized views** store the result and go stale until `REFRESH`. Use a view to hide a join; use a materialized view for an expensive read that tolerates lag.

---

## Normalization

Normalize until writes have one place to change. Denormalize when you can name the read you are buying and the write anomaly you accept.

| Form | Rule | Smell |
| --- | --- | --- |
| **1NF** | Each column holds one value; no repeating groups (`phone1`, `phone2`). | Child table `user_phones`. |
| **2NF** | On a composite key, every non-key column depends on the **whole** key. | `order_items(order_id, product_id, product_name)` — name depends only on product. |
| **3NF** | Non-key columns depend on the key, not on each other. | `employees(dept_id, dept_name)` — name depends on department. |
| **BCNF** | Every determinant is a candidate key. Mention it; day-to-day designs stop at 3NF. | |

Copying `product_name` onto an order line is often **history**, not sloppy design: the name at sale time must survive later renames.

---

## Indexes

An index is a sorted structure (B-tree by default) the planner can binary-search instead of reading the table. Every secondary index is updated on `INSERT`, `UPDATE`, and `DELETE`. Extra indexes speed reads and slow writes.

**Leftmost prefix.** Index `(user_id, created_at)` serves:

- `WHERE user_id = ?`
- `WHERE user_id = ? AND created_at > ?`
- `WHERE user_id = ? ORDER BY created_at`

It does not serve `WHERE created_at > ?` alone. Equality columns come first, then the range or `ORDER BY` column.

**Selectivity. An index pays for itself when few rows match (`email = ?`). A boolean column usually does not: reading half the table through random index lookups is slower than a sequential scan.**

**Covering index.** The index contains every column the query needs, so the engine skips the table. In Postgres that is an index-only scan, and only while the visibility map says the pages are all-visible (vacuum keeps this true). In InnoDB, secondary indexes already store the primary key; add the remaining columns to the index.

**Keep the column bare** so the predicate is sargable (the index can bound a range):

```sql
-- uses (created_at)
WHERE created_at >= DATE '2026-01-01'
  AND created_at <  DATE '2026-01-02'

-- function on the column: sequential scan, unless you index the expression
WHERE DATE(created_at) = DATE '2026-01-01'
```

`LIKE 'abc%'` can use a B-tree. `LIKE '%abc'` cannot (Postgres: `pg_trgm` GIN index if you need contains-search).

Cast the constant to the column’s type. Casting the column (`WHERE varchar_phone = 12345`) hides the index.

| Index | Use |
| --- | --- |
| **B-tree** | Equality, range, `ORDER BY`. The default. |
| **Hash** | Equality only. Rarely better than B-tree. |
| **GIN** | Arrays, `jsonb`, full text: many values per row. |
| **GiST** | Ranges, geometry, exclusion constraints (“no overlapping bookings”). |
| **BRIN** | Huge tables that are naturally ordered (append-only time). Tiny index, coarse. |
| **Partial** | Index a subset: `WHERE deleted_at IS NULL`. Postgres. |
| **Expression** | `CREATE INDEX ON users (lower(email))` and query with `lower(email)`. |

**Clustered vs heap.** InnoDB stores the row in the primary-key B-tree. Secondary indexes point at the PK, so a wide PK is copied into every index. Postgres stores the heap separately; indexes point at a tuple id, and an update that moves the row leaves an index entry to chase.

---

## The Planner

The [logical order](#how-a-query-is-evaluated) is the meaning of the query. The planner may run filters, joins, and projections in a different physical order when the result is the same. It picks the cheapest plan it can find. Cost is an estimate of I/O plus CPU, in units of sequential page reads, not milliseconds.

```
parse → rewrite → plan (cost the alternatives) → execute
```

- **Parse** turns text into a tree.
- **Rewrite** applies rules: inline views, pull up subqueries, fold constants, push predicates down, turn `EXISTS` / `IN` into semi-joins, drop a left join whose inner side is unique and unused. Postgres 12+ inlines CTEs here unless they are `MATERIALIZED`.
- **Plan** chooses, for each step, an access method, a join order, a join algorithm, and how to sort or aggregate.
- **Execute** runs that tree. `EXPLAIN` stops after planning. `EXPLAIN ANALYZE` executes.

### What it searches

For each table: sequential scan, index scan, index-only scan, or bitmap scan. For each join: nested loop, hash, or merge, and which input is outer. Inner joins can be reordered. Outer joins cannot move freely, because the preserved side is part of the result.

The number of join orders grows factorially. Postgres searches them with dynamic programming. At `geqo_threshold` (default 12 relations) it switches to a genetic search, which can miss the best order. A separate knob, `join_collapse_limit` (default 8), caps how far explicit `JOIN` syntax is flattened into that search. Above it, the order you wrote constrains the plan.

`LIMIT` changes the winner. A sort pays its cost before the first row (high **startup** cost). An index walk pays as it goes and can stop after 10 rows. The planner picks the index for `ORDER BY created_at DESC LIMIT 10` even when a full sort would be fine for an unordered read of the whole table.

### Cardinality is the whole game

Every later choice hangs on **how many rows** survive each filter and join.

| Statistic | What the planner believes |
| --- | --- |
| **`null_frac`** | Fraction of nulls. |
| **`n_distinct`** | How many distinct values. A negative number is a fraction of the row count (`-0.5` means about half as many distinct values as rows; `-1` means unique). |
| **Most common values** | Exact frequency of the hottest values. `status = 'active'` uses this, not the average. |
| **Histogram** | Distribution of everything that is not a most-common value. Range predicates (`created_at > …`) use the buckets. |
| **Correlation** | Whether the heap is physically ordered by this column (`-1..1`). High correlation makes a range index scan cheap, because matching rows sit on neighboring pages. Near 0, the same scan looks like random I/O. |

**Selectivity** is the fraction of rows a predicate keeps. The planner multiplies selectivities and assumes the predicates are independent. `country = 'US' AND city = 'Seattle'` is not independent: Seattle already implies a country. The product underestimates the rows, often by orders of magnitude, and the error **compounds at each join**.

| Bad estimate | Plan you tend to get |
| --- | --- |
| Too few rows | Nested loop. The inner side runs far more times than the plan predicted. |
| Too many rows | Hash join or sequential scan, including cases where an index would have been cheaper. |

Fix the estimate before hinting the plan:

- `ANALYZE` after a bulk load. Autovacuum analyzes after about 10% of the table changes, which is late for a table you just filled.
- Raise the sample on a skewed column: `ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000` (default target is 100), then `ANALYZE`.
- Correlated columns: `CREATE STATISTICS ON country, city FROM addresses` so the planner stores a joint distribution (Postgres extended stats: distinct counts, dependencies, most-common combinations).

A function or cast on the column removes the statistic the planner can use, unless an expression index exists and the query writes the same expression.

### Cost knobs that are estimates, not allocations

| Setting | Effect |
| --- | --- |
| **`seq_page_cost` / `random_page_cost`** | Defaults 1 and 4. On SSD, random I/O is closer to sequential; leaving 4 in place biases the planner toward sequential scans. Operators often set `random_page_cost` around `1.1`. |
| **`effective_cache_size`** | Tells the planner how much of the database is likely cached. It allocates nothing. Too low, and index scans look more random than they are. |
| **`work_mem`** | Budget for **one** sort or hash, per node. Three hash joins can use three times that. A low value shows up as `external merge` or hash batches spilled to disk. A wildly low cardinality estimate makes the planner request a hash that then spills. |

Bitmap scan is the middle ground: build a bitmap of matching pages from one or more indexes, then read the heap in physical order. Postgres switches to a **lossy** bitmap when it does not fit in `work_mem`, and rechecks the predicate on each row.

### Prepared statements

A prepared query’s parameter value is unknown at plan time, so the planner cannot use the most-common-value frequency. Postgres runs a few **custom** plans with the real bound value, then may switch to a **generic** plan that assumes an average value (`plan_cache_mode = auto`). On a skewed column the generic plan is stable and wrong: the rare status wants an index, the common status wants a sequential scan, and one plan has to serve both.

`plan_cache_mode = force_custom_plan` keeps planning with the bound value. The SQL Server name for the same trap is parameter sniffing.

MySQL is cost-based too. Histograms are optional (`ANALYZE TABLE t UPDATE HISTOGRAM ON col`). InnoDB index cardinality is approximate. MySQL accepts optimizer hints (`/*+ HASH_JOIN(t) */`); Postgres has no hint syntax in core. Reach for hints after statistics and query shape are already honest.

---

## Reading a Query Plan

`EXPLAIN` shows the plan the [planner](#the-planner) chose. `EXPLAIN ANALYZE` **runs** the query and adds actual time and row counts. Read it on production-sized data; a plan that is fine on 1k rows often flips on 10M. Compare estimated rows with actual rows from the leaves upward: a leaf that is off by 10× becomes a join that is off by far more.

What to look at:

- **Node type** — `Seq Scan` on a large filtered table, `Index Scan`, `Index Only Scan`, `Bitmap Heap Scan`.
- **Estimated rows vs actual rows** — a large gap means bad statistics or a correlated predicate. `ANALYZE` the table.
- **Loops** — a nested loop with `loops = 100000` is the N+1 shape in one query.
- **Sort / Hash** — `external merge` or batches spilled to disk means `work_mem` is too small for this query, or the sort can be removed with an index that matches `ORDER BY`.

| Join | Planner picks it when |
| --- | --- |
| **Nested loop** | Small outer side, indexed inner side. Excellent for “few rows lookup.” Painful when the outer side is large and the inner side is scanned each time. |
| **Hash join** | Equality join, no useful order. Builds a hash table on the smaller side, probes with the larger. Memory-hungry. |
| **Merge join** | Both sides ordered on the join key (index or an explicit sort). Stable for large inputs. |

**N+1:** one query for the parents, then one query per parent for the children. Fix with a join, a single `WHERE id IN (...)`, or a dataloader that batches. The ORM will not do this unless you ask.

**Keyset pagination** replaces deep `OFFSET`, which reads and discards every skipped row:

```sql
SELECT id, created_at
FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Index `(created_at, id)` matches the order. `OFFSET 100000` does not get cheaper with that index.

---

## Transactions, Isolation, and Locks

**ACID**

| | |
| --- | --- |
| **Atomicity** | The transaction commits entirely or rolls back. A crash mid-statement does not leave half an `UPDATE`. |
| **Consistency** | Committed data satisfies constraints. Cross-row rules the schema cannot see are the application’s job — or `SERIALIZABLE`. |
| **Isolation** | Concurrent transactions do not observe each other’s partial work. |
| **Durability** | A successful commit survives a crash. The engine flushes the write-ahead log before returning. |

**MVCC:** an update creates a new row version. Readers see a snapshot and do not block writers. Postgres reclaims dead versions with `VACUUM`. InnoDB purges them from the undo log. Long transactions pin old versions and bloat the table.

### Anomalies

| Anomaly | What you observe |
| --- | --- |
| **Dirty read** | A value another transaction has not committed. |
| **Non-repeatable read** | The same row has a new value later in your transaction. |
| **Phantom** | A range query gains or loses rows. |
| **Lost update** | Two transactions read a balance, each write `balance = read - 10`, one decrement disappears. |
| **Write skew** | Each transaction reads a condition that is true, then updates a **different** row, and together they make the condition false. Two doctors both go off call; the “at least one on call” rule never saw a conflicting row. |

### Isolation levels

| Level | Dirty | Non-repeatable | Phantom | Write skew |
| --- | --- | --- | --- | --- |
| Read uncommitted | possible | possible | possible | possible |
| Read committed | blocked | possible | possible | possible |
| Repeatable read (SQL standard) | blocked | blocked | possible | possible |
| Serializable | blocked | blocked | blocked | blocked |

Engine reality, which is what interviewers push on:

- **Postgres** default is **Read Committed**. `Read Uncommitted` behaves as Read Committed (no dirty reads). **Repeatable Read** is snapshot isolation: phantoms disappear, write skew does not. **Serializable** (SSI) aborts one transaction with `40001` instead of allowing write skew. The application retries.
- **MySQL InnoDB** default is **Repeatable Read**. Consistent reads use a snapshot. Locking reads (`SELECT FOR UPDATE`) take **next-key locks** (row + gap), which block phantoms and are a common deadlock source.

Pick **Read Committed** for ordinary request work. Pick **Repeatable Read** when one transaction must see a stable snapshot. Pick **Serializable**, or lock the rows that form the invariant, when write skew can break a business rule.

### Lost updates

Three fixes, in the order you should reach for them:

```sql
-- 1. Atomic write. No read-modify-write in the app.
UPDATE accounts SET balance = balance - 10 WHERE id = 1 AND balance >= 10;

-- 2. Pessimistic. Lock the row for the rest of the transaction.
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- 3. Optimistic. Succeeds only if nobody else wrote.
UPDATE accounts
SET balance = 90, version = version + 1
WHERE id = 1 AND version = 3;
-- 0 rows → reload and retry
```

`SKIP LOCKED` is the job-queue pattern: claim a row nobody else has locked, without waiting.

```sql
UPDATE jobs
SET status = 'running'
WHERE id = (
  SELECT id FROM jobs
  WHERE status = 'queued'
  ORDER BY id
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

### Deadlocks

Two transactions lock the same rows in opposite orders. The engine aborts one (`40P01` in Postgres, `1213` in MySQL). The application retries.

Prevent them by locking in a stable order and keeping the transaction short:

```sql
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- move money
COMMIT;
```

Do not hold a transaction open across an HTTP call, a user prompt, or a slow external API. Locks and MVCC snapshots live as long as the transaction does.

---

## Scaling

Order that matches [data_store_selection.md](./data_store_selection.md): indexes and query shape, then pooling, then replicas, then cache, then async, then sharding. Read replicas add read QPS. They do not add write throughput. The primary is still one machine.

| Lever | What it changes |
| --- | --- |
| **Connection pool** (PgBouncer, ProxySQL) | Postgres pays about one process per connection. Thousands of app instances need a pool in front, transaction-mode pooling for short OLTP queries. |
| **Read replica** | Scale reads. Replica lag breaks read-your-writes: send a user to the primary for a short window after their own write. |
| **Partitioning** | One logical table, many physical pieces, **one primary**. Range-partition by time so queries that include the partition key prune the rest. A unique key must include the partition key (Postgres). |
| **Sharding** | Many primaries, split by `tenant_id` or `user_id`. Cross-shard transactions and unique keys become application problems. |
| **Outbox** | Write the business row and an event row in the **same** transaction. A worker publishes the event. That is how you get a reliable side effect without a dual write. |

`COUNT(*)` with no `WHERE` on a large InnoDB or Postgres table walks the table. Maintain a counter, or use an approximate (`pg_class.reltuples`) when “about right” is enough.

---

## Postgres vs MySQL

| | Postgres | MySQL (InnoDB) |
| --- | --- | --- |
| Default isolation | Read Committed | Repeatable Read |
| Row store | Heap + secondary indexes | Clustered on the primary key |
| FK indexes | You create them | Created with the FK |
| Partial indexes | Yes | No (generated column + index is the workaround) |
| DDL | Transactional | Most DDL implicitly commits |
| Upsert | `ON CONFLICT` | `ON DUPLICATE KEY UPDATE` |
| Returning rows | `RETURNING` | Not in upstream MySQL |
| Boolean | Real `boolean` | `tinyint(1)` |
| Identity | `GENERATED AS IDENTITY` | `AUTO_INCREMENT` |
| “Contains” text | `tsvector` or `pg_trgm` | `FULLTEXT` |

Both are correct defaults for transactional data. Reach for Postgres unless the team or the platform is already MySQL.

---

## Patterns

**Second distinct salary**

```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

Ties: `DENSE_RANK()` and filter `rk = 2`, which still returns one salary even when several people share the top one.

**Duplicate emails**

```sql
SELECT email, COUNT(*)
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

**Delete duplicates, keep the smallest id** (Postgres)

```sql
DELETE FROM users u
USING (
  SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
  FROM users
) d
WHERE u.id = d.id AND d.rn > 1;
```

**People paid more than their manager**

```sql
SELECT e.name
FROM employees e
JOIN employees m ON m.id = e.manager_id
WHERE e.salary > m.salary;
```

**Customers who ordered all of a set of products** — relational division via counts:

```sql
SELECT user_id
FROM orders
WHERE product_id IN (1, 2, 3)
GROUP BY user_id
HAVING COUNT(DISTINCT product_id) = 3;
```

**Gaps** in a sequence: `LEAD(id) OVER (ORDER BY id)` and keep rows where `next_id > id + 1`.

---

## Drill Questions

**Why does this left join return no unmatched users?**
A predicate on the right table is in `WHERE`, so unmatched rows (right side null) fail the test. Move that predicate into `ON`.

**`NOT IN` against a subquery suddenly returns nothing.**
The subquery produced a null. `x NOT IN (1, NULL)` is unknown. Use `NOT EXISTS`.

**Two transfers debit the same account and one is lost.**
The app did read-modify-write. Use `SET balance = balance - :amt`, or `SELECT … FOR UPDATE`, or a `version` column.

**Both workers took the last on-call slot.**
Write skew under snapshot isolation. Each updated a different row. Use `SERIALIZABLE`, or lock the whole on-call set up front.

**The query is fast in staging and slow in prod.**
Different statistics, data volume, or parameter values (a plan cached for a rare value used on a common one). Compare `EXPLAIN ANALYZE`, estimated vs actual rows, and whether the predicate can use the index.

**`OFFSET 500000` is slow even with an index on the sort key.**
The engine still walks 500000 index entries to discard them. Switch to keyset pagination on `(sort_col, id)`.

**Where do you put a unique email if users can soft-delete and re-register?**
Partial unique index on `email WHERE deleted_at IS NULL`. A plain `UNIQUE(email)` blocks the new signup forever.

**Why index every foreign key in Postgres?**
Parent deletes and cascading checks look up the child by the FK column. Without an index that lookup is a sequential scan, and it runs while holding a lock on the parent.

**An index exists and the planner still sequential-scans.**
It thinks the scan is cheaper. Typical reasons: the predicate matches a large fraction of the table, statistics are stale, a function wraps the column, or `random_page_cost` still says a random read costs 4× a sequential one. Check estimated rows against reality before adding a hint.

**Estimated rows are 100 and the node actually returned 100,000.**
The cardinality estimate is wrong, so the join algorithm above it was chosen for the wrong size. `ANALYZE`, raise the statistics target on the skewed column, or add extended statistics when the filters are correlated.

---

## Quick Reference

| Need | Tool |
| --- | --- |
| Unknown | `IS NULL`, `COALESCE`, `NOT EXISTS` |
| Keep unmatched left rows | `LEFT JOIN`, optional filters in `ON` |
| “Has a child” / “has none” | `EXISTS` / `NOT EXISTS` |
| Collapse rows | `GROUP BY` + `HAVING` |
| Per-row rank or running total | Window function |
| Exactly N per group | `ROW_NUMBER()` in a derived table |
| Second place with ties | `DENSE_RANK()` |
| Insert or update | `ON CONFLICT` / `ON DUPLICATE KEY` |
| Stable page | `(created_at, id) < (?, ?)` + matching index |
| Prevent lost update | Atomic `UPDATE`, `FOR UPDATE`, or `version` |
| Cross-row invariant | `SERIALIZABLE` or lock the rows involved |
| Deadlock / serialization failure | Retry the transaction |
| Equality + range + order | B-tree, equality columns leftmost |
| Planner picked a bad join | Fix the row estimate (`ANALYZE`, extended stats), then the plan |
| Skewed prepared statement | `plan_cache_mode = force_custom_plan`, or split the rare value into its own query |
| Case-insensitive email | Unique index on `lower(email)`, query the same expression |
| Side effect after commit | Transactional outbox |
