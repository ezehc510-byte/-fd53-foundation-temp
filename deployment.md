# FD53 — Deployment Guide

Two paths:

- **Local / self-hosted single node** — Docker Compose. Enough to run the whole
  platform on one machine.
- **Production** — Kubernetes. Multi-region, autoscaling, self-healing.

The current Convex-hosted deployment needs neither: it is managed. This guide
covers the target architecture in [`architecture.md`](./architecture.md).

---

## 1. Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Go | 1.22+ | Gateway, workers |
| Rust | 1.78+ | Auth/crypto crates, realtime hubs |
| Node.js | 20 LTS | Workers, dashboard build |
| Docker | 24+ | Local stack |
| kubectl | 1.29+ | Production |
| Helm | 3.14+ | Production |

---

## 2. Local development (Docker Compose)

### docker-compose.yml

```yaml
name: fd53

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: fd53
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD}
      POSTGRES_DB: fd53
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      # Gate dependent services on readiness, not on "the container started".
      test: ["CMD-SHELL", "pg_isready -U fd53 -d fd53"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    # Rate limits and presence are ephemeral — eviction is correct here.
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 10

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:?set MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:?set MINIO_ROOT_PASSWORD}
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

  migrate:
    build: { context: ., dockerfile: deploy/docker/core.Dockerfile }
    command: ["/app/migrate", "up"]
    environment:
      DATABASE_URL: postgres://fd53:${POSTGRES_PASSWORD}@postgres:5432/fd53
    depends_on:
      postgres: { condition: service_healthy }
    restart: "no"   # migrations run once, then exit

  gateway:
    build: { context: ., dockerfile: deploy/docker/gateway.Dockerfile }
    environment:
      DATABASE_URL: postgres://fd53:${POSTGRES_PASSWORD}@postgres:5432/fd53
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET: fd53-media
      S3_ACCESS_KEY: ${MINIO_ROOT_USER}
      S3_SECRET_KEY: ${MINIO_ROOT_PASSWORD}
      FD53_JWT_SECRET: ${FD53_JWT_SECRET:?set FD53_JWT_SECRET}
      PORT: "8080"
    ports: ["8080:8080"]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      migrate: { condition: service_completed_successfully }
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  realtime:
    build: { context: ., dockerfile: deploy/docker/realtime.Dockerfile }
    environment:
      REDIS_URL: redis://redis:6379
      PORT: "8081"
    ports: ["8081:8081"]
    depends_on:
      redis: { condition: service_healthy }
    restart: unless-stopped

  worker:
    build: { context: ., dockerfile: deploy/docker/worker.Dockerfile }
    environment:
      DATABASE_URL: postgres://fd53:${POSTGRES_PASSWORD}@postgres:5432/fd53
      REDIS_URL: redis://redis:6379
      QUEUE_URL: ${QUEUE_URL:-memory://}
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      migrate: { condition: service_completed_successfully }
    restart: unless-stopped

volumes:
  pgdata:
  miniodata:
```

### Run it

```bash
cat > .env <<'EOF'
POSTGRES_PASSWORD=change-me-in-production
MINIO_ROOT_USER=fd53minio
MINIO_ROOT_PASSWORD=change-me-too
FD53_JWT_SECRET=$(openssl rand -hex 32)
EOF

docker compose up --build
docker compose exec gateway wget -qO- http://localhost:8080/healthz
```

The API is on `:8080`. Verify end to end:

```bash
curl -s http://localhost:8080/v1/health | jq
curl -s http://localhost:8080/v1/openapi.json | jq '.info.title'
```

---

## 3. Production Kubernetes

### Namespace and configuration

```yaml
# deploy/kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fd53
```

### Secrets

Never bake secrets into images or commit them to git.

```bash
kubectl -n fd53 create secret generic fd53-secrets \
  --from-literal=postgres-password="$(openssl rand -hex 32)" \
  --from-literal=jwt-secret="$(openssl rand -hex 32)" \
  --from-literal=s3-access-key="$AWS_ACCESS_KEY_ID" \
  --from-literal=s3-secret-key="$AWS_SECRET_ACCESS_KEY"
```

For anything longer-lived, use External Secrets Operator syncing from AWS
Secrets Manager or Vault so rotation does not require a redeploy.

### Gateway deployment

```yaml
# deploy/kubernetes/gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fd53-gateway
  namespace: fd53
spec:
  replicas: 3          # ≥2 so a node failure cannot take the gateway down
  selector:
    matchLabels: { app: fd53-gateway }
  template:
    metadata:
      labels: { app: fd53-gateway }
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      containers:
        - name: gateway
          image: ghcr.io/fd53/gateway:v1.0.0
          ports: [{ containerPort: 8080 }]
          envFrom:
            - secretRef: { name: fd53-secrets }
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef: { name: fd53-secrets, key: postgres-password }
            - name: PORT
              value: "8080"
          # Readiness gates traffic; liveness restarts a wedged process.
          # Deliberately different: a slow-to-respond pod should be pulled
          # from the load balancer, not killed.
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 15
            failureThreshold: 3
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { memory: 512Mi }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: { app: fd53-gateway }
```

> **Memory limits but no CPU limits.** Requests reserve capacity; a CPU limit
> throttles a hot pod instead of letting it burst, which adds tail latency
> exactly when the platform is busiest. Set a memory limit — leaking memory must
> be fatal.

### Service, autoscaling and disruption budget

```yaml
# deploy/kubernetes/gateway-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: fd53-gateway
  namespace: fd53
spec:
  type: ClusterIP
  selector: { app: fd53-gateway }
  ports: [{ port: 80, targetPort: 8080 }]

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fd53-gateway
  namespace: fd53
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: fd53-gateway }
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: fd53-gateway
  namespace: fd53
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: fd53-gateway }
```

### Ingress

```yaml
# deploy/kubernetes/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fd53-api
  namespace: fd53
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/enforce-cors: "false"   # FD53 sets its own
    nginx.ingress.kubernetes.io/proxy-body-size: "26m"    # matches WAF max
spec:
  tls:
    - hosts: [api.fd53.dev]
      secretName: fd53-api-tls
  rules:
    - host: api.fd53.dev
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: fd53-gateway
                port: { number: 80 }
```

> **Keep the ingress dumb.** No auth, no rate limiting, no CORS — FD53's gateway
> does all of it. Duplicating logic at the ingress guarantees the two drift
> apart, and the weaker one wins.

### Database

Managed Postgres (RDS/Cloud SQL) rather than in-cluster. Then:

```bash
# Backups: point-in-time recovery plus daily snapshots.
aws rds modify-db-instance --db-instance-identifier fd53 \
  --backup-retention-period 35 \
  --preferred-backup-window 03:00-04:00
```

Multi-AZ gives a failover under ~60s. Run two read replicas for analytics so
reporting queries never contend with the primary.

### Realtime hubs

```yaml
# deploy/kubernetes/realtime-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: fd53-realtime, namespace: fd53 }
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: hub
          image: ghcr.io/fd53/realtime:v1.0.0
          ports: [{ containerPort: 8081 }]
          envFrom: [{ secretRef: { name: fd53-secrets } }]
          readinessProbe:
            httpGet: { path: /readyz, port: 8081 }
            periodSeconds: 5
          resources:
            requests: { cpu: 500m, memory: 512Mi }
            limits:   { memory: 1Gi }
```

Realtime scales on **concurrent connections, not CPU** — a pod holding 10k
mostly-idle sockets looks idle to a CPU-based HPA. Scale on
`realtime_connections` with a CPU floor as a backstop.

---

## 4. Migrations

Migrations run as a **Kubernetes Job**, not inside the application image. The
application must never attempt to migrate on boot: with three replicas starting
simultaneously, an in-process migration runs three times concurrently.

```yaml
# deploy/kubernetes/migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata: { name: fd53-migrate, namespace: fd53 }
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: ghcr.io/fd53/core:v1.0.0
          command: ["/app/migrate", "up"]
          envFrom: [{ secretRef: { name: fd53-secrets } }]
```

```bash
kubectl -n fd53 apply -f migrate-job.yaml
kubectl -n fd53 wait --for=condition=complete job/fd53-migrate --timeout=300s
kubectl -n fd53 set image deployment/fd53-gateway gateway=ghcr.io/fd53/gateway:v1.0.0
```

### Expanding/contracting

Never rename or drop a column in one deploy — old and new pods run
concurrently during a rollout and will disagree.

```
Deploy N   : add the new column (nullable). Write to both.
Deploy N+1 : backfill. Read from the new column.
Deploy N+2 : stop writing the old column.
Deploy N+3 : drop the old column.
```

Four deploys, zero downtime, reversible at every step.

---

## 5. Rollouts

```bash
kubectl -n fd53 set image deployment/fd53-gateway gateway=ghcr.io/fd53/gateway:v1.1.0
kubectl -n fd53 rollout status deployment/fd53-gateway --timeout=300s
kubectl -n fd53 rollout undo deployment/fd53-gateway     # instant rollback
```

Every image is tagged with a git SHA, never `latest`. A rollback must be
possible without rebuilding.

---

## 6. Observability

### Metrics

```yaml
# deploy/kubernetes/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: fd53-gateway, namespace: fd53 }
spec:
  selector:
    matchLabels: { app: fd53-gateway }
  endpoints:
    - port: metrics
      interval: 15s
```

Export OpenTelemetry traces with the request id as a trace attribute, so a
customer's `X-FD53-Request-Id` maps directly to a trace.

### Alerts worth having

| Alert | Condition | Why it matters |
|---|---|---|
| `GatewayErrorRateHigh` | 5xx > 1% over 5m | Customer-facing breakage |
| `GatewayLatencyHigh` | p99 > 1s | Feels broken before it errors |
| `PostgresConnectionsHigh` | > 80% of max | Connection exhaustion precedes outage |
| `RedisMemoryHigh` | > 85% | Eviction starts dropping rate-limit state |
| `RealtimeConnectionsDrop` | −50% in 5m | Hub or network event |
| `QueueBacklogGrowing` | depth rising 15m | Workers not keeping up |
| `CertExpiringSoon` | < 14 days | Avoids an outage nobody planned for |

### Logs

Ship stdout to Loki or Elasticsearch with the request id as a label. Retention
14 days hot, 90 days cold.

The audit trail is separate and **not prunable** — it is evidence, not a
debugging aid.

---

## 7. Backups and recovery

| Asset | Method | RPO | RTO |
|---|---|---|---|
| Postgres | PITR + daily snapshots | < 60s | < 15 min |
| Object storage | Versioned bucket + cross-region | ~0 | < 30 min |
| Redis | None needed | — | Cache rebuilds |
| Config | Git | 0 | Rebuild |

Run the restore at least quarterly. An untested backup is a hypothesis.

```bash
# Verify a point-in-time restore
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier fd53 \
  --db-instance-identifier fd53-verify \
  --restore-time 2026-09-01T12:00:00Z
```

### What is not backed up

Rate-limit counters, presence and cache entries are deliberately excluded. All
are reconstructible, and restoring them would roll back abuse protection — the
opposite of what you want during an incident.

---

## 8. Security checklist

Before go-live:

- [ ] TLS 1.2+ everywhere; TLS 1.3 where clients support it
- [ ] HSTS enabled with a one-year max-age
- [ ] Secrets in a secret manager, never in images or git
- [ ] Non-root containers, read-only root filesystem, all capabilities dropped
- [ ] Network policies denying cross-namespace traffic by default
- [ ] Postgres not publicly reachable; only the gateway and workers may connect
- [ ] Redis auth enabled and bound to a private network
- [ ] `FD53_JWT_SECRET` rotated on any suspected compromise — this invalidates
      every issued token
- [ ] Audit log retention meets your compliance requirement
- [ ] WAF rules reviewed against real traffic before enabling blocking mode
- [ ] `GET /v1/openapi.json` reviewed — it is public by design and must contain
      no project data

### Rotating the JWT secret

Bumping `FD53_JWT_SECRET` invalidates every outstanding access token
immediately. Sessions are server-side and revocable, so this is a clean global
sign-out — plan it during a low-traffic window and tell customers.

---

## 9. Deploying the current Convex build

The implementation in this repository runs on Convex and needs no Kubernetes.
To ship it:

```bash
bun install
bunx convex dev --once          # push functions, regenerate types
bunx tsc -b --noEmit            # typecheck before anything ships
```

Deploy to production with:

```bash
npx convex deploy
```

Frontend configuration comes from environment variables set in the dashboard,
never from a committed `.env`:

| Variable | Value |
|---|---|
| `VITE_CONVEX_URL` | `https://<deployment>.convex.cloud` |
| `VITE_CONVEX_SITE_URL` | `https://<deployment>.convex.site` |

---

## 10. Related

- [`architecture.md`](./architecture.md) — what is being deployed and why
- [`folder-structure.md`](./folder-structure.md) — where the Dockerfiles live
- [`roadmap.md`](./roadmap.md) — deployment milestones through v2.0