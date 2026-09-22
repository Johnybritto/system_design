# System Design Session 01 — Five Must-Know Areas

Before starting system-design whiteboarding for systems such as Twitter/X, Amazon, WhatsApp, and video-streaming platforms, understand these five core areas. They repeatedly appear in architecture discussions and help when the interviewer gives an unfamiliar design problem.

## 1. Microservices

Large-scale systems are commonly decomposed into smaller services such as:

- User Service
- Order Service
- Payment Service
- Search Service
- Notification Service

### Load Balancer vs API Gateway

**Load Balancer**
- Primarily distributes traffic across healthy service instances.
- Helps with horizontal scaling and availability.

**API Gateway**
- Routes requests to multiple backend services.
- Can provide authentication/authorization.
- Can apply rate limiting and API policies.
- Can cache API responses.

Example:

```text
Client
  |
  v
API Gateway
  |
  +--> User Service
  +--> Order Service
  +--> Payment Service
```

### Synchronous vs Asynchronous Communication

**Synchronous**
- Caller waits for a response.
- Typical examples: REST, gRPC.

```text
Service A ---- request ----> Service B
          <--- response ----
```

**Asynchronous**
- Caller publishes work/event and does not have to wait for the consumer.
- Typical examples: Kafka, RabbitMQ, Amazon SQS.
- Useful for loosely coupled and highly scalable architectures.

```text
Service A
   |
   v
Message Queue / Event Bus
   |
   v
Service B
```

## 2. Database Selection

A common system-design decision is **SQL vs NoSQL**.

### SQL
Examples:
- PostgreSQL
- MySQL
- Oracle

Use when:
- Relationships between entities matter.
- Transactions are important.
- Strong consistency is required.
- Relational queries are important.

### NoSQL
Examples:
- DynamoDB
- Cassandra
- MongoDB

Use when:
- Massive horizontal scaling is required.
- Very high read/write throughput is needed.
- A flexible data model is useful.

### Database areas interviewers may probe
- Replication
- Read replicas
- Partitioning / sharding
- Indexing
- Query optimization
- Scaling reads
- Scaling writes
- Handling large datasets

Example:

```text
                 +--> Read Replica 1
Application --> Primary DB
                 +--> Read Replica 2
                 +--> Read Replica 3
```

Read replicas help offload read-heavy workloads from the primary database.

## 3. Caching

Caching reduces repeated expensive operations and lowers latency.

### Database / Application Cache

Common technologies:
- Redis
- Memcached
- DynamoDB Accelerator (DAX)

Typical flow:

```text
Application
   |
   v
Cache
   |
   +--> HIT  -> Return response
   |
   +--> MISS -> Database -> Update cache
```

### CDN Caching

A Content Delivery Network caches content close to users.

Typical cached content:
- Images
- JavaScript
- CSS
- Videos
- Static pages

```text
User
 |
 v
CDN Edge
 |
 +--> Cached -> Return immediately
 |
 +--> Miss -> Origin
```

### API Gateway Caching

Some API gateways can cache backend API responses directly, reducing traffic to microservices and databases.

**Interview takeaway:** do not think of caching as only "use Redis." Think about every point in the request path where caching can reduce latency or backend load.

## 4. Security

Security should be considered as part of the design rather than added later.

### Authentication
Answers: **Who are you?**

Examples:
- Username/password
- OAuth
- JWT
- Identity providers

### Authorization
Answers: **What are you allowed to do?**

Examples:
- RBAC
- Roles
- Permissions

### Encryption at Rest

Protect stored data such as:
- Database files
- Object storage
- Disks
- Backups

### Encryption in Transit

Protect data moving between components using TLS/HTTPS.

```text
Client
  |
 HTTPS/TLS
  v
API Gateway
  |
 TLS
  v
Microservice
```

Security checklist:
- Authentication
- Authorization
- Encryption at rest
- Encryption in transit

## 5. Scalability and High Availability

Every major component should be evaluated for:

**Scalability**
- Can it handle increasing traffic and data volume?

**High Availability**
- Can it continue operating when a component fails?

This applies to:
- Microservices
- Databases
- Caches
- Messaging systems
- API gateways
- Load balancers

Example:

```text
            Load Balancer
                 |
      +----------+----------+
      v          v          v
 Service 1   Service 2   Service 3
```

Database HA often uses replicas:

```text
Primary DB
   |
   +--> Replica
   +--> Replica
```

## Five Core Areas to Remember

1. **Microservices**
   - Load Balancer vs API Gateway
   - Synchronous vs asynchronous communication

2. **Database**
   - SQL vs NoSQL
   - Replication
   - Scaling reads/writes
   - Optimization

3. **Caching**
   - Redis / Memcached / DAX
   - CDN caching
   - API Gateway caching

4. **Security**
   - Authentication
   - Authorization
   - Encryption at rest
   - Encryption in transit

5. **Scalability & High Availability**
   - Scale every important layer
   - Avoid single points of failure

## Interview Mental Model

For almost any system-design question, walk through:

```text
1. What services do I need?
        |
        v
2. How does traffic reach those services?
        |
        v
3. Synchronous or asynchronous communication?
        |
        v
4. What database should I use?
        |
        v
5. Where can I cache?
        |
        v
6. How do I secure it?
        |
        v
7. How will it scale?
        |
        v
8. What happens when something fails?
```

## Transcript Corrections Captured

Notable speech-to-text corrections from the session:
- "five mosques knows" -> **five must-knows**
- "SQL versus SQL" -> **SQL versus NoSQL**
- Caching references interpreted as **DynamoDB Accelerator (DAX), Redis, and Memcached**
- "data at transit" -> **data in transit**

## Key Interview Takeaway

The instructor's five must-knows contain several recurring interview subtopics. Pay special attention to:
- Load Balancer vs API Gateway
- Sync vs async communication
- SQL vs NoSQL
- Cache placement
- Scalability and high availability

These should be expanded further as they appear in later system-design sessions.
