# 7. API Design

> Status: **Documented**

[<- Back to master index](../README.md)

---

## Overview

API design defines how services expose capabilities to clients and each other - shaping developer experience, evolvability, performance, and security. Modern systems choose among **REST** (resource-oriented HTTP), **GraphQL** (client-driven queries), **gRPC** (binary RPC over HTTP/2), and legacy **SOAP** based on audience, latency needs, and contract rigor.

Beyond protocol choice, production APIs need **gateways** for cross-cutting concerns (auth, rate limits, routing), consistent **pagination and filtering**, explicit **versioning**, and **idempotency** for safe retries. Poor API design creates coupling, outage amplification, and breaking changes that stall client teams.

This chapter covers protocol trade-offs, gateway patterns, query conventions, documentation (OpenAPI/Swagger), security, traffic control, and contract testing - everything needed to design and defend API decisions in system design interviews.

---

## Sub-topics

| # | Sub-topic | Status |
|---|-----------|--------|
| 7.1 | [REST](#71-rest) | Done |
| 7.2 | [GraphQL](#72-graphql) | Done |
| 7.3 | [gRPC](#73-grpc) | Done |
| 7.4 | [SOAP](#74-soap) | Done |
| 7.5 | [API Gateway](#75-api-gateway) | Done |
| 7.6 | [API Aggregation](#76-api-aggregation) | Done |
| 7.7 | [API Composition](#77-api-composition) | Done |
| 7.8 | [API Versioning](#78-api-versioning) | Done |
| 7.9 | [Pagination](#79-pagination) | Done |
| 7.10 | [Filtering](#710-filtering) | Done |
| 7.11 | [Sorting](#711-sorting) | Done |
| 7.12 | [OpenAPI](#712-openapi) | Done |
| 7.13 | [Swagger](#713-swagger) | Done |
| 7.14 | [Request Validation](#714-request-validation) | Done |
| 7.15 | [Contract Testing](#715-contract-testing) | Done |
| 7.16 | [API Security](#716-api-security) | Done |
| 7.17 | [Webhooks](#717-webhooks) | Done |
| 7.18 | [Rate Limiting](#718-rate-limiting) | Done |
| 7.19 | [Throttling](#719-throttling) | Done |
| 7.20 | [Idempotency](#720-idempotency) | Done |
| 7.21 | [Idempotency Keys](#721-idempotency-keys) | Done |





```mermaid
flowchart LR
    Client --> GW[API Gateway]
    GW --> REST[REST Service]
    GW --> GQL[GraphQL]
    GW --> GRPC[gRPC Backend]
```

---


---

## 7.1 REST


### What is it?

**REST** (Representational State Transfer) is an architectural style for networked APIs defined by Roy Fielding. It models the system as **resources** (nouns) identified by URLs, manipulated with **HTTP verbs**, exchanged as **representations** (JSON, XML), and governed by **stateless** request/response semantics and standard **status codes**.

REST is not a protocol — it is a set of constraints. A "RESTful" API uses HTTP as the application protocol and lets clients navigate via links (HATEOAS) when hypermedia is adopted.

### Why it matters

- **Universal tooling:** browsers, CDNs, proxies, load balancers, and every language have HTTP clients
- **Cache-friendly reads:** `GET` + `Cache-Control` / `ETag` reduce backend load at the edge
- **Predictable semantics:** verbs and status codes communicate intent without custom error enums
- **Interview default:** when asked "design a public API," REST is the baseline unless latency or client-flexibility demands GraphQL/gRPC

### How it works

**1. Resources (nouns, not verbs)**

Resources are things in your domain — not actions. URLs identify resources; HTTP verbs express actions on them.

| Good (resource-oriented) | Bad (RPC disguised as REST) |
|--------------------------|-----------------------------|
| `GET /users/42` | `POST /getUser` |
| `POST /orders` | `POST /createOrder` |
| `GET /orders/99/items` | `GET /fetchOrderItems?orderId=99` |

Naming conventions:

```text
/users              collection
/users/{id}         single resource
/users/{id}/orders  sub-collection (use when orders are owned by user)
/orders/{id}        prefer flat top-level when order is a first-class aggregate
```

**2. HTTP verbs and safety**

| Verb | Safe | Idempotent | Request body | Typical success code | Use |
|------|------|------------|--------------|---------------------|-----|
| `GET` | Yes | Yes | No | `200 OK` | Read resource or collection |
| `POST` | No | No | Yes | `201 Created` | Create; non-idempotent actions |
| `PUT` | No | Yes | Yes | `200 OK` / `204 No Content` | Full replace / upsert |
| `PATCH` | No | No* | Yes | `200 OK` | Partial update (JSON Patch, merge patch) |
| `DELETE` | No | Yes | Rare | `204 No Content` | Remove resource |

\*PATCH idempotency depends on patch semantics — design patches to be repeatable when possible.

**Safe** = no side effects on server state (client can prefetch). **Idempotent** = multiple identical requests have the same effect as one (safe to retry).

**3. Stateless**

Each request must contain everything the server needs: auth token, resource ID, representation. The server does **not** store client session state between requests (session data may live in a DB keyed by token, but the protocol is stateless).

```mermaid
flowchart LR
    C1[Client Request 1 / Authorization + body] --> API[REST API]
    C2[Client Request 2 / Authorization + body] --> API
    API --> DB[(Resource Store)]
```

Implications:

- No server-side "conversation context" — use URLs, headers, and body
- Horizontal scaling is trivial — any instance can serve any request
- Auth via `Authorization: Bearer <token>` or API key per request

**4. Representations and content negotiation**

A resource can have multiple representations. Client sends `Accept: application/json`; server returns JSON. Version via `Accept: application/vnd.myapi.v2+json` or URL prefix `/v2/users`.

**5. HATEOAS basics (Hypermedia as the Engine of Application State)**

HATEOAS embeds **links** in responses so clients discover next actions without hardcoding URLs:

```json
{
  "id": "ord_123",
  "status": "pending",
  "total": 4999,
  "_links": {
    "self": { "href": "/orders/ord_123" },
    "cancel": { "href": "/orders/ord_123/cancel", "method": "POST" },
    "pay": { "href": "/orders/ord_123/payments", "method": "POST" }
  }
}
```

In practice, many "REST" APIs skip full HATEOAS and use OpenAPI docs instead — interview answer: "HATEOAS is ideal for evolvable public APIs; most internal APIs use documented URL conventions."

**6. HTTP status codes (interview cheat sheet)**

| Code | Meaning | When to use |
|------|---------|-------------|
| `200 OK` | Success with body | `GET`, `PUT`, `PATCH` success |
| `201 Created` | Resource created | `POST` create; include `Location: /resources/{id}` |
| `204 No Content` | Success, no body | `DELETE`, some `PUT`/`PATCH` |
| `400 Bad Request` | Client malformed input | Validation failure |
| `401 Unauthorized` | Auth missing/invalid | No or bad token |
| `403 Forbidden` | Auth OK, not allowed | Insufficient scope/role |
| `404 Not Found` | Resource does not exist | Unknown ID (or hide existence with 404) |
| `409 Conflict` | State conflict | Duplicate unique key, version mismatch |
| `422 Unprocessable Entity` | Semantic validation | Business rule failure |
| `429 Too Many Requests` | Rate limited | With `Retry-After` header |
| `500 Internal Server Error` | Server bug | Never leak stack traces in prod |
| `502` / `503` / `504` | Gateway / overload / timeout | Retriable by client with backoff |

Use **problem+json** (`application/problem+json`) for consistent error bodies:

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Account balance 10.00 is less than charge 49.99",
  "instance": "/payments/pay_abc"
}
```

### Diagram

```mermaid
sequenceDiagram
    participant C as Client
    participant API as REST API
    participant CDN as "CDN / Cache"

    C->>CDN: GET /users/42 (If-None-Match: "etag-xyz")
    alt cache hit
        CDN-->>C: 304 Not Modified
    else cache miss
        CDN->>API: GET /users/42
        API-->>CDN: 200 + ETag + Cache-Control
        CDN-->>C: 200 JSON body
    end
    C->>API: POST /orders (Idempotency-Key: uuid)
    API-->>C: 201 Created + Location: /orders/ord_99
```

### Key details

- **Caching:** `GET` responses cacheable; `POST`/`PUT`/`DELETE` invalidate via keys or short TTL
- **Pagination:** `?cursor=abc&limit=20` on collections; never unbounded list endpoints
- **Partial updates:** prefer `PATCH` with JSON Merge Patch or JSON Patch over misusing `PUT`
- **Idempotency:** `PUT`/`DELETE` naturally idempotent; `POST` needs `Idempotency-Key` (see 7.21)

### When to use

- Public HTTP APIs, mobile/web clients, CDN-cacheable reads
- CRUD-heavy domains with clear resource boundaries
- Teams prioritizing simplicity, debuggability, and broad client support

### Trade-offs / Pitfalls

- **Over/under-fetching:** fixed resource shapes → multiple round trips or bloated payloads (BFF or GraphQL mitigate)
- **RPC-in-REST:** verbs in URLs (`/createUser`) lose HTTP semantics and cacheability
- **Wrong status codes:** returning `200` with `{ "error": true }` breaks clients and monitoring
- **Ignoring idempotency on POST:** retries after timeout cause duplicate side effects
- **Chatty clients:** N+1 calls without aggregation — fix with BFF, batch endpoints, or field expansion

### References

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Problem Details for HTTP APIs (RFC 9457)](https://www.rfc-editor.org/rfc/rfc9457)

---


## 7.2 GraphQL


### What is it?

**GraphQL** is a **query language** and **server runtime** where clients describe exactly what data they need in a single request. A **schema** defines types, fields, queries (reads), mutations (writes), and subscriptions (real-time). One HTTP endpoint (typically `POST /graphql`) serves all operations.

Unlike REST's fixed resource endpoints, GraphQL returns a **graph-shaped response** matching the client's query tree.

### Why it matters

- **Over-fetching:** REST `GET /user/123` returns all fields; mobile may only need `name` and `avatar`
- **Under-fetching:** REST needs 3 calls (`/user`, `/orders`, `/recommendations`); GraphQL nests in one round trip
- **Strong typing:** schema is contract + introspection powers GraphiQL, codegen, and validation
- **Evolving clients:** add fields without new endpoints; deprecate with `@deprecated`

### How it works

1. **Schema definition (SDL):**

```graphql
type User {
  id: ID!
  name: String!
  orders(first: Int): [Order!]!
}
type Query {
  user(id: ID!): User
}
type Mutation {
  updateUser(id: ID!, name: String!): User
}
```

2. **Client query** - only requested fields returned:

```graphql
query {
  user(id: "123") {
    name
    orders(first: 5) { id total }
  }
}
```

3. **Execution:** GraphQL engine parses query, validates against schema, calls **resolvers** per field
4. **N+1 problem:** naive resolver per order -> N DB calls; fix with **DataLoader** batching
5. **Mutations** for writes; **subscriptions** over WebSocket for push

```mermaid
flowchart TB
    Client -->|POST /graphql| GQL[GraphQL Server]
    GQL --> R1[User resolver]
    GQL --> R2[Order resolver - batched via DataLoader]
    R1 --> DB[(Database)]
    R2 --> DB
```

**Federation (Apollo):** multiple **subgraphs** (team-owned services) compose one **supergraph**; router delegates fields to subgraphs.

### Key details

| Topic | Detail |
|-------|--------|
| **Query complexity** | Assign cost per field; reject expensive queries (depth limit, complexity score) |
| **Caching** | No URL-level HTTP cache; use persisted queries, CDN for public read-only, or APQ |
| **Errors** | Partial success: `200` with `{ data, errors[] }` - monitor both |
| **Auth** | Field-level auth in resolvers; don't expose sensitive fields in schema |
| **vs REST** | GraphQL for flexible clients; REST for simple CRUD, CDN caching, public APIs |
| **vs BFF** | BFF is one backend per client; GraphQL is one schema many clients |

**DataLoader pattern:**

```text
orders(userId) called 50 times in one query
-> DataLoader batches: SELECT * FROM orders WHERE user_id IN (...)
-> returns Map userId -> orders to each resolver
```

### When to use

- Multiple clients (web, iOS, Android) with different data shape needs
- Aggregating microservices behind one graph (with federation)
- Rapid frontend iteration without backend deploy per screen change
- Internal admin tools with ad-hoc field selection

### Trade-offs / Pitfalls

- **DoS via expensive queries** - must enforce depth/complexity limits and rate limiting
- **Caching harder** than REST - whole query is POST body
- **File upload** needs multipart spec or separate REST endpoint
- **Error semantics** less crisp than HTTP status per resource
- **Operational complexity** - schema versioning, resolver performance profiling, N+1 debugging
- **Not for everything** - simple CRUD APIs are often better as REST

### References

- GraphQL specification (graphql.org); Apollo Federation docs

---


## 7.3 gRPC


### What is it?

**gRPC** is a high-performance **RPC framework** using **Protocol Buffers (protobuf)** for binary serialization over **HTTP/2**. Services define contracts in `.proto` files; code generators produce typed client/server stubs in Java, Go, Python, etc.

Four RPC types: **unary**, **server streaming**, **client streaming**, **bidirectional streaming**.

### Why it matters

Default choice for **internal microservice** communication where browsers are not direct clients:
- **3-10x smaller payloads** than JSON REST
- **Strong contracts** - breaking changes caught at compile time
- **HTTP/2 multiplexing** - many RPCs on one connection
- **First-class streaming** for logs, metrics, live feeds
- **Built-in deadlines**, metadata (headers), and status codes

### How it works

**1. Define contract (`user.proto`):**

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}
message User { string id = 1; string name = 2; }
```

**2. Codegen:** `protoc --go_out=... --grpc-go_out=...`

**3. Server** implements `UserServiceServer` interface; registers on gRPC server

**4. Client** calls stub: `client.GetUser(ctx, &GetUserRequest{Id: "123"})`

**5. HTTP/2 framing** underneath - single TCP connection, header compression (HPACK)

```mermaid
sequenceDiagram
    participant C as gRPC Client
    participant S as gRPC Server
    C->>S: Unary GetUser (protobuf)
    S-->>C: User message
    C->>S: Server-stream WatchOrders
    loop events
        S-->>C: Order event
    end
```

**Status codes:** `OK`, `INVALID_ARGUMENT`, `NOT_FOUND`, `DEADLINE_EXCEEDED`, `UNAVAILABLE` (mapped from HTTP-style semantics)

**Load balancing:** client-side (pick from resolver list) or **service mesh** (Envoy L7)

**grpc-web:** browser clients need proxy (Envoy, grpc-web) - browsers can't speak native gRPC

### Key details

| vs REST/JSON | gRPC |
|--------------|------|
| Payload | Binary protobuf (~compact) |
| Contract | `.proto` + codegen |
| Streaming | Native 4 modes |
| Browser | Needs grpc-web proxy |
| Debugging | grpcurl, reflection |
| Versioning | Add fields with new numbers; never reuse numbers |

**Protobuf field rules:**
- Field numbers are permanent; reserve deprecated numbers
- `optional`, `repeated`, `oneof` for schema evolution
- Unknown fields ignored (forward compatibility)

**Deadlines:** always set `ctx` timeout - `context.WithTimeout(ctx, 2*time.Second)` prevents hung calls

**mTLS:** mutual TLS for service identity in zero-trust networks (common with Istio)

### When to use

- Service-to-service sync calls in microservices
- High-throughput internal APIs (payments orchestration, catalog)
- Streaming: log tailing, price feeds, bidirectional chat backend
- Polyglot teams needing generated clients in 10+ languages

### Trade-offs / Pitfalls

- **Not human-readable** - need grpcurl/protobuf decoding for debugging
- **Browser limitation** - public APIs still often REST/GraphQL at edge, gRPC internal
- **Load balancer** must support HTTP/2 (L7) or use client-side LB
- **Breaking proto changes** (renumbering, type change) break all clients - use Buf breaking change detection
- **JSON transcoding** (gRPC-Gateway) adds hop for REST compatibility

### References

- gRPC.io documentation; Protocol Buffers language guide

---


## 7.4 SOAP


### What is it?

**SOAP** (Simple Object Access Protocol) is an XML-based messaging protocol with strict contracts (WSDL), WS-* standards (security, transactions), common in enterprise and legacy integrations.

### Why it matters

Still present in banking, telecom, and government integrations - understanding SOAP helps maintain and wrap legacy systems behind modern APIs.

### How it works

1. WSDL defines operations, types, and endpoint binding.
2. Client sends SOAP envelope (XML) over HTTP(S) or JMS.
3. Server validates against schema, executes operation, returns SOAP response.
4. WS-Security for signing/encryption; often ESB middleware.

### Diagram

```mermaid
sequenceDiagram
    participant C as SOAP Client
    participant S as SOAP Service
    C->>S: XML Envelope
    S-->>C: XML Response
```

### Key details

- Verbose XML payloads vs JSON/gRPC.
- Strong tooling in Java/.NET enterprise stacks.
- Often replaced by REST/gRPC with adapter layer.

### When to use

- Integrating with legacy enterprise systems requiring SOAP.
- Contract mandates (B2B partner WSDL).
- Not for greenfield public APIs.

### Trade-offs / Pitfalls

- High ceremony, poor developer ergonomics vs REST.
- Performance and payload size overhead.
- WS-* stack complexity and interoperability pain.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.5 API Gateway


### What is it?

An **API gateway** is a managed **reverse proxy** at the system edge — the single public entry point (`api.example.com`) that terminates TLS, authenticates callers, enforces rate limits, routes to internal services, and emits observability signals. Examples: Kong, AWS API Gateway, Azure API Management, Envoy Gateway, NGINX Plus.

The gateway is **infrastructure cross-cutting concern** — not a place for domain business rules.

### Why it matters

- **One front door:** clients never need internal hostnames or topology changes
- **Centralized security:** JWT validation, API keys, mTLS, WAF integration once at the edge
- **Operational control:** canary routing, blue/green, A/B tests without client redeploys
- **Cost efficiency:** rate limiting and auth at edge before expensive backend work
- **Interview framing:** "Gateway = edge policy; BFF = client-specific aggregation" (see below)

### How it works

**Request path through a gateway:**

```mermaid
flowchart TB
    Client[External Client] --> TLS[TLS Termination]
    TLS --> Auth[Authentication]
    Auth --> AuthZ[Authorization / Scopes]
    AuthZ --> RL[Rate Limit / Quota]
    RL --> Route[Path / Host Routing]
    Route --> Transform[Optional Transform]
    Transform --> S1[Users Service]
    Transform --> S2[Orders Service]
    Route --> Trace[Inject traceId / requestId]
```

**1. Authentication at the gateway**

| Mechanism | Gateway role | Backend receives |
|-----------|--------------|------------------|
| **JWT (Bearer)** | Validate signature via JWKS; check `exp`, `iss`, `aud` | Trusted headers (`X-User-Id`, `X-Scopes`) or forwarded JWT |
| **API key** | Lookup key in store; map to tenant/plan | `X-Tenant-Id`, rate-limit tier |
| **OAuth2 token** | Introspect at auth server or local JWKS verify | User identity claims |
| **mTLS** | Client cert validation at edge | Client SPIFFE ID / cert DN |

Gateway can **terminate auth** (backend trusts gateway headers on private network) or **pass-through** (backend re-validates — defense in depth).

```text
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
→ Gateway: verify RS256 signature, exp not passed
→ Forward: X-User-Sub: usr_42, X-Scopes: orders:read
```

**2. Routing**

Routing rules map external paths to internal services:

| Rule type | Example | Use case |
|-----------|---------|----------|
| Path prefix | `/v1/users/*` → `user-service:8080` | Versioned REST |
| Host | `mobile.api.example.com` → mobile BFF | Per-client hostname |
| Header | `X-Api-Version: 2` → v2 cluster | Header-based versioning |
| Weight | 90% v1, 10% v2 | Canary deployment |
| Method | `POST /payments` → payment cluster | Fine-grained rules |

```mermaid
flowchart LR
    GW[API Gateway]
    GW -->|/v1/users/*| US[User Service]
    GW -->|/v1/orders/*| OS[Order Service]
    GW -->|/v1/search/*| SS[Search Service]
```

**3. Rate limiting at the gateway**

Enforce per-API-key, per-IP, or per-tenant quotas **before** backends (see 7.18 for algorithms). Gateway is the ideal placement: shared Redis/DynamoDB counter, consistent `429` + `Retry-After`, plan-based tiers.

```text
Free tier:    100 req/min per API key
Pro tier:     10,000 req/min
/login:       5 req/min per IP (anti brute-force)
```

**4. API Gateway vs BFF — critical interview distinction**

| Dimension | API Gateway | BFF (Backend for Frontend) |
|-----------|-------------|----------------------------|
| **Primary job** | Edge security, routing, quotas, TLS | Client-specific aggregation and response shaping |
| **Audience** | All external clients uniformly | One client type (mobile, web, partner) |
| **Business logic** | None — policy only | Thin orchestration, field trimming |
| **Ownership** | Platform / infra team | Client or domain team |
| **Examples** | Kong, AWS API Gateway | Mobile BFF, Web BFF services |
| **Typical stack** | `Client → Gateway → Services` | `Client → Gateway → BFF → Services` |

```mermaid
flowchart TB
    Mobile --> GW[API Gateway]
    Web --> GW
    GW --> MobileBFF[Mobile BFF]
    GW --> WebBFF[Web BFF]
    MobileBFF --> Core[Core Microservices]
    WebBFF --> Core
```

**When both:** Gateway handles auth/rate-limit/TLS; BFF handles "mobile needs 3 fields, web needs 20" and parallel fan-out. Anti-pattern: putting aggregation logic in gateway plugins ("smart gateway").

**5. Additional gateway capabilities**

- Protocol translation: REST → gRPC, HTTP/1.1 → HTTP/2
- Request/response transformation: header injection, body mapping
- Observability: access logs, Prometheus metrics, W3C `traceparent` propagation
- WAF integration: OWASP CRS at edge

### Key details

| Capability | Benefit | Pitfall if misused |
|------------|---------|-------------------|
| TLS termination | Central cert rotation | End-to-end TLS needed for some compliance |
| JWT validation | Offload crypto from every service | Trust boundary — secure gateway→service link |
| Rate limiting | Protect backends | Per-instance limits without shared store fail in K8s |
| Path routing | Hide internal topology | Routing table becomes ops burden |
| Canary weights | Safe rollout | Sticky sessions may skew canary metrics |

Products: Kong, Apigee, AWS API Gateway, Azure APIM, Envoy Gateway, NGINX.

### When to use

- Multiple microservices exposed to external clients
- Centralized API keys, OAuth, and quota tiers
- Need canary/blue-green at edge without client changes
- Combine with BFF when client-specific shaping is required

### Trade-offs / Pitfalls

- **SPOF / bottleneck:** gateway must be HA (multi-AZ, autoscaling, health checks)
- **Smart gateway anti-pattern:** business rules in Lua/plugins — belongs in services or BFF
- **Latency hop:** ~1–5 ms per gateway layer; co-locate in same region/AZ as backends
- **Header trust:** if backend trusts `X-User-Id` from gateway, mTLS or private network required
- **Config sprawl:** hundreds of routes need GitOps / declarative config (Kong deck, Envoy xDS)

### References

- [AWS API Gateway patterns](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/latest/)

---


## 7.6 API Aggregation


### What is it?

**API aggregation** combines data from multiple backend services into **one response** for the client - reducing round trips (mobile on slow networks).

### Why it matters

Clients shouldn't orchestrate five REST calls per screen; aggregation improves latency and simplifies client code.

### How it works

1. Client calls aggregated endpoint (BFF or gateway plugin).
2. Aggregator fans out parallel requests to services A, B, C.
3. Waits for all (or partial with timeout/fallback).
4. Merges results into unified DTO.
5. Returns single JSON to client.

### Diagram

```mermaid
flowchart LR
    Client --> Agg[Aggregator BFF]
    Agg --> S1[Service A]
    Agg --> S2[Service B]
    Agg --> S3[Service C]
    Agg --> Client
```

### Key details

- Parallel `async` fetches minimize wall-clock latency.
- Partial failure: return degraded response vs fail entire request.
- Caching aggregated responses reduces backend load.

### When to use

- Mobile/web screens needing data from many domains.
- Public API simplifying partner integration.
- GraphQL resolvers perform aggregation implicitly.

### Trade-offs / Pitfalls

- Aggregator couples to backend schemas - changes ripple.
- Tail latency = slowest dependency; set per-call deadlines.
- Without caching, aggregator amplifies load on backends.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.7 API Composition


### What is it?

**API composition** (choreographed aggregation) builds a response by **sequentially** calling services when later calls depend on earlier results - e.g., get user, then orders for that user.

### Why it matters

Differs from parallel aggregation; necessary when data dependencies exist. Common in saga-style read paths and GraphQL resolver chains.

### How it works

1. Receive client request.
2. Call service A -> extract ID needed for B.
3. Call service B with ID from A.
4. Optionally call C with combined context.
5. Compose final response; handle any step failure.

### Diagram

```mermaid
sequenceDiagram
    participant BFF as BFF
    participant U as Users
    participant O as Orders
    BFF->>U: GET user
    U-->>BFF: user_id
    BFF->>O: GET orders(user_id)
    O-->>BFF: orders list
```

### Key details

- Latency sums across sequential hops - minimize chain depth.
- Cache intermediate results when keys repeat.
- Circuit breakers per hop prevent cascade.

### When to use

- Dependent data fetches unavoidable in business flow.
- Server-side alternative to client-side waterfall requests.

### Trade-offs / Pitfalls

- Deep chains fragile - prefer event-carried state or denormalized read models.
- Error in step 2 wastes step 1 work - compensate or return partial.
- Testing requires mocking full chain.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.8 API Versioning


### What is it?

**API versioning** manages breaking changes without stranding existing clients - via URL path (`/v2/`), headers (`Accept-Version`), query param, or content negotiation.

### Why it matters

Clients update on different schedules; breaking changes without versioning cause production outages for integrators.

### How it works

1. Define compatibility policy: additive changes only within major version.
2. Introduce `/v2` when breaking (field removal, semantic change).
3. Run v1 and v2 in parallel during migration window.
4. Deprecation headers (`Sunset`, `Deprecation`) signal timeline.
5. Retire old version after metrics show zero traffic.

### Diagram

```mermaid
flowchart LR
    ClientOld --> V1[/v1/users]
    ClientNew --> V2[/v2/users]
    V1 --> Svc[User Service]
    V2 --> Svc
```

### Key details

| Strategy | Pros | Cons |
|----------|------|------|
| URL path | Obvious, cacheable | URL proliferation |
| Header | Clean URLs | Harder to test manually |
| Separate deploy | Full isolation | Ops overhead |

### When to use

- Public APIs with external consumers.
- Any breaking schema or behavior change.

### Trade-offs / Pitfalls

- Maintaining N versions doubles test matrix.
- "Version everything" for internal APIs may be overkill - use compatibility discipline instead.
- Forgetting sunset dates leaves eternal v1 debt.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.9 Pagination


### What is it?

**Pagination** splits large result sets into pages - via **offset/limit**, **cursor/keyset**, or **seek** methods - protecting server and client from unbounded responses.

### Why it matters

Returning 1M rows crashes browsers and databases; pagination is mandatory for list APIs at scale.

### How it works

**Offset pagination:**

1. Client requests `?offset=100&limit=20`.
2. Server runs `LIMIT 20 OFFSET 100`.
3. Returns data + total count (optional).

**Cursor pagination:**

1. Server returns opaque cursor with page.
2. Client requests `?cursor=abc&limit=20`.
3. Server queries `WHERE id > last_id ORDER BY id LIMIT 20`.

### Diagram

```mermaid
flowchart LR
    Req[Request page 3] --> API
    API --> DB[(Database)]
    DB --> Page[20 items + next_cursor]
```

### Key details

- Offset: simple but slow/deep pages (`OFFSET` scans rows).
- Cursor: stable under concurrent inserts; no random page jump.
- Include `has_next`, `next_cursor` in response contract.

### When to use

- Cursor: infinite scroll, high-churn feeds, large tables.
- Offset: admin UIs with page numbers, small datasets.

### Trade-offs / Pitfalls

- Offset pagination inconsistent if rows inserted/deleted during browsing.
- Omitting max `limit` allows `limit=999999` abuse.
- Total count queries expensive - make optional.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.10 Filtering


### What is it?

**Filtering** lets clients narrow collections with query parameters - `?status=active&role=admin` - translated to safe database/API predicates.

### Why it matters

Clients need subsets without downloading full collections; filtering must be indexed and validated to stay performant.

### How it works

1. Define allowed filter fields whitelist in API contract.
2. Parse query params -> AST or SQL WHERE clauses (parameterized).
3. Reject unknown fields with 400.
4. Ensure DB indexes match common filter combinations.
5. Document operators: `eq`, `gt`, `in`, `like` (RSQL/FIQL patterns).

### Diagram

```mermaid
flowchart LR
    Q['?status=open&priority=high'] --> API
    API --> Valid[Validate whitelist]
    Valid --> DB[(Indexed query)]
```

### Key details

- Never concatenate user input into SQL - parameterized queries only.
- Composite indexes for multi-filter queries.
- GraphQL: arguments on list fields serve same role.

### When to use

- Search/list APIs with varied client slice needs.
- Admin dashboards and reporting endpoints.

### Trade-offs / Pitfalls

- Unindexed filters cause full table scans.
- Overly flexible filter DSL enables expensive queries - complexity limits needed.
- Filter + sort + pagination interaction must be documented.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.11 Sorting


### What is it?

**Sorting** orders results by one or more fields - `?sort=created_at:desc,name:asc` - with whitelist validation and index backing.

### Why it matters

Consistent ordering required for cursor pagination and user-visible lists; arbitrary sort fields are a common DoS vector.

### How it works

1. Client specifies `sort` parameter with field and direction.
2. API validates against allowed sort fields.
3. Translate to `ORDER BY` with parameter binding.
4. Combine with pagination key (sort field often = cursor field).
5. Default sort when omitted (e.g., `created_at desc`).

### Diagram

```mermaid
flowchart LR
    Sort['sort=-price'] --> API
    API --> IDX[(Index on price)]
    IDX --> Ordered[Ordered page]
```

### Key details

- Multi-column sort order matters for stable pagination.
- Sorting on unindexed JSON fields is slow.
- Locale-aware string sort needs explicit collation rules.

### When to use

- Any list API where order matters to users.
- Cursor pagination requires deterministic sort key (ideally unique).

### Trade-offs / Pitfalls

- Sorting + filtering on different indexes -> planner may fail to use index.
- Changing default sort is breaking for cursor clients.
- Random sort (`ORDER BY RAND()`) doesn't scale.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.12 OpenAPI


### What is it?

**OpenAPI Specification (OAS)** is a machine-readable YAML/JSON format describing REST API paths, parameters, request/response schemas, and security schemes - language-agnostic contract.

### Why it matters

Single source of truth for codegen, documentation, mock servers, contract tests, and gateway import - API-first design enabler.

### How it works

1. Author `openapi.yaml` (design-first) or generate from code annotations.
2. Validate spec in CI (Spectral lint rules).
3. Generate server stubs or client SDKs.
4. Publish to portal (Redoc, Stoplight).
5. Gate deployments on breaking change detection (oasdiff).

### Diagram

```mermaid
flowchart LR
    Spec[openapi.yaml] --> Gen[Code Generator]
    Spec --> Doc[Documentation]
    Spec --> Test[Contract Tests]
```

### Key details

- OpenAPI 3.1 aligns with JSON Schema.
- Reusable components: `schemas`, `parameters`, `responses`.
- `examples` and `description` fields improve consumer UX.

### When to use

- REST APIs with external or multi-team consumers.
- CI/CD SDK generation pipeline.
- API review process before implementation.

### Trade-offs / Pitfalls

- Spec drift from implementation if not generated from code in CI.
- Complex APIs produce huge specs hard to review.
- Doesn't cover gRPC (use protobuf) or GraphQL (use SDL).

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.13 Swagger


### What is it?

**Swagger** is the tooling ecosystem around OpenAPI - Swagger UI (interactive docs), Swagger Editor, and historical name for the spec before OpenAPI 3 rebranding.

### Why it matters

De facto interactive API explorer during development; stakeholders try endpoints without Postman setup.

### How it works

1. Host OpenAPI spec at `/openapi.json`.
2. Swagger UI renders try-it-out forms per operation.
3. OAuth2 flows configured for authenticated tryouts.
4. Editor validates syntax live during design sessions.

### Diagram

```mermaid
flowchart LR
    Spec[OpenAPI Spec] --> UI[Swagger UI]
    UI --> Dev[Developer try-out]
```

### Key details

- Swagger UI vs Redoc: UI interactive, Redoc prettier read-only.
- Don't expose Swagger UI in production without auth (info disclosure).
- Codegen: OpenAPI Generator (successor to swagger-codegen).

### When to use

- Dev/staging API documentation and manual testing.
- Onboarding partners with live contract explorer.

### Trade-offs / Pitfalls

- Production exposure leaks full API surface to attackers.
- "Try it out" against prod risky - disable or protect.
- Confusion between Swagger 2.0 and OpenAPI 3.x feature sets.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.14 Request Validation


### What is it?

**Request validation** checks incoming payloads against schema - types, required fields, formats, bounds - before business logic runs, returning `400 Bad Request` with field-level errors.

### Why it matters

First defense against injection, malformed data, and confusing 500 errors; shifts failures left with clear client feedback.

### How it works

1. Define JSON Schema / OpenAPI / bean validation rules.
2. Middleware validates body, query, path params on entry.
3. Reject with structured error: `{ "field": "email", "error": "invalid format" }`.
4. Pass validated DTO to handler - no raw map parsing in business code.
5. Validate content-type and payload size limits.

### Diagram

```mermaid
flowchart LR
    Req[Request] --> Val[Validator]
    Val -->|ok| Handler[Business Logic]
    Val -->|fail| E400[400 Error]
```

### Key details

- Frameworks: Jakarta Validation, Pydantic, JSON Schema middleware.
- Whitelist unknown fields vs strip (fail-closed preferred for APIs).
- Consistent error envelope across all endpoints.

### When to use

- Every production API boundary - non-negotiable baseline.
- Especially public APIs with untrusted clients.

### Trade-offs / Pitfalls

- Validation only at edge; internal service calls still need trust boundaries.
- Overly leaky validation messages aid attackers (user enumeration).
- Divergence between OpenAPI spec and runtime validation rules.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.15 Contract Testing


### What is it?

**Contract testing** verifies consumer and provider agree on API shape and behavior without full end-to-end tests - consumer-driven contracts (Pact) or schema validation against OpenAPI.

### Why it matters

Microservices break when provider changes response without notice; contract tests catch incompatibilities in CI before deploy.

### How it works

**Pact (consumer-driven):**

1. Consumer test defines expected request/response mock (pact file).
2. Pact file published to broker.
3. Provider CI verifies it can satisfy all consumer pacts.
4. Breaking change fails provider build before production.

### Diagram

```mermaid
flowchart LR
    Consumer --> Pact[Pact File]
    Pact --> Broker[Pact Broker]
    Broker --> Provider[Provider Verify]
```

### Key details

- Consumer-driven: consumers define needs; avoids gold-plated provider APIs.
- OpenAPI diff: provider spec vs consumer expectations.
- Not replacement for E2E - tests contract slice only.

### When to use

- Many microservices with independent deploy cycles.
- Public APIs with external consumers publishing pacts.
- Preventing "works on my machine" integration failures.

### Trade-offs / Pitfalls

- Pact maintenance overhead for large consumer counts.
- Contracts don't test latency, auth integration, or side effects.
- False confidence if provider verifies against stale pacts.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.16 API Security


### What is it?

**API security** encompasses authentication (who), authorization (what), transport encryption, input sanitization, rate limiting, and audit logging for API endpoints.

### Why it matters

APIs are the primary attack surface - OWASP API Security Top 10 (BOLA, broken auth, unbounded consumption) targets API-specific failures.

### How it works

1. **TLS 1.2+** for all traffic; mTLS for service-to-service.
2. **OAuth2/OIDC + JWT** for user delegation; API keys for partners.
3. **Scope/role checks** per endpoint (RBAC/ABAC).
4. Validate object ownership (prevent BOLA/IDOR).
5. Rate limit, WAF, audit sensitive operations.

### Diagram

```mermaid
flowchart TB
    Client --> TLS[TLS]
    TLS --> Auth[JWT Validate]
    Auth --> AuthZ[Authorization]
    AuthZ --> API[Handler]
```

### Key details

| Threat | Mitigation |
|--------|------------|
| BOLA | Check resource ownership |
| Broken auth | Short-lived tokens, rotation |
| Injection | Parameterized queries, validation |
| Excessive data | Field-level authz |

### When to use

Always - from design phase, not bolted on after launch.

### Trade-offs / Pitfalls

- JWT in localStorage -> XSS theft; prefer HttpOnly cookies or short-lived tokens.
- API keys in repos - use secret managers.
- Security theater: auth without authorization checks per object.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.17 Webhooks


### What is it?

**Webhooks** are HTTP callbacks: when an event occurs, the API **POSTs** a payload to a subscriber-configured URL - inverse of polling.

### Why it matters

Real-time integrations (Stripe payments, GitHub pushes) without client polling overhead; standard pattern for SaaS extensibility.

### How it works

1. Subscriber registers URL + secret via API.
2. Event occurs (payment succeeded).
3. Provider signs payload (HMAC-SHA256 with secret).
4. POST to subscriber URL with retry backoff on failure.
5. Subscriber verifies signature, returns 2xx quickly, processes async.

### Diagram

```mermaid
sequenceDiagram
    participant P as Provider
    participant S as Subscriber
    P->>S: POST event + signature
    S-->>P: 200 OK
    S->>S: async process
```

### Key details

- Idempotent processing by `event_id`.
- Exponential retry for days; DLQ for dead endpoints.
- Challenge verification on URL registration (echo token).

### When to use

- Push notifications to partner systems.
- Event-driven integrations without message bus access.

### Trade-offs / Pitfalls

- Subscriber downtime -> retry queues backlog at provider.
- SSRF risk if provider fetches user-supplied URLs - validate allowlists.
- Ordering not guaranteed - use sequence numbers.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.18 Rate Limiting


### What is it?

**Rate limiting** controls how much traffic a client or service may send over a time window. It caps the number of requests allowed per user, IP, API key, or tenant; when the threshold is exceeded, extra requests are **rejected** (typically HTTP `429 Too Many Requests`) or **queued/throttled**.

Rate limiting can live at different layers:
- **Client-side** - SDK backoff (cooperative, not security)
- **Server-side** - inside each app instance (weak in clusters unless shared state)
- **Middleware / API gateway** - recommended: one enforcement point before backends (Kong, Envoy, nginx, AWS API Gateway, custom Spring filter)

### Why it matters

- **DoS and abuse protection** - bots and misbehaving clients cannot exhaust CPU, DB connections, or memory
- **Fair multi-tenancy** - one noisy neighbor cannot starve others on shared infrastructure
- **Cost control** - expensive endpoints (search, ML inference) stay within budget
- **SLA tiers** - free vs paid plans get different quotas (`100 req/min` vs `10,000 req/min`)
- **Brute-force mitigation** - login and OTP endpoints get strict per-IP limits

### How it works

**Placement in request path:**

```mermaid
flowchart LR
    Client --> GW[API Gateway / Middleware]
    GW --> RL{Rate limit check}
    RL -->|allowed| API[Backend services]
    RL -->|denied| R429[429 + Retry-After]
```

**1. Token bucket**

- Bucket has capacity **b** (max tokens) and refill rate **r** tokens/second
- Each request consumes **1 token**; if tokens available -> allow; else -> reject
- Tokens stop refilling when bucket is full (overflow discarded)
- **Per-client bucket** - typically one bucket per API key, user ID, or IP + endpoint

Example: `b=10`, `r=2/sec` -> burst of 10 requests instantly, then steady 2/sec.

**2. Leaky bucket**

- Requests enter a **FIFO queue** with max size **b**
- Processor drains queue at **fixed outflow rate** (e.g. 5 req/sec)
- Queue full -> drop new request
- Output is smooth (good for protecting downstream with fixed capacity); responses may feel async/delayed

**3. Fixed window counter**

- Timeline split into windows (e.g. 1 minute); counter per window per client
- Each request increments counter; if counter > limit -> reject until window resets
- **Edge burst problem:** limit 3/min -> 3 requests at `2:00:59` + 3 at `2:01:00` = **6 in 2 seconds** across window boundary

**4. Sliding window log**

- Store **timestamp of every request** in the lookback window (Redis sorted set is common)
- On new request: remove timestamps older than `now - window`; count remaining; if count < limit -> allow and add timestamp
- Most accurate; higher memory (stores every request time even for rejected attempts in some designs)

**5. Sliding window counter (hybrid - production favorite)**

Combines fixed windows with smoothing:

```
weighted_count = (prev_window_count * (1 - overlap_fraction)) + current_window_count
```

If `weighted_count > limit` -> reject.

Example: limit **4 req/min**. Previous window had 4 requests; current window (25% elapsed) has 2.  
`weighted = 4 * (1 - 0.25) + 2 = 5` -> **reject** even though neither window alone exceeded 4.

**Distributed implementation with Redis (atomic):**

For each client key `rate:{userId}`:

1. `ZREMRANGEBYSCORE` - remove entries older than `now - windowMs`
2. `ZADD` - add current timestamp as score and member
3. `EXPIRE` - TTL = window size (cleanup)
4. `ZRANGE 0 -1` - count entries in window
5. Wrap steps 1-4 in **`MULTI`/`EXEC`** so no race between app instances

Return `429` when count > `maxRequests`; include headers:

| Header | Purpose |
|--------|---------|
| `X-RateLimit-Limit` | Max requests in window |
| `X-RateLimit-Remaining` | Tokens/requests left |
| `X-RateLimit-Reset` | Unix time when window resets |
| `Retry-After` | Seconds client should wait |

Pseudo-flow:

```text
allowed = redis.transaction {
  zremrangebyscore(key, 0, now - windowMs)
  zadd(key, now, now)
  expire(key, windowMs)
  count = zcard(key)
  return count <= maxRequests
}
```

### Key details

| Algorithm | Burst behavior | Memory | Distributed-friendly | Best for |
|-----------|----------------|--------|-------------------|----------|
| Token bucket | Allows bursts up to bucket size | O(1) per client | Yes (Redis INCR + TTL) | Public APIs, Stripe-style quotas |
| Leaky bucket | Smooth constant outflow | O(queue size) | Harder (needs queue) | Protecting fixed-capacity workers |
| Fixed window | Boundary spikes | O(1) | Yes (INCR + key per window) | Simple internal limits |
| Sliding window log | Accurate | O(requests in window) | Yes (Redis ZSET) | Strict per-minute limits |
| Sliding window counter | Smooth, approximate | O(1)-O(2) windows | Yes | **High-scale production APIs** |

- **Identity for limiting:** prefer API key or user ID over raw IP (corporate NAT shares one IP)
- **Granularity:** global limit + per-endpoint limit (e.g. `1000/min` overall, `10/min` on `/search`)
- **Fail-open vs fail-closed:** if Redis down, allow traffic (availability) or deny (safety) - product decision
- **429 body:** return structured JSON with `retryAfter` so clients back off correctly

### When to use

- All **public and partner APIs**
- Login, password reset, OTP (anti brute-force)
- Expensive operations: GraphQL complexity limits, search, file upload
- Internal microservices calling shared DB or third-party APIs with quotas

### Trade-offs / Pitfalls

- **Per-instance counters** in a cluster multiply effective limit by instance count - always use shared store (Redis, DynamoDB) for distributed limits
- **Shared NAT / corporate IP** blocks innocent users - combine IP + API key or use authenticated identity
- **Fixed window** allows 2x burst at boundaries - use sliding window counter or token bucket instead
- **429 without `Retry-After`** causes clients to hammer harder
- **Leaky bucket** can starve recent requests if queue fills with old ones during a spike
- Rate limit != throttling: rate limit **rejects**; throttling **slows/queues** (see 7.19)

### References

- [Rate Limiting - Hareram Singh (algorithms + Redis sliding window)](https://medium.com/@hareramcse/rate-limiting-cc6702fed0ed)

---


## 7.19 Throttling


### What is it?

**Throttling** slows or queues requests when load exceeds capacity - graceful degradation vs hard reject (rate limit). May delay responses or shed low-priority traffic.

### Why it matters

Keeps system alive under overload - returns slower responses instead of cascading failures and OOM crashes.

### How it works

1. Monitor queue depth, CPU, or error rate.
2. When threshold exceeded, apply delay or priority queue.
3. Premium tenants bypass or get dedicated capacity.
4. Combine with autoscaling when delay insufficient.
5. Communicate degraded mode via headers/monitoring.

### Diagram

```mermaid
flowchart LR
    Load[High Load] --> Queue[Request Queue]
    Queue --> Delay[Added latency]
    Delay --> API[Stable throughput]
```

### Key details

- Throttling = backpressure at API layer.
- Adaptive concurrency (AIMD) in load balancers.
- Different from rate limit: system protection vs per-client quota.

### When to use

- Sudden traffic spikes beyond provisioned capacity.
- Protecting shared databases during incidents.
- Tiered SLA: throttle free tier before paid.

### Trade-offs / Pitfalls

- Queues increase tail latency - clients may timeout anyway.
- Without autoscaling, throttling masks capacity debt.
- User frustration if throttling opaque - return meaningful status.

### References

*(No curated references for this sub-topic in `_topics.json`.)*

---


## 7.20 Idempotency


### What is it?

An operation is **idempotent** if executing it **once or many times** produces the **same system state** and the client can treat duplicate responses equivalently. In distributed APIs, idempotency is what makes **safe retries** possible after network timeouts, gateway `504`s, and message redelivery.

**Natural HTTP idempotency:**

| Method | Idempotent? | Notes |
|--------|-------------|-------|
| `GET`, `HEAD`, `OPTIONS` | Yes | Safe reads |
| `PUT` | Yes | Same body → same resource state |
| `DELETE` | Yes | Second delete → `404` or `204` (both acceptable) |
| `PATCH` | Depends | Design patch to be repeatable |
| `POST` | **No** | Creates new resource each call unless you add idempotency |

### Why it matters

Clients **cannot distinguish** "request failed" from "request succeeded but response lost":

```mermaid
sequenceDiagram
    participant C as Client
    participant API as API
    C->>API: POST /charges (payment)
    Note over API: Payment succeeds
    Note over C,API: Network drops response
    C->>API: RETRY POST /charges
    Note over API: Without idempotency -> DOUBLE CHARGE
```

Without idempotency: duplicate orders, double charges, repeated emails, inventory oversell.

### How it works

**Pattern 1 — Natural idempotency (PUT/DELETE)**

```http
PUT /accounts/acc_42/balance
Content-Type: application/json

{ "balance": 1000 }
```

Retrying the same `PUT` overwrites with identical state — safe.

**Pattern 2 — Business idempotency key (POST)**

Client supplies unique key per logical operation; server deduplicates:

```text
1. Client generates key once per user action (UUID v4)
2. POST /orders with Idempotency-Key: 7f3a-...
3. Server atomically: INSERT idempotency_record OR return cached response
4. On timeout, client retries SAME key → same outcome
```

**Pattern 3 — Database unique constraint**

```sql
CREATE UNIQUE INDEX idx_payments_external_id ON payments(external_id);
-- Retry INSERT with same external_id → conflict → return existing row
```

**Pattern 4 — Message consumer dedup**

At-least-once delivery (Kafka, SQS) requires consumers to check `message_id` before processing side effects.

**Server-side state machine:**

```mermaid
stateDiagram-v2
    [*] --> CheckKey: POST arrives
    CheckKey --> Processing: key not seen
    CheckKey --> ReplayCached: key seen (completed)
    CheckKey --> WaitOrConflict: key seen (in-flight)
    Processing --> StoreResult: success
    StoreResult --> [*]: return response
    ReplayCached --> [*]: return stored response
    WaitOrConflict --> [*]: 409 or poll
```

**Idempotency record contents:**

| Field | Purpose |
|-------|---------|
| `idempotency_key` | Client-supplied unique token |
| `request_hash` | Detect same key + different body → `409 Conflict` |
| `status` | `processing` / `completed` / `failed` |
| `response_code` | HTTP status to replay faithfully |
| `response_body` | Serialized response for replay |
| `created_at` / `expires_at` | TTL (Stripe: 24 hours) |

### Key details

- **Scope:** key unique per **account/tenant** — `user_42 + key` prevents cross-tenant collision
- **Concurrency:** use DB unique constraint or Redis `SETNX` to win exactly one processor
- **Side effects beyond DB:** email/SMS need outbox pattern + dedup by `event_id`
- **Failed operations:** decide if failed attempt allows retry with same key (Stripe: yes, returns same error)
- **Time window:** 24h–72h typical; after expiry, same key may create new resource

### When to use

- Payment, order creation, inventory reservation, subscription signup
- Any `POST` over unreliable networks (mobile, cross-region)
- Webhook handlers and queue consumers (at-least-once delivery)
- Partner APIs where clients implement exponential backoff retries

### Trade-offs / Pitfalls

- **Storage growth:** every mutating request stores a record — TTL and archival required
- **Race on concurrent duplicates:** two requests same key simultaneously — need atomic claim (`INSERT ... ON CONFLICT` or distributed lock)
- **Different status on replay:** first call returns `201`, replay must also return `201` (not `200`) — store full response
- **Partial failure:** payment succeeded, order update failed — saga + idempotent steps required
- **Assuming GET is always safe:** `GET` with side effects (trigger export) breaks idempotency semantics

### References

- [Stripe idempotency documentation](https://docs.stripe.com/api/idempotent_requests)
- [RFC 9110 — HTTP method properties](https://www.rfc-editor.org/rfc/rfc9110#name-method-properties)

---


## 7.21 Idempotency Keys


### What is it?

An **idempotency key** is a client-generated unique token (typically **UUID v4**) sent in a header on mutating requests — most commonly `Idempotency-Key` — so the server recognizes retries and returns the **original result** without re-executing side effects.

The **Stripe pattern** is the industry reference implementation for payment APIs.

### Why it matters

The canonical retry scenario:

```text
Client POST /v1/charges → server processes → response lost (504 Gateway Timeout)
Client MUST retry with SAME Idempotency-Key
Server MUST NOT charge twice
```

Without keys, `POST` is unsafe to retry — clients either duplicate operations or fail open on errors.

### How it works — Stripe pattern

**Client rules:**

1. Generate **one UUID per logical user action** (button click), not per HTTP attempt
2. Send on every attempt including retries:

```http
POST /v1/payment_intents
Authorization: Bearer sk_live_...
Idempotency-Key: 7f3a8b2c-4e1d-4a9f-b3c2-1d8e9f0a2b3c
Content-Type: application/json

{ "amount": 4999, "currency": "usd", "customer": "cus_123" }
```

3. On `409 Conflict` (same key, different body) — fix client bug, do not retry blindly
4. On `429` / `503` — backoff and retry **same key**

**Server rules (Stripe-compatible):**

```mermaid
sequenceDiagram
    participant C as Client
    participant API as API
    participant DB as Idempotency Store

    C->>API: POST Idempotency-Key: uuid-1 (attempt 1)
    API->>DB: INSERT key=uuid-1, status=processing
    API->>API: Execute payment
    API->>DB: UPDATE status=completed, store response
    Note over C,API: Response lost - client retries
    C->>API: POST Idempotency-Key: uuid-1 (attempt 2)
    API->>DB: SELECT key=uuid-1
    DB-->>API: completed + cached response
    API-->>C: 200 same body (no re-charge)
```

| Scenario | Server behavior |
|----------|-----------------|
| New key | Process request; store result |
| Known key, same request body | Return cached response (same HTTP status + body) |
| Known key, **different** body | `409 Conflict` — client misconfiguration |
| Known key, still `processing` | `409` or block until complete (Stripe waits) |
| Key expired (>24h) | Treat as new request |

**Implementation sketch (atomic claim):**

```sql
-- PostgreSQL example
INSERT INTO idempotency_keys (key, account_id, request_hash, status)
VALUES ($1, $2, $3, 'processing')
ON CONFLICT (key, account_id) DO NOTHING
RETURNING id;

-- If no row returned → lookup existing and replay or 409
```

Redis alternative: `SET idem:{account}:{key} processing NX EX 86400` — only winner processes.

**Request hash for conflict detection:**

```text
hash = SHA256(method + path + canonical_json_body)
same key + different hash → 409 Conflict
```

### Key details

| Topic | Recommendation |
|-------|----------------|
| Header name | `Idempotency-Key` (Stripe); also seen: `X-Idempotency-Key` |
| Key format | UUID v4 — unguessable, no sequential IDs |
| Retention | 24h (Stripe default); document in API spec |
| Response replay | Store **exact** status code + headers + body |
| Scope | Per authenticated account/tenant |
| GET/PUT | Usually unnecessary — methods already idempotent |

**Interview one-liner:** "Idempotency keys turn unsafe `POST` retries into safe at-least-once delivery, same as message dedup in event systems."

### When to use

- Payment and money movement APIs (mandatory)
- Order/subscription/resource creation where duplicate is unacceptable
- Any partner API documenting retry-on-`5xx` behavior
- Internal microservices with retry policies on mutating calls

### Trade-offs / Pitfalls

- **Client bug:** new key per retry → duplicates — SDKs should accept key from caller or auto-generate once per operation object
- **Distributed store consistency:** idempotency record must be written **before** side effect or in same transaction as outbox
- **Long-running operations:** `processing` state needs timeout and cleanup job
- **409 on in-flight:** clients need backoff, not parallel duplicate posts with same key
- **Leaking keys in logs:** treat like credentials in debug output

### References

- [Stripe — Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [PayPal idempotency header](https://developer.paypal.com/api/rest/reference/info-standards/#idempotency)

---


## Quick Reference

| # | Topic | Summary |
|---|-------|---------|
| 7.1 | REST | **REST** (Representational State Transfer) models APIs as **resources** (noun... |
| 7.2 | GraphQL | **GraphQL** is a query language and runtime where clients request exactly the... |
| 7.3 | gRPC | **gRPC** is a high-performance RPC framework using **Protocol Buffers** over ... |
| 7.4 | SOAP | **SOAP** (Simple Object Access Protocol) is an XML-based messaging protocol w... |
| 7.5 | API Gateway | An **API gateway** is a reverse proxy at the edge routing client requests to ... |
| 7.6 | API Aggregation | **API aggregation** combines data from multiple backend services into **one r... |
| 7.7 | API Composition | **API composition** (choreographed aggregation) builds a response by **sequen... |
| 7.8 | API Versioning | **API versioning** manages breaking changes without stranding existing client... |
| 7.9 | Pagination | **Pagination** splits large result sets into pages - via **offset/limit**, **cu... |
| 7.10 | Filtering | **Filtering** lets clients narrow collections with query parameters - `?status=... |
| 7.11 | Sorting | **Sorting** orders results by one or more fields - `?sort=created_at:desc,name:... |
| 7.12 | OpenAPI | **OpenAPI Specification (OAS)** is a machine-readable YAML/JSON format descri... |
| 7.13 | Swagger | **Swagger** is the tooling ecosystem around OpenAPI - Swagger UI (interactive d... |
| 7.14 | Request Validation | **Request validation** checks incoming payloads against schema - types, require... |
| 7.15 | Contract Testing | **Contract testing** verifies consumer and provider agree on API shape and be... |
| 7.16 | API Security | **API security** encompasses authentication (who), authorization (what), tran... |
| 7.17 | Webhooks | **Webhooks** are HTTP callbacks: when an event occurs, the API **POSTs** a pa... |
| 7.18 | Rate Limiting | **Rate limiting** caps requests per client/IP/API key over a time window - prot... |
| 7.19 | Throttling | **Throttling** slows or queues requests when load exceeds capacity - graceful d... |
| 7.20 | Idempotency | An operation is **idempotent** if performing it multiple times has the same e... |
| 7.21 | Idempotency Keys | An **idempotency key** is a client-generated unique token (UUID) sent in head... |

---

[Ã¢ - Â Back to master index](../README.md)
