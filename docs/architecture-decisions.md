# Architecture Decision Records

Format: **Context → Decision → Alternatives → Trade-offs → Consequences.**
Status is one of: Proposed, Accepted, Superseded.

---

## ADR-001: Microservices from day one, for learning

**Status:** Accepted — 2026-09-28

**Context**
This is a learning lab. The goal is hands-on experience with Kubernetes, multiple languages, multiple databases, Kafka, RabbitMQ, Redis and observability tooling. The business domain matters less than the infrastructure. A real team of one with no traffic would build a monolith.

**Decision**
Design the target as ~7 services from the start. Build and deploy them **one at a time**, in a fixed order (see `system-design.md` → Roadmap).

**Alternatives**
- Monolith → modular monolith → extract services (the "earned complexity" path).
- Fewer, larger services (3 instead of 7).

**Trade-offs**
- ✅ Real multi-service infrastructure early: networking, auth between services, deployment, partial failure.
- ✅ Each language/framework/database gets its own home.
- ❌ We skip the experience of *discovering* where the boundaries should be. Boundaries are chosen up front and some will be wrong.
- ❌ Distributed-system costs (network failures, eventual consistency, ops overhead) arrive before the business needs them.

**Consequences**
- We will deliberately review boundaries after `ordering` and `shipping` exist, and write an ADR if one is wrong.
- In interviews, the honest answer is: "For a real product at this size I'd start with a modular monolith; this was a learning decision."

---

## ADR-002: Polyglot persistence — one datastore per service

**Status:** Accepted — 2026-09-28

**Context**
We want practical exposure to PostgreSQL, MySQL, MongoDB, Redis and Elasticsearch, and each service must own its data.

**Decision**

| Service | Store | Real reason | Learning reason |
|---|---|---|---|
| `identity` | PostgreSQL | Relational users/roles, constraints | — |
| `catalog` | MongoDB | Products of different shapes (grocery vs clothing variants) | Document modeling, indexes on nested fields |
| `inventory` | MySQL | Transactional stock rows | **Compare InnoDB locking/MVCC/binlog with PostgreSQL** |
| `ordering` | PostgreSQL | Money, multi-row transactions, constraints | Isolation levels, advisory locks |
| `shipping` | PostgreSQL + Redis GEO | Shipment state in PG, hot locations in Redis | Geo queries |
| `chat` | MongoDB | Append-only messages per conversation | Time-ordered document access patterns |

DynamoDB is deferred to the AWS phase (candidate: chat message history).

**Alternatives**
PostgreSQL everywhere (the realistic choice for a small team: `jsonb` covers catalog, one ops skill set).

**Trade-offs**
- ✅ Hands-on comparison of engines under the same workload.
- ❌ Five storage engines to run, back up, monitor and upgrade. In a real company this operational cost usually outweighs the benefit.

**Consequences**
`inventory` on MySQL is a *learning* choice and is labeled as such.

---

## ADR-003: Kafka for events, RabbitMQ for commands

**Status:** Accepted — 2026-09-28

**Context**
Services need to react to each other's state changes, and some work must be handed off to a worker.

**Decision**
- **Kafka** carries domain events ("order placed", "stock reserved"). Many consumers, replay, per-key ordering.
- **RabbitMQ** carries commands/tasks ("send this notification", "offer this shipment to couriers"). One consumer per message, per-message ack, TTL, dead-lettering, routing.

**Alternatives**
- Kafka only (commands as topics). Works, but per-message TTL/delay and competing-consumer work queues are awkward.
- RabbitMQ only (streams or fanout exchanges for events). Works, but replay and long retention are weaker.
- Redis Streams for both. Simpler, fewer guarantees.

**Trade-offs**
- ✅ Each broker used for what it does best; we learn both mental models.
- ❌ Two brokers to operate. Most real teams at this scale pick one.

**Consequences**
Every message definition declares which broker it belongs to and why.

---

## ADR-004: JWT verified locally by each service

**Status:** Accepted — 2026-09-28

**Context**
Every request must be authenticated. Calling `identity` on each request couples every service's availability and latency to it.

**Decision**
`identity` issues short-lived access tokens (15 min) signed with RS256 and long-lived refresh tokens. Services verify access tokens locally with the cached public key (JWKS).

**Alternatives**
- Opaque tokens + introspection call per request.
- Session cookies with a shared session store.
- Verification at the gateway only.

**Trade-offs**
- ✅ `identity` down ≠ whole system down.
- ❌ Revocation is not instant: a revoked token stays valid until it expires (≤ 15 min). Mitigation later: denylist in Redis for critical cases.

**Consequences**
Key rotation must be supported (multiple keys in JWKS, `kid` header).

---

## ADR-005: Node pools instead of one node per service

**Status:** Accepted — 2026-09-28

**Context**
The initial idea was "node 1 runs FastAPI + MongoDB, node 2 runs Django + PostgreSQL, node 3 runs Go".

**Decision**
Use node **pools** (`apps` and `data`) and let the scheduler place pods. Spread replicas of each service across nodes.

**Alternatives**
Pin each service to a dedicated node with `nodeSelector`.

**Trade-offs of pinning (why rejected)**
- ❌ If that node dies, the whole service dies — replicas on one node give no failure isolation.
- ❌ Wasted capacity: one node idle while another is full.
- ❌ Fights the scheduler instead of using it.
- ✅ It is only justified for special hardware (GPU, local SSD) or strict isolation.

**Consequences**
We'll run an experiment: kill `worker-1` and verify every service stays up.
