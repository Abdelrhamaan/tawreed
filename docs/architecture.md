# Tawreed — Architecture

> **Tawreed** (توريد, "supply") is a B2B supply platform: suppliers sell to shops (supermarkets and clothing stores), each supplier ships from its own warehouse, couriers deliver, and couriers chat with shop owners.
>
> The business logic is **intentionally thin**. It exists to give every technology in this lab a real job. See [ADR-001](architecture-decisions.md#adr-001-microservices-from-day-one-for-learning).

---

## 1. Guiding rules

1. **Microservices from day one, built one at a time.** The target architecture is below. We never build two services in parallel.
2. **Every service owns its data.** No service reads or writes another service's database. Ever.
3. **Every technology has a job written next to it.** If we can't write the job, we don't add the technology.
4. **Sync for questions that need an answer now. Async for facts and work.**

---

## 2. Services

| # | Service | Owns | Stack | Datastore | Why this stack |
|---|---|---|---|---|---|
| 1 | `identity` | Users, roles, login, JWT issuing | Python · Django + DRF | PostgreSQL | Django's auth, admin, and migrations are mature; auth is CRUD-heavy, not throughput-heavy |
| 2 | `catalog` | Products, variants, categories, supplier price lists | Python · FastAPI | MongoDB (+ Redis cache, + Elasticsearch search) | Supermarket items and clothing variants have different shapes → flexible documents; read-heavy → cache |
| 3 | `inventory` | Warehouses, stock per batch, reservations | Go · Gin | **MySQL** | Hot contention on stock rows; Go for concurrency; MySQL on purpose to compare InnoDB locking with PostgreSQL (see ADR-002) |
| 4 | `ordering` | Carts, orders, per-supplier split, line quantities, credit limits, ledger | Python · Django | PostgreSQL | Money + multi-step workflows → strong transactions; the Saga lives here |
| 5 | `shipping` | Shipments, courier assignment, delivery confirmation, courier location | Go · `net/http` + gRPC | PostgreSQL + Redis GEO | High-rate location writes; standard library first to learn what frameworks hide |
| 6 | `chat` | Conversations per shipment, messages, presence | Python · FastAPI (WebSockets) | MongoDB + Redis Pub/Sub | Async I/O with many open connections; append-only messages fit documents |
| 7 | `notifier` | Sending push/SMS/email (simulated) | Python worker | none (stateless) | Pure consumer of work from RabbitMQ |
| 8 | `web` | Frontend for shop owners, suppliers, couriers | React + TypeScript (Vite), served by nginx | — | — |

**Languages across the system:** Python (Django ×2, FastAPI ×2, 1 worker), Go (Gin ×1, stdlib ×1).

---

## 3. Big picture

```text
                           ┌──────────────┐
      Browser (React) ───> │   Gateway    │  Kubernetes Gateway API
                           └──────┬───────┘
          ┌──────────┬────────────┼─────────────┬─────────────┬──────────────┐
          ▼          ▼            ▼             ▼             ▼              ▼
     ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌───────────┐ ┌──────────┐  ┌──────────┐
     │identity │ │ catalog │ │ ordering │ │ inventory │ │ shipping │  │   chat   │
     │ Django  │ │ FastAPI │ │  Django  │ │  Go/Gin   │ │ Go/stdlib│  │ FastAPI  │
     └────┬────┘ └────┬────┘ └────┬─────┘ └─────┬─────┘ └────┬─────┘  └────┬─────┘
          │           │           │  gRPC ───>  │            │  WebSocket  │
       Postgres    MongoDB     Postgres       MySQL       Postgres       MongoDB
                   + Redis                               + Redis GEO    + Redis Pub/Sub
                   + Elastic
          │           │           │             │            │             │
          └───────────┴─────┬─────┴─────────────┴────────────┴─────────────┘
                            ▼
              ┌──────────────────────────────┐      ┌────────────────────┐
              │ Kafka  (events: facts)       │      │ RabbitMQ (commands)│
              │ order.placed, stock.reserved │      │ send-notification  │
              │ shipment.delivered, ...      │      │ courier-offer      │
              └──────────────────────────────┘      └─────────┬──────────┘
                                                              ▼
                                                         ┌──────────┐
                                                         │ notifier │
                                                         └──────────┘
```

---

## 4. Communication rules

### 4.1 Browser → services
- Everything enters through the **Gateway** (Gateway API implementation, e.g. Envoy Gateway).
- Routes: `/api/identity/*`, `/api/catalog/*`, `/api/orders/*`, `/api/shipping/*`, `/ws/chat`.

### 4.2 Authentication
- `identity` signs JWTs with **RS256** (private key).
- Every other service verifies tokens **locally** using the public key from `identity`'s JWKS endpoint, cached in memory.
- **No service calls `identity` on every request.** If `identity` is down, existing sessions keep working until tokens expire.

### 4.3 Synchronous service-to-service: gRPC
- Used only when the caller **cannot continue without the answer**.
- First case: `ordering → inventory` to reserve stock.
- Every call has a **deadline**. No deadline = rejected in code review.
- We start with REST for this call, feel its pain, then move to gRPC (Phase 4).

### 4.4 Kafka — events (facts that already happened)
- Topics per aggregate, versioned: `identity.user.v1`, `catalog.product.v1`, `ordering.order.v1`, `inventory.stock.v1`, `shipping.shipment.v1`.
- **Key = aggregate ID** → all events of one order land on one partition → ordered.
- Many independent consumers, replayable history.
- Producers publish through the **Transactional Outbox** — never "write DB then publish" in two steps.
- Consumers are **idempotent** (inbox / processed-events table in the same DB transaction).

### 4.5 RabbitMQ — commands (work somebody must do)
- `notifications.send` → `notifier`.
- `shipping.courier-offer` → offers to freelance couriers, with **TTL + dead-letter exchange** so unaccepted offers expire and get re-offered.
- One consumer handles each message; ack after success; poison messages go to a DLQ.

> **Rule of thumb:** if the sentence is "X happened" → Kafka. If it is "please do Y" → RabbitMQ.

---

## 5. Redis — every job it has

| Job | Service | Pattern |
|---|---|---|
| Product cache | `catalog` | Cache-aside with TTL, invalidated by `catalog.product.v1` events |
| Login rate limiting | `identity` | Fixed / sliding window counters |
| Courier live location | `shipping` | `GEOADD` / `GEOSEARCH` |
| Chat fan-out across pods | `chat` | Pub/Sub channel per conversation |
| Presence | `chat` | Keys with TTL refreshed by heartbeats |

Redis is **never** the source of truth. Losing Redis must degrade the system, not corrupt it.

---

## 6. Kubernetes layout (local, `kind`)

### 6.1 Nodes: pools, not one-service-per-node

```text
kind cluster "tawreed"
 ├── control-plane
 ├── worker-1   label pool=apps      → stateless services (Deployments)
 ├── worker-2   label pool=apps      → stateless services (Deployments)
 └── worker-3   label pool=data      → databases, Kafka, RabbitMQ, Elasticsearch
                taint  pool=data:NoSchedule
```

- Stateless services run on **any** `apps` node; the scheduler decides. Replicas of the same service are spread across nodes (`topologySpreadConstraints`) so losing one node doesn't take the service down.
- Stateful workloads tolerate the `data` taint and select `pool=data`.
- Why not "one node per service": see [ADR-005](architecture-decisions.md#adr-005-node-pools-instead-of-one-node-per-service).

### 6.2 Namespaces

| Namespace | Contents |
|---|---|
| `tawreed` | The 7 services + `web` |
| `data` | PostgreSQL ×3, MySQL, MongoDB, Redis |
| `messaging` | Kafka, RabbitMQ |
| `search` | Elasticsearch (product search) |
| `observability` | Prometheus, Grafana, Fluent Bit, Elasticsearch (logs), Kibana, OpenTelemetry Collector, Jaeger |
| `gateway` | Gateway controller |

### 6.3 Which workload type, and why

| Workload | Used for | Why |
|---|---|---|
| `Deployment` (→ `ReplicaSet` → Pods) | All services, `web` | Stateless, interchangeable pods, rolling updates |
| `StatefulSet` | Databases, Kafka, RabbitMQ, Elasticsearch | Stable identity + own persistent volume per pod |
| `DaemonSet` | Fluent Bit, node-exporter | Exactly one per node: they collect per-node data |
| `Job` | DB migrations per release | Runs to completion once |
| `CronJob` | Expire stale stock reservations | Scheduled |
| `HorizontalPodAutoscaler` | `catalog`, `shipping` (later) | Scale on load |

### 6.4 Stateful components: hand-written first, operators later
- Phase 1: write the PostgreSQL `StatefulSet` + headless `Service` + `PersistentVolumeClaim` **by hand** to learn the primitives.
- Later, when we need replication/failover, move to operators: **CloudNativePG** (PostgreSQL), **Strimzi** (Kafka, KRaft mode), **ECK** (Elasticsearch), **MongoDB Community Operator**.

### 6.5 Laptop budget
The full stack (3× Postgres, MySQL, MongoDB, Redis, Kafka, RabbitMQ, 2× Elasticsearch, Prometheus, Grafana, 7 services) needs roughly **16 GB RAM minimum**, 32 GB comfortable. We only run what the current phase needs.

---

## 7. Observability

```text
Metrics: service /metrics ──> Prometheus ──> Grafana
Logs:    stdout (JSON) ──> Fluent Bit (DaemonSet) ──> Elasticsearch ──> Kibana
Traces:  OpenTelemetry SDK ──> OTel Collector ──> Jaeger
```

- Every request gets a **correlation ID** at the gateway (`X-Request-ID`), propagated over HTTP, gRPC, Kafka headers and RabbitMQ headers, and written in every log line.
- Every service exposes `/healthz/live` (shallow) and `/healthz/ready` (checks its own critical dependencies).

---

## 8. Repository layout

```text
Learn/
├── docs/                    # these documents
├── services/
│   ├── identity/            # Django + DRF
│   ├── catalog/             # FastAPI
│   ├── inventory/           # Go + Gin
│   ├── ordering/            # Django
│   ├── shipping/            # Go stdlib + gRPC
│   ├── chat/                # FastAPI WebSockets
│   └── notifier/            # Python worker
├── web/                     # React + TypeScript
├── proto/                   # gRPC contracts (shared)
├── contracts/events/        # Kafka event schemas (JSON Schema)
└── deploy/
    ├── kind/                # cluster config
    ├── k8s/                 # raw manifests (early phases)
    └── helm/                # our own charts (later phases)
```
