# Data Store Selection & Scaling

Pick and scale the **data layer**: throughput limits, SQL vs NoSQL, hybrid stacks, write bottlenecks, sharding. End-to-end BOE (traffic, app tier, CDN, queues, fan-out) is in [system_capacity.md](./system_capacity.md).

## Table of Contents

1. [Prerequisites from System Capacity](#prerequisites-from-system-capacity)
2. [Store Throughput Limits](#store-throughput-limits)
3. [Hockey Stick (Data Layer)](#hockey-stick-data-layer)
4. [Store Selection (SQL vs NoSQL)](#store-selection-sql-vs-nosql)
5. [Why Writes Are Hard (RDBMS)](#why-writes-are-hard-rdbms)
6. [Data-Layer Scaling Levers](#data-layer-scaling-levers)
7. [Data-Layer Checklist](#data-layer-checklist)
8. [Worked Example (Data Path)](#worked-example-data-path)
9. [Interview Drill Questions](#interview-drill-questions)

---

## Prerequisites from System Capacity

Before choosing stores, compute from [system_capacity.md](./system_capacity.md):

- **Write TPS** — peak API writes that hit the DB (or async equivalent)
- **Read QPS** — after Redis/CDN, what still reaches the DB
- **Storage** — retention × replication × index overhead
- **Hot keys / skew** — shard and store choice depend on even distribution

Compare **write TPS** to the tables below; use **~60–70% of knee TPS** as a safe operating point.

---

## Store Throughput Limits

Order-of-magnitude anchors — not guarantees. Depends on hardware, workload, keys, durability.

### Relational & distributed SQL

| System                                   | Typical throughput                                                 | Main constraint                           |
| ---------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------- |
| PostgreSQL / MySQL (single node, writes) | **10k–40k write TPS** (tuned, NVMe)                                | WAL/fsync, row/index locking              |
| PostgreSQL / MySQL (single node, reads)  | **~10k–50k read QPS** (hot data in buffer cache)                   | Disk on cache miss                        |
| PostgreSQL / MySQL (cluster)             | **50k–200k+ aggregate read QPS** (Redis/Memcached + read replicas) | **Writer still single-node**, replica lag |
| CockroachDB / YugabyteDB / Spanner       | Cluster **100k+ TPS** possible                                     | Consensus latency on every write          |

AWS Aurora has a hard quota of 15 read replicas per region. A self-hosted Postgres/MySQL cluster has no fixed cap — bounded by replication lag, ops cost, and connection pooling.

### NoSQL & specialized stores

| System                | Typical throughput                                  | Main constraint                          |
| --------------------- | --------------------------------------------------- | ---------------------------------------- |
| MongoDB (single node) | **5k–20k write TPS**                                | Document size, indexes, WiredTiger cache |
| MongoDB (sharded)     | Scales with shards                                  | Shard key, scatter-gather                |
| DynamoDB              | **~1,000 WCU/s per partition** (1 WCU = 1 KB write) | Hot partitions                           |
| Cassandra / ScyllaDB  | **100k+ write TPS** (cluster)                       | Partition-key access only                |
| Redis                 | **100k–1M ops/s**                                   | RAM, single-thread, AOF fsync            |
| Elasticsearch         | **10k–50k docs/s** ingest (bulk)                    | Refresh, mapping                         |
| ClickHouse / BigQuery | **Millions rows/s** ingest                          | OLAP, not OLTP                           |


### Data-adjacent (often in hybrid stack)

| System   | Role in hybrid stack                              | Main constraint |
|----------|---------------------------------------------------|-----------------|
| Kafka    | Durable **event log**; replay; multiple consumers | Partitions, `acks` — [kafka.md](./kafka.md) |
| S3 / GCS | **Blob** storage (media, exports, data lake)      | **~3,500 PUT/s per prefix** — hash key prefixes |

---

## Store Selection (SQL vs NoSQL)

**Default to RDBMS for transactional data; justify NoSQL when the access pattern demands it.**

Start with Postgres (or MySQL). Add Redis, Kafka, Elasticsearch, S3 when you can name a bottleneck SQL alone cannot solve cleanly.

Beware about the **Hockey Stick** curve — full treatment in [system_capacity.md](./system_capacity.md#hockey-stick-latency-curve). Sharded clusters: **one knee per shard** — aggregate metrics can hide a hot shard.

### Relational / OLTP — the default


| Use RDBMS when                | Examples                               |
| ----------------------------- | -------------------------------------- |
| Consistency across entities   | Orders + inventory, payments + ledger  |
| Ad-hoc queries and joins      | Admin, reporting, evolving queries     |
| Constraints                   | UNIQUE email, FKs, CHECK balance ≥ 0   |
| Multi-row transactions        | Transfer money, order + line items     |
| Write TPS fits single primary | Most SaaS up to **~10k–40k write TPS** |


**SQL bad at alone:** unbounded writes on one primary, search at scale, hot global counters, blobs, firehose ingest.

### Picking a relational engine


| Engine                             | When to pick                                                    |
| ---------------------------------- | --------------------------------------------------------------- |
| **PostgreSQL**                     | Default; JSON, partial indexes, extensions                      |
| **Cockroach / Yugabyte / Spanner** | Write TPS exceeds single-node Postgres + need SQL               |


Don't use distributed SQL until Postgres + pool + replicas + cache + async + shard are insufficient.

### Scaling SQL before leaving SQL

1. **Indexes + tuning** — [sql.md](./sql.md)
2. **Connection pooling** — PgBouncer, ProxySQL
3. **Read replicas** — scale reads only
4. **Cache (Redis)** — cache-aside for hot reads
5. **Async (Kafka/SQS)** — events, notifications, indexing pipeline
6. **Shard primary** — `user_id`, `tenant_id`, time

Read replicas **do not** increase write TPS.

### Hybrid architecture


| Layer           | Store                 | Role                             |
| --------------- | --------------------- | -------------------------------- |
| Source of truth | Postgres / MySQL      | ACID entities                    |
| Cache           | Redis                 | Sessions, hot reads, rate limits |
| Search          | Elasticsearch         | Full-text (CDC / outbox from DB) |
| Events          | Kafka                 | Activity, async workers          |
| Media           | S3 + CDN              | Blobs — not DB BLOBs             |
| Analytics       | ClickHouse / BigQuery | Not production OLTP              |


Sync: **CDC**, **transactional outbox**, async workers, batch rebuild.

### When to leave SQL


| Signal                           | Consider                                  |
| -------------------------------- | ----------------------------------------- |
| Write TPS > single-node headroom | Cassandra, sharded DynamoDB, Kafka ingest |
| Key-only access                  | DynamoDB, Redis                           |
| Variable nested documents        | MongoDB                                   |
| Full-text + facets               | Elasticsearch + SQL source of truth       |
| Graph traversal                  | Graph DB or precomputed adjacency         |
| Append-only firehose             | Kafka → Cassandra / S3                    |
| Ephemeral sub-ms data            | Redis                                     |


### NoSQL categories


| Category    | Examples             | Best for                 |
| ----------- | -------------------- | ------------------------ |
| Key-value   | Redis, DynamoDB      | Cache, sessions, lookups |
| Document    | MongoDB              | Flexible nested records  |
| Wide-column | Cassandra, ScyllaDB  | High write, time-series  |
| Graph       | Neo4j, Neptune       | Relationship traversal   |
| Search      | Elasticsearch        | Full-text, logs          |
| OLAP        | ClickHouse, BigQuery | Aggregations             |


### Consistency spectrum


| Level                | Examples                                    |
| -------------------- | ------------------------------------------- |
| Strong               | RDBMS row, Spanner, DynamoDB strong read    |
| Sequential / session | MongoDB default, Cassandra LOCAL_QUORUM     |
| Eventual             | Cassandra, DynamoDB eventual, Redis replica |
| At-least-once        | Kafka, SQS — idempotent consumers           |

---

## Why Writes Are Hard (RDBMS)

### WAL / fsync

Durability requires WAL flush before ack → caps **fsync/s** on NVMe regardless of CPU.

### Locking

2PL on conflicting rows — hot keys serialize writes.

### B-tree indexes

In-place index updates → random I/O per write. LSM stores (Cassandra) append first, compact later.

### Other DB bottlenecks


| Bottleneck          | Mitigation                          |
| ------------------- | ----------------------------------- |
| Connection overhead | Pooling                             |
| Hot partition / key | Key sharding, write-behind counters |
| Sync replication    | Async replica where safe            |
| Large transactions  | Smaller batches                     |
| No app cache        | Redis cache-aside                   |


---

## Data-Layer Scaling Levers

### Single-node / primary

- Connection pooling, group commit (latency trade-off)
- Read replicas (reads only)
- NVMe, RAM for buffer cache
- Tune durability only where loss is acceptable (never payments default)

### Beyond one node

- Shard by `user_id`, tenant, time — even distribution
- Async ingest via Kafka/SQS
- CQRS / materialized views / search indexes
- Distributed SQL when you need horizontal **SQL** writes

```
shards ≈ peak_write_TPS / per_shard_write_capacity
```

Leave **2× headroom** for rebalance and spikes. Kafka: `partitions ≥ consumer parallelism`.

See [sql.md](./sql.md), [redis.md](./redis.md), [kafka.md](./kafka.md).

---

## Data-Layer Checklist

- `write_TPS` and `read_QPS` from [system_capacity.md](./system_capacity.md)
- Peak write TPS vs store knee (with **~60–70%** headroom)
- Access pattern matches store (not throughput alone)
- Source of truth + sync to cache / search / queue
- Read replicas vs Redis roles clear
- Hot key / shard skew mitigated
- Async path for non-critical / firehose writes
- Storage: indexes, replication, retention

---

## Worked Example (Data Path)

**Given** (from [system_capacity.md](./system_capacity.md) full-stack example): **~1,485 peak write TPS**, **~148,500 read QPS**, **~1.5 PB** metadata storage (5y, 3× rep).

### Compare write path

Single-node Postgres ceiling **~10k–40k write TPS**. At **~1.5k TPS**, primary is **comfortable** for writes if schema and indexes are sane.

### Compare read path

**~148k read QPS** cannot sit on one Postgres alone → **Redis** (hot timelines) + **read replicas** for cache misses; aim for **<10%** traffic to DB.

### Hybrid sketch


| Data                                | Store                                      |
| ----------------------------------- | ------------------------------------------ |
| Users, relationships, post metadata | Postgres (source of truth)                 |
| Hot feeds / sessions                | Redis                                      |
| Post-created events                 | Kafka → workers (notifications, analytics) |
| Media                               | S3 + CDN                                   |
| Search users/posts                  | Elasticsearch via outbox                   |


### Say it out loud

> "Writes fit a single Postgres primary with headroom; reads need Redis and replicas. I'd outbox to Kafka for fan-out and index search async, not synchronous writes on the post path."

---

## Interview Drill Questions

1. 10M DAU, 20 writes/user/day, 3× peak — average and peak **write TPS**? Fits single Postgres?
2. Peak **15k write TPS** — pool, replica, cache, async, or shard first? Why?
3. Why Redis **500k ops/s** but Postgres **~20k write TPS**?
4. Feed **100:1** read:write — cache where, invalidate how?
5. Hot counter **50k updates/s** — fix without melting Postgres?
6. DynamoDB `video_id` key, **100k writes/s** on one video — what breaks?
7. E-commerce search — why Elasticsearch + Postgres?
8. **10 TB logs/day** — Postgres, Cassandra, or S3?
9. MongoDB vs Postgres vs Cassandra — one sentence each?
10. Aurora vs self-hosted Postgres — what scales, what does not?

**System-wide drills:** [system_capacity.md](./system_capacity.md#interview-drill-questions)