# Message Queues

System-design prep: when a queue belongs in the diagram, what guarantee you are actually buying, and the failure modes interviewers probe. Kafka internals are in [kafka.md](./kafka.md). Where the queue sits on the sync vs async path is in [system_design.md](./system_design.md).

## Table of Contents

1. [Mental Model](#mental-model)
2. [When to Use a Queue](#when-to-use-a-queue)
3. [When a Queue Hurts](#when-a-queue-hurts)
4. [Broker Comparison](#broker-comparison)
5. [Durability](#durability)
6. [Delivery Semantics](#delivery-semantics)
7. [Latency-Sensitive Systems](#latency-sensitive-systems)
8. [Poison Messages and DLQ](#poison-messages-and-dlq)
9. [Partition Key and Ordering](#partition-key-and-ordering)
10. [Patterns That Come Up](#patterns-that-come-up)
11. [What to Say in the Interview](#what-to-say-in-the-interview)

---

## Mental Model

A message queue is a **durable buffer between a producer that emits work and a consumer that does it**. The producer returns once the broker has accepted the message. The consumer pulls (or is pushed) later, does the work, and acknowledges.

Two shapes show up in designs:

| Shape | Contract | Typical brokers |
| ----- | -------- | --------------- |
| **Task queue** | One message, one worker. Competing consumers. Message leaves the queue after ack. | RabbitMQ, SQS, ActiveMQ, NATS work queues |
| **Log / pub-sub** | Append-only history. Many independent readers. Retention, not deletion, bounds the data. | Kafka, Pulsar, Kinesis, Redis Streams |

Say which shape you mean. "We'll put it on Kafka" is a product choice. "This is a task queue so email and search index can fail independently of the write" is a design choice.

Three timestamps matter:

1. **Produce** — caller handed the message to the broker.
2. **Broker ack** — broker will not lose it across the failure mode you configured (disk, replica, or both).
3. **Consumer ack** — worker finished the side effect.

Anything between 2 and 3 is **lag**. Lag is the metric, not a bug.

---

## When to Use a Queue

Use a queue when at least one of these is a real requirement. Tie the box on the diagram to the signal, the same way [system_design.md](./system_design.md) treats cache and CDN.

### Async work

The user needs to know the command was **accepted**, and the heavy step can finish later: email, push, search indexing, feed fan-out, image transcode, webhook delivery, report generation.

Sync path: validate, write the source of truth, enqueue, return. Workers own the slow part.

### Bursty traffic

Producers spike (flash sale, top of the hour, webhook storm). Downstream has a steady capacity (payment API, single-primary DB, third party with a rate limit). The queue absorbs the spike and workers drain at the downstream's rate.

Size the design by **drain rate and acceptable lag**, and by how large the backlog gets during the peak. A queue does not add capacity. It moves the overload from "requests fail now" to "work finishes late."

### Decoupling

Producer and consumer deploy, scale, and fail separately. The producer does not call N downstreams. New consumers subscribe later (billing, analytics, audit) without changing the write path.

This is the main reason a log beats a task queue: one event, many independent consumer groups, each with its own lag and retry policy.

### Reliability

If the consumer is down, redeploying, or the dependency it calls returns 503, the message waits instead of failing the user request. Retries, backoff, and a dead-letter queue live here, off the user-facing path.

The guarantee is only as strong as the produce ack. A process that crashes after "business success" and before a durable enqueue has lost the work.

### Fan-out

One fact, many effects. Order placed → charge, reserve inventory, notify, index, analytics. Each effect gets its own consumer (Kafka consumer group, SNS→SQS subscription, Rabbit exchange binding) so one slow effect does not block the others.

---

## When a Queue Hurts

Leave it off the diagram when the requirement fights the buffer.

| Situation | What to do instead |
| --------- | ------------------ |
| Caller needs the **result** to render the response | Synchronous call. A queue plus request/reply adds a correlation id, a reply queue, and a timeout, and still fails when the worker is slow. |
| The next read must show this write for **this user**, and there is no other store | Write the source of truth on the sync path. Enqueue only the side effects. |
| Strong invariants (no oversell, no double charge) | A transaction or a conditional write in the database. The queue delivers side effects after the decision is committed. |
| Latency budget is a few milliseconds | Direct RPC or in-process work. Batching and lag dominate once a broker is on the path. |
| One small service, modest QPS, one consumer | A job table polled by workers is enough until you need fan-out, replay, or multi-service decoupling. |

---

## Broker Comparison

Order-of-magnitude interview map. Throughput depends on message size, ack mode, and hardware.

| | Kafka | RabbitMQ | SQS | Redis Streams | Pulsar | Kinesis |
| --- | --- | --- | --- | --- | --- | --- |
| **Model** | Replicated log | Broker: exchanges + queues | Managed task queue | In-memory log, optional AOF | Log (BookKeeper) + queue semantics | Managed log |
| **Consumption** | Pull, offset per group; message kept until retention | Push or pull; deleted after ack | Pull; hidden during visibility timeout, deleted on delete | Consumer groups, pending entries list | Subscriptions (exclusive, shared, failover, key-shared) | Pull, shard iterators |
| **Ordering** | Per partition | Per queue while a single consumer is active; lost across competing consumers | Standard: best effort. FIFO: per message group id | Per stream | Per partition; key-shared sub orders per key | Per shard |
| **Replay** | Reset offset | Not the model (broker already deleted it) | Gone after delete | `XRANGE` until trimmed | Reset cursor | Within retention |
| **Fan-out** | Extra consumer groups, same topic | Exchanges (fanout, topic, direct) | SNS → many queues | Several groups on one stream | Several subscriptions | Several consumers per shard need enhanced fan-out or a fan-out service |
| **Ops** | You run the cluster (or MSK / Confluent) | You run it; quorum queues for HA | Fully managed | Usually already there beside cache | Heavy (broker + BookKeeper) | Fully managed |
| **Sweet spot** | Event backbone, high throughput, replay, many consumers | Task queues, routing, priority, per-message TTL | AWS async work, little ops | Short-lived jobs, cache-adjacent streams | Multi-tenant streaming, geo-replication | AWS log when you do not want to run Kafka |
| **Watch** | Partition count caps classic consumers; hot keys | Queue depth and memory; mirroring vs quorum queues | 256 KB payload; FIFO throughput ceiling; visibility timeout | Pub/Sub is not durable; data lives in RAM | Operational complexity | Shard splits, per-shard write cap (~1 MB/s or 1k records/s) |

**How to pick in a design review**

- **Many services must read the same fact, and you may replay** → Kafka, Pulsar, or Kinesis.
- **One worker pool, competing consumers, AWS, no replay** → SQS. Add SNS in front when more than one consumer type needs a copy.
- **Routing, priorities, TTLs, classic task queue, self-hosted** → RabbitMQ (quorum queues).
- **Ephemeral, already on Redis, loss is acceptable** → Redis Streams. Redis Pub/Sub has no persistence and no backlog; it is a broadcast, and a queue only in the loosest sense.

NATS (Core) is a low-latency pub/sub with at-most-once delivery. JetStream adds a durable log. Mention it when the interviewer cares about sub-millisecond messaging and small messages. IBM MQ shows up when the requirement is XA transactions with a mainframe or a database — rare in a whiteboard design, real in banks.

---

## Durability

"The queue is durable" is incomplete. Spell out **which failure** the message survives, and **which ack** the producer waited for.

| Level | Survives | Typical config | Does not survive |
| ----- | -------- | -------------- | ---------------- |
| **Memory only** | Nothing, or only a process restart if the OS keeps the page | Rabbit non-persistent; Redis Pub/Sub; NATS Core | Broker restart, node loss |
| **Disk on one node** | Process restart | Rabbit persistent message + durable queue; Redis AOF `appendfsync everysec` / `always` | Disk loss, that node dies before replication |
| **Replicated** | Loss of a minority of brokers | Kafka `acks=all` and `min.insync.replicas`; Rabbit quorum queues; SQS (multi-AZ by default); Pulsar bookie ensemble | Datacenter loss, unless you replicate across regions |

Producer-side rules that interviewers listen for:

- **Wait for the broker ack** before telling the user "saved" or before deleting the source row. Fire-and-forget produce is at-most-once.
- **Kafka:** `acks=all` plus ISR. `acks=1` returns when the leader has it; a leader crash can drop it. Details in [kafka.md](./kafka.md).
- **RabbitMQ:** three switches, all required — durable queue, persistent messages, publisher confirms. A persistent message on a non-durable queue is gone on restart. A confirm you did not wait for is a lie.
- **SQS:** `SendMessage` success means the message is stored redundantly. Durability is the default; the knob you actually tune is the **visibility timeout** and the redrive policy.
- **Redis:** a stream plus AOF can still lose the last second (`appendfsync everysec`). Fine for a feed fan-out buffer. Wrong for payments.

Replication is not a transaction with your database. The crash window between "row committed" and "message acked" is the [outbox](#outbox).

---

## Delivery Semantics

Brokers give you one of these. End-to-end behavior also depends on when the consumer acks and whether the side effect is idempotent.

| Semantic | What you observe | How you get it | Fits |
| -------- | ---------------- | -------------- | ---- |
| **At most once** | Some messages vanish. Re-delivery is rare. | Produce without waiting for ack, or ack **before** the side effect. | Metrics, best-effort telemetry, lossy notifications. |
| **At least once** | Every message is delivered. Crashes cause **duplicates**. | Broker holds the message until the consumer acks **after** the side effect. On timeout, crash, or lost ack, it redelivers. | Almost every business workflow. |
| **Exactly once** | One effect per logical message, **inside a defined boundary**. | Idempotent producer + transactions (Kafka EOS), or FIFO dedup id (SQS), **plus** an idempotent side effect. | Pipelines that read and write the same log. External HTTP calls need their own idempotency key. |

Default design sentence:

> The broker is at-least-once. The consumer is idempotent, keyed by `order_id` or an idempotency key stored beside the side effect. A retry is a no-op.

Why duplicates happen even with a careful consumer:

1. Worker writes the email provider, then crashes before ack.
2. Visibility timeout expires (SQS) or the session times out (Kafka), and another worker gets the same message.
3. Producer retries a publish that actually succeeded (timeout on the ack).

Kafka's exactly-once scope is **Kafka-to-Kafka** (idempotent producer, transactions, read-process-write). A transaction does not make a Stripe charge exactly-once. The charge needs `Idempotency-Key`. SQS FIFO deduplicates produces with the same message dedup id for a short window (5 minutes by default); the consumer can still see a redelivery if it fails to delete.

**Ack timing**

| Ack | Effect |
| --- | ------ |
| Before processing | At most once. Fast, lossy. |
| After processing | At least once. Safe default. |
| Batch / auto-commit on a timer | At least once, and a crash redoes the whole uncommitted batch. Kafka auto-commit is this. |

---

## Latency-Sensitive Systems

A queue trades **tail latency and freshness** for **isolation and smoothing**. Put numbers on which path the user is waiting for.

### Keep the queue off the path the user is timing

| Path | User-visible latency | Queue's job |
| ---- | -------------------- | ----------- |
| **Sync command** | p99 of API (often 100–500 ms) | Not on it, or only a durable enqueue of a few ms after the DB commit |
| **Async completion** | "Done" when the worker finishes — seconds acceptable | Absorb work; SLO is **lag**, measured separately |
| **Read-your-writes** | The user's next GET shows their write | Served from the DB or cache updated on the sync path, while the queue catches search and fan-out up |

Returning HTTP 202 "accepted" is honest. Returning 200 "done" before the worker runs is a product bug you will defend poorly in the interview.

### Where the milliseconds go

- Extra network hop to the broker.
- **Batching.** Kafka `linger.ms` and producer batch size raise throughput and add wait. For a latency-sensitive produce, linger stays near 0 and batches stay small; you pay with lower throughput.
- **Consumer lag.** A healthy queue at 0 lag still has one poll interval of delay. A backed-up queue adds minutes. That delay is invisible to the API dashboard unless you graph lag (consumer group lag, `ApproximateAgeOfOldestMessage`, queue depth).
- **Head-of-line blocking.** Ordered processing inside a partition means one slow or poison message holds every later message for that key. Competing consumers on an unordered queue avoid this and give up per-key order.
- **Retries.** Exponential backoff on a hot message inflates end-to-end time. Cap attempts and park the message (DLQ) so the rest of the partition moves.

### If the product really is latency-sensitive

Chat fan-out, presence, in-game events, trading signals:

- Prefer a direct pub/sub (NATS Core, Redis Pub/Sub) when **dropping** a stale message is better than delivering it late.
- If you need the log, isolate a short topic with small batches, enough partitions, and a consumer whose only job is the hot path. Analytics reads the same topic in another group and is allowed to lag.
- Do not share a partition or a queue between "render the typing indicator" and "build the nightly aggregate."

Backpressure: when lag crosses a bound, shed load on the producer (429, drop best-effort events), scale consumers, or split the topic. An unbounded queue hides the overload until disk fills or the product is minutes late.

---

## Poison Messages and DLQ

A **poison message** fails every time it is attempted: schema the worker cannot parse, a bug on one payload, a referenced row that will never exist. A **transient failure** (timeout, 503, deadlock) succeeds on a later attempt. Treat them differently.

Without a cap, a poison message:

- burns CPU in a retry loop, or
- **blocks the partition** if you process in order and refuse to skip it.

**Dead-letter queue (DLQ):** after N attempts, move the message aside, alert, keep consuming.

| Broker | Mechanism |
| ------ | --------- |
| **SQS** | Visibility timeout hides the message while you work. If you do not delete it, it reappears. Redrive policy sends it to a DLQ after `maxReceiveCount`. |
| **RabbitMQ** | `nack` / reject with requeue, or a dead-letter exchange. Quorum queues track delivery count. |
| **Kafka** | The broker will redeliver forever if you do not commit. You build retry topics (attempt 1, 2, 3 with delay) and a DLQ topic, or use a framework that does. QfK can `REJECT` and archive after an attempt limit — see [kafka.md](./kafka.md). |

Practical policy:

1. **Classify the error.** Permanent (bad payload, 4xx-equivalent) → DLQ immediately. Transient → retry with exponential backoff and jitter.
2. **Cap attempts** (often 3–10). Record the last error and the original payload on the DLQ message.
3. **Alert on DLQ depth and age**, not only on consumer error rate. A quiet DLQ filling up is the page.
4. **Replay is a deliberate action** after the bug fix: move DLQ → main queue, or reset a consumer. Replaying blindly repeats the outage.
5. **Idempotency still applies.** Attempts 1..N-1 may have half-finished (email sent, DB not updated).

Ordering interaction: if partition key = `user_id` and one of that user's events is poison, every later event for that user waits behind retries. DLQ that one message (or that key) so the partition proceeds. Say this tradeoff out loud: you are choosing **progress** over **gapless order** for that key.

---

## Partition Key and Ordering

Ordering and parallelism are the same decision.

- A **partition** (Kafka, Pulsar, Kinesis shard, SQS FIFO message group) is the unit of order and, usually, the unit of serial consumption.
- The **partition key** picks the partition: `hash(key) % N`. Same key → same partition → one ordered log for that entity.
- No key → round-robin. Better balance. No cross-message order.
- Global order means **one partition**, which means one consumer doing the ordered work. That is a throughput ceiling. Interviewers who ask for "totally ordered events" at high QPS want you to narrow the scope: order **per order**, **per conversation**, **per account**, not across the whole topic.

### Choosing the key

| Key | Works when | Breaks when |
| --- | ---------- | ----------- |
| `order_id`, `conversation_id` | Updates for one entity must apply in order, and cardinality is high | You also needed order across entities |
| `user_id`, `tenant_id` | Per-user sequence, and no tenant dominates traffic | One celebrity or one tenant is a **hot partition**: one partition's throughput cap, everyone else idle |
| Constant (`"all"`) | You truly need global order and the rate fits one partition | Anything above that rate |
| Timestamp, "random but we still want order" | — | All current traffic hashes into a narrow band, or order is meaningless |

Hot key mitigations: subdivide (`tenant_id + bucket`), isolate the whale on its own topic, or drop order for that flow and use idempotent writes.

**Adding partitions reshuffles keys** for `hash(key) % N`. In-flight per-key order breaks across the migration unless you use a partitioner that sticks old keys (Kafka sticky / explicit partition map) or you accept a short reorder window.

### Competing consumers vs partitions

- **Classic Kafka:** at most one consumer in a group per partition. More workers than partitions sit idle. Raise partitions to scale, within broker and key-design limits.
- **Task queue (SQS, Rabbit, Kafka share groups):** many workers on one queue, no per-message order unless you add a group id (SQS FIFO `MessageGroupId` is the partition key: parallel across groups, serial inside one).

SQS standard has no key and only best-effort order. Pick FIFO when per-entity order matters and the throughput ceiling is acceptable; pick standard when throughput matters and the consumer is idempotent.

---

## Patterns That Come Up

### Outbox

Dual write fails: DB commit succeeds, enqueue crashes, or the reverse. **Transactional outbox:** in the same DB transaction as the business write, insert a row into `outbox`. A relay process publishes rows to the broker and marks them sent. Relay crashes are safe because publish is at-least-once and the consumer is idempotent.

Use it whenever the source of truth and the queue must agree (orders, payments, inventory). Skip it for telemetry.

### Inbox / idempotent consumer

Store `message_id` (or natural key) in a unique column **in the same transaction** as the side effect. A duplicate insert hits the constraint; the worker acks and moves on.

### Inbox is not exactly-once I/O

Sending email, charging a card, or calling a webhook can succeed and then the process can die before the inbox row commits. The outbound call itself needs an idempotency key, or the effect must be safe to repeat (set-to-state, not increment).

### Retry topic ladder

`main → retry.5s → retry.1m → dlq`. Keeps the main consumer fast. Delay lives on the retry topics. Permanent errors skip the ladder.

### Queue as a write buffer in front of a database

Acceptable for **ingest** (logs, clickstream, IoT) where the log is allowed to be ahead of the warehouse. Risky for **user commands** that must be durable and readable: the queue becomes an unofficial source of truth with weaker query and consistency than the DB. Prefer DB commit, then async projection.

### Payload size

SQS and SNS cap at **256 KB**. Kafka's default max record is **1 MB**. Over that, put the blob in object storage and enqueue a pointer. Large messages wreck batching and replication.

---

## What to Say in the Interview

Walk this checklist when you draw the queue. Thirty seconds, then move on.

1. **Why this box.** Async work, burst smoothing, decoupling, or fan-out. Name the signal (slow downstream, spike, second consumer).
2. **Shape.** Task queue or log. One consumer or many.
3. **What the user waits for.** Ack after DB + durable enqueue. Worker lag is a separate SLO.
4. **Delivery.** At-least-once, idempotent consumer, key named (`order_id`).
5. **Order.** Per which key, and what you do about a hot key.
6. **Failure.** Retries for transient errors, DLQ for poison, alert on depth. Partition progress vs gapless order.
7. **Durability.** Which ack (`acks=all`, publisher confirm, SQS `SendMessage`) and that DB and broker are tied together with an outbox if both must agree.
8. **Scale.** Consumers scale with lag. Partitions (or FIFO group ids) cap ordered parallelism. The queue does not raise downstream capacity; it bounds how late you are willing to be.

Red flags interviewers notice: queue on the read path, "exactly once" with no idempotency story, Kafka chosen for a single email worker, no DLQ, partition key = timestamp, and promising the user the work is finished when it is only queued.
