# Production Depth Roadmap

How we deepen the System Design repo **incrementally** — without rewriting all 293 sub-topics.

[← Back to master index](./README.md)

---

## Three tiers

| Tier | Label | Goal | When to use |
|------|-------|------|-------------|
| **P1** | Production-deep | Runtime flows, control plane, sizing rules, real pitfalls | You operate this in prod or debug it on call |
| **P2** | Standard | Current template (What / Why / How / Pitfalls) + light enrichment | Interview + design literacy |
| **P3** | Reference | Concise definition; links to P1/P2 neighbors | Theory, history, rare edge cases |

**P1 sections add (when you have notes or scars):**

1. **Problem at scale** — what breaks without this design
2. **Runtime flow** — numbered steps + diagram
3. **Control plane / routing** — metadata, config, who decides what
4. **Production rules** — explicit do / don't
5. **Sizing** — rules of thumb (2× capacity, reserve buffers, QPS math)
6. **Pitfalls table** — symptom → cause → fix

---

## Status legend

| Status | Meaning |
|--------|---------|
| ✅ P1 done | Production-deep content added |
| 🔄 P1 partial | Strong standard doc; needs prod layer |
| 📋 P1 queued | Tier 1; not yet deepened |
| P2 | Keep standard format |
| P3 | Keep concise |

---

## Phase plan (recommended order)

| Phase | Chapter | Focus | P1 count |
|-------|---------|-------|----------|
| **0** | Done | Sharding/bucketing, TLS, TCP, caching layers | 4 |
| **1** | [05 Distributed DB](./05-distributed-databases/README.md) | Hot partitions, rebalancing, quorums | 6 |
| **2** | [03 Caching](./03-caching/README.md) | Invalidation, stampede, penetration, patterns | 8 |
| **3** | [06 Messaging](./06-messaging-and-events/README.md) | Kafka ops, delivery, outbox, DLQ, replay | 9 |
| **4** | [08 Microservices](./08-microservices/README.md) | Saga, circuit breaker, discovery failures | 5 |
| **5** | [12 Reliability](./12-reliability-engineering/README.md) | RPO/RTO, DR runbooks, backup/restore | 6 |
| **6** | [09 Observability](./09-observability/README.md) | SLOs, alerting, tracing incidents | 5 |
| **7** | [11 Cloud/K8s](./11-cloud-and-kubernetes/README.md) | Deployments, HPA, multi-region, StatefulSets | 6 |

Add your own production notes to the matching P1 row as you go.

---

## P1 — Production-deep topics (52)

### Chapter 1 — Networking

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 1.3 | TCP Handshake | ✅ P1 done | TIME_WAIT, SYN flood, latency budget |
| 1.8 | DNS | 🔄 P1 partial | TTL migration, failover records |
| 1.9 | DNS Resolution | 🔄 P1 partial | Cache layers, happy eyeballs incidents |
| 1.11 | SSL/TLS | ✅ P1 done | Cert lifecycle, session keys, mTLS gap |
| 1.19 | CDN | 🔄 P1 partial | Push/pull, purge failures |
| 1.20 | Load Balancer | 🔄 P1 partial | Health check flapping, drain |
| 1.21 | LB Algorithms | 🔄 P1 partial | Already strong; add NAT/mobile pitfalls |
| 1.22 | SSE / WebSockets | 📋 P1 queued | Sticky sessions, pub/sub backplane |

*Remaining 1.x → P2*

---

### Chapter 2 — Databases

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 2.2 | Indexing | 📋 P1 queued | Missing index incidents, write amplification |
| 2.6 | Isolation Levels | 📋 P1 queued | Phantom reads in prod, ORM defaults |
| 2.11 | Vacuum Process | 📋 P1 queued | Bloat, autovacuum tuning, lock contention |
| 2.22 | SQL Tuning | 🔄 P1 partial | EXPLAIN on call, N+1 in APIs |

*Remaining 2.x → P2 (internals) or P3 (store types)*

---

### Chapter 3 — Caching

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 3.1 | Cache Fundamentals | 🔄 P1 partial | Layer stack added; add monitoring |
| 3.2 | Cache Aside | ✅ P1 done | Invalidation races, read path |
| 3.7 | Distributed Cache | ✅ P1 done | Redis cluster failover, hot keys |
| 3.9 | Cache Invalidation | ✅ P1 done | Stale data bugs, TTL vs event |
| 3.10 | Cache Stampede | ✅ P1 done | Single-flight, lock timeouts |
| 3.11 | Cache Avalanche | ✅ P1 done | TTL jitter at deploy |
| 3.12 | Cache Penetration | ✅ P1 done | Bloom filter, negative cache |
| 3.14 | Write Around | 📋 P1 queued | When writes skip cache |

*Remaining 3.x → P2*

---

### Chapter 4 — Distributed System

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 4.11 | Failover | 📋 P1 queued | Split-brain, flapping, cold standby |
| 4.20 | Backpressure | 📋 P1 queued | Queue depth, 503 vs slow death |
| 4.22 | Capacity Planning | 📋 P1 queued | Headroom, peak multipliers |
| 4.23 | Bottleneck Analysis | 📋 P1 queued | USE method, on-call workflow |

*CAP, PACELC, consistency models → P2 (theory with interview framing)*

---

### Chapter 5 — Distributed Databases

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 5.2 | Sharding | 🔄 P1 partial | Cross-link 5.29; scatter-gather costs |
| 5.4 | Range Partitioning | 🔄 P1 partial | Rolling partitions, DROP vs DELETE |
| 5.6 | Hot Partitions | ✅ P1 done | Celebrity key, salting |
| 5.7 | Rebalancing | ✅ P1 done | Dual-write, cutover runbook |
| 5.14 | Quorum Reads | ✅ P1 done | W+R>N math in incidents |
| 5.15 | Quorum Writes | ✅ P1 done | CL tuning Cassandra/Dynamo |
| 5.29 | Sharding + Bucketing + Partitioning | ✅ P1 done | Full production pattern |

*Paxos, Lamport, 3PC → P3*

---

### Chapter 6 — Messaging & Events

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 6.5 | Kafka | ✅ P1 done | Broker failure, ISR, lag |
| 6.7 | Consumer Groups | ✅ P1 done | Rebalance storms, stuck consumers |
| 6.13 | At Least Once | ✅ P1 done | Duplicate handling idempotency |
| 6.14 | Exactly Once | ✅ P1 done | Transactions vs outbox reality |
| 6.15 | Dead Letter Queue | ✅ P1 done | DLQ replay runbook |
| 6.16 | Retry Queue | ✅ P1 done | Exponential backoff, poison pills |
| 6.20 | Outbox Pattern | ✅ P1 done | Dual-write fix |
| 6.21 | Event Replay | ✅ P1 done | Reindex, ordering, 2× load |

*RabbitMQ, ActiveMQ, Pulsar → P2 unless you use them daily*

---

### Chapter 7 — API Design

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 7.5 | API Gateway | 📋 P1 queued | Timeout cascades, auth at edge |
| 7.18 | Rate Limiting | 📋 P1 queued | Token bucket prod tuning |
| 7.20 | Idempotency | 📋 P1 queued | Retry after timeout |
| 7.21 | Idempotency Keys | 📋 P1 queued | Storage, TTL, duplicate POST |

*SOAP, Swagger → P3*

---

### Chapter 8 — Microservices

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 8.3 | Microservices | 📋 P1 queued | When split hurts, ops tax |
| 8.13 | Service Discovery | ✅ P1 done | Stale registry, cache |
| 8.16 | Circuit Breaker | ✅ P1 done | Half-open flapping, fallbacks |
| 8.17 | Retry Pattern | ✅ P1 done | Retry storms, jitter |
| 8.19 | Saga Pattern | ✅ P1 done | Compensation failures, dirty reads |

*Hexagonal, onion, DI → P2/P3*

---

### Chapter 9 — Observability

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 9.5 | Distributed Tracing | 📋 P1 queued | Sampling, broken traces |
| 9.8 | SLI | 📋 P1 queued | Picking the right SLI |
| 9.9 | SLO | ✅ P1 done | Error budget policy |
| 9.11 | Error Budgets | 📋 P1 queued | Freeze vs ship |
| 9.12 | Alerting | 📋 P1 queued | Alert fatigue, SLO burn |

*OpenTelemetry overview → P2*

---

### Chapter 10 — Security

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 10.3 | OAuth2 | 📋 P1 queued | Token leakage, refresh rotation |
| 10.5 | JWT | 📋 P1 queued | Revocation, clock skew |
| 10.12 | Secret Management | 📋 P1 queued | Rotation, leak response |
| 10.18 | DDoS Protection | 📋 P1 queued | Layered defense runbook |

*Clickjacking, CSRF → P2*

---

### Chapter 11 — Cloud & Kubernetes

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 11.7 | Multi Region | 📋 P1 queued | Active-active pitfalls |
| 11.12 | Autoscaling | 📋 P1 queued | Lag, scale-down safety |
| 11.22 | Deployments | 📋 P1 queued | Rolling vs blue/green failures |
| 11.25 | StatefulSets | 📋 P1 queued | PVC, ordered startup |
| 11.34 | HPA | 📋 P1 queued | Metric choice, thrashing |
| 11.35 | Cluster Autoscaler | 📋 P1 queued | Pending pods, cost |

*IaaS/PaaS/SaaS definitions → P3*

---

### Chapter 12 — Reliability Engineering

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 12.1 | High Availability | 📋 P1 queued | Nines math, SPOF audit |
| 12.4 | RPO | ✅ P1 done | Data loss acceptance |
| 12.5 | RTO | ✅ P1 done | Runbook timeboxes |
| 12.6 | Disaster Recovery | ✅ P1 done | Game day, failover drill |
| 12.7 | Backup Strategy | ✅ P1 done | Backup ≠ restore |
| 12.8 | Restore Strategy | ✅ P1 done | Tested restore timeline |

*Chaos engineering → P2 unless practiced regularly*

---

### Chapter 16 — Interview Guide

| ID | Topic | Status | Prod focus to add |
|----|-------|--------|-------------------|
| 16.1–16.7 | All | ✅ P1 done | Scenario-based; add more walkthroughs over time |

---

## P2 — Standard (keep template, light touch)

~200 sub-topics. Enrich only when:

- You hit a production incident tied to that topic
- Interview feedback says a section felt too thin

Examples: REST, GraphQL, Bloom filters, replication theory, HTTP/2, VPC basics, ranking algorithms.

---

## P3 — Reference (stay short)

~40 sub-topics. No production runbook unless promoted to P1.

Examples: OSI model, SOAP, Paxos formalism, Lamport clocks, Hadoop history, Anycast theory, multi-model DB overview.

---

## How to deepen one P1 topic (checklist)

Copy this into a PR or note before editing:

```markdown
- [ ] Problem at scale (what breaks on single node / no pattern)
- [ ] Runtime read flow
- [ ] Runtime write flow
- [ ] Control plane / routing (if applicable)
- [ ] Production do / don't rules
- [ ] Sizing rule of thumb
- [ ] Pitfalls: symptom | cause | fix (≥5 rows)
- [ ] Cross-links to related sub-topics
- [ ] Update status in this roadmap: 📋 → 🔄 → ✅
```

---

## Contributing your notes

When you have internal docs (like the sharding/bucketing guide):

1. Find the matching **P1** row in this file
2. Deepen that **one** sub-topic in the chapter README
3. Mark ✅ here — do not duplicate into a separate doc unless it's >500 lines

---

## Summary counts

| Tier | Approx. count | Action |
|------|---------------|--------|
| P1 Production-deep | **52** | Deepen in phases; **32 done**, **20 queued/partial** |
| P2 Standard | **~200** | Maintain; enrich on demand |
| P3 Reference | **~41** | Keep concise |

**Do not rewrite the repo.** Deepen P1 topics as you bring production notes or incident learnings.
