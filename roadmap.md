# FD53 — Development Roadmap

A phased plan to grow FD53 from a working BaaS into a complete cloud platform.
Each phase states what ships, what it unlocks, and how completion is judged.

**Status legend:** ✅ shipped · 🟡 in progress · ⬜ not started

---

## Phase 0 — Foundation ✅

**Shipped.** The platform core with the security perimeter in place.

| Deliverable | Where |
|---|---|
| API gateway + `X-API-Key` middleware | `src/convex/rest/gateway.ts` |
| WAF rules | `src/convex/lib/net.ts` |
| Per-IP flood limiting + auth-failure bans | `src/convex/edge.ts` |
| Audit trail | `src/convex/audit.ts` |
| Key lifecycle: environment, expiry, IP/CIDR, origins | `src/convex/apiKeys.ts` |
| Auth: email/password, phone OTP, sessions, RBAC | `src/convex/rest/auth.ts` |
| Database: tables, rows, unique indexes | `src/convex/rest/database.ts` |
| Storage: buckets, uploads, quota | `src/convex/rest/storage.ts` |
| Realtime event log + polling | `src/convex/rest/realtime.ts` |
| OpenAPI 3.1, generated from the route table | `src/convex/rest/openapi.ts` |
| Developer dashboard | `src/pages/dashboard/` |

**Unlocked:** a developer can create a project, mint a key, and ship auth,
database and storage from any HTTP client — including FlutterFlow.

---

## Phase 1 — Developer experience ⬜

*Focus: remove every reason a developer would leave FD53 for another backend.*

### 1.1 First-party SDKs

Generated from `packages/sdk-spec`, which is derived from the OpenAPI document
— so an SDK method can never drift from the API.

| SDK | Priority | Notes |
|---|---|---|
| Dart / Flutter | **P0** | Needed for FlutterFlow and native mobile |
| TypeScript / Node | **P0** | Next.js, React Native, server backends |
| JavaScript (browser) | P1 | React, Vue, Angular web clients |
| Swift | P1 | iOS |
| Kotlin | P1 | Android |
| Python | P2 | Scripts, notebooks |

Each ships auth, database, storage and realtime clients with typed models and
automatic token refresh.

**Exit criteria:** `dart pub publish --dry-run` and `npm publish --dry-run`
succeed; an app can be built against only the SDK, with no raw HTTP.

### 1.2 Query builder and visual schema editor

Extend the existing `Database.tsx` into a full editor: drag columns, set types,
define `reference` relationships visually, and preview generated queries.

**Exit criteria:** a developer builds a working CRUD screen without writing HTTP.

### 1.3 API testing console

An in-dashboard request builder: pick a method and path, fill headers and a
body, run it, and see the status, headers and parsed JSON — with your real key
applied server-side so it never leaves the dashboard.

### 1.4 Self-service audit log UI

Surface `auditLogs` in the dashboard: filter by event, key, IP hash or request
id, and jump from a failed auth straight to the request that caused it.

---

## Phase 2 — Realtime 🟡

**In progress.** The event log and polling fallback work today. The gap is
low-latency delivery.

### 2.1 WebSocket hubs

Regional hubs with a shared pub/sub backplane, so a publish fans out to every
region holding a subscriber.

```
wss://<deployment>/realtime?token=<bearer>
  → {"type":"subscribe","channel":"room:abc"}
  ← {"type":"event","channel":"room:abc","data":{…}}
```

**Exit criteria:** a message published in region A appears in region B within
100ms p99; 10,000 concurrent sockets per pod with < 50ms p99 echo latency.

### 2.2 Live database subscriptions

Subscribe to a query, receive an event when matching rows change:

```
→ {"type":"query","table":"todos","filter":{"status":"open"}}
← {"type":"change","op":"insert","row":{…}}
```

**Exit criteria:** a Convex-style reactive query works from every SDK.

### 2.3 Presence, typing indicators, notifications

Ephemeral state in Redis with a short TTL — presence is deliberately never
persisted. Typing indicators share the same channel machinery.

---

## Phase 3 — Extending the platform ⬜

### 3.1 Cloud Functions

Server functions per project, with:

| Capability | Design |
|---|---|
| Invocation | `POST /v1/functions/:name/invoke` + HTTP triggers |
| Runtime | Sandboxed isolate, project-scoped env vars |
| Secrets | Per-project encrypted secret store |
| Cold start | Target < 500ms |

### 3.2 Queues, cron and webhooks

| Feature | Model |
|---|---|
| Queues | At-least-once delivery, configurable retries, DLQ |
| Cron | Per-project schedules, minimum 1-minute granularity |
| Webhooks | HMAC-signed payloads, retry with exponential backoff, delivery log |

### 3.3 Notifications

Transactional email, SMS and push. Provider-agnostic adapters so a project
picks Resend, Twilio or FCM/APNs without code changes. Every message is
templated, rate-limited and logged.

**Exit criteria:** a developer can send an email, an SMS and a push notification
through three SDK calls, with delivery status queryable.

### 3.4 GraphQL

Auto-generate a typed schema from a project's tables:

```
POST /v1/graphql
{ todos(where: { status: "open" }, limit: 20) { id title } }
```

Depth-limited, cost-estimated against the project's quota, and rate-limited
separately from REST.

---

## Phase 4 — Storage and media ⬜

### 4.1 Image optimisation pipeline

On upload into a media bucket: resize, strip EXIF (including GPS), generate
WebP/AVIF variants, emit blurhash placeholders.

### 4.2 Video and audio

Transcode to HLS/DASH for adaptive streaming, waveform extraction for audio,
and signed CDN URLs.

### 4.3 CDN

Signed URLs for private buckets, cache-control per bucket, and edge image
resizing — which cuts egress cost substantially versus proxying through the API.

### 4.4 Buckets v2

Developer-defined buckets with configurable visibility, quota and lifecycle
rules, replacing today's three fixed buckets (which remain as defaults).

---

## Phase 5 — Database depth ⬜

### 5.1 Relationships

Enforce and query `reference` fields across tables, with joins in the query
builder and cascading-reference validation.

### 5.2 Indexes and migrations

Developer-managed indexes, plus a migration system:

```
2026-09-01_add_order_status.up.sql
2026-09-01_add_order_status.down.sql
```

Applied as versioned, reversible operations with a dry-run mode.

### 5.3 Import and export

CSV and JSON import with per-field mapping and error reporting; export with
streaming, so a million-row export does not buffer in memory.

### 5.4 Backups

Scheduled snapshots, point-in-time restore, and one-click export to the
developer's own bucket.

### 5.5 SQL surface

A read-only SQL endpoint for analytics over a project's data, with a hard
statement timeout and read-only replica routing. A full write path is a much
larger commitment and is deliberately deferred.

---

## Phase 6 — Security and compliance ⬜

### 6.1 2FA / MFA

TOTP (RFC 6238) and WebAuthn, with recovery codes. Enforceable per project or
per role, so admin accounts can require it.

### 6.2 OAuth providers

Google, Apple and generic OAuth2 / OIDC. Authorization-code flow with PKCE,
provider linking to existing end users, and per-project redirect allowlists.

### 6.3 Refresh tokens

Opaque, rotating, single-use refresh tokens with reuse detection: presenting a
revoked refresh token revokes the entire family, the standard defence against
token theft.

### 6.4 Encryption at rest

Per-project envelope encryption with KMS-managed keys, plus field-level
encryption for values a developer marks sensitive.

### 6.5 Compliance

GDPR export and erasure endpoints, configurable retention, and region pinning.

---

## Phase 7 — Scale and billing ⬜

### 7.1 Multi-region

Per-project home region with cross-region replication, and DNS failover.
Described in [`architecture.md` §5](./architecture.md#5-consistency-and-tenancy-strategy).

### 7.2 Billing

Plans and quotas as configuration, usage-based metering against a payment
provider, invoices, and webhooks. The metering data already exists —
`usageDaily` is the billing ledger.

### 7.3 Operations

Provider abstraction per plane, so a workload can move between managed and
self-hosted as economics change without touching the API contract.

---

## Timeline

Rough sequencing, assuming one developer part-time on infrastructure:

| Phase | Focus | Duration |
|---|---|---|
| 0 | Foundation | ✅ done |
| 1 | Developer experience | 4–6 weeks |
| 2 | Realtime | 4–6 weeks |
| 3 | Cloud Functions + notifications | 6–8 weeks |
| 4 | Storage and media | 4–6 weeks |
| 5 | Database depth | 6–8 weeks |
| 6 | Security and compliance | 4–6 weeks |
| 7 | Scale and billing | 8–12 weeks |

Phases 3, 4 and 5 are largely independent and can overlap if bandwidth allows.

---

## Design principles

These have held across every phase and should survive the next one.

1. **Security is the gateway's job, not each endpoint's.** A route added to
   `ROUTES` inherits the full perimeter for free. There must never be a way to
   add an endpoint that skips it.
2. **Fail closed on configuration errors.** An unconfigured limit, a
   malformed key or an unparseable IP rule denies access. An attacker should
   never be able to widen their own permissions with a typo.
3. **Never fail a customer request over bookkeeping.** Audit writes, usage
   counters and request logs are all wrapped so they cannot turn a good
   response into a 500. Losing a log row is bad; failing a purchase is worse.
4. **Prefer the honest error.** `501 not_implemented` beats a stub that appears
   to succeed.
5. **Hash anything identifying.** Client IPs are hashed before storage; the
   audit trail stays useful without becoming a privacy liability.
6. **The spec is generated, not written.** `openapi.ts` builds the document
   from the live route table, so documentation cannot drift from behaviour.
   Extend that discipline to the SDKs.

---

## Related

- [`architecture.md`](./architecture.md) — system design
- [`folder-structure.md`](./folder-structure.md) — where new code lands
- [`deployment.md`](./deployment.md) — shipping each phase
- [`flutterflow.md`](./flutterflow.md) — the client integration surface