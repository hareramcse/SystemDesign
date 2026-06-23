# 12. Reliability Engineering

> Status: **Done** — concise notes for all sub-topics below.

[← Back to master index](../README.md)

---

## At a glance

```mermaid
flowchart LR
    Monitor[Monitor] --> SLO[SLO / Error Budget]
    SLO --> DR[Disaster Recovery]
    DR --> HA[High Availability]
```

---

## Sub-topics

| # | Sub-topic |
|---|-----------|
| 12.1 | [Chaos Engineering](#121-chaos-engineering) |
| 12.2 | [Disaster Recovery](#122-disaster-recovery) |
| 12.3 | [Backup Strategy](#123-backup-strategy) |
| 12.4 | [Restore Strategy](#124-restore-strategy) |
| 12.5 | [RPO](#125-rpo) |
| 12.6 | [RTO](#126-rto) |
| 12.7 | [High Availability](#127-high-availability) |
| 12.8 | [Active Active](#128-active-active) |
| 12.9 | [Active Passive](#129-active-passive) |
| 12.10 | [Fault Injection](#1210-fault-injection) |

---

## 12.1 Chaos Engineering

**Summary:** Deliberately inject failures in production-like environments to discover weaknesses before real outages.

- Start with a steady-state hypothesis: "If X fails, users still succeed"
- Run controlled experiments with blast-radius limits and rollback plans
- Netflix Chaos Monkey pioneered the practice; tools include Litmus, Gremlin

**References:** _None yet._

---

## 12.2 Disaster Recovery

**Summary:** Processes and infrastructure to restore systems after catastrophic failure (region loss, ransomware, data corruption).

- DR plan covers people, runbooks, failover, and communication
- Tiers: backup-restore, pilot light, warm standby, hot standby, active-active
- Test DR regularly — untested plans fail when needed most

**References:** _None yet._

---

## 12.3 Backup Strategy

**Summary:** Regular copies of data and config so you can recover from loss or corruption.

- 3-2-1 rule: 3 copies, 2 media types, 1 offsite
- Full + incremental/differential schedules balance cost and recovery speed
- Encrypt backups; verify restore integrity with periodic test restores

**References:** _None yet._

---

## 12.4 Restore Strategy

**Summary:** Defined steps to recover data and services from backups within target timeframes.

- Document RTO/RPO per system; prioritize critical paths first
- Automate restore where possible; manual runbooks for edge cases
- Practice game days — measure actual restore time vs targets

**References:** _None yet._

---

## 12.5 RPO

**Summary:** Recovery Point Objective — maximum acceptable data loss measured in time.

- RPO = 0 requires synchronous replication (expensive, latency cost)
- RPO = 1 hour means backups/replication at least hourly
- Drives backup frequency and replication topology decisions

**References:** _None yet._

---

## 12.6 RTO

**Summary:** Recovery Time Objective — maximum acceptable downtime before service is restored.

- Includes detection, failover, and validation — not just data restore
- Lower RTO requires hot standby, automation, and pre-provisioned capacity
- Trade cost of standby infrastructure against business impact of downtime

**References:** _None yet._

---

## 12.7 High Availability

**Summary:** Designing systems to remain operational despite component failures — typically 99.9%+ uptime.

- Eliminate single points of failure via redundancy and health checks
- Automatic failover beats manual intervention for speed
- HA ≠ zero downtime — plan for maintenance and cascading failures

**References:** _None yet._

---

## 12.8 Active Active

**Summary:** Multiple sites serve traffic simultaneously; both handle reads and writes (with conflict handling).

- Lowest RTO/RPO — no warm-up on failover
- Requires data replication, split-brain prevention, and conflict resolution
- Best for global users needing low latency from nearest region

**References:** _None yet._

---

## 12.9 Active Passive

**Summary:** Primary site handles traffic; standby site activates only on failure.

- Simpler consistency — single writer at a time
- Standby can be cold (slow RTO), warm (faster), or hot (near-instant)
- Lower cost than active-active; higher failover latency

**References:** _None yet._

---

## 12.10 Fault Injection

**Summary:** Introducing specific failures (latency, errors, killed pods) to test resilience mechanisms.

- Validates circuit breakers, retries, timeouts, and graceful degradation
- Can run in staging or controlled production (chaos engineering subset)
- Measure impact: error rate, latency, and user-facing SLOs during injection

**References:** _None yet._

---

## Quick Reference

| Sub-topic | Metric / Focus | Key question |
|-----------|----------------|--------------|
| **Chaos Engineering** | Proactive failure discovery | What breaks first? |
| **Disaster Recovery** | Full recovery plan | Can we survive region loss? |
| **Backup / Restore** | Data recovery | How old / how fast? |
| **RPO** | Max data loss (time) | How much can we lose? |
| **RTO** | Max downtime (time) | How fast must we recover? |
| **High Availability** | Uptime design | What if a node dies? |
| **Active Active** | Dual live sites | Who resolves write conflicts? |
| **Active Passive** | Primary + standby | How fast can standby take over? |
| **Fault Injection** | Controlled failure tests | Do our defenses actually work? |

---

[← Back to master index](../README.md)
