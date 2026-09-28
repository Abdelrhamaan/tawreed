# Tawreed — System Design

> Business rules are deliberately minimal. They exist to create realistic technical problems (contention, multi-step workflows, events, real-time). They can change later.

---

## 1. Actors

| Actor | Does |
|---|---|
| Shop owner | Browses catalogs, places orders (possibly from several suppliers), receives deliveries, chats with the courier |
| Supplier admin | Manages products, prices, warehouse stock, credit limits for shops |
| Courier | Employee of a supplier **or** freelancer; accepts shipments, shares location, confirms delivery |
| Platform admin | Manages users and suppliers |

---

## 2. Functional requirements (v1)

### identity
- Register, login, refresh token, logout.
- Roles: `shop_owner`, `supplier_admin`, `courier`, `platform_admin`.

### catalog
- Supplier creates products. Grocery products have unit/pack size; clothing products have variants (size × color), each with its own SKU.
- Shop owners browse and search products across suppliers.

### inventory
- Each supplier has one warehouse in v1 (model supports many).
- Stock is tracked per **SKU + warehouse + batch**. A batch may have an expiry date (grocery) or none (clothing).
- Reserve stock for an order; release it on cancel or after a timeout; commit it on shipment.

### ordering
- One order may contain items from several suppliers → split into one **supplier order** per supplier.
- Payment is on **credit** (آجل): each supplier sets a credit limit per shop. An order that would exceed the limit is rejected.
- Debt is recorded only for **delivered** quantities.
- Each line tracks: `ordered`, `reserved`, `shipped`, `delivered`, `rejected` (+ reason: `missing` / `damaged`).

### shipping
- One shipment per supplier order.
- Supplier employees are assigned directly; freelancers receive offers and the **first to accept** wins.
- Courier location is updated every 5 seconds while a shipment is active.
- Courier confirms delivery per line (accepted / rejected quantities).

### chat
- One conversation per shipment between courier and shop owner.
- Real-time messages, presence (online/offline), delivery acks, history.

### notifier
- Notify on: order confirmed, shipment assigned, courier nearby, delivered.

### Out of scope for v1
Selling to end consumers (POS), online payments, returns and re-delivery, promotions, multi-warehouse routing.

---

## 3. Non-functional requirements & scale assumptions

Numbers are **learning-scale but realistic**, chosen so load tests produce visible bottlenecks on a laptop cluster.

| Dimension | Assumption |
|---|---|
| Shops | 5,000 |
| Suppliers | 200 |
| Products (SKUs) | 50,000 |
| Couriers active at peak | 500 |
| Orders / day | 20,000 (peak 5× average, morning restock 8–11 AM) |
| Location updates | 500 couriers ÷ 5 s = **100 writes/s** sustained |
| Catalog reads | ~50× writes → cache hit ratio target ≥ 90% |
| Concurrent WebSocket connections | 2,000 |

| Quality | Target |
|---|---|
| Catalog read latency | p95 < 100 ms, p99 < 250 ms |
| Place order latency | p95 < 500 ms |
| Order placement availability | 99.5% (SLO), measured monthly |
| Stock correctness | **Zero overselling** under concurrent orders |
| Credit correctness | **Zero** orders beyond the credit limit under concurrency |
| Message processing | At-least-once delivery + idempotent consumers = effectively-once effects |
| Chat delivery | Messages delivered in order per conversation |

---

## 4. Core flow: place order (target design)

```text
Shop ──POST /orders──> ordering
                         │ 1. validate token (local JWT check)
                         │ 2. split cart per supplier
                         │ 3. check + hold credit per supplier   (PostgreSQL transaction)
                         │ 4. reserve stock  ──gRPC──> inventory (MySQL transaction)
                         │ 5. save order + outbox row            (same transaction)
                         ▼
                   outbox relay ──> Kafka: ordering.order.v1 {OrderPlaced}
                                         │
                   ┌─────────────────────┼──────────────────────┐
                   ▼                     ▼                      ▼
               shipping             notifier (via            catalog/search
           creates shipment         RabbitMQ command)        (analytics later)
```

Open design question (Challenge, Phase 5): what happens when supplier A reserves stock and supplier B fails? → Saga.

---

## 5. Roadmap

Each phase ends with: working deploy on `kind`, an entry in `learning-log.md`, and at least one "break it on purpose" experiment in `experiments.md`.

| Phase | Build | Main things learned |
|---|---|---|
| **0** | `kind` cluster (1 control plane + 3 workers, node pools), Gateway, namespaces, a hello-world Deployment | kind, kubectl, Pods, Deployments, ReplicaSets, Services, Gateway API, scheduling, labels/taints |
| **1** | `identity` (Django + PostgreSQL). Hand-written Postgres StatefulSet. Migrations as a Job. Secrets & ConfigMaps. Probes | StatefulSets, PVCs, headless services, secrets, readiness vs liveness, JWT/RS256, JWKS |
| **2** | `catalog` (FastAPI + MongoDB) + `web` (React login + product list). Redis cache-aside | Document modeling, JWT verification in another service, caching, invalidation, CORS |
| **3** | Observability baseline: Prometheus, Grafana, Fluent Bit DaemonSet → Elasticsearch → Kibana, correlation IDs | Metrics vs logs, DaemonSets, structured logging, first dashboards |
| **4** | `inventory` (Go + Gin + MySQL). `ordering` (Django + PostgreSQL) calling inventory over REST, then gRPC | Locking in MySQL vs PostgreSQL, overselling experiment, timeouts, deadlines, gRPC, protobuf |
| **5** | Kafka (Strimzi, KRaft). Outbox in `ordering`, idempotent consumers. Saga for multi-supplier orders | Partitions, keys, consumer groups, lag, outbox, inbox, Saga, compensations |
| **6** | `shipping` (Go stdlib + PostgreSQL + Redis GEO). RabbitMQ courier offers with TTL + DLX. `notifier` | Work queues, ack, DLQ, poison messages, first-accept-wins contention, geo queries |
| **7** | `chat` (FastAPI WebSockets + MongoDB + Redis Pub/Sub) | Connection management, heartbeats, presence, fan-out across pods, reconnect, ordering |
| **8** | Resilience: retries with backoff + jitter, circuit breakers, bulkheads, rate limiting. OpenTelemetry tracing | Failure modes, cascading failure experiment, distributed tracing |
| **9** | Elasticsearch product search fed by Kafka events | CDC-style indexing, eventual consistency between stores |
| **10** | Helm charts for our services, GitHub Actions CI/CD, architecture fitness functions | Packaging, pipelines, dependency rules, contract tests |
| **11** | Load testing (k6), HPA, replication (CloudNativePG, MySQL replica), chaos experiments | p95/p99, bottlenecks, autoscaling, failover |
| **12** | AWS: EKS or parts of it, DynamoDB for chat history, Terraform | Cloud networking, IaC, managed services trade-offs |
