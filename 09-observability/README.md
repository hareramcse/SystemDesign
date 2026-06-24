# 9. Observability

> Status: **Documented**  -  MASTER reference depth for all sub-topics below.

[<- Back to master index](../README.md)

---

## Sub-topics

| # | Sub-topic | Status |
|---|-----------|--------|
| 9.1 | [Logging](#91-logging) | Done |
| 9.2 | [Structured Logging](#92-structured-logging) | Done |
| 9.3 | [Metrics](#93-metrics) | Done |
| 9.4 | [Monitoring](#94-monitoring) | Done |
| 9.5 | [Distributed Tracing](#95-distributed-tracing) | Done |
| 9.6 | [OpenTelemetry](#96-opentelemetry) | Done |
| 9.7 | [Correlation IDs](#97-correlation-ids) | Done |
| 9.8 | [SLI](#98-sli) | Done |
| 9.9 | [SLO](#99-slo) | Done |
| 9.10 | [SLA](#910-sla) | Done |
| 9.11 | [Error Budgets](#911-error-budgets) | Done |
| 9.12 | [Alerting](#912-alerting) | Done |
| 9.13 | [Dashboards](#913-dashboards) | Done |
| 9.14 | [Health Checks](#914-health-checks) | Done |
| 9.15 | [Synthetic Monitoring](#915-synthetic-monitoring) | Done |



## Overview

Observability is the ability to understand internal system state from externally emitted signals - logs, metrics, and traces - without redeploying code. In distributed systems, it answers *what broke*, *where*, and *for whom* by correlating telemetry across services, infrastructure, and user journeys.

```mermaid
flowchart TB
    subgraph Apps['Application Layer']
        S1[Service A]
        S2[Service B]
        S3[Service C]
    end

    subgraph Pillars['Three Pillars']
        L[Logs / Discrete events]
        M[Metrics / Aggregated numbers]
        T[Traces / Request paths]
    end

    subgraph Pipeline['Telemetry Pipeline']
        C[OTel Collector]
        ST[(Storage)]
    end

    subgraph Consume['Consumption']
        D[Dashboards]
        A[Alerts]
        I[Investigation]
    end

    S1 & S2 & S3 --> L & M & T
    L & M & T --> C --> ST
    ST --> D & A & I
```

### Three Pillars of Observability

```mermaid
flowchart LR
    subgraph Logs
        L1[What happened?]
        L2[Error stack traces]
        L3[Audit events]
    end

    subgraph Metrics
        M1[How much?]
        M2[How fast?]
        M3[Saturation?]
    end

    subgraph Traces
        T1[Where did time go?]
        T2[Which service failed?]
        T3[Dependency map]
    end

    Q[Production Question] --> Logs & Metrics & Traces
```

---


---

## 9.1 Logging


### What is it

Immutable, timestamped records of discrete events - errors, requests, state transitions, security actions - written by applications and infrastructure components.

### Why it matters

Logs are the primary forensic tool when debugging a specific failure. They capture context (user, request, error) that aggregated metrics cannot, and they persist long after the incident for postmortems and compliance.

### How it works

Applications emit log lines via a logging framework (Logback, slog, winston). A shipper agent (Fluent Bit, Filebeat) tails files or receives stdout, batches records, and forwards them to a centralized store (Elasticsearch, Loki, CloudWatch). Operators search and filter by time, level, service, and correlation ID.

### Diagram

```mermaid
flowchart LR
    App[Application] -->|stdout/file| Agent[Log Shipper]
    Agent -->|batch| Store[(Log Store)]
    Store --> Search[Search & Filter]
    Search --> Debug[Incident Debug]
```

### Key details

- **Levels:** ERROR, WARN, INFO, DEBUG - tune per environment; production rarely needs DEBUG globally
- **Centralization:** never rely on pod-local disks; logs vanish on restart
- **Retention:** hot (7 - 30 days searchable), warm/cold archive for compliance
- **PII/secrets:** redact tokens, passwords, and regulated data at source

### When to use

- Debugging specific errors and stack traces
- Audit trails and security investigations
- Reconstructing request timelines when paired with correlation IDs

### Trade-offs

| Pros | Cons |
|------|------|
| Rich contextual detail | Expensive at high volume |
| Human-readable | Unstructured text hard to query |
| Easy to adopt | Can become noisy without discipline |

### References

- [Google SRE  -  Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

---


## 9.2 Structured Logging


### What is it

Logging where each event is a machine-parseable record (typically JSON) with named fields instead of free-form text strings.

### Why it matters

Structured logs enable fast filtering (`level=ERROR AND service=payment`), aggregation (count errors by `errorCode`), and automatic correlation with traces - without fragile regex parsing.

### How it works

The logging library serializes each event to JSON with standard fields (`timestamp`, `level`, `message`, `traceId`, `spanId`, custom dimensions). The log platform indexes fields as columns. Queries become SQL-like or Lucene field filters.

### Diagram

```mermaid
flowchart TB
    E[Event] --> F{Format}
    F -->|Unstructured| T['2024-01-15 ERROR payment failed']
    F -->|Structured| J["{level:ERROR, service:payment, orderId:123}"]
    J --> Q[Field Queries]
    T --> R[Regex Parsing]
```

### Key details

- Standard fields: `timestamp`, `level`, `service`, `traceId`, `message`
- Use consistent naming (snake_case or camelCase - pick one)
- Libraries: Logback JSON encoder, structlog, zerolog, winston JSON
- Include exception type, stack (truncated), and business IDs

### When to use

- Any production service at scale
- Microservices where cross-service log correlation is required
- Platforms feeding logs into metrics or alerting rules

### Trade-offs

| Pros | Cons |
|------|------|
| Fast, reliable queries | Slightly larger payload per line |
| Schema evolution possible | Requires discipline on field names |
| Integrates with traces | Bad schemas create indexing bloat |

### References

- [OpenTelemetry Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)

---


## 9.3 Metrics


### What is it

Numeric measurements collected over time - counters, gauges, histograms - that describe system behavior as time series.

### Why it matters

Metrics compress billions of events into trend lines suitable for dashboards, capacity planning, and automated alerting. They answer *how much* and *how fast* at a glance.

### How it works

Instrumentation libraries expose metrics via pull (Prometheus scrapes `/metrics`) or push (StatsD, OTLP). A time-series database stores samples with labels (service, region, endpoint). Query languages (PromQL) aggregate, rate, and percentile over windows.

### Diagram

```mermaid
flowchart LR
    S[Service] -->|expose| EP['/metrics']
    P[Prometheus] -->|scrape| EP
    P --> TSDB[(Time Series DB)]
    TSDB --> G[Grafana Dashboard]
    TSDB --> R[Alert Rules]
```

### Key details

- **Counter:** monotonically increasing (`http_requests_total`)
- **Gauge:** point-in-time value (`queue_depth`, `memory_bytes`)
- **Histogram/Summary:** distributions for latency percentiles
- Label cardinality: avoid high-cardinality labels (user IDs) on metrics

### When to use

- SLI measurement (availability, latency)
- Capacity and saturation monitoring
- Alerting on symptoms (error rate, p99 latency)

### Trade-offs

| Pros | Cons |
|------|------|
| Cheap storage vs logs | Loses per-event detail |
| Great for trends | Cardinality explosions are costly |
| Standard alerting input | Requires careful instrumentation design |

### References

- [Prometheus metric types](https://prometheus.io/docs/concepts/metric_types/)
- [RED method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)

---


## 9.4 Monitoring


### What is it

The practice of continuously observing system health by collecting metrics (and sometimes logs/traces), comparing them to expectations, and surfacing anomalies through dashboards and alerts.

### Why it matters

Monitoring detects degradation before users report outages. It operationalizes reliability goals and gives teams shared situational awareness during incidents.

### How it works

Instrumentation feeds a monitoring stack. SRE frameworks (RED for services, USE for resources) define what to measure. Dashboards visualize golden signals; alert rules fire when thresholds breach; on-call responds via runbooks.

### Diagram

```mermaid
flowchart TB
    subgraph Signals
        R[Rate]
        E[Errors]
        D[Duration]
        S[Saturation]
    end
    Signals --> DB[(Metrics Store)]
    DB --> Dash[Dashboards]
    DB --> Alert[Alert Manager]
    Alert --> OC[On-Call]
```

### Key details

- **RED:** Rate, Errors, Duration - for request-driven services
- **USE:** Utilization, Saturation, Errors - for CPU, disk, network
- **Four golden signals:** latency, traffic, errors, saturation
- Monitor symptoms (user-visible), not only causes (CPU)

### When to use

- Always, for any production workload
- Before launching new services (define dashboards first)
- During incidents for real-time triage

### Trade-offs

| Pros | Cons |
|------|------|
| Proactive failure detection | Alert fatigue if poorly tuned |
| Shared team visibility | Wrong metrics create false confidence |
| Drives capacity decisions | Tooling cost at scale |

### References

- [Google SRE  -  Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)

---


## 9.5 Distributed Tracing


### What is it

**Distributed tracing** tracks a single logical request as it crosses process and network boundaries, represented as a **trace** (the whole journey) composed of **spans** (individual timed operations) linked by a shared **trace ID** and parent-child relationships.

Tracing answers: *where did the 2-second checkout latency go?* and *which downstream service returned 500?*

### Why it matters

In monoliths, a stack trace suffices. In microservices:

- One API call fans out to 5–20 internal HTTP/gRPC/DB calls
- Logs alone cannot reconstruct timing per hop
- Metrics show p99 latency spike but not **which dependency** caused it
- **Interview staple:** "Logs = what; metrics = how much; traces = where time went"

### How it works

**Core concepts:**

| Concept | Definition | Example |
|---------|------------|---------|
| **Trace** | End-to-end path of one request | Checkout `trace_id=abc123` |
| **Span** | One timed operation within a trace | `HTTP GET /users/42` — 12 ms |
| **Root span** | First span; no parent | API gateway ingress |
| **Child span** | Created by downstream call | Order service → Payment gRPC |
| **Span attributes** | Key-value metadata | `http.status_code=200`, `db.system=postgresql` |
| **Span events** | Timestamped annotations | `exception` at T+45ms |

**Span tree example (waterfall):**

```text
[trace_id: abc123]  total 847ms
├── gateway.http          847ms
│   ├── order.getOrder    320ms
│   │   ├── postgres.query  45ms
│   │   └── inventory.grpc  180ms
│   └── payment.charge    410ms
│       └── stripe.http     390ms
```

```mermaid
flowchart TB
    Root[gateway span / trace_id=abc]
    Root --> S1["order span / parent=abc:gateway"]
    Root --> S2["payment span / parent=abc:gateway"]
    S1 --> S3[db query span]
    S1 --> S4[inventory grpc span]
    S2 --> S5[stripe http span]
```

**Trace context propagation**

For spans to link across services, context must **flow with the request**:

| Standard | Header | Format |
|----------|--------|--------|
| **W3C Trace Context** | `traceparent` | `00-{trace-id}-{parent-span-id}-{flags}` |
| **W3C** | `tracestate` | Vendor-specific key-value pairs |
| Legacy B3 | `X-B3-TraceId`, `X-B3-SpanId` | Zipkin format (still common) |

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as Gateway
    participant O as Order Service
    participant P as Payment Service

    C->>GW: GET /checkout / traceparent: 00-abc-span1-01
    Note over GW: Start root span / trace_id=abc
    GW->>O: Forward / traceparent: 00-abc-span2-01
    Note over O: Child span, parent=span2
    O->>P: gRPC metadata traceparent
    Note over P: Child span under O
    P-->>O: response
    O-->>GW: response
    GW-->>C: response
    Note over GW,P: All spans exported with trace_id=abc
```

**Propagation rules:**

1. **Ingress:** extract context from incoming headers or start new trace
2. **Egress:** inject context into outbound HTTP headers / gRPC metadata / Kafka headers
3. **Broken propagation** → orphan spans; trace appears fragmented in UI
4. **Async:** pass context in message headers (`traceparent` in Kafka record headers)

**Instrumentation layers:**

| Layer | What it traces | Example |
|-------|----------------|---------|
| Auto-instrumentation | HTTP server/client, gRPC, DB drivers | OpenTelemetry Java agent |
| Manual spans | Business logic | `span.addEvent("payment_authorized")` |
| Service mesh | Envoy generates spans | Istio without app changes |

**Sampling (production necessity)**

100% trace collection at 50K RPS is expensive. Strategies:

| Strategy | Behavior | When |
|----------|----------|------|
| **Head-based** | Decide at root (1–10% random) | Default prod |
| **Tail-based** | Keep traces that ended in error or slow | OTel Collector tail sampling |
| **Always sample** | Dev/staging, errors | Debug sessions |

Rule of thumb: sample 1–5% in prod; **always** capture errors if tail sampling available.

**Linking traces to logs:**

```json
{
  "level": "ERROR",
  "message": "Payment declined",
  "traceId": "abc123def456",
  "spanId": "span789",
  "service": "payment-service"
}
```

Click trace ID in Jaeger → see exact log lines in Loki/Elasticsearch.

### Production rules

| Rule | Rationale |
|------|-----------|
| **W3C Trace Context everywhere** | `traceparent` on HTTP, gRPC metadata, Kafka headers — broken hop = orphan spans |
| **Auto-instrument + manual business spans** | Framework spans show HTTP; add spans for `authorize_payment`, `reserve_inventory` |
| **Sample in prod, full in staging** | 1–5% head-based default; tail sampling for errors and slow traces |
| **Inject traceId into structured logs** | `traceId` + `spanId` fields — pivot from metric spike to exact trace |
| **Set span status on errors** | Non-2xx and exceptions → `ERROR` status; green spans hide failures in UI |
| **Control span attribute cardinality** | Never `user_id` as span attribute at 1M RPS — use logs or low-cardinality buckets |
| **Collector between apps and backends** | OTel Collector: batch, tail-sample, scrub PII before export |
| **Alert on propagation gaps** | Metric: orphan span rate — rising = new service missing instrumentation |
| **Retention policy by tier** | Hot 7d for debug; longer for sampled errors only — cost control |
| **Test propagation in CI** | Contract test: service A → B must share `trace_id` in integration env |

**Sampling math:** 50K RPS × 100 spans/trace × 100% sample = 5M spans/sec — unsustainable. Tail sampling keeping 100% of errors + 1% success is typical production compromise.

### Key details

- **Critical path:** longest chain of dependent spans determines latency floor
- **Span kind:** `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER`, `INTERNAL`
- **Status:** `OK`, `ERROR` — set on non-2xx and exceptions
- **Baggage:** propagate business context (`tenant_id`) — use sparingly (PII risk)
- Tools: Jaeger, Zipkin, Grafana Tempo, Honeycomb, Datadog APM

### When to use

- Any microservice with multi-hop synchronous or async paths
- Latency debugging (p99 investigations, regression after deploy)
- Dependency mapping and architecture discovery
- SLO debugging — correlate burn-rate spikes to specific downstream

### Trade-offs

| Pros | Cons |
|------|------|
| Pinpoints bottleneck span | Storage cost at high sampling |
| Visual call graph for onboarding | Broken context = incomplete traces |
| Links logs ↔ traces via trace ID | Manual instrumentation for business spans |
| Mesh can add tracing without code | Cardinality of span attributes must be controlled |

| Pitfall | Symptom | Mitigation |
|---------|---------|------------|
| **Broken propagation** | Fragmented traces; can't find payment span | Middleware injects/extracts on every egress; lint in CI |
| **100% sampling in prod** | APM bill explosion; collector OOM | Head 1–5% + tail sample errors/slow |
| **High-cardinality attributes** | Backend index blow-up | Allowlist attributes; user IDs in logs only |
| **Missing error status** | Failed requests show green spans | Set `ERROR` on exceptions and 5xx |
| **Async context loss** | Worker spans detached from HTTP request | Pass context in message headers / `context.attach` |
| **Baggage with PII** | GDPR violation in trace export | Avoid baggage; use traceId to join logs |
| **No business spans** | "payment took 400ms" but not which step | Manual spans around domain operations |

### References

- [OpenTelemetry — Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Jaeger architecture](https://www.jaegertracing.io/docs/latest/architecture/)

---


## 9.6 OpenTelemetry


### What is it

A vendor-neutral, CNCF-standard API, SDK, and collector for generating and exporting traces, metrics, and logs - one instrumentation layer for many backends.

### Why it matters

Avoids vendor lock-in and duplicate instrumentation. Teams instrument once with OTel and export to Jaeger, Prometheus, Datadog, or cloud-native backends via configuration.

### How it works

The OTel SDK auto-instruments frameworks (HTTP, gRPC, DB) and supports manual spans. The **Collector** receives OTLP, processes (filter, batch, enrich), and exports to backends. Semantic conventions standardize attribute names across languages.

### Diagram

```mermaid
flowchart LR
    subgraph Apps
        A1[Java + OTel]
        A2[Go + OTel]
        A3[Node + OTel]
    end
    Apps -->|OTLP| Col[OTel Collector]
    Col --> J[Jaeger]
    Col --> P[Prometheus]
    Col --> CW[CloudWatch]
```

### Key details

- Signals: traces, metrics, logs (unified pipeline)
- Auto-instrumentation vs manual spans for business logic
- Collector processors: tail sampling, attribute scrubbing
- W3C Trace Context is the default propagation format

### When to use

- Greenfield microservices (default choice in 2024+)
- Migrating off vendor-specific agents
- Polyglot environments needing consistent telemetry

### Trade-offs

| Pros | Cons |
|------|------|
| Vendor neutral | Collector ops overhead |
| Rich ecosystem | Semantic convention adoption takes time |
| Single SDK per language | Some vendor features need proprietary exporters |

### References

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [OTel Collector](https://opentelemetry.io/docs/collector/)

---


## 9.7 Correlation IDs


### What is it

A unique identifier assigned to a request at the system edge and propagated through every service, log line, trace, and support ticket for that request.

### Why it matters

Correlation IDs stitch together fragmented telemetry across dozens of services, turning "find needle in haystack" into "filter by `requestId=xyz`".

### How it works

The API gateway generates an ID (UUID) if the client did not send one. The ID is passed via HTTP header (`X-Request-ID`, `X-Correlation-ID`) or embedded in W3C trace context. Every service copies it into structured logs and span attributes.

### Diagram

```mermaid
flowchart LR
    GW["Gateway / reqId: 7f3a"] --> S1["Order / reqId: 7f3a"]
    S1 --> S2["Payment / reqId: 7f3a"]
    S1 --> L1[(Logs)]
    S2 --> L2[(Logs)]
    L1 & L2 --> Q["Query: reqId=7f3a"]
```

### Key details

- Generate at edge; never trust client-supplied IDs for security decisions
- Log the ID on every line - including async workers and message consumers
- Align with `traceId` when using distributed tracing (often the same or linked)
- Pass through message queues via message headers

### When to use

- All distributed systems
- Support tooling linking user reports to backend logs
- Async pipelines (include ID in event payload)

### Trade-offs

| Pros | Cons |
|------|------|
| Simple, high-value | Useless if propagation breaks mid-chain |
| Works without full tracing | Not a substitute for span timing |
| Low implementation cost | Duplicate IDs if clients resend same header |

### References

- [W3C Trace Context](https://www.w3.org/TR/trace-context/)

---


## 9.8 SLI


### What is it

A **Service Level Indicator (SLI)** is a **quantitative measure** of some aspect of service behavior from the **user's perspective** — the raw metric you actually observe before setting targets.

SLIs answer: *"What exactly are we measuring?"* SLOs answer: *"How well must we measure?"*

### Why it matters

Vague goals ("keep the site fast") are unmeasurable. Precise SLI definitions prevent:

- Engineering optimizing server CPU while users see errors at the CDN
- Disputes during incidents ("was that an outage?")
- Alerts that fire on irrelevant internal metrics

**Google SRE principle:** measure SLIs at the **user boundary** (load balancer, edge, synthetic probe) when possible — not only inside the data center.

### How it works

**The SLI formula:**

```text
SLI = (good events) / (valid events)   × 100%
```

| Term | Definition | Example |
|------|------------|---------|
| **Valid events** | Requests that should be counted | All HTTP requests excluding health checks |
| **Good events** | Events meeting "good" criteria | 2xx/3xx responses, or latency < 300ms |
| **Bad events** | Valid but not good | 5xx, timeout, latency > threshold |

**Common SLI types:**

| SLI type | Good event | Valid events | Measurement point |
|----------|------------|--------------|-------------------|
| **Availability** | Successful response (2xx/3xx) | All non-healthcheck requests | Edge LB, synthetic |
| **Latency** | Response time < T ms | All successful requests | Edge histogram |
| **Quality / Correctness** | Business-valid outcome | All attempts | App-level (e.g., search with results) |
| **Freshness** | Data age < N seconds | All reads | Pipeline watermark |
| **Durability** | Data not lost | All writes | Storage audit (rare for most teams) |

**Availability SLI — definition debates (pick one explicitly):**

| Definition | Formula | Trade-off |
|------------|---------|-----------|
| Strict | `(2xx + 3xx) / all` | Client 404 counts against you |
| Server-focused | `2xx / (all - 4xx)` | Excludes client errors |
| Success-only | `non-5xx / all` | Treats 4xx as "available" |

**Latency SLI — percentile over threshold:**

```text
Latency SLI = (requests with duration < 300ms) / (all valid requests)
```

Or define as: "99th percentile latency < 500ms" — the SLI is the **fraction of windows** meeting that percentile target.

```mermaid
flowchart LR
    Req[Incoming Requests] --> V{Valid event?}
    V -->|healthcheck| Skip[Exclude]
    V -->|yes| G{Good event?}
    G -->|2xx in 200ms| Good[Good +1]
    G -->|5xx or slow| Bad[Bad +1]
    Good & Bad --> SLI['SLI = good / (good+bad)']
    SLI --> SLO[Compare to SLO target]
```

**Instrumentation examples:**

```promql
# Availability SLI (5-minute rate)
sum(rate(http_requests_total{status!~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# Latency SLI — fraction under 300ms (histogram)
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))
```

**Multi-window SLIs:** measure over rolling 1h, 24h, 30d for SLO compliance reporting.

### Production rules

| Rule | Rationale |
|------|-----------|
| **Measure at the user boundary** | Edge LB or synthetic probe — internal green + user-facing red is common |
| **Document good/valid/bad explicitly** | Written SLI spec prevents "was 404 an outage?" debates in postmortems |
| **Exclude health checks and scrapes** | `/health`, `/metrics` inflate availability numerator |
| **2–3 SLIs per service max** | Availability + latency + one domain metric — more creates conflicting goals |
| **Same SLI definition in code and docs** | PromQL in runbook matches dashboard — no drift between teams |
| **Segment SLIs by tier** | Free vs paid, region, mobile vs web — aggregate hides segment pain |
| **Pair availability with latency** | 100% availability at 30s latency is not "good" for users |
| **Version SLI changes** | Changing "good event" definition resets historical comparison |
| **Synthetic + real traffic** | Synthetics catch DNS/CDN issues; real traffic catches long-tail paths |

### Key details

- **User-centric:** measure what users experience — synthetic + real traffic combined
- **Few SLIs per service:** typically availability + latency + one domain-specific (correctness)
- **Exclude noise:** health checks, internal metrics scrapes, pen-test traffic
- **Document exclusions:** planned maintenance — in or out of SLI?
- SLI without SLO is just a dashboard metric — pair them (see 9.9)

### When to use

- Defining any reliability target before writing SLO
- Building golden-signal dashboards
- Incident retrospectives — "did we breach SLO?"
- Capacity planning — latency SLI degradation precedes outages

### Trade-offs

| Pros | Cons |
|------|------|
| Objective, auditable measurement | Definition debates (4xx, 3xx) take time |
| Foundation for SLO/error budget | Edge vs server measurement can differ |
| Aligns teams on "good" | Lagging indicator — misses UX nuance (slow but "success") |
| Drives correct instrumentation | Too many SLIs → conflicting goals |

| Pitfall | Symptom | Mitigation |
|---------|---------|------------|
| **Internal-only measurement** | Dashboard green, users complaining | SLI at edge LB + synthetics |
| **4xx definition ambiguity** | Postmortem arguments | Document: client 404 in or out of availability |
| **Too many SLIs** | Team optimizes wrong metric | Cap at 2–3 per service |
| **Health check in numerator** | Fake 100% availability | Exclude `/health`, kube probes |
| **Latency SLI on all requests** | Includes failed fast 500s as "fast" | Measure latency on successful requests only |
| **Unsegmented global SLI** | EU users suffering, global SLI fine | Slice by region/client tier |

### References

- [Google SRE Workbook — SLI selection](https://sre.google/workbook/implementing-slos/#sli-selection)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

---


## 9.9 SLO


### What is it

A **Service Level Objective (SLO)** is an **internal reliability target** — a threshold applied to an SLI over a defined **time window**, representing the level of reliability your team commits to deliver.

Example: *"99.9% of checkout requests return a successful response within 30 rolling days."*

### Why it matters

SLOs translate "users should be happy" into engineering decisions:

- When to page on-call (SLO burn rate)
- Whether Friday deploys are allowed (error budget remaining)
- How to prioritize reliability vs feature work
- **Interview chain:** SLI (measure) → SLO (target) → Error budget (allowed failure) → SLA (contract)

### How it works

**SLI → SLO relationship:**

```mermaid
flowchart TB
    subgraph Measure
        SLI["SLI: good / valid events"]
    end
    subgraph Internal
        SLO["SLO: 99.9% over 30 days"]
        EB["Error Budget: 0.1% = 43.2 min downtime"]
    end
    subgraph External
        SLA["SLA: 99.95% to customers"]
    end
    SLI --> SLO
    SLO --> EB
    SLO -.->|safety margin| SLA
```

**Writing good SLOs:**

| Component | Example | Notes |
|-----------|---------|-------|
| SLI | Availability of `POST /checkout` | Specific user journey beats "all endpoints" |
| Target | 99.9% | Three nines = 43.2 min/month budget |
| Window | Rolling 30 days | Calendar month also common |
| Measurement | Edge load balancer logs | Document data source |

**Example SLO set for a payment API:**

| SLO | SLI definition | Target | Window |
|-----|----------------|--------|--------|
| Availability | Non-5xx responses / valid requests | 99.95% | 30d rolling |
| Latency | Requests < 500ms / successful requests | 99% | 30d rolling |
| Correctness | Charges without duplicate / total charges | 99.99% | 30d rolling |

**Nines cheat sheet (30-day window):**

| Availability | Downtime/month | Error budget |
|--------------|----------------|--------------|
| 99% (two nines) | 7h 12m | 1% |
| 99.9% (three nines) | 43m 12s | 0.1% |
| 99.95% | 21m 36s | 0.05% |
| 99.99% (four nines) | 4m 19s | 0.01% |

**SLO-based alerting (multi-burn-rate):**

Alert when error budget is **burning too fast** — not on every blip:

| Alert | Burn rate | Window | Meaning |
|-------|-----------|--------|---------|
| Page | 14.4× | 1h | Budget gone in ~2 days if continues |
| Page | 6× | 6h | Serious degradation |
| Ticket | 3× | 3d | Slow burn worth investigating |

```promql
# Simplified burn-rate concept
# (error_rate / (1 - SLO_target)) > burn_rate_threshold
```

**SLO review cadence:**

- Quarterly: adjust targets with product — tighter if users demand, looser if cost prohibitive
- Post-incident: did SLO breach? Was target realistic?
- Avoid **SLO sprawl** — 3–5 SLOs per service maximum

```mermaid
flowchart LR
    Users[User happiness] --> SLI[SLI measurement]
    SLI --> SLO{SLO met?}
    SLO -->|yes| Ship[Feature development]
    SLO -->|budget left| Ship
    SLO -->|budget exhausted| Rel[Reliability work only]
```

### Key details

- **User-centric journeys:** SLO on `/checkout` not `/internal/metrics`
- **SLO < SLA:** internal target stricter than contractual SLA (buffer for disputes)
- **Shared SLOs:** one customer-facing SLO may map to multiple backend services
- **Composite SLOs:** end-to-end only if you own full path; else per-service SLOs
- Document **exclusions:** maintenance windows, third-party dependency failures

### When to use

- Any production service past MVP — before alert fatigue sets in
- Release governance and error budget policies
- Prioritizing tech debt vs features with product leadership
- On-call paging criteria — page on SLO burn, not CPU > 80%

### Trade-offs

| Pros | Cons |
|------|------|
| Quantifies reliability goals | Wrong SLO optimizes wrong behavior |
| Enables error budget culture | Hard to measure end-to-end in microservices |
| Reduces subjective release debates | Setting too loose → meaningless; too tight → no shipping |
| Aligns eng and product | Requires instrumentation investment first |

### References

- [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Google SRE — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)

---


## 9.10 SLA


### What is it

**Service Level Agreement** - a contractual commitment to customers defining measurable service targets, often with financial remedies (credits) for breach.

### Why it matters

SLAs set external expectations and legal obligations. They must be achievable, measurable, and backed by internal engineering targets (SLOs) with safety margin.

### How it works

Legal and product define SLA terms (e.g., 99.95% monthly uptime). Engineering measures compliance via SLIs. Internal SLO is set stricter than SLA (buffer). Breach triggers service credits per contract.

### Diagram

```mermaid
flowchart LR
    SLI[SLI Measurement] --> SLO[SLO Internal 99.99%]
    SLO --> SLA[SLA External 99.95%]
    SLA -->|breach| Credit[Service Credits]
```

### Key details

- External-facing, legally binding
- Define measurement method, exclusions (planned maintenance), reporting
- Always set SLA below internal SLO
- Multi-tenant: per-customer vs platform-wide SLAs

### When to use

- B2B SaaS with enterprise contracts
- Cloud provider service offerings
- Regulated industries with uptime guarantees

### Trade-offs

| Pros | Cons |
|------|------|
| Customer trust | Financial liability on miss |
| Clear expectations | Conservative targets limit agility |
| Sales differentiator | Disputes over measurement |

### References

- [AWS SLA overview](https://aws.amazon.com/legal/service-level-agreements/)

---


## 9.11 Error Budgets


### What is it

An **error budget** is the **allowed amount of unreliability** implied by an SLO — the mirror image of the reliability target. If your SLO is 99.9% availability, your error budget is **0.1%** of valid events in the measurement window that may fail.

Error budgets make reliability a **finite resource** product and engineering spend together.

### Why it matters

Without error budgets:

- Reliability debates are subjective ("is one outage too many?")
- Teams ship recklessly OR never ship from fear
- On-call pages on noisy thresholds unrelated to user impact

With error budgets:

- **Budget remaining** → justified risk-taking (deploys, experiments)
- **Budget exhausted** → freeze features, focus on stability
- **Interview one-liner:** "Error budget = 1 − SLO; it's how much downtime/latency failure you can afford per month."

### How it works

**Calculation:**

```text
Error budget (fraction) = 1 − SLO target
Error budget (time)     = window_duration × (1 − SLO target)
```

**30-day rolling window examples:**

| SLO target | Error budget % | Allowed downtime / month |
|------------|----------------|--------------------------|
| 99% | 1% | ~7h 18m |
| 99.9% | 0.1% | ~43m 48s |
| 99.95% | 0.05% | ~21m 54s |
| 99.99% | 0.01% | ~4m 23s |

**Request-based example:**

```text
SLO: 99.9% availability over 30d
Valid requests/month: 100 million
Error budget = 0.1% × 100M = 100,000 failed requests allowed
```

```mermaid
flowchart TB
    SLO['SLO 99.9%'] --> EB["Total budget: 43.2 min / month"]
    EB --> Used['Consumed by incidents, / bad deploys, experiments']
    EB --> Remaining['Remaining budget']
    Remaining -->|> 50%| Green[Ship features freely]
    Remaining -->|10-50%| Yellow[Cautious deploys, more canaries]
    Remaining -->|< 10% or 0| Red[Feature freeze - reliability sprint]
```

**What consumes error budget:**

| Event | Consumption |
|-------|-------------|
| User-visible outage (5xx storm) | Large — minutes of bad SLI |
| Partial degradation (elevated latency) | Latency SLO budget |
| Bad deploy rolled back in 5 min | Small but non-zero |
| Planned maintenance | Policy choice — exclude or consume |
| Third-party dependency down | Depends — exclude if out of your control |

**Error budget policy (example):**

```text
Budget > 50% remaining  → normal development velocity
Budget 25–50%           → require canary + extra review for risky changes
Budget < 25%            → reliability tasks prioritized in sprint
Budget = 0                → feature freeze until budget recovers OR SLO revised
```

**Burn rate — how fast budget is consumed:**

```text
burn_rate = (current error rate) / (error budget rate)

error budget rate = 1 − SLO = 0.001 for 99.9% SLO

If 1% of requests failing now:
burn_rate = 0.01 / 0.001 = 10×
→ at this rate, monthly budget exhausted in ~3 days
```

```mermaid
flowchart LR
    subgraph Alerts
        B1["1h window: 14.4× burn -> PAGE"]
        B2["6h window: 6× burn -> PAGE"]
        B3["3d window: 3× burn -> TICKET"]
    end
    SLI[SLI error rate] --> Burn[Burn rate calc]
    Burn --> B1 & B2 & B3
```

**Product + engineering collaboration:**

- Error budget is a **product decision tool** — not only SRE
- Product accepts: higher SLO = slower feature velocity (more careful deploys)
- Postmortem when budget burned unexpectedly — was deploy process at fault?

### Production rules

| Rule | Rationale |
|------|-----------|
| **Publish budget policy in writing** | Product agrees: budget < 25% → feature freeze — avoids subjective debates |
| **Dashboard budget remaining %** | Visible to eng + product; projection "exhausted in N days at current burn" |
| **Tie deploy policy to budget** | Friday deploys OK when budget > 50%; require extra canary when < 25% |
| **Postmortem on significant burn** | >5% budget in one incident → action items on prevention |
| **Per-SLO budgets, not one global** | Latency budget exhausted while availability fine — different response |
| **Don't exclude everything** | Over-broad maintenance exclusions make budget meaningless |
| **Review SLO quarterly** | Too loose = fake budget; too tight = no shipping |
| **Burn-rate alerts, not static thresholds** | 1% errors for 5 min may be fine; same rate for 6h exhausts budget |
| **Track budget by deploy** | Attribute burn to release version — identifies bad deploy patterns |

### Key details

- Track budget **per SLO**, per service, rolling window (30d common)
- Dashboard: budget remaining % with projection ("at current burn, exhausted in 4 days")
- **Partial outages** consume proportional budget — not binary
- Budget recovery is automatic as window rolls forward (old bad minutes age out)
- Don't game with loose SLOs — meaningless budget helps no one

### When to use

- Teams with defined SLOs practicing SRE
- Release governance: "Can we deploy Friday 4pm?"
- Sprint planning: reliability work vs features
- Incident severity classification tied to budget impact

### Trade-offs

| Pros | Cons |
|------|------|
| Objective ship/no-ship decisions | Requires SLO maturity first |
| Aligns product and engineering incentives | Loose SLO → fake budget |
| Reduces blame — "we spent budget" | Cultural buy-in takes time |
| Focuses alerts on user impact | Multi-service budgets are complex |
| Encourages calculated risk | Budget freeze can frustrate product if overused |

| Pitfall | Symptom | Mitigation |
|---------|---------|------------|
| **Loose SLO (99%)** | Budget never consumed; false confidence | Tighten SLO to user expectations |
| **No written policy** | Subjective "can we ship?" every Friday | Published error budget policy |
| **Single global budget** | One service burns everyone's budget | Per-service or per-journey budgets |
| **Ignoring latency budget** | Availability fine, users leave due to slowness | Separate latency SLO + budget |
| **Gaming exclusions** | All outages "excluded" | Limit exclusion categories; document each |
| **Budget freeze without recovery plan** | Features blocked indefinitely | Define exit criteria (budget > X% for 7d) |

### References

- [Google SRE Book — Embracing Risk](https://sre.google/sre-book/embracing-risk/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)

---


## 9.12 Alerting


### What is it

**Alerting** is automated notification when monitored signals indicate **user-impacting** degradation — routing on-call engineers or automation to respond with runbooks. Good alerting optimizes for **signal-to-noise**: page humans only when immediate action prevents SLO breach or ongoing outage.

Alerting is distinct from **monitoring** (observation) and **dashboards** (exploration). An alert must answer: *who acts, how fast, and what do they do first?*

### Why it matters

Users should not be the first detectors of outages. Well-designed alerts shorten **mean time to detect (MTTD)** and direct engineers to **actionable** problems — not to graphs that merely look interesting.

Poor alerting causes **alert fatigue**: on-call ignores pages, real outages get missed, burnout rises. Google SRE guidance: **every page should be urgent, actionable, and user-symptom-based.**

### How it works

**Alert pipeline:**

```mermaid
flowchart TB
    subgraph Sources
        M[Metrics / PromQL]
        L[Log-based rules]
        S[Synthetics]
        T[Trace anomaly]
    end
    Sources --> R[Alert Rule Evaluator]
    R -->|firing| AM[Alertmanager]
    AM --> Group[Group + Dedupe + Inhibit]
    Group --> Route{Severity Router}
    Route -->|P1| PD[PagerDuty / Phone]
    Route -->|P2| Slack[Slack + Ticket]
    Route -->|P3| Dash[Dashboard only]
    PD --> RB[Runbook link]
```

**1. Symptom vs cause alerting**

| Alert on | Example | Problem |
|----------|---------|---------|
| **Symptom (good)** | Checkout error rate > 1% for 5m | User-impacting; page |
| **Cause (bad alone)** | Single pod CPU > 90% | May self-heal; noisy |
| **Cause (OK with symptom)** | DB connection pool exhausted **and** API 5xx up | Actionable root hint |

Rule: **page on symptoms**; attach cause metrics as context in notification, not separate pages.

**2. SLO multi-burn-rate alerts (production standard)**

Page when error budget burns too fast — not on every error blip:

| Severity | Burn rate | Window | Budget exhausted if sustained |
|----------|-----------|--------|-------------------------------|
| **Page (critical)** | 14.4× | 1h | ~2 days |
| **Page (critical)** | 6× | 6h | ~5 days |
| **Ticket** | 3× | 3d | ~10 days |
| **Ticket** | 1× | 3d | Full window |

```text
burn_rate = (current_error_rate) / (1 − SLO_target)

For 99.9% SLO: budget rate = 0.001
If 1% failing now: burn_rate = 0.01 / 0.001 = 10×
```

```promql
# Simplified availability burn — adjust SLI to match your definition
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  /
  sum(rate(http_requests_total[1h]))
)
/
(1 - 0.999)  # 99.9% SLO → 0.001 budget
> 14.4
```

**3. Severity tiers**

| Tier | Response | Channel | Examples |
|------|----------|---------|----------|
| **P1** | Immediate, 24/7 page | PagerDuty + phone | Checkout down, data loss, security breach |
| **P2** | Business hours or 30m SLA | Slack + Jira ticket | Elevated latency, single-AZ degradation |
| **P3** | Next business day | Dashboard annotation | Disk 70%, cert expires in 30d |
| **P4** | Informational | Weekly review | Capacity trend, non-urgent tech debt |

**4. Alertmanager behaviors**

- **Grouping:** collapse 50 `PodCrashLooping` into one notification
- **Inhibition:** suppress `NodeNotReady` alerts when `RegionDown` already firing
- **Silences:** maintenance windows with ticket reference — not permanent mute
- **Repeat interval:** re-page every N hours if still firing — not every 30s

**5. Runbook requirements**

Every P1/P2 alert links to runbook with:

```text
1. Impact statement (what users see)
2. First 5-minute checks (dashboards, logs, recent deploys)
3. Mitigation steps (rollback, scale, failover)
4. Escalation path (who to call if step 3 fails)
5. Communication template (status page update)
```

### Diagram

```mermaid
flowchart TB
    M[Metrics] --> R[Alert Rules]
    R -->|fire| AM[Alert Manager]
    AM -->|P1| Page[PagerDuty]
    AM -->|P2| Ticket[Jira]
    AM -->|P3| Slack[Slack Channel]
    Page --> RB[Runbook]
```

### Key details

- Alert on **symptoms** (error rate, SLO burn), not causes (single CPU spike)
- Severity tiers: P1 pages, P2 business hours, P3 dashboard-only
- Every alert must be **actionable** — if no action, delete the alert
- Use multi-window burn-rate alerts for SLO-based paging
- **Alert budget:** team tracks pages/week; high volume → tuning sprint
- Test alerts in staging; verify routing and runbook links quarterly
- `for: 5m` duration prevents flapping on brief spikes

### Production rules

| Rule | Rationale |
|------|-----------|
| **If it pages, it must be actionable** | Non-actionable pages train on-call to ignore |
| **One alert per user-visible symptom** | Multiple causes → one page with diagnostic context |
| **Runbook link in every page** | 3 AM responder needs steps, not just a graph |
| **Inhibit dependent alerts** | Node down shouldn't page 50 pod alerts |
| **Review fired alerts weekly** | Tune or delete alerts with no human response |
| **No alert without owner** | Orphan rules drift; `owner: team-checkout` label |
| **Deploy annotations on dashboards** | Correlate alert time with release — fastest rollback decision |
| **Synthetic alerts for edge failures** | Internal metrics green but DNS/CDN broken |
| **Escalation policy tested quarterly** | PagerDuty rotation gaps cause silent failures |
| **Post-incident alert review** | "Did we page too late / too early / not at all?" |

### When to use

- Production SLO violations and burn-rate thresholds
- Dependency failures affecting users
- Security and capacity threshold breaches (P2/P3 tier)
- Synthetic probe failures on critical user journeys

### Trade-offs

| Pros | Cons |
|------|------|
| Fast incident detection | Alert fatigue destroys trust |
| Automates vigilance | Flapping alerts waste time |
| Ties to on-call rotation | Poor thresholds → false positives |

| Pitfall | Symptom | Mitigation |
|---------|---------|------------|
| **Alert fatigue** | On-call ignores pages | Weekly tuning; delete non-actionable alerts |
| **Cause-based paging** | Pages on CPU spike with no user impact | Page on symptoms; causes as context |
| **Missing `for` duration** | Flap on every blip | `for: 5m` or burn-rate windows |
| **No runbook** | Responder guesses at 3 AM | Mandatory runbook link in alert template |
| **Alert sprawl** | 500 rules nobody owns | Alert budget; owner labels; quarterly audit |
| **Static thresholds** | Seasonal traffic triggers false positive | SLO burn-rate or anomaly detection |
| **Duplicate notifications** | 50 pages for one root cause | Alertmanager grouping + inhibition |
| **Untested escalation** | Page goes to ex-employee | Quarterly drill; rotation verification |

### References

- [Google SRE  -  Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/overview/)

---


## 9.13 Dashboards


### What is it

Visual compositions of panels (graphs, gauges, tables) displaying key metrics and logs for real-time and historical system overview.

### Why it matters

Dashboards provide at-a-glance health during incidents and planning sessions. They align teams on golden signals and reduce ad-hoc query friction under pressure.

### How it works

Tools like Grafana connect to metric and log data sources. Panels run PromQL or LogQL queries with templated variables (`$service`, `$region`). Hierarchy: global overview -> service dashboard -> instance drill-down.

### Diagram

```mermaid
flowchart TB
    subgraph Overview['Golden Signals Dashboard']
        L[Latency p99]
        T[Traffic RPS]
        E[Error Rate]
        S[Saturation]
    end
    Overview -->|drill-down| Svc[Service Dashboard]
    Svc --> Inst[Instance / Pod View]
```

### Key details

- One overview + one dashboard per critical service
- Chart decisions, not vanity metrics (total requests ever)
- Use consistent time ranges and annotation markers for deploys
- Mobile-friendly layouts for incident response

### When to use

- Daily operational health checks
- Incident war rooms
- Capacity review meetings

### Trade-offs

| Pros | Cons |
|------|------|
| Fast situational awareness | Stale dashboards mislead |
| Shared team context | Maintenance burden grows |
| Drill-down accelerates triage | Can hide problems behind averages |

### References

- [Grafana best practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)

---


## 9.14 Health Checks


### What is it

HTTP or TCP endpoints (or exec probes) that report whether a process is alive and whether it can accept traffic.

### Why it matters

Orchestrators and load balancers use health checks to restart failed containers and stop routing to unhealthy instances - automating recovery without human intervention.

### How it works

**Liveness** probes detect deadlocks - failure triggers restart. **Readiness** probes detect temporary unavailability (DB down) - failure removes instance from load balancer. **Startup** probes allow slow-starting apps extra time before liveness kicks in.

### Diagram

```mermaid
flowchart TB
    LB[Load Balancer] -->|ready only| Pod1[Pod Ready]
    LB -.->|not routed| Pod2[Pod Not Ready]
    K8s[Kubelet] -->|liveness fail| Restart[Restart Container]
```

### Key details

- Shallow: `/health` returns 200 if process up
- Deep: `/ready` checks DB, cache, downstream dependencies
- Keep liveness cheap - avoid cascading failures from deep checks
- Kubernetes: `livenessProbe`, `readinessProbe`, `startupProbe`

### When to use

- Container orchestration (Kubernetes, ECS)
- Load balancer target group health
- Dependency-gated traffic shifting (blue/green)

### Trade-offs

| Pros | Cons |
|------|------|
| Automated healing | Deep checks can flap during partial outages |
| Prevents bad traffic routing | Misconfigured probes cause restart loops |
| Standard K8s pattern | Probe latency affects rollout speed |

### References

- [Kubernetes probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)

---


## 9.15 Synthetic Monitoring


### What is it

Automated external probes that simulate user journeys - HTTP checks, browser scripts, API workflows - from outside your network on a schedule.

### Why it matters

Internal metrics can show green while users cannot reach the site (DNS, CDN, certificate issues). Synthetic monitoring detects user-visible outages before real traffic does.

### How it works

A scheduler runs probes every 1 - 5 minutes from multiple geographic locations. Failures trigger alerts. Results feed availability SLIs. Tools: Datadog Synthetics, Pingdom, Grafana k6 cloud, AWS CloudWatch Synthetics.

### Diagram

```mermaid
flowchart LR
    subgraph Regions
        R1[US-East Probe]
        R2[EU-West Probe]
        R3[APAC Probe]
    end
    R1 & R2 & R3 --> App[Public Endpoint]
    App --> SLI[Availability SLI]
    SLI --> Alert[Outage Alert]
```

### Key details

- Black-box: tests what users experience, not internal metrics
- Multi-region catches DNS/geo routing failures
- Critical path scripts: login, checkout, search
- Separate from load testing - low frequency, high signal

### When to use

- Public-facing APIs and web apps
- SLO availability measurement
- Certificate and DNS expiry monitoring

### Trade-offs

| Pros | Cons |
|------|------|
| User-perspective truth | Does not cover all code paths |
| Catches external failures | Scripted flows need maintenance |
| Geographic coverage | Can miss internal-only issues |

### References

- [Google SRE  -  SLI measurement](https://sre.google/workbook/implementing-slos/)

---

[<- Back to master index](../README.md)
