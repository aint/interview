# System High Availability

Design for **surviving failure** without unacceptable downtime or data loss. End-to-end capacity and peak load live in [system_capacity.md](./system_capacity.md); data-layer replication and store limits in [data_store_selection.md](./data_store_selection.md) and [rdbms_internals.md](./rdbms_internals.md).

## Table of Contents

1. [Definitions & SLA Math](#definitions--sla-math)
2. [Failure Modes](#failure-modes)
3. [Eliminating Single Points of Failure](#eliminating-single-points-of-failure)
4. [Redundancy Patterns](#redundancy-patterns)
5. [Load Balancing & Health Checks](#load-balancing--health-checks)
6. [Timeouts, Retries & Circuit Breakers](#timeouts-retries--circuit-breakers)
7. [Graceful Degradation & Bulkheads](#graceful-degradation--bulkheads)
8. [Data Layer HA](#data-layer-ha)
9. [CAP & Consistency Under Failure](#cap--consistency-under-failure)
10. [Failover, RPO & RTO](#failover-rpo--rto)
11. [Multi-AZ & Multi-Region DR](#multi-az--multi-region-dr)
12. [Deploy Without Downtime](#deploy-without-downtime)
13. [Observability & Runbooks](#observability--runbooks)
14. [Chaos & Resilience Testing](#chaos--resilience-testing)
15. [HA Design Checklist](#ha-design-checklist)
16. [Worked Example](#worked-example)
17. [Interview Drill Questions](#interview-drill-questions)

---

## Definitions & SLA Math

**High availability (HA)** = the system keeps serving acceptable traffic when components fail. It is **not** the same as scale (handling peak load) — see [Availability vs Scale](./system_capacity.md#availability-vs-scale).

| Term | Meaning |
|------|---------|
| **Uptime / availability** | Fraction of time the service meets SLA (often measured monthly) |
| **MTBF** | Mean time between failures |
| **MTTR** | Mean time to **recover** (detect + failover + restore) |
| **Error budget** | Allowed downtime derived from SLA; spend on deploys, incidents, experiments |

**Availability formula:**

```
availability = MTBF / (MTBF + MTTR)
```

Lower **MTTR** (fast detection, automated failover) buys nines as effectively as more redundancy.

### Downtime budget (quick reference)

| SLA | Downtime / year | Downtime / month |
|-----|-----------------|------------------|
| 99.9% ("three nines") | ~8.8 hours | ~43 minutes |
| 99.99% ("four nines") | ~52 minutes | ~4.3 minutes |
| 99.999% ("five nines") | ~5 minutes | ~26 seconds |

**Interview line:** each extra nine is roughly **10× harder** — requires removing SPOFs, automation, and tested failover, not just doubling servers.

---

## Failure Modes

Plan for **expected** failures; assume **independent** component failure unless proven otherwise.

| Category | Examples | Typical mitigation |
|----------|----------|-------------------|
| **Hardware** | Disk, NIC, rack, AZ power | Multi-instance, multi-AZ, replicated storage |
| **Process / software** | OOM, deadlock, bad deploy | Health checks, rollbacks, circuit breakers |
| **Network** | Partition, DNS blip, TLS expiry | Retries with backoff, multi-path, cert automation |
| **Dependency** | DB slow, third-party API down | Timeouts, fallbacks, async queues |
| **Overload** | Traffic spike, hot key | Rate limits, autoscale, shed load |
| **Human / ops** | Misconfig, fat-finger delete | IaC, least privilege, backups, runbooks |
| **Cascading** | Retry storm, thread pool exhaustion | Bulkheads, jitter, circuit open |

**Correlated failures** (whole AZ, region, shared control plane) break "N independent servers" math — design for **AZ** and **region** loss explicitly.

---

## Eliminating Single Points of Failure

A **single point of failure (SPOF)** is any component whose failure stops the whole service.

```
                    ┌─── SPOF if only one ───┐
Client → DNS → LB → App → Cache → DB primary → Storage
                    └──────────────────────┘
```

| Layer | SPOF risk | HA pattern |
|-------|-----------|------------|
| DNS | One provider / stale TTL | Multi-provider or anycast; low TTL for failover |
| Load balancer | One LB VM | Managed LB, **≥2** LBs or cloud multi-AZ LB |
| App tier | One pod | **≥2** instances across **≥2 AZs** |
| Session state | Sticky in-memory | External store ([redis.md](./redis.md)) or stateless JWT |
| Cache | One Redis node | Replication + Sentinel or Redis Cluster |
| Database | One primary | Sync/async replicas, automatic promotion — [rdbms_internals.md](./rdbms_internals.md) |
| Queue | One broker | Kafka replication (`min.insync.replicas`) — [kafka.md](./kafka.md) |
| Control plane | One scheduler | Managed K8s multi-master; avoid DIY single etcd |

**Rule:** every tier on the critical path needs **redundancy + automated failure detection + a rehearsed recovery path**.

---

## Redundancy Patterns

| Pattern | Idea | Trade-off |
|---------|------|-----------|
| **N+1** | N needed for load + 1 spare | Survives one unit loss; may hit knee under peak + failure |
| **N+2 / 2× headroom** | Extra capacity for failure **and** spikes | Cost; common for four-nines targets |
| **Active-active** | All nodes serve traffic | Needs shared state or partition tolerance; split-brain risk |
| **Active-passive (hot standby)** | Standby ready, shares replication | Faster failover; standby cost |
| **Active-passive (cold)** | DR region idle | Cheaper; **minutes–hours** RTO |

**Quorum (odd counts):** 3 or 5 nodes for leader election / consensus so split votes resolve (2-of-3, 3-of-5).

```
        Active-active (stateless app)
   ┌─────────┐     ┌─────────┐
   │ App AZ-a│     │ App AZ-b│   both behind LB
   └────┬────┘     └────┬────┘
        └───────┬───────┘
                ▼
         shared DB / cache
```

---

## Load Balancing & Health Checks

LB distributes traffic and **removes unhealthy targets**.

| Check type | What it catches | Risk |
|------------|-----------------|------|
| **TCP** | Port open | App hung but port listens → false healthy |
| **HTTP `/health`** | Process up | Shallow check; DB down but `/health` OK |
| **Deep / readiness** | DB, cache, disk reachable | Slower; use for **readiness**, not liveness |
| **Out-of-band** | Synthetic probes, canaries | Catches partial failures LB misses |

**Kubernetes split:**

- **Liveness** — restart if deadlocked (don't include downstream deps).
- **Readiness** — remove from Service endpoints if can't serve (include critical deps).

**Sticky sessions** improve cache locality but tie users to a dying node — prefer shared session store.

**Connection draining:** on deploy or scale-in, stop **new** connections; wait for in-flight work before kill (see [Deploy Without Downtime](#deploy-without-downtime)).

---

## Timeouts, Retries & Circuit Breakers

Unbounded retries turn a partial outage into a **cascade**.

| Knob | Purpose |
|------|---------|
| **Timeout** | Bound wait; fail fast |
| **Retry** | Transient blips (503, connection reset) |
| **Exponential backoff + jitter** | Avoid synchronized retry spikes |
| **Retry budget / max attempts** | Cap total load on failing dependency |
| **Idempotency key** | Safe retries on writes |
| **Circuit breaker** | Open after error threshold → fail fast; half-open probe |

```
Closed ──(errors > threshold)──► Open ──(after cooldown)──► Half-open
   ▲                              │                           │
   └──────── success ◄────────────┴─────── failure ────────────┘
```

**Interview line:** "Retry only idempotent reads or writes with idempotency keys; always pair retries with timeouts and a circuit breaker on the hot path."

---

## Graceful Degradation & Bulkheads

When full service isn't possible, **reduce scope** instead of total failure.

| Strategy | Example |
|----------|---------|
| **Feature flags** | Disable recommendations; core checkout stays up |
| **Stale cache** | Serve last-known feed if DB slow |
| **Static fallback** | Default avatar, generic homepage |
| **Queue async work** | Accept order; email receipt later |
| **Shed load** | 429 on non-critical endpoints; prioritize paying users |

**Bulkhead:** isolate resources so one slow dependency can't exhaust all threads / connections.

```
┌─────────────────────────────────────┐
│ API pool (200 threads)              │
│  ├─ checkout bulkhead: 120 threads  │
│  └─ search bulkhead: 80 threads     │
└─────────────────────────────────────┘
```

Separate connection pools per downstream (checkout DB vs analytics DB).

---

## Data Layer HA

Most outages and consistency debates center on **state**.

| Mechanism | Durability | Failover speed | Consistency note |
|-----------|------------|----------------|------------------|
| **Sync replication** | Strong; no ack until replica has write | Fast promotion | Lower write latency; quorum needed |
| **Async replication** | Risk **lost writes** on failover | Fast | Higher throughput; **replica lag** |
| **Semi-sync** | At least one replica ack | Middle ground | Common in Postgres/MySQL |
| **Multi-leader** | Region-local writes | Complex | Conflict resolution required |
| **Object storage (S3)** | 11 nines durability (vendor) | N/A for compute | **Eventually consistent** listings; use versioning |

**Split-brain:** two nodes both believe they are primary → divergent writes. Prevent with **quorum**, **fencing** (STONITH), or **external coordinator** (etcd, ZooKeeper, KRaft).

Store-specific detail:

- SQL failover & lag: [rdbms_internals.md](./rdbms_internals.md#reliability-replication-and-failover)
- Redis Sentinel / Cluster: [redis.md](./redis.md#replication--high-availability)
- Kafka ISR & leader election: [kafka.md](./kafka.md#replication--high-availability)
- When to shard for blast radius: [data_store_selection.md](./data_store_selection.md)

**Backup ≠ HA:** backups help **RPO** for disasters; replication helps **MTTR** for node failure. Need both for serious SLAs.

---

## CAP & Consistency Under Failure

During a **network partition**, you choose between strong consistency and availability for writes (CAP — simplified for interviews).

| Choice | Behavior under partition | Example systems |
|--------|--------------------------|-----------------|
| **CP** | Reject writes / elect single writer | ZooKeeper, etcd, traditional RDBMS primary |
| **AP** | Stay available; reconcile later | Cassandra, DynamoDB (configurable) |

**PACELC extension:** if **P**artition, choose A or C; **E**lse (normal operation), choose **L**atency or **C**onsistency.

**Practical API patterns:**

- **Strong read after write** — route to primary or use linearizable read.
- **Monotonic read** — sticky to replica or read-your-writes session token.
- **Eventual consistency** — OK for likes count, search index; not for bank balance without care.

Say explicitly which user flows need which guarantee.

---

## Failover, RPO & RTO

| Metric | Meaning | Drives |
|--------|---------|--------|
| **RTO** (Recovery Time Objective) | Max acceptable **downtime** | Failover automation, hot standby, runbooks |
| **RPO** (Recovery Point Objective) | Max acceptable **data loss** | Sync replication, backup frequency, fsync policy |

```
Timeline of incident
──────────────────────────────────────────────────────►
        │◄── unplanned outage ──►│
        │◄────── RTO ──────────►│
   last good data               service restored
        │◄── RPO ──►│
        (async lag window)
```

| DR tier | RTO (typical) | RPO (typical) | Pattern |
|---------|---------------|---------------|---------|
| **Backup restore** | Hours | Minutes–hours | Cold S3 backup |
| **Warm standby** | Minutes | Seconds–minutes | Async replica, manual promote |
| **Hot standby** | Seconds | Near-zero | Sync replica, auto failover |
| **Active-active multi-region** | Near-zero | Near-zero (hard) | Global load balancer + conflict rules |

**Failover types:**

- **Automatic** — faster MTTR; needs fencing to avoid split-brain.
- **Manual** — safer for data; slower; unacceptable for strict RTO.

**Run the drill:** untested failover **will** exceed RTO in production.

---

## Multi-AZ & Multi-Region DR

### Multi-AZ (same region)

- Spread instances across **≥2 availability zones** (independent power/network).
- Use managed multi-AZ DB (primary + standby in another AZ).
- **Synchronous** replication within region is common; cross-AZ latency ~1–2 ms.

Survives **AZ failure**; not **region** failure.

### Multi-region

| Pattern | Use when | Caveat |
|---------|----------|--------|
| **Active-passive DR** | Regulatory DR, cost-sensitive | Failover DNS/LB; replication lag = RPO |
| **Active-active reads** | Global latency | Write routing, cache invalidation |
| **Active-active writes** | Strict global uptime | Conflict resolution, dual-write bugs |

Cross-region RTT **50–150 ms** — don't put synchronous quorum across regions on the latency-critical path unless using a global DB (see [data_store_selection.md](./data_store_selection.md)).

Capacity impact of geo patterns: [system_capacity.md — Multi-Region & Geo](./system_capacity.md#multi-region--geo).

---

## Deploy Without Downtime

Deploys are a leading cause of "availability" incidents.

| Strategy | Downtime | Risk | Notes |
|----------|----------|------|-------|
| **Rolling update** | None if backward compatible | Mixed versions live | Default in [k8s.md](./k8s.md); respect PDB |
| **Blue-green** | Switch instant | Big bang cutover | Two full stacks; fast rollback |
| **Canary** | None | Blast radius limited | 1% → 10% → 100% with metrics gates |

**Requirements for zero-downtime deploy:**

1. **Backward-compatible API** (additive schema, feature flags).
2. **Expand–contract migrations** — deploy code that reads old+new, migrate data, remove old.
3. **Health checks + draining** before terminating old pods.
4. **PodDisruptionBudget** — don't take down >1 replica at a time during node maintenance.

**Anti-patterns:** `DROP COLUMN` before all readers updated; killing all pods at once; deploy Friday without rollback plan.

---

## Observability & Runbooks

You can't hit low **MTTR** without knowing **what** failed.

| Signal | HA use |
|--------|--------|
| **Metrics** | Error rate, latency p99, saturation, replication lag |
| **Logs** | Correlation ID across services |
| **Traces** | Find slow dependency on critical path |
| **Synthetic checks** | Proactive "can user log in?" from multiple regions |
| **Alerting** | SLO-based pages (burn rate), not every blip |

**Runbook per failure mode:** detect → mitigate → failover steps → verify → comms → postmortem.

**On-call hygiene:** alert on **user impact**, not CPU > 80% alone.

---

## Chaos & Resilience Testing

Validate assumptions before production teaches you.

| Practice | What it proves |
|----------|----------------|
| **Game days** | Failover completes within RTO; runbooks accurate |
| **Chaos experiments** | Kill AZ, kill primary, inject latency |
| **Load + failure** | Survive peak **and** one AZ loss (N+1 under peak) |
| **Restore from backup** | RPO real, not theoretical |

Start in staging; blast radius controls; abort switches.

---

## HA Design Checklist

- [ ] SLA target stated (nines) → downtime budget understood
- [ ] Critical path drawn; **every tier** has redundancy or accepted SPOF
- [ ] **Multi-AZ** for compute and primary data; region DR if required
- [ ] Health checks: liveness vs readiness; deep checks on readiness
- [ ] Timeouts, bounded retries, circuit breakers on sync calls
- [ ] Bulkheads / pool isolation for noisy neighbors
- [ ] **RPO / RTO** defined; replication + backup match them
- [ ] Failover **automatic or manual** — explicit trade-off
- [ ] Split-brain prevention (quorum, fencing)
- [ ] Graceful degradation plan for dependency loss
- [ ] Zero-downtime deploy strategy; backward-compatible migrations
- [ ] Monitoring, SLOs, runbooks, failover drill scheduled
- [ ] Data layer details delegated to [rdbms_internals.md](./rdbms_internals.md) / [data_store_selection.md](./data_store_selection.md)

---

## Worked Example

**Prompt:** Payment API — **99.99%** SLA, **multi-AZ**, must not double-charge. Peak **5k QPS**, **50 ms** p99 budget.

### 1 — Translate SLA

99.99% ≈ **52 min/year** downtime budget → need automated failover, no manual-only recovery on critical path.

### 2 — Critical path & SPOFs

```
Client → Route53 → ALB (multi-AZ) → API pods (3 AZs) → Postgres primary + sync standby → Stripe API
```

Remove SPOFs: multi-AZ ALB, **≥3** API pods (one per AZ), RDS Multi-AZ or Patroni cluster, idempotent payment IDs.

### 3 — Failure handling

| Failure | Response |
|---------|----------|
| API pod crash | K8s restart; LB removes unhealthy |
| AZ loss | Remaining AZs carry load; **2× headroom** so 2 AZs absorb peak |
| DB primary down | Auto promote standby (**RTO ~60s**); sync rep → **RPO ≈ 0** |
| Stripe timeout | 2s timeout, circuit open, return 503 + retry-safe idempotency key |
| Deploy bad binary | Canary 5%; rollback on error-rate SLO burn |

### 4 — Consistency

Use **idempotency-Key** header + unique constraint on `(merchant_id, idempotency_key)` so retries after timeout don't duplicate charge.

### 5 — What you say in the interview

"I'd target 99.99% with multi-AZ stateless API, Postgres sync standby for RPO≈0, circuit breakers to Stripe, idempotent writes, canary deploys, and quarterly failover drills — scale math in [system_capacity.md](./system_capacity.md)."

---

## Interview Drill Questions

1. 99.9% vs 99.99% — downtime difference and **one** architecture change per nine?
2. Draw a three-tier app — label every SPOF and fix each.
3. Liveness vs readiness probes — what goes in each?
4. Why can retries make an outage worse? Name three mitigations.
5. Sync vs async DB replication — trade-off for **RPO** and write latency?
6. What is split-brain and how do you prevent it?
7. Active-active vs active-passive multi-region — when pick each?
8. RPO = 0 and RTO = 5 minutes — what replication and failover setup?
9. Dependency slow (not down) — circuit breaker vs timeout vs bulkhead?
10. How do you deploy schema + code change without downtime?

**Related drills:** [system_capacity.md — Interview Drill Questions](./system_capacity.md#interview-drill-questions) (Q8 availability), [rdbms_internals.md](./rdbms_internals.md) (replication lag)
