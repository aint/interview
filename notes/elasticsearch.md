# Elasticsearch

## Table of Contents

1. [Core Concepts & Mental Model](#core-concepts--mental-model)
2. [Cluster Architecture & Node Roles](#cluster-architecture--node-roles)
3. [The Write Path: Ingestion & Indexing Mechanics](#the-write-path-ingestion--indexing-mechanics)
4. [The Read Path: Search Execution Lifecycle](#the-read-path-search-execution-lifecycle)
5. [Under the Hood: Lucene & The Inverted Index](#under-the-hood-lucene--the-inverted-index)
6. [Cluster Coordination & Consensus](#cluster-coordination--consensus)
7. [Performance & Scalability Optimization](#performance--scalability-optimization)
8. [Common Challenges & System Design Trade-offs](#common-challenges--system-design-trade-offs)
9. [Key System Design Interview Questions](#key-system-design-interview-questions)
10. [Quick Reference](#quick-reference)

---

## Core Concepts & Mental Model

**Elasticsearch** (ES) is a distributed, JSON-based document store and search engine built on top of Apache Lucene. It is optimized for **sub-second full-text search, log analytics, and real-time aggregations**. Unlike a relational database optimized for ACID compliance, ES trades strict immediate consistency for rapid scalability and powerful textual evaluation capabilities.

### Document-Oriented Architecture

- **Document**: The basic unit of information represented in JSON (analogous to a *row* in an RDBMS). Every document has a unique `_id` and an implicit or explicit schema definition called a mapping.
- **Index**: A logical namespace that groups a collection of documents sharing similar characteristics (analogous to a *table* in an RDBMS). In modern ES, time-series workloads often use **Data Streams** (append-only, managed by ILM) instead of manually rolling indices.
- **Mapping**: The schema definition of an index. It defines field types (`text`, `keyword`, `integer`, `date`, `geo_point`, etc.), whether fields are indexed/searchable, and which analyzer runs during ingestion.
- **Text**: Analyzed via tokenization/normalization for full-text search. Supports relevance scoring.
- **Keyword**: Kept intact as a single exact token for filtering, sorting, aggregations, and `term`-level queries.
- **`_source`**: The original JSON document stored on disk. Returned in search hits by default; can be disabled to save space (but then you cannot reindex or update in place without an external store).

### Analyzers (Ingestion vs Query)

Analyzers transform raw strings into tokens stored in the inverted index:

```
Input → Character Filters → Tokenizer → Token Filters → Tokens
```

- **Index time**: Documents are analyzed when indexed (`text` fields).
- **Query time**: `match` queries analyze the search string; `term` queries do not.
- **Common gotcha**: Indexing `"Quick Fox"` as `text` (tokens: `quick`, `fox`) but searching with a `term` query for `"Quick Fox"` returns nothing. Use `match` for analyzed fields, `term` for `keyword` fields.

---

## Cluster Architecture & Node Roles

An Elasticsearch cluster is composed of one or more distributed servers (**nodes**). Sharding allows indices to break out of single-machine storage bottlenecks.

```
                  [ Client / Load Balancer ]
                             │
                             ▼
                    ┌─────────────────┐
                    │ Coordinate Node │
                    └────────┬────────┘
                             │ (Scatter / Gather)
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    Data Node    │ │    Data Node    │ │  Master Node    │
│  ┌───────────┐  │ │  ┌───────────┐  │ │ (Cluster State) │
│  │ Shard P0  │  │ │  │ Shard R0  │  │ └─────────────────┘
│  └───────────┘  │ │  └───────────┘  │
└─────────────────┘ └─────────────────┘
```

### Shards: Primary vs. Replica

An index is logically divided into one or more physical units called **shards**. Each shard is a self-contained instance of an Apache Lucene index.

- **Primary shards**: Handle write operations. The number of primary shards is defined at index creation and **cannot be changed freely** — routing depends on the shard count. Use `_split` / `_shrink` APIs (with constraints) or `_reindex` into a new index to resize.
- **Replica shards**: Redundant copies of primary shards. They handle read queries, provide high availability, and can be adjusted dynamically.

**Routing formula** (simplified):

$$\text{shard} = \text{hash}(\_routing) \bmod \text{number\_of\_primary\_shards}$$

By default, `_routing` is the document `_id`. Custom routing co-locates related documents on the same shard (useful for parent/child or multi-tenant patterns, but can create hot shards if misused).

### Node Roles (Decouple for Production Scale)

Configured via `node.roles` (ES 7+):

- **Master-eligible nodes**: Participate in master election and cluster-state voting. A **dedicated master** sets `node.roles: [master]` — handles lightweight cluster-wide metadata (index creation, shard allocation, mapping updates). Does **not** store data or run heavy queries.
- **Data nodes**: Hold shards. Execute CRUD, search, and aggregations (RAM and I/O bound). `node.roles: [data]`.
- **Ingest nodes**: Run ingest pipelines (Grok parsing, GeoIP enrichment, field mutations) before documents reach a shard. `node.roles: [ingest]`.
- **Coordinating-only nodes**: Accept client requests and scatter/gather without holding data. `node.roles: []` (empty). Every node coordinates by default; dedicated coordinators isolate query fan-out load from data nodes.
- **ML / remote cluster client**: Optional specialized roles for machine learning jobs or cross-cluster search.

> **Note**: The legacy "client node" role was removed. Use coordinating-only nodes instead.

### Index Aliases

An **alias** is a pointer to one or more indices (or a data stream). Enables zero-downtime migrations: write to alias `logs`, reindex into `logs-v2`, atomically swap the alias, delete the old index.

---

## The Write Path: Ingestion & Indexing Mechanics

Writes operate on a two-phase model: replicate to shard copies, then acknowledge based on durability settings.

```
Client ──> Coordinate Node ──> Primary Shard (Buffer + Translog) ──> Replica Shards
```

### Step-by-Step Execution

1. **Routing**: The coordinate node computes the destination primary shard using the routing hash (see above).
2. **Primary ingestion**: The primary validates mapping, assigns `_id` if missing, indexes locally into the memory buffer, and appends to the translog.
3. **Replica replication**: The primary forwards the operation in parallel to all in-sync replica shards.
4. **Acknowledgment**: The client receives success once **`wait_for_active_shards`** copies have acknowledged (default: `1`, meaning only the primary). Set to `all` for maximum durability at the cost of write latency.

### Memory Structures & Durability

ES balances ingestion throughput with durability using two in-memory/disk structures:

```
                  ┌───────────────┐     Every 1s (Refresh)    ┌────────────────────────┐
Incoming Write ──>│ Memory Buffer ├──────────────────────────>│ OS Page Cache (Lucene) │
       │          └───────────────┘                           └────────────────────────┘
       │          ┌───────────────┐     On threshold (Flush)  ┌────────────────────────┐
       └─────────>│   Translog    ├──────────────────────────>│   Physical Disk Storage│
                  └───────────────┘                           └────────────────────────┘
```

- **Memory buffer**: Holds incoming documents. **Not yet searchable.**
- **Translog (transaction log)**: Append-only log on disk for crash recovery. Every indexing/deleting operation is written here first (durability depends on `index.translog.durability`: `request` fsyncs each op; `async` fsyncs on interval).
- **Refresh (near-real-time)**: By default every `1s` (`index.refresh_interval`), the buffer is written to a new immutable **Lucene segment** in the OS page cache. The document becomes **searchable** at this point — even before a full flush to persistent storage.
- **Flush (Lucene commit)**: Triggered when the translog exceeds `index.translog.flush_threshold_size` (default 512 MB) or on a full cluster restart. Commits all segments to disk with `fsync`, then clears the translog. This is **durability**, not searchability.

### Bulk API

For high-throughput ingestion, batch documents with the `_bulk` API (one HTTP request, many index/update/delete actions). Reduces network overhead. Tune `refresh_interval` during bulk loads (see trade-offs table).

### Optimistic Concurrency Control

Each document has `_seq_no` (sequence number) and `_primary_term` (primary shard generation). Updates can include `if_seq_no` + `if_primary_term` to prevent lost updates in concurrent write scenarios (compare-and-swap semantics).

---

## The Read Path: Search Execution Lifecycle

Full-text searches follow a **scatter-gather** pattern split into two phases: **Query** and **Fetch**.

```
Coordinate Node ──(Scatter Query)──> All Target Shards
Coordinate Node <──(Gather Top-K)─── All Target Shards (Returns IDs & Scores)
Coordinate Node ──(Fetch Docs)─────> Relevant Shards (Returns raw JSON)
```

### Phase 1: The Query Phase

1. The client sends a query requesting the top *K* results.
2. The coordinate node selects shard copies (primary or replica) for the targeted index/alias.
3. It broadcasts a lightweight query to all selected shards.
4. Each shard runs the search locally over Lucene segments, builds a sorted priority queue of the top *K* `(doc_id, score)` pairs, and returns only those to the coordinator.

### Phase 2: The Fetch Phase

1. The coordinator merges all shard-level top-*K* queues into a global top-*K*.
2. It identifies which shards hold the winning document IDs.
3. Targeted fetch requests retrieve the full `_source` JSON from those shards.
4. The coordinator returns the final ranked result set to the client.

### Query Types (Interview Essentials)

| Query | Behavior | Typical Use |
| --- | --- | --- |
| `match` | Analyzes input, then searches tokens | Full-text on `text` fields |
| `term` | Exact match, no analysis | `keyword` fields, IDs, enums |
| `bool` | Combines must/should/filter/must_not | Complex queries |
| `filter` context | No scoring, cacheable | Structured filtering (cheaper than scored queries) |

### Aggregations

Aggregations compute analytics (counts, averages, histograms, terms buckets) alongside or instead of returning documents. They run on shard data locally, then merge partial results at the coordinator — similar scatter-gather to search. **Doc values** (columnar on-disk storage) back most aggregations on keyword/numeric fields.

---

## Under the Hood: Lucene & The Inverted Index

Every Elasticsearch shard manages an independent **Apache Lucene index** consisting of immutable segments.

### The Inverted Index

Maps terms → document postings (optimized for "find all docs containing X"):

- **Term dictionary**: Sorted list of unique terms.
- **Term index**: Finite-state transducer (FST) stored in memory pointing to offsets in the term dictionary — enables fast term lookup without scanning disk.
- **Postings list**: Document IDs (and optionally term frequency, positions, payloads) for each term. Positions enable phrase queries; frequencies feed BM25 scoring.

### Segment Immutability & Merging

- **Immutability**: Lucene segments are write-once. No in-place edits — avoids lock contention and lets the OS page cache cache them aggressively.
- **Deletions**: Deletes append the doc ID to a `.del` bitset (or use `DocValues` for soft deletes in newer versions). Search filters out deleted IDs.
- **Segment merge**: Background threads merge small segments into larger ones, purge deleted docs, and reduce segment count. Heavy merging increases I/O and CPU — a common cause of latency spikes during bulk ingestion.

### Refresh vs Flush vs Merge

| Operation | Effect | Searchable? | Durable on disk? |
| --- | --- | --- | --- |
| **Refresh** | New segment in page cache | Yes | No (page cache only) |
| **Flush** | Lucene commit + fsync | Yes | Yes |
| **Merge** | Combine segments, purge deletes | Yes | Yes (background) |

---

## Cluster Coordination & Consensus

ES 7+ replaced Zen Discovery with a **cluster coordination** layer inspired by Raft (but not full Raft — optimized for ES's master-only metadata updates).

### Master Election & Cluster State

- **Single active master**: Only one elected master publishes cluster-state changes at a time.
- **Voting configuration**: A subset of master-eligible nodes form the voting configuration. A master must receive acknowledgments from a **majority of voting nodes** to publish a cluster-state update — this prevents split-brain.
- **Split-brain prevention**: Set `discovery.seed_hosts` and `cluster.initial_master_nodes` on first boot. Never run two independent clusters with the same cluster name on the same network.
- **Cluster state**: Contains routing tables, index mappings, shard allocation, node membership. Data nodes apply state updates locally; they do not participate in metadata voting unless master-eligible.

### Cluster Health

| Status | Meaning |
| --- | --- |
| **Green** | All primary and replica shards are assigned |
| **Yellow** | All primaries assigned, but some replicas are missing (still fully searchable) |
| **Red** | At least one primary shard is unassigned (data loss risk for that shard) |

---

## Performance & Scalability Optimization

### Deep Pagination Dilemma

- **Problem**: `from: 9990, size: 10` forces every shard to collect and rank 10,000 hits internally. With 10 shards, the coordinator merges 100,000 entries to return 10.
- **Solutions**:
  - **`search_after`**: Cursor-based pagination using the sort values of the last hit. Requires a stable, unique sort key (e.g., `timestamp` + `_shard_doc` as tiebreaker).
  - **Point in Time (PIT) + `search_after`**: Opens a consistent index snapshot so concurrent writes do not shift pages between requests.
  - Avoid **`scroll`** for user-facing pagination (designed for bulk reprocessing; holds resources open; deprecated in favor of PIT in modern ES).

### JVM Heap Management

- **Compressed OOPs**: Keep JVM heap **≤ ~31 GB** on 64-bit JVMs. Below ~32 GB, pointer compression doubles effective heap efficiency. Above 32 GB, compression disables and usable capacity can actually *decrease*.
- **50% rule**: Allocate ~50% of RAM to the ES JVM heap; leave the rest for the **OS page cache** (Lucene reads segments from page cache — this is critical for search performance).
- **Circuit breakers**: ES limits memory for aggregations, fielddata, and request sizes. Tripped breakers return `429` rather than OOM-killing the node.

### Shard Sizing & Capacity Planning

- Target **10–50 GB per shard** (rule of thumb; workload-dependent).
- Fewer, larger shards reduce cluster-state overhead; too few shards limit parallelism.
- Use `_cat/shards` and `_cat/indices` to monitor shard sizes and distribution.

---

## Common Challenges & System Design Trade-offs

| Challenge / Trade-off | Architectural Strategy / Remedy |
| --- | --- |
| **Near-real-time latency vs bulk throughput** | Increase `index.refresh_interval` to `30s` or `-1` (disabled) during bulk loads. Re-enable after. |
| **Over-sharding** | Each shard consumes file handles, heap overhead, and merge threads. Thousands of micro-shards bottleneck cluster-state propagation. |
| **Re-sharding limitations** | Routing depends on primary shard count. Resize via `_split`/`_shrink` (must meet size constraints) or `_reindex` into a new index and swap an alias. |
| **Time-series ingestion (logs/metrics)** | Use **ILM** + **Data Streams**. Roll over daily/weekly; transition Hot (SSD, active writes) → Warm (read-only) → Cold (minimal replicas) → Delete/Frozen. |
| **Mapping explosions** | Dynamic mapping of high-cardinality fields (e.g., UUID as `text`) bloats the cluster state. Use **dynamic templates**, explicit mappings, and `index.mapping.total_fields.limit`. |
| **Hot shards** | Custom routing, uneven tenant distribution, or too few shards cause write/read hotspots. Monitor per-shard CPU and indexing rate. |
| **Split-brain / dual master** | Proper seed hosts, dedicated masters, odd number of master-eligible nodes, and never `minimum_master_nodes` on mixed-version clusters (removed in ES 7). |

---

## Key System Design Interview Questions

### Q1: Why can't you easily change the number of primary shards after an index is created?

**Answer**: Document routing depends on the primary shard count: `hash(routing) % num_primary_shards`. Changing the divisor relocates every existing document to a different shard, breaking reads and updates. Options: `_reindex` into a new index with the desired shard count and swap an alias, or use `_split`/`_shrink` APIs when their constraints are met (e.g., `_split` requires the target count to be a multiple of the source count and the index must be read-only).

### Q2: What is the difference between a `match` query and a `term` query?

**Answer**: A `term` query is an exact match — no analyzer is applied. It looks up the literal token in the inverted index (best for `keyword` fields, IDs, status codes). A `match` query is full-text: the query string is analyzed (tokenized, lowercased, stemmed) and a boolean query runs over the resulting tokens against analyzed `text` fields.

### Q3: How does Elasticsearch handle a node failure in a cluster?

**Answer**:

1. The active master detects the failure via heartbeat/timeouts and updates cluster state.
2. If a **primary shard** was on the failed node, the master promotes an in-sync **replica** on a surviving node to primary.
3. Cluster health turns **Yellow** if replicas are still rebuilding; **Red** if no replica existed for a lost primary.
4. The master reallocates missing replica copies to surviving data nodes. Once replication catches up, health returns to **Green**.

### Q4: What happens between indexing a document and it appearing in search results?

**Answer**: The document is written to the translog and memory buffer immediately, but is **not searchable** until the next **refresh** (default 1s), when the buffer becomes a new Lucene segment in the page cache. A **flush** (separate operation) fsyncs segments to persistent disk and clears the translog — this is about durability, not search visibility.

### Q5: When would you choose Elasticsearch over a relational database?

**Answer**: Full-text search with relevance scoring, log/analytics aggregation at scale, fuzzy/prefix/phrase matching, and schema-flexible document ingestion. ES is a poor fit for strong transactional consistency, complex joins, or workloads requiring immediate read-your-writes across all nodes.

---

## Quick Reference

| Concept | Detail |
| --- | --- |
| **Default refresh interval** | `1s` (near-real-time search) |
| **Default replicas** | `1` (one copy per primary) |
| **Routing** | `hash(_routing) % num_primary_shards`; default `_routing` = `_id` |
| **Write ack** | `wait_for_active_shards` (default `1`) |
| **Translog flush threshold** | 512 MB (`index.translog.flush_threshold_size`) |
| **Heap sizing** | ≤ 31 GB; ~50% of total RAM |
| **Health** | Green / Yellow / Red |
| **Deep pagination** | `search_after` + PIT (not `from`/`size`) |
| **Bulk load tuning** | `refresh_interval: -1`, increase replicas after load |
| **Resize shards** | `_split`, `_shrink`, or `_reindex` + alias swap |
| **Time-series pattern** | Data Streams + ILM |
| **Exact vs full-text** | `term` / `keyword` vs `match` / `text` |
| **Concurrency** | `_seq_no` + `_primary_term` optimistic locking |
