# System Design — Order Management API

We'll reuse our existing Customer-Facing API architecture and introduce **order transactions, idempotency, inventory reservation, payment processing and asynchronous events**.

## Step 1 — Requirements

### Functional Requirements (4)

1. Users should be able to create an order containing multiple products.
2. Users should be able to retrieve orders and view their status.
3. Users should be able to cancel eligible orders.
4. The system should coordinate inventory reservation and payment before confirming an order.

### Non-Functional Requirements (4)

1. **Availability:** 99.95% availability SLO, deployed across three AZs.
2. **Consistency:** Strong consistency for order updates; prevent duplicate orders and payments.
3. **Scalability:** Support 2,000 average RPS and 10,000 peak RPS (assumed interview numbers).
4. **Performance and reliability:** p95 read latency below 200 ms; durable orders, bounded retries and recoverable failures.

---

## Step 2 — API Contracts

| Operation | API endpoint |
|---|---|
| Create order | `POST /orders` |
| Get order | `GET /orders/{order_id}` |
| List customer orders | `GET /orders` |
| Cancel order | `POST /orders/{order_id}/cancel` |

**Create Order — Request**

~~~http
POST /orders
Authorization: Bearer <token>
Idempotency-Key: abc123
~~~

~~~json
{
  "items": [
    {"product_id": "P101", "quantity": 2}
  ]
}
~~~

**Response — HTTP 201 Created**

~~~json
{
  "order_id": "ORD123",
  "status": "PENDING"
}
~~~

`PENDING` means the order was durably created, but inventory reservation and payment processing have not yet finished. The authenticated customer comes from the token, not an untrusted request field.

---

## Step 3 — Core Database Entities

| Table | Important columns | Purpose |
|---|---|---|
| **Users** | user_id (PK), name, email | Customer information |
| **Orders** | order_id (PK), user_id (FK), status, total_amount, currency, idempotency_key, created_at, updated_at | Main order information |
| **Order_Items** | order_item_id (PK), order_id (FK), product_id, quantity, unit_price | Products belonging to an order |
| **Outbox** | event_id (PK), order_id, event_type, payload, published_at | Reliable asynchronous event publishing |

**Database indexes:**

- `Orders(order_id)` — Primary key index for fast order lookup.
- `Orders(user_id, created_at DESC)` — Efficient customer order history.
- `UNIQUE(user_id, idempotency_key)` — Prevents duplicate order creation for the same customer and request.

Inventory reservations and payment transactions will be stored by their respective services, if those are separated from the Order Service.

---

## Step 4 — Database Selection

**Choice: Amazon RDS PostgreSQL (Multi-AZ)**

| Why PostgreSQL? | Explanation |
|---|---|
| Structured data | Orders and order items have defined schemas |
| Relationships | One order contains multiple order items |
| ACID transactions | Insert Orders, Order_Items and Outbox atomically |
| Strong consistency | Prevent inconsistent order states |
| Indexing | Fast retrieval of orders by ID and customer |
| High availability | RDS Multi-AZ offers automatic standby failover |

### Why not DynamoDB?

DynamoDB can also support transactional writes and strong consistency. However, PostgreSQL is a natural starting choice because the order model has relational data, constraints and transactional updates.

At significantly larger scale, DynamoDB could be evaluated for specific key-based access patterns.

**Scaling choice:** One writable PostgreSQL primary, Multi-AZ standby for HA, and optional read replicas for order history. A standard RDS Multi-AZ standby isn't a read-scaling replica.

---

## Step 5 — High-Level Design

~~~mermaid
flowchart TD
    A["Client — POST /orders"] --> B["Route 53 → API Gateway"]
    B --> C["VPC Link → Internal NLB"]
    C --> D["EKS Ingress → Kubernetes Service"]
    D --> E["EKS Order Service Pods"]
    E --> F["PostgreSQL Transaction"]
    F --> G["Orders + Order_Items + Outbox"]
    G --> H["Atomic COMMIT"]
    H --> I["HTTP 201 — Order PENDING"]
    H --> J["Outbox Publisher"]
    J --> K["Amazon SQS"]
    K --> L["Order Worker Pods"]
    L --> M["Inventory + Payment"]
    M --> N["Update order status — CONFIRMED / FAILED"]
~~~

**Important:** The database commit and SQS processing happen at different times. The API does not wait for inventory or payment completion in this design.

Redis and read replicas support the separate GET/order-history path, rather than being required on every order write.

---

## Step 6 — Deep Dive

### A. Create Order — Exact Flow

1. Client sends `POST /orders` with an idempotency key.
2. API Gateway validates JWT; Order Service verifies the customer and request.
3. Order Service validates product IDs and obtains authoritative prices. It calculates the total instead of trusting client-submitted prices.
4. PostgreSQL transaction inserts `Orders (PENDING)`, `Order_Items` and `Outbox (OrderCreated)` and commits atomically.
5. API returns `201 Created` with the order ID.
6. Outbox Publisher reads committed events and publishes to SQS, marking delivery progress after successful publication.
7. Worker receives the event, attempts inventory reservation and payment through their services.
8. Worker updates the order to `CONFIRMED` or `FAILED` and records the resulting outbox event transactionally.

A database failure before commit rolls back the initial inserts together.

### B. Why do we need an Outbox table?

Consider this failure:

~~~text
Order saved in PostgreSQL
           |
           v
Attempt to publish to SQS
           |
           X
SQS unavailable
~~~

Without the Outbox pattern, the order exists, but downstream processing might never start.

With the Outbox pattern, the event is stored in the **same PostgreSQL transaction** as the order. The publisher can retry later. This avoids the database-plus-queue dual-write problem.

### C. Idempotency — Prevent duplicate orders

Alice submits an order, but her connection times out after PostgreSQL commits it.

She retries with the same `Idempotency-Key`.

The Order Service looks up the committed request and returns the existing order instead of creating another. A unique constraint enforces this even under concurrent retries.

**Also:** Payment and inventory workers need their own idempotency protection. SQS Standard can deliver duplicate messages.

### D. What if payment succeeds but inventory reservation fails?

This is a **distributed transaction problem**.

PostgreSQL ACID cannot atomically cover an external payment provider and an independent inventory service.

We use a **Saga / compensating actions** approach:

- Reserve inventory before initiating payment whenever possible.
- If payment fails, release reserved inventory.
- If payment succeeds but the order cannot be completed, initiate a compensating refund or void.
- Keep the order in an intermediate state until compensation is confirmed; retry and reconcile uncertain results.

---

## Step 7 — Why Each Component?

| Component | Why we're using it |
|---|---|
| Route 53 | Resolve the public API hostname |
| API Gateway | Centralized JWT authentication, API throttling and routing |
| Internal NLB + EKS Ingress | Forward requests into the private Kubernetes application |
| EKS Order Service | Run order-validation and transaction logic; horizontally scalable |
| PostgreSQL Multi-AZ | Reliable relational storage, ACID transactions and availability |
| RDS Proxy / bounded pool | Manage database connections across many Pods |
| Redis | Cache eligible order-history and status reads; not the source of truth |
| Read replicas | Scale stale-tolerant order-history reads |
| Outbox Publisher | Reliably transfer committed DB events to the queue |
| SQS | Buffer asynchronous jobs and decouple processing from client requests |
| Order Workers | Process inventory and payment workflows independently |
| Inventory Service | Own stock availability and reservations |
| Payment Service | Own payment state, retries and reconciliation |
| CloudWatch / Grafana / APM | Monitor success rate, latency, queue backlog and dependencies |

**Why SQS instead of Kafka?** SQS is a simpler starting point for background processing of individual order jobs. If several independent downstream systems need the same event history, Kafka—or SNS fanout to separate SQS queues—becomes worth considering.

---

## Step 8 — Important Failures and Remediation

| Failure | Impact | Remediation |
|---|---|---|
| Duplicate POST | Duplicate orders | Idempotency key + database uniqueness |
| PostgreSQL primary failure | Temporary write unavailability | Multi-AZ failover, reconnect, safe retries |
| SQS unavailable | Order processing delayed | Durable Outbox + publisher retries |
| Worker crashes | Incomplete order processing | SQS visibility timeout, idempotent retry, DLQ |
| Inventory unavailable | Order stuck PENDING | Timeout, bounded retries, compensating workflow |
| Payment timeout | Payment outcome uncertain | Query payment status before retry; idempotency |
| Redis fails | More DB reads | Bounded database fallback and rate limiting |
| Heavy traffic | API/DB saturation | HPA, Karpenter, caching, connection limits, load shedding |
| One AZ fails | Lost Pods or DB primary | Multi-AZ capacity, scheduler recovery, DB failover |

For monitoring, track API 5xx, p95/p99 latency, order creation failures, time spent in `PENDING`, SQS oldest-message age, DLQ count, PostgreSQL saturation, inventory failures and payment reconciliation backlog.

---

## Final Interview Summary

> I would design an Order Management API using API Gateway, EKS and PostgreSQL. PostgreSQL is my source of truth because orders require relational data and ACID transactions.
>
> During order creation, I would atomically commit the order, order items and an outbox event. An Outbox Publisher would reliably send the event to SQS, where workers coordinate inventory and payment.
>
> I would use idempotency to prevent duplicate orders, a Saga approach for distributed failures, Redis for eligible read caching, and RDS Multi-AZ for database availability.
>
> Finally, I'd monitor customer-facing SLOs, database saturation, queue backlog and order-processing failures.

**The three most important concepts that differentiate this from our Document Metadata API are: transactional Outbox, idempotency and distributed transaction compensation.**

---

## Detailed End-to-End Architecture Diagram — Order Management API

~~~mermaid
flowchart TD
    CL["Customer / Browser / Mobile App"]
    DNS["Route 53 — DNS lookup"]
    GW["API Gateway — HTTPS, JWT, throttling"]
    VPC["Private VPC Link"]
    NLB["Internal NLB"]
    ING["EKS Ingress Controller"]
    SVC["Kubernetes Service"]

    subgraph EKS["EKS — Order Platform across 3 AZs"]
      API["Order API Pods — validate, price, idempotency"]
      PUB["Outbox Publisher Pods"]
      WORK["Order Worker Pods — idempotent"]
    end

    subgraph DATA["Database / Caching"]
      PR["RDS Proxy / bounded connections"]
      PG["RDS PostgreSQL Primary — source of truth"]
      ST["Multi-AZ Standby — HA failover"]
      RR["Optional read replicas — stale-tolerant reads"]
      CACHE["Redis — order history / status cache"]
    end

    subgraph TX["Single PostgreSQL ACID transaction"]
      ORD["Orders — PENDING"]
      ITEMS["Order_Items — product snapshot"]
      OB["Outbox — OrderCreated"]
      COMMIT["COMMIT — all or nothing"]
    end

    Q["Amazon SQS — durable async jobs"]
    DLQ["Dead-Letter Queue — repeated failures"]
    INV["Inventory Service — reserve stock"]
    PAY["Payment Service — charge / verify"]
    OK{"Inventory and payment successful?"}
    CONF["PostgreSQL — CONFIRMED + outbox event"]
    FAIL["PostgreSQL — FAILED / compensating state"]
    COMP["Release inventory / refund or void as needed"]
    RES["HTTP 201 — order_id, PENDING"]
    RET["GET /orders — cached eligible reads"]
    OBS["CloudWatch / Grafana / APM — SLOs and tracing"]

    CL -. "Resolve hostname" .-> DNS
    CL -->|"POST /orders + Idempotency-Key"| GW
    GW --> VPC
    VPC --> NLB
    NLB --> ING
    ING --> SVC
    SVC --> API

    API -->|"Atomic insert with unique idempotency key"| PR
    PR --> PG
    PG --> ORD
    ORD --> ITEMS
    ITEMS --> OB
    OB --> COMMIT
    COMMIT --> RES
    RES --> CL

    PG -. "Synchronous standby replication" .-> ST
    PG -. "Asynchronous read replication" .-> RR

    PG -->|"Poll unpublished committed events"| PUB
    PUB -->|"Publish OrderCreated"| Q
    Q -->|"Deliver message"| WORK
    WORK -->|"Reserve first"| INV
    INV -->|"If inventory reserved"| PAY
    PAY --> OK
    OK -->|"Yes"| CONF
    OK -->|"No / uncertain"| FAIL
    FAIL --> COMP
    CONF --> PG
    FAIL --> PG
    Q -. "After repeated failed processing" .-> DLQ

    CL -->|"GET /orders or GET /orders/{id}"| GW
    API --> RET
    RET --> CACHE
    CACHE -. "Cache miss" .-> RR
    RET -. "Strong consistency / fallback" .-> PR

    API -. "Metrics / traces" .-> OBS
    PUB -. "Publication lag / errors" .-> OBS
    WORK -. "Queue age / failures" .-> OBS
    PG -. "CPU / locks / connection limits" .-> OBS
    Q -. "Backlog / DLQ alerts" .-> OBS
~~~

**Reading the diagram:** The synchronous request ends with the committed `PENDING` order and HTTP 201. After that, the Outbox Publisher and SQS trigger idempotent inventory/payment workers. Success confirms the order; failures enter a compensating workflow. Redis and read replicas support GET requests, while RDS Multi-AZ and multi-AZ EKS placement support availability.
