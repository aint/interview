# Caching

System-design prep: when a cache belongs on the diagram, which pattern you are actually using, and the failure modes interviewers probe. Redis as a product (structures, persistence, cluster, eviction command names) is in [redis.md](./redis.md). Hit-ratio math and CDN origin sizing are in [system_capacity.md](./system_capacity.md). Where the box sits on the request path is in [system_design.md](./system_design.md).

## Table of Contents

1. [Mental Model](#mental-model)
2. [Where the Cache Sits](#where-the-cache-sits)
3. [When to Cache](#when-to-cache)
4. [When a Cache Hurts](#when-a-cache-hurts)
5. [Read and Write Patterns](#read-and-write-patterns)
6. [Invalidation](#invalidation)
7. [Eviction](#eviction)
8. [Consistency](#consistency)
9. [Failure Modes](#failure-modes)
10. [Keys, TTL, and the Value](#keys-ttl-and-the-value)
11. [Hit Ratio and Capacity](#hit-ratio-and-capacity)
12. [HTTP and CDN](#http-and-cdn)
13. [Local and Distributed](#local-and-distributed)
14. [What to Say in the Interview](#what-to-say-in-the-interview)

---

## Mental Model

A cache is a **fast copy of something expensive to fetch or compute**. The database, object store, or origin service remains the source of truth. The cache is derived. It can be missing, stale, or wrong, and the design still has to be correct.

Three questions decide the design:

1. **Who is allowed to be stale, and for how long?** The writer usually needs read-your-writes. Everyone else can often wait seconds.
2. **What happens on a miss?** A miss is the normal path, including right after eviction, deploy, or failover. The origin must survive the miss rate you are promising.
3. **What happens when the cache is down?** The app fails open to the origin with a concurrency cap, or it fails closed and sheds load. Pick one out loud.

Caching works because access is skewed. A small set of keys (popular posts, config, celebrity timelines) takes most of the reads. A uniform key space (every request a unique search) has nothing to reuse.

There is no hardware-style coherence protocol across app servers. Freshness comes from **TTL**, **delete-on-write**, or **rebuilding the key**. Those are application decisions.

---

## Where the Cache Sits

Put the cache at the narrowest layer that already has the reuse. A closer cache is faster and smaller. A farther cache is shared by every instance.

| Layer | What it stores | Latency | Shared? | Typical TTL |
| ----- | -------------- | ------- | ------- | ----------- |
| **Browser** | HTTP responses the user already fetched | 0 (local) | One user | `max-age`, often minutes to a year for assets |
| **CDN** | Public bytes: JS, CSS, images, video, some anonymous GETs | Edge RTT, often 10–50 ms | All users in that POP | Long for versioned assets; short or bypass for APIs |
| **Reverse proxy** (nginx, Varnish, API gateway) | Full HTTP responses | Sub-ms to a few ms inside the DC | All app instances behind it | Seconds to minutes |
| **In-process** (Caffeine, Guava) | Hot objects inside one JVM/process | RAM, ~100 ns | One instance | Milliseconds to a few seconds |
| **Distributed** (Redis, Memcached) | Shared objects, counters, sessions | Same-DC RTT ~0.5–1 ms | The fleet | Seconds to hours |
| **Database buffer pool** | Pages the engine keeps in RAM | RAM on a hit; disk on a miss | That database node | Managed by the engine |

Latency numbers match [system_capacity.md](./system_capacity.md). A Redis GET is a network round trip, so it is much slower than an in-process hit and much faster than a cold disk read or a cross-region call.

Two layers that people lump in with caching, and that behave differently:

- **Request memoization** (DataLoader, per-request maps) dedupes work inside a single request. It cannot go stale across requests. Use it for N+1 fan-out. It is gone when the request ends.
- **A derived store** (search index, feed timeline, materialized view) is rebuilt asynchronously and queried on its own. Treat it as its own data model with its own freshness SLO, using the same invalidation thinking.

Default placement: **CDN for immutable public bytes, Redis for hot computed or entity reads, a small in-process cache in front of a single hot key.** The database buffer pool is already there; it is a reason a primary-key lookup is cheap, and it is a reason a second cache must earn its keep.

---

## When to Cache

Cache a read when all of these are true:

- The same key is read many times between writes.
- Building the value is expensive (query, join, fan-out, third-party call).
- Callers tolerate a stated staleness window.
- The hot set fits in the memory you are willing to pay for.

Good fits: user profiles, product catalog, feature flags, rendered timelines, session lookups, rate-limit counters (those are often Redis-as-state — see [redis.md](./redis.md) for durability), config, social graphs that change slowly.

Name the staleness in the same sentence as the cache. "Profile cache, 60s TTL, delete the key on update, so a stranger can be a minute behind and the editor sees the write because we return it from the POST."

---

## When a Cache Hurts

Skip the cache, or keep it off the correctness path, when:

- **The access pattern has no reuse.** Unique searches, one-shot reports, and "every id exactly once" fill memory with entries that never hit again. That crowds out keys that would have hit.
- **The working set is larger than RAM.** Eviction thrashes, hit ratio collapses, and you added a network hop to every read.
- **The value must be exact at read time.** Wallet balance, inventory that cannot oversell, authorization that just changed. Those reads go to the source of truth. A cache can sit beside them for display, with a reconciliation path.
- **The origin is already fast enough.** An indexed primary-key read served from the buffer pool can be ~1 ms. A 50% hit rate in Redis can make the average slower and add a failure domain. Compute it (see [Hit Ratio and Capacity](#hit-ratio-and-capacity)).
- **Writes dwarf reads of the same key.** You pay invalidation on every write and almost never hit. Write-around or no cache.
- **The response is private and you were about to share it.** A CDN or reverse-proxy entry keyed without the user id serves one user's payload to another.

A cache also hides origin pain until the cache dies. Size the database for the miss path you will actually have during a cold start, plus a cap so a total cache outage sheds load before it takes the primary down. See [system_high_availability.md](./system_high_availability.md).

---

## Read and Write Patterns

The names differ by **who talks to the database** and **whether the write ack waits for the database**.

| Pattern | Read path | Write path | Ack waits for DB? | Staleness |
| ------- | --------- | ---------- | ----------------- | --------- |
| **Cache-aside** | App checks cache; on miss the app loads the DB and fills the cache | App writes the DB, then deletes the key | Yes | Until the next read fills; race window plus TTL |
| **Read-through** | App checks cache; on miss the **cache** loads the DB and stores it | (reads only) | — | Same as cache-aside |
| **Write-through** | Served from cache | Cache writes through to the DB synchronously | Yes | Fresh for writers that go through the cache |
| **Write-around** | Filled on a later read miss | DB only; cache skipped | Yes | Old entry stays until TTL or an explicit delete |
| **Write-back** | Served from cache | Cache acks, flushes to the DB later | No | Fresh in cache; DB lags; loss if cache dies first |
| **Refresh-ahead** | Hot keys reloaded before expiry | Independent of the write | — | Hides miss latency on keys you can predict |

### Cache-aside

The application owns the protocol.

```
read:   cache?  --hit--> return
              \--miss--> DB --> SET cache --> return

write:  DB commit --> DEL cache key --> return
```

This is the default in interviews and in most services. Any Redis or Memcached client can do it. The cache stores only keys that have been read.

Fill the cache **after the database transaction commits**. A fill from inside an open transaction can publish a row that later rolls back.

### Read-through

Same shape as cache-aside, with the loader behind the cache API (`LoadingCache`, DAX-style read-through). Application code calls `cache.get(id)` and the library does the miss. Use it when you want one code path and singleflight inside the library. The consistency rules are the same as cache-aside.

### Write-through

```
write:  cache SET --> DB write --> ack to client
```

The cache and the database move together, at the cost of write latency equal to both. Every written key occupies memory, including keys nobody reads again. A writer that bypasses the cache (a migration, another service, a manual SQL fix) still leaves stale entries. Keep a TTL.

Fits inventory displays and finance-style reads where the same service is the only writer and the value is read immediately.

### Write-around

```
write:  DB only
read:   miss --> DB --> fill cache
```

Fits append-only or write-heavy data that is rarely read back soon (logs, analytics events). If an older value is already cached, write-around leaves it in place until TTL or eviction. For mutable rows, pair the DB write with `DEL`, which is cache-aside again.

### Write-back

```
write:  cache SET --> ack to client
              \--> async flush to DB
```

Write latency is the cache RTT. Many updates to one key collapse into one flush. The cache holds the newest value, and a crash before flush loses it.

Use it when loss is acceptable or covered: counters, presence, dirty analytics, a buffer with its own WAL or replica ack. For orders and payments the database commits on the sync path; the cache is filled after. See the outbox discussion in [message_queues.md](./message_queues.md) for the durable version of "ack now, persist the side effect later."

### Refresh-ahead

Reload a key when it is still popular and close to expiry, on a timer or on the read that crosses a soft TTL. The user who would have eaten the miss is served the previous value. You spend origin QPS even if the next read never comes, so limit this to the top keys.

### The combination to default to

**Cache-aside reads, delete-on-write, jittered TTL as a backstop, singleflight on the miss.**

Delete the key after a successful commit. The next read refills from the database. An in-place `SET` of the new value races: a miss that started earlier can write the old row back on top of the new one.

```
T1  miss, read v1 from DB
T2  commit v2, SET cache = v2
T1  SET cache = v1        # stale until TTL
```

`DEL` after commit shrinks that window. A remaining race still exists: T1 read v1, T2 committed and deleted, T1 then `SET` v1. It needs a slow read overlapping a write. Bound it with a short TTL, or store a version and `SET` only if the cached version is older (compare-and-swap / Lua). Say the race out loud if the data is money or inventory; send that read to the primary.

Update-in-place is reasonable when the write path is the only filler and it writes the full new value after commit. Delete-on-write is the safer default because a missed delete heals on the next TTL, and a missed update looks like a hit forever.

---

## Invalidation

Invalidation is the product decision. Pick a mechanism per key family and say the staleness it allows.

| Mechanism | How it heals | What it costs | Use when |
| --------- | ------------ | ------------- | -------- |
| **TTL only** | Entry expires, next read refills | Stale for up to TTL; stampede at expiry | Derived data, no reliable write hook (third-party, cross-service) |
| **Delete on write** | Next read sees the database | Every writer must know the key; the race above | You own the write path |
| **Write-through** | Cache updated with the write | Write latency; bypassed writers still stale | Single writer, read-after-write from cache |
| **Versioned prefix** | `user:42:v7` → bump to `v8`; old keys age out on TTL | Memory holds garbage until TTL; readers must see the new version | Bulk invalidate (schema change, "drop this user's cache", tenant flush) |
| **Event / CDC** | Writer or the WAL publishes "key changed"; holders delete | Bus lag; a lost event needs the TTL safety net | Many services, or in-process caches that must hear about writes |
| **Lease / version CAS** | Fill succeeds only if the DB version still matches | Extra conditional write | The stale-fill race is unacceptable |

TTL is still required with delete-on-write. Writers crash before `DEL`, a bug skips a key, a pub/sub message is dropped (Redis pub/sub is fire-and-forget — [redis.md](./redis.md)). TTL is how those leaks end.

**Soft TTL and hard TTL.** Store `value`, `soft_expiry`, and `hard_expiry`. Before the soft deadline, return the value. Between soft and hard, return it and let one worker refresh. After the hard deadline, treat it as a miss. This is stale-while-revalidate inside the application, and it is the practical fix for expiry stampedes.

**Versioned prefix** invalidates a set without scanning Redis: readers resolve `user:42` → current generation, then fetch `user:42:v9:profile`. Bump the generation on a schema change or a "purge tenant" operation. Old generations expire. The generation key itself is a hot key; cache it in-process for a second.

**Who computes the key.** If service B writes the row and service A caches it, the delete has to cross that boundary (shared library, domain event, or CDC). A TTL-only cache is the honest design until that hook exists.

Phil Karlton's line belongs in the interview: invalidation, and naming the key, are the hard parts. Building the GET is not.

---

## Eviction

Expiry and eviction solve different problems.

- **Expiry** drops a key at its TTL to bound staleness, even when memory is free.
- **Eviction** drops keys because **memory is full**. A key can disappear before its TTL. Callers already have to handle a miss.

| Policy | Keeps | Weakness |
| ------ | ----- | -------- |
| **LRU** | Recently used | A full scan or a batch job looks "recent" and evicts the real hot set |
| **LFU** | Frequently used | A new viral key looks cold until it accumulates hits; stale popular keys stick |
| **FIFO** | Oldest inserted | Ignores how often a key is read |
| **Random** | Whoever it picks | Cheap; hit rate is fine when the hot set is a large fraction of memory |
| **TinyLFU** (Caffeine) | Frequency, with a small admission window | More moving parts; the usual answer inside one process |
| **TTL / volatile-*** | Only keys that have an expiry | A key written without a TTL is pinned until you notice memory pressure |

For a **dedicated cache**, evict from all keys (Redis `allkeys-lru` or `allkeys-lfu`) and still set a TTL on every entry. `allkeys-lfu` fits stable skew (catalog, celebrities). `allkeys-lru` fits recency (news, feeds).

For a **mixed Redis** that also holds locks, queues, or sessions, evict only keys that carry a TTL (`volatile-lru` / `volatile-lfu`). A missing TTL then means "this key is not cache." Forgetting the TTL on a flood of cache keys fills memory until writes fail. A dedicated cache node avoids that class of mistake.

**Admission** matters as much as eviction. A one-hit key that replaces a hot key lowers the hit ratio. TinyLFU and "only cache if this key was seen twice" exist to reject those. In an interview: "I would not cache the long tail of unique keys; I cache ids that repeat."

Production local caches (Caffeine) combine an admission window with frequency. Redis approximates LRU/LFU with sampling, which is good enough and stays O(1). Policy names and persistence tradeoffs: [redis.md](./redis.md).

---

## Consistency

Decide the consistency you are selling, per endpoint.

| Promise | What to do |
| ------- | ---------- |
| **Stale within TTL** | TTL only. Easiest. Say the number. |
| **Stale until the next write is observed** | Delete-on-write. Strangers see the new value on their next miss. In-flight reads can still refill the old one for up to the TTL. |
| **Read-your-writes** | The writer does not depend on the shared cache. Return the new resource from the POST, `DEL` before responding, or send that user to the primary for a few seconds. |
| **Read-your-writes across devices** | A second device has no in-memory copy of the POST body. Delete-on-write plus a read from primary, or a version the client sends (`If-None-Match` / a `rev` query). |
| **Monotonic reads** | A lagging replica refill can move a user backward. Fill the cache from the primary, or from a replica that is caught up to the write's LSN. |
| **Always current** | Skip the cache. Read the source of truth. |

**Replica fill.** Cache-aside that reads a lagging replica caches the lag for the whole TTL, which is longer than the replica lag you thought you had. Fill from the primary, or accept that the cache extends replica lag up to TTL.

**"Cache stampede of correctness."** Two writers, last `SET` wins, and the cache can disagree with the database even when both writes committed in order. Version CAS or delete-on-write plus TTL closes this. Locks around the cache are rarely worth it.

**Stale on purpose.** `stale-while-revalidate` and `stale-if-error` prefer a known-old successful response over a slow or failed origin. Product pages and feeds can. Balances and permissions cannot.

**Multi-region.** The cache is regional. An invalidate in the write region reaches other regions after replication lag or after a bus message. Until then those regions serve the old TTL. Call that out when the prompt is global; keep writes single-homed. See [system_high_availability.md](./system_high_availability.md).

---

## Failure Modes

Interviewers often use three names for three different overloads. Keep them separate.

| Name | What is happening | Origin sees | First fix |
| ---- | ----------------- | ----------- | --------- |
| **Stampede** (thundering herd, dogpile, "breakdown") | One **hot** key expires or is deleted. Thousands of requests miss together and all recompute it | A spike of identical queries | Singleflight, soft TTL, stale-while-revalidate |
| **Penetration** | The key is not in the cache **and not in the database** (bad id, scraped id space, attack). Nothing is ever stored, so every request misses | One query per attack request | Validate ids, negative cache, bloom filter, rate limit |
| **Avalanche** | A **large set** of keys expires together, or the cache cluster is down, so miss traffic becomes full traffic | A step-change toward 100% of read QPS | Jittered TTLs, warm-up, failover, shed load |

### Stampede

A popular key hits TTL. Every app server observes the miss in the same millisecond. Each one runs the expensive query. The database sees a cliff, and the clients that were supposed to be protected wait on the origin.

Mitigations, in the order you should reach for them:

1. **Singleflight (request coalescing).** Inside one process, one loader runs per key; waiters share the result. Stops self-inflicted fan-out on one box. It does nothing across the fleet by itself.
2. **Soft TTL / stale-while-revalidate.** Keep serving the old bytes. One worker refreshes. Users never block on the rebuild.
3. **Distributed lock.** `SET lock:key NX EX <seconds>`. The winner loads and fills. Losers serve stale, wait briefly, or return a retry. Set the lock TTL above the p99 rebuild time, and delete the lock with a compare-and-delete so you do not drop someone else's lock. A lock that expires mid-rebuild brings the stampede back.
4. **Probabilistic early expiration.** As expiry approaches, a read may refresh early with probability that rises when the rebuild is slow. One of the concurrent readers tends to win before the cliff. The usual form (x-fetch) treats "time to recompute" as part of the decision. Soft TTL is the same idea and is easier to explain on a whiteboard.
5. **Do not expire the hottest keys on a hard edge.** Refresh them in the background and keep a long hard TTL only as a crash backstop.

Per-request singleflight plus a soft TTL covers most designs. Mention the lock when the rebuild is so heavy that two concurrent origin calls are already too many (a report, a feed fan-out).

### Penetration

Random or malicious ids miss the cache, miss the database, and never create an entry. The cache provides no protection, which is the point of the attack.

- Reject ids that cannot exist (bad format, over the allocated range) before the cache.
- **Negative cache:** store a short-lived "absent" marker (10–60s). Real creates of that id must delete the marker or wait out the TTL. An attacker who invents a new key every time fills memory with markers; cap the marker's TTL and its share of memory, and rate-limit at the edge.
- A **bloom filter** of existing ids answers "definitely absent" without a database round trip. False positives cost a database lookup. False negatives would hide real rows, so the filter is built to avoid them. New ids have to be added to the filter; deletions need a counting bloom or a periodic rebuild. The filter is a process-local or a rebuilt snapshot, and it lags creates by that rebuild interval.
- Rate-limit and WAF on endpoints that take unconstrained ids.

### Avalanche

Two different triggers share the name:

- **Synchronized expiry.** A deploy writes a million keys with `TTL = 3600`. An hour later they expire together. **Jitter:** `ttl = base + random(0, base/10)`, and stagger the warm-up.
- **Cache process down.** Redis failover, a flushed instance, a bad deploy that flushed the prefix. Traffic that was at a 95% hit ratio becomes 100% origin. A 20× jump. The database was never sized for that.

For the outage path: short client timeouts (tens of milliseconds, fail fast), a **concurrency limiter** on the origin fallback, and a circuit breaker that sheds or serves stale once the database latency climbs. In-process caches absorb a slice of the hot set during a Redis blip if their TTL has not also ended. Replica or cluster failover belongs in [redis.md](./redis.md) and [system_high_availability.md](./system_high_availability.md). A cache outage is an availability event for the database, so say what the database sees.

### Hot key

Sharding spreads keys. It does not spread one key. A celebrity post, a global counter, or a feature flag sits on one Redis shard and one thread. That shard saturates while the cluster looks idle. A single shard handles on the order of 100k–200k simple ops/s; past that the key needs a different shape.

- **In-process cache** of that one key, TTL of a second or less. A thousand app servers then ask Redis a thousand times a second, which is the point.
- **Replicate the value** across `key#{0..N}` and read a random copy. Writers update every copy. N is small (10–100), sized off the op ceiling.
- **Isolate** the key on its own node if it is also large.
- If the value is an HTTP response and it is public, **cache it at the CDN** and take it off Redis entirely.

A **big value** (multi-megabyte blob in one key) is a related failure: it stalls the Redis thread, blows the network budget, and evicts many small keys. Cache an id or a summary; put the blob in object storage and the CDN.

### Cold start

A new cluster, a flush, or a region failover starts at a 0% hit ratio. Pre-warm the top keys from a list (yesterday's hot set), jitter the fill, and ramp traffic. Blocking the deploy until the cache is hot couples release time to the database. Prefer a short elevated-miss window with a limiter.

### Wrong value cached

A bug writes a bad payload with a one-day TTL. Every hit serves it. Defenses: TTL short enough that you can wait, a **schema version inside the key** so a new deploy misses old entries, and a purge by prefix or version bump. Know which purge you have before you need it. CDN purges are slower and less complete than a Redis `DEL`; versioned URLs avoid the purge.

### Cache poisoning

Two variants:

- **App cache:** an attacker controls part of the key or the value (unbounded query string, reflected content) and stores it under a key other users will hit. Canonicalize the key, ignore unknown query parameters, and do not let user input choose a shared key.
- **HTTP cache:** a response varies on a header that is **not** part of the cache key (unkeyed header). The attacker sends a poisoned header, the shared cache stores the poisoned response for the URL, later users receive it. Lock down which headers enter the cache key (`Vary`), and do not let CDNs cache responses that reflected request headers.

---

## Keys, TTL, and the Value

### Key shape

```
{service}:{entity}:{id}:{variant}
profile:user:42
feed:user:42:v3
catalog:sku:99:locale:de
```

- Include a **schema version** when the serialized shape can change across a deploy.
- The key is a contract. Writers and readers in every service have to build the same string. A shared helper beats tribal knowledge.
- Bound cardinality. `search:{raw query}` is an unbounded set of one-hit keys. Cache the popular queries or skip it.
- Canonicalize: sort query parameters, lowercase hosts, drop tracking parameters, or the same page is N entries.
- Per-user data includes the user id in the key. A shared key for a personalized body is a data leak.

### TTL

Pick TTL from **how wrong a stale read is**, then jitter it.

| Data | Starting point |
| ---- | -------------- |
| Feature flag, kill switch | 1–5 s, or pub/sub plus a 30 s backstop |
| Feed, counts, "who to follow" | 5–60 s |
| Profile, catalog row | 1–15 min, with delete-on-write |
| Session | Lifetime of the session; this is state, size it as such |
| Versioned static asset | Months, because the URL changes when the bytes change |
| Negative marker | 10–60 s |

A longer TTL raises the hit ratio and the staleness. A shorter TTL is a freshness and a stampede decision. If you cannot name an acceptable staleness, you do not have a TTL yet; you have a hope.

### What you store

- **Entity cache.** One key per row or aggregate (`user:42`). Invalidation is one delete. A page fans out to several GETs; pipeline them. This is the default.
- **Assembled view.** One key per page (`feed:user:42`). One GET, and any underlying write has to know every view key that embedded it. Easy to get wrong. Fits a timeline you rebuild as a unit.
- **Computed result.** Expensive ranking, a permission snapshot, a rendered fragment. The TTL is your correctness bound.

Store compact bytes (a small binary or JSON you measured). A cache of full HTML for every user is a memory and an invalidation problem; cache the ids and assemble.

Do not cache:

- The only copy of a write (unless you deliberately chose write-back and its loss story).
- Authorization decisions for longer than you can stand a revoked user still passing.
- Raw SQL text. Any write to the underlying table used to make MySQL's query cache throw the entries away, which is why that feature is gone. Cache by entity id.

---

## Hit Ratio and Capacity

```
origin_qps = read_qps × (1 − hit_ratio)
avg_latency ≈ hit_ratio × cache_rtt + (1 − hit_ratio) × (cache_rtt + origin_latency)
```

Work a number in the interview. 10k read QPS, 90% hit → 1k origin QPS. 99% hit → 100 origin QPS. The last nine points of hit ratio are a 10× cut in database load, and they are also the points you only get if the hot set actually fits.

| Hit ratio | Fraction that reaches origin |
| --------- | ---------------------------- |
| 0% | all of it (cold, or nothing is reused) |
| 80% | 1/5 |
| 90% | 1/10 |
| 99% | 1/100 |

Example that says "do not add Redis yet": origin 2 ms, cache RTT 1 ms, hit ratio 50%.

```
avg = 0.5 × 1 ms + 0.5 × (1 + 2) ms = 2 ms
```

Same latency as the database alone, plus a new outage mode. Example that says "add it": origin 15 ms, cache 1 ms, hit ratio 90% → `0.9 × 1 + 0.1 × 16 = 2.5 ms`.

**Size memory to the working set you intend to hold**, plus per-key overhead (Redis is on the order of tens of bytes of overhead per key, more with small-value fragmentation), plus headroom so eviction is rare. If the mandatory working set is larger than RAM, the hit ratio falls off a cliff once you spill; that is a partitioning or a "cache only the head of the Zipf curve" decision.

Measure hit ratio **per endpoint and per key family**. A global 99% can hide a single uncacheable route that is the database's whole load.

Watch: hit ratio, miss latency, evictions per second, memory vs maxmemory, expired keys, origin QPS and p99, and the hottest keys. An eviction rate that climbs while hit ratio drops means the working set no longer fits.

CDN origin math is the same formula and is worked in [system_capacity.md](./system_capacity.md).

---

## HTTP and CDN

Application caches and HTTP caches follow the same ideas. HTTP caches are shared between users, so the key and the `Cache-Control` directives are the whole design.

### Freshness vs revalidation

- **Freshness** (`max-age`) lets the cache serve the stored response and skip the origin.
- **Revalidation** (`ETag`, `Last-Modified`) still calls the origin. A `304 Not Modified` saves the body, and it still costs a round trip. Use it when bytes are large and staleness is tight.

### Directives that come up

| Directive | Meaning |
| --------- | ------- |
| `max-age=N` | Fresh for N seconds in a private cache |
| `s-maxage=N` | Fresh for N seconds in a **shared** cache (CDN). Overrides `max-age` there |
| `public` | A shared cache may store it |
| `private` | The browser may store it. A CDN must not |
| `no-store` | Do not store the response (credentials, banking, anything user-specific you will not key carefully) |
| `no-cache` | May store it, and must revalidate before reuse. The name misleads; it does not mean "do not cache" |
| `must-revalidate` | Once stale, do not serve it without a successful revalidation |
| `stale-while-revalidate=N` | After freshness, serve stale for N seconds while refreshing in the background |
| `stale-if-error=N` | After freshness, serve stale for N seconds if the origin errors |
| `immutable` | The URL's bytes will never change. Skip revalidation. Pair with a hashed filename |

### CDN

The CDN caches at the edge so origin bandwidth and origin QPS follow the miss rate. It fits **public, reusable bytes**.

- **Versioned assets.** `app.3f2a1c.js` with `max-age` of a year and `immutable`. A deploy changes the URL. You never purge.
- **User media.** Object storage is the origin, the CDN sits in front, URLs are content-addressed or immutable after upload. The database stores the key.
- **Anonymous catalog GET.** Short `s-maxage` (30–120 s) if the product allows it. Personalization (`Vary: Cookie` or `Authorization`) fragments the cache until the hit ratio is gone. Prefer `private` and cache that response in Redis keyed by user, or assemble the page from a public cached shell plus a small private fragment.
- **Origin shield.** One cache tier in front of origin, shared by all edge POPs, so a cold edge does not turn into N origin fetches of the same object.
- **Purge** is best-effort and slow globally. Use it for takedowns. Use versioned URLs for deploys.

`Vary` is part of the cache key. `Vary: Accept-Encoding` is normal. `Vary: *` or `Vary: Cookie` on a shared cache means you will not hit.

Dynamic JSON APIs usually bypass the CDN. Say that, then make an exception only for a public GET whose staleness you can name.

---

## Local and Distributed

| | In-process | Redis / Memcached |
| --- | ---------- | ----------------- |
| Hit latency | Nanoseconds to microseconds | ~1 ms in-region |
| Consistency across the fleet | Each box has its own copy; a write is invisible until TTL or a bus message | One shared entry; delete-on-write is visible to the next GET |
| Size | MB to a couple GB before you steal the heap from the service | The cluster's RAM |
| Failure | Dies with the process; other instances unaffected | A down cluster is a fleet-wide miss |
| Hot key | This is how you protect Redis | This is what melts one shard |

**Two-level cache:** in-process (TTL 0.5–2 s) in front of Redis (TTL minutes) in front of the database. The short local TTL is the invalidation story. Pub/sub can cut it further; a missed message heals when the local TTL ends. Use the local tier for keys hot enough to threaten a shard, and for data that is fine a second stale (flags, rankings). Skip it when read-your-writes across instances matters at the sub-second level.

**Redis vs Memcached, as caches.** Memcached is a multithreaded cache: slabs, no persistence, no rich types, horizontal scale by the client. Redis is single-threaded per shard, has structures, replication, and optional persistence. For a pure cache, either works. If you pick Redis, treat persistence as incidental: the database is the source of truth, eviction is on, every cache key has a TTL. Turning on AOF does not make the cache safe to be the only copy. Product details: [redis.md](./redis.md).

**Client timeouts.** A cache call uses a tight timeout and a small connection pool. A stuck Redis that holds request threads for seconds takes the API down even though the database was healthy. Fail the cache call, then apply the origin limiter.

**Deploy.** Rolling deploys mix old and new code. Version the key or the serializer so old bytes are a miss, and so both versions can run during the rollout.

---

## What to Say in the Interview

Walk this when you draw the cache. Thirty seconds, then go to the bottleneck that made you add it.

1. **Why this box.** Name the repeated read and the origin cost. State a hit-ratio assumption and the origin QPS that remains.
2. **Layer.** CDN for public bytes, Redis for shared computed data, in-process only for a hot key or a sub-second TTL. Database stays the source of truth.
3. **Pattern.** Cache-aside, delete-on-write, jittered TTL. Say write-back only if you also say what a crash loses.
4. **Freshness.** Who can be stale, for how long, and how the writer sees their own write.
5. **Miss path.** Who loads the database, singleflight or soft TTL so a hot expiry is one rebuild, negative cache if ids can be garbage.
6. **Failure.** Cache down → short timeout, limited origin fallback, shed or serve stale. Jitter so keys do not expire together.
7. **Hot key.** If one id can outrun a shard, local cache or replicated keys. Sharding does not fix it.
8. **Key.** What is in it (id, version, locale), what would leak if it were shared, and that cardinality is bounded.

Red flags interviewers notice: caching the balance that prevents oversell, no TTL, no story for a flushed Redis, a personalized response on a CDN without a user in the key, "we'll just use Redis" with no hit ratio, write-back for orders, and a lock on every read.

Prompts where this is the deep dive: a read-heavy feed or catalog, a celebrity hot key, a CDN in front of media, or "the database is on fire and the working set fits in memory." Prompts where it is a trap: inventory, payments, and anything that required linearizability in the requirements. Say you are not caching that path.