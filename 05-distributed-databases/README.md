# 5. Distributed Databases

> Status: **Documented**

[<- Back to master index](../README.md)

---

## Overview

Distributed databases spread data and query load across multiple nodes - often in different racks, regions, or continents - so a single machine is never the bottleneck or single point of failure. They combine **partitioning** (how data is split), **replication** (how copies are kept), and **consensus** (how nodes agree on state) to deliver scale, availability, and durability together.

Designing or operating a distributed database means choosing trade-offs you cannot escape: partition tolerance is mandatory on real networks, so you pick between consistency and availability (CAP), and between latency and consistency when the network is healthy (PACELC). Interview discussions usually center on partition keys, replication mode, quorum math, failure handling, and when distributed transactions are worth the cost.

This chapter walks from horizontal scaling mechanics (sharding, hashing) through replication and quorums, distributed transactions, consensus algorithms, and logical clocks — including a **production multi-layer placement pattern** (shard → bucket → partition) for high-volume OLTP on PostgreSQL-style databases.

---

## Sub-topics

| # | Sub-topic | Status |
|---|-----------|--------|
| 5.1 | [Partitioning](#51-partitioning) | Done |
| 5.2 | [Sharding](#52-sharding) | Done |
| 5.3 | [Hash Partitioning](#53-hash-partitioning) | Done |
| 5.4 | [Range Partitioning](#54-range-partitioning) | Done |
| 5.5 | [Geo Partitioning](#55-geo-partitioning) | Done |
| 5.6 | [Hot Partitions](#56-hot-partitions) | Done |
| 5.7 | [Rebalancing](#57-rebalancing) | Done |
| 5.8 | [Consistent Hashing](#58-consistent-hashing) | Done |
| 5.9 | [Virtual Nodes](#59-virtual-nodes) | Done |
| 5.10 | [Rendezvous Hashing](#510-rendezvous-hashing) | Done |
| 5.11 | [Replication](#511-replication) | Done |
| 5.12 | [Leader Follower Replication](#512-leader-follower-replication) | Done |
| 5.13 | [Multi Leader Replication](#513-multi-leader-replication) | Done |
| 5.14 | [Quorum Reads](#514-quorum-reads) | Done |
| 5.15 | [Quorum Writes](#515-quorum-writes) | Done |
| 5.16 | [Distributed Transactions](#516-distributed-transactions) | Done |
| 5.17 | [Two Phase Commit](#517-two-phase-commit) | Done |
| 5.18 | [Three Phase Commit](#518-three-phase-commit) | Done |
| 5.19 | [Distributed Locking](#519-distributed-locking) | Done |
| 5.20 | [Split Brain](#520-split-brain) | Done |
| 5.21 | [Consensus](#521-consensus) | Done |
| 5.22 | [Paxos](#522-paxos) | Done |
| 5.23 | [Raft](#523-raft) | Done |
| 5.24 | [Leader Election](#524-leader-election) | Done |
| 5.25 | [Lamport Clocks](#525-lamport-clocks) | Done |
| 5.26 | [Vector Clocks](#526-vector-clocks) | Done |
| 5.27 | [Gossip Protocol](#527-gossip-protocol) | Done |
| 5.28 | [Membership Protocols](#528-membership-protocols) | Done |
| 5.29 | [Sharding, Bucketing & Partitioning](#529-sharding-bucketing--partitioning) | Done |





```mermaid
flowchart TB
    Client --> Router[Shard Router]
    Router --> S1[Shard A]
    Router --> S2[Shard B]
    S1 --> R1[Replicas]
    S2 --> R2[Replicas]
    R1 --> Consensus[Consensus Layer]
```

---


---

## 5.1 Partitioning


### What is it?

**Partitioning** divides a logical table or dataset into disjoint **segments** (partitions), each owning a subset of rows or keys. In distributed systems, **horizontal partitioning** splits rows by a partition key; **vertical partitioning** splits columns or related tables onto different stores.

Partitioning is the **unit of placement**: each partition is assigned to one or more nodes, replicated as a group, migrated during rebalancing, and often the scope of ordering guarantees.

**Contrast with replication:** partitioning answers *which subset of data lives here*; replication answers *how many copies of that subset exist*.

### Why it matters

Every distributed database partitions data — the design question is *how*, not *whether*. Partitioning drives:

- **Write/read parallelism** — independent partitions process concurrently
- **Failure isolation** — one bad partition does not take down the whole dataset
- **Operational boundaries** — backup, compaction, and migration happen per partition
- **Query routing** — single-partition queries stay local; scatter-gather costs multiply

Poor partition design creates hot spots, expensive cross-partition transactions, and painful rebalancing.

For a **full production stack** combining sharding, schema-as-bucket, and in-table range partitions (PostgreSQL-style), see [5.29 Sharding, Bucketing & Partitioning](#529-sharding-bucketing--partitioning).

### How it works

1. **Choose partition key** — high cardinality, aligned with dominant query pattern (`user_id`, `tenant_id`, `order_id`).
2. **Apply scheme** — hash, range, list, or composite (see 5.3–5.5).
3. **Map partition → node** — metadata service (Cassandra system tables, DynamoDB partitions, Spanner tablets) or client-side consistent hash.
4. **Route requests** — coordinator or driver looks up partition for key; sends read/write to owning replica group.
5. **Coordinate multi-partition ops** — scatter-gather queries, 2PC, or application-level fan-out when query spans partitions.

```mermaid
flowchart TB
    subgraph Logical
        Table["Logical Table: orders"]
    end
    Table --> P1['Partition 0 / keys hash 0..33%']
    Table --> P2['Partition 1 / keys hash 34..66%']
    Table --> P3['Partition 2 / keys hash 67..100%']
    P1 --> RG1['Replica Group A / Node 1,2,3']
    P2 --> RG2['Replica Group B']
    P3 --> RG3['Replica Group C']
    Client[Client] --> Meta[Partition Map]
    Meta --> RG1
    Meta --> RG2
```

**Worked example — e-commerce orders:**

| Partition key | Query | Partitions touched |
|---------------|-------|-------------------|
| `customer_id` | `SELECT * FROM orders WHERE customer_id = 42` | **1** (ideal) |
| `order_id` | `SELECT * FROM orders WHERE order_id = 99` | **1** if keyed by order |
| `status` | `SELECT * FROM orders WHERE status = 'PENDING'` | **All** (scatter-gather) |
| `created_at` | Range scan last 7 days | **Subset** (range partitioning) |

### Key details

| Type | Split by | Best for | Risk |
|------|----------|----------|------|
| **Horizontal** | Row/key | Scale-out OLTP | Cross-partition joins |
| **Vertical** | Column/table | Wide rows, cold columns | Joins across stores |
| **Functional** | Tenant/region | Compliance, isolation | Uneven tenant sizes |

**Production patterns:**

- **Partition = replication unit** — Cassandra vnodes, Kafka partitions, Spanner tablets all replicate at partition granularity.
- **Partition = ordering scope** — Kafka and DynamoDB Streams guarantee order *within* a partition key, not globally.
- **Co-partitioning** — place related tables on same key (`user_id` on `users` and `profiles`) to enable local joins.
- **Pre-splitting** — create enough empty partitions upfront (Cassandra, HBase) to avoid hot latest-range partition on time-series ingest.

**Partition count guidance:**

| Factor | Too few | Too many |
|--------|---------|----------|
| Throughput | Single-node ceiling | — |
| Metadata | — | Routing table bloat, gossip overhead |
| Rebalancing | Large moves per change | Fine-grained but noisy |
| Consumers | Kafka: under-parallelized | File handle / election cost |

### When to use

Always in distributed storage. Choose strategy based on access pattern:

- **Point lookups, even spread** → hash partitioning (5.3)
- **Range scans, time-series** → range partitioning (5.4)
- **Data residency** → geo/list partitioning (5.5)

### Trade-offs / Pitfalls

- **Wrong partition key** — low-cardinality keys (`status`, `country`) collapse to few partitions; fix at schema time, not after petabytes.
- **Partition boundary changes** — re-keying data is a migration project; prefer keys stable for entity lifetime.
- **Secondary indexes** — often global indexes scatter writes (DynamoDB GSI, Cassandra SASI); understand hidden cross-partition cost.
- **Assuming partitions = shards** — in some systems one shard hosts many partitions (vnode model); terminology varies by product.

### References

- Designing Data-Intensive Applications, Ch. 6 (Partitioning)
- Dynamo (Amazon) partition and consistent hashing paper

---


## 5.2 Sharding


### What is it?

**Sharding** is horizontal partitioning of a dataset across **independent database instances** (shards), each holding a disjoint subset of rows with its own storage engine, CPU, and memory. The cluster collectively stores the full dataset; no single node holds everything.

Sharding is partitioning at **deployment scale** — each shard is often a full database process (MySQL shard, MongoDB shard, Vitess tablet), not just a logical segment on a shared cluster.

**Sharding vs partitioning:**

| Term | Typical meaning |
|------|-----------------|
| Partitioning | Logical segment; may share nodes (Cassandra, Kafka) |
| Sharding | Physical instance boundary; shared-nothing architecture |
| In practice | Used interchangeably — context matters |

### Why it matters

Sharding is the primary path to **write scalability** beyond one machine's limits:

- **Disk** — single-node storage caps (~few TB practical)
- **CPU/memory** — index and buffer pool pressure
- **Connection limits** — connection pools saturate one host
- **Blast radius** — isolate failures and noisy neighbors per shard

Foundational for multi-tenant SaaS (shard per tenant tier), social graphs (shard by `user_id`), and hyperscale OLTP (Vitess, Citus, MongoDB sharded cluster).

### How it works

A **shard key** (partition key) determines shard ownership on every read/write.

```mermaid
flowchart TB
    App[Application] --> Router{Shard Router}
    Router -->|"hash(user_id)=0"| S1[(Shard 1 / users 0-33M)]
    Router -->|"hash(user_id)=1"| S2[(Shard 2 / users 33-66M)]
    Router -->|"hash(user_id)=2"| S3[(Shard 3 / users 66-100M)]
    S1 --> R1[Replicas]
    S2 --> R2[Replicas]
    S3 --> R3[Replicas]
```

**Routing layers:**

| Layer | Examples | Notes |
|-------|----------|-------|
| Application | Custom hash in service code | Simple; logic duplicated |
| Proxy | Vitess VTGate, ProxySQL, mongos | Centralized routing, connection pooling |
| Driver | DynamoDB SDK, Cassandra driver | Client-side token awareness |
| Directory | ZooKeeper shard map, etcd | Flexible; lookup on every request |

**Request flow:**

1. Client sends `GET user:12345`.
2. Router computes `shard = hash(12345) mod N` (or range/directory lookup).
3. Connection pool opens to target shard primary.
4. Query executes locally — no cross-shard coordination.

**Cross-shard query (expensive):**

```mermaid
sequenceDiagram
    participant App
    participant Router
    participant S1 as Shard 1
    participant S2 as Shard 2
    participant S3 as Shard 3
    App->>Router: SELECT COUNT(*) FROM users WHERE country='US'
    Router->>S1: partial count
    Router->>S2: partial count
    Router->>S3: partial count
    S1-->>Router: 1.2M
    S2-->>Router: 0.9M
    S3-->>Router: 1.1M
    Router-->>App: aggregate 3.2M
```

### Key details

| Aspect | Detail |
|--------|--------|
| Shard key | Must appear in **most** queries; high cardinality; avoid monotonic hot keys |
| Cross-shard ops | Joins, global `UNIQUE`, serial `AUTO_INCREMENT` break without extra coordination |
| Resharding | Dual-write, copy-and-cutover, or consistent-hash migration (5.7, 5.8) |
| Shared nothing | Each shard = independent failure domain; no shared disk |
| Schema migrations | Run per shard; version drift is an operational nightmare |

**Production patterns:**

- **Vitess / Citus** — managed sharding with distributed query planner for limited cross-shard SQL.
- **MongoDB** — `mongos` routes; chunks migrated by balancer; shard key immutable after collection creation.
- **DynamoDB** — transparent sharding; partitions split/merge automatically; hot key problem remains.
- **Tenant isolation** — enterprise tier on dedicated shard; free tier on shared shard pool.
- **Schema-as-bucket (PostgreSQL)** — each customer group in its own schema on a shared instance; see [5.29](#529-sharding-bucketing--partitioning).
- **Read replicas per shard** — scale reads without multiplying write shards.

**Resharding strategies:**

| Strategy | Downtime | Complexity | When |
|----------|----------|------------|------|
| Double capacity + split | Low | Medium | Planned growth |
| Consistent hash add node | Low | High | Elastic cache/NoSQL |
| Directory update | Config window | Low | Small clusters |
| Re-key application | High | Very high | Wrong key chosen — avoid |

### When to use

- Dataset or write throughput exceeds one node after vertical scaling.
- Access patterns are **mostly single-key or single-shard** (`user_id`, `tenant_id`).
- Team can operate N databases (monitoring, backups, failover per shard).
- Cross-shard transactions are rare or delegated to saga/outbox (6.20).

### Trade-offs / Pitfalls

| Pitfall | Symptom | Mitigation |
|---------|---------|------------|
| Celebrity / hot shard key | One shard at 100% CPU; others idle | Salt key (5.6); dedicated shard; cache hot reads |
| Wrong shard key (`status`, `country`) | Scatter-gather on every query | Re-key at design time; `tenant_id` / `user_id` |
| Cross-shard ACID | 2PC p99 > 500 ms; partial commit risk | Saga, outbox, or single-shard invariant |
| Global secondary index | Write amp × shard count | Denormalize; local index; async search index |
| Schema drift across shards | Migration 31/32 applied | Orchestrated migration with version gate per shard |
| Monotonic shard key (`AUTO_INCREMENT`) | All inserts hit last shard | Hash or snowflake IDs |
| Resharding under load | Dual-write bugs; lost rows | Feature-flagged cutover; checksum per shard |
| Join across shards in OLTP | Multi-second queries | Denormalize; CQRS read model; scatter-gather async |

#### Production rules

| Rule | Why |
|------|-----|
| **Shard key must be in every hot-path query** | Missing key → broadcast to all shards |
| **Design for single-shard transactions** | Cross-shard 2PC is last resort, not default |
| **Automate schema migration across all shards** | Manual per-shard DDL → version drift incidents |
| **Per-shard monitoring, not cluster average** | Hot shard hidden in aggregate CPU metrics |
| **Plan resharding at 60% shard capacity** | Resharding at 95% runs under production load |
| **Global uniqueness via app-generated ID** | `AUTO_INCREMENT` and cross-shard `UNIQUE` break |
| **Backup and restore drill per shard** | 32 shards = 32 independent failure domains |
| **Document shard map in control plane** | Hard-coded `% 4` in app code blocks elastic growth |

#### Production monitoring

| Signal | Source | Alert when |
|--------|--------|------------|
| Per-shard QPS / CPU skew | Vitess `vttablet`, MongoDB `shardingStatus` | One shard > 2× cluster average |
| Cross-shard query rate | Proxy/router metrics | OLTP scatter-gather increasing → schema smell |
| Chunk / balance lag | MongoDB balancer, Citus rebalance | Migrations stalled > 24h |
| Replication lag per shard | Per-shard `replay_lag` | Lag > SLO before promoting replica |
| Connection pool per shard | App metrics by `shard_id` | One pool exhausted while others idle |
| Row count drift after resharding | Checksum job per shard | Mismatch → halt cutover |

### References

- Vitess architecture docs; MongoDB sharding guide
- Designing Data-Intensive Applications, Ch. 6

---


## 5.3 Hash Partitioning


### What is it?

**Hash partitioning** assigns rows to partitions using `hash(partition_key) mod N` (or similar). Keys hash to a fixed bucket count, distributing rows pseudo-randomly.

### Why it matters

Delivers even spread when keys are uniformly distributed - ideal for point lookups without range locality requirements.

### How it works

1. Client supplies partition key.
2. System computes hash -> partition index.
3. Router directs request to the node hosting that partition.
4. Range queries must query all partitions (scatter-gather).

### Diagram

```mermaid
flowchart LR
    Key[user_id=1234] --> Hash[hash fn]
    Hash --> Mod[mod 4]
    Mod --> P2[Partition 2]
```

### Key details

| Pro | Con |
|-----|-----|
| Even distribution | No range-scan locality |
| Simple routing | Resizing N remaps most keys |
| Fast point lookups | Hot keys still hot (same hash bucket) |

### When to use

- Equality lookups on a high-cardinality key.
- No need for range queries on the partition key.
- Stable partition count or consistent hashing for elasticity.

### Trade-offs / Pitfalls

- Adding shards with naive `mod N` requires massive data movement - use consistent hashing instead.
- Skewed key distribution (e.g., celebrity users) still creates hot partitions.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.4 Range Partitioning


### What is it?

**Range partitioning** assigns contiguous key ranges to partitions - e.g., A - M on shard 1, N - Z on shard 2, or time buckets for events.

### Why it matters

Enables efficient range scans, time-series ingestion patterns, and ordered iteration - common in analytics and event stores.

### How it works

1. Define ordered key space (string, timestamp, composite).
2. Assign ranges to partitions; metadata tracks boundaries.
3. Point and range queries hit only relevant partitions.
4. Split partitions when a range grows too large (dynamic splitting).

### Diagram

```mermaid
flowchart LR
    Keys[Key Space] --> R1[2024-01 -> Shard 1]
    Keys --> R2[2024-02 -> Shard 2]
    Keys --> R3[2024-03 -> Shard 3]
```

### Key details

- Excellent for `WHERE ts BETWEEN ...` and prefix scans.
- Risk of hot spots on latest time range or popular prefixes.
- Split/merge operations rebalance without full rehash.

#### Rolling partitions (retention at scale)

For append-heavy tables (events, logs, audit), **do not let one partition grow forever**. Use a **rolling window**:

```text
Before: P1  P2  P3  P4  P5
After:       P2  P3  P4  P5  P6   (drop P1, add P6)
```

| Operation | Cost | When to use |
|-----------|------|-------------|
| `DELETE FROM events WHERE id < X` | Slow; vacuum/bloat; index churn | Small tables only |
| `DROP TABLE events_p1` | Fast metadata operation | Retention at millions+ rows |

**Safe drop workflow:**

1. Identify unprocessed rows in the partition to be dropped.
2. Move them to the latest (active) partition if still needed.
3. `DROP TABLE` the old partition — instant reclaim of storage.

**Capacity planning for spikes (2× rule):**

During **reindex**, **replay**, or **bulk import**, live data may temporarily **double** (old + new copy). Size partitions and reserves for **2× active data**, not just steady state.

**Reserve partitions:**

Keep at least **2 empty partitions** ready so burst inserts never fail waiting for partition creation.

```text
total_partitions ≈ (2 × active_partitions) + reserve
Example: 10 active → 20 + 2 reserve = 22 partitions provisioned
```

> **Capacity ≠ reserve** — `space_left` on a shard tracks headroom; reserve partitions are empty tables waiting for traffic.

Create partitions via **background scheduler**, not synchronously in the API hot path.

### When to use

- Time-series, logs, and ordered event data.
- Range-heavy queries on the partition key.
- Keys have natural ordering you want to preserve locally.

### Trade-offs / Pitfalls

| Pitfall | Symptom | Mitigation |
|---------|---------|------------|
| Hot latest partition | All writes hit one shard/range | Pre-split; hash sub-key + range composite |
| Partition creation in API path | p99 spikes; lock contention on catalog | Background scheduler only |
| No reserve partitions | `INSERT` fails during traffic spike | Keep 2+ empty partitions ahead |
| `DROP` without row migration | Silent data loss | Move unprocessed rows to active partition first |
| Uneven range sizes | One partition 10× larger than peers | Split range; rebalance boundaries |
| Retention via `DELETE` | Vacuum bloat; hours-long locks | `DROP TABLE` child partition |
| Reindex without 2× headroom | Disk full mid-migration | Size for double active data during replay |
| Range queries without partition key | Full scan all partitions | Include partition key in `WHERE` |

#### Production rules

| Rule | Why |
|------|-----|
| **Create partitions in scheduler, never synchronously in API** | DDL locks catalog; blocks inserts under load |
| **Maintain 2+ empty reserve partitions** | Burst traffic cannot wait for partition creation |
| **Use `DROP PARTITION` for retention, not `DELETE`** | O(metadata) reclaim vs O(rows) vacuum storm |
| **Move in-flight rows before drop** | Pipelines still processing old partition → data loss |
| **Plan for 2× data during reindex/replay** | Old + new copy coexist temporarily |
| **Align range boundaries with query patterns** | Monthly ranges for monthly reports → partition pruning |
| **Monitor fill rate of active partition** | Create next partition at 70% capacity, not 100% |
| **Composite key for time-series** | `(hash(device_id), event_date)` spreads hot "today" writes |

#### Production monitoring

| Signal | Source | Alert when |
|--------|--------|------------|
| Active partition fill rate | Row count / max rows per partition | > 70% of planned capacity |
| Partition count vs reserve | Catalog query / scheduler logs | < 2 empty future partitions |
| Insert failures | App errors `no partition for row` | Any occurrence → emergency partition create |
| Per-partition scan size | `EXPLAIN` partition pruning | Query scanning all children → missing predicate |
| Drop job duration | Scheduler metrics | Drop blocked → rows still referenced |
| Disk per partition | Per-child table size | One child >> peers → split range |

### References

- PostgreSQL declarative partitioning docs; DDIA Ch. 6 (partitioning)
- See [5.29 Sharding, Bucketing & Partitioning](#529-sharding-bucketing--partitioning) for combined production pattern

---


## 5.5 Geo Partitioning


### What is it?

**Geo partitioning** (geo-sharding, data residency partitioning) places data in specific geographic regions based on tenant location, user country, or compliance rules.

### Why it matters

Required for GDPR, data sovereignty laws, and latency-sensitive global apps. Keeps personal data in-region and routes users to nearest replicas.

### How it works

1. Partition key includes region/tenant locale (e.g., `EU-tenant-42`).
2. Metadata pins partitions to datacenters in that region.
3. Global router sends requests to the correct regional cluster.
4. Cross-region reads/writes are explicit and often restricted.

### Diagram

```mermaid
flowchart TB
    UserEU[EU User] --> ClusterEU[(EU Cluster)]
    UserUS[US User] --> ClusterUS[(US Cluster)]
    ClusterEU -.->|restricted| ClusterUS
```

### Key details

- Compliance: data never leaves jurisdiction without legal basis.
- Latency: reads served locally; cross-region replication optional.
- Disaster recovery may require cross-region replicas with policy controls.

### When to use

- Multi-region products with residency requirements.
- Latency-sensitive regional user bases.
- Regulatory mandates on data location.

### Trade-offs / Pitfalls

- Global queries and reports require federated query or aggregation layer.
- Cross-region failover conflicts with residency - design DR per jurisdiction.
- Operational complexity: N regional deployments instead of one global cluster.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.6 Hot Partitions


### What is it?

A **hot partition** (hot spot) is a shard or partition receiving disproportionate read/write traffic - often from a skewed key (viral post, celebrity user, latest time bucket).

### Why it matters

One hot partition caps throughput at a single node's limit despite many shards - defeating horizontal scale and causing tail latency spikes.

### How it works

Detection and mitigation follow a loop:

1. Monitor per-partition QPS, CPU, and queue depth.
2. Identify skewed keys via metrics or tracing.
3. Apply mitigation: split partition, add salt to key, cache, or async write path.
4. Validate even spread after change.

### Diagram

```mermaid
flowchart TB
    Traffic[Traffic] --> Hot[Hot Partition]
    Traffic --> Cold1[Cold Partition]
    Traffic --> Cold2[Cold Partition]
    Hot --> Mitigate[Salt / Split / Cache]
```

### Key details

#### Symptoms

Hot partitions show up in metrics before user complaints — one partition diverges from cluster norms.

| Signal | What you see | Tools |
|--------|--------------|-------|
| **Per-partition QPS** | One partition 10–1000× others | DynamoDB `ConsumedReadCapacityUnits`, Cassandra `nodetool tablestats` |
| **Throttle errors** | `ProvisionedThroughputExceeded`, 429 on one key | CloudWatch, API metrics |
| **CPU / IOPS saturation** | Single node pegged while peers idle | Node dashboards |
| **Tail latency** | p99 spikes correlate with one shard | Distributed tracing, partition tags |
| **Replication lag** | One replica falls behind on hot key range | `ReplicationLag`, PostgreSQL `pg_stat_replication` |
| **Queue depth** | Kafka partition consumer lag on one partition | `kafka.consumer.lag` |

**Common hot-key causes:**

```text
Partition key = status          →  all "ACTIVE" rows on one shard
Partition key = trending_post_id →  viral content
Time-series key = current_date  →  all writes hit "today" range
Tenant key = mega_customer      →  one B2B tenant dominates
```

#### Fixes and salting

| Fix | How | Trade-off |
|-----|-----|-----------|
| **Key salting** | Append random suffix: `user_id#3` spread across N logical shards | Reads must fan-out to N keys and merge |
| **Write sharding** | Split hot logical key into `post:123:shard0..7` | Application aggregates on read |
| **Caching** | CDN / Redis in front of hot key | Helps reads; write-heavy celebs still hot |
| **Dedicated partition** | Isolate known hot tenant to own shard | Ops overhead; plan ahead |
| **Partition split** | DynamoDB auto-splits; Cassandra split range | Eventually helps; lag during split |
| **Async write path** | Queue writes for counters, likes | Eventual consistency on count |

**Salting pattern (writes):**

```text
# Before: all traffic → partition hash(celebrity_user_42)
partition_key = celebrity_user_42

# After: spread across 8 salted partitions
partition_key = celebrity_user_42#${random 0..7}
write to one random suffix

# Read total followers: query all 8 suffixes, sum in app
```

**Salting pattern (DynamoDB):**

```text
PK = HOT#${user_id % 10}   SK = user_id
→ 10 partitions absorb write load
GSI or scatter-gather read to reconstruct per-user view
```

**Prevention at schema time:** choose high-cardinality partition keys aligned with access (`user_id`, not `country`); for time-series use **hash prefix + timestamp** (`hash(device_id) + date`).

### When to use

Mitigate when p99 latency or throttle errors correlate with single partition metrics.

### Trade-offs / Pitfalls

- Salting breaks single-key locality - reads must fan out and merge.
- Caching hot data helps reads but not write-heavy hot keys.
- Prevention at schema design time beats reactive firefighting.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.7 Rebalancing


### What is it?

**Rebalancing** moves **partitions** (shards, hash slots, token ranges) between nodes to restore even load, disk utilization, and replication health after cluster resize, node failure, or traffic skew.

Without rebalancing, adding 3 new nodes leaves them **idle** while old nodes stay at 95% disk — you paid for capacity you cannot use.

### Why it matters

| Without rebalancing | With rebalancing |
|---------------------|------------------|
| New nodes sit empty | Load spreads across cluster |
| Hot node stays hot after split | Partitions migrate off saturated hosts |
| RF=3 but data only on 2 live nodes | Under-replicated ranges repaired |
| Manual ops firefight during growth | Planned background migration |

Rebalancing is **continuous operations** in Cassandra, Kafka, Redis Cluster, and Vitess — not a one-time migration project.

### How it works

**Trigger conditions:**

```text
1. Scale-out: added brokers/shards/nodes
2. Scale-in: removed node (decommission)
3. Skew: one node > 120% of average partition size or QPS
4. Failure: node died → re-replicate to restore replication factor
5. Planned: token-aware repair before peak season
```

**Generic migration phases:**

```mermaid
sequenceDiagram
    participant Meta as Routing metadata
    participant Src as Source node
    participant Dst as Target node
    participant Client as Clients
    Note over Meta: Mark partition P as MOVING
    Src->>Dst: Stream/copy partition data
    Note over Src,Dst: Dual-read period (both may serve)
    Meta->>Meta: Atomically update token map
    Client->>Dst: New writes/read ownership
    Src->>Src: Drop local copy of P
```

1. **Select partitions to move** — largest, hottest, or random under consistent hash.
2. **Copy data** — streaming replication (Cassandra `nodetool move`, Kafka partition reassignment).
3. **Dual-write / catch-up** — target catches up to latest offset/version.
4. **Atomic routing flip** — metadata service updates partition → node map.
5. **Drain source** — remove old copy after verification.

**Platform-specific patterns:**

| System | Rebalance mechanism | Client impact |
|--------|---------------------|---------------|
| **Cassandra** | `nodetool repair` + automatic token range movement on add/remove | Brief extra read latency during stream |
| **Kafka** | `kafka-reassign-partitions.sh` / Cruise Control | Consumer rebalance + possible lag spike |
| **Redis Cluster** | `redis-cli --cluster reshard` | `MOVED`/`ASK` redirects storm |
| **MongoDB** | Balancer migrates chunks between shards | Chunk migration locks brief |
| **Vitess** | VReplication / resharding workflows | Dual-read cutover with traffic switch |
| **Postgres (Citus)** | Rebalance shards across workers | Background move + metadata update |

**Vitess / application-level resharding (conceptual):**

```text
Phase 1: Dual-write to old + new shard map (feature flag)
Phase 2: Backfill historical rows to new shards (batch job)
Phase 3: Verify row counts + checksums per shard
Phase 4: Flip read traffic to new routing table
Phase 5: Stop writes to old shards; decommission
```

### Key details

#### Production rules

| Rule | Why |
|------|-----|
| **Throttle migration bandwidth** | Full-speed copy saturates disk/network → user-facing latency |
| **Never rebalance only in API path** | Long-running partition moves belong in background jobs |
| **Use locks / leader election** | Two operators running resharding scripts → corrupt routing |
| **Verify replication factor after** | Each partition must have RF healthy copies |
| **Schedule during low traffic** | Kafka reassignment during peak → consumer lag incident |
| **Monitor lag throughout** | `replication_lag`, `under_replicated_partitions`, `streaming_state` |

#### Sizing and duration

```text
Partition size = 50 GB, network = 500 MB/s effective
Copy time ≈ 50 GB / 500 MB/s ≈ 100 seconds (ideal)
Reality: 2-10× longer due to compaction, concurrent traffic, checksums

Plan: start rebalance at 60% disk, not 95%
```

**Worked example — Kafka partition reassignment:**

```bash
# Generate reassignment JSON (move partitions off broker 3)
kafka-reassign-partitions.sh --bootstrap-server kafka:9092 \
  --reassignment-json-file plan.json --execute

# Monitor until complete (hours for TB-scale)
kafka-reassign-partitions.sh --bootstrap-server kafka:9092 \
  --reassignment-json-file plan.json --verify
```

During reassignment: consumers may see **leader election** per partition → brief pause.

#### Failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Split reads | Routing updated before copy complete | Roll back metadata; finish stream first |
| Lost writes during flip | Writes went to old owner after map change | Dual-write window + version checks |
| Rebalance never finishes | Throttle too low or node failing mid-stream | Check disk; increase concurrency carefully |
| Hotter after rebalance | Hash skew unchanged — moved wrong keyspace | Re-key or salt hot tenants (5.6) |
| `UNDER_REPLICATED` alert | Node down during rebalance | Restore failed node or replace |

### When to use

- After **adding/removing** nodes in sharded cluster
- When **disk or CPU skew** > 20% between nodes persists 24h+
- Post-failure to restore **replication factor**
- Before **decommissioning** hardware (drain node first)
- Pre-scale event: rebalance **before** traffic spike, not during

### Trade-offs / Pitfalls

- Large partition moves take **hours** — capacity plan early
- Rebalance + production traffic **compete for I/O** — throttle mandatory
- Incorrect routing update → **split brain reads** or lost writes
- Kafka rebalance triggers **consumer group rebalance** — double pain during deploy
- Consistent hashing minimizes moved keys on **±1 node** change; naive `% N` resharding moves ~everything
- **Do not confuse** Kafka broker rebalance with consumer group rebalance (6.7)

### References

- Kafka partition reassignment; Cassandra nodetool operations
- See [5.8 Consistent Hashing](#58-consistent-hashing), [5.29 Sharding, Bucketing & Partitioning](#529-sharding-bucketing--partitioning)

---


## 5.8 Consistent Hashing


### What is it?

**Consistent hashing** maps both **keys** (cache entries, user IDs, objects) and **nodes** (servers) onto a fixed **hash ring** (positions `0` to `2^32-1`). Each key is assigned to the **first node clockwise** from the key's position on the ring.

When a node is **added** or **removed**, only keys in the arc between that node and its predecessor move - not the entire key space. This minimizes data migration during cluster resize.

**Contrast with naive `hash(key) % N`:**
- When N changes from 3 to 4, **almost all keys** remap -> mass cache miss / data shuffle
- Consistent hashing: only ~`1/N` of keys move per node change

### Why it matters

- **Distributed caches** (Memcached, Redis Cluster) - add/remove nodes without invalidating entire cache
- **Dynamo-style databases** - partition data across nodes with minimal rebalance
- **Load balancers** (some L7/L4 designs) - stable backend selection
- **CDNs and P2P** (Chord, Kademlia) - routing in large peer networks

### How it works

1. Place each physical server at one or more points on the ring (hash of node ID)
2. Hash the key: `position = hash("user:123")`
3. Walk clockwise from `position` until you hit the first server -> that server owns the key
4. **Add node D:** only keys between D's predecessor and D migrate to D
5. **Remove node B:** keys owned by B migrate to B's successor on the ring

```mermaid
flowchart LR
    Ring[Hash Ring 0 to 2^32] --> N1[Node A]
    Ring --> N2[Node B]
    Ring --> N3[Node C]
    Key[K hash position] -->|clockwise first match| N2
```

**Virtual nodes (vnodes):**

One physical server maps to **many** points on the ring (e.g. 100-256 vnodes per host). Without vnodes, uneven arc sizes cause hot servers when node count is small.

Example: 3 physical nodes with 100 vnodes each = 300 ring points -> near-uniform distribution.

**Replication on the ring:**

For replication factor R=3, key owner is first clockwise node; replicas are 2nd and 3rd clockwise nodes.

**Worked example:**

Ring with nodes A, B, C. Key `K` hashes between B and C -> owned by C.  
Add node D between B and K -> only keys that were on C and fall between D and K move to D (~fraction `1/(N+1)` of keys).

### Key details

| Approach | Remap on node change | Load balance | Notes |
|----------|---------------------|--------------|-------|
| `hash % N` | ~all keys | Even if uniform hash | Simple; bad for elastic clusters |
| Consistent hash | ~`1/N` keys per change | Uneven without vnodes | Memcached Ketama |
| Consistent hash + vnodes | ~`1/N` keys | Good | Cassandra, DynamoDB |
| Jump consistent hash | Minimal | Good | No ring metadata; Google paper |

- **Hot keys:** consistent hashing spreads **keys** evenly but not **traffic** - `celebrity_user` still hammers one node; fix with key splitting or local cache
- **Client vs server routing:** client computes owner (Dynamo) or server redirects (Redis Cluster MOVED/ASK)
- **Metadata service:** ring membership must be consistent across clients; gossip or ZooKeeper/KRaft
- Libraries: **Ketama** (Memcached), **libketama**, **jump hash** (no virtual nodes needed)

### When to use

- Clusters that **grow and shrink** frequently (auto-scaling cache pools)
- Distributed storage where rebalancing terabytes is expensive
- Any system design interview involving "how do you shard data across servers?"

### Trade-offs / Pitfalls

- Few physical nodes without vnodes -> **uneven load** (one node owns 50% of ring)
- Hot keys are **not solved** by hashing alone
- Ring membership changes during migration need **handoff protocol** (read from old + new owner during move)
- Jump consistent hash avoids vnodes but limits flexibility for heterogeneous node capacities (weighted vnodes help)

### References

- Used in Dynamo (Amazon), Cassandra, Riak, Memcached client libraries

---


## 5.9 Virtual Nodes

> Covered under [5.8 Consistent Hashing](#58-consistent-hashing) — **virtual nodes (vnodes)** map each physical server to many points on the ring (e.g. 100–256 tokens per host) for even load when the cluster is small. Interview answer: "more vnodes = smoother balance, more metadata."

---


## 5.10 Rendezvous Hashing


### What is it?

**Rendezvous hashing** (highest random weight hashing) assigns each key to the node with the highest score from `hash(node, key)` among all nodes.

### Why it matters

Minimal remapping on node add/remove (only keys that preferred the new/changed node move), with simpler logic than ring maintenance for some deployments.

### How it works

1. For key K, compute weight `W(N,K)` for every node N.
2. Select node with maximum W.
3. On node add: only keys where new node wins move to it.
4. On node remove: keys on removed node re-pick max among survivors.

### Diagram

```mermaid
flowchart TB
    Key[K] --> W1[score A]
    Key --> W2[score B]
    Key --> W3[score C]
    W2 --> Winner[Node B wins]
```

### Key details

- O(nodes) per lookup - fine for tens of nodes, costly at thousands.
- Excellent distribution properties without virtual nodes.
- Used in CDN routing and some load balancers.

### When to use

- Moderate node counts with frequent membership changes.
- When you want uniform spread without ring complexity.
- Client-side sharding with simple implementation.

### Trade-offs / Pitfalls

- Does not scale to huge node sets - cache scores or use hierarchical hashing.
- Still subject to hot key problems at application level.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.11 Replication


### What is it?

**Replication** maintains multiple **copies** of the same data on different nodes for **durability** (survive disk/node loss), **availability** (serve reads during failure), and **latency** (read from nearest replica). Each partition typically has a **replication factor (RF)** — commonly 3 in production.

Replication is orthogonal to partitioning: every partition has its own replica set.

```mermaid
flowchart TB
    subgraph Partition P
        L[Leader / Primary]
        F1[Follower 1]
        F2[Follower 2]
    end
    Write[Write] --> L
    L -->|replicate| F1
    L -->|replicate| F2
    Read[Read] --> L
    Read --> F1
    Read --> F2
```

### Why it matters

Single-copy data dies with the disk. Without replication:

- Node failure = **data loss** and **downtime**
- Maintenance windows require full outage
- Reads cannot scale horizontally off primaries
- Geographic users pay WAN latency on every read

Replication is non-negotiable for production data you cannot afford to lose.

### How it works

**Three replication topologies:**

| Topology | Write path | Read path | Conflict handling |
|----------|------------|-----------|-------------------|
| **Leader-follower** | Single leader orders writes | Leader or stale follower | None (single writer) |
| **Multi-leader** | Multiple leaders accept local writes | Nearest leader | LWW, version vectors, CRDTs |
| **Leaderless** | Quorum write to any N replicas | Quorum read from R replicas | Version merge on read |

**Replication sync spectrum:**

```mermaid
flowchart LR
    subgraph Sync['Synchronous (strong)']
        W1[Write] --> L1[Leader]
        L1 --> F1[Follower ACK]
        F1 --> ACK[Client ACK]
    end
    subgraph Async['Asynchronous (fast)']
        W2[Write] --> L2[Leader]
        L2 --> ACK2[Client ACK]
        L2 -.->|background| F2[Follower]
    end
```

1. **Choose topology** — match consistency needs and write locality (5.12, 5.13).
2. **On write** — leader or coordinator replicates to RF nodes per sync policy.
3. **On read** — fetch from leader, nearest replica, or quorum (5.14, 5.15).
4. **On divergence** — anti-entropy repair, read repair, or consensus log replay.

**Worked example — RF=3, N=3:**

- Data written to nodes A, B, C
- A fails → reads/writes continue from B, C if quorum allows
- A returns → catch-up via hinted handoff or full rebuild

### Key details

| Parameter | Typical value | Meaning |
|-----------|---------------|---------|
| RF | 3 | Tolerate 1 node loss with quorum |
| `min.insync.replicas` | 2 | Kafka: min replicas for `acks=all` |
| Sync replicas | All or quorum | Durability vs latency trade-off |
| Read preference | `nearest`, `primary` | MongoDB, Cassandra drivers |

**Production patterns:**

- **3 AZ deployment** — one replica per availability zone; survive full AZ loss with quorum.
- **Read replicas for analytics** — async follower serves reporting; lag monitored; never used for money reads without policy.
- **Chain replication** — leader → follower1 → follower2 reduces leader fan-out (some systems).
- **Geo-replication** — async cross-region for DR; sync only if product requires global strong consistency (Spanner).
- **Anti-entropy** — Cassandra `nodetool repair` fixes silent divergence between replicas.

**Replication vs backup:**

| | Replication | Backup |
|---|-------------|--------|
| Purpose | HA, read scale | Point-in-time recovery, human error |
| Lag | Seconds or less | Hours (snapshot schedule) |
| Corruption | Replicates bad writes | Snapshot may predate corruption |

### When to use

Always for production durable data. Defaults:

- **RF=3** for HA databases and Kafka topics
- **Leader-follower** when strong per-partition ordering required
- **Leaderless + quorum** when AP availability during partition matters (Dynamo family)
- **Multi-leader** only with explicit conflict resolution (geo writes)

### Trade-offs / Pitfalls

- **Sync replication** — latency = slowest replica RTT; cross-region sync hurts write p99.
- **Async replication** — promoted follower may **lag**; lost writes on failover if not monitored.
- **Stale reads** — reading followers without `read-your-writes` confuses users and breaks invariants.
- **Write amplification** — RF=3 means 3x disk writes; RF=5 rare except extreme durability needs.
- **Split brain** — two primaries without quorum fencing (5.20); always require majority for leadership.

### References

- Designing Data-Intensive Applications, Ch. 5 (Replication)
- Dynamo paper (leaderless replication)

---


## 5.12 Leader Follower Replication


### What is it?

**Leader-follower** (primary-replica) replication sends all writes through a single **leader** node, which replicates an ordered log to **follower** replicas. Reads may hit leader or followers depending on consistency needs.

### Why it matters

The dominant model for RDBMS (PostgreSQL, MySQL) and many distributed stores (Kafka partitions, MongoDB replica sets). Simple consistency story: one write ordering authority.

### How it works

1. Client sends write to leader.
2. Leader appends to local log and streams to followers.
3. Followers apply entries in order.
4. Leader acknowledges after sync to quorum or all followers.
5. Follower promotion on leader failure via election.

### Diagram

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    C->>L: WRITE x=1
    L->>F1: Replicate
    L->>F2: Replicate
    F1-->>L: ACK
    F2-->>L: ACK
    L-->>C: OK
```

### Key details

#### Synchronous vs asynchronous replication

| Mode | Ack when | Durability (RPO) | Latency | Partition behavior |
|------|----------|------------------|---------|-------------------|
| **Sync** | Quorum/all followers persist | **0** lost writes on leader death after ack | Highest (WAN RTT per replica) | CP: minority partition cannot commit |
| **Async** | Leader local write done | **> 0** — promoted replica may lack last writes | Lowest | AP: leader accepts writes; followers catch up |
| **Semi-sync** | At least 1 follower ack | Bounded loss (1 replica) | Middle | Common MySQL compromise |

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F as Follower
    Note over C,F: Synchronous
    C->>L: WRITE
    L->>F: replicate + fsync
    F-->>L: ACK
    L-->>C: OK

    Note over C,F: Asynchronous
    C->>L: WRITE
    L-->>C: OK
    L->>F: replicate (background)
```

**PostgreSQL examples:**

```text
synchronous_commit = on          → wait for sync standby (strong)
synchronous_commit = remote_write → wait for standby receive
synchronous_commit = off         → async (faster, riskier)
```

**MySQL:** `innodb_flush_log_at_trx_sync` + semi-sync plugin waits for one replica.

**Read consistency:**

| Read from | Consistency |
|-----------|-------------|
| Leader / primary | Latest committed (strong) |
| Async follower | **Stale** — replication lag seconds |
| Sync quorum follower | Strong if read-after-quorum |

Use `read-your-writes` session routing or `WAIT FOR REPLICA` after write when reading followers.

#### Failover

When the leader dies, a follower is **promoted** to accept writes.

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Old Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    participant M as Monitor
    L--xM: heartbeat lost
    M->>F1: promote (most caught-up)
    F1->>F1: accept writes as new leader
    M->>C: update connection / VIP / DNS
    C->>F1: WRITE resumes
```

| Step | Action |
|------|--------|
| 1. **Detect** | Missed heartbeats (3× interval), `/health` fails, Raft election timeout |
| 2. **Elect** | Raft majority vote; PostgreSQL `pg_promote()`; MongoDB replica set election |
| 3. **Fence** | STONITH old leader — revoke VIP, isolate network — prevent split-brain |
| 4. **Catch-up** | New leader at highest LSN; other replicas resync |
| 5. **Redirect** | VIP, DNS TTL, driver `primary` endpoint, K8s Service update |

**RPO / RTO by replication mode:**

| Mode | RPO on failover | Typical RTO |
|------|-----------------|-------------|
| Sync replication | 0 | Seconds (automated) |
| Async replication | Seconds–minutes of writes | Seconds–minutes |
| Manual promotion | Depends on lag at death | Minutes (human) |

**Split-brain prevention:** require **quorum** for promotion (`majority of N` replicas). Two-node clusters need **witness** or **arbiter** (MongoDB) in third AZ.

**Failover pitfalls:**

- Promoting async replica → **lost writes** not yet replicated
- Clients cache old primary IP → connection pool refresh needed
- Long failover → brief write unavailability (CP) or dual-write risk (misconfigured AP)
- **Logical replication** lag in PostgreSQL — verify `pg_last_wal_replay_lsn` before promote

- Follower reads may be stale unless read-from-leader or quorum read.

### When to use

- Strong ordering required per partition.
- Workloads tolerate single-leader write bottleneck or shard widely.
- Familiar ops model with clear failover story.

### Trade-offs / Pitfalls

- Leader is a write bottleneck and failover sensitivity point.
- Async lag causes "split brain" risk if auto-promote without quorum.
- Cross-datacenter leader-follower adds WAN latency to every commit.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.13 Multi Leader Replication


### What is it?

**Multi-leader** (multi-master) replication allows multiple nodes to accept writes, synchronizing changes between leaders - often one leader per region.

### Why it matters

Enables write-locality for geo-distributed apps: users write to the nearest datacenter without cross-WAN round trips on every operation.

### How it works

1. Each region has a leader accepting local writes.
2. Leaders exchange changes via replication log or conflict-free structures.
3. Conflicts detected when same row updated concurrently at two leaders.
4. Resolution via LWW, version vectors, or application merge logic.

### Diagram

```mermaid
flowchart LR
    L1[Leader EU] <-->|sync| L2[Leader US]
    L1 --> F1[Followers EU]
    L2 --> F2[Followers US]
```

### Key details

- Common in CouchDB, some MySQL geo clusters, and mobile sync (offline writes).
- Conflict-free Replicated Data Types (CRDTs) avoid conflicts by design for specific data types.
- Not suitable when application assumes single global order.

### When to use

- Multi-region write locality is mandatory.
- Conflicts are rare or mergeable (counters, sets, CRDTs).
- Brief inconsistency across regions is acceptable.

### Trade-offs / Pitfalls

- Write-write conflicts are inevitable - must have resolution strategy.
- Debugging "which write won" is harder than single-leader.
- Global uniqueness constraints (auto-increment IDs) break without coordination.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.14 Quorum Reads


### What is it?

A **quorum read** contacts multiple replicas and returns a value only after **R** of **N** replicas respond, selecting the **highest version** among responses. Combined with write quorums (5.15), quorums provide **tunable consistency** without a single leader for reads.

Foundation of **Dynamo-style** (AP) systems: Cassandra, Riak, DynamoDB (with `ConsistentRead`), Voldemort.

### Why it matters

Leaderless architectures must answer: *"Which replica has the latest value?"* Quorum reads:

- Survive replica unavailability (read from any R alive nodes)
- Detect stale replicas via version metadata
- Enable **read repair** — fix divergent replicas on the read path
- Tune latency vs freshness per query (`ONE` vs `QUORUM` vs `ALL`)

### How it works

**Parameters:**

- **N** — replication factor (total replicas)
- **R** — read quorum size (replicas contacted for read)
- **W** — write quorum size (see 5.15)

```mermaid
sequenceDiagram
    participant C as Client
    participant Co as Coordinator
    participant N1 as Replica 1
    participant N2 as Replica 2
    participant N3 as Replica 3
    C->>Co: READ key=K
    Co->>N1: read K
    Co->>N2: read K
    Co->>N3: read K
    N1-->>Co: value=v3, version=3
    N2-->>Co: value=v3, version=3
    N3-->>Co: timeout
    Note over Co: R=2 met; max version=3
    Co-->>C: return v3
    Co->>N3: async read repair v3
```

**Step-by-step:**

1. Coordinator (often any node, or token owner) receives read.
2. Sends parallel read to **preferred replicas** (often local + replicas in same DC).
3. Waits for **R** responses with `(value, version)` tuples.
4. Returns value with **highest version** (version vector or timestamp).
5. If replicas disagree and `R + W > N`, returned value is guaranteed to include latest quorum write.
6. **Read repair** — async write of newest value to stale replicas.

**Consistency levels (Cassandra example):**

| Level | R | Behavior |
|-------|---|----------|
| `ONE` | 1 | Fastest; may return stale data |
| `LOCAL_ONE` | 1 in local DC | Low latency multi-DC |
| `QUORUM` | majority of N | Balanced |
| `LOCAL_QUORUM` | majority in local DC | Avoid cross-DC on every read |
| `ALL` | N | Slowest; fails if any replica down |

### Key details

**The overlap guarantee (`R + W > N`):**

For N=3, W=2, R=2: any read quorum of 2 nodes **must overlap** any write quorum of 2 nodes by at least one replica — that replica holds the latest write.

```
N=3 replicas: {A, B, C}
Write quorum W=2: {A,B} or {A,C} or {B,C}
Read quorum R=2: must share at least 1 node with any write set
→ reader sees latest committed quorum write
```

| Config | Consistency | Availability on partition |
|--------|-------------|---------------------------|
| R=1, W=1 | Eventual | High |
| R=2, W=2, N=3 | Strong quorum | Minority partition cannot R or W |
| R=3, W=3 | Linearizable-ish | Low — any down node blocks |

**Sloppy quorum / hinted handoff:**

When preferred replicas are down, coordinator may write/read from **fallback** nodes outside the natural replica set, returning hints for when primary returns. Improves availability but **weakens** strict quorum guarantees — document when enabled.

**Production patterns:**

- **LOCAL_QUORUM** in multi-DC Cassandra — avoid cross-WAN on every read.
- **DynamoDB `ConsistentRead=true`** — reads from leader replica; eventually consistent is default (cheaper).
- **Monitor read repair rate** — high repair rate signals replication lag or node issues.
- **Speculative retry** — if R responses slow, issue extra read to different replica (latency vs cost).

### When to use

- Leaderless systems (Cassandra, Scylla, Riak).
- Need read availability when some replicas are down.
- Tunable consistency per query — analytics `ONE`, billing `QUORUM`.
- Pair with idempotent writes and version columns for conflict detection.

### Trade-offs / Pitfalls

- **`R=1` reads** — fast but may return **stale** or conflicting siblings; never for financial balances without application merge.
- **Latency cost** — QUORUM waits for slowest of R replicas; tail latency matters.
- **`R + W ≤ N`** — no overlap guarantee; you have **eventual consistency** only — know your math.
- **Sloppy quorum** — availability win that breaks strict consistency claims in minority partitions.
- **Not linearizable by default** — need `R=W=majority` + careful conflict handling; compare to Raft leader reads.

### References

- Dynamo paper (quorum replication)
- Cassandra consistency level documentation

---


## 5.15 Quorum Writes


### What is it?

A **quorum write** acknowledges a write only after **W** of **N** replicas persist it. Together with read quorum **R**, the pair `(W, R, N)` defines the **consistency/latency/availability envelope** — the core of **tunable consistency** in Dynamo-family databases.

**The fundamental rule:** when `R + W > N`, read and write quorums **overlap**, so a quorum read is guaranteed to see the latest quorum write (for a single register, absent concurrent writes).

### Why it matters

Leaderless systems have no single authority to serialize writes. Quorum writes let operators **dial consistency**:

- `W=1` — optimize write latency; risk loss if that node dies before async replicate
- `W=QUORUM` — durable commit with minority node failure tolerance
- `W=N` — all replicas must ACK; strongest durability; fails on any replica outage

Interview must-know: **write the math**, explain overlap, and state when concurrent writes need version resolution.

### How it works

```mermaid
sequenceDiagram
    participant C as Client
    participant Co as Coordinator
    participant A as Replica A
    participant B as Replica B
    participant D as Replica C
    C->>Co: WRITE key=K, value=v4, version=4
    Co->>A: replicate v4
    Co->>B: replicate v4
    Co->>D: replicate v4
    A-->>Co: ACK
    B-->>Co: ACK
    Note over Co: W=2 met -> success
    Co-->>C: OK
    D-->>Co: ACK (async catch-up)
```

**Write path:**

1. Coordinator receives write with new version (timestamp, UUID, or counter).
2. Computes replica set from partition placement (token ring).
3. Sends write to all N replicas (or preferred set).
4. Waits for **W** acknowledgments.
5. Returns success to client; remaining replicas catch up via **hinted handoff** or **read repair**.
6. Concurrent writes to same key with `W < N` may create **siblings** — resolved on read via LWW or application logic.

### Key details

**W + R > N math (N=3):**

| W | R | W+R | Overlap? | Typical use |
|---|---|-----|----------|-------------|
| 1 | 1 | 2 | No | Fast writes + fast reads; eventual |
| 2 | 2 | 4 | **Yes** | Balanced quorum (most common) |
| 3 | 1 | 4 | Yes | Strong writes, fast stale reads |
| 1 | 3 | 4 | Yes | Fast writes, strong reads |
| 3 | 3 | 6 | Yes | Strongest; any down node blocks |

**Visual overlap (N=3, W=2, R=2):**

```text
Write to {A,B}:     ●●○  or  ●○●  or  ○●●
Read from {B,C}:    ○●●  or  ●●○  etc.
Any write {A,B} and read {B,C} share at least B → latest write visible
```

**Tunable consistency presets (Cassandra):**

| Use case | CL write | CL read | Notes |
|----------|----------|---------|-------|
| Analytics counter | `ONE` | `ONE` | Loss acceptable |
| User profile | `QUORUM` | `QUORUM` | Default balanced |
| Financial ledger | `ALL` | `ALL` | Or use LWT / external lock |
| Multi-DC | `LOCAL_QUORUM` | `LOCAL_QUORUM` | Per-DC quorum |

**Concurrent write problem:**

When two clients write with `W=2` concurrently:

```text
Client 1: write v1 to {A,B}  ✓
Client 2: write v2 to {B,C}  ✓
Both succeed! No single latest value without version merge on read.
```

Mitigations: **lightweight transactions (LWT / Paxos)** in Cassandra, **conditional writes** in DynamoDB, or design for commutative updates.

**Production patterns:**

- **LOCAL_QUORUM + LOCAL_QUORUM** — standard multi-DC Cassandra production default.
- **`min.insync.replicas=2` + `acks=all`** — Kafka's quorum write analog.
- **Hinted handoff** — write to temporary node when preferred replica down; ship hint when it returns.
- **Monitor `W` latency p99** — slow replica dominates quorum write tail.
- **Avoid `W=1` for money** — node death between ACK and replicate = silent loss.

### When to use

- Dynamo-family databases with per-query consistency levels.
- Need writes to continue when minority replicas unavailable (AP during partition).
- Willing to handle sibling conflicts for availability.
- Pair with `R + W > N` when reads must see latest quorum write.

### Trade-offs / Pitfalls

- **`W=1`** — fastest write; **data loss** if acknowledging node fails before background replicate.
- **`W + R ≤ N`** — no overlap; claiming "strong consistency" is wrong.
- **Concurrent writes** — quorum does not serialize writers; need LWT, locks, or CRDTs.
- **Sloppy quorum** — writes to non-replica nodes during outage weaken guarantees.
- **Not a distributed transaction** — quorum is per-key; multi-key atomicity needs separate mechanism (5.16).
- **Latency** — `W=ALL` blocked by slowest or dead replica.

### References

- Dynamo paper (W, R, N tunable consistency)
- Cassandra consistency levels and lightweight transactions

---


## 5.16 Distributed Transactions


### What is it?

A **distributed transaction** spans multiple nodes or shards, requiring atomic commit or abort across all participants - preserving ACID across partition boundaries.

### Why it matters

Business invariants often cross shards (debit one account, credit another). Without distributed transactions, applications must implement compensating logic manually.

### How it works

Common patterns:

1. **Two-phase commit (2PC):** coordinator prepares all, then commits all.
2. **Saga:** sequence of local transactions with compensations.
3. **Percolator / Calvin:** layered timestamps or ordered locking across cells.
4. **Spanner:** TrueTime + Paxos per shard + external consistency.

### Diagram

```mermaid
flowchart TB
    App --> TC[Transaction Coordinator]
    TC --> S1[Shard 1]
    TC --> S2[Shard 2]
    S1 --> Decision{All prepared?}
    S2 --> Decision
    Decision -->|yes| Commit
    Decision -->|no| Abort
```

### Key details

- Cross-shard joins in one SQL statement often imply distributed transaction underneath.
- Latency = slowest participant + coordination round trips.
- Many systems avoid them - design bounded contexts per shard instead.

### When to use

- Financial transfers and inventory where atomicity is non-negotiable.
- Systems built for it: Spanner, CockroachDB, TiDB.
- Low-frequency cross-shard ops, not bulk ETL.

### Trade-offs / Pitfalls

- 2PC blocks on coordinator or participant failure (in doubt state).
- Throughput ceiling far below single-shard transactions.
- Prefer sagas or single-shard design when eventual consistency is acceptable.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.17 Two Phase Commit


### What is it?

**Two-phase commit (2PC)** is a distributed **atomic commit protocol**. A **coordinator** (transaction manager) runs:

1. **Phase 1 — Prepare:** ask all participants to vote; each locks resources and writes undo log
2. **Phase 2 — Commit or Abort:** if all vote YES, send COMMIT; otherwise ABORT

Goal: all participants commit or all abort — **atomicity** across nodes. Used in **XA transactions** (JDBC across two databases), some distributed SQL engines, and as the conceptual baseline for why modern systems prefer sagas and consensus.

### Why it matters

The textbook answer for cross-node atomicity — and a **cautionary tale** for blocking, coordinator failure, and partition intolerance. Interviewers expect you to draw both phases, explain **in-doubt** state, and contrast with **3PC** (5.18), **Saga** (6.x), and **Raft-backed** transactions (CockroachDB, Spanner).

### How it works

```mermaid
sequenceDiagram
    participant TC as Coordinator
    participant P1 as "Participant 1 (Shard A)"
    participant P2 as "Participant 2 (Shard B)"
    TC->>TC: write decision log (START)
    TC->>P1: PREPARE(txn_id)
    TC->>P2: PREPARE(txn_id)
    P1->>P1: validate, lock rows, write undo log
    P2->>P2: validate, lock rows, write undo log
    P1-->>TC: VOTE YES
    P2-->>TC: VOTE YES
    TC->>TC: log COMMIT decision
    TC->>P1: COMMIT(txn_id)
    TC->>P2: COMMIT(txn_id)
    P1->>P1: apply, release locks
    P2->>P2: apply, release locks
    P1-->>TC: ACK
    P2-->>TC: ACK
```

**Phase 1 — Prepare (voting):**

| Step | Coordinator | Participant |
|------|-------------|-------------|
| 1 | Persist `txn_id` in **transaction log** | — |
| 2 | Send `PREPARE(txn_id)` to all | Receive prepare |
| 3 | — | Validate constraints, acquire **locks** |
| 4 | — | Write **undo log** (for abort rollback) |
| 5 | — | Vote `YES` or `NO` (vote NO on any failure) |
| 6 | Collect votes | Hold locks until COMMIT/ABORT |

**Phase 2 — Commit/Abort (decision):**

| Outcome | Coordinator action | Participant action |
|---------|-------------------|-------------------|
| All YES | Log COMMIT; send `COMMIT` to all | Apply changes; release locks |
| Any NO | Log ABORT; send `ABORT` to all | Rollback via undo log; release locks |
| Coordinator crash after prepare | — | **Blocked** in prepared state |

**Abort path:**

```mermaid
sequenceDiagram
    participant TC as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2
    TC->>P1: PREPARE
    TC->>P2: PREPARE
    P1-->>TC: YES
    P2-->>TC: NO (constraint violation)
    TC->>TC: log ABORT
    TC->>P1: ABORT
    TC->>P2: ABORT
    P1->>P1: rollback undo log
```

### Key details

**Coordinator failure scenarios:**

| Failure point | Participant state | Recovery |
|---------------|-------------------|----------|
| Before prepare sent | No locks | Safe to abort locally |
| After some YES votes | Prepared, locks held | **In-doubt** until coordinator recovers |
| After COMMIT logged | Must commit | Replay coordinator log |
| Coordinator dead post-prepare | **Blocked indefinitely** | Manual intervention or timeout heuristic (unsafe) |

**The blocking problem:**

If coordinator crashes **after** participants voted YES but **before** sending COMMIT/ABORT:

- Participants hold **locks** and wait forever (or until coordinator recovers)
- Other transactions block on locked rows
- This is why 2PC is **not partition tolerant** and avoided on hot paths

**In-doubt transactions:**

```text
Participant log: PREPARED txn_42 — outcome unknown
Cannot unilaterally COMMIT (coordinator may have aborted)
Cannot unilaterally ABORT (coordinator may have committed)
→ must contact coordinator or heuristic rollback (dangerous)
```

**XA / JDBC example:**

```text
UserService DB (RM1) + Inventory DB (RM2) under JTA coordinator
BEGIN → prepare both → commit both
Any RM timeout → entire XA transaction rolls back
```

**Production patterns (where 2PC still appears):**

- **Infrequent admin operations** — cross-database migrations with human oversight
- **XA across two RDBMS** — legacy enterprise integration (declining in microservices)
- **Internal control planes** — coordinator is HA with durable log (ZooKeeper transaction, not user-facing)
- **NOT microservice hot paths** — use **outbox** (6.20) or **saga** instead

**2PC vs alternatives:**

| Protocol | Atomicity | Blocking | Partition tolerant | Throughput |
|----------|-----------|----------|-------------------|------------|
| 2PC | Yes | **Yes** | No | Low |
| 3PC | Yes | Reduced* | No* | Lower |
| Saga | Compensating | No | Yes | High |
| Raft + per-shard txn | Yes | No* | Majority | Medium |
| Outbox | Eventual | No | Yes | High |

*3PC assumes bounded delay; still fails under async network partitions.

### When to use

- Rare cross-database operations with **strong atomicity** and low frequency.
- HA coordinator with **durable transaction log** and recovery procedures.
- Internal tooling, not customer-facing checkout at 10k TPS.

### Trade-offs / Pitfalls

- **Coordinator SPOF** — must replicate coordinator log (Spanner uses Paxos for this layer).
- **Locks held through WAN** — prepare phase latency = sum of participant RTTs.
- **Minority partition cannot commit** — CP behavior during network split.
- **Heuristic rollback** — forcing abort on in-doubt txn risks inconsistency if coordinator actually committed.
- **Prefer saga/outbox** for microservices — 2PC couples availability of all participants.

### References

- Gray & Reuter: Transaction Processing (2PC specification)
- Designing Data-Intensive Applications, Ch. 9 (Distributed transactions)

---


## 5.18 Three Phase Commit


### What is it?

**Three-phase commit (3PC)** adds a **pre-commit** phase after prepare so participants know global decision before committing - reducing indefinite blocking if coordinator fails *after* pre-commit.

### Why it matters

Illustrates evolution beyond 2PC's blocking problem; rarely deployed in production but useful for understanding consensus trade-offs.

### How it works

1. **CanCommit:** coordinator asks if participants *can* commit (non-blocking probe).
2. **PreCommit:** if all agree, coordinator sends pre-commit; participants ready but not final.
3. **DoCommit:** coordinator sends commit; participants apply.
4. Timeout rules allow participants to commit if pre-commit received and coordinator silent.

### Diagram

```mermaid
sequenceDiagram
    participant TC as Coordinator
    participant P as Participant
    TC->>P: CanCommit?
    P-->>TC: Yes
    TC->>P: PreCommit
    TC->>P: DoCommit
    P-->>TC: ACK
```

### Key details

- Assumes network bounded delay (synchronous model) - fails under async networks with partitions.
- More round trips than 2PC -> higher latency.
- Largely superseded by Paxos/Raft for production coordination.

### When to use

- Academic / interview context more than greenfield systems.
- When comparing why modern systems chose consensus logs over 3PC.

### Trade-offs / Pitfalls

- Still vulnerable under network partition + timing assumptions.
- Operational complexity without wide library support.
- Real systems use Raft/Paxos + transaction layer instead.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.19 Distributed Locking


### What is it?

A **distributed lock** grants exclusive access to a resource across processes/nodes - implemented via consensus stores, databases with TTL leases, or dedicated services (Redis Redlock, ZooKeeper ephemeral nodes).

### Why it matters

Prevents duplicate work, enforces single-writer invariants, and coordinates leader election - but is a frequent source of outages when misused.

### How it works

1. Client acquires lock on key `/locks/resource` with TTL lease.
2. Perform critical section work.
3. Release lock (delete key) before TTL if healthy.
4. Fencing: stale lock holder must be blocked via monotonic token (fencing token) on storage writes.

### Diagram

```mermaid
sequenceDiagram
    participant A as Service A
    participant L as Lock Service
    participant DB as Database
    A->>L: acquire lock + token=5
    L-->>A: granted
    A->>DB: write (fencing token=5)
    Note over A: B with token=4 rejected
```

### Key details

- Lease TTL protects against dead holder - but work must finish before expiry.
- Redlock debate: clock skew and GC pauses can violate safety without fencing.
- Prefer idempotency and database constraints over locks when possible.

### When to use

- Short critical sections (cron leader, migration runner).
- Resources without natural compare-and-swap semantics.
- With fencing tokens on downstream writes.

### Trade-offs / Pitfalls

- Long-held locks -> availability killer on holder crash until TTL.
- Without fencing, delayed old holder can corrupt data after lease expires.
- Heavy lock contention -> redesign for optimistic concurrency.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.20 Split Brain


### What is it?

**Split brain** occurs when a cluster partitions into two groups that each believe they are the legitimate primary - both accepting writes and diverging irreconcilably.

### Why it matters

Classic catastrophic failure mode for databases and distributed locks; causes duplicate IDs, double spending, and data corruption.

### How it works

Typical scenario:

1. Network partition isolates leader from majority of replicas.
2. Minority side still serves if misconfigured; majority elects new leader.
3. Both sides accept writes during partition.
4. On heal, conflicting histories require manual merge or last-writer-wins data loss.

### Diagram

```mermaid
flowchart TB
    subgraph Partition A
        L1[Old Leader]
    end
    subgraph Partition B
        L2[New Leader]
    end
    L1 -.->|network cut| L2
```

### Key details

- Prevention: require **quorum** for leadership and writes (`majority of N`).
- STONITH: fence old leader (shutdown or revoke credentials).
- Witness nodes in third AZ break ties for 2-node clusters.

### When to use

Understanding split brain is prerequisite to designing HA - not something to "use," but to prevent.

### Trade-offs / Pitfalls

- Auto-failover without quorum checks causes split brain.
- Manual failover during partial outages often triggers it.
- "Brain peering" in misconfigured Redis/Mongo clusters is a real incident pattern.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.21 Consensus


### What is it?

**Consensus** is the problem of getting multiple nodes to agree on a single value or ordered log of values, despite crashes and network delays. Solved protocols guarantee safety (never two decisions) and liveness (eventually decides, with sufficient majority).

### Why it matters

Underpins leader election, replicated logs, and distributed configuration (etcd, ZooKeeper, Raft in CockroachDB/TiDB). Without consensus, replicas diverge with no automatic reconciliation.

### How it works

General pattern (replicated state machine):

1. Leader receives client command.
2. Leader appends to replicated log via consensus rounds.
3. Majority acknowledges persistence.
4. Leader applies committed entry to state machine.
5. Followers apply same entries in order.

### Diagram

```mermaid
flowchart LR
    C[Client] --> L[Leader]
    L --> Log[Replicated Log]
    Log --> SM[State Machine]
    F1[Follower] --> Log
    F2[Follower] --> Log
```

### Key details

- Requires majority (quorum) for fault tolerance: `2f+1` nodes tolerate `f` failures.
- FLP impossibility: no deterministic async consensus with one faulty process - protocols use timeouts/randomization.
- Consensus ≠ distributed transactions (but transactions can use consensus per shard).

### When to use

- Strongly consistent metadata, locks, and small critical state.
- Building or operating Raft/Paxos-backed systems.
- Whenever multiple nodes must share one authoritative ordered history.

### Trade-offs / Pitfalls

- Latency tied to WAN round trips if leaders and majorities span regions.
- Not for high-volume bulk data - only coordination metadata at scale.
- Mis-sized clusters (even counts without witness) reduce fault tolerance.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.22 Paxos

> **Interview:** Paxos is the classic **consensus** protocol (proposers, acceptors, learners, majority quorums). Correct but hard to implement. New systems usually use **[5.23 Raft](#523-raft)** — same idea, clearer leader-based structure. Mention Paxos when discussing Google Chubby, Spanner, or early ZooKeeper.

---


## 5.23 Raft


### What is it?

**Raft** is a **consensus algorithm** designed for understandability. It elects a **leader** per **term**, replicates an **ordered log** to followers, and commits entries once stored on a **majority**. Used in **etcd**, **Consul**, **CockroachDB**, **TiKV**, **NATS JetStream**, and countless control planes.

Raft solves the same problem as Paxos (5.22) — agreed ordered log despite crashes — with a clearer structure: **leader election**, **log replication**, **safety rules**.

### Why it matters

Default teaching and implementation choice for **strongly consistent replicated logs**. Understanding Raft explains:

- How Kubernetes stores cluster state (etcd)
- How distributed SQL shards agree on writes (CockroachDB)
- Why leader failure causes brief unavailability
- What **term** numbers fence stale leaders

### How it works

**Node states:**

```mermaid
stateDiagram-v2
    [*] --> Follower
    Follower --> Candidate: election timeout
    Candidate --> Leader: majority votes
    Candidate --> Follower: discover higher term
    Leader --> Follower: discover higher term
    Leader --> Follower: new leader elected
```

**1. Leader election**

1. Follower misses **heartbeat** from leader → election timeout fires (randomized 150–300ms typical).
2. Increments **term** T, becomes **candidate**, votes for self.
3. Sends `RequestVote(term=T)` to all peers.
4. Each node votes **at most once per term** for first valid candidate.
5. **Majority votes** → becomes **leader**; sends heartbeats to suppress elections.
6. **Split vote** (no majority) → new random timeout; retry term T+1.

```mermaid
sequenceDiagram
    participant F1 as Follower 1
    participant F2 as Follower 2
    participant F3 as Follower 3
    Note over F1,F3: Leader died
    F1->>F1: timeout -> Candidate term=5
    F1->>F2: RequestVote term=5
    F1->>F3: RequestVote term=5
    F2-->>F1: Grant vote
    F3-->>F1: Grant vote
    Note over F1: Majority -> Leader term=5
    F1->>F2: AppendEntries heartbeat
    F1->>F3: AppendEntries heartbeat
```

**2. Log replication**

1. Client sends command to **leader only**.
2. Leader appends entry `(term, index, command)` to local log.
3. Leader sends `AppendEntries` RPC to followers with **prevLogIndex/prevLogTerm** for consistency check.
4. Follower rejects if log mismatch; leader **decrements nextIndex** and retries (backtrack).
5. Follower appends matching entries, responds success.
6. Entry **committed** when replicated on **majority** (not merely sent).
7. Leader applies committed entries to **state machine**, responds to client.
8. Followers apply entries in order as they become committed.

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    C->>L: SET x=1
    L->>L: append index=10 term=3
    L->>F1: AppendEntries index=10
    L->>F2: AppendEntries index=10
    F1-->>L: success
    F2-->>L: success
    Note over L: majority=2/3 -> commit index 10
    L->>L: apply to state machine
    L-->>C: OK
```

**3. Terms — logical clocks for leadership**

| Concept | Purpose |
|---------|---------|
| **Term** | Monotonically increasing epoch number |
| **Higher term** | Always supersedes lower; stale leader ignored |
| **Vote per term** | At most one vote prevents split leadership |
| **Log entry term** | Identifies which leader created entry |

**Stale leader scenario:**

```text
Leader L (term 3) partitioned from majority
Majority elects L2 (term 4)
L recovers, sends AppendEntries term=3
Followers reject: "my term is 4" → L steps down to follower
```

**4. Commit rule (majority)**

Entry at index `i` is **committed** when:

- Stored on replicas in **majority**, AND
- Entry's term equals **current leader term** (Raft paper safety detail)

Leader then applies and notifies followers via subsequent `AppendEntries`.

**Cluster size math:**

| Nodes N | Majority | Tolerates failures |
|---------|----------|-------------------|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

Always use **odd** counts or add **witness** for tie-breaking.

### Key details

| Rule | Purpose |
|------|---------|
| **Election safety** | At most one leader per term |
| **Leader append-only** | Leader never overwrites own log |
| **Log matching** | Same index+term → identical prefix |
| **Leader completeness** | Committed entries present in all future leader logs |
| **State machine safety** | Same apply order on all nodes |

**Production patterns:**

- **etcd** — 3 or 5 nodes for K8s control plane; never even count without witness.
- **CockroachDB** — **Range** = Raft group per shard; many Raft groups per cluster.
- **Joint consensus** — membership changes use transitional config to avoid two majorities (node add/remove safely).
- **Snapshots** — compact log; new follower installs snapshot instead of replaying full history.
- **Pre-vote** (etcd enhancement) — prevent disrupted follower from spuriously incrementing term.
- **Read index / lease read** — leader serves linearizable reads without log entry (with caveats).

**Raft vs quorum (Dynamo):**

| | Raft | Quorum (W/R/N) |
|---|------|----------------|
| Ordering | Total order via leader | Per-key; concurrent writers |
| Consistency | Strong (linearizable writes) | Tunable |
| Write path | Leader only | Any coordinator |
| Failover | Election + brief unavailability | Continues with quorum |

### When to use

- New **coordination service** (config, locks, service discovery).
- **Per-shard replication** in distributed SQL (CockroachDB, TiDB/TiKV).
- Replacing ZooKeeper when simpler ops model desired.
- Any system needing **one authoritative ordered history** per partition.

### Trade-offs / Pitfalls

- **Leader write bottleneck** — scale writes by sharding into many Raft groups.
- **WAN latency** — cross-region Raft pays RTT on every commit; keep majority in one region.
- **Membership changes** — use joint consensus; naive add/remove risks two leaders.
- **Election storms** — tune `election_timeout`, use pre-vote; flaky network causes leadership flapping.
- **Even node counts** — 4 nodes tolerates only 1 failure (majority=3), same as 3 nodes but more cost.
- **Not for bulk data** — Raft coordinates metadata; data plane uses object storage or SSTables.

### References

- Raft paper: "In Search of an Understandable Consensus Algorithm" (Ongaro & Ousterhout)
- etcd Raft implementation docs; CockroachDB architecture

---


## 5.24 Leader Election


### What is it?

**Leader election** selects one node as coordinator for a term - using Raft votes, ZooKeeper sequential ephemeral nodes, or lease-based campaigns in Kubernetes.

### Why it matters

Avoids split brain by ensuring at most one active leader per epoch; required for single-writer replication and distributed task runners.

### How it works

**Raft election example:**

1. Follower misses heartbeats -> increments term, votes for self.
2. Requests votes from peers; each node votes at most once per term.
3. Candidate with majority becomes leader, sends heartbeats.
4. Split vote -> new random timeout, retry next term.

### Diagram

```mermaid
sequenceDiagram
    participant F1 as Follower 1
    participant F2 as Follower 2
    participant F3 as Follower 3
    F1->>F2: RequestVote term=5
    F1->>F3: RequestVote term=5
    F2-->>F1: Grant
    F3-->>F1: Grant
    Note over F1: Becomes Leader
```

### Key details

- Epoch/term numbers fence stale leaders automatically in Raft.
- Bully algorithm and ring election used in simpler LAN contexts.
- K8s Lease API: lightweight election for controllers.

### When to use

- HA services needing exactly one active worker (scheduler, stream processor).
- Raft/ZK-backed clusters during failover.
- Any system transitioning from follower to leader on failure.

### Trade-offs / Pitfalls

- Flapping leadership (thrashing) if timeouts too aggressive.
- Even-sized clusters need tie-breaker or witness for clean majority.
- Election storms during network glitches - tune backoff and quorum.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.25 Lamport Clocks


### What is it?

A **Lamport clock** is a logical timestamp: each node increments a counter on local events and sends max(local, received)+1 on message send - establishing **happens-before** ordering for causally related events.

### Why it matters

Physical clocks drift; Lamport clocks give consistent event ordering for debugging, replication metadata, and conflict comparison without synchronized NTP.

### How it works

1. Initialize counter L=0 on each process.
2. On local event: L := L+1, stamp event.
3. On send: L := L+1, attach L to message.
4. On receive: L := max(L, message.L)+1.

### Diagram

```mermaid
sequenceDiagram
    participant A
    participant B
    A->>A: L=1
    A->>B: msg L=2
    B->>B: L=3
```

### Key details

- If A -> B causally, then L(A) < L(B).
- Converse false: L(a) < L(b) does not imply causality.
- Cannot detect concurrent (unordered) events - use vector clocks.

### When to use

- Total ordering of events in single log merge.
- Backup conflict resolution when concurrency rare.
- Teaching foundation for vector clocks and version vectors.

### Trade-offs / Pitfalls

- Concurrent events get arbitrary order - may violate application semantics.
- Not sufficient for Dynamo-style conflict detection alone.
- Distributed tracing sometimes uses hybrid logical + physical clocks.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.26 Vector Clocks


### What is it?

A **vector clock** is a vector of counters - one per node - updated on local and receive events. Compare vectors to detect if events are ordered, concurrent, or equal.

### Why it matters

Enables **true concurrency detection** for multi-leader and leaderless stores - foundation of version vectors in Riak, Dynamo, and CRDT metadata.

### How it works

1. Each node maintains vector V of length N (nodes).
2. Local event: V[self]++.
3. Send message with V attached.
4. On receive: V[i] = max(V[i], msg.V[i]) for all i; then V[self]++.
5. Compare: V1 < V2 if all V1[i]≤V2[i] and strict; incomparable = concurrent.

### Diagram

```mermaid
flowchart LR
    E1["Event A: 1,0"] --> E2["Event B: 1,1"]
    E3["Event C: 0,1"] --> E2
    E1 -.concurrent.- E3
```

### Key details

- Version vectors are pragmatic finite-node variant with pruning.
- Concurrent writes require application merge or sibling versions.
- Size grows with replica count - compact with dotted version vectors.

### When to use

- Multi-master replication conflict detection.
- Causal consistency tracking across services.
- Building or debugging eventually consistent data stores.

### Trade-offs / Pitfalls

- Vector size and comparison cost scale with node count.
- Pruning old entries risks misclassifying concurrency.
- Application must handle sibling conflicts - storage won't guess semantics.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.27 Gossip Protocol


### What is it?

**Gossip** (epidemic) protocols spread information peer-to-peer by random node pairs exchanging state - each round doubling reach until all nodes converge.

### Why it matters

Scalable, decentralized membership and metadata dissemination without central coordinator - used in Cassandra, Consul, and failure detectors.

### How it works

1. Each node holds local state (membership, hash ring, health).
2. Every T seconds, pick random peer, exchange summaries.
3. Merge received state (newer timestamps win).
4. Repeat until cluster converges (typically O(log N) rounds).

### Diagram

```mermaid
flowchart LR
    A -->|gossip| B
    B -->|gossip| C
    C -->|gossip| D
    A -.->|eventually| D
```

### Key details

- Types: anti-entropy (state sync), dissemination (event broadcast), aggregation.
- Phi accrual failure detector often pairs with gossip membership.
- Bounded bandwidth per node regardless of cluster size.

### When to use

- Large clusters where centralized registry doesn't scale.
- Eventually consistent cluster view acceptable (seconds delay).
- Cassandra/Scylla ring propagation, Consul LAN gossip.

### Trade-offs / Pitfalls

- Convergence delay - not for sub-millisecond consistency needs.
- Split brain periods if partition + aggressive failure detection.
- Malicious or buggy nodes can spread false membership without auth.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.28 Membership Protocols


### What is it?

**Membership protocols** track which nodes are alive, joining, or leaving the cluster - via heartbeats, gossip, ZooKeeper ephemeral nodes, or SWIM (scalable weakly-consistent membership).

### Why it matters

Correct routing, replication, and consensus all depend on accurate membership. Wrong view -> writes to dead nodes, lost quorum, or split brain.

### How it works

**SWIM-style example:**

1. Nodes ping random peers periodically.
2. Indirect probe via third party on timeout.
3. Broadcast `suspect` then `confirm dead` with incarnation numbers.
4. New joins announce via seed nodes or gossip merge.

### Diagram

```mermaid
flowchart TB
    Join[New Node] --> Seed[Seed Nodes]
    Seed --> Gossip[Membership Gossip]
    Gossip --> View[Cluster View]
    Heartbeat[Heartbeats] --> View
```

### Key details

| Protocol | Characteristic |
|----------|----------------|
| Strong (ZK/etcd) | Accurate, centralized consensus |
| SWIM | Scalable, weakly consistent |
| Static config | Simple, poor elasticity |

### When to use

- Auto-scaling database clusters and service meshes.
- Failure detection integrated with load balancer backends.
- Choosing between strong membership (small control plane) vs gossip (large data plane).

### Trade-offs / Pitfalls

- False positive failure detection removes healthy nodes (flapping).
- Slow detection delays failover; fast detection increases false positives.
- Join storms during mass restart - use gradual rejoin and health gates.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 5.29 Sharding, Bucketing & Partitioning


### What is it?

A **three-layer data placement pattern** used in production OLTP systems (especially PostgreSQL) when a single database and single table cannot keep up:

| Layer | Purpose | Typical implementation |
|-------|---------|------------------------|
| **Sharding** | Spread load across machines | Multiple DB instances (PhDB) |
| **Bucketing** | Isolate tenant/customer groups | Schema per bucket inside a logical DB |
| **Partitioning** | Keep individual tables fast | Range (or hash) sub-tables inside each bucket |

**Mental model:**

```text
Customer → Bucket → Schema → LoDB (logical DB) → PhDB (physical DB)
```

```mermaid
flowchart TB
    subgraph PhDB1["PhDB-1 (PostgreSQL instance)"]
        LoDB1[installbase_db]
        LoDB1 --> B1[bucket_0001 schema]
        LoDB1 --> B2[bucket_0002 schema]
    end
    subgraph PhDB2["PhDB-2"]
        LoDB2[installbase_db]
        LoDB2 --> B3[bucket_0003 schema]
    end
    B1 --> P1[events_p1..pN partitions]
    CP[(Control plane / metadata)] -->|routes| B1
    CP --> B3
```

**Analogy:** PhDB = country · LoDB = state · Schema (bucket) = city · Customer = resident

### Why it matters

Without design, one table (`index_events`) growing 1M → 100M rows causes:

| Symptom | Root cause |
|---------|------------|
| Slow INSERT | Every insert updates larger indexes; more disk I/O |
| Slow SELECT | Large working set; even indexed queries degrade |
| Slow DELETE | Row-by-row delete; vacuum/bloat overhead |
| Painful maintenance | Reindex, vacuum lock or throttle production |
| Reindex / replay breaks SLA | Millions of writes in hours overload one node |

**Combined approach:**

- **Sharding** → horizontal **scale** (CPU, disk, connections)
- **Bucketing** → **isolation** (no cross-tenant blast radius)
- **Partitioning** → **performance** (smaller indexes, fast retention via DROP)

### How it works

#### Layer 1 — Physical DB (PhDB)

The actual PostgreSQL **instance or cluster** (CPU, memory, disk). Often **primary + replica** per instance for HA.

```text
PhDB-1 (Server A)    PhDB-2 (Server B)
```

#### Layer 2 — Logical DB (LoDB)

Application-facing database: `CREATE DATABASE installbase_db;` — connection string usually targets LoDB, not raw instance name.

#### Layer 3 — Bucket (= schema)

Each bucket is a **namespace** with identical table structure, holding a **disjoint subset of customers**:

```sql
CREATE SCHEMA bucket_0001;
-- same tables in every bucket: customer, events, ...
```

**Distribution example** (1M customers, ~100k per bucket):

```text
bucket_0001 → customers 1–100k      on PhDB-1
bucket_0002 → customers 100k–200k   on PhDB-1
bucket_0003 → customers 200k–300k   on PhDB-2
...
```

#### Layer 4 — Partitioning (inside bucket)

High-volume tables (events, logs) use **range partitioning** within the bucket:

```sql
CREATE TABLE events (
  id BIGINT,
  data TEXT
) PARTITION BY RANGE (id);

-- P1: 0–1M, P2: 1M–2M, P3: 2M–3M ...
```

PostgreSQL routes inserts to the correct child partition; queries with range predicates **prune** irrelevant partitions.

#### Shared-nothing rules

1. **All data for one customer lives in exactly one bucket** — never split a customer across buckets.
2. **No cross-bucket joins** in the hot path.

```sql
-- Allowed (single bucket)
SELECT * FROM bucket_0003.customer WHERE id = 'CUST_101';

-- Avoid (cross-bucket — slow, breaks scale)
-- JOIN bucket_0001.customer WITH bucket_0005.orders
```

Cross-shard queries need **scatter-gather**, **denormalization**, or an **analytics pipeline** — not OLTP SQL joins.

### Control plane (metadata)

A small **routing catalog** (often a separate lightweight DB) answers: *where is this customer?*

**`buckets` — where buckets live and capacity:**

```sql
CREATE TABLE buckets (
  bucket_id        INT PRIMARY KEY,
  db_instance_id   TEXT NOT NULL,      -- e.g. 'PhDB-2'
  space_left       INT NOT NULL DEFAULT 1000000,
  privileged       BOOLEAN NOT NULL DEFAULT FALSE,
  is_active        BOOLEAN NOT NULL DEFAULT TRUE,
  bucket_schema    TEXT               -- e.g. 'bucket_0003'
);
```

**`entity_buckets` — customer → bucket (most important routing table):**

```sql
CREATE TABLE entity_buckets (
  entity_id   TEXT PRIMARY KEY,
  bucket_id   INT REFERENCES buckets(bucket_id)
);
```

```text
CUST_101 → bucket_0002
CUST_202 → bucket_0005
```

Use a sequence (`seq_bucket`) for new bucket IDs. This is your **shard map** — equivalent to Vitess topology or DynamoDB partition metadata.

### Runtime request flows

#### Read flow

```mermaid
sequenceDiagram
    participant API
    participant Meta as Control plane
    participant DB as PhDB + LoDB
    API->>Meta: entity_id = CUST_101
    Meta->>Meta: SELECT bucket_id, db_instance_id
    API->>DB: CONNECT installbase_db
    API->>DB: SET search_path TO bucket_0002
    API->>DB: SELECT ... WHERE id = CUST_101
```

1. Lookup `entity_buckets` → `bucket_id`
2. Lookup `buckets` → `db_instance_id`, `bucket_schema`
3. Connect to target PhDB / LoDB
4. `SET search_path TO bucket_000X` (PostgreSQL)
5. Execute query

**Important:** If mapping **not found** on read → return empty / not found. **Do not** auto-assign a bucket on read.

#### Write flow (new customer)

1. Check `entity_buckets` — exists?
2. If **no** → assign bucket (see below), insert mapping
3. Connect + `SET search_path`
4. Insert / update business data
5. Decrement `space_left` on bucket (or recompute periodically)

#### Bucket assignment (even distribution)

```sql
SELECT bucket_id
FROM buckets
WHERE space_left > 0 AND privileged = FALSE AND is_active = TRUE
ORDER BY space_left DESC
LIMIT 1;
```

If none available → **create new bucket** (pick least-loaded PhDB, `CREATE SCHEMA`, run DDL, register in `buckets`).

Prefer buckets with the **most remaining capacity** (`ORDER BY space_left DESC`) to avoid hotspots.

### Bucket creation

1. Choose **least-loaded** PhDB (by bucket count or disk/CPU).
2. `nextval('seq_bucket')` → new `bucket_id`.
3. `CREATE SCHEMA bucket_XXXX;` + create all tables (migrations).
4. Insert row in `buckets`; ready for assignments.

Run DDL via **migration job**, not user-facing API.

### Partition lifecycle (inside bucket)

```mermaid
flowchart LR
    Inserts[Inserts to active partition] --> Monitor[Monitor sequence / size]
    Monitor --> Create[Scheduler creates next partition]
    Monitor --> Drop[Drop oldest after data move]
    Drop --> Move[Move unprocessed rows to latest]
```

1. Inserts land in current range partition.
2. Monitor ID sequence or row count.
3. **Background scheduler** adds future partitions (keep **2+ empty reserve**).
4. Before **DROP** old partition → move any unprocessed rows to latest partition.
5. `DROP TABLE` old partition — O(metadata), not O(rows).

**Full reindex / replay scenario:**

1. Read master data
2. Bulk insert into `events` (may hit **2×** row count during migration)
3. Process / index
4. Drop old partitions when cutover complete

**Why partition + DROP beats DELETE:**

```text
DELETE millions of rows → vacuum storm, index bloat, long locks
DROP partition           → instant, predictable
```

**Partition count formula:**

```text
total_partitions ≈ (2 × active_partitions) + reserve_partitions
Example: 10 active → 20 + 2 = 22 provisioned
```

Plan for **2× active data** during reindex, replay, or bulk import.

### Combined stack (final picture)

```text
PhDB-2
└── installbase_db (LoDB)
    ├── bucket_0003 (schema)
    │   ├── customer
    │   └── events → events_p1, events_p2, ... (range partitions)
    └── bucket_0004
        └── ...
```

| Concern | Layer that solves it |
|---------|---------------------|
| Machine limits | Sharding (PhDB) |
| Tenant isolation | Bucketing (schema) |
| Table size / retention | Partitioning (range + DROP) |
| Routing | Control plane (`entity_buckets`) |

### Key details

| Practice | Reason |
|----------|--------|
| Partition creation in **scheduler** | Avoid API latency spikes and lock contention |
| **Locks** around bucket creation / partition DDL | Prevent duplicate bucket IDs or half-created schemas |
| Monitor `space_left`, partition fill rate, vacuum | Early warning before insert failures |
| **Privileged** buckets | Dedicated enterprise tenants — excluded from shared pool assignment |
| `search_path` per request | Wrong schema = wrong tenant data (critical security bug) |

### When to use

- Millions of customers or entities with **per-entity locality** (all rows for customer X together)
- Continuous writes + heavy deletes (retention) + periodic **full reindex**
- PostgreSQL (or similar) where **schema-per-tenant** is acceptable vs database-per-tenant cost
- Team can operate **control plane** + automated bucket/partition jobs

### Trade-offs / Pitfalls

| Pitfall | Consequence | Mitigation |
|---------|-------------|------------|
| Missing `search_path` | Cross-tenant data leak or wrong reads | Set schema in connection/session middleware |
| Uneven bucket fill | One PhDB overloaded | `space_left` + least-loaded PhDB on create |
| No reserve partitions | Insert fails during spike | Pre-create 2+ empty partitions |
| Large partitions never dropped | Storage and query decay | Rolling window + DROP not DELETE |
| Cross-bucket joins in API | Latency, distributed tx pain | Denormalize; async aggregation |
| Auto-assign bucket on read | Phantom tenants, routing corruption | Assign only on explicit create/write |
| Partition DDL in API | Timeouts under load | Background scheduler only |
| Ignoring 2× during reindex | Disk full mid-migration | Size for double active set |

### Final takeaway

```text
Sharding      → SCALE      (many machines)
Bucketing     → ISOLATION  (customer groups)
Partitioning  → PERFORMANCE (manageable tables, fast retention)
```

Together: data is **distributed**, customers are **isolated**, and tables stay **operationally small**.

#### Shard vs partition (one-line mental model)

| Term | Where it lives | Purpose |
|------|----------------|---------|
| **Shard** | Different DB instance / machine | Horizontal scale across servers |
| **Partition** | Inside one DB instance | Query speed + easy retention (DROP) |

```text
Machine 1 → Shard 1 → orders_2024_01_0 .. orders_2024_01_7
Machine 2 → Shard 2 → orders_2024_01_0 .. orders_2024_01_7
```

They are **combined** at petabyte scale: shard for machine limits, partition for table size.

#### Worked sizing — 1,000 TB at ~1 KB/row

```text
Total data     = 1,000 TB
Target shard   = 2 TB per primary  → 500 primary shards
RF = 3         → 3,000 TB physical storage (1 PB logical × 3)

Node usable disk (10 TB disk - 2 OS - 1 reserve) = 7 TB
Nodes needed   = 3,000 / 7 ≈ 429 nodes (round up for headroom)
```

**Partition strategy per shard (avoid hotspots):**

```text
Level 1: monthly range partition (12/year)
Level 2: hash(user_id) % 8 sub-buckets
→ 12 × 8 = 96 partitions per year per shard
```

#### Worked sizing — 100 billion rows, 100 shards

```text
Rows per shard = 100B / 100 = 1B rows/shard

Monthly only:     1B / 12 ≈ 83M rows/month  (risk: hot month)
Monthly + hash×8: 1B / 96 ≈ 10.4M rows/partition  (safer)
```

**Disk per server sanity check:**

```text
Total disk 10 TB - OS 2 TB - logs/backup 1 TB = 7 TB usable
Plan shards so primary data + indexes + 2× headroom fits usable disk
```

### References

- PostgreSQL declarative partitioning; `search_path` and schemas
- Vitess / Citus (alternative sharding routers)
- Designing Data-Intensive Applications, Ch. 6

---

[<- Back to master index](../README.md)
