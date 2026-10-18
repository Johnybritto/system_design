# Caching in System Design — Strategies, Redis HA, Failures and Interview Q&A

> **System Design Notes | Interview-ready consolidated guide**  
> Focus: caching fundamentals, cache-aside vs read-through, write strategies, failure mitigation, Redis high availability, consistency, performance, durable writes, monitoring, and real-world troubleshooting.  
> **Mental model:** Caching improves read performance; HA reduces downtime; durability protects accepted data. These solve different problems.

---

## 1. What is caching?

Caching stores frequently used data in a faster layer to avoid repeated, expensive requests to the original source (often a relational or NoSQL database).

**Example:** An e-commerce API receives 10,000 product reads/second. If cache hit rate is 90%, approximately 9,000 reads/second are served from Redis and 1,000 reach the database (simplified: assumes one cache lookup per read, one DB read per miss, and no additional background reads).

### Diagram 1 — Cache-aside API with Redis

```mermaid
flowchart TD
  A["Client / Mobile App"]
  B["API Gateway / Load Balancer"]
  C["Application Pods - EKS / AKS"]
  D[("Redis Cache")]
  E[("Database - RDS / Azure SQL")]
  A --> B --> C
  C -->|"1. GET cached key"| D
  D -->|"Cache HIT: value"| C
  D -.->|"Cache MISS"| C
  C -->|"2. On miss: query"| E
  E -->|"3. Value"| C
  C -->|"4. SET key + TTL"| D
  C -->|"5. Response"| A
```

**Flow:** Client calls the API -> app checks Redis -> if hit, returns from cache -> if miss, app queries DB -> saves result with a TTL -> responds.

**Typical cache layers:**

| Layer | Examples | What it helps |
|---|---|---|
| Browser | HTTP/browser cache | Repeat image/CSS/JS requests |
| CDN / edge | CloudFront, Azure Front Door | Reduce origin requests and global latency |
| Local process | In-memory LRU | Sub-millisecond access per pod (not shared across pods) |
| Distributed cache | Redis / Memcached | Reusable data across application pods |
| DB internal | Buffer pool, plan cache | Reduce DB disk I/O / planning overhead |

---

## 2. Caching strategies

### Q1. Are cache-aside and read-through the same?

**They look similar because both load from the DB on a miss. The key difference is WHO fetches the missing value.**

- **Cache-aside (lazy loading):** the *application* detects a cache miss, queries DB and sets Redis.
- **Read-through:** a *cache abstraction / integrated loader* detects a miss, fetches DB data and populates cache without exposing the miss-handling logic to the caller.

**Important:** Plain Redis alone does not magically know which SQL database or query to use. Read-through requires a loader, middleware, or application cache abstraction.

### Diagram 2A — Cache-aside (application owns cache miss)

```mermaid
flowchart TD
  A["Application"]
  R[("Redis")]
  D[("Database")]
  A -->|"GET key"| R
  R -.->|"MISS"| A
  A -->|"DB SELECT"| D
  D -->|"Row"| A
  A -->|"SET key + TTL"| R
```

### Diagram 2B — Read-through (cache layer owns miss)

```mermaid
flowchart TD
  A["Application"]
  C["Cache abstraction / loader"]
  D[("Database")]
  A -->|"GET key"| C
  C -->|"On MISS: fetch"| D
  D -->|"Row"| C
  C -->|"Value to caller"| A
```

### Q2. What about the write strategies?

| Strategy | Actual write path | Main trade-off |
|---|---|---|
| **Write-through** | App -> cache layer -> *synchronously* persist DB -> acknowledge | Fresh cached reads after successful coordinated write, but slower writes |
| **Write-behind / write-back** | App -> cache/buffer -> acknowledge -> asynchronously persist DB | Low write latency, but accepted changes may be lost without durability |
| **Write-around** | App -> DB directly; do not populate cache on write | Avoids polluting cache with rarely read data, but later reads miss |

### Diagram 3 — Three write paths

**Write-through:**

```mermaid
flowchart LR
  A["Application"] -->|"Write"| B["Cache layer"]
  B -->|"Synchronous persist"| C[("Database")]
  C -.->|"Committed"| B
  B -.->|"Acknowledge"| A
```

**Write-behind:**

```mermaid
flowchart LR
  A["Application"] -->|"Write"| B["Cache / Buffer"]
  B -.->|"Early ack"| A
  B -->|"Flush asynchronously"| C[("Database")]
```

**Write-around:**

```mermaid
flowchart LR
  A["Application"] -->|"Write"| B[("Database")]
  A -.->|"Invalidate old key after commit"| C[("Cache")]
```

---

## 3. Which strategy should an architect select?

| Strategy | Advantages | Disadvantages | **How to mitigate the disadvantage** | Best use |
|---|---|---|---|---|
| **Cache-aside** | Simple; caches requested data | Cold misses, stale cached entries, stampede | **TTL + invalidate after DB commit + cache warming + single-flight loading** | Product catalog, profile GET |
| **Read-through** | Centralized miss logic | Loader dependency, miss latency, possible DB overload | **Replicated cache + bounded timeouts + controlled DB fallback + preload hot keys** | Shared read abstractions |
| **Write-through** | Cache updated in same write workflow; read-after-write easier | Higher write latency, partial DB/cache failures | **Durable DB commit; retry/invalidate failed cache update; idempotency and versioning** | Frequently updated/read values |
| **Write-behind** | Fast writes; batching | Crash before flush can lose acknowledged changes | **Durable queue/WAL + replication + retries + DLQ + recovery/replay** (when queue-backed) | Delay-tolerant counters, telemetry |
| **Write-around** | Avoids filling cache with rarely read writes | Next read misses; stale old entry possible | **Invalidate on DB commit, TTL; optionally preload hot keys** | Write-heavy, infrequently read data |

**Default for a customer-facing EKS/AKS API backed by RDS/Azure SQL:** **Cache-aside + managed Redis HA + TTL + invalidation + bounded timeouts + circuit breaker + rate-limited DB fallback.** For payment and inventory correctness, do not trust an arbitrary cached value as transactional authority.

---

## 4. Interview Q&A — Cache consistency and performance

### Q3. What is invalidation on DB writes?

When a record changes in the DB, remove or update the corresponding cached key so the next read does not keep serving stale data.

**Example:**

- Redis has `product:1001 = ₹50,000`.
- DB has `product:1001 = ₹50,000`.
- Admin updates price to ₹48,000.
- After DB change commits, delete `product:1001` from Redis.
- Next GET misses -> reads ₹48,000 from DB -> repopulates Redis.

**Failure risk:** Cache invalidation can fail even after the DB succeeds. TTL limits how long old data can remain under ordinary expiry semantics; reliable event delivery / retries may also be needed.

### Q4. What does "invalidate on DB commit" add?

It specifies *when* to invalidate: after the DB transaction **successfully commits**, rather than when an update was merely attempted.

### Diagram 4 — Commit then invalidate

```mermaid
flowchart TD
  A["Client updates product"]
  B["Application updates database"]
  C{"Transaction committed?"}
  D["Delete old Redis key"]
  E["Next GET: cache MISS"]
  F["Read fresh DB value"]
  G["Populate Redis"]
  H["Rollback / return error"]
  A --> B --> C
  C -->|"Yes"| D --> E --> F --> G
  C -->|"No"| H
```

**Why not invalidate on a failed transaction?** The DB may have rolled back, creating a needless miss.

**Architect's nuance:** "Commit then delete key" by itself is not atomic across DB and Redis. Invalidation can be lost; a concurrent read can repopulate an old value in a race. For important consistency use versioned values/keys, CDC or a transactional outbox, idempotent invalidation handlers, and well-defined TTL/consistency expectations.

### Q5. What is cache warming?

Intentionally populate useful cache entries **before** normal user requests (for example, after Redis restart or before a flash sale), instead of making the first users cause all the DB reads.

**Example:** A restarted Redis has an empty cache. Load popular catalog entries from the DB, then gradually ramp up production traffic. This reduces cold-start DB pressure.

### Q6. What is single-flight loading?

When many concurrent requests miss **the same key**, coordinate them so only one performs the DB fetch and the others share the result.

### Diagram 5 — Cache stampede prevention

```mermaid
flowchart LR
  A["1,000 requests for one missing key"]
  B["Per-key single-flight coordinator"]
  C["1 database read"]
  D["Populate cache"]
  E["Share result"]
  A --> B --> C --> D --> E
```

**Without it:** 1,000 users might trigger 1,000 reads. **With it:** ideally one read *per coordination scope/key*.

**Caveat:** A mutex in one pod coalesces only that pod's requests. In multi-pod EKS/AKS, use cross-pod coordination or accept some duplicate loads. Other mitigations: TTL jitter, stale-while-revalidate, and refresh-ahead.

---

## 5. Interview Q&A — HA and graceful degradation

### Q7. What is a replicated cache?

A primary Redis node replicates changes to one or more replicas, often placed across different AZs. If the primary fails, one suitable replica can be promoted, **provided automatic failover is configured**.

**Benefit:** reduces the single-node failure risk.

**Limitation:** Redis replication is usually asynchronous; recently acknowledged cache writes might not exist on the promoted replica. In a conventional cache-aside design, this is normally acceptable because the DB is authoritative and missing data can be rebuilt.

### Q8. What are bounded timeouts?

Put an upper limit on each Redis connection/read attempt (and on all retries together) so an unhealthy cache cannot stall the entire API request.

**Illustrative, not recommended universal defaults:**

```text
Redis connect timeout = 100 ms
Redis read timeout    = 50 ms
Retries               = limited by total request deadline
If unavailable        = circuit-break and controlled fallback
```

Choose actual values based on network latency, SLO and overall request deadline. Long timeouts can exhaust app worker threads/connections.

### Q9. What is fallback to DB?

When Redis is down, the application queries the database directly **within a controlled fallback budget**. Useful for availability, dangerous if unconstrained.

Example: 10,000 RPS, 90% hit rate -> normally about 1,000 DB reads/sec. If Redis completely fails and *all* requests fall back, DB load may surge to 10,000 reads/sec and cause cascading failure.

**Mitigation:** short timeouts, circuit breaker, fallback concurrency cap, token bucket/rate limit, DB connection-pool limits, serve acceptable stale data when allowed, and 429/503 load shedding if DB cannot keep up.

### Q10. What are "preload hot keys"?

Proactively cache the most frequently requested keys before peak load.

**Example:** Out of one million products, 500 generate most traffic. Load `product:1001` through `product:1500` from the DB ahead of the sale.

**Difference:** Cache warming is the broad practice; preloading hot keys is a targeted warming technique.

---

## 6. Redis HA on EKS / AKS — Architecture and failover

For AWS, a common production approach is **Amazon ElastiCache for Redis/Valkey** with Multi-AZ replicas and automatic failover. In Azure, choose an HA-managed Redis option with appropriate zone redundancy.

### Diagram 6 — Managed Redis across three AZs

```mermaid
flowchart TD
  U["Customer"]
  LB["Load Balancer"]
  APP["EKS Application Pods"]
  EP["Redis Primary Endpoint"]
  DB[("RDS / Authoritative DB")]
  U --> LB --> APP
  APP -->|"Cache operations"| EP
  APP -.->|"Controlled DB fallback"| DB
  subgraph AZ1["Availability Zone A"]
    P[("Redis Primary")]
  end
  subgraph AZ2["Availability Zone B"]
    R1[("Redis Replica 1")]
  end
  subgraph AZ3["Availability Zone C"]
    R2[("Redis Replica 2")]
  end
  EP --> P
  P -.->|"Async replication"| R1
  P -.->|"Async replication"| R2
```

**Deployment checklist:**

1. Provision a Redis replication group / managed HA deployment.
2. Place primary and replicas in separate AZs; enable automatic failover.
3. Use supported primary/reader endpoints or failover-aware clients, **not a fixed node IP**.
4. Set sensible connection/read timeouts and bounded retry/reconnect policies.
5. Configure monitoring: availability, replication lag, memory, evictions, connections, errors and cache hit ratio.
6. Plan cold-cache warming and protect RDS during failover.
7. If using Redis Cluster for sharding, also plan replicas and failover **per shard**. Sharding alone is not HA.

### Diagram 7 — Redis primary fails

```mermaid
flowchart TD
  A["Primary in AZ-A fails"]
  B["ElastiCache detects failure"]
  C["Healthy eligible replica selected"]
  D["Replica promoted in another AZ"]
  E["Primary endpoint follows new primary"]
  F["Redis clients reconnect"]
  G["Reads/writes resume"]
  H["Replacement replica / resync"]
  A --> B --> C --> D --> E --> F --> G
  D --> H
```

**Failure impact:** Some cache operations and client connections fail during detection/promotion. Failover is not zero downtime or automatically zero data loss. Applications require reconnect/retry behavior.

### Diagram 8 — Wider Redis outage and cascading DB failure

```mermaid
flowchart TD
  A["10,000 incoming requests/sec"]
  B["Application pods"]
  C[("Redis unavailable")]
  D{"Fallback permitted?"}
  E[("RDS / Azure SQL")]
  F["Controlled response"]
  G["429 / 503 / acceptable stale response"]
  A --> B --> C
  C -->|"Timeout quickly"| D
  D -->|"Within DB capacity budget"| E --> F
  D -->|"DB capacity exhausted"| G
```

| Redis failure | Expected behavior | SRE/architect response |
|---|---|---|
| Replica fails | Primary still serves | Replace and resync replica; confirm redundancy |
| Primary fails | Replica promoted if available | Managed failover, reconnect clients, verify service |
| One AZ lost | Remaining nodes may serve after failover | Multi-AZ distribution and spare capacity |
| All Redis nodes unreachable | Cache traffic fails | Circuit breaker, capped DB fallback, graceful degradation |
| Redis memory pressure | Evictions or rejected writes, depending on policy | Correct maxmemory/eviction policy, TTL, scaling |
| Network issue | Timeouts/connection errors | Check DNS, SG/NSG, routes, network policies, endpoints |
| Entire region lost | Regional cache unavailable | Fail over app and DB by DR plan; rebuild/warm cache |

### Q11. What about Redis Sentinel?

For **self-managed** non-clustered Redis on VMs/Kubernetes, Sentinel can monitor nodes, discover the current primary and coordinate failover. A common robust design uses at least **three independently placed Sentinel instances** with quorum/majority configuration.

```mermaid
flowchart TD
  A["Redis-aware application client"]
  S["3 Sentinel instances"]
  P[("Redis Primary")]
  R1[("Replica - AZ B")]
  R2[("Replica - AZ C")]
  A -->|"Discover current primary"| S
  A -->|"Reads / writes"| P
  S -.->|"Monitor / authorize failover"| P
  S -.->|"Monitor"| R1
  S -.->|"Monitor"| R2
  P -->|"Replication"| R1
  P -->|"Replication"| R2
```

**Important:** Redis Sentinel and Redis Cluster are different: Sentinel is a failover/discovery system typically for non-sharded Redis; Redis Cluster shards data and also supports replication/failover per shard. Managed AWS/Azure cache options generally reduce the need to run Sentinel yourself.

---

## 7. Interview Q&A — Durability for write-behind

### Q12. What is a durable queue / WAL?

A **durable queue** stores accepted work persistently for later workers. A **write-ahead log (WAL)** records pending changes before applying them to the final storage state, enabling crash recovery in systems designed around it.

**Why:** If you acknowledge a cache write but lose the only cached copy before a DB flush, you may lose accepted data. A durable handoff **before acknowledgement** reduces this risk.

### Diagram 9 — Durable asynchronous write pipeline

```mermaid
flowchart TD
  A["Application"]
  Q[("Durable Queue / Log")]
  W["Background Worker"]
  D[("Database")]
  L[("Dead Letter Queue")]
  A -->|"1. Persist command/event"| Q
  Q -.->|"2. Durable acceptance acknowledgement"| A
  Q -->|"3. Consume"| W
  W -->|"4. Persist / commit"| D
  W -.->|"Repeated processing failures"| L
```

**Important distinction:** This is a *durable asynchronous write pipeline*. Classic write-behind caching acknowledges a cache write before DB persistence. Redis alone is **not automatically equivalent** to a durable queue/WAL with end-to-end guaranteed acceptance; Redis AOF/replication/ack policy and failure model matter.

### Q13. What does replication mean for write-behind?

Keep copies of pending changes/log entries on multiple nodes so a single node crash does not necessarily erase the only copy.

**Warning:** Asynchronous replication can still lose recent entries. Durability depends on write acknowledgement policy, persistence and the failure scenario—not only the existence of replicas.

### Q14. What are retries and DLQ?

- **Retry:** Temporarily failed DB writes get another attempt after backoff (preferably with jitter).
- **DLQ:** When using a queue with a redrive policy, a repeatedly failing message can be isolated for inspection.
- **Idempotency:** A retry/replay must not double-charge, duplicate orders, or accidentally double-apply a counter update.

**Illustrative flow:** write fails -> retry 1 -> retry 2 -> retry 3 -> DLQ, if configured with that threshold. A DLQ is **not a feature of every WAL or cache**.

### Q15. What is recovery/replay?

After a failed database/worker recovers, consume previously durable pending changes and persist them.

**Scenario:** DB down -> queue accepts updates -> worker pauses/retries safely -> DB recovers -> worker replays backlog -> successful records are acknowledged/deleted -> processing catches up.

**Operational controls:** keep message retention longer than expected outage/recovery, rate-limit replay to protect DB, monitor oldest message age, use idempotency and reconcile final state.

---

## 8. Cache expiry and eviction

**TTL (time to live)** expires cached entries after an interval; **eviction** removes keys due to capacity policy (or another memory management rule).

Example Redis command:

```redis
SET product:1001 "iPhone" EX 300
```

This sets a 5-minute expiry.

| Policy / technique | Purpose |
|---|---|
| TTL | Bound age of ordinary cached data |
| LRU | Evict least recently used keys (policy/implementation dependent) |
| LFU | Evict least frequently used keys |
| TTL jitter | Spread expirations so many hot keys do not expire simultaneously |
| Negative caching | Cache short-lived “not found” results carefully |
| Stale-while-revalidate | Allow acceptable stale data during asynchronous refresh |
| Refresh-ahead | Proactively update keys expected to be requested soon |

**Never treat a TTL as a substitute for a correctness/authorization policy.**

---

## 9. Monitoring and troubleshooting

| Component | Metric / signal | Interpretation |
|---|---|---|
| Application | p95/p99 API latency; 5xx | Customer impact |
| Redis | Cache hit/miss ratio | Whether reads are being served from cache |
| Redis | Latency, connection errors/timeouts | Network or server health |
| Redis | Evictions, used memory | Capacity / eviction-policy issues |
| Redis replication | Replica lag, role/failover events | HA health and risk |
| Database | Queries/sec, connections, CPU, latency | Fallback pressure or DB bottleneck |
| Worker / queue | Retry count, queue depth, oldest age, DLQ count | Write-behind backlog, persistent failures |

### Diagram 10 — Ten-minute Redis outage investigation

```mermaid
flowchart TD
  A["API p99 / 5xx spikes"]
  B{"Redis health?"}
  C["Check failover, endpoint, client reconnects"]
  D{"DB fallback load safe?"}
  E["Constrain fallback / rate limit"]
  F["Check cache misses, evictions, hot keys"]
  G["Restore cache and warm gradually"]
  H["Verify API, Redis and DB metrics"]
  A --> B
  B -->|"Down / failover"| C --> D
  B -->|"Healthy"| F --> D
  D -->|"No"| E --> G
  D -->|"Yes"| G
  G --> H
```

**Incident response:** identify customer impact -> check Redis node/endpoint events and cache client errors -> confirm DB safety -> allow managed failover or restore cache -> reconnect -> gradual warming -> validate hit ratio, p99, errors and DB load.

---

## 10. Fast revision — Terms and problems solved

| Term | In one line |
|---|---|
| Invalidation on DB writes | Remove/update stale cached data after DB changes |
| Invalidate on DB commit | Do so only after successful transaction commit |
| Cache warming | Pre-fill cache before normal demand |
| Single-flight loading | Coalesce concurrent misses for the same key |
| Replicated cache | Maintain copies to survive cache-node failure |
| Bounded timeouts | Stop waiting for slow Redis quickly |
| Fallback to DB | Query authoritative DB when cache unavailable, within limits |
| Preload hot keys | Warm only the most frequently used cache keys |
| Durable queue / WAL | Record accepted pending changes before final persistence |
| Replication | Reduce single-node loss risk; not equal to zero-data-loss |
| Retries | Reattempt transiently failed work safely |
| DLQ | Isolate repeated failures in queue-backed processing |
| Recovery/replay | Reprocess durable pending work after outage |

---

## 11. Interview follow-ups

**Q. Why should I use cache-aside over read-through?**  
Cache-aside is straightforward when the application owns the DB query and cache-population code. Read-through is useful with a shared loader abstraction. Both need invalidation/TTL and cache-failure policies.

**Q. Redis primary fails: is there zero downtime?**  
No. Replicas/managed failover reduce downtime, but clients may see transient errors and need reconnect logic.

**Q. Can my DB survive 100% cache misses?**  
Do not assume so. Design a bounded fallback and load-shedding policy. Warm hot keys and prevent stampedes.

**Q. Is Redis replication a backup?**  
No. Replication provides availability of copies but can replicate bad data/deletes; durability/backups/DR need independent consideration.

**Q. Are WAL and DLQ interchangeable?**  
No. WAL is a write-recovery mechanism; DLQ isolates failed processing messages in queue-based workflows.

**Q. What happens if DB commit succeeds but cache invalidation fails?**  
Cache can be stale. Alert/retry invalidation using CDC/outbox where required, use TTL and race-aware/versioned strategies for stricter consistency.

**Q. Should payments use write-behind?**  
Not by default. Payment correctness normally needs durable, strongly controlled transaction semantics. If asynchronous processing is used, persist commands durably, apply idempotency and expose states clearly.

---

## 12. 90-second architect / SRE interview answer

> Caching stores frequently accessed data in a faster layer such as Redis, reducing database traffic and API latency. For a typical EKS or AKS API I would start with cache-aside: check Redis first, and on a miss read the authoritative database and populate cache with TTL. Read-through works similarly, except a cache abstraction loads the missing value rather than the application doing it explicitly.
>
> For consistency, I would invalidate stale keys after successful DB commits, and consider versioning, CDC or an outbox if reliable invalidation is essential. For performance, I would warm hot keys and use single-flight loading to avoid stampedes.
>
> For HA, I would select managed Redis with replicas across AZs and automatic failover, then configure client timeouts, reconnection, circuit breakers and a rate-limited database fallback. A Redis failure must not turn into a database failure.
>
> If the workload needs write-behind behavior, I would distinguish a disposable cache from acknowledged business writes and ensure an appropriate durable log or queue, retry handling, DLQ where applicable and idempotent replay.

**Remember:** **Consistency** = invalidation/TTL/versioning. **Performance & availability** = warming/single-flight/replicas/timeouts/fallback. **Durability & recovery** = queue/WAL/replication/retries/DLQ/replay.
