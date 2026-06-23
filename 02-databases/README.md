# 2. Databases

> Status: **Documented**

[← Back to master index](../README.md)

---

## Sub-topics

| # | Sub-topic | Status |
|---|-----------|--------|
| 2.1 | [Isolation Levels](#isolation-levels) | Done |
| 2.2 | [Indexing](#indexing) | Done |
| 2.3 | [Normalization/Denormalizatio](#normalization-denormalizatio) | Done |
| 2.4 | [Views/ Materialized View](#views-materialized-view) | Done |
| 2.5 | [Query Planner/ optimizer](#query-planner-optimizer) | Done |
| 2.6 | [B Tree/B+ Tree](#b-tree-b-tree) | Done |
| 2.7 | [LSM Tree/SSTables/WAL](#lsm-tree-sstables-wal) | Done |
| 2.8 | [MVCC](#mvcc) | Done |
| 2.9 | [Page Cache](#page-cache) | Done |
| 2.10 | [Redo/undo/bin Logs](#redo-undo-bin-logs) | Done |
| 2.11 | [Vacuum Process](#vacuum-process) | Done |
| 2.12 | [Key Value Stores](#key-value-stores) | Done |
| 2.13 | [Document Databases](#document-databases) | Done |
| 2.14 | [Wide Column Databases](#wide-column-databases) | Done |
| 2.15 | [Graph Databases](#graph-databases) | Done |
| 2.16 | [Time Series Databases](#time-series-databases) | Done |
| 2.17 | [Search Databases](#search-databases) | Done |
| 2.18 | [Vector Databases](#vector-databases) | Done |
| 2.19 | [Multi Model Databases](#multi-model-databases) | Done |

```mermaid
flowchart LR
    Query --> Planner
    Planner --> Index
    Index --> Storage
```

---

## 2.1 Isolation Levels

**Summary:** Define how concurrent transactions interact—what anomalies (dirty read, phantom read) are permitted. Higher isolation reduces anomalies but increases locking and aborts.

**Key points:**
- Read Uncommitted → Serializable: increasing strictness, decreasing concurrency
- PostgreSQL default READ COMMITTED; SERIALIZABLE uses SSI for full isolation
- Choose lowest level that preserves correctness for your workload

**References:**
- [Video](https://www.youtube.com/watch?v=89YYHMYfymk)

---

## 2.2 Indexing

**Summary:** Auxiliary data structures that avoid full table scans by mapping lookup keys to row locations. The primary lever for read performance—and write overhead.

**Key points:**
- B-tree default for range/equality; hash for exact match only
- Composite indexes: leftmost prefix rule; column order matters
- Covering indexes eliminate table lookups; watch write amplification

---

## 2.3 Normalization/Denormalizatio

**Summary:** Normalization removes redundancy via table decomposition (1NF→3NF); denormalization duplicates data to speed reads. Trade storage integrity for query performance.

**Key points:**
- 3NF eliminates transitive dependencies; reduces update anomalies
- Denormalize for read-heavy dashboards, aggregations, or join-heavy paths
- Eventual consistency acceptable when denormalized copies lag source

---

## 2.4 Views/ Materialized View

**Summary:** Views are virtual queries; materialized views persist query results to disk. MVs trade freshness for precomputed read speed.

**Key points:**
- Regular views always reflect current data; no storage cost
- Materialized views need refresh strategy: on commit, schedule, or trigger
- Index materialized views for further query acceleration

**References:**
- [Video](https://www.youtube.com/watch?v=XI1rk1Uaf7U)

---

## 2.5 Query Planner/ optimizer

**Summary:** Cost-based optimizer chooses execution plan (join order, index scan vs seq scan) from statistics. Bad stats or stale plans cause catastrophic slow queries.

**Key points:**
- EXPLAIN ANALYZE reveals actual vs estimated row counts
- Statistics histograms drive cost estimates—run ANALYZE after bulk loads
- Index hints and plan freezing are last resorts; fix stats first

**References:**
- [Video](https://www.youtube.com/watch?v=fj9pK8isRA4)

---

## 2.6 B Tree/B+ Tree

**Summary:** Balanced tree index keeping data sorted; B+ trees store all values in leaves with linked sibling pointers. Default index structure for most RDBMS.

**Key points:**
- O(log n) lookups, range scans via leaf traversal
- B+ tree: internal nodes keys-only; leaves hold data + next pointer
- Page-sized nodes align with disk block I/O—typically 4–16 KB

```mermaid
flowchart LR
    Root --> Internal
    Internal --> Leaf1[Leaf Pages]
    Leaf1 --> Leaf2[Linked Leaves]
```

**References:**
- [Video](https://www.youtube.com/watch?v=aZjYr87r1b8)

---

## 2.7 LSM Tree/SSTables/WAL

**Summary:** Log-Structured Merge trees buffer writes in memory, flush immutable SSTables to disk, and periodically compact. Optimized for write-heavy workloads.

**Key points:**
- WAL ensures durability before memtable flush; crash recovery replays log
- SSTables are immutable sorted files; compaction merges and deduplicates
- Read path checks memtable → bloom filter → SSTable levels

**References:**
- [Video](https://www.youtube.com/watch?v=P2xtlLymqqI)
- [Article](https://medium.com/@dwivedi.ankit21/lsm-trees-the-go-to-data-structure-for-databases-search-engines-and-more-c3a48fa469d2)

---

## 2.8 MVCC

**Summary:** Multi-Version Concurrency Control lets readers see snapshot versions without blocking writers. Each transaction sees a consistent point-in-time view.

**Key points:**
- Writes create new row versions; old versions retained until vacuum
- No read locks needed—high read concurrency
- Transaction ID visibility rules determine which version is visible

**References:**
- [Video](https://www.youtube.com/watch?v=TBmDBw1IIoY)

---

## 2.9 Page Cache

**Summary:** In-memory buffer pool caching database pages (typically 8 KB). Avoids disk I/O on hot data—often the biggest performance factor.

**Key points:**
- LRU or clock eviction when cache exceeds `shared_buffers`/buffer pool size
- Cache hit ratio > 99% is target for OLTP; monitor buffer pool metrics
- OS page cache also caches files—double buffering can waste RAM

**References:**
- [Video](https://www.youtube.com/watch?v=syPEMXQwaYQ)

---

## 2.10 Redo/undo/bin Logs

**Summary:** WAL (redo log) records changes before applying to pages for crash recovery. Undo log rolls back uncommitted transactions; binlog enables replication.

**Key points:**
- Write-ahead logging: log first, then data page—durability guarantee
- Redo replays committed changes after crash; undo discards in-flight txns
- Binary log (MySQL) or WAL shipping (PostgreSQL) feeds replicas

**References:**
- [Video](https://www.youtube.com/watch?v=47LvbDGD4cc)

---

## 2.11 Vacuum Process

**Summary:** Reclaims dead tuple space from MVCC updates and prevents transaction ID wraparound. Critical maintenance for PostgreSQL-style engines.

**Key points:**
- UPDATE/DELETE leave dead rows until vacuum marks space reusable
- Autovacuum triggers on dead tuple threshold; tune for write-heavy tables
- Freeze old XIDs to prevent catastrophic wraparound shutdown

**References:**
- [Video](https://www.youtube.com/watch?v=fTl8-pnaJCE)

---

## 2.12 Key Value Stores

**Summary:** Simple get/put/delete by key—no joins, no schema. Extreme throughput and horizontal scale for caching, sessions, and feature flags.

**Key points:**
- Redis: in-memory with persistence options; rich data structures
- DynamoDB/Riak: distributed, partition-tolerant at scale
- No ad-hoc queries—design access patterns around key structure

**References:**
- [Video](https://www.youtube.com/watch?v=VfcRxtBKI54)

---

## 2.13 Document Databases

**Summary:** Store semi-structured JSON/BSON documents with flexible schema. Natural fit for content, catalogs, and rapid iteration without migrations.

**Key points:**
- MongoDB, CouchDB: document = row; nested objects avoid joins
- Index embedded fields; aggregation pipeline for analytics
- Schema validation optional—enforce at application or DB level

**References:**
- [Video](https://www.youtube.com/watch?v=cODCpXtPHbQ)

---

## 2.14 Wide Column Databases

**Summary:** Column-family storage optimized for sparse, wide rows and massive scale. Reads/writes target column groups, not full rows.

**Key points:**
- Cassandra, HBase: partition key determines node; clustering key sorts within
- Tunable consistency per query; eventual consistency default
- Excellent for time-series, IoT, and write-heavy append workloads

---

## 2.15 Graph Databases

**Summary:** Native storage for nodes and edges with index-free adjacency. Traversals (friends-of-friends, shortest path) are O(edges) not O(rows).

**Key points:**
- Neo4j, Neptune: Cypher/Gremlin query languages for pattern matching
- Relationship properties stored on edges—no expensive join tables
- Poor fit for bulk analytics; excels at deep link traversal

---

## 2.16 Time Series Databases

**Summary:** Optimized for timestamped metrics with high ingest and range queries. Built-in downsampling, retention policies, and compression.

**Key points:**
- InfluxDB, TimescaleDB, Prometheus: append-mostly, time-range scans
- Compression exploits temporal locality and value similarity
- Downsampling aggregates old data to save storage

---

## 2.17 Search Databases

**Summary:** Full-text and faceted search engines built on inverted indexes. Ranked relevance scoring beyond SQL `LIKE` capabilities.

**Key points:**
- Elasticsearch/OpenSearch: distributed inverted index with analyzers
- Tokenization, stemming, and scoring (BM25) drive relevance
- Near-real-time indexing; refresh interval affects search latency

---

## 2.18 Vector Databases

**Summary:** Store and query high-dimensional embeddings for similarity search. Powers semantic search, RAG, and recommendation systems.

**Key points:**
- Approximate nearest neighbor (HNSW, IVF) trades accuracy for speed
- Pinecone, Weaviate, pgvector: cosine/dot-product distance metrics
- Combine with metadata filters for hybrid search

---

## 2.19 Multi Model Databases

**Summary:** Single engine supporting document, graph, key-value, and relational models. Reduces operational overhead for polyglot persistence needs.

**Key points:**
- ArangoDB, Azure Cosmos DB: unified query across models
- Choose when team wants one ops stack for varied access patterns
- May not match best-of-breed performance per model

**References:**
- [Video](https://www.youtube.com/watch?v=hwYadL33HdI)

---

## Quick Reference

| # | Sub-topic | One-liner |
|---|-----------|-----------|
| 2.1 | Isolation Levels | Concurrency anomaly trade-offs |
| 2.2 | Indexing | Fast lookups at write cost |
| 2.3 | Normalization/Denormalizatio | Reduce redundancy vs speed reads |
| 2.4 | Views/ Materialized View | Virtual vs persisted query results |
| 2.5 | Query Planner/ optimizer | Cost-based execution plan selection |
| 2.6 | B Tree/B+ Tree | Sorted balanced tree index |
| 2.7 | LSM Tree/SSTables/WAL | Write-optimized log-structured storage |
| 2.8 | MVCC | Snapshot reads without blocking writes |
| 2.9 | Page Cache | In-memory buffer pool for hot pages |
| 2.10 | Redo/undo/bin Logs | WAL recovery + replication logs |
| 2.11 | Vacuum Process | Reclaim dead MVCC tuple space |
| 2.12 | Key Value Stores | Simple key→value at massive scale |
| 2.13 | Document Databases | Flexible JSON document storage |
| 2.14 | Wide Column Databases | Column-family for sparse wide rows |
| 2.15 | Graph Databases | Native node/edge traversal |
| 2.16 | Time Series Databases | Timestamped metrics at high ingest |
| 2.17 | Search Databases | Inverted index full-text search |
| 2.18 | Vector Databases | Embedding similarity search |
| 2.19 | Multi Model Databases | Multiple data models in one engine |

[← Back to master index](../README.md)
