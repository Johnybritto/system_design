# Gmail / Outlook — Email Delivery System Design

> Interview-focused notes: concise requirements, APIs, high-level design, deep dives, and database choices.

## 1. Requirements

**Functional (4)**
1. Send and receive emails (internal and external domains).
2. Support attachments.
3. List inbox/sent messages and read individual emails.
4. Search emails and mark read/unread.

**Non-functional (4)**
1. High availability (target 99.99%).
2. Durability — no *durably accepted* email is lost.
3. Scalability to millions of users and large email volume.
4. Low-latency email acceptance, with eventual delivery, retries, and idempotency.

## 2. API Contracts (short)

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/emails` | Send an email |
| `GET` | `/emails?folder=inbox` | List inbox |
| `GET` | `/emails/{id}` | Read email |
| `PATCH` | `/emails/{id}` | Mark read/unread |
| `POST` | `/attachments` | Upload an attachment |

**Send request**
```http
POST /emails
Content-Type: application/json

{
  "to": ["bob@gmail.com"],
  "subject": "Hello",
  "body": "Hi Bob",
  "attachmentIds": ["file123"]
}
```

Response: **`202 Accepted`** with `messageId` and `status: "queued"`.
It means the system has *durably accepted* the request, **not** that Bob received it. Use a client idempotency key on send retries.

## 3. High-Level Design

```text
Alice: Web/Mobile Email Client
             |
             v
   API Gateway + Authentication
             |
             v
        Email Service
           /      \
          v        v
   DynamoDB        S3
 (mailbox metadata,   (email body,
  delivery state,      attachments)
  outbox)
          |
          v
    Outbox Publisher
          |
          v
       SQS Queue
          |
          v
     Delivery Workers
       /           \
      v             v
 Local recipient    SMTP Relay
 inbox record          |
 (DynamoDB)            v
                   DNS MX lookup
                       |
                       v
                Gmail/Outlook MX
```

**High-level explanation:** The API service validates and durably accepts email. Content goes to S3; metadata and an outbox event are committed atomically to DynamoDB. A publisher sends delivery jobs to SQS. Workers route mail into our own users' mailboxes or deliver to external mail servers via SMTP.

### Exact Send Flow

1. **Client -> API Gateway -> Email Service:** Alice submits an email.
2. **Email Service -> S3:** Store body and any uploaded attachment references.
3. **Email Service -> DynamoDB:** Atomically save email/mailbox metadata and a pending outbox item.
4. **Outbox Publisher -> SQS:** Publish a job; retry safely on publish failure.
5. **Delivery Worker:** Consume job and decide recipient domain.
6. **Delivery:**
   - **Internal:** idempotently create Bob's inbox entry in DynamoDB.
   - **External:** find the destination mail exchanger (MX) and hand off using SMTP; track delivery outcome.

For multiple recipients, track delivery status **per recipient**, not only per message.

### Incoming External Email Flow

```text
External sender SMTP
       |
       v
 Our MX / Inbound SMTP Server
       |
       v
 Spam, virus and policy checks
       |
       v
 S3 message body + DynamoDB recipient mailbox record
       |
       v
 Recipient reads from inbox API
```

## 4. Database Selection — Important Interview Topic

| Store | Technology | Why |
|---|---|---|
| Mailbox metadata / status / outbox | **DynamoDB** | Fast per-user lookups, scalable writes and conditional/transactional operations |
| Message content and attachments | **S3** | Durable, cost-effective object storage for large content |
| Search index | **OpenSearch** | Full-text search across sender, subject and message body |
| Asynchronous delivery tasks | **SQS** | Managed durable queue, visibility timeout, retries and dead-letter queue (DLQ) |

**Why DynamoDB instead of RDS?** For a very large mail system, most reads are predictable: fetch a user's latest inbox messages, fetch one message, or update read state. DynamoDB scales these key-based access patterns without provisioning a relational database cluster. **Aurora/RDS is also valid** if relational joins, flexible querying or sophisticated SQL transactions are higher priorities; neither DynamoDB nor RDS replaces a dedicated full-text search index.

### Example DynamoDB Mailbox Table

| PK | SK | Folder | Sender | Read | S3 object |
|---|---|---|---|---|---|
| `USER#alice` | `MAIL#2026-10-25T10:00#msg101` | Sent | Alice | true | `messages/msg101` |
| `USER#bob` | `MAIL#2026-10-25T10:00#msg101` | Inbox | Alice | false | `messages/msg101` |
| `USER#bob` | `MAIL#2026-10-25T11:00#msg102` | Inbox | Carol | true | `messages/msg102` |

- **PK (partition key):** `USER#<user-id>` — group each user's mailbox records.
- **SK (sort key):** `MAIL#<timestamp>#<message-id>` — retrieve and paginate by time.
- **GSI:** User + folder + timestamp for efficient Inbox / Sent listing without scanning.
- **Body:** Stored once in S3 and referenced from mailbox entries.
- **Outbox:** Separate DynamoDB items/table holding delivery jobs, status and retry metadata.
- **Idempotency:** Message ID + recipient ID uniquely identifies each recipient mailbox write.

**Design nuance:** The example PK works as an interview baseline; extremely active users or large tenants can create hot partitions, requiring bucketing/partitioning. DynamoDB item limits are another reason not to store large email bodies or attachments inline.

## 5. Deep Dive (Short)

### A. Reliable delivery — transactional outbox

```text
Email Service
    |
    v
DynamoDB transaction: message metadata + outbox event
    |
    v
Outbox Publisher -> SQS -> Delivery Worker
                           |        |
                        success   failure
                           |        |
                     Mark state   retry/backoff
                                   |
                                DLQ / bounce
```

**Why outbox?** Without it, the DB write could succeed but the app could crash before publishing to SQS. The outbox preserves the delivery intent, allowing safe retries.

**Why idempotency?** SQS provides **at-least-once** processing, so jobs can be delivered more than once. Conditional mailbox writes prevent duplicate local inbox entries. External SMTP cannot guarantee exactly-once delivery after ambiguous remote acceptance.

### B. SMTP and external delivery

```text
Delivery Worker -> SMTP Relay -> DNS MX -> Gmail/Outlook receiving MX
```

- **MX record:** DNS tells us which mail server receives the destination domain's mail.
- **SMTP:** Server-to-server email delivery protocol.
- **SPF/DKIM/DMARC:** Sender authorization, signatures and policy checks.
- **SMTP 4xx:** Usually temporary; retry with backoff and a delivery deadline.
- **SMTP 5xx:** Usually permanent; record a failure/bounce.
- A successful SMTP handoff means a remote mail server accepted responsibility; it does **not** guarantee placement in the user's Inbox.

### C. Scaling and search

| Concern | Approach |
|---|---|
| Email traffic spike | SQS absorbs burst; autoscale worker pods |
| Large attachments | Presigned S3 upload URLs; avoid streaming large files through API pods |
| Inbox pagination | DynamoDB Query + keyset pagination |
| Full-text search | Asynchronously index searchable fields/body into OpenSearch |
| Search index outage | Sending/receiving continues; catch up indexing later |

### D. High availability and regional failure

| Layer | HA / DR |
|---|---|
| API / Email Service / Workers | Multi-AZ deployment, health checks, autoscaling |
| DynamoDB | Regional multi-AZ managed resilience; optional Global Tables for cross-region replication |
| SQS | Regional managed service with multi-AZ resilience; design queue recovery in another region separately |
| S3 | Regional multi-AZ durability; optional cross-region replication |
| OpenSearch | Multi-AZ replicas; rebuildable from durable records |

**Recovery principle:** Replay unfinished delivery work after failover, deduplicate recipient writes, and reconcile message status. Cross-region replication alone does not replicate an SQS queue.

## 6. Important Interview Follow-up Q&A

| Question | Short answer |
|---|---|
| Why SQS instead of Kafka? | Email delivery is task-oriented; SQS naturally provides ack/delete, retries and DLQ. Kafka suits durable replayable streams and multiple independent readers. |
| How do you prevent email loss? | Durable acceptance + transactional outbox + queue retries + recovery/reconciliation. |
| What if Gmail is unavailable? | Retry transient SMTP failures with backoff; eventually fail/bounce. |
| What if a worker crashes? | SQS visibility timeout expires; another worker retries. |
| How do you avoid duplicates? | Idempotency key and conditional mailbox writes; SMTP external delivery is not exactly once. |
| How do you search email? | OpenSearch asynchronously indexed from durable email records. |
| Why use S3? | Store large bodies/attachments durably, without inflating metadata database items. |

## 7. 30-Second Interview Summary

> I would design the mail system using API Gateway, an Email Service, DynamoDB for mailbox metadata, S3 for email bodies and attachments, and SQS for asynchronous delivery. An atomic transactional outbox ensures accepted emails are not lost between the database and the queue. Workers create local recipients' mailbox entries or use SMTP to deliver to external domains. OpenSearch provides full-text search. Multi-AZ deployment, retries, idempotency, DLQ, and recovery/replay support reliability.

**Remember:** SMTP, DNS MX, DynamoDB PK/SK/GSI, transactional outbox, SQS retries, idempotency, search indexing, and bounce handling.
