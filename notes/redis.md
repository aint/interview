# Redis

**Redis (Remote Dictionary Server)** is an in-memory data structure store that can be used as a database, cache, message broker, and streaming engine.

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Data Structures](#data-structures)
3. [Persistence](#persistence)
4. [Consistency](#consistency)
5. [Replication & High Availability](#replication--high-availability)
6. [Clustering](#clustering)
7. [Streams](#streams)
8. [Pub/Sub](#pubsub)
9. [Common Challenges](#common-challenges)
10. [Quick Reference](#quick-reference)

---

## Core Concepts

### Key Characteristics

- **In-memory**: Data stored in RAM for fast access (microsecond latency)
- **Single-threaded**: Redis uses a single thread for command execution (prevents race conditions, simplifies design)
- **Atomic operations**: All commands are atomic (no partial operations)
- **Persistence**: Optional persistence to disk (RDB snapshots, AOF logs)
- **Replication**: Master-replica architecture for high availability
- **Clustering**: Horizontal scaling via Redis Cluster

### Data Model

- **Key-value store**: Each key maps to a value
- **Keys**: Binary-safe strings (up to 512MB, but typically small)
- **Values**: Various data structures (strings, lists, sets, hashes, sorted sets, streams, etc.)
- **No schema**: Flexible data modeling

---

## Data Structures

- **Strings**:
    - Binary-safe strings (text, numbers, binary data).
    - *Use cases*: Caching, counters, session storage, simple key-value pairs.
- **Lists**:
    - Ordered collection (doubly-linked list, O(1) head/tail ops).
    - *Use cases*: Queues, stacks, timelines, message queues.
- **Sets**:
    - Unordered collection of unique strings (O(1) membership testing).
    - *Use cases*: Tags, unique items, set operations.
- **Sorted Sets (ZSET)**:
    - Ordered collection with scores for ordering.
    - *Use cases*: Leaderboards, rankings, time-series data, priority queues.
- **Hashes**:
    - Map of string fields to string values.
    - *Use cases*: User profiles, object representation, reducing memory overhead.
- **Bitmaps**:
    - Efficient bit arrays for boolean flags.
    - *Use cases*: User activity tracking, analytics, feature flags.
- **HyperLogLog**:
    - Probabilistic data structure for cardinality estimation with minimal memory.
    - *Use cases*: Unique visitor counting, approximate distinct count.
- **Geospatial**:
    - Geographic data structure for location storage and queries.
    - *Use cases*: Location-based services, nearby searches, geofencing.
- **Streams**:
    - Append-only log with consumer groups.
    - *Use cases*: Event sourcing, message queues, activity feeds (see [Streams](#streams) section).

---

## Persistence

Redis offers two persistence mechanisms to survive restarts:

### RDB (Redis Database Backup)

- **Snapshot-based**: Point-in-time snapshots (binary dump file) of the dataset.
- **How it works**:
  - Fork a child process
  - Child process writes snapshot to disk
  - Parent process continues serving requests
- **Pros**:
  - Compact file size
  - Fast recovery
  - Minimal performance impact (fork is fast on modern systems)
- **Cons**:
  - May lose data between snapshots
  - Fork can be expensive for large datasets (copy-on-write memory)

### AOF (Append-Only File)

- **Log-based**: Logs every write operation (creates a text file with Redis commands).
- **How it works**:
  - Writes each write command to AOF file
  - Can rewrite AOF to remove redundant commands (BGREWRITEAOF)
- **Sync modes**:
  - **always**: Sync on every write (safest, slowest)
  - **everysec**: Sync every second (default, good balance)
  - **no**: Let OS decide (fastest, may lose ~1 second of data)
- **Pros**:
  - Better durability (can lose at most 1 second with everysec)
  - Human-readable format
  - Can recover from partial writes
- **Cons**:
  - Larger file size
  - Slower recovery (must replay all commands)
  - Slight performance overhead

### Hybrid Approach

- **Best practice**: Use both RDB and AOF
- RDB for fast recovery and backups
- AOF for durability and point-in-time recovery

---

## Consistency

### Consistency Models

1. **Strong Consistency (Single Master)**:
   - All reads go to master
   - Guarantees latest data
   - Trade-off: Master becomes bottleneck

2. **Strong Consistency with WAIT**:
   - Use `WAIT` command to ensure writes are replicated to N replicas before returning
   - Reads from those acknowledged replicas will see the write (strong consistency)
   - Trade-off: Increased write latency, must wait for replica acknowledgment

3. **Eventual Consistency (Read Replicas)**:
   - Reads can go to replicas
   - Replicas may lag behind master
   - Trade-off: May read stale data

4. **Causal Consistency (Redis Cluster)**:
   - Within a slot, operations are ordered (if key A and key B are in the same slot, writes to A then B are seen in that order by all clients. If they're in different slots, clients might see B before A)
   - Cross-slot operations may not be ordered
   - Trade-off: Complexity in distributed scenarios

### Consistency Guarantees

- **Single-key operations**: Always atomic and consistent
- **Multi-key operations**:
  - Transactions (MULTI/EXEC) are atomic but not isolated
  - Lua scripts are atomic and isolated
- **Replication lag**: Replicas may be behind master (typically milliseconds)

### Handling Staleness

- **Read-after-write**: Read from master after writes
- **WAIT command**: Use `WAIT` to ensure writes are replicated before returning (enables strong consistency with replicas)
- **Version numbers**: Use version fields to detect stale reads
- **TTL-based invalidation**: Use expiration to limit staleness window

---

## Replication & High Availability

### Master-Replica Architecture

- **Master**: Handles all writes and reads
- **Replica**: Receives write stream from master, serves read requests
- **Async replication**: Replicas asynchronously replicate data from master.
- Optional **sync replication** with `WAIT`
- **Multiple replicas**: Can have multiple replicas for read scaling

### High Availability with Sentinel

Redis Sentinel is providing high availability (HA) for **non-clustered** Redis deployments. It acts as a dedicated monitoring and failover management system that automatically detects when a master instance is down and promotes a replica to take its place.

### Read Scaling

- **Read replicas**: Distribute read traffic across replicas
- **Consistency trade-off**: Replicas may have slightly stale data (eventual consistency)
- **Use case**: Read-heavy workloads where slight staleness is acceptable

---

## Clustering

### Redis Cluster

- **Horizontal scaling**: Distributes data across multiple nodes
- **Sharding**: Uses hash slots (16384 slots) to partition data
- **No proxy needed**: Clients can directly connect to any node
- **Automatic sharding**: Keys are distributed using CRC16(key) % 16384

### Cluster Architecture

- **Hash slots**: 16384 slots distributed across nodes
- **Node roles**: Each node can be master or replica
- **Gossip protocol**: Nodes communicate to share cluster state
- **Client routing**: Clients use MOVED/ASK redirects to find correct node

### Cluster Operations

- **Key distribution**: `CRC16(key) % 16384` determines slot
- **Hash tags**: `{user:1000}` ensures keys with same tag go to same slot
- **Resharding**: Can move slots between nodes (online operation)
- **Failover**: Automatic failover when master fails (replica promotion)

### Cluster Limitations

- **Multi-key operations**: Only work if all keys are in same slot
- **Transactions**: Only work within single slot
- **Lua scripts**: Must ensure all keys are in same slot (use hash tags)

---

## Streams

Redis Streams is a log-like data structure for message queues and event sourcing.

### Key Concepts

- **Stream**: Append-only log of messages
- **Message ID**: Auto-generated (timestamp-sequence) or manual
- **Consumer Groups**: Multiple consumers processing same stream
- **Pending entries**: Messages delivered but not acknowledged

### Consumer Groups

- **Load balancing**: Messages distributed among consumers in group
- **Message delivery**: Each message delivered to one consumer in group
- **Multiple groups**: Different groups can process same stream independently
- **Pending list**: Messages delivered but not ACKed are tracked

### Use Cases

- **Event sourcing**: Store all events as they occur
- **Message queues**: Reliable message delivery with consumer groups
- **Activity feeds**: Real-time activity streams
- **Log aggregation**: Centralized logging

---

## Pub/Sub

Redis Pub/Sub provides publish-subscribe messaging.

### Key Concepts

- **Channels**: Topics to which messages are published
- **Publishers**: Send messages to channels
- **Subscribers**: Receive messages from channels
- **Pattern subscriptions**: Subscribe to channels matching a pattern

### Characteristics

- **Fire-and-forget**: No message persistence (messages lost if no subscribers)
- **No delivery guarantees**: No acknowledgment mechanism
- **Fan-out**: One message delivered to all subscribers
- **Decoupling**: Publishers don't know about subscribers

### Use Cases

- **Real-time notifications**: Push notifications to clients
- **Event broadcasting**: Broadcast events to multiple services
- **Cache invalidation**: Notify all instances to invalidate cache
- **Live updates**: Real-time updates in web applications

### Limitations

- **No persistence**: Messages not stored
- **No delivery guarantees**: Messages lost if no active subscribers
- **Not suitable for**: Critical messages requiring guaranteed delivery (use Streams instead)

---

## Common Challenges

1. **Memory limits**: Redis is limited by available RAM
   - **Solution**: Use eviction policies, optimize data structures, consider clustering

2. **Persistence trade-offs**: Balancing durability vs performance
   - **Solution**: Use both RDB and AOF, tune sync frequency

3. **Replication lag**: Replicas may lag behind master
   - **Solution**: Monitor lag, read from master for critical reads

4. **Data loss risk**: In-memory data lost on crash (without persistence)
   - **Solution**: Enable persistence, use replication, consider Redis as cache not primary DB

5. **Network partitions**: Split-brain scenarios in distributed setups
   - **Solution**: Use Sentinel with proper quorum, understand consistency trade-offs

6. **Memory fragmentation**: Memory usage can grow beyond actual data size
   - **Solution**: Monitor memory usage, restart if needed, use memory allocator tuning

---

## Quick Reference

**Common Interview Questions**:
- **Why is Redis fast?** In-memory storage, single-threaded (no context switching), efficient data structures
- **What's the difference between RDB and AOF?** RDB is faster recovery, AOF is better durability
- **How do you handle Redis failures?** Use Sentinel for automatic failover, replication for redundancy
- **How do you ensure data consistency in Redis?** Single-key operations are atomic; use transactions/Lua for multi-key; understand replication lag
- **What are Redis eviction policies?** LRU, LFU, TTL-based, random eviction when memory limit reached
- **How does Redis handle concurrent access?** Single-threaded execution ensures atomicity, no race conditions
- **What is the Redis consistency model?** Strong consistency for single master, eventual consistency with read replicas
- **How do you scale Redis?** Vertical scaling (more RAM), read replicas, Redis Cluster (sharding)
