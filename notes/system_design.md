# System Design Framework

How to **run a design**: what to clarify, which decisions change the diagram, and where to spend depth. Numbers live in [system_capacity.md](./system_capacity.md), failure design in [system_high_availability.md](./system_high_availability.md), store choice in [data_store_selection.md](./data_store_selection.md).

## Table of Contents

1. [What to Focus On](#what-to-focus-on)
2. [Time Budget](#time-budget)
3. [Decision Order](#decision-order)
4. [Clarify and Scope](#clarify-and-scope)
5. [Functional Requirements](#functional-requirements)
6. [Non-Functional Requirements](#non-functional-requirements)
7. [Estimate Only What Changes the Design](#estimate-only-what-changes-the-design)
8. [Entities, Access Patterns, API](#entities-access-patterns-api)
9. [High-Level Design](#high-level-design)
10. [Building Blocks](#building-blocks)
11. [Pick the Deep Dive](#pick-the-deep-dive)
12. [Tradeoff Language](#tradeoff-language)
13. [Close: Failure, Bottleneck, Next](#close-failure-bottleneck-next)
14. [Prompt Archetypes](#prompt-archetypes)
15. [Traps](#traps)
16. [Interview Checklist](#interview-checklist)

---

## What to Focus On

Interviewers score **fit**, not a catalog of technologies. A correct simple design beats a distributed one that ignores the constraint.

Spend depth, in order, on:

1. **The core user path** — the one or two actions the product exists for (post, read feed, send message, redirect).
2. **The first thing that breaks** at the stated scale — write TPS, fan-out, hot key, connection count, media bandwidth, or cross-region latency.
3. **The correctness constraint you cannot hand-wave** — read-your-writes, ordering, no double charge, no lost message, freshness of search.
4. **One sentence per other box** — why it is there and what it owns.

Leave thin (name it, do not design it) unless the prompt *is* that system:

- Login, OAuth, password storage
- Admin, analytics, billing UI
- Kubernetes, CI, exact cloud SKUs
- Full schema of every table
- ML ranking internals

**Rule:** every box on the diagram earns its place from a requirement or a number. If you cannot say which, remove it.

---

## Time Budget

Typical loop is **35–45 minutes**. Drive it; do not wait to be walked.

| Minutes | Phase | Done when |
|---------|--------|-----------|
| 0–5 | Clarify | Users, 2–3 core actions, in/out of scope, consistency and latency called out |
| 5–8 | Requirements + rough scale | FR list, NFR numbers or explicit defaults, one QPS and one storage figure |
| 8–20 | High-level design | Read path and write path drawn; each box has a job |
| 20–35 | Deep dive | The unique or fragile part worked through (data model, fan-out, pipeline, failure) |
| 35–45 | Close | Bottleneck named, one failure story, tradeoff stated, "next" list if time is gone |

If the interviewer steers ("let's talk about the feed"), **drop the plan and follow**. The budget is a default, not a script to finish.

Do not spend the first half on requirements. Do not open with Kafka.

---

## Decision Order

Think in this order even when you talk in a different order. Later steps depend on earlier ones.

1. **Users and core flows** — who acts, what they wait for.
2. **NFRs that change shape** — latency, consistency, durability, availability, geo.
3. **Scale class** — single node, one primary + cache, or sharded / async. See [Estimate Only What Changes the Design](#estimate-only-what-changes-the-design).
4. **Access patterns** — which keys, how often, how fresh, how consistent.
5. **API and entities** — only for those patterns.
6. **Synchronous path** — what the user waits on.
7. **Asynchronous path** — what can finish after the response.
8. **Data ownership** — source of truth vs derived stores; partition key.
9. **Bottleneck** — name the saturated component and the lever.
10. **Failure of that component** — timeout, retry, fallback, data loss window.
11. **One deep dive** — the step that is not a standard box.

---

## Clarify and Scope

Restate the problem in one sentence, then ask only questions whose answers **move a box**.

### Product

- Who is the user (consumer, tenant, internal)?
- What are the two or three actions that must work on day one?
- Is the user waiting for the result, or is completion async acceptable?
- Read-heavy, write-heavy, or connection-heavy?
- One region or global users?

### Correctness

- After a write, must the same user see it immediately (**read-your-writes**)?
- Must other users see it immediately, or is seconds of lag fine?
- Is order per user, per conversation, or global?
- What is an acceptable loss (none for payments and messages; some for view counts and metrics)?

### Scale (only if they have a number)

- DAU or MAU, actions per user per day
- Payload size (text vs photo vs video)
- Retention
- Peak vs average (if unknown, use **3×**)

### If they say "you decide"

State the assumption and move:

> "I'll assume 50M DAU, 100:1 reads to writes, read-your-writes for the author, a few seconds of lag for everyone else, single region, 99.9% availability. Stop me if any of those are wrong."

### Scope line

Write three bullets of **out of scope** out loud. That is what keeps the diagram small.

---

## Functional Requirements

Phrase requirements as **observable behavior**, prioritized.

| Priority | Meaning | How many |
|----------|---------|----------|
| **Must** | The design is wrong without it | 3–5 |
| **Should** | Real, but a follow-on | 1–2 |
| **Out of scope** | Explicitly dropped | 2–4 |

Template:

```
Must
- User can create X and receive an id.
- User can read X by id within the latency target.
- User can list their own X, newest first.

Out of scope
- Edit history, admin moderation, recommendations, multi-region writes.
```

Tie each "must" to a path you will draw. A requirement with no path is decoration.

---

## Non-Functional Requirements

These are the requirements that **change the diagram**. Pick a number or a default; do not list twelve "-ilities".

| NFR | What it forces | Reasonable default if unspecified |
|-----|----------------|-----------------------------------|
| **Read latency** | Cache, CDN, denormalized read model, regional replicas | Interactive read p99 **< 200 ms** |
| **Write latency** | Sync path stays short; heavy work goes async | Ack p99 **< 300–500 ms** |
| **Availability** | No SPOF on the critical path, multi-AZ | **99.9%** (about 8.8 h/year) |
| **Durability** | Replication, WAL, queue acks before "saved" | User-generated writes are not silently dropped |
| **Consistency** | Primary read, quorum, or cache TTL | **Read-your-writes** for the actor; eventual for everyone else |
| **Freshness** | TTL, invalidation, push vs pull | Seconds acceptable unless the product is a ticker or inventory |
| **Scale** | Replicas, shards, async, CDN | Compute it; do not invent a global active-active design for small QPS |
| **Geo** | Where reads and writes land | Single region until they ask for global latency or disaster recovery |

**Consistency in one line:** the user's own write is visible to them on the next read; other users and search indexes can lag. Stronger than that (global linearizability, inventory that cannot oversell) must be called out, because it removes cache-first and async-ack designs.

Availability math, SPOFs, and failover: [system_high_availability.md](./system_high_availability.md). Do not re-derive nines in the design interview unless the prompt is about HA.

---

## Estimate Only What Changes the Design

Full workflow: [Estimation Workflow](./system_capacity.md#estimation-workflow). In the interview, compute the **three numbers that pick an architecture class**, then stop.

| Number | Formula (sketch) | Decision it unlocks |
|--------|------------------|---------------------|
| **Peak write TPS** | DAU × writes/day / 86400 × peak factor | One primary, or shard / queue |
| **Peak read QPS** | writes × read:write ratio, then after cache | Cache + replicas, or a separate read model |
| **Storage** | writes/day × record size × retention × replication | One disk, or tiering / object storage |

**Architecture class** (order of magnitude, not a quote):

| Scale signal | Default shape |
|--------------|----------------|
| Write TPS **< ~1k**, data fits one machine | One primary, one replica, app tier. Say so and move to the product problem. |
| Read QPS high, writes still fit one primary | Cache + read replicas. Working set in memory. |
| Write TPS **above one primary's knee** (~10k–40k for a tuned single-node SQL primary) | Shard, or append-only log + async materialization |
| Payload is media | Object storage + CDN. The database stores metadata only. |
| Clients hold connections (chat, presence) | Size **concurrent connections**, not QPS |

State assumptions in one breath: "50% DAU/MAU, 3× peak, 90% cache hit on the hot read." If the result is small, **do not distribute it**. Over-sharding a 100 QPS system is a miss.

Store limits and when to leave a single primary: [data_store_selection.md](./data_store_selection.md).

---

## Entities, Access Patterns, API

Design the **queries** before the tables. Schema that does not match a query will be wrong.

### Access patterns

Fill this before choosing indexes or a partition key:

| Query | Rate | Lookup key | Consistency | Freshness |
|-------|------|------------|-------------|-----------|
| Get X by id | high | `id` | read-your-writes for owner | immediate |
| List X by user, time-desc | high | `user_id` | eventual ok | seconds |
| Search text | medium | query string | stale index ok | seconds–minutes |

The **high-rate key** is the partition / cache key. A query you cannot point at a key is a future full-table scan.

### Entities

Nouns only, with ownership:

- **Source of truth** — the store you repair from (usually the OLTP row or an append-only log).
- **Derived** — cache, search index, feed, counters. These can be rebuilt.

Note what is immutable (events, messages) vs updated in place (profile, inventory). Immutable data is how you get a simple log, idempotent consumers, and cheap retries.

### API

Specify only the must-have operations. For each: verb, path or RPC name, caller, sync or async, idempotency key.

```
POST /v1/messages
  body: conversation_id, client_msg_id, body
  sync ack: { message_id, accepted_at }
  idempotent on (conversation_id, client_msg_id)

GET /v1/conversations/{id}/messages?before=<cursor>
  sync, cursor pagination, order is per conversation
```

Call out:

- **Idempotency** on any retryable write (client token or natural key).
- **Pagination** by cursor, not large offsets, for high-QPS lists.
- **Who is allowed** in one phrase ("member of the conversation"), not an auth subsystem.

---

## High-Level Design

Draw **two paths** if reads and writes differ. Label each arrow **sync** (user waits) or **async** (user does not).

```
                 ┌──────── CDN (static / media) ────────┐
Client → DNS → LB → API (stateless) → Cache → DB primary
                         │                  ↘ read replicas
                         │
                         └─ async → Queue → Workers → search / feed / object storage
```

### Defaults that are fine until a requirement breaks them

- **Stateless API** behind a load balancer. Session and rate-limit state live in a shared store, not in the process.
- **One service** until a boundary is forced (different scale, different failure domain, different team). "Microservices" is not a requirement.
- **Short sync path.** Parsing, authz, write to the source of truth, enqueue. Ranking, fan-out, transcode, search indexing, email happen on the queue.
- **Cache in front of the hot read**, with an explicit TTL or invalidation story. Patterns: [cache.md](./cache.md).
- **Object storage for blobs**; the database holds the id and metadata.
- **Database owns truth.** Cache, search, and feeds are derived.

### Per box, say four things

1. What it owns.
2. Sync or async.
3. What happens when it is slow or down.
4. How it scales (more stateless copies, partition, or it does not scale and that is ok).

### Data model on the diagram

- Partition key comes from the hottest access pattern (`user_id`, `conversation_id`), not from a random UUID, unless the lookup is by that id.
- Secondary reads that do not match the partition key need a second store, a global index, or an acceptance that they are rare.
- Hot keys (celebrity, global counter) do not shard away. Call them out and isolate them (cache, local aggregation, dedicated shard).

---

## Building Blocks

Reach for a block when a **signal** appears. Adding one without a signal is how designs get noisy.

| Signal | Block | Notes |
|--------|-------|-------|
| Same read, many times, staleness ok | Cache | [cache.md](./cache.md), [redis.md](./redis.md). State hit ratio; origin QPS is what is left. |
| Bytes are images, video, files | Object storage + CDN | DB stores the key. Multipart upload and async processing for large media. |
| Write spikes or slow downstream | Queue | [kafka.md](./kafka.md). Protects the sync path. Size consumers by drain rate, not by hope. |
| User must not wait for fan-out, email, index | Worker | Ack after durable enqueue. At-least-once delivery → **idempotent consumer**. |
| Full-text, filters, facets | Search index | [elasticsearch.md](./elasticsearch.md). It is derived; rebuild from the source of truth. |
| Write TPS above one primary | Shard or log | [data_store_selection.md](./data_store_selection.md). Pick the partition key from the access pattern. |
| Counters, presence, sessions, locks, rate limits | In-memory store | Redis. Know the durability story (AOF/replica vs lossy cache). |
| Abuse, multi-tenant noisy neighbor | Rate limit + backpressure | Limit at the edge. Shed load before the database falls over. |
| Client holds a socket | Connection gateway | Scale by **connection count**. Fan-in to a message log; do not put a DB write on every keystroke unless you must. |
| Cross-region readers | Regional replicas or cache | Writes stay on one primary until you accept conflict resolution. |
| Two stores must agree (charge + order) | Outbox or single transaction | Do not dual-write two systems and hope. |
| Need a unique sortable id at high TPS | Id generator | DB sequence at low TPS. Snowflake-style (time + node + sequence) when many writers cannot share one counter. |

**Coordination is expensive.** Prefer partition-local decisions (shard owns its keys, consumer owns its offset) over a global lock.

---

## Pick the Deep Dive

Say which part you are opening and why:

> "The API and primary table are standard. The part that breaks this design is fan-out on write, so I'll go through that."

Good deep dives are **specific to the prompt** or **the first bottleneck**. Weak deep dives are reciting how a load balancer works.

| If the prompt is really about… | Open this |
|--------------------------------|-----------|
| Fan-out (feed, notify) | Push vs pull vs hybrid, celebrity problem, storage amplification |
| Chat / presence | Connection gateways, per-conversation order, message log, read cursors |
| Upload / video | Chunked upload, object storage, async transcode, playback via CDN |
| Search | Index pipeline, freshness, what you do **not** put in the search engine |
| Ids at scale | Uniqueness, sortability, coordination avoidance |
| Money, inventory | Idempotency, single source of truth, oversell, reconciliation |
| Untrusted public traffic | Rate limit key (user, IP, token), where the limit is enforced |
| Global users | What is regional (reads, cache) vs what stays single-homed (writes) |

Inside the deep dive, cover: data layout, the request path step by step, the failure mode, and the tradeoff you refused.

---

## Tradeoff Language

Use a fixed shape so the answer is a decision, not a list of options.

> "I choose **X** because the requirement is **Y**. The cost is **Z**. I would switch to **W** if **V**."

| Decision | Choose the left when | Cost you should say out loud |
|----------|----------------------|-------------------------------|
| Sync ack vs async ack | User must know it is durable now | Sync path is only as available as the primary |
| Cache aside vs write-through | Reads dominate, slight staleness ok | Miss stampede, invalidation bugs |
| Push (fan-out on write) vs pull (fan-out on read) | Readers >> writers, or readers are offline | Push explodes storage for huge follower graphs; pull is slow for heavy readers |
| Normalize vs denormalize | Writes and consistency dominate | Denormalized reads are stale and must be repaired |
| Single primary vs shard | Writes fit one node | Shard splits transactions and unique constraints |
| SQL vs document / wide-column | Relations, transactions, moderate TPS | See [data_store_selection.md](./data_store_selection.md) |
| Queue vs direct call | Spike or slow work | Lag, duplicates, harder debugging |
| Single region vs multi-region active | Latency or DR was required | Conflict handling, replication lag, cost |

You do not need the "optimal" side. You need a side that **matches the NFR you stated**, plus the condition that would flip it.

---

## Close: Failure, Bottleneck, Next

Leave three minutes for this even if the deep dive is unfinished. It is where senior signal shows up.

### Bottleneck

Name one component and the lever:

> "At this QPS the primary writes are fine; the timeline read is the hotspot. I'd keep recent timelines in Redis and fall back to the pull path on miss."

### One failure story

Pick the dependency on the sync path:

- **Slow:** timeout, bounded retries with jitter, circuit breaker. No retry storm. See [Timeouts, Retries & Circuit Breakers](./system_high_availability.md#timeouts-retries--circuit-breakers).
- **Down:** what the user sees (error, degraded read, queued write) and whether a retry can double-apply (idempotency key).
- **Data:** replication lag, RPO if the primary dies before replica ack. Do not claim zero data loss unless the write waited for the replica or a quorum.

### Next, if they ask "what else?"

Keep a short tail. Do not open a new design.

- Metrics: latency, error rate, saturation, queue lag, replication lag
- Rate limits and hot keys
- Backup / restore
- Expand–contract schema changes
- The out-of-scope items you already named

---

## Prompt Archetypes

The archetype tells you **which row of the framework matters most**. Estimate and HA still happen; they are just not the story.

| Archetype | Dominant constraint | Deep dive | Usually enough elsewhere |
|-----------|---------------------|-----------|--------------------------|
| URL shortener | Redirect latency, id space | Cache + id generation | One SQL primary |
| Social feed | Fan-out, ranking freshness | Push / pull / hybrid | Object storage only if media is in scope |
| Chat | Connections, per-thread order | Gateway + message log | Don't model global total order |
| News / search | Query shape, index lag | Ingest pipeline + index | OLTP is the source of truth |
| File / video | Bandwidth, processing time | Upload pipeline + CDN | Metadata DB stays small |
| Rate limiter | Accuracy vs coordination | Local + centralized counters | Algorithm is the product |
| Metrics / logging | Write volume, retention, cost | Append log + rollups | Lossy aggregation may be acceptable |
| Payments / inventory | No double apply, no oversell | Idempotency + ledger or reservation | Do not cache the balance as truth |
| Notification | Fan-out, device tokens, retries | Queue + provider adapters | Dedup on notification id |
| Collaborative edit / presence | Conflict or last-write, fan-out | OT/CRDT **or** explicit last-write; say which | Don't pretend both are free |

If the prompt spans two archetypes (Instagram = feed + media), design the **must** path of each and deep-dive only one. Say which one you are deferring.

---

## Traps

- Starting from a stack ("we'll use Kafka, Cassandra, and Elasticsearch") before a requirement needs them.
- A single box labeled "database" with no read path and no key.
- Six synchronous service calls on the path the user is waiting for.
- Designing auth, feed ranking, and a data warehouse in the same 40 minutes.
- Global active-active writes when the requirement was "users in Europe should read quickly."
- "Exactly once" without saying what is actually guaranteed (idempotent effect, at-least-once delivery).
- Sharding key that does not match the query, then a scatter-gather on every request.
- Ignoring the hot key (one celebrity, one SKU, one counter).
- Cache with no TTL, no invalidation, and no stampede story.
- Forgetting pagination, idempotency, or what the client retries.
- Treating capacity math as the whole interview. The math picks the class; the product picks the shape.

---

## Interview Checklist

Use this as the last pass before you stop talking.

**Scope**

- [ ] Problem restated in one sentence
- [ ] 3–5 must-have behaviors and an explicit out-of-scope list
- [ ] Assumptions spoken where the interviewer did not specify

**Constraints**

- [ ] Latency, consistency, durability, availability each have a number or a default
- [ ] Read-your-writes vs eventual is explicit
- [ ] Peak write TPS, peak read QPS, and storage class stated (see [capacity checklist](./system_capacity.md#capacity-planning-checklist))

**Design**

- [ ] Access patterns listed before schema
- [ ] Read path and write path both drawn; sync vs async labeled
- [ ] Source of truth vs derived data named
- [ ] Partition or cache key matches the hot query
- [ ] Each extra box (cache, queue, search, CDN) tied to a signal

**Depth**

- [ ] One deep dive on the unique or fragile part
- [ ] Bottleneck named, with a lever
- [ ] One failure: timeout, user-visible behavior, idempotency
- [ ] Tradeoff in the form "X because Y, cost Z, switch if V"

**Tail**

- [ ] Hot key / skew considered
- [ ] Observability mentioned only as a tail (lag, errors, latency), not as the design
