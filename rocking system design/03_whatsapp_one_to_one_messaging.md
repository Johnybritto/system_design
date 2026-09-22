# System Design Session 03 — WhatsApp / One-to-One Messaging System

## Goal

Design the core of a WhatsApp-like messaging platform while preserving the lecture's AWS architecture and adding Azure equivalents in brackets.

> **Cloud mapping convention:** AWS components from the lecture are kept as-is. Where useful, the closest Azure service is shown in brackets. The final AWS and Azure diagrams are intentionally separated so the architecture stays readable.

The session focuses on three features:

1. **One-to-one messaging**
2. **Sent / delivered / read receipts**
3. **Last online / last seen status**

The main non-functional requirements are:

- Scalable
- Resilient / highly available
- Secure

---

# 1. Start With Requirements Before Drawing

The instructor repeats an important whiteboarding rule:

> Do not jump straight into architecture.

WhatsApp has many features, so first define the subset being designed.

## Functional scope

- Bob can send a message to Tina.
- Bob can see whether the message was sent, delivered, or read.
- Tina can see Bob's last-online / online status.

## Non-functional scope

- The system must scale to very large traffic.
- It must remain available if individual backend instances fail.
- Messages and metadata must be protected in transit and at rest.

---

# 2. Basic Message Model

At the highest level, Bob's phone sends a message to the backend.

A simplified message envelope could contain:

- senderId
- recipientId
- messageId
- timestamp
- payload

Conceptually:

~~~text
Bob
 |
 v
Backend
 |
 +--> Persist / update metadata
 |
 +--> Deliver message
       |
       v
      Tina
~~~

The backend needs enough information to determine who sent the message, who should receive it, what the message is, and its current delivery state.

---

# 3. Why Introduce a Message Queue?

The lecture first considers a synchronous request path:

~~~text
Bob
 |
 v
API
 |
 v
Backend
 |
 v
Database
~~~

If the entire request depends on every downstream component completing before the caller can proceed, the system becomes tightly coupled.

The lecture therefore introduces asynchronous messaging.

~~~text
Bob
 |
 v
API Gateway
[Azure: API Management]
 |
 v
Message Queue
AWS SQS
[Azure: Service Bus Queue]
 |
 v
Backend Workers
 |
 +--> Database
 |
 +--> Tina
~~~

## Why this helps

The queue acts as a buffer between ingress and processing.

Benefits:

- absorbs traffic spikes
- lets producers and consumers scale independently
- provides retry capability
- reduces direct coupling
- helps protect the backend from sudden bursts
- allows failed messages to be isolated in a dead-letter queue

### Interview-safe correction

The lecture says there is "no way to retry" in a synchronous design. That is too absolute.

A better interview answer is:

> Synchronous systems can retry, but retries increase coupling and can amplify load during failure. A durable queue gives the system a cleaner buffering, retry, backpressure, and recovery mechanism.

---

# 4. Message Queue — Amazon SQS [Azure Service Bus]

The lecture uses **Amazon SQS**.

Azure equivalent:

**Azure Service Bus Queue**

The queue should be:

- highly available
- durable
- horizontally scalable
- able to absorb bursts

The lecture also introduces a **Dead-Letter Queue (DLQ)**.

~~~text
Incoming Messages
       |
       v
+----------------+
| Main Queue     |
| SQS            |
| [Service Bus]  |
+-------+--------+
        |
        v
 Backend Workers
        |
   processing fails
        |
        v
+----------------+
| DLQ            |
+----------------+
~~~

A DLQ keeps repeatedly failing messages out of the normal flow so they can be investigated or replayed.

---

# 5. Ordering — Standard Queue vs FIFO

The interviewer may ask:

> What if messages must arrive in order?

The lecture highlights that a standard queue may not preserve strict ordering.

## AWS

### SQS Standard

- very high throughput
- at-least-once delivery
- best-effort ordering

### SQS FIFO

- preserves ordering within a **Message Group**
- supports deduplication semantics
- throughput characteristics differ from Standard queues

For one-to-one messaging, a good design is:

~~~text
MessageGroupId = conversationId
~~~

This preserves ordering per conversation rather than globally serializing all traffic.

## Azure

Equivalent idea:

**Azure Service Bus Sessions**

Use:

~~~text
SessionId = conversationId
~~~

to preserve ordered processing for messages in the same conversation.

### Interview point

Ask whether **strict ordering is actually required**. Do not pay the complexity or throughput cost of strict ordering unless the requirement needs it.

---

# 6. Backend Processing

The lecture gives two backend options.

## Option A — EC2 Auto Scaling Group

**AWS:** EC2 Auto Scaling Group  
**Azure:** Virtual Machine Scale Sets

~~~text
            SQS
             |
       +-----+-----+
       |     |     |
       v     v     v
      EC2   EC2   EC2
       \     |     /
        Auto Scaling
~~~

The backend can scale based on queue depth.

~~~text
SQS queue depth
      |
      v
Scaling policy
      |
      v
Add / remove workers
~~~

Azure equivalent:

~~~text
Azure Service Bus queue depth
          |
          v
Azure Monitor / Autoscale
          |
          v
VM Scale Sets
~~~

## Option B — Serverless Workers

**AWS:** Lambda  
**Azure:** Azure Functions

~~~text
Queue
 |
 v
Lambda / Azure Functions
 |
 v
Process message
~~~

This can be cost effective when traffic is bursty because compute scales with demand instead of keeping instances continuously running.

---

# 7. API Gateway vs Load Balancer

The lecture specifically calls this out as an interview probe.

## Load Balancer

A load balancer mainly distributes traffic across backend application instances.

~~~text
Client
  |
  v
Load Balancer
  |
  +--> Server 1
  +--> Server 2
  +--> Server 3
~~~

## API Gateway

An API Gateway provides API-specific capabilities such as:

- routing
- authentication
- throttling
- quotas
- request transformation
- API policies
- managed integrations

The lecture prefers **Amazon API Gateway** because it can integrate with backend AWS services and supports WebSocket APIs.

Azure equivalents:

- **Azure API Management** for HTTP/API management
- **Azure Web PubSub** for large-scale persistent real-time WebSocket connections

### Interview takeaway

Do not say that API Gateway is always better than a load balancer.

Say:

> They solve overlapping but different problems. I would use an API gateway for API management and a load balancer when I primarily need Layer-4/Layer-7 traffic distribution to backend instances.

---

# 8. Why WebSockets?

Traditional request/response APIs are client initiated.

Messaging needs the backend to push a new message to Tina even when Tina did not just initiate a request.

The lecture uses **WebSocket APIs**.

~~~text
Client                      Server
  |                            |
  |------ establish ---------->|
  |<==== persistent link =====>|
  |                            |
  |------ client message ----->|
  |<----- server message ------|
~~~

A WebSocket connection is:

- persistent
- full-duplex
- bi-directional

So the server can push data to the connected client.

## AWS

**Amazon API Gateway WebSocket API**

## Azure

**Azure Web PubSub** is the closest Azure-native service for scalable real-time WebSocket connections.

**Azure API Management** can still be used for normal HTTP APIs.

---

# 9. Connection Routing — Important Missing Detail

The lecture draws the WebSocket connection but does not deeply explain how the backend knows **which live connection belongs to Tina**.

At scale, maintain a mapping such as:

~~~text
userId -> connectionId / server / region
~~~

Example:

~~~text
tina -> connection-9812
bob  -> connection-1211
~~~

Then the delivery worker can look up Tina's active connection and push the message.

Possible implementations:

### AWS
- DynamoDB
- Redis-style distributed presence / connection store

### Azure
- Cosmos DB
- Azure Web PubSub-managed connections plus a small connection/presence registry where needed

---

# 10. Database Choice

The lecture chooses **Amazon DynamoDB** because of:

- horizontal scalability
- low-latency key-based access
- managed availability
- elastic capacity

Azure equivalent:

**Azure Cosmos DB**

Possible stored metadata:

- conversationId
- messageId
- senderId
- recipientId
- createdAt
- status

Possible message states:

~~~text
SENT
DELIVERED
READ
~~~

### Interview-safe correction

The lecture says relational databases cannot be horizontally scaled. That is too absolute.

A stronger answer is:

> Relational databases can scale horizontally, but distributed sharding and consistency may add operational complexity. For a messaging workload with huge key-based traffic and predictable access patterns, a horizontally partitioned NoSQL store such as DynamoDB or Cosmos DB can be a strong fit.

---

# 11. Sent / Delivered / Read Receipts

The session explains the familiar messaging states.

## 1. Sent

Bob's client successfully submits the message to the platform.

~~~text
Bob
 |
 v
Ingress / queue accepted
 |
 v
SENT
~~~

## 2. Delivered

The backend successfully pushes the message to Tina's connected client and receives acknowledgement.

~~~text
Backend
 |
 v
Tina
 |
 v
delivery acknowledgement
 |
 v
Status = DELIVERED
~~~

The backend then notifies Bob over Bob's WebSocket connection.

## 3. Read

Tina opens / reads the message and sends a read acknowledgement.

~~~text
Tina
 |
 v
Read Receipt API
 |
 v
Queue
 |
 v
Backend
 |
 v
DB status = READ
 |
 v
WebSocket notification
 |
 v
Bob
~~~

For WhatsApp-style UI, think conceptually:

- one check -> sent / accepted
- two checks -> delivered
- two blue checks -> read

---

# 12. Receipt Flow — End to End

~~~text
Bob
 |
 | Send message
 v
API Gateway
 |
 v
SQS
 |
 v
Worker
 |
 | update status = SENT
 |
 | push message
 v
Tina WebSocket
 |
 | ACK
 v
Worker
 |
 | update status = DELIVERED
 |
 | push receipt
 v
Bob WebSocket


Later...

Tina opens message
 |
 v
Read Receipt API
 |
 v
Queue
 |
 v
Worker
 |
 | update status = READ
 |
 v
Bob WebSocket
~~~

Azure follows the same logical sequence with:

- API Management
- Service Bus
- Azure Functions / VM Scale Sets
- Cosmos DB
- Azure Web PubSub

---

# 13. Last Online / Last Seen

The lecture implements presence using periodic heartbeats.

Bob's phone sends a health / presence signal every few seconds.

~~~text
Bob App
  |
  | heartbeat
  v
Presence API
  |
  v
Health-Check Queue
  |
  v
Backend
  |
  v
Presence Store
~~~

The backend records:

~~~text
bob.lastSeen = timestamp
~~~

When Tina asks for Bob's status:

~~~text
Current time - lastSeen <= threshold
        |
        +--> yes -> ONLINE
        |
        +--> no  -> show LAST SEEN timestamp
~~~

## AWS mapping

- Presence API -> API Gateway
- Presence queue -> SQS
- Worker -> Lambda / EC2
- Presence state -> DynamoDB

## Azure mapping

- Presence API -> API Management
- Presence queue -> Service Bus
- Worker -> Azure Functions / VM Scale Sets
- Presence state -> Cosmos DB

---

# 14. Presence — Interview-Safe Improvement

Writing a durable database record every 10 seconds for every online user can become extremely expensive at WhatsApp scale.

The lecture's heartbeat concept is valid, but in a stronger production design:

- use the existing WebSocket connection as a strong online signal
- update presence only when needed
- store hot presence state in a low-latency distributed store
- use TTL / expiry
- persist durable "last seen" information only when appropriate
- avoid unnecessarily writing every heartbeat to a durable database

Conceptually:

~~~text
WebSocket connected
      |
      v
Presence = ONLINE
      |
disconnect / timeout
      |
      v
Persist lastSeen
~~~

This reduces write amplification.

---

# 15. Scalability Design

The overall system scales because each layer can scale independently.

~~~text
Users
  |
  v
API / WebSocket Layer
  |
  v
Queue
  |
  v
Worker Fleet
  |
  v
Distributed Database
~~~

Important scaling dimensions:

- API requests
- concurrent WebSocket connections
- queue throughput
- worker count
- database partitions
- presence updates

For global scale, workloads may be partitioned by:

- region
- user ID
- conversation ID

The lecture also mentions that multiple queues may exist behind the simplified diagram.

~~~text
Region A -> Queue set A
Region B -> Queue set B
Region C -> Queue set C
~~~

---

# 16. Resilience

The design includes several failure-handling mechanisms.

## Queue durability

Messages survive temporary backend failures.

## Dead-Letter Queue

Repeatedly failing messages are isolated instead of blocking normal processing.

## Multi-instance backend

EC2 Auto Scaling / VM Scale Sets prevent a single worker from becoming a single point of failure.

## Multi-AZ deployment

Backend instances should run across multiple availability zones.

Azure equivalent:

- Availability Zones
- zone-redundant managed services where supported

## Retry + idempotency

At-least-once delivery means a message may occasionally be processed more than once.

Therefore the consumer should use a unique **messageId** as an idempotency key.

This prevents duplicate side effects.

---

# 17. Security

The lecture divides security into:

1. Data in transit
2. Data at rest

## In Transit

Use:

**TLS / HTTPS / WSS**

The transcript says SSL, but modern interview terminology should be **TLS**.

~~~text
Client
  |
 TLS / WSS
  v
API Gateway / WebSocket service
~~~

## At Rest

### AWS

Use AWS KMS-backed encryption for:

- DynamoDB
- SQS
- EBS volumes attached to EC2
- other persisted data

### Azure

Use:

- Azure Key Vault / Managed HSM
- Service Bus encryption
- Cosmos DB encryption
- encrypted managed disks

---

# 18. Critical Security Enhancement — End-to-End Encryption

The lecture covers transport and storage encryption, but a WhatsApp-like design should also discuss **End-to-End Encryption (E2EE)**.

TLS protects:

~~~text
Client <--> Server
~~~

E2EE protects:

~~~text
Bob <====================> Tina

Only endpoints can decrypt message content.
The messaging backend transports ciphertext.
~~~

For a real WhatsApp-style architecture, this is a major interview point.

A strong answer is:

> I would use TLS for transport security, KMS/Key Vault for server-side encryption of infrastructure data, and end-to-end encryption for the actual message payload.

---

# 19. AWS Component Mapping

| Capability | AWS |
|---|---|
| HTTP API ingress | Amazon API Gateway |
| Persistent WebSocket API | API Gateway WebSocket API |
| Durable message queue | Amazon SQS |
| Ordered queue | SQS FIFO |
| Dead-letter queue | SQS DLQ |
| Compute workers | EC2 Auto Scaling Group |
| Serverless workers | AWS Lambda |
| Message / metadata database | Amazon DynamoDB |
| Encryption key management | AWS KMS |
| VM disk encryption | EBS encryption |
| Autoscaling signal | CloudWatch metrics / SQS queue depth |

---

# 20. Azure Component Mapping

| Capability | Azure |
|---|---|
| HTTP API ingress | Azure API Management |
| Persistent real-time WebSocket messaging | Azure Web PubSub |
| Durable message queue | Azure Service Bus Queue |
| Ordered per-conversation processing | Service Bus Sessions |
| Dead-letter queue | Service Bus DLQ |
| Compute workers | Azure Virtual Machine Scale Sets |
| Serverless workers | Azure Functions |
| Message / metadata database | Azure Cosmos DB |
| Encryption key management | Azure Key Vault / Managed HSM |
| VM disk encryption | Azure managed disk encryption |
| Autoscaling signal | Azure Monitor / Autoscale / queue metrics |

---

# 21. Final AWS Architecture

~~~text
                         +----------------------+
                         |   Amazon DynamoDB    |
                         | messages / receipts  |
                         | presence / metadata  |
                         +----------+-----------+
                                    ^
                                    |
                                    |
+----------+                 +------+-------+                 +----------+
|          |   HTTPS / WSS   |              |    WSS Push     |          |
|   Bob    +---------------->| API Gateway  |<--------------->|   Tina   |
|  Mobile  |                 | WebSocket API |                 |  Mobile  |
|          |                 +------+-------+                 |          |
+----------+                        |                         +----------+
                                    |
                                    | service integration
                                    v
                           +------------------+
                           |    Amazon SQS    |
                           |  Message Queue   |
                           +--------+---------+
                                    |
                          +---------+---------+
                          |                   |
                          v                   v
                   +-------------+      +-------------+
                   | EC2 Worker  | ...  | EC2 Worker  |
                   | ASG Multi-AZ|      | ASG Multi-AZ|
                   +------+------+      +------+------+
                          |                    |
                          +---------+----------+
                                    |
                                    v
                               DynamoDB

                           Failed processing
                                    |
                                    v
                           +------------------+
                           |     SQS DLQ      |
                           +------------------+


Presence / Last Seen:

Bob Mobile
    |
    | heartbeat
    v
API Gateway
    |
    v
Presence SQS
    |
    v
Worker
    |
    v
DynamoDB presence record
~~~

---

# 22. Final Azure Architecture

~~~text
                         +----------------------+
                         |   Azure Cosmos DB    |
                         | messages / receipts  |
                         | presence / metadata  |
                         +----------+-----------+
                                    ^
                                    |
                                    |
+----------+                  +-----+------+                  +----------+
|          |    HTTPS         |            |                  |          |
|   Bob    +----------------->| API Mgmt   |                  |   Tina   |
|  Mobile  |                  |            |                  |  Mobile  |
|          |                  +-----+------+                  |          |
+----+-----+                        |                         +-----+----+
     |                              |                               |
     | WebSocket                    |                               | WebSocket
     |                              v                               |
     |                      +---------------+                       |
     +--------------------->| Azure Web     |<----------------------+
                            | PubSub        |
                            +-------+-------+
                                    |
                                    v
                           +------------------+
                           | Azure Service Bus|
                           | Message Queue    |
                           +--------+---------+
                                    |
                         +----------+----------+
                         |                     |
                         v                     v
                +----------------+     +----------------+
                | Azure Function | ... | VM Scale Set   |
                | Worker         |     | Worker         |
                +-------+--------+     +-------+--------+
                        |                      |
                        +----------+-----------+
                                   |
                                   v
                              Cosmos DB

                           Failed processing
                                   |
                                   v
                           +------------------+
                           | Service Bus DLQ  |
                           +------------------+


Presence / Last Seen:

Bob Mobile
    |
    | heartbeat / connection signal
    v
API Management / Web PubSub
    |
    v
Service Bus  (if queued processing is required)
    |
    v
Azure Functions
    |
    v
Cosmos DB / hot presence store
~~~

---

# 23. Message Lifecycle to Remember

~~~text
SEND
 |
 v
Accept message durably
 |
 v
SENT
 |
 v
Route to recipient
 |
 v
DELIVERED
 |
 v
Recipient opens message
 |
 v
READ
~~~

---

# 24. Interview Mental Model

For a WhatsApp / Telegram / Messenger design, walk through:

~~~text
1. Define features
        |
        v
2. Define scale / availability / security
        |
        v
3. Decide connection model
        |
        +--> WebSocket for real-time push
        |
        v
4. Introduce durable queue
        |
        v
5. Decide ordering requirement
        |
        v
6. Scale workers independently
        |
        v
7. Choose distributed message store
        |
        v
8. Track sent / delivered / read
        |
        v
9. Handle presence / last seen
        |
        v
10. Add retries, DLQ and idempotency
        |
        v
11. Add TLS + encryption at rest
        |
        v
12. Add E2EE for message payloads
~~~

---

# 25. Important Speech-to-Text Corrections

Notable corrections from the transcript:

- "one SAP" -> **WhatsApp**
- "Adam Lewis services" -> **AWS services**
- "synchronized design" -> **synchronous / asynchronous design**
- "way of socket API" -> **WebSocket API**
- "SKUs / rescue earth / ask us" -> **Amazon SQS**
- "DL queue" -> **Dead-Letter Queue (DLQ)**
- "easy to / easy tools" -> **EC2**
- "heartless killing group" -> **Auto Scaling Group**
- "50 excuse Q / fearful excuse Q" -> **FIFO SQS queue**
- "LAMDA" -> **AWS Lambda**
- "dynamic devices / dining room table" -> **DynamoDB**
- "health probe" -> **heartbeat / presence probe**
- "security addressed" -> **security at rest**
- "cams / cameras" -> **AWS KMS**
- "Tell US" -> **TLS**

---

# 26. Stronger Interview Answers Than the Literal Lecture

### 1. Do not say synchronous systems cannot retry

Say:

> They can retry, but asynchronous queues provide better buffering, decoupling, backpressure, and failure recovery.

### 2. Do not globally order every message

Use per-conversation ordering:

~~~text
conversationId -> FIFO MessageGroupId
~~~

or:

~~~text
conversationId -> Service Bus SessionId
~~~

### 3. Add idempotency

Queues can redeliver messages. Consumers should safely handle duplicates using a unique message ID.

### 4. Explain WebSocket connection routing

The worker needs to know where the recipient is connected. Maintain a distributed connection registry or use a managed real-time service such as Azure Web PubSub.

### 5. Avoid a durable DB write every few seconds for every online user

Use connection state / TTL / hot presence storage and persist last-seen only when useful.

### 6. Mention End-to-End Encryption

For a WhatsApp-like system, this is one of the most important security points.

---

# 27. Key Takeaways

1. **Scope the messaging features before drawing the architecture.**
2. Use a **durable message queue** to decouple ingress from processing.
3. Use **WebSockets** for real-time server-to-client delivery.
4. Scale backend workers from **queue depth**.
5. Ask whether message **ordering** is strict; preserve it per conversation if necessary.
6. Use a horizontally scalable datastore such as **DynamoDB / Cosmos DB** for high-volume message metadata.
7. Model receipt state explicitly: **SENT -> DELIVERED -> READ**.
8. Presence can use **heartbeats**, but avoid excessive durable writes at massive scale.
9. Build for **retry, DLQ, idempotency and Multi-AZ** failure handling.
10. Use **TLS + encryption at rest + E2EE** for a serious messaging design.
