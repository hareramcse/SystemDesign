# System Design

A collection of system design concepts, patterns, and best practices.

---

## Table of Contents

- [Caching Concepts](#caching-concepts)
  - [What is Near Cache](#what-is-near-cache)
  - [What is Cache Stampede](#what-is-cache-stampede)
  - [What is Cache Avalanche](#what-is-cache-avalanche)
  - [What is Cache Penetration](#what-is-cache-penetration)
  - [What is Cache Warming](#what-is-cache-warming)

---

## Caching Concepts

### What is Near Cache

**Near Cache** (also called **local cache** or **L1 cache**) is a small, fast cache that sits **close to the application**—typically in the same JVM process or on the same host—as opposed to a remote/distributed cache (e.g., Redis, Memcached) that requires a network round trip.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Application │ ──► │ Near Cache  │ ──► │ Remote Cache│ ──► Database
│             │     │  (in-memory)│     │   (Redis)   │
└─────────────┘     └─────────────┘     └─────────────┘
     L0/L1               L2                    Source of truth
```

**How it works**

- On a read, the application checks the near cache first.
- On a **hit**, data is returned immediately with no network call.
- On a **miss**, the application fetches from the remote cache (or database), then stores a copy locally for future requests.

**Benefits**

| Benefit | Description |
|---------|-------------|
| **Low latency** | In-process memory access is orders of magnitude faster than network I/O |
| **Reduced load** | Fewer calls to the remote cache and database |
| **Better throughput** | Handles hot keys locally without saturating the network |

**Trade-offs**

- **Memory per instance** — Each node holds its own copy; total memory usage scales with instance count.
- **Consistency** — Local copies can become stale when data changes elsewhere. Mitigations include TTL, event-driven invalidation, or write-through updates.
- **Duplication** — The same key may be cached on many nodes (acceptable for read-heavy, eventually consistent data).

**Common implementations**

- Caffeine, Guava Cache (Java)
- `sync.Map` with TTL (Go)
- Application-level caching in Hazelcast/Ignite (embedded near-cache mode)

**When to use**

- Read-heavy workloads with hot keys
- Latency-sensitive services where even a 1–2 ms Redis call is too slow
- Data that can tolerate brief staleness (configuration, product catalogs, user profiles)

---

### What is Cache Stampede

**Cache Stampede** (also called **cache dog-piling** or **thundering herd on cache miss**) occurs when a **popular cache entry expires** and **many concurrent requests** simultaneously discover the miss and all try to **rebuild the cache** at once—typically by hitting the database.

```
Time ─────────────────────────────────────────────────────────►

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

- A single hot key expires (TTL reached) or is evicted.
- Thousands of threads/processes request that key at the same moment.
- Every miss triggers an expensive backend call instead of one coordinated rebuild.

**Impact**

- Database or downstream service spike in load
- Increased latency and possible timeouts
- Risk of cascading failure under heavy traffic

**Mitigation strategies**

| Strategy | How it works |
|----------|--------------|
| **Locking / single-flight** | Only one thread rebuilds; others wait or get a stale value |
| **Probabilistic early expiration** | Refresh before TTL expires (e.g., random jitter per key) |
| **Stale-while-revalidate** | Serve stale data while one worker refreshes in the background |
| **Mutex per key** | Distributed lock (Redis `SETNX`) so only one instance rebuilds |
| **Never expire hot keys** | Background refresh instead of hard TTL expiry |
| **Request coalescing** | Batch duplicate in-flight requests for the same key |

**Example (single-flight pattern)**

```text
if cache.miss(key):
    if acquire_lock(key):
        value = db.fetch(key)
        cache.set(key, value)
        release_lock(key)
    else:
        wait_for_lock_or_retry(key)
```

---

### What is Cache Avalanche

**Cache Avalanche** (also called **cache avalanche effect**) occurs when **a large number of cache entries expire at roughly the same time** (or the cache layer fails entirely), causing a **mass wave of cache misses** that overwhelms the backend.

```
Many keys set with same TTL at deploy time
         │
         ▼
   T + 3600s: ALL expire together
         │
         ▼
   ████████████████████  ← Mass miss wave
         │
         ▼
   Database / services collapse under sudden load
```

**Cache Stampede vs Cache Avalanche**

| | Cache Stampede | Cache Avalanche |
|---|----------------|-----------------|
| **Scope** | Usually one **hot key** | **Many keys** (or entire cache) |
| **Trigger** | Single expiry / eviction | Bulk expiry, cache restart, or cluster failure |
| **Scale** | Concentrated on popular data | System-wide miss storm |

**Common causes**

- Setting the same TTL for all keys at once (e.g., after a bulk cache warm)
- Cache cluster restart or failover with cold cache
- Memory pressure causing mass eviction (LRU flush)
- Network partition isolating the cache tier

**Mitigation strategies**

| Strategy | Description |
|----------|-------------|
| **TTL jitter** | Add random offset to expiry: `TTL = base + random(0, jitter)` so keys don't expire together |
| **Staggered warming** | Populate cache gradually, not all at once |
| **Tiered TTL** | Different TTLs by data type or access pattern |
| **Circuit breaker** | Protect backend when miss rate spikes |
| **Graceful degradation** | Return defaults or limited responses instead of hammering DB |
| **High availability cache** | Replication, persistence, multi-AZ to avoid full cold start |

---

### What is Cache Penetration

**Cache Penetration** occurs when requests ask for **data that does not exist** (and will never exist) in the backend. Because there is nothing to cache meaningfully, **every request bypasses the cache** and hits the database—often repeatedly for the same invalid keys.

```
Attacker / bug: GET /user?id=-1, GET /user?id=999999999
         │
         ▼
   Cache: no entry (null not cached, or attacker uses random IDs)
         │
         ▼
   Database queried every time  ← Sustained load, no cache benefit
```

**Typical scenarios**

- Malicious requests with random or non-existent IDs
- Application bugs generating invalid lookups
- Scanning/enumeration attacks on ID-based APIs

**Impact**

- Cache provides no protection for these requests
- Database load grows with attack volume or bug frequency
- Can be used as a denial-of-service vector

**Mitigation strategies**

| Strategy | Description |
|----------|-------------|
| **Cache null / negative caching** | Store a short-TTL placeholder for "not found" results |
| **Bloom filter** | Fast check: "this key definitely does not exist" before DB |
| **Input validation** | Reject malformed IDs at the API layer (format, range) |
| **Rate limiting** | Throttle suspicious clients or IP ranges |
| **Authentication / authorization** | Reduce anonymous abuse |
| **Separate existence check** | Lightweight index before full DB query |

**Example (negative caching)**

```text
value = cache.get(key)
if value == NOT_FOUND_SENTINEL:
    return null  # No DB hit

value = db.get(key)
if value == null:
    cache.set(key, NOT_FOUND_SENTINEL, ttl=60s)
else:
    cache.set(key, value, ttl=3600s)
```

---

### What is Cache Warming

**Cache Warming** (also called **cache preloading** or **cache priming**) is the practice of **loading data into the cache before it is needed**, so that the first real user requests are **cache hits** instead of expensive cold misses.

```
Without warming:          With warming:
  User request ──► MISS     [Deploy] ──► Preload hot data
       │                              User request ──► HIT
       ▼
  Slow DB query
```

**When to use**

- Application startup or deployment (avoid cold-start latency spike)
- Before predictable traffic peaks (sales events, product launches)
- After cache flush, failover, or cluster rebuild
- When certain keys are known to be hot (top products, config, feature flags)

**Approaches**

| Approach | Description |
|----------|-------------|
| **Eager warming at startup** | Load critical keys when the service boots |
| **Lazy + background refresh** | Serve misses but asynchronously populate hot keys |
| **Scheduled warming** | Cron job refreshes cache before peak hours |
| **Event-driven warming** | On deploy or data change, push updates to cache |
| **Read-through on deploy** | Replay recent access logs to preload likely keys |

**Best practices**

- Warm **only hot / critical data** — warming everything is slow and wasteful
- Use **staggered TTL** when warming many keys to avoid cache avalanche
- Monitor warm duration and success rate during deploys
- Combine with **health checks** — don't mark instance ready until warm completes (if latency SLO requires it)
- For distributed systems, coordinate warming to avoid all instances hitting DB simultaneously

**Example flow**

```text
on_startup():
    hot_keys = analytics.get_top_keys(limit=1000)
    for key in hot_keys:
        value = db.fetch(key)
        cache.set(key, value, ttl=3600 + random_jitter())
```

---

## Summary

| Concept | Problem | Core idea |
|---------|---------|-----------|
| **Near Cache** | Remote cache latency | Keep a fast local copy close to the app |
| **Cache Stampede** | Many requests rebuild one expired hot key | Coordinate refresh (lock, single-flight) |
| **Cache Avalanche** | Many keys expire or cache fails at once | TTL jitter, HA cache, staggered warming |
| **Cache Penetration** | Requests for non-existent data never hit cache | Negative caching, Bloom filter, validation |
| **Cache Warming** | Cold cache causes slow first requests | Preload hot data before traffic arrives |
