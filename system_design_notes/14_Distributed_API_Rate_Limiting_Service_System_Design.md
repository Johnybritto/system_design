# 4. Distributed API Rate Limiting Service — System Design (Apple SRE Practice)

> **Scenario:** All EKS application Pods must enforce a **shared per-user limit**, not a separate limit per Pod. Reuse Route 53 → API Gateway → EKS. This document distinguishes managed gateway throttling from strict application-level identity quotas.

## Step 1 — Requirements

### Functional Requirements (4)
1. Enforce configurable limits **per authenticated user, tenant and API route**.
2. Reject requests exceeding their allowance with `HTTP 429 Too Many Requests`.
3. Permit controlled bursts according to the selected algorithm.
4. Allow authorized administrators to create and change rate-limit policies.

### Non-Functional Requirements (4)
1. **Low latency:** Rate-limit decision p99 below 10 ms in the normal case (illustrative target).
2. **Consistency:** All 20+ Pods share the same rate-limit state; an individual Pod cannot give extra allowance.
3. **Availability:** Run across three AZs with an explicitly chosen Redis-failure behavior.
4. **Scale and protection:** Handle an illustrative 10,000 incoming RPS while protecting downstream databases and services.

**Precise policy for this exercise:** Alice can make **at most 100 requests in any rolling 60-second window**. A sliding-window algorithm is appropriate. A token bucket controls a replenishment rate plus bursts; it **does not inherently guarantee** that exact rolling-window ceiling.

---

## Step 2 — API Contracts (Short)

**Protected application APIs remain unchanged:** `GET /orders`, `POST /orders`, etc. A rate-limit decision happens **before** business logic.

| Action | Endpoint / call |
|---|---|
| Internal decision | `POST /internal/rate-limit/check` (or gRPC `RateLimit.Check`) |
| Read policy | `GET /v1/rate-limit-policies/{policy_id}` |
| Create/update policy | `PUT /v1/rate-limit-policies/{policy_id}` |

**Internal check — illustrative**
~~~json
{"user_id":"alice","tenant_id":"T1","route":"GET:/orders","policy_id":"P1"}
~~~
~~~json
{"allowed":false,"remaining":0,"retry_after_seconds":18}
~~~

When rejected, the gateway/middleware responds:
~~~http
HTTP/1.1 429 Too Many Requests
Retry-After: 18
~~~

**Security:** User ID must come from a **verified JWT claim**, such as `sub`, combined with tenant/route; never trust a client-supplied user ID. Only internal trusted components call the decision API. Policy writes require admin authorization.

---

## Step 3 — Core Entities

| Storage | Entity / fields | Purpose |
|---|---|---|
| **PostgreSQL** | `RateLimitPolicies(policy_id PK, tenant_id, route_pattern, algorithm, limit_count, window_seconds, enabled, version)` | Durable configuration |
| **PostgreSQL** | `PolicyOverrides(tenant_id, user_id, route_pattern, policy_id)` | Tenant/user exceptions |
| **PostgreSQL** | `PolicyAudit(audit_id PK, actor, policy_id, change, changed_at)` | Administrative change history |
| **Redis** | `rate:{tenant}:{user}:{route}` → sorted set of request timestamps/unique IDs with expiry | Ephemeral live sliding-window state |

**Indexes:** Unique policy key per applicable tenant/route, primary key on policy ID, lookup index on overrides by tenant/user/route, and audit index on `(policy_id, changed_at DESC)`.

**Retention:** Redis request entries expire after the window has passed. Retain aggregated metrics rather than every request in PostgreSQL; keep policy-audit logs according to governance requirements.

---

## Step 4 — Database Model: Why?

**Redis / ElastiCache is the real-time decision store.** Atomic operations provide consistent admission decisions across multiple Pods with low latency.

**PostgreSQL is the durable policy store.** Relational policies, tenant overrides, audit history and configuration changes do not need a write for every API request.

**Why not local memory in each Pod?** With 60 requests through Pod A and 60 through Pod B, two local limits of 100 would wrongly allow all 120.

**Why not write all request counts to PostgreSQL?** It adds a high-frequency write and transaction bottleneck to a database better used for configuration.

**Why not just API Gateway?** AWS API Gateway offers managed best-effort throttling (useful for coarse limits), but exact custom per-authenticated-user limits, consistent across many services and keyed by verified identity, may require an external limiter.

**Redis HA caveat:** ElastiCache Multi-AZ failover restores availability, but asynchronously replicated Redis state can lose recent counter updates. **An absolutely strict quota even during failover** requires a more strongly consistent coordination design, or the business must accept temporary over-admission/fail-closed behavior.

---

## Step 5 — High-Level Design

~~~text
Client → Route 53 (DNS) → API Gateway (JWT + coarse throttle)
       → Private VPC Link / NLB → EKS Ingress
       → Rate Limit Middleware / Shared Limiter Service
       → Redis atomic rolling-window check
             ALLOW → Order/API Pod → PostgreSQL / other backend
             DENY  → HTTP 429 + Retry-After

Admin → Policy API → PostgreSQL policy store
Policy service → cache/propagate policy to limiter workers
~~~

The rate-limiter decision happens before expensive application/database work.

---

## Step 6 — Deep Dive: Exact Request Flow and Algorithm

### A. Alice sends 60 requests via Pod A and 60 via Pod B

1. API Gateway validates JWT. Trusted gateway/middleware extracts `tenant_id`, `sub=alice`, normalized route and effective policy.
2. Pod A and Pod B call the **same shared rate-limiter state in Redis**.
3. Atomic Redis script removes timestamps older than the previous 60 seconds, counts the remaining requests and checks the limit.
4. If count < 100, add this request's unique ID and timestamp, refresh key expiry, and **allow**.
5. If count >= 100, do not add an entry; return **deny**, then response `429` with a calculated `Retry-After`.
6. For 120 requests within one rolling minute and no other Alice requests, **100 allowed, 20 rejected** (ordering of accepted requests may vary).

### B. Why Redis Lua / Redis Functions rather than separate commands?

**Not safe as three independent calls:**
~~~text
GET count → check limit → INCR count
~~~
Two application Pods may both see 99 and both allow request 100.

**Safer:** Use an atomic server-side operation so cleanup → count → allow/deny → record occurs as a single serialized decision per key.

**Simplified sliding-window pseudocode**:
~~~text
key = "rate:T1:alice:GET-orders"
now = trusted_time_milliseconds()
atomic {
    remove entries older than now - 60000
    count = current number of entries
    if count >= 100:
        return DENY with next retry time
    add unique_request_id with score now
    set key expiration to cover rolling window
    return ALLOW with remaining = 99 - count
}
~~~

For production, prefer server-coordinated time to reduce clock skew, ensure request IDs are unique, and measure sorted-set memory / CPU cost for busy tenants.

### C. Token bucket vs fixed window vs sliding window

| Algorithm | Benefit | Trade-off |
|---|---|---|
| Fixed-window counter + TTL | Simple, inexpensive | Burst across window boundary can exceed rolling maximum |
| Token bucket | Smooth average rate; controlled burst | Does not guarantee exact 100 in **every** rolling minute |
| Sliding-window log (sorted set) | Precise rolling-window request count | More Redis memory/work per request |

For our explicitly **strict 100 in any 60 seconds**, start with the sliding-window log. For ordinary API fairness, token bucket is often an excellent simpler choice.

### D. What if Redis fails?

Choose and document behavior **per API criticality**:

- **Fail-open:** Allow requests temporarily, preserve availability, but risk overload and quota violations.
- **Fail-closed:** Reject uncertain requests, preserve strict security/cost control, but hurts availability.
- **Controlled fallback:** Use conservative per-Pod local limits with reduced capacity plus gateway coarse throttling, and enforce bounded backend concurrency. This is **not a precise global per-user limit**.

The choice must be explicit: for ordinary reads a bounded fail-open may be acceptable; for expensive, abuse-sensitive writes, fail-closed may be warranted.

### E. Other scaling and security concerns

- Separate coarse gateway throttling from identity-aware per-user quotas.
- Avoid untrusted spoofable IP headers and key cardinality explosions.
- A single celebrity/API-key counter can become a **hot Redis key**; shard by quota ownership or use quota leasing with a documented over-admission trade-off, instead of naïvely splitting counters that break correctness.
- Refresh policy caches with versioning; decide whether policy changes are immediately consistent or can have short propagation lag.
- Cap the limiter's own latency; apply a timeout and fallback policy so it doesn't become the application bottleneck.

---

## Step 7 — Why Each Component?

| Component | Why we use it |
|---|---|
| Route 53 | DNS hostname resolution |
| API Gateway | JWT validation and coarse managed throttling |
| VPC Link + NLB + Ingress | Private EKS traffic route |
| EKS Rate Limiter Service / middleware | Apply identity/route policy before backend work |
| Redis / ElastiCache Multi-AZ | Shared, fast, atomic rolling-window state |
| PostgreSQL Multi-AZ | Durable rate-limit policies, overrides and audit |
| Admin Policy API | Safely update user/tenant limits |
| Protected API Pods | Serve requests that passed admission control |
| HPA/Karpenter | Scale decision and application Pods / worker nodes |
| CloudWatch/Grafana/Tracing | Observe decisions, latency, 429s, backend saturation |
| IAM/OIDC + TLS | Secure policy administration and trust identity claims |

---

## Step 8 — Failure, Remediation and Verification

| Failure | Impact | Remediation | Recovery signal |
|---|---|---|---|
| Two Pods use separate counters | User exceeds global limit | Shared Redis atomic check per identity/route | 120 requests → only 100 admitted |
| Atomicity bug | Over-admission | Lua/Redis Function rather than GET+INCR race | Concurrency tests pass |
| Redis primary fails | Limiter unavailable / potential lost counters | Multi-AZ failover + documented fallback / fail-closed choice | Decision availability returns |
| Redis CPU/hot key | High decision latency | Find hot keys; isolate workloads or change algorithm/quota ownership | p99 decision latency declines |
| Policy DB down | Admin updates fail | Serve existing cached policy with version and expiry; recover DB | Policy updates propagate again |
| Misconfigured strict limits | Legitimate user 429 spikes | Roll back bad policy, audit who changed it | Legitimate pass rate recovers |
| Gateway/limiter unavailable | Requests blocked | Healthy replicas, circuit breakers, defined fallback | API success/error budget recovers |
| Backend overload despite limit | 5xx from dependencies | Global concurrency cap and load shedding in addition to user quotas | DB CPU/5xx normalize |
| AZ failure | Lost limiter Pods | Multi-AZ placement + spare capacity; Redis automatic failover | Admission and API both healthy |

**Monitor:** Allow/deny decisions, 429 rate by tenant and route, limiter p50/p95/p99, Redis latency/evictions/CPU, policy cache freshness, downstream 5xx, and error-budget burn.

---

## Interview Summary

> I would authenticate clients at API Gateway, then use an EKS middleware or distributed Rate Limiter Service to enforce a shared per-user/tenant quota before forwarding requests. Redis performs atomic checks; PostgreSQL stores policies and audit records. For a strict 100 requests in any rolling minute, I would use an atomic sliding-window algorithm. Exceeding the quota returns HTTP 429. I would explicitly define fail-open/fail-closed behavior for Redis outages and monitor decision latency and rejection rates.

**Main change from a normal CRUD API:** identity-aware admission control and **shared atomic Redis state** before the business application.

---

## Detailed End-to-End Diagram

~~~mermaid
flowchart TD
  CL["Alice client — 120 requests / minute"]
  DNS["Route 53 — DNS"]
  GW["API Gateway — JWT + coarse throttle"]
  LINK["Private VPC Link + NLB"]
  ING["EKS Ingress"]

  subgraph EKS["EKS — 3 AZs"]
    PODA["Pod A middleware — 60 requests"]
    PODB["Pod B middleware — 60 requests"]
    RL["Distributed Rate Limiter"]
    APP["Protected API Pods"]
    ADMIN["Admin Policy API"]
  end

  subgraph DATA["State and Policy"]
    RED["ElastiCache Redis Multi-AZ"]
    KEY["Sliding-window set — tenant:user:route"]
    PG["PostgreSQL Multi-AZ"]
    POLICY["RateLimitPolicies + Overrides"]
    AUD["PolicyAudit"]
  end

  CHECK{"Allowed under 100 / rolling 60 seconds?"}
  ALLOW["Continue request"]
  DENY["HTTP 429 + Retry-After"]
  BIZ["Application DB / dependencies"]
  FALL{"Redis unavailable?"}
  FALLBACK["Bounded local fallback or fail-closed"]
  MON["CloudWatch / Grafana / Traces"]

  CL -. "Resolve DNS" .-> DNS
  CL --> GW
  GW --> LINK --> ING
  ING --> PODA
  ING --> PODB
  PODA -->|"Trusted user ID and route"| RL
  PODB -->|"Trusted user ID and route"| RL

  RL -->|"Atomic cleanup/count/record"| RED
  RED --> KEY
  RED --> CHECK
  CHECK -->|"Yes — first 100"| ALLOW
  CHECK -->|"No — next 20"| DENY
  ALLOW --> APP --> BIZ
  DENY --> CL

  RED -. "Timeout or failover" .-> FALL
  FALL -->|"Yes"| FALLBACK
  FALLBACK -->|"Controlled policy"| APP

  ADMIN -->|"Write policy + audit"| PG
  PG --> POLICY
  PG --> AUD
  POLICY -. "Cached policy/version"| RL

  RL -. "Decision latency / allow / deny"| MON
  RED -. "CPU / failover / hot key"| MON
  APP -. "5xx and saturation"| MON
  GW -. "Auth and throttle metrics"| MON
~~~

**Diagram reading:** Alice's requests may land on any Pod. All decision paths use the **same Redis quota key**. For 120 requests inside a strict rolling-minute window, the atomic check admits 100 and returns 429 for 20. The fallback is a separate deliberate policy for Redis outages, not a guarantee of exact global enforcement.
