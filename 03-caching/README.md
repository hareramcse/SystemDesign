# 3. Caching

> Status: **Documented**  -  master reference

[<- Back to master index](../README.md)

## Sub-topics

| # | Sub-topic | Status |
|---|-----------|--------|
| 3.1 | [Cache Fundamentals](#31-cache-fundamentals) | Done |
| 3.2 | [Cache Aside Pattern](#32-cache-aside-pattern) | Done |
| 3.3 | [Read Through Cache](#33-read-through-cache) | Done |
| 3.4 | [Write Through Cache](#34-write-through-cache) | Done |
| 3.5 | [Write Back Cache](#35-write-back-cache) | Done |
| 3.6 | [Local Cache](#36-local-cache) | Done |
| 3.7 | [Distributed Cache](#37-distributed-cache) | Done |
| 3.8 | [Near Cache](#38-near-cache) | Done |
| 3.9 | [Cache Invalidation](#39-cache-invalidation) | Done |
| 3.10 | [Cache Stampede](#310-cache-stampede) | Done |
| 3.11 | [Cache Avalanche](#311-cache-avalanche) | Done |
| 3.12 | [Cache Penetration](#312-cache-penetration) | Done |
| 3.13 | [Cache Warming](#313-cache-warming) | Done |
| 3.14 | [Write Around Cache](#314-write-around-cache) | Done |

---

## 3.1 Cache Fundamentals

### What is cache?

Cache is a high-speed storage layer that stores frequently accessed data temporarily so future requests can be served faster without hitting the original data source.

**Goal:**

- Reduce latency
- Reduce database load
- Improve throughput
- Improve user experience

**Example: User Profile Request**

Without Cache:

```text
Client -> App -> Database -> App -> Client
```

With Cache:

```text
Client -> App -> Cache -> Client
```

If data not found in cache:

```text
Client -> App -> Cache(Miss) -> Database
                               -> Cache Store
                               -> Client
```

### Why do we need cache?

**Problems Without Cache:**

- High database load
- Increased response time
- Expensive queries executed repeatedly
- Poor scalability

**Benefits:**

- Faster response time
- Reduced database traffic
- Better application scalability
- Lower infrastructure cost

**Example:**

Database Query Time = 500ms

Without Cache:

```text
Request 1 = 500ms
Request 2 = 500ms
Request 3 = 500ms
```

With Cache:

```text
Request 1 = 500ms (Cache Miss)
Request 2 = 5ms   (Cache Hit)
Request 3 = 5ms   (Cache Hit)
```

### Cache hit and cache miss

**Cache Hit:**
Requested data exists in cache.

```text
Client -> App -> Cache -> Client
```

Response Time: ~1ms to 10ms

**Cache Miss:**
Requested data not found in cache.

```text
Client -> App -> Cache -> Database
                               -> Cache
                               -> Client
```

Response Time: Depends on database.

**Formula:**

```text
Cache Hit Ratio = (Cache Hits / Total Requests) * 100
```

**Example:**

```text
Hits = 900
Misses = 100

Hit Ratio = 900/1000 = 90%
```

### Types of cache

A) Browser Cache
B) CDN Cache
C) Application Cache
D) Distributed Cache
E) Database Cache
F) CPU Cache

### In-memory cache

Data stored in RAM.

**Examples:**

- Redis
- Memcached
- Hazelcast
- Caffeine

**Advantages:**

- Extremely fast

**Disadvantages:**

- Limited by RAM
- Data loss on restart (unless persisted)

### Cache eviction policies

When cache becomes full, some data must be removed.

#### A) LRU (Least Recently Used)

Remove least recently accessed item.

**Example:**

```text
Capacity = 3

A B C

Access A

A C B

Add D

Remove B

Result:
A C D
```

#### B) LFU (Least Frequently Used)

Remove item with lowest access count.

**Example:**

```text
A -> 10 accesses
B -> 2 accesses
C -> 5 accesses

Add D

Remove B
```

#### C) FIFO (First In First Out)

Remove oldest entry.

**Example:**

```text
A B C

Add D

Remove A
```

#### D) TTL (Time To Live)

Data automatically expires after time limit.

**Example:**

```text
User Data TTL = 5 minutes

After 5 minutes:
Data removed automatically
```

---

## 3.2 Cache Aside Pattern

Most commonly used strategy.

**Flow:**

1. Check cache
2. If found return
3. If not found query DB
4. Store in cache
5. Return response

**Diagram:**

```text
Client
  |
Application
  |
Cache
  |
Miss
  |
Database
  |
Store in Cache
  |
Return Data
```

**Advantages:**

- Easy to implement
- Cache only frequently used data

**Disadvantages:**

- First request is slow
- Possible stale data

**Used By:**

- Facebook
- Netflix
- Most microservices

---

## 3.3 Read Through Cache

Cache itself talks to database.

**Flow:**

```text
Client
  |
Application
  |
Cache
  |
Database
```

Application never accesses DB directly.

**Advantages:**

- Simpler application code
- Automatic cache management

**Disadvantages:**

- More complex cache layer

---

## 3.4 Write Through Cache

Whenever data is written:

```text
Application
   |
Cache
   |
Database
```

Both cache and DB updated together.

**Example: Update User Name**

```text
Application
    |
Cache Update
    |
Database Update
```

**Advantages:**

- Cache always fresh
- High read performance

**Disadvantages:**

- Slower writes

---

## 3.5 Write Back Cache

Data first written to cache.

Database updated later asynchronously.

**Flow:**

```text
Application
     |
Cache
     |
Immediate Success

Background Worker
     |
Database
```

**Advantages:**

- Very fast writes

**Disadvantages:**

- Risk of data loss
- Complex implementation

**Used In:**

- High throughput systems
- Analytics systems

---

## 3.6 Local Cache

Cache inside application instance.

**Example:**

```text
App Instance 1
    |
 Local Cache

App Instance 2
    |
 Local Cache
```

**Examples:**

- Caffeine
- Guava Cache

**Advantages:**

- Very fast
- No network call

**Disadvantages:**

- Data inconsistency between nodes

---

## 3.7 Distributed Cache

Separate cache cluster shared by all services.

**Example:**

```text
App1 ----\
App2 ----- Redis Cluster
App3 ----/
```

**Advantages:**

- Shared cache
- Consistent view

**Disadvantages:**

- Network overhead

**Examples:**

- Redis
- Hazelcast
- Ignite

### Redis vs Memcached

**Redis:**

- Rich data structures
- Persistence support
- Replication support
- Pub/Sub
- Lua scripting

**Memcached:**

- Simple key-value
- Pure in-memory
- Multi-threaded
- Less features

---

## 3.8 Near Cache

Combination of local cache and distributed cache.

**Flow:**

```text
Application
     |
Near Cache (Local)
     |
Distributed Cache
     |
Database
```

**Advantages:**

- Faster reads
- Reduced network calls

**Examples:**

- Hazelcast Near Cache
- Apache Ignite Near Cache

### Multi-level cache

```text
L1 Cache = Local Cache
L2 Cache = Distributed Cache
L3 Cache = Database
```

**Flow:**

```text
Application
    |
L1 Cache (Caffeine)
    |
L2 Cache (Redis)
    |
Database
```

**Benefits:**

- Extremely low latency
- Reduced Redis traffic
- Reduced DB traffic

---

## 3.9 Cache Invalidation

One of the hardest problems in software engineering.

**Purpose:**
Remove stale data from cache.

**Methods:**

1. TTL Expiration
2. Manual Invalidation
3. Event Based Invalidation
4. Version Based Invalidation

**Example:**

```text
User Updated Profile

Database Updated
       |
Cache Delete
       |
Next Read Reloads Data
```

### Cache consistency

**Challenge:**

```text
Database = "John"
Cache = "Johnny"
```

Now data is inconsistent.

**Solutions:**

A) Write Through
B) Event Driven Updates
C) Cache Invalidation
D) Short TTL

---

## 3.10 Cache Stampede

### Cache breakdown (hot key problem)

A highly popular key expires.

**Example:**

```text
Product: iPhone

Millions of users request it.

Key expires:
All requests hit DB simultaneously.

Result:
Database overload.
```

**Solutions:**

- Mutex Lock
- Hot Key Never Expires
- Logical Expiration

### Cache stampede

Multiple threads regenerate same cache value.

**Example:**

```text
Key Expired

1000 requests arrive

All hit database.
```

**Solutions:**

- Distributed Lock
- Request Coalescing
- Single Flight Pattern

---

## 3.11 Cache Avalanche

Many cache entries expire together.

**Example:**

```text
100,000 keys
TTL = 1 hour

After 1 hour:
All expire

Traffic hits DB at once.
```

**Solutions:**

- Random TTL
- Multi-level Cache
- Rate Limiting

**Example:**

Instead of:

```text
TTL = 60 min
```

Use:

```text
TTL = 60-90 min random
```

---

## 3.12 Cache Penetration

Requests for non-existing data continuously hit DB.

**Example:**

```text
User ID = 99999999
(not present)

Every request:
Cache Miss -> Database
```

**Solution:**

- Cache null values
- Bloom Filters

---

## 3.13 Cache Warming

Preload cache before traffic arrives.

**Example:**

```text
Application Startup
      |
Load Popular Products
      |
Store in Cache
```

**Benefits:**

- Avoid startup cache misses
- Faster initial responses

---

## 3.14 Write Around Cache

Write directly to database.

```text
Application
     |
Database
```

Cache updated only when read occurs.

**Advantages:**

- Avoid cache pollution

**Disadvantages:**

- First read causes cache miss

**Useful When:**

- Data written frequently
- Data read rarely

---

[<- Back to master index](../README.md)
