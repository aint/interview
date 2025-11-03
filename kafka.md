# Apache Kafka

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Architecture Components](#architecture-components)
3. [Metadata Management: Zookeeper & KRaft](#metadata-management-zookeeper--kraft)
4. [Message Delivery Semantics](#message-delivery-semantics)
5. [Replication & High Availability](#replication--high-availability)
6. [Consumer Groups & Partitioning](#consumer-groups--partitioning)
7. [Performance & Scalability](#performance--scalability)
8. [Common Challenges](#common-challenges)
9. [Key Configuration Settings](#key-configuration-settings)

---

## Core Concepts

**Kafka's mental model**: Kafka is not a queue; it's a **distributed log**. Messages are appended to an immutable log structure, which provides durability and ordering guarantees.

### Topic, Partition, and Offset

- **Topic**: A category or feed name to which messages are published (e.g., `user-events`, `order-updates`)
- **Partition**: Topics are split into partitions to enable parallelism and horizontal scaling
  - Each partition is an ordered, immutable log of records
  - Partitions allow Kafka to scale beyond what a single server can handle
  - Messages within a partition are guaranteed to be in order
  - Messages across partitions are not ordered (unless all messages go to a single partition)
- **Offset**: A unique, monotonically increasing identifier assigned to each message in a partition
  - Offsets are per-partition (partition 0 has its own offset sequence, partition 1 has its own, etc.)
  - Consumers track their position in each partition using offsets

### Message Routing

The producer decides which partition a message goes to:
- **With key**: Uses hash of the key modulo number of partitions → messages with the same key always go to the same partition (enables ordering per key)
- **Without key**: Round-robin distribution across partitions (better load distribution, but no ordering guarantee)

---

## Architecture Components

### Broker

A broker is a Kafka server that stores and serves messages. A Kafka cluster consists of one or more brokers.

**Responsibilities**:
- Accept messages from producers
- Assign offsets to messages
- Store messages on disk
- Serve messages to consumers
- Handle replication of partitions
- Coordinate with other brokers

### Producer

Application that writes messages to topics.

**Key concepts**:
- **Batching**: Producers batch messages for efficiency
- **Compression**: Can compress batches (snappy, gzip, lz4, zstd)
- **Acknowledgment modes** (see [Configuration](#key-configuration-settings))

### Consumer

Application that reads messages from topics.

**Key concepts**:
- Consumers read messages in order within a partition
- Consumers can seek to specific offsets (replay from a point in time)
- **Offset storage**: By default, offsets are stored in an internal Kafka topic `__consumer_offsets`
- **Auto-commit vs manual commit**:
  - Auto-commit: Consumer periodically commits offsets (default: every 5 seconds)
  - Manual commit: Consumer explicitly commits offsets after processing

### Consumer Group

A consumer group is a set of consumers that work together to process a topic.

**Key rules**:
- Each partition is consumed by **exactly one** consumer in the group
- If you have **more consumers than partitions**, some consumers will stay idle (no work assigned)
- If you have **fewer consumers than partitions**, some consumers will handle multiple partitions
- Each consumer group gets its own copy of the messages (allows different applications to process the same topic independently)

**Rebalancing**: When consumers join or leave a group, Kafka redistributes partitions among the remaining consumers.

### Connectors

External systems that push data into Kafka (sources) or pull data out of Kafka (sinks) in a scalable and reliable way.

**Types**:
- **Source connectors**: Import data from external systems (e.g., databases, file systems)
- **Sink connectors**: Export data to external systems (e.g., databases, search engines, file systems)

Examples: JDBC connector, Elasticsearch connector, S3 connector.

### Kafka Streams

A client library for building stream processing applications. Allows you to:
- Process streams of data in real-time
- Perform transformations, aggregations, joins
- Build stateful stream processing applications

---

## Metadata Management: Zookeeper & KRaft

Kafka requires coordination and metadata management. Historically, Kafka used **Apache Zookeeper**, but newer versions (2.8.0+) introduced **KRaft (Kafka Raft)** mode, which eliminates the Zookeeper dependency.

### Zookeeper-Based Architecture

**Zookeeper's Role**: Stores cluster metadata (brokers, topics, partitions), manages controller election, tracks broker health.

**Leader Election with Zookeeper**:
- **Controller Election**: Brokers compete to create ephemeral node `/controller` in Zookeeper; only one succeeds
- **Partition Leader Election**: Controller detects leader failure via Zookeeper, selects new leader from ISR, updates metadata in Zookeeper

**Limitations**: Single controller bottleneck, scalability limits (~200K partitions), requires separate Zookeeper cluster, slower metadata operations.

### KRaft Mode (Zookeeper-Free)

**Architecture**: Uses **Raft consensus algorithm** with multiple quorum controllers (typically 3-5). Metadata stored in internal Kafka topic `__cluster_metadata`. No external dependency.

**Leader Election in KRaft**:
- **Controller Election**: Raft algorithm elects leader controller automatically within quorum
- **Partition Leader Election**: Controller selects new leader from ISR, but metadata updates use Raft consensus instead of Zookeeper

**Benefits**: Scales to millions of partitions (vs ~200K with Zookeeper), faster metadata operations, better handling of network partitions, no external dependency.

---

## Message Delivery Semantics

### At Most Once (Best Effort)

Messages will be delivered **at most once**, meaning:
- Messages may be lost
- Messages will never be duplicated

**How to achieve**:
1. Producer: Set `acks=0` (fire-and-forget) or disable retries
2. Consumer: Commit offset **before** processing the message
   - If consumer crashes after committing but before processing → message is lost (won't be reprocessed)

**Use case**: When progress is more important than completeness (e.g., metrics, logs where occasional loss is acceptable)

### At Least Once

Messages will be delivered **at least once**, meaning:
- All messages will be processed
- Messages may be processed **multiple times** (duplicates possible)

**How to achieve**:
1. Producer: Enable retries with `acks=all` (or `acks=1`)
2. Consumer: Process message **before** committing offset
   - If consumer crashes after processing but before committing → message will be reprocessed (duplicate)

**Use case**: When duplicates are acceptable (e.g., idempotent operations, analytics)

### Exactly Once (Effectively Once)

Messages will be processed **exactly once** (no duplicates, no loss).

**How to achieve**:
1. **Idempotent Producer** (enabled by default since Kafka 0.11.0):
   - Producer sends a Producer ID (PID) and sequence number with each batch
   - Broker deduplicates based on PID + sequence number
   - Prevents duplicate writes from producer retries

2. **Transactions**:
   - Atomic writes across multiple partitions
   - All records in a transaction are committed or none are
   - Enables exactly-once semantics for producers writing to multiple partitions

3. **Transactional Consumer**:
   - Consumer reads messages, processes them, and writes results atomically
   - Uses transactional producer to write results back to Kafka

**Note**: Achieving exactly-once semantics requires careful coordination between producer and consumer code.

### No Guarantee (Default with Auto-Commit)

When using auto-commit (default), you get **no guarantees**:
- Consumer commits offsets periodically (before processing completes)
- If consumer crashes:
  - After offset commit but before processing → messages are lost (never processed)
  - After processing but before offset commit → messages are reprocessed (duplicates)

**This is why** manual offset management is often preferred for production systems.

---

## Replication & High Availability

### Replication Factor

- Each partition can be replicated across multiple brokers (typically 3x replication)
- **Replication factor = 3** means the partition exists on 3 different brokers

### Leader and Followers

- **Leader replica**: Handles all read and write requests for the partition
- **Follower replicas**: Replicate data from the leader (read-only)
- If leader fails, one of the followers becomes the new leader (automatic failover)

### ISR (In-Sync Replicas)

- Set of replicas that are "in sync" with the leader (caught up within a configurable lag threshold)
- Only ISR replicas can become leaders
- If a replica falls too far behind, it's removed from ISR

### Acknowledgment Modes

**Producer `acks` configuration**:
- **`acks=0`**: Fire-and-forget, no acknowledgment (highest throughput, no durability guarantee)
- **`acks=1`**: Leader acknowledges (medium throughput, leader failure can cause data loss)
- **`acks=all`** (or `-1`): All ISR replicas acknowledge (lowest throughput, highest durability)

---

## Consumer Groups & Partitioning

### Partition Assignment

- Kafka automatically assigns partitions to consumers in a group
- Rebalancing occurs when:
  - Consumers join or leave the group
  - Partitions are added to a topic
  - Session timeout occurs

### Consumer Lag

- **Lag**: The difference between the latest offset in a partition and the consumer's current offset
- High lag indicates the consumer is falling behind
- Lag metrics are critical for monitoring consumer health

### Offset Management

- **Auto-commit**: Consumer commits offsets periodically (default: every 5 seconds)
- **Manual commit**:
  - Synchronous: `consumer.commitSync()` - blocks until commit succeeds
  - Asynchronous: `consumer.commitAsync()` - doesn't block, but may fail silently
- **Offset storage**:
  - Default: `__consumer_offsets` topic in Kafka
  - Can be stored externally (database, etc.) for more control

---

## Performance & Scalability

### Message Retention

- **Time-based**: Messages retained for a configured duration (default: 7 days)
- **Size-based**: Messages retained until log reaches a configured size
- **Log compaction**: Keeps only the latest value for each key (useful for maintaining current state)

### Batching & Compression

- **Batching**: Producers batch messages to reduce network overhead
- **Compression**: Reduces network bandwidth and storage (common: snappy, lz4, zstd)
- Trade-off: Lower latency vs higher throughput

### Scaling

**Scaling consumers**: Add more consumers to a consumer group (limited by number of partitions)

**Scaling producers**: Can add more producer instances (messages distributed across partitions)

**Scaling brokers**:
- Can add brokers, but requires partition reassignment (complex operation)
- Partitions cannot be reduced, only increased

**Limitations**:
- Number of partitions per topic affects parallelism
- Too many partitions can impact performance (each partition has overhead)
- Recommended: Start with partitions = expected peak consumer parallelism, scale up as needed

---

## Common Challenges

1. **Scaling complexity**: Kafka clusters are difficult to scale out due to stateful nature of brokers. Adding/removing brokers requires partition reassignment, which is complex and can impact performance.

2. **Cloud costs**: Hosting Kafka in the cloud can be expensive due to:
   - High EBS storage costs (Kafka stores all messages on disk)
   - Cross-AZ traffic costs (replication traffic)
   - Need for over-provisioning due to limited scalability

3. **Consumer lag**: Consumers falling behind can indicate:
   - Insufficient consumer parallelism
   - Slow processing logic
   - Downstream system bottlenecks

4. **Ordering guarantees**:
   - Messages are ordered only within a partition
   - Cross-partition ordering requires careful design (e.g., single partition per key)

5. **Rebalancing**:
   - Can cause temporary unavailability
   - "Stop-the-world" rebalancing pauses all consumers
   - Frequent rebalancing can indicate configuration issues (session timeouts too low, etc.)

---

## Key Configuration Settings

### Producer Settings

- **`acks`**: 0, 1, or all (-1) - acknowledgment requirement
- **`retries`**: Number of retry attempts (default: 2147483647)
- **`idempotence`**: Enable idempotent producer (default: true in recent versions)
- **`max.in.flight.requests.per.connection`**: Must be 1 for exactly-once with idempotence disabled, can be > 1 with idempotence enabled
- **`compression.type`**: none, gzip, snappy, lz4, zstd
- **`batch.size`**: Batch size in bytes
- **`linger.ms`**: Wait time to fill a batch

### Consumer Settings

- **`group.id`**: Consumer group identifier
- **`auto.offset.reset`**: earliest, latest, none - what to do when no offset exists
- **`enable.auto.commit`**: true/false - automatic offset committing
- **`max.partition.fetch.bytes`**: Maximum amount of data per fetch
- **`session.timeout.ms`**: Time before consumer is considered dead
- **`heartbeat.interval.ms`**: Frequency of heartbeats to coordinator

### Broker/Topic Settings

- **`replication.factor`**: Number of replicas (default: 1, recommended: 3)
- **`min.insync.replicas`**: Minimum ISR replicas required (default: 1, recommended: 2 for replication factor 3)
- **`retention.ms`**: How long to retain messages (default: 7 days)
- **`segment.bytes`**: Size of log segments
- **`compression.type`**: Topic-level compression

---

## Quick Reference

**Common Interview Questions**:
- How does Kafka ensure ordering? (Within partition only)
- What happens if a consumer crashes? (Rebalancing, messages may be reprocessed or lost depending on offset commit timing)
- How do you achieve exactly-once processing? (Idempotent producer + transactions + transactional consumer)
- What's the difference between a topic and a partition? (Topic is logical, partition is physical)
- When would you use a key vs no key? (Key for ordering per key, no key for load distribution)
- How does consumer lag work? (Difference between latest offset and consumer offset)
- What is ISR? (In-Sync Replicas - replicas caught up with leader)
- What is Zookeeper's role in Kafka? (Metadata storage, controller election, broker registration)
- How does leader election work in Kafka? (Controller selects new leader from ISR when leader fails)
- What is KRaft mode? (Zookeeper-free mode using Raft consensus for metadata management)
- What are the benefits of KRaft over Zookeeper? (Better scalability, simplified operations, no external dependency)
- How does controller election work? (Zookeeper: ephemeral node creation; KRaft: Raft consensus)
