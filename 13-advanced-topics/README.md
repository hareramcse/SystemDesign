# 13. Advanced Topics

> Status: **Done** — concise notes for all sub-topics below.

[← Back to master index](../README.md)

---

## At a glance

```mermaid
flowchart LR
    Data[Large Dataset] --> Prob[Probabilistic Structures]
    Prob --> IDs[Distributed IDs]
    IDs --> DHT[DHT / Routing]
```

---

## Sub-topics

| # | Sub-topic |
|---|-----------|
| 13.1 | [Bloom Filters](#131-bloom-filters) |
| 13.2 | [HyperLogLog](#132-hyperloglog) |
| 13.3 | [Count Min Sketch](#133-count-min-sketch) |
| 13.4 | [Merkle Trees](#134-merkle-trees) |
| 13.5 | [Skip Lists](#135-skip-lists) |
| 13.6 | [Trie](#136-trie) |
| 13.7 | [Distributed Hash Tables](#137-distributed-hash-tables) |
| 13.8 | [UUID](#138-uuid) |
| 13.9 | [Snowflake IDs](#139-snowflake-ids) |
| 13.10 | [ULID](#1310-ulid) |
| 13.11 | [KSUID](#1311-ksuid) |

---

## 13.1 Bloom Filters

**Summary:** Space-efficient probabilistic set — answers "possibly in set" or "definitely not in set"; no false negatives.

- Fixed memory; cannot delete entries (use counting Bloom filter variant)
- Use for cache penetration checks, DB existence pre-filter, spell-check dictionaries
- Tune false-positive rate via bit array size and hash function count

**References:** _None yet._

---

## 13.2 HyperLogLog

**Summary:** Probabilistic cardinality estimator — counts distinct elements using ~12 KB with ~2% error.

- `PFADD` / `PFCOUNT` in Redis; used for unique visitors, unique IPs
- Mergeable across nodes — sum distinct counts from shards
- Exact count needs O(n) memory; HLL needs O(log log n)

**References:** _None yet._

---

## 13.3 Count Min Sketch

**Summary:** Probabilistic frequency sketch — estimates how often an item appears in a stream.

- Overestimates counts, never underestimates (with proper params)
- Fixed memory regardless of stream size — ideal for heavy hitters
- Used in network monitoring, trending topics, rate limiting

**References:** _None yet._

---

## 13.4 Merkle Trees

**Summary:** Hash tree where each parent is hash of children — efficient integrity verification of large datasets.

- Compare root hashes to detect any difference; pinpoint changed blocks via tree walk
- Used in Git, Cassandra anti-entropy, blockchain, certificate transparency
- O(log n) proof size for membership verification

**References:** _None yet._

---

## 13.5 Skip Lists

**Summary:** Probabilistic linked-list layers giving O(log n) search/insert — alternative to balanced trees.

- Redis sorted sets use skip lists internally
- Simpler to implement than red-black trees; concurrent-friendly variants exist
- Trade slightly more memory for simpler code paths

**References:** _None yet._

---

## 13.6 Trie

**Summary:** Prefix tree where each node represents a character — fast prefix search and autocomplete.

- O(key length) lookup regardless of dictionary size
- Used in autocomplete, IP routing tables, spell checkers
- Compressed tries (radix trees) reduce memory for sparse keys

**References:** _None yet._

---

## 13.7 Distributed Hash Tables

**Summary:** Decentralized key-value overlay where nodes route lookups via consistent hashing — no central index.

- Each node responsible for a key range on a hash ring
- Chord, Kademlia (BitTorrent, IPFS) are classic protocols
- Handles node join/leave with minimal key redistribution

**References:** _None yet._

---

## 13.8 UUID

**Summary:** 128-bit universally unique identifier — random (v4) or time-based (v1); no coordination needed.

- v4: random — safe for opaque IDs, poor for DB index locality
- v1: MAC + timestamp — sortable-ish but leaks host info
- Standard format: `550e8400-e29b-41d4-a716-446655440000`

**References:** _None yet._

---

## 13.9 Snowflake IDs

**Summary:** Twitter's 64-bit time-sortable ID — timestamp + machine ID + sequence per millisecond.

- Roughly increasing — good for B-tree index locality
- Requires unique machine/worker ID assignment per generator
- ~4096 IDs/ms per machine; clock skew must be handled

**References:** _None yet._

---

## 13.10 ULID

**Summary:** Universally Unique Lexicographically Sortable Identifier — 48-bit timestamp + 80-bit randomness, Crockford Base32.

- Sortable by creation time; URL-safe string (26 chars)
- No coordination needed unlike Snowflake machine IDs
- Popular in modern APIs and event logs

**References:** _None yet._

---

## 13.11 KSUID

**Summary:** K-Sortable Unique ID — 32-bit timestamp + 128-bit payload; sortable, 20-byte binary or Base62 string.

- Created by Segment; similar goals to ULID with different encoding
- Sortable for time-range queries without separate created_at index
- Payload space allows embedding custom bits if needed

**References:** _None yet._

---

## Quick Reference

| Sub-topic | Problem solved | Trade-off |
|-----------|----------------|-----------|
| **Bloom Filter** | "Might exist?" fast | False positives |
| **HyperLogLog** | Count distinct cheaply | ~2% error |
| **Count Min Sketch** | Stream frequency | Overcount possible |
| **Merkle Tree** | Verify data integrity | Rebuild on change |
| **Skip List** | Sorted set ops | Extra pointers |
| **Trie** | Prefix / autocomplete | Memory for dense alphabets |
| **DHT** | Decentralized lookup | Complexity on churn |
| **UUID** | Unique opaque ID | v4 not sortable |
| **Snowflake** | Sortable distributed ID | Needs machine ID |
| **ULID / KSUID** | Sortable, no coordination | Clock dependency |

---

[← Back to master index](../README.md)
