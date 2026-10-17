# FD53 — System Architecture

FD53 is a proprietary backend-as-a-service platform. It provides auth, database,
storage, realtime and an API platform so an application team never writes a
server. This document is the technical blueprint for growing FD53 from its
current single-region implementation into a multi-region cloud platform.

---

## 1. Tenancy model

Everything in FD53 hangs off a **project**. A project is one backend — one
Flutter app, one web app, one internal tool.

```
users (console identity)
  └── projects              1 developer : N projects
        ├── apiKeys          server-side credentials (hashed)
        ├── tables           schema-on-write definitions
        │     └── rows       documents
        ├── endUsers         the app's real users
        │     ├── sessions   revocable JWT backing
        │     └── roles      RBAC permissions
        ├── files            storage objects in buckets
        ├── usageDaily       per-day, per-category counters
        ├── requestLogs      rolling per-request window
        └── auditLogs        append-only security evidence
```

Every query in the data plane is scoped by `projectId`. There is no code path
that reads a row without first resolving a project from a validated API key.
That is the primary isolation guarantee: **cross-project data leakage is
structurally impossible, not merely guarded by a check.**

---

## 2. Request pipeline

The API gateway (`src/convex/rest/gateway.ts`) is the single entry point. Every
`/v1` request passes through these stages in order. Nothing reaches a handler
without clearing all of them.

```
                      ┌──────────────────────────────────────────┐
   Client ───────────▶│  Cloudflare / CDN  ·  TLS  ·  HSTS        │
   (SDK / FlutterFlow) └──────────────────┬───────────────────────┘
                                          │
        ┌─────────────────────────────────▼──────────────────────────────┐
  1     │  Correlation      assign or honour X-FD53-Request-Id          │
  2     │  Fingerprint      client IP (hashed, never stored raw) + UA    │
  3     │  WAF              traversal · injection · oversized · bad verb │
  4     │  Origin           CORS resolved against the key's allowlist    │
  5     │  Edge throttle    per-IP flood limit — BEFORE any DB read      │
        ├──────────────────────────────���──────────────────────────────────┤
  6     │  API key          format → hash → revoked → expired → env     │
        │                    → project → suspended → IP policy → scope    │
  7     │  Quota            per-key rate limit + daily usage accounting  │
        ├────────────────────────────────────────────────────────────────┤
  8     │  Dispatch         route handler (auth / db / storage / …)     │
        ├────────────────────────────────────────────────────────────────┤
  9     │  Response         ok/error envelope + security headers        │
 10     │  Telemetry        requestLogs + auditLogs + Server-Timing      │
        └────────────────────────────────────────────────────────────────┘
```

### Why stages 1–5 precede authentication

A flood from an anonymous client must never reach the database, must never burn
a legitimate project's quota, and must be stoppable even if the attacker holds
a valid key. Stages 1–5 therefore run on the hashed client IP alone. This is
the difference between a platform that survives a `GET /v1/db/*` spray and one
that does not.

### Layered rate limiting

FD53 enforces four independent limiters. They answer different questions:

| Limiter | Keyed by | Question it answers | Where |
|---|---|---|---|
| Edge | hashed client IP | Is this network flooding us? | `edge.ts`, pre-auth |
| Auth-failure | hashed client IP | Is someone guessing keys? | `edge.ts`, bans the source |
| API key | `apiKeys._id` | Is this app over its quota? | `usage.ts` |
| Anti-spam | end user id | Is one user abusing the app? | `rateLimits.ts` |

One abusive end user cannot exhaust a shared app's quota, because the end-user
limiter runs in front of the per-key limiter. One flood cannot reach a handler
at all, because the edge limiter runs in front of key resolution.

### API key validation

Keys are `fd53_live_<40 hex>` or `fd53_test_<40 hex>`. FD53 stores only
`sha256(key)`; the plaintext is returned exactly once at creation. Validation
is ordered cheapest-first so a scanner spraying garbage costs one regex, not a
database read:

```
1. present?           → 401 missing_api_key          (no I/O)
2. well-formed?       → 401 invalid_api_key          (no I/O)
3. hash found?        → 401 invalid_api_key
4. revoked?           → 401 key_revoked
5. past expiry?       → 401 key_expired
6. environment match? → 401 key_environment_mismatch
7. project exists?    → 401 invalid_api_key
8. project active?    → 403 project_suspended
9. IP / CIDR policy   → 403 ip_not_allowed
10. scope granted?    → 403 insufficient_scope
```

Each failure has a distinct code, so a developer distinguishes a typo from a
revoked key from an IP restriction without reading server logs. All of them are
written to the audit trail.

---

## 3. Target deployment architecture

The current implementation runs on Convex, which is genuinely good at
serverless functions and reactive queries. It is *not* the right long-term home
for everything: SQL analytics, GraphQL federation, WebSocket fan-out and
video transcoding all want different primitives. The recommended end state is
six planes that scale independently.

```
                          ┌─────────────────────────────┐
                          │      Global Anycast DNS     │
                          │      + GeoDNS routing       │
                          └──────────────┬──────────────┘
                                         │
                         ┌───────────────▼───────────────┐
                         │   Edge PoP  (Cloudflare)      │
                         │   · TLS termination           │
                         │   · WAF + DDoS L3/L4          │
                         │   · static asset cache        │
                         │   · image resizing            │
                         │   · Argo Smart Routing        │
                         └───────────────┬───────────────┘
                                         │
                  ┌──────────────────────┴──────────────────────┐
                  │                                             │
        ┌─────────▼──────────┐                      ┌───────────▼─────────┐
        │  Region: us-east-1  │                      │  Region: eu-west-1  │
        │                     │                      │                     │
        │ ┌─────────────────┐ │                      │ ┌─────────────────┐ │
        │ │ API Gateway     │ │   stateless, scaled  │ │ API Gateway     │ │
        │ │ · authz         │ │◀────────────────────▶│ │ · authz         │ │
        │ │ · WAF           │ │                      │ │ · WAF           │ │
        │ │ · rate limit    │ │                      │ │ · rate limit    │ │
        │ └────────┬────────┘ │                      │ └────────┬────────┘ │
        │          │          │                      │          │          │
        │ ┌────────▼────────┐ │                      │ ┌────────▼────────┐ │
        │ │ Core Services   │ │                      │ │ Core Services   │ │
        │ │ auth · database │ │                      │ │ auth · database │ │
        │ │ storage · rt    │ │                      │ │ storage · rt    │ │
        │ └────────┬────────┘ │                      │ └────────┬────────┘ │
        │          │          │                      │          │          │
        │ ┌────────▼────────┐ │                      │ ┌────────▼────────┐ │
        │ │ Realtime Hubs   │◀┼──── pub/sub (NATS) ──▶│ Realtime Hubs   │ │
        │ │ WebSocket fanout│ │                      │ │ WebSocket fanout│ │
        │ └─────────────────┘ │                      │ └─────────────────┘ │
        └──────────┬──────────┘                      └──────────┬──────────┘
                   │                                             │
     ┌─────────────┼─────────────────────────────┬───────────────┘
     │             │                             │
┌────▼─────┐ ┌─────▼──────┐ ┌──────────────┐ ┌──▼─────────────┐
│ Postgres │ │   Redis    │ │  Object      │ │  Kafka /       │
│ primary  │ │ rate limit │ │  storage     │ │  Redpanda      │
│ + 2 read │ │ sessions   │ │  S3 + CDN    │ │  queues        │
│ replicas │ │ cache      │ │ images/video │ │  events        │
└────┬─────┘ └────────────┘ └──────────────┘ └──┬─────────────┘
     │                                           │
┌────▼──────────────────────────────┐  ┌─────────▼──────────┐
│ Analytics: ClickHouse / DuckDB   │  │ Workers            │
│ · usage · errors · latency       │  │ · media transcode  │
│ · read replicas, never on hot PG │  │ · email · SMS      │
└───────────────────────────────────┘  │ · webhooks · cron  │
                                       └────────────────────┘
```

### Plane-by-plane responsibility

| Plane | Responsibility | Scaling trigger | Why separate |
|---|---|---|---|
| Edge | TLS, WAF, DDoS, CDN, image resize | Traffic | Stateless, must never be the bottleneck |
| Gateway | Auth, quota, routing, audit | Request rate | Security boundary; scales horizontally |
| Core | Auth, database, storage, realtime APIs | CPU / connections | Business logic, long-lived state |
| Realtime | WebSocket fan-out, presence, typing | Concurrent sockets | Socket count ≠ request count |
| Workers | Media, email, SMS, webhooks, cron | Queue depth | Jobs are slow and bursty |
| Analytics | Usage, errors, latency | Log volume | Write-heavy; must not tax the hot path |

### Realtime hubs

Realtime connections are stateful and are the one component that cannot be
treated as a pure function. FD53 runs **regional hubs** with a shared pub/sub
backplane (NATS or Redis Streams):

- A client connects to the hub in its nearest region and holds one socket.
- A publish is written once and fanned out by the pub/sub backplane to every
  region that has subscribers on that channel.
- Presence and typing indicators are ephemeral and **never persisted** — they
  live in Redis with a short TTL.

The current build implements the realtime event log and a polling fallback
(`GET /v1/realtime/channels/:channel/events`), which is correct but not
low-latency. The hub is the documented next step.

---

## 4. Technology recommendations

### Language and runtime

**Go for the gateway and core services; Rust where latency is the product.**

| Component | Language | Rationale |
|---|---|---|
| API gateway | **Go** | Fast compile, trivial static binaries, small container images, excellent concurrency for request/response work. Rust's advantage does not show on an I/O-bound path. |
| Auth + crypto | **Rust** | Password hashing, JWT signing and the WAF's hot path run on untrusted input. Memory safety and predictable latency matter more than ecosystem breadth. |
| Workers | **Node.js (TypeScript)** | Media/email/SMS integrations have huge SDK ecosystems. Runtime cost is irrelevant next to a network call. |
| SDK codegen | **TypeScript** | Must run in the build pipeline, not in production. |

Avoid a polyglot sprawl: three languages, each chosen for a specific reason.

### Data stores

| Store | Choice | Used for |
|---|---|---|
| Primary OLTP | **PostgreSQL 16** | Relational data, transactions, JSONB for schema-on-write. |
| Cache + coordination | **Redis 7** | Rate limits, presence, short-lived sessions, hot query cache. |
| Object storage | **S3-compatible** | File bodies. Never proxy large bytes through the API. |
| Search | **Meilisearch** or **Typesense** | Secondary indexes, faceting. Optional until needed. |
| Queue | **Redpanda** or **Kafka** | Webhooks, fan-out, media jobs. |
| Analytics | **ClickHouse** | Usage, errors, latency percentiles. Read-only for the product. |

PostgreSQL is the recommendation over a bespoke store because it gives
transactions, `JSONB`, mature backup tooling and a real query planner for free —
all of which the platform needs before it needs custom sharding.

### Why not "just use a managed BaaS"

The point of FD53 is ownership: the platform, its data model and its economics
are yours. The cost is that you own the operational burden too. The mitigation
is to keep the *data model and API* the differentiator and treat the runtime as
replaceable — which is exactly what the layered design above does.

---

## 5. Consistency and tenancy strategy

FD53 starts **single-region, single-writer** and stays there until measured
pressure says otherwise. Premature multi-writer sharding buys operational
complexity that a young platform cannot afford.

The escalation path, with the trigger for each step:

| Stage | Topology | Trigger |
|---|---|---|
| 1 | Single region, Postgres primary | Default today |
| 2 | + read replicas, read/write split | Reporting queries hurting OLTP |
| 3 | Multi-region active/standby, DNS failover | Availability SLA commitment |
| 4 | Multi-region active/active, per-project home region | A single region no longer fits the latency budget |
| 5 | Per-project shard | A single tenant's data outgrows a node |

At stage 4 the routing rule is **per-project home region**, not geo-sharding of
individual rows. A project lives entirely in one region; it can move as a unit.
This is what keeps cross-region transactions from ever being a question.

---

## 6. Security model

### Defence in depth

```
 Encryption        TLS 1.3 in transit, AES-256 at rest, KMS-managed keys.
                   Passwords: PBKDF2-SHA256, 60k iterations, 16-byte salt.
                   OTP/reset tokens: HMAC-peppered before storage.

 Rate limiting     Four independent layers (see §2).

 Firewall          Per-key IP allowlist / denylist with CIDR support.
                   Per-key origin allowlist for CORS.
                   WAF rules on every request, pre-auth.

 Access control    Two-tier and deliberately separate:
                     · API key → your backend (project-wide)
                     · Bearer  → your end users (RBAC + row-level `access`)
                   A call with no `Authorization` header is a server-side
                   API-key call and gets full project access. Table `access`
                   applies to end-user bearer tokens only.

 Audit             Append-only `auditLogs`: key lifecycle, auth failures,
                   IP blocks, WAF trips, scope denials, config changes.
                   Client IPs are hashed — the trail is useful without
                   becoming a privacy liability.
```

### The two-credential rule

The single most common source of BaaS bugs is conflating "who is calling my
backend" with "who is calling as my user". FD53 keeps them strictly apart:

- **`X-API-Key`** — authenticates *your backend*. Server-side only. Never ship
  it in a mobile or web client, because a client can be reverse-engineered.
- **`Authorization: Bearer`** — authenticates *your end users*. Obtained from
  `/v1/auth/login`. Safe to ship.

Mobile and web apps should ship the bearer token only, and proxy privileged
operations through their own server or a serverless function holding the key.

---

## 7. Reliability targets

| Layer | Target | Mechanism |
|---|---|---|
| Edge / CDN | 99.99% | Multi-PoP, anycast |
| Gateway | 99.95% | N+1 across ≥2 AZs, health-checked |
| Realtime hubs | 99.9% | Cross-region, reconnect with backoff |
| Postgres | RPO < 60s, RTO < 15min | PITR + streaming replicas |

Every response carries `X-FD53-Request-Id`. That id appears in the request log,
the audit trail and server logs, so one customer report maps to an exact trace
across all three.

---

## 8. Observability

Four signals, each answering a different question:

| Signal | Question | Where |
|---|---|---|
| Request logs | Which routes is my app hitting, how slow, what failed? | `requestLogs` |
| Usage counters | Am I about to hit a plan limit? | `usageDaily` |
| Audit trail | Who did this, and what did they try? | `auditLogs` |
| Metrics | Is the platform itself healthy? | OpenTelemetry → Prometheus |

The first three are **per-project and self-service** — the developer sees them
in the dashboard without asking anyone. The fourth is operator-only.

The telemetry split matters: request logging happens on the response path, so it
is wrapped in a try/catch that can never fail a customer request. Losing a log
row is bad; failing a purchase is worse.

---

## 9. Cost model

The economics of a BaaS are dominated by three line items, and the architecture
above treats each deliberately:

| Cost | Driver | Mitigation |
|---|---|---|
| Storage egress | Images and video leaving the CDN | Aggressive `Cache-Control`, image resizing at the edge, signed URLs so private files never traverse the API |
| Database reads | Chatty SDKs | Cursor pagination, response caching, read replicas, query budgets per project |
| Function invocations | Chatty endpoints | Batching, caching at the gateway, per-project rate limits |

The plan limits in `FREE_PLAN` (120 req/min, 250 MB storage, 50 tables) are
enforced at the gateway, so a runaway project is contained at the edge instead
of consuming the platform's own quota.

---

## 10. Current implementation status

Honest accounting of what exists in code today versus what is designed here.

### Built and verified

| Capability | Status |
|---|---|
| API gateway + `X-API-Key` middleware | **Live** — 10-stage pipeline, audited |
| WAF rules | **Live** — traversal, injection, size, verb |
| Per-IP flood limiting | **Live** — pre-auth, with auth-failure bans |
| Audit trail | **Live** — append-only, IP-hashed |
| Key lifecycle (env, expiry, IP, origins) | **Live** |
| Auth: email/password, phone OTP, sessions, RBAC | **Live** |
| Database: schema-on-write tables, rows, unique indexes | **Live** |
| Storage: buckets, upload (3 styles), quota | **Live** |
| Realtime event log + polling | **Live** (WebSocket hubs are next) |
| OpenAPI 3.1 spec | **Live** at `/v1/openapi.json`, generated from the route table |
| Developer dashboard | **Live** — projects, database, storage, users, keys, usage, docs |

### Designed, not yet implemented

WebSocket hubs · GraphQL · cron/queue workers · transactional email and SMS ·
image optimisation pipeline · video transcoding · 2FA/TOTP · OAuth provider
callbacks (Google/Apple) · SQL surface · billing integration · backups.

Each is scoped in [`roadmap.md`](./roadmap.md). Nothing in that list is
half-implemented in a way that would surprise a user — they return a clean
`501 not_implemented` rather than a misleading success.