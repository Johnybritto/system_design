# 3. Notification Service API — System Design (Apple SRE Practice)

> **Scenario:** A service that sends transactional **email, SMS and push notifications**. Reuse the API Gateway → EKS → PostgreSQL foundation, then add asynchronous workers and durable queues. Numerical objectives below are interview assumptions.

## Step 1 — Requirements

### Functional Requirements (4)
1. Accept email, SMS or mobile-push notification requests.
2. Honor recipient channel preferences and mandatory opt-outs.
3. Retrieve notification status and delivery history.
4. Retry transient provider failures, and isolate repeatedly failing requests.

### Non-Functional Requirements (4)
1. **Availability:** 99.95% request-acceptance SLO; deploy across three AZs.
2. **Durability:** Once accepted, do not lose a request; tolerate at-least-once delivery and retries.
3. **Latency:** p95 API request acceptance below 200 ms; delivery may be asynchronous.
4. **Scale:** Handle an illustrative burst of 100,000 notifications within 10 minutes, with independent channel scaling.

**Important:** `202 Accepted` means the job was durably stored, **not that the message reached the recipient**.

---

## Step 2 — API Contracts (Short)

| Action | Endpoint |
|---|---|
| Submit notification | `POST /v1/notifications` |
| Get status | `GET /v1/notifications/{id}` |
| Get preferences | `GET /v1/users/me/notification-preferences` |
| Update preferences | `PATCH /v1/users/me/notification-preferences` |

**Request**
~~~http
POST /v1/notifications
Authorization: Bearer <access_token>
Idempotency-Key: order123-confirmed-U123
~~~
~~~json
{"recipient_id":"U123","channel":"EMAIL","template_id":"order-confirmed","variables":{"order_id":"ORD123"}}
~~~

**Response — 202 Accepted**
~~~json
{"notification_id":"N1001","status":"PENDING"}
~~~

Validate who is allowed to notify the recipient; do not allow an authenticated user to send arbitrary messages to everyone.

---

## Step 3 — Core Entities

| PostgreSQL table | Important fields | Purpose |
|---|---|---|
| **NotificationRequests** | notification_id PK, tenant_id, recipient_id, channel, template_id, payload_ref, status, idempotency_key, created_at | Durable request and status |
| **NotificationPreferences** | user_id, channel, enabled, updated_at | Consent and opt-outs |
| **DeliveryAttempts** | attempt_id PK, notification_id FK, provider, attempt_no, status, error_code, attempted_at | Attempts, audit and troubleshooting |
| **Outbox** | event_id PK, notification_id, event_type, payload, published_at | Reliable notification-event publication |

**Key indexes and constraints:** `UNIQUE(tenant_id, idempotency_key)`, `NotificationRequests(tenant_id, recipient_id, created_at DESC)`, `DeliveryAttempts(notification_id, attempted_at DESC)` and index of unpublished Outbox events.

**Retention:** Define retention for notification metadata and sensitive message content separately. Minimize PII; expire old delivery logs and DLQ payloads according to business/compliance policy.

---

## Step 4 — Database Model: Why?

**PostgreSQL Multi-AZ:** Structured notifications, preferences, delivery-attempt records and idempotency need relational constraints and ACID updates.

**SQS Standard queues:** Separate **Email**, **SMS**, and **Push** queues so one provider outage does not block other channels. SQS buffers work durably and supports visibility timeout, retries and DLQ. Standard SQS is **at least once**, so workers must handle duplicates.

**Redis (optional):** Rate limits and short-lived preference cache. **Not** the source of truth.

**Why SQS, not Kafka initially?** Independent delivery jobs fit SQS well. Kafka is attractive if many independent consumers need replayable event streams or partition-key ordering.

**Why an Outbox table?** The API cannot atomically commit a PostgreSQL write and publish directly to SQS in one distributed transaction. Insert the notification **and its Outbox event in one DB transaction**, then publish asynchronously.

**Provider caveat:** Exactly-once email/SMS/push delivery across an external provider is not generally guaranteed. Use idempotency, provider callbacks and reconciliation as available.

---

## Step 5 — High-Level Design

~~~text
Client → Route 53 (DNS) → API Gateway → Private VPC Link / NLB
       → EKS Notification API
       → PostgreSQL transaction: NotificationRequests + Outbox (COMMIT)
       → HTTP 202 Accepted

Outbox Publisher → SQS Email / SMS / Push queues
       → Channel-specific EKS Workers → External providers
       → DeliveryAttempts + Notification status in PostgreSQL

Transient failure → retry/backoff → DLQ on repeated failure
~~~

Only the **durable acceptance path** is synchronous. Provider delivery is asynchronous.

---

## Step 6 — Deep Dive: Exact Flows

### A. Submit and send a notification

1. Client sends `POST /v1/notifications` with JWT and idempotency key.
2. API checks recipient authorization, template, channel preference and opt-out. Use a trusted authenticated sender/tenant identity.
3. Start a PostgreSQL transaction: insert `NotificationRequests(PENDING)` plus `Outbox(NotificationRequested)`; enforce unique idempotency key; **COMMIT**.
4. Return `202 Accepted` plus notification ID. A duplicate client retry with the same key returns the existing request.
5. Outbox Publisher reads committed, unpublished events and sends each to the matching SQS channel queue; mark published only **after** successful queue publication.
6. Email, SMS or push workers poll SQS, impose provider-aware concurrency limits and send using SES, an SMS provider or APNs/FCM.
7. Record provider acceptance/failure and a `DeliveryAttempts` row in PostgreSQL. Where providers offer verified status callbacks, update `DELIVERED` or bounce/failure status.
8. Acknowledge the SQS message after durable processing. Workers are idempotent because SQS may redeliver.

**State nuance:** `SENT` / provider-accepted does not necessarily mean `DELIVERED` to a user's device.

### B. Email provider goes down for 10 minutes

- API can still accept requests if PostgreSQL is healthy. Do **not** hold HTTP requests until the email provider recovers.
- Email SQS backlog grows; other channel queues continue.
- Workers use bounded retries, backoff/jitter and configured SQS visibility timeouts. Exceeded retry limits route messages to DLQ.
- Do not scale failed-provider workers without limit: it causes a retry storm. Once recovered, drain the backlog gradually within provider quotas.
- Monitor `ApproximateAgeOfOldestMessage`, queue depth, processing success, provider errors and DLQ count.

### C. Publisher or worker crash

**Publisher crash after sending to SQS but before recording success:** It may publish the same event again on restart. Worker checks notification/event ID to deduplicate processing.

**Worker crash after provider accepts a send:** The worker might not know whether delivery occurred. Query/reconcile status if supported and apply provider idempotency keys where available; cannot promise exactly-once external delivery.

### D. Consumer scaling

- KEDA may scale worker Deployments by SQS backlog; HPA can use suitable queue metrics.
- Karpenter scales worker-node capacity when new Pods are Pending.
- Scale per channel. More Pods cannot repair an unavailable provider.
- Bound database connections and provider concurrency independently.

---

## Step 7 — Why Each Component?

| Component | Why we use it |
|---|---|
| Route 53 | DNS lookup for API hostname |
| API Gateway | JWT validation, routing, coarse throttling |
| VPC Link + Internal NLB + Ingress | Private connectivity to EKS |
| Notification API Pods | Validation, permissions, preferences and idempotency |
| PostgreSQL Multi-AZ | Durable status, relational preferences, attempt log, Outbox |
| Outbox Publisher | Reliable retryable DB → queue handoff |
| SQS Email / SMS / Push | Durable decoupling and provider isolation |
| Delivery Worker Pods | Async send, timeout, retry and status updates |
| DLQ | Investigate and replay poison/repeated failures |
| SES, SMS provider, APNs / FCM | Actual recipient transport |
| Redis (optional) | Rate limits and safe cached preference reads |
| KEDA/HPA + Karpenter | Worker count and node capacity scaling |
| CloudWatch / Grafana / APM | SLOs, provider failures, queue lag and tracing |
| KMS + TLS | Encrypt notification information in transit and at rest |

---

## Step 8 — Failure, Remediation and Verification

| Failure | Impact | Remediation | Recovery signal |
|---|---|---|---|
| PostgreSQL fails | API can't durably accept jobs | Multi-AZ failover, reconnect, bounded safe retries | `202` acceptance returns |
| Outbox Publisher crashes | Delayed publication | Restart; scan pending events; idempotent replay | Outbox lag decreases |
| SQS unavailable | Delivery delayed | Keep committed Outbox rows, retry publish | Queue publishing resumes |
| Email provider down | Email backlog grows | Isolate channel, bounded retry, DLQ, gradual drain | Queue age and errors decline |
| Worker crashes / duplicate | Redelivery, duplicate provider attempt | SQS visibility timeout; idempotency/reconciliation | Duplicate rate controlled |
| Poison message | Persistent retries | DLQ + alert + safe replay after fix | DLQ resolves |
| Sudden burst | Queue age grows | Scale consumer Pods within provider quotas | End-to-end delay returns to SLO |
| Preferences stale | Unwanted notifications | Validate opt-outs before send, invalidate caches | Opt-outs correctly enforced |
| AZ fails | Fewer EKS workers | Multi-AZ placement + spare headroom | API and channel workers healthy |

**Monitor:** API errors and p95, acceptance rate, oldest pending notification, SQS queue age/depth, provider 429/5xx, attempts, DLQs and final delivery outcomes when observable.

---

## Interview Summary

> I would accept authorized notifications through API Gateway and EKS, persist each request with an Outbox event atomically in PostgreSQL, and return HTTP 202. An Outbox Publisher would send jobs to channel-specific SQS queues. Idempotent workers would call email, SMS and push providers, recording delivery status and retries. I would isolate provider outages, use DLQs, and monitor end-to-end delivery delay in addition to API availability.

**Main change from Order API:** provider/channel-specific asynchronous work, delivery-state tracking, retries and DLQ handling.

---

## Detailed End-to-End Diagram

~~~mermaid
flowchart TD
  C["Client or Internal Service"]
  DNS["Route 53 — DNS"]
  GW["API Gateway — JWT and throttling"]
  LINK["VPC Link + Internal NLB"]
  ING["EKS Ingress"]

  subgraph EKS["EKS Platform — 3 AZs"]
    API["Notification API Pods"]
    PUB["Outbox Publisher Pods"]
    EMAIL["Email Worker Pods"]
    SMS["SMS Worker Pods"]
    PUSH["Push Worker Pods"]
    CALLBACK["Callback Handler"]
  end

  subgraph DATA["Durable State"]
    PG["RDS PostgreSQL Multi-AZ"]
    REQ["NotificationRequests"]
    PREF["NotificationPreferences"]
    ATT["DeliveryAttempts"]
    OB["Outbox"]
    R["Redis — optional cache / limits"]
  end

  EQ["SQS Email Queue"]
  SQ["SQS SMS Queue"]
  PQ["SQS Push Queue"]
  DLQ["Per-channel Dead-Letter Queues"]
  SES["SES / Email Provider"]
  SP["SMS Provider"]
  AP["APNs / FCM"]
  OBS["CloudWatch / Grafana / Tracing"]

  C -. "Resolve DNS" .-> DNS
  C -->|"POST /notifications"| GW
  GW --> LINK --> ING --> API
  API -->|"Validate consent"| PREF
  API -. "Optional limits" .-> R
  API -->|"Atomic notification + outbox commit"| PG
  PG --> REQ
  PG --> OB
  PG --> ATT
  PG --> PREF
  API -->|"202 Accepted after commit"| C

  OB -->|"Unpublished committed events"| PUB
  PUB --> EQ
  PUB --> SQ
  PUB --> PQ

  EQ --> EMAIL --> SES
  SQ --> SMS --> SP
  PQ --> PUSH --> AP

  EMAIL -->|"Record attempt / status"| PG
  SMS -->|"Record attempt / status"| PG
  PUSH -->|"Record attempt / status"| PG

  SES -. "Verified callback" .-> CALLBACK
  SP -. "Verified callback" .-> CALLBACK
  AP -. "Feedback if supported" .-> CALLBACK
  CALLBACK -->|"Update delivery status"| PG

  EQ -. "Retry limit exceeded" .-> DLQ
  SQ -. "Retry limit exceeded" .-> DLQ
  PQ -. "Retry limit exceeded" .-> DLQ
  DLQ -. "Alert and replay" .-> OBS
  C -->|"GET /notifications/id"| GW
  API -->|"Authorized status query"| PG

  API -. "API SLIs" .-> OBS
  PUB -. "Outbox lag" .-> OBS
  EQ -. "Queue age" .-> OBS
  SQ -. "Queue age" .-> OBS
  PQ -. "Queue age" .-> OBS
  EMAIL -. "Provider errors" .-> OBS
  SMS -. "Provider errors" .-> OBS
  PUSH -. "Provider errors" .-> OBS
~~~

**Diagram reading:** Synchronous request ends with a committed `PENDING` notification and HTTP 202. The publisher queues delivery by channel, workers deliver asynchronously, and callbacks update final states. DLQs and observability protect the workflow when providers are unavailable.
