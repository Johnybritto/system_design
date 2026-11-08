# DynamoDB Troubleshooting and Recovery — Apple ASE SRE Mock Interview

> **Companion to:** [09 — PostgreSQL/RDS Troubleshooting and Recovery](./09_Apple_ASE_SRE_Database_Troubleshooting_Recovery_Mock_Interview.md)
>
> **Goal:** Scenario-based SRE preparation for DynamoDB incidents: throttling, access patterns, partitions, indexes, consistency, transactions, global tables, PITR, idempotency, and recovery verification.
>
> **Interview method:** **Impact → Evidence → Cause → Immediate mitigation → Permanent fix → Verification.**
>
> All metrics and incident timings below are **illustrative interview scenarios**, not DynamoDB defaults or guaranteed recovery objectives.

---

## 0. Understand the architecture first

### Typical single-Region design

~~~mermaid
flowchart TD
    A["Clients"] --> B["Route 53 / CDN / ALB"]
    B --> C["EKS application pods"]
    C --> D["AWS SDK for DynamoDB"]
    D --> E["Regional DynamoDB endpoint"]
    E --> F["Orders table — managed across multiple AZs"]
    E --> G["GSIs (if configured)"]
    C --> H["CloudWatch, tracing, application logs"]
~~~

- **DynamoDB is managed and regional.** AWS handles partition placement and Multi-AZ durability. There is no PostgreSQL-style primary instance, standby instance, or SQL connection pool to operate.
- **Tables are designed around access patterns.** The partition key (PK), optional sort key (SK), and indexes determine what you can retrieve efficiently.
- **On-demand is not unlimited.** Hot key ranges, quotas, configured maxima, and rapid traffic jumps can still throttle requests.
- **A DynamoDB Global Table is a set of regional replicas.** Applications call a regional endpoint; there is no single global DynamoDB query endpoint.
- **HTTP client connections are reused by AWS SDKs.** Diagnose SDK retries, timeouts, HTTP connection exhaustion, credentials, and endpoint reachability—not database sessions or pg_locks.

### Mapping to our 5 geographies / 10 data centers example

~~~mermaid
flowchart TD
    A["Users"] --> B["Global user traffic routing"]
    B --> C["Geo 1 apps — DC A + DC B"]
    B --> D["Geo 2 apps — DC A + DC B"]
    B --> E["Geo 3 apps"]
    B --> F["Geo 4 apps"]
    B --> G["Geo 5 apps"]
    C --> H["DynamoDB Region 1"]
    D --> I["DynamoDB Region 2"]
    E --> J["DynamoDB Region 3"]
    F --> K["DynamoDB Region 4"]
    G --> L["DynamoDB Region 5"]
    H <-->|"Global Tables MREC"| I
    I <-->|"Replication"| J
    J <-->|"Replication"| K
    K <-->|"Replication"| L
~~~

**Interpretation:** Each geography has two application DCs, but **DynamoDB is not installed inside either DC**. Each application deployment accesses an AWS regional endpoint (privately via suitable AWS networking where applicable). Replication is managed between DynamoDB regional replicas, not between the 10 application DCs. Choose the actual supported AWS Regions and connectivity; geography names in this conceptual diagram do not assert an exact AWS footprint.

**Important 2026 distinction:** Global Tables support **MREC** (multi-Region eventual consistency) and **MRSC** (multi-Region strong consistency). MREC is the practical model for arbitrary five-Region placement. MRSC has restricted Region sets and exactly three Regions (or two replicas plus witness); it is **not** a general-purpose five-Region strong-consistency replacement. MRSC also does not support DynamoDB transaction APIs. See AWS references at the end.

---

## Scenario 1 — API p99 jumps from 200 ms to 5 seconds

**Production:** EKS → DynamoDB Orders table.

| Observation | Value |
|---|---|
| API p99 | 200 ms → 5 s |
| API errors | Increased |
| EKS pods | Healthy |
| DynamoDB | Throttled requests reported |
| Recent deployment | None |

**Interviewer:** "You're on call. What do you check first?"

### Investigation

1. **Impact:** Which endpoint, Region, tenant, read or write operation is slow? Compare request rate, p95/p99, error rate, and customer impact.
2. **Application layer:** Check SDK exceptions, retry attempts, circuit-breaker state, concurrency, cache behavior, and upstream bursts.
3. **DynamoDB layer:** Inspect CloudWatch **SuccessfulRequestLatency**, **ConsumedReadCapacityUnits**, **ConsumedWriteCapacityUnits**, throttling metrics and specific **ThrottlingReasons** returned in exceptions.
4. **Resource:** Determine whether **Table** or **Index** is throttled and whether cause is key range, provisioned throughput, account limit, or configured on-demand maximum.
5. **Mitigate:** Bound retries, reduce noncritical traffic/concurrency, restore cache effectiveness, and adjust capacity *only if capacity is actually the cause*.
6. **Verify:** API latency, end-to-end success, throttling frequency, and correct business outcomes—not just green pod health.

### Interview-ready answer

> I start with the affected API and Region, then correlate latency with SDK errors and retries. I inspect DynamoDB throttling reasons and the affected table or GSI. I protect critical customer traffic by limiting excess concurrency and retry amplification while addressing the actual capacity or key-distribution problem. I close the incident only after customer latency and correctness recover.

**Correction:** Unlike RDS PostgreSQL, increasing DB connection pools or checking SQL locks is not the right first-line DynamoDB response.

---

## Scenario 2 — A hot partition even though table capacity looks healthy

**Production:** 20,000 requests/s arrive, and most reads target the same celebrity account or popular product.

- Table-level read capacity appears available.
- One key range is receiving excessive requests.
- Error reason: **TableReadKeyRangeThroughputExceeded**.

**Interviewer:** "Why is DynamoDB throttling when total capacity is not exhausted?"

**Answer:** DynamoDB distributes items by partition key. A hot key/key range may hit a per-partition throughput ceiling even with unused throughput elsewhere.

| Immediate | Longer-term |
|---|---|
| Cache very hot reads where safe | Distribute access via better key design |
| Limit request concurrency and retries | Add write sharding / bucketing when suitable |
| Investigate Contributor Insights for hot keys | Remove single low-cardinality keys |
| Check adaptive capacity/warm-throughput guidance | Reassess access patterns and GSIs |

**Useful reference limits:** AWS documents an approximate key-range ceiling of **3,000 read units/s or 1,000 write units/s**, subject to item size, read consistency, and partition splitting/adaptive capacity. These are **not table-wide DynamoDB maximums**.

**Interview-ready answer:**

> This is a key-distribution problem, not necessarily a table-wide capacity shortage. I'll use the throttling reason plus Contributor Insights to identify the hot partition key, mitigate with caching/load shedding, and fix the access pattern or partition-key design.

---

## Scenario 3 — Provisioned versus on-demand throttling

**Interviewer:** "The service is serverless. Why would requests still get throttled?"

| Cause | Evidence | Fix |
|---|---|---|
| Provisioned RCU/WCU exhausted | Table/Index read/write ProvisionedThroughputExceeded | Adjust capacity or autoscaling after assessing demand |
| Hot key range | KeyRangeThroughputExceeded | Key distribution, cache, request shaping |
| On-demand account ceiling | AccountLimitExceeded | Validate service quota and request increase if justified |
| Configured on-demand maximum | MaxOnDemandThroughputExceeded | Increase configured maximum with cost approval |
| Sudden > established on-demand warm throughput | Throughput-related throttling during abrupt ramp | Plan pre-warming / gradual ramp-up |

**Don't say:** "I'll switch to on-demand and throttling will stop." It may reduce provisioning issues, but not hot keys, quotas, or all burst-related limits.

**Follow-up:** "Would you increase WCU immediately?" Only if diagnostics identify **write capacity** as the bottleneck; increasing capacity does not fix a poorly distributed partition key.

---

## Scenario 4 — Scan becomes slower as the table grows

**Production:** The Orders table has 20 million items. A new endpoint lists all orders for a given customer.

**Bad approach (illustrative):**

~~~text
Scan Orders
FilterExpression: customerId = 123
~~~

**Question:** "Why is this slow? How do you fix it?"

- **Scan** evaluates data across the table/index. A filter does **not** prevent the underlying reads from consuming capacity.
- **Query** efficiently reads items that match a partition-key value, optionally narrowed with a sort-key condition.
- Design Orders with **PK = CUSTOMER#123**, **SK = ORDER#<timestamp>#<orderId>**, if that matches the main access pattern.
- If the existing key is **orderId**, consider a **GSI keyed by customerId**, accounting for write cost and eventual consistency.
- Query paginates; a single page reads up to **1 MB** before further pagination.

**Interview-ready answer:**

> In PostgreSQL, I would inspect EXPLAIN and indexes. In DynamoDB, I inspect the access pattern, key schema, Query vs Scan behavior, consumed capacity, and GSI design. I avoid a full-table Scan on a customer-facing hot path.

---

## Scenario 5 — GSI is the bottleneck, not the base table

**Production:** Order writes fail intermittently, although the Orders table appears to have adequate write capacity.

**Discovery:** An index has partition key **status** with values like PENDING, PAID, FAILED. A large percentage of traffic writes to PENDING.

**Question:** "How can an index throttle writes to the table?"

**Answer:** GSIs are maintained asynchronously. An underprovisioned or hot GSI can cause **write back-pressure**, throttling base-table writes.

**Check:** The **IndexWrite…** throttling reason and the index ARN; GSI metrics and key distribution.

**Mitigation:** Reduce affected write pressure; change index capacity if provisioned capacity is the cause; redesign a low-cardinality/hot index key (for example, bucket by tenant/time/shard) for the long term. Evaluate the migration and backfill impact.

**Remember:** GSIs support **eventually consistent reads only**. A strong read on the base table does not make a GSI strong.

---

## Scenario 6 — Lost update: two workers modify the same item

**Production:** Two workers read inventory = 10. Each wants to decrement by one. The application performs naive read-modify-write and eventually stores the wrong value.

**Interviewer:** "Is this a PostgreSQL lock/deadlock problem?"

**Answer:** Not the same. Use **atomic UpdateItem operations**, **ConditionExpression**, or version-based optimistic concurrency control.

**Example (conceptual):**

~~~text
UpdateItem(
  Key = {PK: "ITEM#101"},
  UpdateExpression = "SET stock = stock - :one",
  ConditionExpression = "stock >= :one"
)
~~~

If the stock condition fails, the client gets **ConditionalCheckFailedException**. That is normally a business/concurrency outcome—not evidence that DynamoDB is down.

**Interview-ready answer:**

> I would avoid a read-then-write race. I'd use a conditional atomic update, handle condition failures explicitly, and use retries only when the operation is safe to retry.

---

## Scenario 7 — Conditional check versus transaction conflict

**Interviewer:** "DynamoDB doesn't use PostgreSQL row locks the same way. What replaces deadlock troubleshooting?"

| DynamoDB outcome | Interpretation | Response |
|---|---|---|
| ConditionalCheckFailedException | Expected condition/version was false | Re-read or return business conflict; don't blindly retry |
| TransactionCanceledException | Transaction failed; inspect cancellation reasons | Distinguish condition failure, contention, throttling |
| TransactionConflict | Concurrent transactions contend on item(s) | Shorten critical path, limit contention, retry with jitter where safe |
| ThrottlingException | Throughput restriction | Diagnose reason, fix capacity/key load |

**Important:** Don't suggest **pg_locks**, killing PID 101, or restarting the database. DynamoDB offers atomic item operations and regional transaction APIs, not SQL session/lock administration.

---

## Scenario 8 — Multi-item transaction: order + outbox

**Production:** A message or order must be stored together with its outgoing event marker.

~~~mermaid
flowchart TD
    A["Application request"] --> B["TransactWriteItems in one Region"]
    B --> C["Put Order"]
    B --> D["Put Outbox event"]
    C --> E["Atomic commit or transaction failure"]
    D --> E
    E --> F["DynamoDB Streams / Publisher"]
    F --> G["SQS or Kafka"]
~~~

**Question:** "Can DynamoDB guarantee both items are written together?"

**Yes, within the same AWS Region:** **TransactWriteItems** supports atomic multi-item writes across eligible tables in that account/Region. It is **not** a cross-Region transaction; MREC Global Tables may expose only a subset of the transaction's items temporarily as they replicate.

**Limits to remember:** Up to **100 distinct items** and **4 MB total transaction data**; the same item cannot be targeted twice in one transaction.

**Caution:** MREC provides local transaction guarantees, not global atomic visibility. **MRSC Global Tables do not support transaction APIs**. Choose the data model according to the application invariant.

---

## Scenario 9 — Accidental delete of two million orders: PITR

**Incident timeline (UTC):**

| Time | Event |
|---|---|
| 10:05 | A faulty job deletes two million valid items |
| 10:20 | On-call is paged |
| 10:20 onward | Production continues accepting new orders |

**Interviewer:** "How do you recover the deleted items without losing new orders?"

~~~mermaid
flowchart TD
    A["Stop faulty delete job"] --> B["Keep safe production writes"]
    A --> C["Check PITR window"]
    C --> D["Restore just before deletion to NEW table"]
    D --> E["Compare item keys with production"]
    E --> F["Selectively reinsert missing items"]
    F --> G["Reconcile recent updates and downstream effects"]
    G --> H["Validate counts and business invariants"]
~~~

**PITR facts:**
- PITR is configurable for a **1–35 day** recovery period.
- Restore creates a **new table**, not an in-place rollback.
- A recovery point must fall within the supported restorable interval (check **EarliestRestorableDateTime** / **LatestRestorableDateTime**; typically not the last ~5 minutes).
- Do **not** redirect production to a stale restored table without reconciling newer writes.

**Illustrative CLI:**

~~~bash
aws dynamodb describe-continuous-backups \
  --table-name Orders --region us-east-1

aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name OrdersRecovery \
  --restore-date-time "2026-11-08T10:04:00Z" \
  --region us-east-1
~~~

*Use a timestamp verified to be before the destructive operation, within the actual restorable window, and expressed in UTC.*

### Interview-ready answer

> I stop the destructive writer, restore the table to a new isolated table just before the deletion, compare original item keys to live production, and selectively restore missing items using conditional writes. I preserve orders written afterward and validate dependent data and downstream effects before closing the incident.

---

## Scenario 10 — How do you selectively recover missing items?

**Interviewer:** "What prevents you from overwriting a customer's newer order?"

**Recovery algorithm:**

1. Determine the **expected key set** from the restored table and identify known deletes versus legitimate subsequent modifications.
2. Read each target key from production and compare the version / timeline.
3. Insert only genuinely missing items using **attribute_not_exists(PK)** as a conditional PutItem (for a composite key, use the complete key in Key and an appropriate condition).
4. **Never overwrite an existing newer value by default.** Send conflicts to a controlled reconciliation process.
5. Reconcile Streams-driven side effects, payments, and notifications; don't accidentally resend old events.

**Illustrative conditional operation:**

~~~text
PutItem(
  TableName = "Orders",
  Item = restoredOrder,
  ConditionExpression = "attribute_not_exists(PK)"
)
~~~

**Warning:** A count match is not sufficient. Compare exact primary keys, important attributes/versions, totals, and related events. Scans/exports for large-scale verification require planned capacity and may yield a changing view of active production; use a defined cutoff and reconciliation strategy.

---

## Scenario 11 — Entire AWS Region outage: Global Tables

**Production:** Five active application geographies with a DynamoDB MREC Global Table in five AWS Regions.

**Interviewer:** "What happens if one Region fails?"

- Global Tables provide **regional endpoints**, **not** automatic client traffic failover.
- The application/global routing layer must send affected users to another **healthy application Region**; the new application uses its **local DynamoDB replica**.
- Healthy replicas continue serving writes in MREC.
- **RPO is not guaranteed zero** for a sudden failure: recent writes not yet replicated may be absent in another Region.
- Replication catches up after the failed Region recovers, subject to conflict resolution; operationally reconcile any missing writes.

~~~mermaid
flowchart TD
    A["Region A app/data unavailable"] --> B["Global routing removes Region A app"]
    B --> C["Route customer to healthy Region B app"]
    C --> D["Region B DynamoDB replica"]
    D --> E["Verify business record and retry safety"]
    E --> F["Reconcile Region A outstanding writes after recovery"]
~~~

**Interview-ready answer:**

> DynamoDB handles replication, but I must fail over application traffic. I monitor regional health and ReplicationLatency. The target Region may not have the latest unreplicated writes, so I need idempotency, reconciliation, and an RPO defined by the business.

---

## Scenario 12 — Concurrent updates from different Regions

**Production:** Geo 1 writes order state = PAID while Geo 2 writes state = CANCELLED to the same global-table item.

**Question:** "Which update wins?"

For **MREC**, concurrent writes are handled through **last-writer-wins conflict resolution**, based on DynamoDB's internal timestamps. The surviving result may violate application intent; a local conditional expression does **not** enforce global serializability.

**Safer design for critical state:**
- Give each order/customer a **home write Region**, and route all writes for that item there.
- Use event/state-machine constraints, idempotency, and controlled conflict reconciliation.
- Consider MRSC only if its Region availability, latency, feature restrictions, and consistency guarantees fit; **MRSC is not suitable for arbitrary five-Region deployment**.

**Do not claim:** "Global Tables automatically merge business state correctly."

---

## Scenario 13 — Why does a freshly written item look missing?

**Production:** User updates a profile in Geo 1 then immediately reads it in Geo 4.

**Interviewer:** "The write succeeded. Why is the old value returned?"

- The **MREC cross-Region replica may lag**.
- A read on a **GSI** may also lag even within a Region.
- On a **base table in the same Region**, GetItem/Query/Scan may request a strongly consistent read using **ConsistentRead=true**.
- Strong reads on a local MREC replica do **not** make remote-Region updates immediately visible.
- **MRSC** supports globally strongly consistent reads within supported Region configurations, with different latency/availability tradeoffs.

**Solution:** For strict read-after-write behavior, direct the user to the write Region / authoritative item owner or design around consistency guarantees. Do not use blind sleeps as the primary correctness mechanism.

---

## Scenario 14 — 5xx or timeout after a write: did the write commit?

**Production:** An order creation request times out after the client sent PutItem.

**Interviewer:** "Do you retry or assume the write failed?"

**Answer:** The outcome is **unknown** to the client. The original write may have succeeded.

**Safe approach:**
1. Retain stable **orderId/idempotencyKey** across attempts.
2. Use **conditional PutItem** or a transaction containing a durable idempotency record.
3. On retry, read the existing result and respond consistently.
4. Use bounded SDK retries with exponential backoff and full jitter.
5. For financial state changes, reconcile against external payment authority before acting.

**Critical nuance:** The **ClientRequestToken** for TransactWriteItems is valid for only **10 minutes**. For durable business idempotency, store the idempotency key/result as data with appropriate retention. Don't depend on the API token alone.

**One-liner:** "Timeout ≠ rollback. I make writes idempotent and verify the final state."

---

## Scenario 15 — SQS retries cause duplicate database writes

**Production:** An SQS consumer writes an order and crashes before acknowledging/deleting the SQS message. Another consumer receives it.

**Question:** "How do you prevent duplicates?"

~~~text
SQS at-least-once delivery
       ↓
Worker retries message
       ↓
Conditional PutItem using stable orderId
       ↓
Existing order? Treat as already processed (verify payload)
       ↓
Acknowledge only after durable success
~~~

- A **conditional write** or **transaction with dedupe marker** protects the invariant.
- An item with the same key may have a different payload; validate before treating it as a duplicate.
- Do not use read-then-write without a condition—two workers may race.
- If writing multiple tables/items atomically, prefer **TransactWriteItems** in one Region.

---

## Scenario 16 — Massive retry storm after DynamoDB slows down

**Production:** 100 application pods each retry aggressively on throttling, multiplying traffic.

**Question:** "Will increasing application replicas help?"

Usually **not**. More pods may amplify an already overwhelmed key or index.

**Immediate actions:**
- Bounded exponential backoff **with jitter**, and reasonable max attempts/timeouts.
- Concurrency limits, load shedding, queue buffering where appropriate.
- Cache hot reads; prioritize critical operations.
- Track original request rate versus **total SDK attempts** to identify retry amplification.
- Fix the underlying key/capacity problem before scaling traffic.

**SDK detail:** Prefer the AWS SDK's built-in standard/adaptive retry strategies configured for your service profile; avoid stacked application-level retry loops without a clear budget.

---

## Scenario 17 — DynamoDB Streams / outbox consumer falls behind

**Production:** Orders are successfully written but downstream notifications arrive late.

**Interviewer:** "Is DynamoDB losing data?"

Not necessarily. The **database write succeeded**, but a downstream consumer may be lagging.

**Investigate:**
1. Streams-enabled source and event filter.
2. Consumer errors, iterator age/lag, throttling, batch failure, and DLQ.
3. Visibility/retry semantics of downstream SQS or Kafka publishing.
4. Duplicated or out-of-order delivery and poison records.
5. Consumer scaling within stream-shard/order constraints.

**Important:** DynamoDB Streams retention is **24 hours**. Streams are a change feed, not a substitute for PITR or a long-term audit log. Build a recovery/replay strategy beyond stream retention.

**Interview-ready answer:**

> I separate the committed DynamoDB write from asynchronous downstream processing. I diagnose consumer lag and duplicates, restore processing from a known checkpoint when possible, and reconcile missing events from durable application data if the stream history has expired.

---

## Scenario 18 — TTL deleted customer sessions unexpectedly

**Production:** Session records disappear and users are logged out.

**Interviewer:** "Does TTL delete exactly at the expiry timestamp?"

**No.** DynamoDB TTL deletion is asynchronous; expired items can remain for a period before removal. Applications must check TTL values themselves where authorization or expiry correctness matters.

**Diagnose:** Confirm item timestamps are Unix epoch **seconds**, clock calculations, TTL-enabled attribute, and application/session policy. Avoid using TTL timing to guarantee precise business scheduling.

**Recovery:** If a session is intentionally expired, don't resurrect it indiscriminately from PITR. For accidental TTL misconfiguration, isolate the writer/configuration issue and restore only appropriate data after security review.

---

## Scenario 19 — EKS pods are healthy, but DynamoDB returns authorization/network errors

**Production:** API reports errors after a deployment.

| Symptom | Likely area |
|---|---|
| AccessDeniedException | IAM role / IRSA or pod identity / KMS permissions |
| Credentials errors | AWS SDK credential provider / STS connectivity / token expiry |
| Connection timeout / DNS errors | Network routes, DNS, proxy, VPC endpoint, NAT, egress policy |
| ResourceNotFoundException | Table name / AWS Region / deployment configuration |
| ValidationException | Invalid request, key schema, expression, item |

**Troubleshoot:**
1. Check per-operation exception **type** and request ID.
2. Check pod identity and effective IAM permissions, including GSI/Streams/KMS where relevant.
3. Verify the SDK's configured **Region and endpoint**.
4. Check EKS egress, VPC endpoint, route/DNS, TLS, proxy, and **NetworkPolicy**.
5. Verify API latency and successful operations; do not reflexively restart all pods.

**Remember:** Unlike PostgreSQL, DynamoDB uses managed HTTPS APIs—not a database TCP session on port 5432.

---

## Scenario 20 — Recovery verification: APIs green, but records still missing

**Production:** Two million items were expected to be recovered; only 1,995,000 are present. API error rate is normal.

**Interviewer:** "Can you declare the incident resolved?"

**No.** You still have **5,000 unexplained records**.

### Production recovery checklist

- [ ] Identify the exact missing **PK/SK** pairs, not just total count.
- [ ] Confirm no newer legitimate deletes/updates were overwritten.
- [ ] Validate expected versions, amounts, statuses, and key business invariants.
- [ ] Confirm related items (payments, outbox, entitlement, inventory) are consistent.
- [ ] Reconcile Streams/queue events, avoid duplicate side effects, inspect DLQs.
- [ ] Validate application p95/p99, errors, regional behavior, and throttling.
- [ ] Preserve an audit trail of original loss, restored keys, skipped/conflicting keys.
- [ ] Perform targeted read verification and obtain business-owner signoff.

**Interview answer:**

> Healthy APIs are not proof of data integrity. I compare expected keys against production at a controlled cutoff, restore only confirmed missing items, reconcile conflicts and downstream effects, and obtain evidence-backed business validation before closure.

---

## Follow-up 21 — RPO and RTO: single-Region versus Global Tables

| Design / failure | RPO | RTO / caution |
|---|---|---|
| DynamoDB single-Region AZ loss | Designed for durable Multi-AZ operation | Service handles infrastructure; application may still observe transient errors |
| DynamoDB Global Tables MREC Region loss | **May be nonzero** (pending asynchronous replication) | Application traffic rerouting + state reconciliation determine practical recovery |
| DynamoDB Global Tables MRSC | Designed for **zero RPO** committed writes across its quorum configuration | Requires supported Region set/quorum; latency and features differ |
| Accidental delete using PITR | Restore point is in the **past**, so it cannot preserve subsequent writes by itself | New table + selective reconciliation; depends on data volume/process |

**Interviewer trap:** "Global Tables = RPO zero." **Not for MREC**.

**Interviewer trap:** "DynamoDB automatically fails over users globally." It does not route application traffic for you.

---

## Follow-up 22 — Can you use SQL commands, EXPLAIN, indexes, and locks?

| PostgreSQL/RDS concept | DynamoDB equivalent or difference |
|---|---|
| SQL EXPLAIN for slow query | Analyze key schema, access pattern, Query vs Scan, consumed capacity |
| B-tree index | Base key, GSI, LSI; different consistency and write-cost tradeoffs |
| pg_stat_activity / session count | CloudWatch request metrics, SDK errors, API attempts / HTTP client behavior |
| pg_locks / deadlock | Conditional checks, optimistic concurrency, transaction conflicts |
| Multi-AZ primary and standby | Managed Regional Multi-AZ; no user-managed DB instance promotion |
| Read replica | Global Tables regional replicas; within Region GSIs aren't primary DB replicas |
| WAL / PITR | Managed DynamoDB PITR to **new table**; don't describe user-managed WAL replay |
| SQL ON CONFLICT DO NOTHING | PutItem with condition such as attribute_not_exists(PK) |
| Database connection pool | AWS SDK HTTP connection reuse / request concurrency |

**One-liner:** "DynamoDB troubleshooting is mostly request patterns, key distribution, quotas, consistency, and SDK behavior—not SQL sessions and query planners."

---

## Five AWS CLI commands worth knowing

**These commands are observational / example workflow commands; run them against the correct Region, account, table and approved permissions.**

### 1. Check table status, capacity configuration, key schema, GSIs

~~~bash
aws dynamodb describe-table \
  --table-name Orders \
  --region us-east-1
~~~

### 2. Check backups and PITR recovery window

~~~bash
aws dynamodb describe-continuous-backups \
  --table-name Orders \
  --region us-east-1
~~~

### 3. Query by partition key (instead of Scan)

~~~bash
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"CUSTOMER#123"}}' \
  --region us-east-1
~~~

### 4. Strong read for an item in the same Region (base table)

~~~bash
aws dynamodb get-item \
  --table-name Orders \
  --key '{"PK":{"S":"CUSTOMER#123"},"SK":{"S":"ORDER#456"}}' \
  --consistent-read \
  --region us-east-1
~~~

### 5. Inspect table-level throttling and capacity in CloudWatch

~~~bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --statistics Sum --period 60 \
  --start-time 2026-11-08T09:00:00Z \
  --end-time 2026-11-08T09:30:00Z \
  --region us-east-1
~~~

For specific throttling investigations, combine CloudWatch with the **ThrottlingReasons** in SDK exceptions and Contributor Insights. A table-level aggregate alone will not expose every hot-key or GSI problem.

---

## Final one-page cheat sheet

| Incident | First diagnostic question | First safe move |
|---|---|---|
| High p99 + 429/throttling | Which table/index and exact ThrottlingReason? | Bound retries and investigate capacity/key load |
| One user/product causes slowness | Is a PK/key range hot? | Cache, limit concurrency, redesign key |
| Writes throttled but table seems fine | Is a GSI causing back-pressure? | Inspect GSI throttling / ARN |
| Slow search/listing endpoint | Query or Scan? | Stop costly Scan pattern, model access path |
| Lost inventory update | Missing condition/version check? | Atomic conditional update |
| Concurrent transaction errors | Business condition vs contention? | Distinguish and retry selectively |
| Accidental mass delete | Is PITR enabled and within window? | Restore to separate table; reconcile |
| Region outage | Which application Region and data replica are healthy? | Route application traffic, assess RPO |
| Geo A and B update same item | MREC last-writer-wins conflict? | Single-writer per entity / reconcile |
| Read returns stale data | Cross-Region lag or GSI? | Choose authoritative read path |
| Ambiguous write timeout | Did write commit before timeout? | Stable idempotency key + condition |
| SQS redelivery | Can duplicate worker write again? | Conditional Put / transactional dedupe |
| Streams delayed | Consumer iterator age / processing errors? | Repair consumer / replay strategy |
| AccessDenied | Wrong IAM/pod identity/KMS? | Fix authorization; don't restart DB |
| Recovery says "complete" | Have you compared exact keys and business data? | Reconcile and verify before closure |

---

## 10 rapid-fire mock interview questions

1. **What is a partition key?** The attribute used in distributing and locating items. A poor distribution can create hot key ranges.
2. **Sort key?** Orders related items within the same partition key and supports sort-key conditions.
3. **Query vs Scan?** Query uses key conditions; Scan traverses table/index data and is generally costly for selective APIs.
4. **Can a GSI provide strongly consistent reads?** **No**; GSI reads are eventually consistent.
5. **Does on-demand guarantee no throttling?** **No**; partition, quota, configured maximum and ramp behavior still matter.
6. **How do you prevent overwriting the same order?** Conditional writes / version checks / stable idempotency IDs.
7. **What does PITR restore?** A **new table**, not a rollback of existing live production data.
8. **Is Global Tables MREC strongly consistent across Regions?** **No**; asynchronous replication and conflicts need planning.
9. **Do Global Tables automatically redirect application traffic?** **No**; the app's global traffic routing must fail over.
10. **What would you monitor?** Request p99/errors, throttling reason by table/index, consumed capacity, hot keys, SDK retries, and MREC replication latency.

### Recommended revision order (Apple ASE SRE)

**P0:** Scenarios **1–5** (latency, throttling, partition keys, Query/Scan, GSI), **9–14** (PITR, recovery, Global Tables, consistency, idempotency).

**P1:** Scenarios **6–8** (conditional writes, contention, transactions), **16–20** (retry storm, Streams, IAM/network, verification).

**P2:** TTL, DAX/cache tradeoffs, index migrations, advanced capacity planning.

**Final interview formula:**

> Confirm the impact → inspect **SDK exception + ThrottlingReasons + CloudWatch** → identify **table / GSI / hot key / consistency issue** → mitigate safely with **bounded retries, traffic shaping, or recovery** → verify data correctness and customer SLOs.

---

## AWS official references (verified November 2026)

- [DynamoDB throttling diagnostic workflow](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/throttling-diagnosing-workflow.html)
- [Throttling reasons and resolutions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/troubleshooting-throttling-diagnostics.html)
- [GSI write throttling and back-pressure](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/gsi-throttling.html)
- [DynamoDB read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [DynamoDB global table design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-global-table-design.html)
- [DynamoDB global tables consistency modes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)
- [Transactions and Global Tables limitations](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)
- [TransactWriteItems request idempotency](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html)
- [PITR backup and restore](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Backup-and-Restore.html)

**Document scope:** Production SRE troubleshooting and recovery practice, not an endorsement of a specific multi-Region topology without confirmed application access patterns, latency requirements, data residency, recovery objectives, and costs.
