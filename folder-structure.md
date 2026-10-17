# FD53 — Folder Structure

Two layouts are documented here:

1. **[Current implementation](#1-current-implementation)** — what is in the
   repository today, on the Convex runtime.
2. **[Target monorepo](#2-target-monorepo)** — the recommended layout once FD53
   moves to its own Go/Rust services, described in
   [`architecture.md`](./architecture.md).

Keeping both matters: the current layout is the one you edit today, and the
target is what you migrate toward. The module boundaries are identical in both.

---

## 1. Current implementation

```
fd53/
├── src/
│   ├── convex/                     ← the entire backend
│   │   ├── http.ts                 HTTP surface: registers /v1/* for all verbs
│   │   ├── schema.ts               Single source of truth for the data model
│   │   │
│   │   ├── lib/                    ← pure, framework-free helpers
│   │   │   ├── constants.ts        Plan limits, roles, buckets, anti-spam caps
│   │   │   ├── crypto.ts           PBKDF2, HMAC, JWT, hashing
│   │   │   ├── net.ts              IP/CIDR matching, WAF rules, CORS origins
│   │   │   ├── rest.ts             Envelope, errors, quota, session issuance
│   │   │   └── authz.ts            Project ownership checks
│   │   │
│   │   ├── rest/                   ← the HTTP layer
│   │   │   ├── gateway.ts          Security pipeline. All requests enter here
│   │   │   ├── router.ts           Route table + matching. Delegates to gateway
│   │   │   ├── openapi.ts          OpenAPI 3.1 generated from the route table
│   │   │   ├── types.ts            RestContext, RestRoute, RequestContext
│   │   │   ├── actor.ts            End-user token → identity bridge
│   │   │   ├── auth.ts             Auth endpoints
│   │   │   ├── database.ts         Table + row endpoints
│   │   │   ├── storage.ts          Bucket + upload + file endpoints
│   │   │   ├── realtime.ts         Publish + channel endpoints
│   │   │   ├── meta.ts             Service index, usage, OpenAPI handler
│   │   │   ├── social.ts           Social graph, posts, messaging
│   │   │   └── marketplace.ts      Products, orders, earnings
│   │   │
│   │   ├── Security edge
│   │   │   ├── edge.ts             Pre-auth per-IP flood limiting
│   │   │   └── audit.ts            Append-only audit trail
│   │   │
│   │   ├── Platform core
│   │   │   ├── projects.ts         Projects, plans, quotas
│   │   │   ├── apiKeys.ts          Key lifecycle + restrictions
│   │   │   ├── tables.ts           Schema definitions
│   │   │   ├── rows.ts             Document storage + unique enforcement
│   │   │   ├── files.ts            Storage objects + quota accounting
│   │   │   ├── endUsers.ts         App end users
│   │   │   ├── sessions.ts         Revocable sessions
│   │   │   ├── roles.ts            RBAC roles + permissions
│   │   │   ├── secrets.ts          Server-side secrets
│   │   │   ├── usage.ts            Quota consumption + daily counters
│   │   │   ├── requestLog.ts       Per-request log window
│   │   │   ├── rateLimits.ts       Per-end-user anti-spam
│   │   │   ├── phoneOtps.ts        OTP challenges
│   │   │   └── passwordResets.ts   Reset tokens
│   │   │
│   │   ├── Realtime
│   │   │   └── realtime.ts         Channel event log
│   │   │
│   │   ├── Identity (LoopUni domain on top of the platform)
│   │   │   ├── identity.ts         Handles, profiles
│   │   │   ├── social.ts           Follows, posts, feed
│   │   │   ├── messaging.ts        Conversations, messages
│   │   │   ├── marketplace.ts      Products, orders, earnings ledger
│   │   │   └── reports.ts          Moderation queue
│   │   │
│   │   ├── auth.ts                 Console sign-in (dashboard users)
│   │   ├── users.ts                Console user queries
│   │   ├── auth.config.ts          Convex Auth configuration
│   │   ├── auth/
│   │   │   └── emailOtp.ts         Console email OTP
│   │   └── _generated/             Convex codegen — never edit by hand
│   │
│   ├── pages/                       ← dashboard (React)
│   │   ├── dashboard/
│   │   │   ├── Overview.tsx        Project health at a glance
│   │   │   ├── Projects.tsx        Create / configure projects
│   │   │   ├── Database.tsx        Visual table editor
│   │   │   ├── Storage.tsx         Buckets + file browser
│   │   │   ├── Users.tsx           End users + roles
│   │   │   ├── ApiKeys.tsx         Key management
│   │   │   ├── Usage.tsx           Request log, top routes, anti-spam
│   │   │   └── Docs.tsx            API reference + architecture
│   │   ├── Home.tsx                Public landing
│   │   └── Auth.tsx                Console sign-in
│   │
│   ├── components/
│   │   ├── dashboard/
│   │   │   ├── ProjectProvider.tsx Active-project context
│   │   │   └── ui.tsx              Shared dashboard primitives
│   │   ├── ui/                     shadcn/ui component library
│   │   └── RequireAuth.tsx         Protected-route wrapper
│   │
│   ├── lib/                        Frontend utilities (SDK base URL, helpers)
│   ├── main.tsx                    Router + providers
│   └── index.css                   Tailwind v4 theme tokens
│
└── docs/
    ├── architecture.md             System design and tech stack
    ├── folder-structure.md         This file
    ├── deployment.md               Docker + Kubernetes guide
    ├── roadmap.md                  Phased development plan
    └── flutterflow.md              FlutterFlow integration blueprint
```

### The layering rule

```
http.ts  →  router.ts  →  gateway.ts  →  rest/*.ts  →  convex/*.ts  →  schema.ts
   ↑          ↑              ↑              ↑             ↑             ↑
 transport    routing      security       HTTP shape    domain logic   data
```

**Dependencies point downward only.** `lib/` is pure and imports nothing from
the platform. This is why `lib/net.ts` can be lifted into the Go gateway
unchanged, and why the WAF rules are unit-testable without a database.

### The two security entry points

There are exactly two ways into the platform:

| File | Role |
|---|---|
| `rest/gateway.ts` | Every HTTP request. Enforces WAF, throttling, API keys, quota. |
| `convex/*.ts` mutations | Dashboard-originated writes. Enforce ownership via `lib/authz.ts`. |

Anything that bypasses the gateway is a bug. If you add a route, add it to
`ROUTES` in `router.ts` and it inherits the full security posture for free.

---

## 2. Target monorepo

The recommended layout once FD53 runs on its own services. Same boundaries,
different runtime.

```
fd53/
├── apps/
│   ├── gateway/                    Go — auth, quota, routing, audit
│   │   ├── cmd/gateway/main.go
│   │   ├── internal/
│   │   │   ├── middleware/         WAF, cors, ratelimit, tracing, recovery
│   │   │   ├── auth/               apikey.go, enduser.go, scopes.go
│   │   │   ├── routing/            router.go, radix_tree.go
│   │   │   └── handlers/           auth, db, storage, realtime, meta
│   │   └── proto/                  Request/response contracts
│   │
│   ├── core/                       Rust — auth, crypto, WAF hot path
│   │   ├── crates/
│   │   │   ├── fd53-crypto/        argon2id, jwt, hmac, peppering
│   │   │   ├── fd53-auth/          token issuance, verification, RBAC
│   │   │   ├── fd53-waf/           rule evaluation
│   │   │   └── fd53-types/         shared domain types
│   │   └── migrations/             Postgres schema migrations
│   │
│   ├── worker/                     Node/TS — queues, cron, webhooks
│   │   ├── src/
│   │   │   ├── queues/             queue consumers + retry policy
│   │   │   ├── cron/               scheduled jobs
│   │   │   ├── webhooks/           dispatch + signature verification
│   │   │   └── providers/          email, sms, push adapters
│   │   └── package.json
│   │
│   ├── realtime/                   Rust — WebSocket hubs, presence
│   │   └── src/
│   │       ├── hub.rs              connection registry
│   │       ├── presence.rs         ephemeral presence in Redis
│   │       └── fanout.rs           cross-region pub/sub
│   │
│   └── dashboard/                  React + Vite — developer console
│       └── src/                    (the current src/pages, unchanged)
│
├── packages/
│   ├── sdk-js/                     TypeScript SDK
│   ├── sdk-dart/                   Dart + Flutter SDK
│   ├── sdk-swift/                  Swift SDK
│   ├── sdk-kotlin/                 Kotlin SDK
│   └── sdk-spec/                   OpenAPI source → all SDKs generated
│
├── deploy/
│   ├── docker/                     Dockerfiles + compose (local stack)
│   ├── kubernetes/                 Manifests + Helm chart
│   └── terraform/                  VPC, RDS, Redis, S3, CDN
│
├── docs/                           This documentation
└── .github/workflows/              CI: build, test, migrate, deploy
```

### Why a monorepo

Every SDK is generated from `packages/sdk-spec`. Keeping the spec and the
SDKs in one repository means a breaking API change cannot merge without
failing the SDK contract tests. Splitting them into separate repos loses exactly
that guarantee.

---

## 3. Conventions

### Adding a route

1. Add a handler in the matching `rest/*.ts`.
2. Add the entry to `ROUTES` in `router.ts` with `scope` and `category`.
3. Add an `OPERATION_DOCS` entry in `rest/openapi.ts` — the spec is generated,
   so without it the endpoint publishes a generic description.
4. Nothing else. The gateway, quota, audit and logging come for free.

### Where new logic belongs

| If you are adding… | Put it in |
|---|---|
| An endpoint | `rest/<module>.ts` |
| Domain logic | `convex/<module>.ts` |
| A pure helper | `lib/<name>.ts`, with no platform imports |
| A table or column | `schema.ts`, then run `bunx convex dev --once` |
| An SDK method | `packages/sdk-spec` first, never the generated SDK |

### Naming

Tables are plural (`projects`, `tables`, `rows`). Handlers are verbs
(`listRows`, `insertRow`, `deleteFile`). Error codes are `snake_case` and
stable — clients branch on them, so changing one is a breaking change.

---

## 4. Related

- [`architecture.md`](./architecture.md) — why the modules are split this way
- [`deployment.md`](./deployment.md) — running the target architecture
- [`roadmap.md`](./roadmap.md) — what gets built next, and where it lands
- [`flutterflow.md`](./flutterflow.md) — the FlutterFlow integration surface