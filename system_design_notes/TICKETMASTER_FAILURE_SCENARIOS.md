For the **Ticketmaster system design**, here are the failure scenarios for every component in your diagram, including your proposed SQS FIFO approach.

## 1. Component-level failures and remediation

| Component | Failure scenario | Remediation |
|---|---|---|
| **Client** | Network disconnects during booking. | Retry using the same idempotency key; check booking status before retrying payment. |
| **CDN** | CDN unavailable or cached data becomes stale. | Multi-CDN or origin fallback; invalidate stale content. |
| **API Gateway** | Traffic spike, throttling or gateway failure. | Rate limiting, autoscaling where supported, multi-AZ deployment and controlled retries. |
| **Search Service** | Search pods crash or overload. | Multiple replicas, HPA, load balancing and timeouts. |
| **Elasticsearch** | Search nodes fail or indexes become unavailable. | Replica shards, multi-AZ nodes; optionally fall back to limited DB search. |
| **Event CRUD Service** | Service pods crash. | HPA, multiple replicas, readiness probes and retries for safe operations. |
| **Redis Event Cache** | Redis unavailable or cache data expires. | Multi-AZ Redis failover; query PostgreSQL with bounded fallback traffic and repopulate cache. |
| **PostgreSQL** | Primary database crashes or connections are exhausted. | Multi-AZ automatic failover, connection pooling, timeouts and retries. |
| **CDC** | CDC connector fails; search index stops updating. | Resume from durable log position, replay changes, monitor lag and rebuild indexes if needed. |
| **Virtual Waiting Queue (Redis)** | Redis fails and waiting positions are lost. | Redis replicas, persistence, recovery checkpoints; temporarily pause admissions. |
| **Admission Controller** | Controller crashes or admits too many users. | Multiple instances, atomic admission counters and strict concurrency limits. |
| **Booking Service** | Pod crashes during reservation or payment. | Retry/reconcile using booking ID; idempotent operations and database transaction recovery. |
| **Redis Ticket Lock** | Redis fails or lock expires during payment. | Redis HA, DB concurrency checks, reservation ownership validation; extend TTL safely when needed. |
| **Stripe** | Payment times out or webhook is delayed. | Idempotency keys, webhook retries, payment-status reconciliation; refund when booking cannot complete. |
| **SQS FIFO (optional)** | Booking worker fails or message is processed again. | Visibility timeout, consumer retries, DLQ and idempotent database updates. |
| **SQS Booking Worker** | Worker pods crash or processing backlog grows. | Scale workers, monitor queue age and retry failed messages. |

The Hello Interview design particularly emphasizes database concurrency protection, temporary Redis locks and admission control. <Cite refs={["turn144231view0","turn144231search2"]}/>

## 2. Three critical failure scenarios to explain

### Scenario 1: Redis ticket lock fails

\`\`\`text id="lg85kp"
User A and User B select Seat 101
              |
              v
       Redis unavailable
              |
              v
   Temporarily stop new holds
      OR use DB-backed holds
              |
              v
  PostgreSQL Atomic Transaction
              |
       Only one succeeds
              |
              v
      No double booking
\`\`\`

**Key:** PostgreSQL is the final source of truth. Redis must never be the only protection against double booking.

### Scenario 2: Payment successful but Booking Service crashes

\`\`\`text id="16559m"
Stripe: Payment Successful
             |
             v
 Booking Service crashes
             |
             v
 Webhook retry / Reconciliation
             |
             v
 Check booking/payment status
             |
             v
 Confirm ticket if valid
 OR refund the payment
\`\`\`

**Key:** Never charge twice. Use idempotency and reconciliation.

### Scenario 3: SQS worker crashes during booking

\`\`\`text id="1tftxc"
SQS FIFO
    |
    v
Worker receives message
    |
    v
Worker crashes
    |
    v
Visibility timeout expires
    |
    v
Message becomes visible again
    |
    v
Another worker retries
    |
    v
DB idempotency check
    |
    v
Process once logically
\`\`\`

If repeated processing fails, configure a **Dead Letter Queue (DLQ)** and investigate the failed reservation. <Cite ref="turn144231search0"/>

## 3. The four things to remember for your interview

| Failure | Most important solution |
|---|---|
| Service/Pod crash | Multi-AZ, replicas, HPA |
| Redis failure | Multi-AZ failover, persistence, safe fallback |
| Database failure | Multi-AZ failover, transactions, retries |
| Payment/SQS failure | Idempotency, retries, DLQ, reconciliation |

**Interview statement:** "I prioritize consistency over availability for ticket booking because preventing double booking is critical. For search and event viewing, I prioritize availability through caching, replicas and graceful degradation."