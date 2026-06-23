# System Design — Master Reference

A comprehensive, interview-ready knowledge base covering networking, databases, distributed systems, cloud infrastructure, and advanced platform patterns. Each topic folder is a **self-contained master reference** — read any `README.md` top-to-bottom without needing external context.

**Source spreadsheet:** [System Design Fundamentals.xlsx](./System%20Design%20Fundamentals.xlsx)

---

## Topics

| # | Topic | Sub-topics | Status |
|---|-------|------------|--------|
| 1 | [Networking](./01-networking/README.md) | 22 | Documented |
| 2 | [Databases](./02-databases/README.md) | 19 | Documented |
| 3 | [Caching](./03-caching/README.md) | 13 | Documented |
| 4 | [Distributed System](./04-distributed-system/README.md) | 23 | Documented |
| 5 | [Distributed Databases](./05-distributed-databases/README.md) | 28 | Documented |
| 6 | [Messaging & Events](./06-messaging-and-events/README.md) | 23 | Documented |
| 7 | [API Desing](./07-api-desing/README.md) | 21 | Documented |
| 8 | [MicroServices](./08-microservices/README.md) | 21 | Documented |
| 9 | [Observability](./09-observability/README.md) | 15 | Documented |
| 10 | [Security](./10-security/README.md) | 21 | Documented |
| 11 | [Cloud & Kubernetes](./11-cloud-and-kubernetes/README.md) | 35 | Documented |
| 12 | [Reliability Engineering](./12-reliability-engineering/README.md) | 10 | Documented |
| 13 | [Advanced Topics](./13-advanced-topics/README.md) | 11 | Documented |
| 14 | [Search Systems](./14-search-systems/README.md) | 9 | Documented |
| 15 | [Big Data](./15-big-data/README.md) | 10 | Documented |

**Total:** 15 topics · 281 sub-topics · all **Documented**

---

## How to use

### Structured learning path

Follow topics **1 → 15** for breadth-first system design fundamentals:

| Phase | Topics | Goal |
|-------|--------|------|
| **Foundation** | 1 Networking, 2 Databases, 3 Caching | Protocols, storage, and performance primitives |
| **Distributed core** | 4 Distributed System, 5 Distributed Databases, 6 Messaging | Scale, consistency, replication, events |
| **Application layer** | 7 API Desing, 8 MicroServices, 9 Observability, 10 Security | Building and operating production services |
| **Platform** | 11 Cloud & Kubernetes, 12 Reliability Engineering | Deployment, SRE, disaster recovery |
| **Specialized** | 13 Advanced Topics, 14 Search Systems, 15 Big Data | Probabilistic structures, search, data platforms |


### Topic sequencing

Within each topic README, sub-topics are **ordered for learning flow** (foundations -> related concepts -> specialized/advanced). Use the **Reading order** table at the top of each topic file to see how sections group together.
### Per sub-topic reference format

Every sub-topic in a topic `README.md` follows the same master structure:

1. **What is it** — concise definition
2. **Why it matters** — real-world and interview relevance
3. **How it works** — mechanics and flow (with mermaid diagrams where helpful)
4. **Key details** — implementation specifics, production examples
5. **When to use** — decision criteria
6. **Trade-offs** — pros vs cons table
7. **References** — links from the spreadsheet and external resources

### Interview prep workflow

1. Pick a topic area matching your target role (e.g., **5 + 6** for backend, **14 + 15** for data/search).
2. Read each sub-topic section; sketch the mermaid diagram from memory.
3. Practice explaining **trade-offs** aloud — interviewers weight decisions over definitions.
4. Cross-link concepts (e.g., consistent hashing in **5** and DHT in **13**, inverted index in **14** and search DBs in **2**).
5. Use the [spreadsheet](./System%20Design%20Fundamentals.xlsx) for video links and supplemental material per sub-topic.

### Self-contained topic READMEs

Each folder `README.md` is designed as a **standalone master reference**:

- No dependency on other folders for basic understanding.
- Sub-topic table at the top for navigation and progress tracking.
- Quick Reference section at the bottom of newer topics for last-minute revision.
- Status column tracks documentation completeness (**Documented** = interview-ready depth).

Jump directly to any topic for a specific interview area, or follow the phased path above for comprehensive coverage.

---

## Repository layout

```
SystemDesign/
├── README.md                          ← You are here (master index)
├── System Design Fundamentals.xlsx    ← Source links and tracking
├── _topics.json                       ← Machine-readable topic registry
├── 01-networking/README.md
├── 02-databases/README.md
├── …
└── 15-big-data/README.md
```

---

[All 15 topics documented — expand with personal notes, diagrams, and spreadsheet links as you study]
