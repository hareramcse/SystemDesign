# System Design

A single reference for system design concepts, patterns, and best practices.

---

## Topics

| # | Topic | Status |
|---|-------|--------|
| 1 | [Caching](#1-caching) | Documented |
| 2 | Load Balancer | Coming soon |
| 3 | Database | Coming soon |
| 4 | Messaging | Coming soon |
| 5 | API Gateway | Coming soon |

---

# 1. Caching

Caching stores frequently accessed data in fast storage (memory) to reduce latency and backend load.

### Sub-topics

| # | Sub-topic |
|---|-----------|
| 1.1 | [Cache Fundamentals](#11-cache-fundamentals) |
| 1.2 | [Cache Aside Pattern](#12-cache-aside-pattern) |
| 1.3 | [Read Through Cache](#13-read-through-cache) |
| 1.4 | [Write Through Cache](#14-write-through-cache) |
| 1.5 | [Write Back Cache](#15-write-back-cache) |
| 1.6 | [Refresh Ahead Cache](#16-refresh-ahead-cache) |
| 1.7 | [Distributed Cache](#17-distributed-cache) |
| 1.8 | [Near Cache](#18-near-cache) |
| 1.9 | [Cache Invalidation](#19-cache-invalidation) |
| 1.10 | [Cache Stampede](#110-cache-stampede) |
| 1.11 | [Cache Avalanche](#111-cache-avalanche) |
| 1.12 | [Cache Penetration](#112-cache-penetration) |
| 1.13 | [Cache Warming](#113-cache-warming) |

---

## 1.1 Cache Fundamentals

**What is a cache?**

A cache is a temporary storage layer that holds copies of data closer to where it is consumed, so reads (and sometimes writes) are faster than going to the primary data store every time.

```
Client ──► Cache (fast) ──► Database (slow, source of truth)
              │
              └── Hit  → return immediately
              └── Miss → fetch from DB, optionally store in cache
```

**Why use caching?**

| Goal | How caching helps |
|------|-------------------|
| **Lower latency** | Memory access is microseconds vs milliseconds for disk/network |
| **Higher throughput** | Serve more requests with the same backend capacity |
| **Cost reduction** | Fewer database queries and API calls |
| **Resilience** | Serve stale or cached data when backend is degraded |

**Key concepts**

| Term | Meaning |
|------|---------|
| **Cache hit** | Requested data found in cache |
| **Cache miss** | Data not in cache; must be loaded from source |
| **TTL (Time To Live)** | How long an entry stays valid before expiry |
| **Eviction** | Removing entries when cache is full (LRU, LFU, FIFO) |
| **Hit ratio** | `hits / (hits + misses)` — primary health metric |

**Cache placement in the stack**

| Layer | Example | Latency |
|-------|---------|---------|
| CPU L1/L2/L3 | Hardware | Nanoseconds |
| In-process (near cache) | Caffeine, Guava | Microseconds |
| Distributed cache | Redis, Memcached | Sub-millisecond to few ms |
| CDN / edge | CloudFront, Akamai | Edge-local |
| Database buffer pool | InnoDB, PostgreSQL shared buffers | In-DB memory |

**When caching helps**

- Read-heavy workloads (90%+ reads)
- Data that changes infrequently relative to read frequency
- Expensive computations or aggregations
- Hot keys accessed by many users

**When caching hurts or is risky**

- Strong consistency requirements on every read
- Data that changes constantly and must always be fresh
- Very large objects that don't fit memory budget
- Low-traffic data where cache overhead exceeds benefit

**Choosing a caching pattern**

The sections below (1.2–1.6) describe who owns read/write logic — the application or the cache layer. Pick based on consistency needs, complexity tolerance, and write frequency.

---

## 1.2 Cache Aside Pattern

**Also known as:** Lazy loading, Look-aside cache

The **application** is responsible for reading and writing the cache. The cache does not talk to the database on its own.

**Read flow**

```
1. App reads from cache
2. Hit  → return data
3. Miss → App reads from DB
4.        App writes result to cache
5.        Return data
```

**Write flow**

```
1. App writes to DB first (source of truth)
2. App invalidates or updates cache entry
```

**Diagram**

```
┌─────────────┐
│ Application │ ◄── owns all cache logic
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
 Cache   Database
```

**Pros**

- Simple and widely used; full control in application code
- Cache only contains data that was actually requested (no wasted memory)
- Cache failure does not block DB — app can still read/write DB directly

**Cons**

- Application code must handle cache logic (miss, populate, invalidate)
- First request after miss is always slow (cold miss)
- Risk of stale data if invalidation is missed on write

**Best for**

- General-purpose caching (most common pattern in production)
- Read-heavy systems where not all data needs to be cached
- Teams that want explicit control over what gets cached

**Example (pseudo-code)**

```text
function get(key):
    value = cache.get(key)
    if value != null:
        return value
    value = db.get(key)
    if value != null:
        cache.set(key, value, ttl=3600)
    return value

function update(key, data):
    db.update(key, data)
    cache.delete(key)   // or cache.set(key, data)
```

---

## 1.3 Read Through Cache

The **cache** sits in front of the database. On a miss, the **cache layer** (not the application) loads data from the DB and stores it automatically.

**Read flow**

```
1. App asks cache for key
2. Hit  → cache returns data
3. Miss → cache fetches from DB, stores entry, returns data
```

**Diagram**

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │  (only talks to cache)
       ▼
┌─────────────┐
│    Cache    │ ── on miss ──► Database
│  (loader)   │
└─────────────┘
```

**Pros**

- Simpler application code — no miss-handling logic in app
- Centralized loading logic in cache provider
- Consistent load behavior across all clients

**Cons**

- Requires cache product/library that supports read-through
- Less flexibility than cache-aside for custom load logic
- Still need a write strategy (often combined with write-through or cache-aside writes)

**Best for**

- ORM-level or framework-integrated caches (e.g., Hibernate second-level cache)
- Standardized data access layers where cache is a transparent proxy

**Cache Aside vs Read Through**

| | Cache Aside | Read Through |
|---|-------------|--------------|
| **Who loads on miss?** | Application | Cache layer |
| **Complexity** | In app code | In cache config |
| **Flexibility** | High | Lower |

---

## 1.4 Write Through Cache

On every **write**, data is written to the **cache and the database synchronously** before the operation is considered complete.

**Write flow**

```
1. App writes to cache
2. Cache synchronously writes to DB
3. Acknowledge success to app only after both succeed
```

**Read flow**

Usually combined with read-through or cache-aside reads — data in cache is always consistent with what was written.

**Diagram**

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │ write
       ▼
┌─────────────┐ ── synchronous ──► Database
│    Cache    │
└─────────────┘
```

**Pros**

- Cache and DB stay consistent on writes
- No stale writes in cache after successful update
- Good for read-heavy workloads after write

**Cons**

- Higher write latency (two writes per operation)
- Writes to data that is never read still update cache (wasted work unless filtered)
- Cache can hold data nobody reads

**Best for**

- Systems where read-after-write consistency matters
- Moderate write volume with heavy read traffic on same keys

---

## 1.5 Write Back Cache

**Also known as:** Write-behind cache

Writes go to the **cache first**; the cache **asynchronously** flushes updates to the database later.

**Write flow**

```
1. App writes to cache
2. Cache acknowledges immediately
3. Cache batches / queues writes to DB in background
```

**Diagram**

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │ write (fast)
       ▼
┌─────────────┐ ── async batch ──► Database
│    Cache    │
└─────────────┘
```

**Pros**

- Very low write latency for the application
- Batching reduces DB write load
- Good for bursty write workloads

**Cons**

- Risk of **data loss** if cache node fails before flush to DB
- Complexity: ordering, retries, conflict resolution
- Brief window where cache and DB disagree

**Best for**

- Write-heavy, latency-sensitive systems that can tolerate eventual persistence
- Counters, analytics buffers, session-like data with recovery strategy
- Not ideal for financial or critical transactional data without durability guarantees

**Write Through vs Write Back**

| | Write Through | Write Back |
|---|---------------|------------|
| **Write latency** | Higher (sync DB) | Lower (async DB) |
| **Consistency** | Stronger | Eventual |
| **Durability risk** | Lower | Higher if cache fails |

---

## 1.6 Refresh Ahead Cache

The cache **proactively refreshes** entries **before they expire**, based on access patterns or a refresh schedule — so users rarely see a miss on hot keys.

**How it works**

```
1. Key is accessed frequently (hot key)
2. Cache tracks access / remaining TTL
3. Before TTL expires, cache triggers background reload from DB
4. User requests always hit a fresh (or slightly stale) entry
```

**Diagram**

```
Access pattern detected (hot key)
         │
         ▼
   TTL at 20% remaining
         │
         ▼
   Background refresh ──► DB
         │
         ▼
   Cache updated before expiry  →  no stampede on expire
```

**Pros**

- Avoids latency spike on expiry for hot keys
- Reduces cache stampede risk for popular data
- Smooth user experience for frequently read data

**Cons**

- May refresh data that is no longer needed (wasted DB reads)
- Requires access tracking or predictive logic
- More complex than simple TTL expiry

**Best for**

- Hot keys with predictable access (home page, top products, config)
- Often implemented in libraries like Caffeine (`refreshAfterWrite`)
- Pairs well with [Cache Stampede](#110-cache-stampede) mitigation

**Related patterns**

- **Stale-while-revalidate** — serve old value while refreshing in background
- **Refresh ahead** — refresh before expiry, not after

---

## 1.7 Distributed Cache

A **distributed cache** is a shared cache tier used by **multiple application instances**, typically running on separate servers and accessed over the network.

**Architecture**

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App 1   │  │  App 2   │  │  App 3   │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   ▼
         ┌─────────────────┐
         │ Distributed Cache│  (Redis, Memcached, Hazelcast)
         │   Cluster        │
         └────────┬─────────┘
                  ▼
              Database
```

**Why distributed?**

| Benefit | Description |
|---------|-------------|
| **Shared state** | All app nodes see the same cached data |
| **Horizontal scale** | Add cache nodes to grow capacity |
| **Centralized invalidation** | Delete key once → all apps see miss |
| **Persistence options** | Redis RDB/AOF for recovery |

**Common technologies**

| System | Notes |
|--------|-------|
| **Redis** | Rich data structures, pub/sub, persistence, cluster mode |
| **Memcached** | Simple key-value, multi-threaded, no persistence |
| **Hazelcast / Ignite** | In-memory data grid, near-cache support |
| **Amazon ElastiCache** | Managed Redis/Memcached |

**Design considerations**

- **Partitioning (sharding)** — split keys across nodes for scale
- **Replication** — copies for HA and read scaling
- **Consistency** — eventual by default; strong consistency costs latency
- **Serialization** — JSON, Protobuf, Kryo — affects size and speed
- **Network latency** — typically 0.5–2 ms per round trip vs microseconds for local cache

**Distributed vs local cache**

Use **distributed** when multiple instances must share cache. Use **local (near) cache** ([1.8](#18-near-cache)) on top when you need microsecond access for hottest keys.

---

## 1.8 Near Cache

**Also known as:** Local cache, L1 cache

A small, fast cache in the **same process** as the application, in front of a remote distributed cache.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Application │ ──► │ Near Cache  │ ──► │ Remote Cache│ ──► Database
│             │     │  (in-memory)│     │   (Redis)   │
└─────────────┘     └─────────────┘     └─────────────┘
     L0/L1               L2                    Source of truth
```

**How it works**

- On read: check near cache → then remote cache → then database
- On hit in near cache: return with no network call
- On miss: fetch from remote/DB, populate local copy

**Benefits**

| Benefit | Description |
|---------|-------------|
| **Low latency** | In-process memory is orders of magnitude faster than network |
| **Reduced load** | Fewer calls to remote cache and database |
| **Better throughput** | Hot keys served locally without saturating network |

**Trade-offs**

- **Memory per instance** — each node holds its own copy
- **Consistency** — local copies can be stale; use TTL or event-driven invalidation
- **Duplication** — same key cached on many nodes (OK for read-heavy, eventually consistent data)

**Common implementations**

- Caffeine, Guava Cache (Java)
- Hazelcast/Ignite embedded near-cache mode

**When to use**

- Read-heavy workloads with hot keys
- Latency-sensitive services where even a 1–2 ms Redis call is too slow
- Data that tolerates brief staleness (catalogs, config, profiles)

---

## 1.9 Cache Invalidation

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

**Cache invalidation** is the process of removing or updating stale cache entries when the underlying data changes.

**Why it matters**

Without proper invalidation, users read outdated data. With overly aggressive invalidation, you lose cache benefits.

**Strategies**

| Strategy | How it works | Pros | Cons |
|----------|--------------|------|------|
| **TTL expiry** | Entry auto-expires after fixed time | Simple, no coordination | Stale data until expiry |
| **Delete on write** | Remove cache key when DB is updated | Fresh on next read | Extra write; miss after update |
| **Update on write** | Write new value to cache on DB update | Next read is hit | Must keep logic in sync |
| **Event-driven** | DB change → message → all nodes invalidate | Scales across instances | Needs pub/sub or change data capture |
| **Version / ETag** | Cache entry tagged with version; reject if mismatch | Fine-grained control | More complex reads |

**Invalidation flow (delete on write)**

```
1. App updates database
2. App deletes cache key (or publishes invalidation event)
3. Next read → miss → load fresh data from DB
```

**Event-driven invalidation**

```
Database ──► CDC / binlog ──► Kafka ──► All app instances ──► cache.delete(key)
```

**Common pitfalls**

- Forgetting to invalidate on one code path (partial updates)
- Race: read repopulates stale data after delete but before DB commit
- Invalidating too broadly (e.g., flush entire cache on any write)

**Mitigation for races**

- Transaction ordering: commit DB first, then invalidate
- Short TTL as safety net
- Version numbers in cache value

---

## 1.10 Cache Stampede

**Also known as:** Cache dog-piling, thundering herd on cache miss

Occurs when a **popular cache entry expires** and **many concurrent requests** simultaneously miss and all try to **rebuild the cache** at once — typically hammering the database.

```
Cache entry expires at T
         │
         ▼
   Request 1 ──► MISS ──► DB query
   Request 2 ──► MISS ──► DB query   } All at the same time
   Request 3 ──► MISS ──► DB query
   ...
   Request N ──► MISS ──► DB query   ← Database overwhelmed
```

**Cause**

- Single hot key expires or is evicted
- Thousands of concurrent requests for that key
- Every miss triggers a full backend rebuild instead of one coordinated reload

**Mitigation**

| Strategy | How it works |
|----------|--------------|
| **Locking / single-flight** | Only one thread rebuilds; others wait or get stale value |
| **Probabilistic early expiration** | Refresh before TTL with random jitter |
| **Stale-while-revalidate** | Serve stale data while one worker refreshes |
| **Mutex per key** | Distributed lock (Redis `SETNX`) — one instance rebuilds |
| **Never expire hot keys** | Background refresh instead of hard expiry |
| **Request coalescing** | Merge duplicate in-flight requests for same key |

**Example (single-flight)**

```text
if cache.miss(key):
    if acquire_lock(key):
        value = db.fetch(key)
        cache.set(key, value)
        release_lock(key)
    else:
        wait_for_lock_or_retry(key)
```

See also: [Refresh Ahead Cache](#16-refresh-ahead-cache)

---

## 1.11 Cache Avalanche

**Also known as:** Cache avalanche effect

Occurs when **a large number of cache entries expire at roughly the same time** (or the cache layer fails), causing a **mass wave of misses** that overwhelms the backend.

```
Many keys set with same TTL at deploy time
         │
         ▼
   T + 3600s: ALL expire together
         │
         ▼
   Mass miss wave ──► Database collapse
```

**Cache Stampede vs Cache Avalanche**

| | Cache Stampede | Cache Avalanche |
|---|----------------|-----------------|
| **Scope** | Usually one **hot key** | **Many keys** or entire cache |
| **Trigger** | Single expiry / eviction | Bulk expiry, restart, cluster failure |
| **Scale** | Concentrated on popular data | System-wide miss storm |

**Common causes**

- Same TTL for all keys set at once (e.g., after bulk warm)
- Cache cluster restart or cold failover
- Memory pressure → mass LRU eviction
- Network partition isolating cache tier

**Mitigation**

| Strategy | Description |
|----------|-------------|
| **TTL jitter** | `TTL = base + random(0, jitter)` — spread expiry times |
| **Staggered warming** | Populate cache gradually |
| **Tiered TTL** | Different TTLs by data type |
| **Circuit breaker** | Protect backend when miss rate spikes |
| **Graceful degradation** | Defaults instead of hammering DB |
| **HA cache** | Replication, persistence, multi-AZ |

---

## 1.12 Cache Penetration

Occurs when requests ask for **data that does not exist** (and will never exist). Nothing useful gets cached, so **every request hits the database**.

```
GET /user?id=-1, GET /user?id=999999999
         │
         ▼
   Cache: no entry (null not cached)
         │
         ▼
   Database queried every time
```

**Typical scenarios**

- Malicious requests with random or non-existent IDs
- Application bugs with invalid lookups
- ID enumeration / scanning attacks

**Mitigation**

| Strategy | Description |
|----------|-------------|
| **Negative caching** | Store short-TTL placeholder for "not found" |
| **Bloom filter** | Fast "definitely does not exist" check before DB |
| **Input validation** | Reject malformed IDs at API layer |
| **Rate limiting** | Throttle suspicious clients |
| **Auth** | Reduce anonymous abuse |

**Example (negative caching)**

```text
value = cache.get(key)
if value == NOT_FOUND_SENTINEL:
    return null

value = db.get(key)
if value == null:
    cache.set(key, NOT_FOUND_SENTINEL, ttl=60s)
else:
    cache.set(key, value, ttl=3600s)
```

---

## 1.13 Cache Warming

**Also known as:** Cache preloading, cache priming

Loading data into the cache **before it is needed**, so first real user requests are **hits** instead of cold misses.

```
Without warming:          With warming:
  User request ──► MISS     [Deploy] ──► Preload hot data
       │                              User request ──► HIT
       ▼
  Slow DB query
```

**When to use**

- Application startup or deployment
- Before predictable traffic peaks (sales, launches)
- After cache flush, failover, or cluster rebuild
- Known hot keys (top products, config, feature flags)

**Approaches**

| Approach | Description |
|----------|-------------|
| **Eager warming at startup** | Load critical keys when service boots |
| **Lazy + background refresh** | Populate hot keys asynchronously |
| **Scheduled warming** | Cron before peak hours |
| **Event-driven warming** | On deploy or data change, push to cache |
| **Replay access logs** | Preload keys from recent traffic patterns |

**Best practices**

- Warm only **hot / critical** data
- Use **TTL jitter** when warming many keys — avoid [Cache Avalanche](#111-cache-avalanche)
- Monitor warm duration during deploys
- Coordinate across instances so all nodes don't hit DB at once

**Example**

```text
on_startup():
    hot_keys = analytics.get_top_keys(limit=1000)
    for key in hot_keys:
        value = db.fetch(key)
        cache.set(key, value, ttl=3600 + random_jitter())
```

---

### Caching — Quick Reference

| Sub-topic | One-line summary |
|-----------|------------------|
| **1.1 Fundamentals** | Why and when to cache; hits, misses, TTL, eviction |
| **1.2 Cache Aside** | App manages cache; load on miss, invalidate on write |
| **1.3 Read Through** | Cache loads from DB on miss automatically |
| **1.4 Write Through** | Sync write to cache and DB together |
| **1.5 Write Back** | Write to cache first; async flush to DB |
| **1.6 Refresh Ahead** | Proactively refresh before expiry |
| **1.7 Distributed Cache** | Shared remote cache across app instances |
| **1.8 Near Cache** | Fast local L1 in front of remote cache |
| **1.9 Invalidation** | Keep cache consistent when data changes |
| **1.10 Stampede** | Many requests rebuild one expired hot key |
| **1.11 Avalanche** | Mass expiry or cache failure → miss storm |
| **1.12 Penetration** | Non-existent keys bypass cache → DB abuse |
| **1.13 Warming** | Preload hot data before traffic arrives |

---

# 2. Load Balancer

*Coming soon*

---

# 3. Database

*Coming soon*

---

# 4. Messaging

*Coming soon*

---

# 5. API Gateway

*Coming soon*
