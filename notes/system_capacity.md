# System Capacity & Scale

Back-of-the-envelope math for **end-to-end** capacity: traffic → storage → bandwidth → size each tier → find the bottleneck. Data-store pick and shard math live in [data_store_selection.md](./data_store_selection.md).

## Table of Contents
1. [Quick Reference Numbers](#quick-reference-numbers)
2. [Estimation Workflow](#estimation-workflow)
3. [Traffic: QPS, TPS & Peak Load](#traffic-qps-tps--peak-load)
4. [Storage & Retention](#storage--retention)
5. [Bandwidth](#bandwidth)
6. [Throughput Limits (Non-Data Tiers)](#throughput-limits-non-data-tiers)
7. [Hockey Stick Latency Curve](#hockey-stick-latency-curve)
8. [Sizing the App / Compute Tier](#sizing-the-app--compute-tier)
9. [End-to-End Bottleneck Walk](#end-to-end-bottleneck-walk)
10. [Fan-Out & Traffic Skew](#fan-out--traffic-skew)
11. [Queues & Backpressure](#queues--backpressure)
12. [CDN & Static / Media Capacity](#cdn--static--media-capacity)
13. [Multi-Region & Geo](#multi-region--geo)
14. [Connections & Long-Lived Clients](#connections--long-lived-clients)
15. [Availability vs Scale](#availability-vs-scale)
16. [Capacity Planning Checklist](#capacity-planning-checklist)
17. [Worked Example (Full Stack)](#worked-example-full-stack)
18. [Interview Drill Questions](#interview-drill-questions)

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

1. **Clarify scope** — reads vs writes, consistency, retention, geo, real-time vs async.
2. **Estimate traffic** — DAU/MAU → actions/day → `total_QPS` (peak for sizing).
3. **Split read/write** — ratio → `read_QPS`, `write_TPS` (see [data_store_selection.md](./data_store_selection.md) for data-layer limits).
4. **Estimate storage** — record size × writes/day × retention × replication.
5. **Estimate bandwidth** — peak QPS × payload; separate API vs media/CDN.
6. **Size non-DB tiers** — app instances, LB connections, queue consumers, CDN origin.
7. **Walk the path** — label the first saturated component (often not the DB).
8. **Propose fixes** — scale stateless tier, cache, async, shed load, then data layer.
9. **State assumptions** — "50% DAU/MAU, 3× peak, 5k RPS per app pod, 90% CDN hit ratio."

---

## Traffic: QPS, TPS & Peak Load

### Key terms

- **QPS**: any request (reads + writes).
- **TPS**: state-changing operations; often the **database** write bottleneck (not always 1:1 with API writes if async).
- **DAU / MAU** — typical DAU/MAU: social **40–60%**, SaaS **10–30%**, e-commerce **5–20%**.

### Formulas

**`total_QPS`** = all requests per second (reads + writes). Compute first, then split.

**From DAU:**

```
daily_actions     = DAU × actions_per_user_per_day
average_QPS       = daily_actions / 86,400   (round to 100,000 in interviews)
peak_QPS          = average_QPS × peak_factor
total_QPS         = peak_QPS                 (size tiers for peak)
```

Use `total_QPS = average_QPS` only for steady-state **cost**, not capacity limits.

**Peak factor**: **2×–5×**; default **3×** (time zones, spikes, viral).

**From concurrent users:**

```
total_QPS = concurrent_users × actions_per_user_per_second
```

Example: 1M users × 1 action/min → `1,000,000 / 60 ≈ 17,000` QPS.

### Read/write split

Ratio **R:W** (reads : writes):

```
read_ratio  = R / (R + W)
write_ratio = W / (R + W)
read_QPS    = total_QPS × read_ratio
write_TPS   = total_QPS × write_ratio   (API write rate; DB may be lower if async)
```

Example: `total_QPS = 15,000`, **100:1** → `read_QPS ≈ 14,850`, `write_TPS ≈ 149`.

Pass **`write_TPS`** (and storage/bandwidth) to [data_store_selection.md](./data_store_selection.md) for store selection.

---

## Storage & Retention

```
daily_new_data  = writes_per_day × avg_record_size
total_storage   = daily_new_data × retention_days × replication_factor
```

Add **20–30%** for indexes and metadata; **2–3×** growth buffer over 2–3 years.

| Factor | Typical range |
|--------|---------------|
| Replication factor | 2–3 (DB), 3 (Kafka) |
| Index overhead | +20–50% on raw size |
| Retention | 7 days (logs), years (user data), forever (compliance) |

**Hot** (30–90 days): SSD, cache, primary DB. **Cold**: S3/GCS archival.

---

## Bandwidth

```
ingress  = write_QPS × avg_upload_size
egress   = read_QPS × avg_response_size
total    = ingress + egress (+ CDN origin on cache miss)
Gbps     = bytes_per_second × 8 / 10^9
```

Example: 10,000 read QPS × 10 KB = 100 MB/s ≈ **0.8 Gbps egress** (API only).

Size **NIC / LB / NAT** for peak egress + headroom. Media bandwidth is usually **CDN**, not app servers (see [CDN & Static / Media Capacity](#cdn--static--media-capacity)).

---

## Throughput Limits (Non-Data Tiers)

Order-of-magnitude anchors — not guarantees. Data stores: [data_store_selection.md](./data_store_selection.md#store-throughput-limits).

| Tier | Typical limit | Main constraint |
|------|---------------|-----------------|
| App server (stateless) | **1k–10k RPS** / instance | CPU, threads, downstream I/O |
| Load balancer | **100k+ concurrent connections** | Connection table, TLS CPU |
| CDN edge | Absorbs most read bytes | Hit ratio; origin sees misses only |
| Kafka (cluster) | **Millions msg/s** | Partitions, `acks`, disk |
| SQS / Pub/Sub | Very high (soft quotas) | Visibility timeout, duplicates |

---

## Hockey Stick Latency Curve

Throughput does **not** scale linearly. p99 stays flat, then spikes at the **knee**:

```
latency
    │                              ╱
    │                            ╱
    │────────────────────────╱   ← knee
    └────────────────────────┴────────► load
```

Applies to **any** tier: app CPU, LB, Redis, DB partition, S3 prefix, Kafka partition.

**Interview line:** size at **~60–70% of measured knee** so spikes stay on the flat region.

Data-layer knees (WAL, hot partition): [data_store_selection.md](./data_store_selection.md#why-single-node-postgresmysql-write-tps-caps-10k40k).

---

## Sizing the App / Compute Tier

Stateless API tier scales horizontally until a **shared** dependency saturates (DB, Redis, queue).

```
instances = ceil(peak_QPS / RPS_per_instance) × headroom_factor
```

| Assumption | Typical value |
|------------|---------------|
| RPS per instance | **2k–5k** (light JSON); **500–1k** (heavy CPU/serialization) |
| Headroom factor | **2×** (deploys, AZ failure, spikes) |

Example: `peak_QPS = 15,000`, **3k RPS**/pod → `15,000 / 3,000 = 5` pods × **2×** → **~10 instances**.

Also check:

- **CPU per request** — image resize, PDF gen → far fewer RPS per pod.
- **Downstream pool** — `DB_connections ≤ instances × pool_size`; don't melt Postgres with `instances × 100` connections (use PgBouncer).
- **Memory** — large payloads or session state break "stateless" assumption.

---

## End-to-End Bottleneck Walk

Walk the **critical path** for the dominant use case (usually peak **read** or peak **write**):

```
Client → DNS → CDN (static?) → LB → API pods → [Redis] → [DB / queue / search]
```

| Step | Capacity question |
|------|-------------------|
| CDN | Hit ratio high enough? Origin QPS = `read_QPS × (1 - hit_ratio)` |
| LB | Enough connections? TLS termination CPU? |
| API | `peak_QPS` vs `instances × RPS_per_instance` |
| Redis | Hot keys? Memory? **~100k–1M ops/s** per node (see [redis.md](./redis.md)) |
| DB | `write_TPS` vs primary knee — [data_store_selection.md](./data_store_selection.md) |
| Queue | Producer rate vs consumer `partitions × consume_rate` |

**Rule:** fixing a non-bottleneck tier wastes money; find **one** saturated resource first.

Example (read-heavy API, 90% CDN hit): origin `1,485 QPS` vs API `15,000 QPS` → size **app tier** first; DB sees cache-filtered load.

---

## Fan-Out & Traffic Skew

**Fan-out** = one action triggers N downstream calls (push feed, broadcast notification, post to followers).

```
downstream_load = trigger_rate × fan_out_count
```

Example: 1k posts/s × 10k followers each (push) = **10M deliveries/s** — impossible synchronously → **pull feed**, **fan-out on write** only for small accounts, or **hybrid** (fan-out for <1k followers, pull for celebrities).

**Skew (celebrity / hot key):**

| Symptom | Mitigation |
|---------|------------|
| One user with 100M followers | Pull model, precomputed feeds, cache celebrity timeline |
| One shard / partition hot | Key sharding (`user_id#bucket`), random write spreading |
| 90/10 traffic | Don't size for average; size for **top percentile** |

Always ask: **"Is load evenly distributed across keys and time?"**

---

## Queues & Backpressure

Queues **absorb spikes** and **decouple** producers from consumers (notifications, indexing, analytics).

```
sustainable_drain_rate = consumers × messages_per_second_per_consumer
```

If `producer_peak > drain_rate`, backlog grows without bound → **lag**, aged messages, OOM risk.

| Knob | Effect |
|------|--------|
| More partitions / consumers | Higher drain rate (up to partition count) |
| Batch consume | Higher throughput, worse latency |
| Shed load | Drop, sample, or delay non-critical work |
| Rate limit at API | Protect queue and DB |

**Kafka partition hint:** `partitions ≥ peak_parallel_consumer_throughput need`.

**Backpressure:** when downstream is slow, **429 / queue full / circuit open** — don't buffer forever in memory.

See [kafka.md](./kafka.md) for producer `acks`, retention, and replay.

---

## CDN & Static / Media Capacity

CDNs scale **egress at the edge**; you size **origin** for cache misses.

```
origin_QPS   = client_read_QPS × (1 - cache_hit_ratio)
origin_Gbps  = origin_QPS × avg_asset_size × 8 / 10^9
```

| Asset type | Typical strategy |
|------------|------------------|
| JS/CSS/images | Long TTL, versioned URLs → **>90%** hit ratio |
| User uploads (photos/video) | S3 + CDN; **immutable URLs** after upload |
| Dynamic API JSON | Usually **not** CDN-cached (short TTL or bypass) |

**S3 prefix limit:** **~3,500 PUT/s per prefix** — hash prefixes for bulk upload. Details in [data_store_selection.md](./data_store_selection.md).

**Interview line:** "Billions of image views hit CDN; origin only sees misses and uploads."

---

## Multi-Region & Geo

| Pattern | Reads | Writes | Trade-off |
|---------|-------|--------|-----------|
| Single region + CDN | Global via CDN | One primary region | Simple; cross-region RTT on API |
| Active-passive DR | Failover region idle | Replication lag RPO/RTO | Cheaper; failover drill required |
| Multi-region active | Read replicas / caches local | Conflict resolution, dual-write risk | Lower read latency; ops complexity |
| Global DB (Spanner, etc.) | Local reads | Consensus latency on write | See [data_store_selection.md](./data_store_selection.md) |

**Capacity impact:**

- Cross-region RTT **50–150 ms** adds to every cache miss and uncached read.
- **Replicate data** → storage × regions; **egress** between regions costs money.
- Size each region for **local peak**, not global average, unless traffic follows the sun perfectly.

---

## Connections & Long-Lived Clients

| Model | Sizing concern |
|-------|----------------|
| Short HTTP request | QPS-driven (see app tier) |
| HTTP keep-alive | Fewer handshakes; **LB connection table** |
| WebSocket / SSE | **Concurrent connections** dominate, not QPS |
| Mobile push | Fan-out to device tokens via queue + provider APIs |

```
required_LB_connections ≈ concurrent_users × connections_per_user
```

Example: 2M WebSocket users → **2M** LB/backend connections (not 2M QPS). Shard connection state across gateway pods.

**Sticky sessions** reduce flexibility; prefer shared Redis for session if possible.

---

## Availability vs Scale

Scale (handle load) and availability (survive failure) overlap but differ.

| Technique | Scale | Availability |
|-----------|-------|--------------|
| More stateless app pods | ✓ | ✓ (lose pods) |
| Multi-AZ replicas | Read scale | ✓ AZ failure |
| Multi-region | Geo latency | ✓ region failure |
| Queue buffering | Spike absorption | At-least-once; lag under failure |
| **N+1 / 2× headroom** | Spike room | One AZ/node loss without knee |

**Extra nine (99.9 → 99.99):** remove SPOFs, automated failover, chaos testing, avoid manual deploys — not just "more servers." Deep dive: [system_high_availability.md](./system_high_availability.md).

Checklist ties to [Capacity Planning Checklist](#capacity-planning-checklist).

---

## Capacity Planning Checklist

- [ ] Assumptions stated (DAU, actions/user, peak factor, payload sizes)
- [ ] Peak `total_QPS`, `read_QPS`, `write_TPS` computed
- [ ] Storage + bandwidth (+ CDN origin) estimated
- [ ] App instances: `peak_QPS / RPS_per_pod` with **2×** headroom
- [ ] End-to-end path walked; **bottleneck named**
- [ ] Fan-out / skew / hot key risks called out
- [ ] Queue producer vs consumer drain rate (if async)
- [ ] CDN hit ratio → origin load separated from API QPS
- [ ] Multi-region / AZ / SLA requirements explicit
- [ ] Long-lived connections sized (if WebSocket/push)
- [ ] Data layer passed to [data_store_selection.md](./data_store_selection.md) checklist

---

## Worked Example (Full Stack)

**Prompt:** Social app — **100M MAU**, **50% DAU/MAU**, **100 requests/user/day** (timeline reads + posts), **100:1** read:write, **3×** peak. JSON **10 KB**/read, **500 B** write. **90% CDN hit** on static; API is dynamic.

### 1 — Traffic

```
DAU           = 100M × 50%              = 50M
daily_actions = 50M × 100               = 5B/day
average_QPS   = 5B / 100,000            = 50,000
total_QPS     = 50,000 × 3              = 150,000 peak
read_QPS      = 150,000 × (100/101)      ≈ 148,500
write_TPS     = 150,000 × (1/101)       ≈ 1,485
```

### 2 — Storage (writes only, 5-year retention, 3× replication)

```
daily_new   = 50M users × 10 writes × 500 B  ≈ 250 GB/day (posts/metadata)
5-year      ≈ 450 TB raw × 3 rep             ≈ 1.3 PB (+ indexes → ~1.5 PB)
```

Media: store on **S3 + CDN**, not OLTP row size above.

### 3 — Bandwidth (API JSON)

```
egress ≈ 148,500 × 10 KB  ≈ 1.4 GB/s  ≈ 11 Gbps
ingress ≈ 1,485 × 500 B  ≈ 0.7 MB/s  (negligible vs egress)
```

Plan **per-AZ egress** and LB capacity for **~11 Gbps** API (+ CDN for static).

### 4 — App tier

```
150,000 / 3,000 RPS per pod  = 50 pods × 2× headroom  → ~100 stateless API instances (order of magnitude)
```

### 5 — CDN / origin

```
origin_static_QPS ≈ image_read_QPS × 0.1   (if 90% hit; often separate from timeline API)
```

Timeline API stays dynamic; user photos served from CDN.

### 6 — Fan-out (push notification on post)

If 10% of posts trigger notify **1k followers** each:

```
notify_rate ≈ 148 write/s × 0.1 × 1,000  ≈ 14,800 messages/s  → queue + workers, not synchronous API
```

### 7 — Bottleneck call

| Tier | Load | Verdict |
|------|------|---------|
| App | 150k QPS | Needs ~100 pods — feasible |
| API egress | ~11 Gbps | Size network/LB |
| DB writes | ~1,485 TPS (if sync) | Within single Postgres knee with headroom — see data doc |
| Redis | Hot timelines | Cache + fan-out design |

**Say:** "I'd CDN-static assets, scale ~100 API pods, async notifications via Kafka, Redis for hot feeds, Postgres for source of truth — details in [data_store_selection.md](./data_store_selection.md)."

---

## Interview Drill Questions

1. 10M DAU, 200 requests/user/day, 3× peak — what is `total_QPS`? Split **10:1** read:write.
2. Same traffic — how many app pods at **2k RPS** each with **2×** headroom?
3. 10,000 read QPS × 20 KB response — egress in **Gbps**?
4. CDN **95%** hit ratio, 1M asset reads/s — origin QPS?
5. Celebrity posts 1k/s; push to 10M followers each — why sync fan-out fails; three mitigations?
6. Producers **50k msg/s**, consumers **5k msg/s** each, 8 consumers — does queue backlog grow?
7. 2M concurrent WebSockets — what do you size: QPS or connections?
8. 99.9% vs 99.99% — downtime difference and one architecture change per nine?
9. Single-region API + global users — what breaks first, latency or throughput?
10. Walk the path for a **read-heavy** product — where do you put CDN, Redis, and DB?

**Data-layer drills:** [data_store_selection.md](./data_store_selection.md#interview-drill-questions)
