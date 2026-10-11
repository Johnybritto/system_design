# Apple ASE SRE — Order API Quick Revision (R23-D03 / D04 / D05)

> **Purpose:** One-page-style interview revision for the **same Order API** across high availability, overload, and regional disaster recovery. Read each **diagram → concept table → failure table → memory line**. These are **design choices and example numbers**, not claims about Apple's systems.
>
> **Baseline:** Client → DNS → AWS WAF/API Gateway → internal ALB → EKS Order API (3 AZs) → Redis (optional cache) / RDS PostgreSQL (source of truth). Example forecast: **3,762 peak RPS**, **24 pods = 8 per AZ**, and **300 planned RPS/pod**. All capacity depends on load testing.

---

## R23-D03 — Highly Available Customer-Facing API

**Goal:** A pod, node, AZ, cache, or DB instance failure should not bring down the whole service.

### Diagram 1 — Components and failure logic

~~~mermaid
flowchart TD
    A["Client / Browser<br/>Timeout: bounded retry; POST: same idempotency key"]
    B["Route 53 DNS<br/>Lookup failure: inspect DNS"]
    C["AWS WAF + API Gateway<br/>401/403: auth/rules; 429: throttling"]
    D["VPC Link V2 + Internal ALB<br/>Unhealthy targets: health checks / routing"]
    E["EKS Order API in 3 AZs<br/>Pod/node failure: replicas; Pending: node scaling"]
    R["Redis (HA cache)<br/>Miss/failure: bounded PostgreSQL fallback"]
    P["RDS PostgreSQL primary<br/>Slow: inspect queries/pools; down: HA failover"]
    S["RDS synchronous standby<br/>Promoted if primary fails"]
    A --> B --> C --> D --> E
    E -->|"cacheable GET"| R
    E -->|"POST / consistent GET"| P
    R -->|"miss (if capacity permits)"| P
    P -.->|"synchronous HA replication"| S
~~~

*AWS detail:* With ALB **IP targets**, the ALB routes directly to pod IPs; Kubernetes Ingress/Service configuration is not an extra proxy hop. A standard RDS Multi-AZ **standby is not a read replica**.

### Concepts — Simple explanation

| Concept | Simple explanation |
|---|---|
| High availability | Multiple healthy instances across failure domains |
| HPA vs node autoscaler | HPA requests pods; autoscaler provides nodes when capacity is missing |
| Redis cache | Speeds up eligible reads; PostgreSQL remains authoritative |
| PostgreSQL Multi-AZ | Primary + synchronous standby for automatic instance/AZ failover |
| Idempotency | Same POST key must not create a second order after a timeout |
| SLO | Measure successful customer requests and p95/p99 latency, not just Running pods |

### Failures — What we do

| Failed component | What we do |
|---|---|
| DNS / gateway / WAF | Check DNS, TLS, auth, 403/429/5xx, rate limits and policy logs |
| ALB | Inspect healthy targets, ports, health-check path and security groups |
| EKS pod / node | Route to Ready targets; recreate pods; check node headroom |
| One AZ | Use surviving AZs; verify application **and DB/cache** capacity |
| Redis | Short timeout + circuit breaker; **bounded** DB fallback |
| PostgreSQL primary | Managed standby promotion; reconnect; verify committed orders |
| Bad deployment | Stop rollout / pipeline rollback; validate user-journey metrics |

**Remember:** **DNS → Gateway → ALB → EKS → Redis/DB → HA failover**.

---

## R23-D04 — API Overload and Dependency Failure

**Goal:** A 10× traffic spike or slow dependency must not trigger a cascading outage.

### Diagram 2 — Components and protection logic

~~~mermaid
flowchart TD
    A["Client<br/>Bounded retries + backoff + jitter"]
    B["API Gateway<br/>10x traffic: quotas / throttling (429)"]
    C["ALB<br/>Target errors: health / latency / 502-504"]
    D["EKS Order API<br/>Limit in-flight work; shed excess (503)"]
    H["HPA + Node Autoscaler<br/>Scale only when dependencies have headroom"]
    R["Redis<br/>Slow: short timeout; down: bounded fallback"]
    CP["DB connection pool / RDS Proxy<br/>Pool full: bounded waits + concurrency limits"]
    P["PostgreSQL<br/>Slow: inspect locks, query plans, CPU, I/O"]
    X["External service<br/>Timeout + circuit breaker + bulkhead"]
    A --> B --> C --> D
    D -.->|"capacity metrics"| H
    D -->|"cacheable GET"| R
    D -->|"writes / consistent reads"| CP
    R -->|"miss; admit safely"| CP
    CP --> P
    D -->|"optional API call"| X
~~~

**Example:** 10× forecast peak = **37,620 RPS**. At 300 RPS/pod, 24 pods can plan for only **7,200 RPS**. HPA cannot instantly create enough *safe end-to-end* capacity.

### Concepts — Simple explanation

| Concept | Simple explanation |
|---|---|
| Rate limit | Control requests per client/API; excess typically **429** |
| Back-pressure | Limit admitted work when dependencies cannot keep up |
| Load shedding | Reject excess work quickly, often **503**, to preserve critical calls |
| Timeout | Bound how long a request waits |
| Backoff + jitter | Delay and randomize safe retry attempts; avoid retry storms |
| Retry budget | Cap extra attempts; retry at one deliberate layer where possible |
| Circuit breaker | Stop repeatedly calling a failing dependency; reopen cautiously |
| Bulkhead | Isolate concurrency so one slow dependency doesn't exhaust all workers |

### Failures — What we do

| Failed component / symptom | What we do |
|---|---|
| Gateway: 10× traffic | Throttle/rate limit; protect critical operations |
| ALB: 502/503/504 | Check health, targets, connections, backend latency |
| EKS: high CPU / many requests | HPA if useful; cap in-flight work, shed load |
| EKS: new pods Pending | Check scheduling, requests, quotas; provision nodes |
| Redis: slow / down | Short timeout, breaker, **controlled** DB fallback |
| DB pool: exhausted | Check pool waits; cap concurrency; **don't blindly grow pools** |
| PostgreSQL: high CPU/slow queries | Find locks/slow SQL/I-O; reduce expensive workload |
| External API: slow | Deadline, circuit breaker, bulkhead, safe fallback only |
| Retry storm | Reduce retries; deadline + jitter + budget + idempotency |

**Remember:** **Rate limit → Back-pressure → Timeouts → Breaker → Shed load → Verify recovery**.

---

## R23-D05 — Multi-Region Recovery

**Goal:** Restore the same Order API after the **entire primary region** fails.

**Design choice:** **Active-passive / warm standby**. Mumbai serves traffic; Hyderabad is provisioned for recovery. PostgreSQL cross-region replication is **asynchronous** (unlike synchronous standby replication *within* Mumbai).

### Diagram 3 — Regional failover and failback

~~~mermaid
flowchart TD
    U["Customer"]
    DNS["Route 53 failover DNS<br/>Cut traffic over only when DR is ready"]
    subgraph M["Mumbai — ACTIVE"]
       MA["WAF / API Gateway / ALB / EKS<br/>Local pod/AZ loss: HA"]
       MP["PostgreSQL writer + Multi-AZ standby<br/>Whole region down: DR activation"]
       MR["Redis cache<br/>Rebuildable, not source of truth"]
       MA --> MP
       MA --> MR
    end
    subgraph H["Hyderabad — WARM STANDBY"]
       HA["WAF / API Gateway / ALB / EKS<br/>Scale and validate for full traffic"]
       HP["PostgreSQL cross-region replica<br/>Promote to writer after DR decision"]
       HR["Redis cache<br/>Warm or rebuild gradually"]
       HA --> HP
       HA --> HR
    end
    U --> DNS
    DNS -->|"normal"| MA
    MP -.->|"async DB replication; possible lag"| HP
    DNS -.->|"after readiness / safe cutover"| HA
    HP -.-> FB["Failback: reconcile data → reverse sync → verify → shift back"]
~~~

### Concepts — Simple explanation

| Concept | Simple explanation |
|---|---|
| Active-passive | One serving region, one recovery region; simpler writes |
| Active-active | Both serve traffic; harder write consistency/conflict control |
| Async replication | Copies changes to DR; may lag and lose recent acknowledged writes |
| RTO | Time allowed to restore service; **e.g. 15 min target** |
| RPO | Acceptable data loss window; **e.g. 1 min target** |
| Recovery capacity | DR must sustain forecast **3,762 RPS**, including dependencies |
| Failback | Safely return to Mumbai after data reconciliation and validation |

*RTO/RPO values are hypothetical business targets, **not AWS guarantees**; prove them with DR exercises.*

### Failures — What we do

| Failed component | What we do |
|---|---|
| Mumbai pod / one AZ | Stay in-region; use healthy instances and HA DB |
| Entire Mumbai region | Declare DR; prevent competing writers / split-brain |
| Primary regional PostgreSQL | Assess replication lag; promote Hyderabad replica; reconcile uncertain POSTs |
| Hyderabad pods/nodes inadequate | Scale and verify EKS, quotas, Redis and DB capacity |
| DR cache empty | Gradually warm Redis; throttle DB fallback |
| Regional routing | Shift Route 53 **after** DR write path and app are ready; consider DNS caching |
| Mumbai recovers | Reconcile → establish reverse replication → validate → controlled failback |

**Remember:** **Detect → Prevent dual writers → Promote DB → Scale → Verify → Switch DNS → Failback**.

---

## Final 30-second revision

| Design | Core question | One-line answer |
|---|---|---|
| **D03 — HA** | What if a component or AZ dies? | Use replicas, health-aware routing, HA dependencies and enough survivor capacity. |
| **D04 — Overload** | What if traffic or dependency latency explodes? | Restrict incoming and concurrent work, bound retries, shed excess, and protect the DB. |
| **D05 — DR** | What if the whole region dies? | Promote the DR DB safely, validate capacity/correctness, then switch routing and fail back later. |

**Universal SRE sequence:** **Customer impact → Evidence → Safe mitigation → Recovery checks → Prevention**.

**AWS ↔ Azure mental mapping:** Route 53 ↔ Azure DNS/Traffic Manager; API Gateway ↔ API Management; ALB ↔ Application Gateway; EKS ↔ AKS; ElastiCache ↔ Azure Managed Redis; RDS PostgreSQL ↔ Azure Database for PostgreSQL. These are role equivalents, **not identical network flows**.
