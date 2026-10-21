# Google Pay / UPI Payment System Design — Consolidated Mock Interview Notes
## Step-by-step flow, capacity estimation, candidate feedback, interview Q&A, failure remediation, RPO and RTO

> **Purpose:** Consolidated notes from our Google Pay/Apple Pay system-design walkthrough, the candidate's spoken mock answer, the answer review, and the short capacity-estimation revision sheet. Same stepwise format as the customer-facing API design mock.
>
> **Scope:** A **Google Pay-like UPI application in India**. It is an **illustrative interview architecture**, not a claim about Google's private implementation. NPCI/PSP-bank/participant-bank exchanges are shown at a high level: actual regulated payment execution follows supported participant interfaces. Google Pay does **not** own the payer's or beneficiary's authoritative bank balance.
>
> **Key principle:** An infrastructure timeout does **not** establish that a bank payment failed. Never blindly re-send an uncertain financial instruction. Preserve payment identity, enquire using the original reference, reconcile and only then decide what is safe.
>
> **Notation:** All volumes, SLIs, SLOs, availability targets, recovery timings and storage sizes below are **proposed interview assumptions**, not measured Google Pay figures or external-provider SLAs. **RPO = Recovery Point Objective; RTO = Recovery Time Objective.** RTO to restore our app is not the same as reversal/settlement turnaround at a bank.

---

# Step 1 — Functional and Non-functional Requirements

## Interviewer question

> Design a high-availability, scalable UPI payment application like Google Pay. What requirements would you gather?

## Critical user journey — functional

1. User opens the app and signs in.
2. User identifies a recipient through a UPI ID, QR code or a supported contact method.
3. App securely resolves or validates the recipient and shows confirmation details.
4. User enters an amount, confirms and completes the approved payment authorization flow.
5. Application submits the payment through its PSP Bank integration and the UPI network.
6. User can retrieve the latest status (PENDING, SUCCESS, FAILED, REVERSED) and payment history.
7. In-progress/uncertain outcomes are reconciled; reversal/refund and notification workflows are supported where applicable.

**Original mock answer, distilled:** "Making the payment or receiving the payment" is the central functionality. Find a user by UPI ID or QR code, then make the payment.

## Non-functional requirements

| Requirement | Interview clarification / proposed target |
|---|---|
| Availability | 99.99% availability of app-controlled critical APIs (proposed target) |
| Reliability | Do not lose acknowledged instructions or make unintended duplicate transfers |
| Scalability | Support peak events, business hours, weekends and festival spikes |
| Consistency | Strongly consistent, durable payment identity and state transitions |
| Latency | Aggressive p99 for lightweight API paths; separate external bank completion |
| Security | Device/session authentication, approved UPI authorization, encryption, fraud evaluation |
| Auditability | Payment attempts, status transitions, references and reconciliation evidence |
| Fault isolation | Separate third-party bank outages from app outages |
| Disaster recovery | Three-AZ deployment with cross-region recovery and tested objectives |
| Compliance | Retention, privacy, payments and applicable participant requirements |

**Follow-up Q: Can the service promise p99 below 200 ms for the entire transfer?**

**Answer:** Not safely. A **lightweight API** or an already persisted **payment status lookup** can target p99 < 200 ms (if load tests support it), whereas final payment completion relies on external bank/UPI processing. Define distinct SLOs for (a) request validation, (b) durable acceptance and (c) confirmed payment completion.

---

# Step 2 — Capacity Estimation (Short Interview List)

**State these as estimates, not real Google Pay statistics.**

| Metric | Proposed estimation |
|---|---:|
| Registered users | 100 million |
| Daily active users (DAU) | 20 million |
| Transactions per day | 20 million |
| Average TPS | ~231 (20,000,000 / 86,400) |
| Peak multiplier | 10x |
| Peak TPS | ~2,315 |
| Design capacity | 5,000 TPS |
| Raw record/event data | 2–4 KB per payment |
| Daily raw data | 40–80 GB |
| Monthly raw data (30 days) | 1.2–2.4 TB |
| Annual raw data (365 days) | 14.6–29.2 TB |
| p99 lightweight API target | Under 200 ms (subject to validation) |
| Application availability objective | 99.99% |
| Critical acknowledged payment-intent RPO | 0 **only if synchronously protected** |
| Regional recovery RTO target | 5–15 minutes |
| Deployment | 3 AZs + warm DR region |

**Important:** These storage estimates are raw data only. Add indexes, WAL/transaction logs, replication, backups, audit retention and reconciliation records. Average TPS is not peak TPS; load-test the peak and plan for external PSP/bank quotas.

**Follow-up Q: Why use a 10x peak factor?**

**Answer:** It is a planning assumption for spikes during festivals, salary periods or shopping. Validate actual peaks with traffic history and capacity tests; do not pretend the factor is a measured product fact.

**Follow-up Q: Should every service handle 5,000 TPS?**

**Answer:** Size by request type and downstream limits. Stateless APIs may handle that rate, but payment DB write IOPS, fraud decisions, queue throughput and partner bank limits need separate budgets and backpressure.

---

# Step 3 — Complete High-level Architecture

~~~mermaid
flowchart TD
  App["UPI Mobile App"] --> DNS["DNS"]
  DNS --> WAF["WAF and ALB"]
  WAF --> GW["API Gateway on EKS"]
  GW --> Auth["Authentication and Fraud Controls"]
  Auth --> Pay["Payment Service"]
  Pay --> DB[("PostgreSQL: Payments + Outbox")]
  DB --> Dispatch["Outbox Dispatcher"]
  Dispatch --> PSP["PSP Bank Integration"]
  PSP --> Network["PSP Bank and NPCI UPI"]
  Network --> Banks["Payer and Beneficiary Banks"]
  Banks --> Network
  Network --> Status["Status Handler"]
  Status --> DB
  DB --> Publisher["Internal Event Publisher"]
  Publisher --> Kafka["Kafka"]
  Kafka --> Notify["Notifications / Audit / Analytics"]
  DB --> Reconcile["Reconciliation Workers"]
  Reconcile --> PSP
  DB -. "Protected cross-region replication" .-> DR[("Warm DR Region")]
~~~

## Core components

| Component | Primary responsibility |
|---|---|
| DNS, WAF, ALB | Resolve, filter and route client traffic to healthy services |
| API Gateway | Route APIs, validate session/access, enforce quotas and rate limits |
| Auth / Fraud | Device, session, authorization and risk assessment |
| Payment Service | Persist durable payment intent; control payment state |
| PostgreSQL | Authoritative **application payment records**, idempotency and durable outbox |
| Redis | Non-authoritative caching/rate-limiting acceleration |
| Outbox Dispatcher | Find committed payment instructions and initiate permitted external call |
| PSP Integration | Invoke the bank-approved UPI participant interfaces |
| PSP/NPCI/Banks | Execute/authenticate/route the actual financial transfer |
| Payment Status Handler | Authenticate partner response; apply valid state transitions |
| Kafka | Asynchronous **internal** events, notification and analytics consumers |
| Reconciliation | Query uncertain transactions, recover confirmed states and reversals |
| Observability | Customer-level SLIs, component health and audit investigations |
| DR Region | Recover application and protected durable state |

**Critical correction to the mock explanation:** Kafka does **not** directly "talk to NPCI". A purpose-built PSP-bank integration service interacts through the approved participant APIs. You *can* queue payment commands, but external-execution idempotency and reconciliation are still required; Kafka alone is not an exactly-once debit guarantee.

---

# Step 4 — API Contracts and Idempotency

Use REST-style HTTPS from mobile client to API Gateway.

| Method | Example endpoint | Purpose |
|---|---|---|
| GET | /v1/payees/resolve?vpa=bob@bank | Validate/resolve payee through approved integration |
| POST | /v1/payments | Create one intended payment |
| GET | /v1/payments/{payment_id} | Fetch current payment status |
| GET | /v1/payments?limit=20 | Read user payment history |

**Illustrative API, NOT the proprietary Google Pay API:**

~~~http
POST /v1/payments
Authorization: Bearer <access_token>
Idempotency-Key: pay-12345
Content-Type: application/json

{
  "payer_account_ref": "account-123",
  "payee_vpa": "bob@bank",
  "amount_minor": 100000,
  "currency": "INR"
}
~~~

Amount is integer **paise**, not floating-point INR. The server derives trusted payer identity from the session and validates allowed accounts and recipients.

**Possible response after durably accepting intent (not bank success):**

~~~json
{
  "payment_id": "TXN1001",
  "status": "PROCESSING"
}
~~~

**Follow-up Q: Why POST for payments and GET for status?**

**Answer:** POST initiates a new operation with durable idempotency; GET reads existing state without causing a financial side effect.

**Follow-up Q: What if the user presses Pay twice, or HTTP retry occurs?**

**Answer:** A scoped unique idempotency key plus a stored payload fingerprint returns the same payment for a repeated *identical* request. Reject the same key with a different request body. Disabling the button is only a UI optimization.

**Follow-up Q: Why not store only the key in Redis?**

**Answer:** Redis may lose cache state. The authoritative idempotency record and uniqueness constraint must be durable in PostgreSQL.

---

# Step 5 — Client, DNS, Load Balancer and EKS

### Request path

~~~text
Mobile -> DNS resolution -> HTTPS -> WAF -> ALB
       -> API Gateway / Ingress -> EKS Service -> Payment Pod
~~~

- DNS resolves a hostname; it does not host the transaction session.
- For HTTPS REST, **ALB** is the natural AWS application-layer load balancer.
- ALB target groups may use **pod IP targets** with the appropriate EKS controller/configuration, or instance/node targets in other deployments.
- Use 3 AZs, readiness probes, topology spread constraints, disruption budgets, HPA and cluster node autoscaling.
- Rate-limit by approved dimensions; avoid overwhelming payment services or external partner quotas.

**Follow-up Q: What happens if a pod dies?**

**Answer:** Readiness/health routing removes unhealthy endpoints; request recovery uses durable payment state and idempotency. A retry must not create a second payment.

**Follow-up Q: Can LRU scale an ALB?**

**Answer:** No. LRU is a cache eviction policy, not load-balancer capacity management. ALB scaling, sufficient subnet IP addresses, healthy targets, autoscaling and load tests address load-balancer/application capacity.

---

# Step 6 — Authentication, Authorization and Fraud

1. Authenticate the application session and bind the request to the registered user/device.
2. Validate the payer's selected account and intended recipient.
3. Apply appropriate risk, frequency and policy checks.
4. Use the **approved UPI authorization flow** for PIN/transaction authentication; actual handling involves regulated participants and the remitter bank.
5. Never put a plaintext UPI PIN in an app database, Redis, Kafka or logs.

**Follow-up Q: What if the Fraud Service is down?**

**Answer:** Follow a previously approved risk-based degraded mode. Do not bypass mandatory authorization; high-risk transfers may need to fail closed until validation is available.

**Follow-up Q: Can I log the UPI PIN for debugging?**

**Answer:** No. Log opaque correlation IDs, statuses and sanitized diagnostics; never raw PIN/authentication secrets.

---

# Step 7 — Persist Payment and Transactional Outbox Atomically

**Why PostgreSQL?** The schema is structured, but the more compelling reason is ACID commits, strong constraints, idempotency, transactional state and durable audit.

Keep **two tables in one database** initially, not two independently committed databases.

~~~sql
BEGIN;

INSERT INTO payments (
    payment_id, payer_id, payee_vpa, amount_minor,
    currency, status, idempotency_key
) VALUES (
    'TXN1001', 'alice', 'bob@bank', 100000,
    'INR', 'CREATED', 'pay-12345'
);

INSERT INTO payment_outbox (
    event_id, payment_id, event_type, status
) VALUES (
    'EVT1001', 'TXN1001', 'DISPATCH_PAYMENT', 'PENDING'
);

COMMIT;
~~~

Also store request fingerprint, attempts, timestamps, original participant/network references and append-only audit evidence as appropriate. Establish a UNIQUE constraint for idempotency within its scope.

**Atomicity:** Either the payment row **and** its outbox entry commit, or neither commits.

**Follow-up Q: What if the service crashes after DB COMMIT but before returning HTTP 202?**

**Answer:** The client reuses the same idempotency key and retrieves the existing payment ID; no second logical payment is created.

**Follow-up Q: What does outbox guarantee?**

**Answer:** A committed payment intent will leave a durable dispatch instruction. It does **not** guarantee exactly-once execution by a third-party bank.

**Follow-up Q: What if the payment DB crashes before COMMIT?**

**Answer:** Do not claim acceptance. Restore DB service, then retry with the same idempotency key; never issue an external bank transfer for an uncommitted intent.

---

# Step 8 — Outbox Dispatcher, PSP Bank and NPCI

### End-to-end transfer sequence (conceptual)

~~~mermaid
sequenceDiagram
  autonumber
  participant Alice as Alice App
  participant API as Payment API
  participant DB as PostgreSQL
  participant Worker as Dispatcher
  participant PSP as PSP Bank
  participant NPCI as NPCI UPI
  participant Payer as Alice Bank
  participant Payee as Bob Bank
  Alice->>API: Send INR 1,000
  API->>DB: Atomic payment + outbox commit
  DB-->>API: Committed
  API-->>Alice: payment_id; PROCESSING
  Worker->>DB: Claim durable dispatch intent
  Worker->>PSP: Authorized instruction + original refs
  PSP->>NPCI: Route payment
  NPCI->>Payer: Payment authentication / debit
  Payer-->>NPCI: Debit result
  NPCI->>Payee: Beneficiary credit
  Payee-->>NPCI: Credit result
  NPCI-->>PSP: Transaction result
  PSP-->>API: Trusted result / status
  API->>DB: Persist verified state
~~~

The message sequence is illustrative; PIN authorization and bank settlement have participant-specific details.

### What the worker must do

1. Read the committed dispatch intent and persist an attempt/correlation record.
2. Call the configured PSP integration using stable original transaction references.
3. On confirmed result, persist legal state transitions.
4. On **uncertain timeout**, do not blindly re-send the instruction.
5. Enquire via supported status APIs; reconcile before another financial action.

**Follow-up Q: If the dispatcher pod crashes immediately after sending the bank request?**

**Answer:** The bank may have acted. On recovery, fetch attempt state and enquire on the **same transaction reference**, rather than sending a fresh payment.

**Follow-up Q: Do we need Kafka in the dispatch path?**

**Answer:** Not necessarily. DB outbox -> dispatcher -> PSP is simpler to explain. If Kafka is introduced for durable payment commands, a consumer calls the PSP; deduplication and status reconciliation remain essential.

---

# Step 9 — Payment State Machine, Timeouts and Callbacks

~~~mermaid
flowchart TD
  A["CREATED"] --> B["AUTHORIZING"]
  B -->|Approved| C["PROCESSING"]
  B -->|Rejected| F["FAILED"]
  C -->|Credit confirmed| S["SUCCESS"]
  C -->|Definitive failure| F
  C -->|Outcome unknown| P["PENDING"]
  P -->|Credit confirmed| S
  P -->|Verified failure| F
  P -->|Reversal required| RP["REVERSAL_PENDING"]
  RP -->|Confirmed| R["REVERSED"]
~~~

**Rule:** A network timeout means **UNKNOWN / PENDING**, not FAILED. Treat SUCCESS as based on trusted participant confirmation; local DB success does not mean money has arrived.

**Callback handler:** Authenticate/verify sender and reference; compare amount and identities; enforce valid state transitions, optimistic concurrency/versioning and append-only audit. Duplicate or late callbacks cannot regress SUCCESS to PROCESSING. Contradictory authoritative events require investigation.

**Follow-up Q: How does the user know the outcome?**

**Answer:** App can poll GET /v1/payments/{id}, with optional push. Push notification is never the sole source of truth.

**Follow-up Q: What if a SUCCESS callback arrives twice?**

**Answer:** Idempotent update produces one stable SUCCESS state and at most one logical status event.

---

# Step 10 — Kafka, Outbox Events and Notifications

~~~mermaid
flowchart LR
  DB[("Committed Payment Status + Event Outbox")] --> Pub["Outbox Publisher"]
  Pub --> Kafka["Kafka"]
  Kafka --> N["Notification Consumer"]
  Kafka --> A["Audit Consumer"]
  Kafka --> M["Analytics Consumer"]
~~~

- Partition by payment_id where per-payment event order matters.
- Use replicated Kafka brokers across AZs, idempotent consumers, controlled offset commits, backlog visibility and DLQ/quarantine for poison records.
- Offset is committed after durable processing.
- On Kafka outage, retain events in PostgreSQL outbox and publish after recovery; essential payment state must not depend on a push notification being sent.

**Follow-up Q: Does Kafka's exactly-once processing prevent an external duplicate bank debit?**

**Answer:** No. Kafka's semantics do not turn a separate bank transfer into an exactly-once atomic transaction.

**Follow-up Q: What if Kafka consumers crash before committing offsets?**

**Answer:** Replayed events must be deduplicated by event_id or processed idempotently.

---

# Step 11 — Reconciliation, Reversal and Correctness

**Interviewer scenario:** Alice's bank debited INR 1,000 but Bob has not received it; our Payment Service timed out.

~~~mermaid
flowchart TD
  A["Bank timeout / inconsistent status"] --> B["Keep durable PENDING"]
  B --> C["Query original transaction via PSP"]
  C --> D{"Authoritative evidence?"}
  D -->|Debit and credit confirmed| E["SUCCESS"]
  D -->|No debit and failure confirmed| F["FAILED"]
  D -->|Debit; credit unknown| G["Keep PENDING and enquire later"]
  D -->|Debit; credit failed| H["Approved reversal process"]
  H --> I["Verify reversal and mark REVERSED"]
  G --> J["Reconciliation queue and escalation"]
  J --> C
~~~

### Recovery procedure

1. Detect aged PENDING records and participant mismatches.
2. Query original transaction status using stable UPI/bank references.
3. Compare any available payer debit and beneficiary credit confirmations.
4. If both confirmed, mark SUCCESS.
5. If no debit and definitive failure, mark FAILED.
6. If debit confirmed but credit uncertain, keep pending, check again and escalate.
7. If reversal required, use approved PSP/bank/UPI reversal process and record verified evidence.
8. Publish the final status and notify the user.

**Do not use a PostgreSQL ROLLBACK to "undo" a bank debit**: those are different systems.

**Follow-up Q: If 100,000 transactions become PENDING during a PSP outage, what do you do?**

**Answer:** Persist the backlog; group by PSP and age; run bounded, rate-limited status enquiries with jitter and concurrency caps; prioritize oldest/high-risk cases; respect partner quotas; store progress checkpoints and emit discrepancy alerts. No blind replay of debits. Escalate unresolved cases through the approved process.

**Follow-up Q: Can a client manually retry the same payment?**

**Answer:** A status check is always preferable while outcome is uncertain. A new financial payment must be permitted only when the original is conclusively resolved according to product/participant rules, and the user explicitly starts another intended payment.

---

# Step 12 — Database Scaling, Redis and Backpressure

### Scaling and persistence

| Lever | Correct usage |
|---|---|
| HPA | Increase stateless gateway/payment replicas as traffic increases |
| Node autoscaler | Add EKS nodes when scheduling capacity is insufficient |
| Connection pooling | Cap app-to-PostgreSQL connections and reduce connection storms |
| Query/index optimization | Index payment_id, scoped idempotency_key, references and relevant status/age fields |
| Table partitioning | Partition historical payments/events by time or access pattern |
| Read replicas | History and non-authoritative reads; not for decisions about fresh external dispatch |
| Sharding | When necessary, shard using stable ownership/routing and cross-shard consistency design |
| Redis | Cache non-authoritative reference data; accelerate non-critical paths and rate limiting |
| Queue / backpressure | Bound work and partner requests; a queue does not replace durable DB acceptance |
| Rate limiting / circuit breaker | Prevent a failed PSP/cache from cascading into DB/service failure |

**Follow-up Q: If Redis goes down, can all traffic simply fall back to PostgreSQL?**

**Answer:** Only bounded, prioritized traffic. Otherwise a cache failure becomes a DB outage. Use timeouts, jitter, bulkheads, circuit breakers, traffic shaping and replica/managed Redis failover.

**Follow-up Q: Can we use a Redis LRU policy to pre-warm or increase ALB capacity?**

**Answer:** No. LRU is for eviction from caches. ALB has different scaling/capacity mechanisms.

---

# Step 13 — Multi-AZ High Availability

~~~mermaid
flowchart TD
  LB["Regional ALB"] --> AZ1["AZ-1: Gateway + Payment Pods"]
  LB --> AZ2["AZ-2: Gateway + Payment Pods"]
  LB --> AZ3["AZ-3: Gateway + Payment Pods"]
  AZ1 --> DB[("HA PostgreSQL Cluster")]
  AZ2 --> DB
  AZ3 --> DB
  DB --> Standby["Synchronous Standby / Quorum Replica"]
~~~

### Failure: One AZ dies

1. ALB detects unhealthy endpoints and stops routing to them.
2. Surviving AZs serve traffic and have enough **pre-provisioned headroom**.
3. If DB primary is lost, only a safely synchronized, fenced standby may be promoted.
4. Surviving services reconnect to the promoted primary.
5. Controllers schedule replacement pods; node autoscaler may supply missing worker capacity.
6. Recover incomplete **durable** payment attempts and reconcile uncertain financial calls.

**Capacity correction from mock:** For three equally loaded AZs, surviving two AZs need 100% of normal traffic capacity; simply having 30% aggregate spare headroom is **not** sufficient for one-AZ failure without immediate reliable scale-up. Roughly 50% additional total capacity above normal aggregate load is a straightforward full-capacity planning case, subject to traffic/load-test details.

**Follow-up Q: Does HPA restore an entire AZ?**

**Answer:** No. HPA increases requested pod replicas; scheduling and node infrastructure are separate. Existing surviving capacity must carry critical traffic during node provisioning.

---

# Step 14 — Regional Disaster Recovery: RPO, RTO, Fencing

~~~mermaid
flowchart TD
  Users["Mobile Clients"] --> Route["Global Routing"]
  Route --> Primary["Region A: Active Payment Writer + EKS"]
  Route -. "Failover after validation" .-> Secondary["Region B: Warm Standby"]
  Primary --> PDB[("Primary Payment DB / Journal")]
  PDB -->|Protected cross-region state| RDB[("Standby DB / Journal")]
  Secondary --> RDB
~~~

### Runbook for complete primary-region loss

1. Confirm the failure with independent health and dependency checks.
2. Fence old payment writer and dispatcher: prevent **two** regions from executing the same instruction.
3. Determine the last cross-region durable **acknowledged** payment intent.
4. Promote only a safe, consistent secondary database/journal.
5. Bring up payment API, integration, workers and secrets in DR.
6. Validate PSP connectivity and permitted partner routing.
7. Recover the outbox and original payment attempts; reconcile uncertain external bank effects.
8. Cut over global routing after readiness checks.
9. Validate bank success metrics and pending backlog before declaring recovery.
10. Restore Region A later as a standby, not a concurrent writer.

### Crucial RPO nuance

- **Synchronous cross-region protection of acknowledged payment instructions is required to target RPO 0 for those instructions**. Asynchronous replication does *not* guarantee zero regional loss.
- For any financial effects that occurred but were not yet fully acknowledged/replicated, recover by original bank reference and external reconciliation.
- Claiming universal "RPO 0" across all parties and independent bank ledgers would be incorrect.

**Follow-up Q: What if the primary database committed but its HTTP response was lost, then we failed over?**

**Answer:** The request is repeated with the **same** scoped idempotency key. The promoted durable state or journal establishes whether it exists. If external execution might have happened, enquire and reconcile before issuing another call. The old region stays fenced.

**Follow-up Q: Why start with active/warm-standby instead of active-active?**

**Answer:** Single-writer ownership reduces duplicate execution and split-brain risk. Active-active needs strict per-payment ownership partitioning, fencing and a more complex global consistency model.

---

# Step 15 — Security, Observability and SRE Incident Response

## Security checklist

- TLS, authenticated workload-to-workload channels, least-privilege RBAC and secrets management.
- Approved device/account authentication and UPI authorization.
- Do not store plaintext UPI PIN or sensitive authentication secrets.
- Encrypt database and journal at rest; protect keys with managed KMS/HSM.
- Redact bank and customer-sensitive values from logs and Kafka.
- Validate callbacks and preserve immutable/auditable payment reference changes.
- Mandatory fraud controls fail safely rather than silently being bypassed.

## Monitoring table

| SLI / Signal | Why it matters |
|---|---|
| Confirmed payment success by bank/PSP | Most important customer outcome; not merely HTTP 200 rate |
| Payment acceptance errors | App-controlled availability |
| p95/p99 of API, acceptance and final confirmation separately | Avoid conflating client latency with bank delays |
| Pending payment count and age | Detect unresolved financial effects |
| Reconciliation queue depth/age | Detect accumulating unresolved transactions |
| Duplicate external instructions | Potential monetary harm |
| PSP/bank timeout and decline rates | Localize external outages |
| Outbox age and publication lag | Detect loss of progress |
| Kafka lag and partitions health | Asynchronous workflow reliability |
| DB replication lag, quorum and connection exhaustion | Data recovery/capacity risk |
| Fraud-risk/authorization errors | Security and conversion impact |
| AZ/region availability | Infrastructure reliability |

## Sample outage runbook

**Incident:** Confirmed payment success collapses from normal to a low value.

1. Determine actual customer impact; compare PSP, bank, region, deployment and error category.
2. Check API/ALB health, error latency, DB saturation, outbox age and PSP timeouts.
3. Roll back our bad deployment if correlated and verified.
4. Apply controlled throttling/circuit breaking to an unhealthy partner; do not retry blindly.
5. Maintain durable pending transactions and reconciliation progress.
6. Restore payment capacity safely and confirm external end-to-end outcomes.
7. Reconcile inconsistent payments; notify impacted users appropriately.
8. Capture prevention, alerts, owner and follow-up remediation.

**Follow-up Q: Would you restart every Payment Service pod when the bank is down?**

**Answer:** No. It won't restore an external bank and may create connection storms. Isolate the failing dependency and protect the remainder of the system.

---

# Step 16 — Interview Mock Answer: What Was Correct and What to Fix

## Strengths in the candidate's answer

- Started with the **critical user journey**: find recipient by UPI ID or QR and send/receive.
- Defined reliability, scalability and high availability.
- Selected REST-style APIs for request/response from the app.
- Selected **PostgreSQL** for structured, transactional payment data.
- Correctly introduced a **payment table + transactional outbox** and atomic commit.
- Considered **ALB/NLB, EKS, worker pods, HPA, database indexing, sharding and replicas**.
- Mentioned **warm DR, region failover, monitoring and payment timeout handling**.
- Recognized the importance of **idempotency** and avoiding unsafe payment retries.

## Review and corrections (interview revision table)

| Candidate statement / approach | Assessment | Correct interview statement |
|---|---|---|
| "Hundreds of users transacting daily" | Improve sizing | Assume millions/day; calculate average + 10x peak + headroom |
| "p99 150–200 ms" | Clarify SLO scope | Good target for lightweight API; measure bank completion independently |
| "Structured data, so SQL" | Valid but incomplete | State ACID, durable uniqueness, idempotency and consistent state transitions |
| "Two databases: transactions and outbox" | Correct terminology | Start with two tables in **one DB transaction** |
| "ALB/NLB to worker nodes" | Mostly right | ALB for REST; can route to pod IPs with IP target mode |
| "Kafka takes message and talks to NPCI" | Correct architecture | PSP Integration Service calls participant APIs; Kafka is an internal event/optional command transport |
| "LRU to extend load balancer capacity" | Incorrect | LRU applies to caches, not ALB capacity |
| "Read replicas in same AZ and other region" | Needs correctness | Read replicas serve suitable reads; dispatch decisions need authoritative, fresh state |
| "DR secondary promoted" | Good but missing safeguards | Check replication/quorum, **fence** old writer, then promote and reconcile |
| "RPO as minimal as possible" | Too vague | Specify target 0 for synchronously protected acknowledged intent and tested RTO |
| "On timeout, tell user and let user retry" | Important nuance | Show **PENDING** and enquire/reconcile; never encourage blind duplicate payment |

**Informal evaluation from the mock:** 7.5/10 — good architectural foundation, improve payment-specific financial correctness and precise RPO/RTO. This is a coaching estimate, not an interview result.

---

# Step 17 — Interview-ready RPO and RTO Objectives

**These are illustrative targets, not guarantees or provider-published numbers.**

| Layer | RPO | Target RTO | Important condition |
|---|---|---|---|
| Stateless EKS API / Payment pods | N/A for owned state | 1–2 min | Pods distributed across AZs |
| PostgreSQL across local AZs | 0 for quorum-acknowledged writes | 1–2 min | Sync commits, safe failover/fencing |
| Redis disposable cache | N/A | 2–5 min | Bounded fallback; never authoritative funds state |
| Kafka internal events | 0 recoverable committed events | ~5 min | Durable DB outbox can replay |
| Regional app recovery | **0 for acknowledged payment intents** | 5–15 min | Requires cross-region synchronous protection + tested cutover |
| NPCI / PSP / Payer and Payee Banks | Provider-controlled | Provider-controlled | Their own recovery targets apply |
| Financial reconciliation / reversal | N/A as infrastructure RTO | Participant/rule dependent | Not interchangeable with app RTO |

**Important:** A regional failover RTO of 5–15 minutes is an aspirational **tested objective**; a safe cross-region RPO-zero design can increase latency and reduce availability during a partition. RPO/RTO must be agreed after cost and architecture trade-offs.

---

# Step 18 — End-to-End Component, Failure, Fallback, Remediation, RPO/RTO Table

_N/A = no authoritative durable business state at that component. Time targets below are interview proposals only. Externally owned RPO/RTO cannot be guaranteed by our application._

| # | Component | Failure | Fallback / remediation | Data RPO | Target RTO |
|---:|---|---|---|---|---|
| 1 | Mobile App | App/network crash | Reconnect, query original payment ID; no blind resend | 0 server-accepted intents | Device/network-dependent |
| 2 | DNS | Lookup fails | Redundant DNS, routing fallback | N/A | <= 5 min |
| 3 | WAF | Valid requests blocked | Restore safe rule, redundant WAF controls | N/A | <= 5 min |
| 4 | ALB | LB/endpoint failure | Managed HA LB and health-based routing | N/A | <= 1 min |
| 5 | API Gateway | Crash/overload | Other replicas, rate limits, HPA | N/A | <= 1 min |
| 6 | Authentication | Service unavailable | Redundant auth, fail closed for unverified users | 0 committed identity changes | <= 2 min |
| 7 | Device Binding | Device not verified | Approved account/device recovery | 0 committed binding state | <= 5 min |
| 8 | Fraud Checks | Fraud service outage | Approved policy; fail closed for high risk | 0 durable risk decisions | <= 5 min |
| 9 | Payment Service | Pod crash / bad release | Healthy pod, rollback; DB recovery | N/A, durable state in DB | <= 1 min |
| 10 | PostgreSQL Primary | Crash | Synchronous standby promotion with fencing | 0 quorum commits | <= 2 min |
| 11 | PostgreSQL Replicas | Lag/unavailable | Other replica; primary for authoritative reads | 0 quorum commits | <= 5 min |
| 12 | Connection Pool | Saturation | Bounded pool, timeout, reduce excessive clients | N/A | <= 2 min |
| 13 | Redis | Cache lost | Replica promotion / bounded DB fallback | N/A disposable cache | <= 2 min |
| 14 | Idempotency | Redis key lost | Persistent uniqueness and request fingerprint in DB | 0 committed records | <= 2 min |
| 15 | Transactional Outbox | Publisher down | Replay committed entries | 0 committed outbox rows | <= 5 min |
| 16 | Payment Dispatcher | Crash after request | Check original PSP references before retry | 0 durable attempts | <= 5 min |
| 17 | PSP Integration | API timeout | Status enquiry; authorized alternate when supported | 0 local intent | External-dependent |
| 18 | NPCI UPI Network | Unavailable | Pending and enquiry; no unsafe retries | External-controlled | External-controlled |
| 19 | Payer Bank | Debit unavailable/unknown | Check authoritative debit outcome; reconcile | Bank-controlled | Bank-controlled |
| 20 | Beneficiary Bank | Credit fails/unknown | Status enquiry / approved reversal workflow | Bank-controlled | Bank-controlled |
| 21 | Callback Handler | Callback missing/duplicate | Authenticated status enquiry, idempotent state machine | 0 committed states | <= 5 min |
| 22 | Kafka Brokers | Broker/AZ outage | Replication; replay events from outbox | 0 recoverable outbox events | <= 5 min |
| 23 | Kafka Consumers | Lag/crash | Rebalance/replay, dedup by event ID | 0 durable source events | <= 5 min |
| 24 | Notifications | Push/SMS fails | Queue retry; user queries payment status | N/A for financial records | <= 15 min |
| 25 | Payment History API | Read failure | Replica/failover; bounded authoritative fallback | 0 committed history | <= 5 min |
| 26 | Reconciliation Service | Worker offline | Durable backlog, restart and throttle enquiries | 0 persisted jobs | <= 15 min |
| 27 | Reversal Workflow | Reversal status unknown | Participant enquiry, escalation, audit | 0 durable cases | External-dependent |
| 28 | KMS/HSM | Key system down | Permitted short key cache, redundancy; fail closed | 0 committed key material | <= 15 min |
| 29 | Secrets Manager | Secret lookup unavailable | HA secret store, controlled short-lived credentials | 0 committed secrets | <= 15 min |
| 30 | Monitoring | Telemetry loss | Buffer metrics/logs, independent alert path | Best-effort telemetry | <= 15 min |
| 31 | EKS Worker Node | Node dies | Reschedule pods, replenish capacity | N/A | <= 5 min |
| 32 | Availability Zone | Total AZ failure | Surviving AZs + DB failover + reserved headroom | 0 sync quorum commits | <= 2 min |
| 33 | Primary Region | Regional disaster | Fence, activate standby, restore protected journal, reconcile | 0 *only if* cross-region sync | 5–15 min |
| 34 | Audit Journal | Journal write fails | Required operations fail safe; recover protected replica | 0 critical commits | <= 5 min |

**Recovery validation:** Check DB write health, application error rate, verified partner payment success, oldest pending age, reconciliation queue, duplicate external instructions, original idempotency behavior and PSP response consistency. Green Kubernetes pods are **not** proof of successful financial recovery.

---

# Step 19 — Apple Pay Comparison: What Changes?

**Apple Pay card payment (typical merchant purchase) is not a UPI transfer.** A device-protected payment token/cryptogram is sent via the merchant/payment processor and card network to the issuer for card authorization; capture and settlement are separate downstream steps.

~~~mermaid
flowchart LR
  Phone["iPhone / Apple Watch"] --> Token["Apple Pay Protected Payment Data"]
  Token --> Merchant["Merchant / Payment Processor"]
  Merchant --> Network["Card Network"]
  Network --> Issuer["Card Issuing Bank"]
  Issuer --> Decision["Approved / Declined"]
~~~

| Google Pay-like UPI app | Apple Pay card payment |
|---|---|
| Identify payee by UPI ID/QR | Select tokenized card at merchant |
| UPI authorization flow and PIN where required | Device/cardholder authentication and token cryptogram |
| PSP Bank / NPCI / participant banks | Merchant PSP/acquirer, card network, issuer |
| Transfer status, enquiry and UPI reversal | Card authorization, capture, refund, dispute/chargeback |
| Direct account-to-account payment semantics | Card-purchase authorization and eventual clearing/settlement |

**Shared architectural lessons:** trusted transaction identity, durable records, idempotency, uncertain-outcome handling, partner isolation, reconciliation, rate limits and tested DR.

---

# Step 20 — Final Follow-up Q&A (Rapid Revision)

| Interviewer question | Answer |
|---|---|
| What is the most important flow? | Payee discovery -> secure authorization -> durable payment intent -> PSP/UPI execution -> verified final state |
| Why relational DB? | ACID, constraints, payment-state consistency and auditability |
| Why transactional outbox? | Atomic durable intent and recovery of dispatch/event work |
| Can outbox or Kafka give exactly-once bank debit? | No. The external protocol needs original references, permitted dedup/status enquiry and reconciliation |
| Why two tables instead of two databases? | Payment and outbox must share a single DB commit |
| What if client retries after response loss? | Scoped idempotency key returns existing payment ID |
| What if bank debit happened but response was lost? | PENDING -> enquire using original ref -> reconcile; do not re-debit |
| Is HTTP 200 proof funds moved? | No; only trusted participant outcome is adequate for confirmed payment status |
| Why use Kafka? | Internal asynchronous events, notification and analytics decoupling |
| What if Kafka is unavailable? | Persist outbox; continue critical transactions if financial dependencies are healthy; replay later |
| Why not store money in Redis? | It is not the authoritative durable bank ledger or payment record |
| What if Redis fails? | Bounded fallback with breaker, backoff/jitter, rate limits and DB protection |
| Does HPA heal an AZ? | No; surviving capacity and node/cluster rescheduling are also required |
| What is the region-failover danger? | Split-brain writers and reissuing an already-executed payment |
| What if async DR replication is used? | Cannot guarantee RPO 0 for regional loss |
| What is the SRE's core payment metric? | Confirmed end-user payment outcome plus aged pending and reconciliation gaps |
| How do you handle 100k pending transfers? | Controlled rate-limited status enquiries and persisted reconciliation, not a debit replay |
| How does Apple Pay differ? | Card token, processor/acquirer, card network, issuer; not the UPI rail |

---

# Step 21 — Interview-ready 90-second Answer

> I would design the UPI payment service using REST APIs, an application gateway and stateless microservices running on EKS across multiple Availability Zones.
>
> When a user initiates payment, the Payment Service validates the request and establishes an idempotency key. It commits the payment instruction and transactional outbox event atomically into PostgreSQL.
>
> A dispatcher then communicates through the PSP Bank integration with the UPI network, which handles financial processing with the participating banks.
>
> I would use Kafka for asynchronous transaction events and notifications, Redis for non-critical caching, and PostgreSQL for durable payment state.
>
> For scaling, I would use HPA, node autoscaling, database indexing, connection pooling and partitioning. For high availability, I would deploy across three AZs and maintain a regional DR environment with protected payment records.
>
> Most importantly, if an external payment times out, I would never blindly retry the debit. I would retain the transaction as pending, check its authoritative status and reconcile the result.
>
> My recovery objectives would prioritize zero loss of acknowledged financial instructions, while targeting fast infrastructure restoration and defining separate recovery expectations for external banking failures.

## Most important conclusion

> A reliable payment system is not just an available API. It must **not lose acknowledged payment intent, must not cause unintended duplicate financial effects, and must reconcile the real bank outcome even after ambiguous failures.**

---

## Interview revision checklist

- [x] Functional/non-functional requirements and capacity math
- [x] REST contracts and idempotency
- [x] AWS ALB, EKS, Kubernetes scaling
- [x] ACID PostgreSQL transaction and outbox
- [x] PSP, NPCI and banks' distinct responsibilities
- [x] Payment state machine and callback safety
- [x] Kafka, Redis and async consumers
- [x] Payment reconciliation, reversal and unknown outcome
- [x] DB, AZ and regional DR with fencing and RPO caveats
- [x] Component-level failures, fallbacks, remediation, RPO/RTO
- [x] Original mock-answer corrections, rapid Q&A and 90-second pitch
- [x] Apple Pay card-rail comparison
