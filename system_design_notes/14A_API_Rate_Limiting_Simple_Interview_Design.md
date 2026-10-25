# 4. API Rate Limiting Service — Concise System Design (Interview Version)

> **Goal:** Reuse our API Gateway → EKS architecture and prevent one authenticated user from exceeding a shared quota across any number of application Pods. Keep the **last diagram easy to redraw from memory**. Example target: **100 requests per user per rolling 60 seconds**, not a statement about AWS quotas.

## Step 1 — Requirements

### Functional (4)
1. Allow or reject requests based on a configurable per-user, per-tenant and per-API limit.
2. Enforce **one shared limit across all EKS Pods**, rather than one limit per Pod.
3. Return **HTTP 429** plus `Retry-After` when the quota is exceeded.
4. Let authorized administrators change quota policies and exceptions.

### Non-functional (4)
1. **Latency:** p99 rate-limit decision under 10 ms (interview target).
2. **High availability:** Multi-AZ deployment; define behavior if Redis fails.
3. **Consistency:** Atomic quota decisions, no concurrency races during normal operation.
4. **Scalability:** Handle a sample 10,000 incoming RPS and protect downstream services.

## Step 2 — Short API Contracts

| Use | Endpoint |
|---|---|
| Internal decision | `POST /internal/rate-limit/check` |
| Read policy | `GET /v1/rate-limit-policies/{id}` |
| Update policy | `PUT /v1/rate-limit-policies/{id}` |

Input to internal check: `tenant_id`, validated `user_id`, route and policy ID. Output: `allowed`, `remaining` and `retry_after_seconds`.

If denied: `HTTP 429 Too Many Requests` and `Retry-After: 18`.

**Crucial:** Derive user identity from a verified JWT claim (`sub`) and tenant context, not a client-controlled request header.

## Step 3 — Core Entities / Indexes

| Entity | Storage and fields | Reason |
|---|---|---|
| `RateLimitPolicies` | PostgreSQL: policy_id, tenant_id, route, algorithm, max_requests, window_seconds, version | Durable configurable rule |
| `PolicyOverrides` | PostgreSQL: tenant_id, user_id, route, policy_id | Premium customers/tenant exceptions |
| `PolicyAudit` | PostgreSQL: audit_id, actor, policy_id, old/new value, changed_at | Trace changes |
| `RequestWindow` | Redis: `rate:{tenant}:{user}:{route}` → timestamp entries, TTL | Live distributed quota state |

**Indexes:** primary key on policy_id; unique applicable tenant+route policy; override index on tenant/user/route; policy audit index on policy_id+time. Redis TTL evicts expired request history automatically.

## Step 4 — Database Selection: Why?

**Redis / ElastiCache:** Low-latency **shared atomic state** across 20+ Pods. Atomic Redis Lua/Function avoids read-then-increment races.

**PostgreSQL Multi-AZ:** Durable policy, exception and audit data; no high-frequency request writes to PostgreSQL.

**Why not individual Pod counters?** Alice sends 60 requests via Pod A and 60 via Pod B: two independent counters would incorrectly allow 120.

**Why not API Gateway alone?** Managed throttling is suitable for a coarse first layer. It is best-effort and may not satisfy an *exact*, custom, identity-aware global quota across services.

**Consistency caveat:** Redis replicas can lag during failover. If the business requires an absolutely strict cross-failover ceiling, a stronger coordination approach or explicit fail-closed policy is needed.

## Step 5 — High-Level Design

**Client → DNS → API Gateway (JWT + coarse throttle) → EKS middleware / limiter → Redis atomic check → allow Application or reject HTTP 429.**

The rate-limit check occurs **before** expensive business logic or PostgreSQL work. A separate admin API stores policies in PostgreSQL, which the limiter caches with versioning.

## Step 6 — Deep Dive (Exact Flow)

### A. Alice: 60 requests on Pod A + 60 on Pod B
1. API Gateway authenticates Alice and applies coarse throttling.
2. EKS gateway/middleware sends trusted user, tenant and normalized route to the Rate Limiter.
3. Rate Limiter uses one Redis key, e.g. `rate:T1:alice:GET-orders`.
4. **Atomic operation:** discard timestamps older than 60 seconds → count entries → permit if count is below 100 → append unique request ID/timestamp → set expiry.
5. The first 100 requests are **allowed**; the other 20 are denied with **429** (assuming all 120 fall in the same 60-second interval, without prior usage).
6. Only allowed requests reach the application and its database.

### B. Algorithm choice

| Algorithm | Best for | Drawback |
|---|---|---|
| Fixed-window counter + TTL | Cheap approximate quota | Window-boundary double burst |
| Token bucket | Bursty traffic with bounded average rate | Does not guarantee exact 100 in *every* rolling minute |
| Sliding-window log in Redis sorted set | Strict rolling-window count during healthy Redis operation | Memory and atomic operation cost |

**Choice here:** Sliding window, because the requirement says **maximum 100 in *any* 60-second window**. A counter with a 60s TTL is easier but does not precisely implement that rule.

### C. Why atomic?
Do **not** perform `GET count → compare → INCR` as separate commands: two Pods can simultaneously see 99 and both admit a request. One server-side atomic operation makes the decision against shared state.

### D. Redis failure — interview trade-off
- **Fail-open:** Preserve API availability, risk overload and quota violation.
- **Fail-closed:** Preserve strict access control, may reject legitimate users.
- **Controlled fallback:** Conservative local admission + global gateway throttling + bounded backend concurrency. **Not** an exact shared limit.

Select a policy based on business criticality; e.g. fail-closed for abuse-sensitive paid write APIs, controlled fallback for some low-risk reads.

### E. Scaling and monitoring
- HPA scales the limiter; Karpenter scales nodes. Monitor Redis latency/hot keys; adding Pods does not eliminate a hot shared quota key.
- Metrics: allowed/denied requests, `429` by tenant/route, decision p95/p99, Redis CPU/latency, backend 5xx and policy propagation lag.

## Step 7 — Why Every Component?

| Component | Why |
|---|---|
| Route 53 | DNS |
| API Gateway | JWT validation and coarse throttling |
| EKS Ingress / middleware | Shared interception point before business processing |
| Rate Limiter Pods | Decide using policy + Redis |
| ElastiCache Redis | Atomic global counters / rolling windows |
| PostgreSQL | Persistent policies and audit history |
| HPA / Karpenter | Scale Pods and nodes |
| CloudWatch / Grafana | Rate-limit accuracy, p99 and backend overload alerts |

## Step 8 — Failures and Remediation

| Failure | Remediation |
|---|---|
| Redis down | Explicit fail-open/fail-closed/controlled fallback; alert; Multi-AZ failover |
| Redis hot key / high latency | Inspect cardinality and hot tenants; optimize scripts or quota placement |
| Concurrent over-admission | Atomic Lua/Redis Function, not separate GET and INCR |
| Wrong policy pushes legitimate 429s | Version, audit, rollback the policy |
| Backend saturated despite quotas | Separate global concurrency cap, backpressure and load shedding |
| Limiter Pods unavailable | Multi-AZ replicas and bounded timeout/fallback |

**Interview-ready answer:** "I would authenticate the client at API Gateway, apply shared per-user rate limiting before the application, and use atomic Redis operations to enforce the rolling window across Pods. PostgreSQL stores policies, and excess requests get HTTP 429. I would explicitly choose the failure behavior for Redis."

---

## Easy-to-Remember Final Flow

~~~mermaid
flowchart TD
 A["Client"] --> B["API Gateway — JWT"]
 B --> C["EKS Rate Limiter"]
 C --> D["Redis — atomic shared count"]
 D --> E{"Under 100 in 60s?"}
 E -->|"Yes"| F["Application API"]
 E -->|"No"| G["HTTP 429"]
~~~

**Memory line:** **Client → Gateway → Limiter → Redis → Allow or 429.**
