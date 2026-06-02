# System Capacity & Scale

Back-of-the-envelope math for capacity planning: estimate traffic → storage → bandwidth → compare against known system limits → pick scaling strategy.

## Table of Contents
1. [Quick Reference Numbers](#quick-reference-numbers)
2. [Estimation Workflow](#estimation-workflow)
3. [Traffic: QPS, TPS & Peak Load](#traffic-qps-tps--peak-load)
4. [Storage & Retention](#storage--retention)
5. [Bandwidth](#bandwidth)
6. [System Throughput Limits](#system-throughput-limits)
7. [Hockey Stick Latency Curve](#hockey-stick-latency-curve)
8. [Store Selection (SQL vs NoSQL)](#store-selection-sql-vs-nosql)
9. [Why Writes Are Hard (RDBMS)](#why-writes-are-hard-rdbms)
10. [Scaling Levers](#scaling-levers)
11. [Capacity Planning Checklist](#capacity-planning-checklist)
12. [Worked Example](#worked-example)
13. [Interview Drill Questions](#interview-drill-questions)

---

## Quick Reference Numbers

### Latency (order of magnitude)

| Operation | Typical |
|-----------|---------|
| L1 cache reference | ~1 ns |
| RAM reference | ~100 ns |
| SSD random read | ~100 μs |
| SSD sequential (1 MB) | ~1 ms |
| Same-DC network RTT | ~0.5 ms |
| Cross-region RTT | 50–150 ms |
| HDD seek | ~10 ms |

Rule of thumb: **RAM is ~1,000× faster than SSD; SSD is ~100× faster than cross-region network.**

### Payload & record sizes (rough defaults)

| Item | Size |
|------|------|
| UUID / ID | 16 B |
| Short text (tweet, comment) | ~300 B – 1 KB |
| User profile row | ~1–2 KB |
| Thumbnail image | ~20–50 KB |
| Photo (compressed) | 200 KB – 2 MB |
| 1 min video (720p) | ~10–50 MB |
| JSON API response | 1–10 KB |

### Availability (downtime budget)

| SLA | Downtime / year |
|-----|-----------------|
| 99.9% ("three nines") | ~8.8 hours |
| 99.99% ("four nines") | ~52 minutes |
| 99.999% ("five nines") | ~5 minutes |

---

## Estimation Workflow

Use this order in every design interview:

1. **Clarify scope** — reads vs writes, consistency needs, retention, geo distribution.
2. **Estimate traffic** — DAU/MAU → actions/day → average QPS/TPS → peak QPS/TPS.
3. **Estimate storage** — record size × writes/day × retention × replication factor.
4. **Estimate bandwidth** — peak QPS × avg payload size (ingress + egress).
5. **Compare to limits** — single-node DB? cache? queue? CDN?
6. **Propose architecture** — shard, replicate, cache, async, relax durability where safe.
7. **State assumptions** — "I'm assuming 50% DAU/MAU, 10 writes/user/day, 3× peak factor."

---

## Traffic: QPS, TPS & Peak Load

### Key terms

- **QPS (Queries Per Second)**: any request (reads + writes). What most APIs report.
- **TPS (Transactions Per Second)**: state-changing, ACID-bound operations. Usually the write bottleneck.
- **DAU / MAU**: daily / monthly active users. Typical DAU/MAU ratios:
  - Social / messaging: **40–60%**
  - Productivity / SaaS: **10–30%**
  - E-commerce: **5–20%**

### Formulas

```
daily_actions     = DAU × actions_per_user_per_day
average_QPS       = daily_actions / 86,400   (round to 100,000 in interviews)
peak_QPS          = average_QPS × peak_factor
total_QPS         = peak_QPS (for capacity limits)
total_QPS         = average_QPS (for steady-state cost)
```

**Peak factor**: typically **2×–5×** average. Use 3× as a safe default; say why (time zones, lunch/evening spikes, viral events).

### Read/write split

Ask or assume a ratio, e.g. **100:1 reads:writes** for feeds, **10:1** for e-commerce product pages.

```
read_QPS  = total_QPS × read_ratio
write_TPS = total_QPS × write_ratio
```

Example: `total_QPS = 15,000` (peak), ratio **100:1** → `read_QPS ≈ 14,850`, `write_TPS ≈ 149`.

Design for **peak write TPS** when choosing the primary database.

### Concurrent users vs QPS

Not the same. 1M concurrent users with 1 action/minute ≈ **~17,000 QPS**, not 1M QPS.

```
total_QPS = concurrent_users × (actions_per_user_per_second)
```

Example: 1M users, 1 action/minute each → `1,000,000 / 60 ≈ 17,000` total QPS (not 1M).

---

## Storage & Retention

### Formula

```
daily_new_data  = writes_per_day × avg_record_size
total_storage   = daily_new_data × retention_days × replication_factor
```

Add **20–30% headroom** for indexes, metadata, and growth.

### Common assumptions

| Factor | Typical range |
|--------|---------------|
| Replication factor | 2–3 (DB), 3 (Kafka) |
| Index overhead | +20–50% on top of raw row size |
| Retention | 7 days (logs), 1–7 years (user data), forever (compliance) |
| Growth buffer | Plan for **2–3×** current size over 2–3 years |

### Hot vs cold

- **Hot data** (last 30–90 days): SSD, in-memory cache, primary DB.
- **Cold data** (older): object storage (S3/GCS), cheaper tiers, archival.

---

## Bandwidth

```
ingress  = write_QPS × avg_upload_size
egress   = read_QPS × avg_response_size
total    = ingress + egress (+ CDN origin fetch if applicable)
```

Convert to Gbps: `bytes_per_second × 8 / 10^9`.

Example: 10,000 QPS × 10 KB response = 100 MB/s ≈ **0.8 Gbps egress**.

Watch **CDN offload** — static assets should not hit origin at full read QPS.

---

## System Throughput Limits

There is no fixed software TPS cap. Limits depend on hardware, workload shape, partition design, and consistency/durability requirements. Treat numbers as **order-of-magnitude anchors**, not guarantees.

### Relational & distributed SQL

| System | Typical throughput | Main constraint |
|--------|-------------------|-----------------|
| PostgreSQL / MySQL (single node) | **10k–40k write TPS** (tuned, NVMe) | WAL/fsync, row/index locking |
| PostgreSQL / MySQL reads | **50k–200k+ aggregate read QPS** (Redis/Memcached + read replicas) | Replica lag, cache invalidation |
| Aurora (MySQL/PostgreSQL) | Reads scale with replicas (up to 15 in a region) | Writer still single-node; cross-AZ latency |
| CockroachDB / YugabyteDB / Spanner | Scales with cluster (100k+ TPS possible) | Consensus latency on every write |

### NoSQL & specialized stores

| System | Typical throughput | Main constraint |
|--------|-------------------|-----------------|
| MongoDB (single node) | **5k–20k write TPS** | Document size, index count, WiredTiger cache |
| MongoDB (sharded cluster) | Scales with shards | Shard key choice, scatter-gather queries |
| DynamoDB | **~1,000 WCU/s per partition** (1 WCU = 1 KB write); auto-splits | Hot partitions, access-pattern rigidity |
| Cassandra / ScyllaDB | **100k+ write TPS** per node/cluster | Eventual consistency, query-by-partition-key only |
| Redis | **100k–1M ops/s** | RAM size, persistence mode (AOF/RDB) |
| Elasticsearch / OpenSearch | **10k–50k docs/s** ingest (bulk, cluster-dependent) | Index size, refresh interval, mapping changes |
| ClickHouse / BigQuery (OLAP) | **Millions rows/s** ingest (batch/stream) | Not for OLTP; eventual query freshness |

### Messaging, cache & object storage

| System | Typical throughput | Main constraint |
|--------|-------------------|-----------------|
| Kafka (cluster) | **Millions msg/s** | Partitions, replication factor, `acks` setting |
| SQS / Pub/Sub | **Nearly unlimited** (soft limits apply) | At-least-once delivery, visibility timeout tuning |
| S3 / GCS (object storage) | **3,500 PUT/s and 5,500 GET/s per prefix** | Prefix hot spots; use random/key-hash prefixes |
| CDN (CloudFront, etc.) | **Absorbs read traffic** at edge | Cache hit ratio, origin bandwidth on miss |

### Application layer

| System | Typical throughput | Main constraint |
|--------|-------------------|-----------------|
| App server (stateless) | **1k–10k RPS** per instance | CPU, thread pool, downstream DB |
| Load balancer | **100k+ connections** | Connection table, SSL termination CPU |

**Interview move**: after computing peak write TPS, compare to the relevant row and justify sharding, async writes, a different store, or a hybrid (e.g. Postgres for transactions + Kafka for events + S3 for media).

---

## Hockey Stick Latency Curve

Throughput does **not** scale linearly with load on any store or service. While headroom remains, p99 latency stays relatively flat — then the curve bends sharply at a **knee** (workload- and system-dependent):

```
latency
    │                              ╱  ← latency explodes
    │                            ╱
    │                          ╱
    │────────────────────────╱   ← knee (see table below)
    │                        │
    └────────────────────────┴────────► load (TPS / QPS / ops/s)
         flat region              plateau / collapse
```

**Flat region (under the knee):** CPU, memory, disk, network, or partition capacity has headroom. More load adds modest queueing; latency grows slowly.

**Knee:** A bottleneck saturates — WAL `fsync`, a hot partition, single Redis thread, S3 prefix limit, etc. Small load increases no longer buy proportional throughput.

**Post-knee:** Throughput **plateaus or drops** (retries, lock waits, throttling, checkpoint stalls) while latency shoots up. Fix by staying below the knee or removing the bottleneck (shard, async, scale out, different store).

Sharded clusters have **one knee per shard/partition** — aggregate throughput can look healthy while a hot shard hockey-sticks.

**Interview line:** "I'd size below ~60–70% of measured knee throughput so viral spikes stay on the flat part of the curve, not the vertical wall."

---

## Store Selection (SQL vs NoSQL)

**Default to RDBMS for transactional data; justify NoSQL when the access pattern demands it.**

Start with Postgres (or MySQL) for core business data. Only introduce NoSQL, queues, or object storage when you can name a concrete bottleneck — write TPS, hot keys, search, cache, blob storage — that SQL alone cannot solve cleanly.

### Relational / OLTP — the default

| Use RDBMS when | Examples |
|----------------|----------|
| Data must stay consistent across entities | Orders + inventory, payments + ledger, account balances |
| You need ad-hoc queries and joins | Reporting, admin tools, evolving product queries |
| Constraints enforce business rules | UNIQUE email, FK order → user, CHECK balance ≥ 0 |
| Transactions span multiple rows/tables | Transfer money, reserve inventory, create order + line items |
| Write TPS fits a single primary (see [System Throughput Limits](#system-throughput-limits)) | Most SaaS, e-commerce, fintech up to ~10k–40k write TPS |

**What SQL is good at:** flexible schema with strong invariants, multi-table joins, indexes for varied read patterns, mature tooling (migrations, backups, replication), and full ACID without application-level compensation logic.

**What SQL is bad at alone:** unbounded horizontal writes on one primary, full-text search at scale, sub-ms hot-key counters, blob/media storage, and firehose append-only ingest at millions of events/s.

### Picking a relational engine

| Engine | Strengths | When to pick |
|--------|-----------|--------------|
| **PostgreSQL** | Rich types (JSON, arrays), partial indexes, strong extensibility | Default choice; complex queries, JSON columns, geo/full-text via extensions |
| **MySQL / Aurora** | Wide managed hosting, read-replica scaling on Aurora | Existing MySQL fleet, AWS-native stack, read-heavy OLTP |
| **CockroachDB / YugabyteDB / Spanner** | Horizontal write scale + SQL | Peak write TPS exceeds single-node Postgres; need global distribution + strong consistency |

Distributed SQL trades **consensus latency on every write** for horizontal scale. Don't reach for it until single-node Postgres + sharding/async paths are clearly insufficient.

### Scaling SQL before leaving SQL

Apply these in order — most systems never need more:

1. **Indexes + query tuning** — fix slow reads/writes before changing stores (see [sql.md](./sql.md)).
2. **Connection pooling** — PgBouncer, HikariCP; DB CPU spent on connect/auth is wasted.
3. **Read replicas** — offload read QPS; primary handles writes only. Accept replica lag for non-critical reads.
4. **Cache (Redis)** — hot keys, sessions, rate limits; cache-aside for read-heavy paths.
5. **Async writes (Kafka/SQS)** — decouple user-facing latency from durable ingest for events, logs, notifications.
6. **Shard the primary** — split by `user_id`, `tenant_id`, or time when peak write TPS exceeds single-node headroom.

Read replicas **do not** increase write TPS — they only scale reads. When write TPS is the bottleneck, shard, async, or move append-heavy data to a write-optimized store.

### Hybrid architecture — the norm at scale

Production systems rarely use one database. Name the **source of truth** and how everything else stays in sync:

| Layer | Store | Role |
|-------|-------|------|
| **Source of truth** | Postgres / MySQL | Accounts, orders, payments, inventory — anything needing ACID |
| **Cache** | Redis | Sessions, hot reads, rate limits, leaderboards |
| **Search index** | Elasticsearch | Full-text product/user search (rebuilt from DB via CDC or outbox) |
| **Event log** | Kafka | Activity stream, async workers, analytics pipeline |
| **Blob / media** | S3 + CDN | Images, video, exports — never BLOB columns at scale |
| **Analytics** | ClickHouse / BigQuery | Dashboards and aggregations — never query production OLTP for heavy reports |

Sync patterns: **CDC** (Debezium), **transactional outbox** (write event in same DB txn → worker publishes), **async workers**, **periodic batch rebuild**. Say which pattern you pick and why.

### When to leave SQL (justify NoSQL)

| Signal | Consider | Instead of |
|--------|----------|------------|
| Peak write TPS > single-node Postgres with headroom | Cassandra, sharded DynamoDB, Kafka ingest | Forcing all writes through one primary |
| Access pattern is key-only, no joins | DynamoDB, Redis | Relational model with unused join capability |
| Flexible nested documents, schema varies per record | MongoDB | Wide nullable columns or EAV anti-patterns in SQL |
| Full-text + faceted search | Elasticsearch (+ SQL as source of truth) | `LIKE '%term%'` or heavy Postgres full-text at scale |
| Graph traversal (friends-of-friends, fraud rings) | Neo4j, Neptune, or precomputed adjacency in SQL | Deep recursive joins |
| Append-only firehose (IoT, logs, activity) | Kafka → Cassandra / S3 | Synchronous INSERT per event into OLTP |
| Sub-ms reads/writes on ephemeral data | Redis | Postgres on every request |

### NoSQL categories (know which bucket you're in)

| Category | Examples | Best for |
|----------|----------|----------|
| **Key-value** | Redis, DynamoDB, Memcached | Cache, sessions, counters, simple lookups by key |
| **Document** | MongoDB, Couchbase, Firestore | Flexible schemas, nested JSON, content/catalog |
| **Wide-column** | Cassandra, ScyllaDB, HBase | High write volume, time-series, append-heavy logs |
| **Graph** | Neo4j, Neptune | Relationship traversal (friends-of-friends, fraud rings) |
| **Search** | Elasticsearch, OpenSearch | Full-text search, faceted filtering, log analytics |
| **OLAP / columnar** | ClickHouse, BigQuery, Redshift | Aggregations, dashboards, reporting (not live OLTP) |

### Core NoSQL trade-offs (what you give up vs RDBMS)

| Gain | Cost |
|------|------|
| Horizontal scale on writes | Weaker or tunable consistency (not full ACID by default) |
| Flexible / schemaless data | No ad-hoc joins — denormalize or join in application code |
| High throughput on simple access patterns | Queries must match partition/shard key design upfront |
| Managed auto-scaling (DynamoDB, Aurora) | Cost at scale; throttling if mis-provisioned |
| Built-in TTL, replication, geo (many NoSQL) | Cross-partition transactions are limited or expensive |

### Consistency spectrum (state this in interviews)

| Level | Systems | Behavior |
|-------|---------|----------|
| **Strong / linearizable** | RDBMS (single row), Spanner, DynamoDB (strong reads) | Read sees latest write; highest latency |
| **Sequential / session** | MongoDB (default reads), Cassandra (LOCAL_QUORUM) | Consistent within a session or quorum |
| **Eventual** | Cassandra, DynamoDB (eventually consistent reads), Redis replicas | Replicas converge; stale reads possible |
| **Best-effort / at-least-once** | Kafka consumers, SQS | Duplicates possible; design idempotent handlers |

CAP shorthand: under partition, choose **CP** (consistent but may reject writes) or **AP** (available but stale). Most distributed systems let you tune per operation.

### Per-system quick notes

**PostgreSQL / MySQL**
- Default OLTP store: ACID transactions, joins, constraints, secondary indexes.
- Single primary handles all writes; read replicas + Redis scale **aggregate** reads (~50k–200k+ read QPS), not one node alone (~10k–50k).
- Good: orders, accounts, inventory, anything needing invariants. Bad: hot global counters, blob storage, search at scale, write TPS beyond single-node headroom.
- Scale path: pool → replicas → cache → async → shard (see [Scaling SQL before leaving SQL](#scaling-sql-before-leaving-sql)).

**MongoDB**
- Document model with secondary indexes; shard by `shard key` (often `user_id`).
- Multi-document ACID transactions exist but add overhead — avoid for high-TPS paths.
- Good: catalogs, user profiles, content with nested fields. Bad: heavy cross-document joins, hot global counters.

**DynamoDB**
- Provisioned or on-demand capacity; partition key (+ optional sort key) defines access pattern.
- **~1,000 WCU/s and 3,000 RCU/s per partition** — hot keys throttle the whole partition.
- Good: key-value lookups, session store, gaming leaderboards (with sharded keys). Bad: flexible ad-hoc queries, unbounded scans.

**DynamoDB capacity math**
```
WCU = ceil(item_size_KB) × writes_per_second     (max 400 KB per item)
RCU = ceil(item_size_KB / 4) × reads_per_second  (eventually consistent: halve RCU)
```
Example: 2 KB item, 500 writes/s → 500 × 2 = **1,000 WCU** (fills one partition). Fix: shard key (`video_id#shard_0..N`) or write sharding pattern.

**Cassandra / ScyllaDB**
- Write-optimized LSM; partition key determines which nodes hold data.
- Queries must include partition key; no efficient secondary-index-heavy workloads.
- Good: activity feeds, IoT time-series, messaging metadata. Bad: multi-partition transactions, complex joins.

**Elasticsearch**
- Inverted index for full-text search; near-real-time (default 1s refresh).
- Good: search, log aggregation, faceted browse. Bad: primary source of truth, strong consistency, frequent updates to same doc.

**Redis**
- In-memory; sub-ms latency. Use for cache, rate limiting, leaderboards (sorted sets), pub/sub.
- Good: hot read offload, ephemeral data. Bad: primary durable store without AOF/cluster tuning.

**S3 / object storage**
- Virtually unlimited capacity; pay per GB + request. Not a database — no indexing beyond key prefix.
- Good: images, video, backups, data lake. Bad: small random reads/writes, transactional updates.

### Pick-by-use-case (fast reference)

| Use case | Prefer | Avoid |
|----------|--------|-------|
| Orders, payments, inventory | **RDBMS (Postgres)** — source of truth | Eventual-consistency NoSQL as sole store |
| User profiles, accounts | **RDBMS** | DynamoDB unless access is strictly key-value at huge scale |
| User session / cache | Redis | Postgres on every request |
| Activity log / events | Kafka → Cassandra or S3 | Synchronous writes to RDBMS |
| Product search | Elasticsearch + **Postgres as source of truth** | SQL `LIKE` at scale |
| User-uploaded media | S3 + CDN | BLOB columns in RDBMS |
| Real-time analytics dashboard | ClickHouse / OLAP pipeline | Querying production OLTP DB |
| Social graph traversal | Graph DB or precomputed adjacency in **Postgres** | Deep recursive SQL joins at query time |
| Global counter (likes, views) | Redis / sharded counters / HyperLogLog | Single row `UPDATE` in RDBMS |

**Interview close:** "Postgres owns accounts and orders. Redis caches sessions. Kafka ingests activity events. Elasticsearch indexes products via outbox. S3 stores media."

---

## Why Writes Are Hard (RDBMS)

See [Hockey Stick Latency Curve](#hockey-stick-latency-curve) for the saturation pattern; below is why a single relational primary hits its knee early.

### The WAL / fsync bottleneck

To satisfy **Durability** in ACID, the DB must flush the Write-Ahead Log to disk (`fsync`) before acknowledging success. NVMe has physical latency limits → caps sequential fsyncs/sec regardless of CPU.

### Locking & contention

Strong consistency requires **two-phase locking (2PL)** on conflicting rows. Hot keys (counters, popular records) serialize writes → TPS drops sharply.

### B-tree index maintenance

RDBMS indexes (B-tree/B+tree) are updated **in place** → random I/O per write. Contrast with LSM trees (Cassandra, RocksDB): **append-only writes**, compaction later.

### Other bottlenecks (all systems)

| Bottleneck | Symptom | Mitigation |
|------------|---------|------------|
| Connection overhead | DB CPU on connect/auth | Connection pooling (PgBouncer, HikariCP) |
| Hot partitions / keys | Single shard saturated | Key sharding, write-behind counters |
| Sync replication | Write latency + lower TPS | Async replicas where acceptable |
| Large transactions | Long lock hold | Smaller batches, idempotent retries |
| Missing cache | DB hit for every read | Read-through / write-through cache |

---

## Scaling Levers

### Maximize single-node TPS

- **Connection pooling** — PgBouncer, HikariCP; avoid one connection per request.
- **Group commit** — batch multiple transactions into one fsync (trade latency for throughput).
- **Read replicas** — offload reads; primary handles writes only.
- **Hardware** — NVMe SSD, more RAM (keep working set in buffer cache).
- **Tune durability** — `synchronous_commit=off` or relaxed `fsync` only where data loss is acceptable (never default for money/orders).

### Scale beyond one node

- **Sharding / partitioning** — split by user_id, tenant, or time; aim for even key distribution.
- **Async processing** — queue writes (Kafka/SQS), process at sustainable TPS.
- **CQRS / event sourcing** — separate write path from read-optimized stores.
- **Denormalize for reads** — materialized views, search indexes, cache layers.
- **Distributed SQL** — CockroachDB, YugabyteDB, Spanner when you need SQL + horizontal writes.
- **Choose the right store** — don't force all traffic through one RDBMS (see [sql.md](./sql.md), [redis.md](./redis.md), [kafka.md](./kafka.md)).

### Partition / shard count hint

```
partitions ≥ peak_consumer_parallelism   (Kafka)
shards     ≈ peak_write_TPS / per_shard_write_capacity
```

Always leave **2× headroom** for rebalancing and traffic spikes.

---

## Capacity Planning Checklist

Before finalizing architecture, confirm:

- [ ] Stated assumptions (DAU, actions/user, record size, retention)
- [ ] Average **and** peak QPS/TPS computed
- [ ] Read vs write split explicit
- [ ] Storage includes indexes + replication + growth buffer
- [ ] Bandwidth fits NIC / CDN budget
- [ ] Peak write TPS compared to chosen DB limit
- [ ] Store choice matches access pattern (not just throughput)
- [ ] Source of truth identified; async replicas/search indexes explained
- [ ] Single point of failure addressed (replicas, multi-AZ)
- [ ] Hot key / skew risk called out
- [ ] Async path for non-critical writes

---

## Worked Example

**Prompt**: Design write path for a social app — 100M MAU, 10 writes/user/day.

### Step 1 — Traffic

```
DAU         = 100M × 50%           = 50M users/day
Daily writes = 50M × 10             = 500M writes/day
Average TPS  = 500M / 100,000 s     ≈ 5,000 TPS
Peak TPS     = 5,000 × 3            ≈ 15,000 peak write TPS
```

### Step 2 — Storage (assume 500 B/write, 5-year retention, 3× replication)

```
Daily raw   = 500M × 500 B          = 250 GB/day
5-year raw  = 250 GB × 365 × 5      ≈ 450 TB
With 3× rep = ~1.3 PB               (+ indexes → plan ~1.5 PB)
```

### Step 3 — Compare to DB limit

Single-node PostgreSQL ceiling ≈ **10k–40k write TPS**. At **~15k peak TPS**, a tuned single node is borderline — tight headroom, SPOF, no room for viral spikes.

**Verdict**: shard by `user_id`, or offload high-volume writes (activity feed, analytics) to Kafka + async consumers; keep transactional data (accounts, payments) on RDBMS.

### Step 4 — Say it out loud

> "At 15k peak write TPS I'm near single-node Postgres limits, so I'd shard the write path and use async ingestion for non-critical events, keeping ACID on the core user/account store."

---

## Interview Drill Questions

1. 10M DAU, each user sends 20 messages/day — what's average and peak write QPS? (Use 3× peak.)
2. Photo app: 1M uploads/day, avg 2 MB — daily storage? Bandwidth at peak if uploads cluster in 4 hours?
3. 99.9% vs 99.99% — how much downtime budget difference? What architecture changes buy an extra nine?
4. Why can Redis do 500k ops/s but Postgres struggles at 20k write TPS?
5. Feed is 100:1 read:write — where do you cache, and how do you invalidate?
6. One hot counter gets 50k updates/s — how do you handle it without melting the DB?
7. DynamoDB table keyed by `video_id` — a viral video gets 100k writes/s. What breaks and how do you fix it?
8. Design search for an e-commerce catalog — why Elasticsearch + Postgres, not Postgres alone?
9. 10 TB of logs/day — store in Postgres, Cassandra, or S3? Justify retention and query pattern.
10. When would you pick MongoDB over Postgres? When would you pick neither and use Cassandra?
