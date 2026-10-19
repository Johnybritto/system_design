# Apple ASE SRE — Complete Customer-Facing Document Metadata API System Design Mock

> **Purpose:** A solid, chronological system design interview document from the interactive mock. It follows the same flow we used: requirements → API contracts → request flow → database → security → monitoring → scaling → AZ failure. The explanations, interview-ready answers, examples, and corrections below retain the wording used in our discussion wherever reproduced. The end-of-document failure/remediation table is newly consolidated for revision.
>
> **Scenario:** Design a highly available customer-facing API on AWS EKS. Users create, retrieve, update, delete, and list **document metadata**, not the document file contents. These are practice design choices and illustrative numbers—not claims about Apple's infrastructure.
>
> **Diagram compatibility:** Short single-line Mermaid labels and text-based examples are intentional so the diagrams remain readable on GitHub. The architecture below is one valid implementation, not the only AWS design.

---

# Step 1 — Requirements Gathering

## Interviewer question

> Design a highly available, scalable customer-facing API running on AWS EKS.
>
> The API allows users to create, retrieve, update, and delete document metadata. It must support millions of users while maintaining high availability and low latency.
>
> Before proposing an architecture, what requirements and clarifications would you gather from stakeholders?

## Requirements to gather

| Requirement | What to clarify |
|---|---|
| Business functionality | Critical user journeys; create/read/update/delete/list; document sharing |
| Cloud and tenancy | Cloud vs on-premises; shared or dedicated cluster |
| Traffic | Average and peak RPS; read/write split; growth |
| Reliability | SLA versus availability SLO; failure tolerance; RPO and RTO |
| Performance | Read and write latency objectives, especially p95 and p99 |
| Data | Size, number of records, growth, schema and consistency |
| Security | Authentication, authorization, encryption and compliance |
| Retention | How long the metadata and documents must be retained |

**Three improvements for the actual interview**

1. **Define the critical user journey more precisely.** Ask what operations users perform, which are business-critical, and whether the API requires immediate consistency after updates.
2. **Distinguish SLA and SLO.** Ask for the availability SLO and latency SLO. The SLA is the contractual commitment, if one exists.
3. **Include data scale.** Ask how many documents/records exist, their expected growth, and whether users need search or pagination.

One terminology correction: The correct terms are **RPO (Recovery Point Objective)** and **RTO (Recovery Time Objective)**.

## Interviewer provides the requirements

| Requirement | Interviewer response |
|---|---|
| Business functionality | Create, read, update, delete and list document metadata |
| Deployment | AWS EKS, shared cluster |
| Traffic | 2,000 average RPS; 10,000 peak RPS |
| Read/write ratio | 80% reads, 20% writes |
| Availability SLO | 99.95% |
| Read latency | p95 below 200 ms |
| Data volume | 50 million metadata records |
| Consistency | Strong consistency for a document's updates |
| Retention | 7 years, subject to deletion policy |
| Authentication | OAuth 2.0 / OIDC |
| RPO | 5 minutes |
| RTO | 30 minutes |
| Region | One AWS region, across 3 AZs |
| Encryption | In transit and at rest |

---

# Step 2 — API Design and Contracts

## 2A — What is an API endpoint?

An API is a way for one application to communicate with another.

An **endpoint** is a specific URL that a client uses to perform an operation.

For example, think of iCloud Drive. A user wants to create a document, retrieve its metadata, rename it, or delete it.

~~~mermaid
flowchart LR
  Client[Client] --> API[API Service]
  API --> DB[Database]
  DB --> API
  API --> Client
~~~

## Five REST API endpoints you should know

| Operation | API endpoint | Purpose |
|---|---|---|
| **POST** | /documents | Create a new document's metadata |
| **GET** | /documents/123 | Retrieve document 123's metadata |
| **GET** | /documents | List documents |
| **PATCH** | /documents/123 | Update document 123's metadata |
| **DELETE** | /documents/123 | Delete document 123 |

These are examples of **REST-style endpoints**. REST is a popular API design approach that uses HTTP methods and resource URLs.

**Remember for interviews:**

- GET — Read data
- POST — Create a resource
- PUT — Replace a resource
- PATCH — Partially update a resource
- DELETE — Delete a resource

### Creating document metadata

The client sends:

~~~http
POST /documents
Content-Type: application/json
Authorization: Bearer <access_token>
~~~

Request body:

~~~json
{
  "name": "report.pdf",
  "type": "application/pdf",
  "size": 102400
}
~~~

The API validates the request, authenticates the caller, checks permissions, and stores the metadata in the database.

The API responds:

~~~http
HTTP/1.1 201 Created
~~~

~~~json
{
  "id": "123",
  "name": "report.pdf",
  "status": "created"
}
~~~

Notice that we're storing the **metadata about the document**, not the actual PDF content.

### Interview exercise — rename document

**Question:** A user wants to change the name of document 123 from report.pdf to final-report.pdf. Which method?

**Answer:** PATCH.

~~~http
PATCH /documents/123
Content-Type: application/json
~~~

~~~json
{
  "name": "final-report.pdf"
}
~~~

The API updates only the name field in the database, leaving other fields unchanged.

## 2B — REST vs GraphQL

**REST (Representational State Transfer)** is an architectural style for designing APIs.

In REST, we typically expose different URLs for resources and use HTTP methods to perform operations.

~~~mermaid
flowchart LR
  C[Client] --> A[GET documents]
  C --> B[GET document by ID]
  C --> D[PATCH document by ID]
~~~

For example, to retrieve document metadata:

~~~http
GET /documents/123
~~~

~~~json
{
  "id": "123",
  "name": "report.pdf",
  "size": 102400,
  "owner": "alice",
  "created_at": "2026-10-19"
}
~~~

Even if the client only needs the document name, a conventional REST endpoint may return all these fields.

**GraphQL is an API query language and runtime** that lets the client request exactly the fields it needs.

Typically, the client sends a query to one endpoint.

~~~http
POST /graphql
~~~

~~~graphql
query {
  document(id: "123") {
    name
    size
  }
}
~~~

~~~json
{
  "data": {
    "document": {
      "name": "report.pdf",
      "size": 102400
    }
  }
}
~~~

Notice that GraphQL returns only name and size, as requested.

| Concept | REST | GraphQL |
|---|---|---|
| Endpoints | Multiple resource URLs | Typically one |
| Data returned | Server-defined response | Client selects fields |
| Learning curve | Simpler | More complex |
| HTTP caching | Usually straightforward for GET | Often requires extra work |
| Multiple related resources | May require multiple requests | Can fetch in one query |
| Rate limiting | Commonly per endpoint/request | Query complexity also matters |
| Typical usage | CRUD APIs, microservices | Complex mobile/web applications |

Both support authentication, authorization, pagination, scaling and secure communication.

### What would we select for our Apple SRE mock design?

**I would start with REST.**

Why?

1. We have straightforward document metadata operations: create, read, update, delete and list.
2. REST is easy to operate, monitor, cache and troubleshoot.
3. We can expose versioned APIs such as /v1/documents.
4. We don't currently have complex client queries that justify GraphQL.

GraphQL could be useful later if multiple client applications need different combinations of document, owner and permission data.

**Interview-ready answer:**

> I would choose REST because our requirements are primarily CRUD-based. It gives us predictable HTTP methods, straightforward caching and simpler observability. If clients later need complex relationships or flexible field selection, I'd evaluate GraphQL.

## 2C — Authentication and authorization before implementation

- **OIDC:** Establishes who the user is (authentication).
- **OAuth 2.0:** Provides a framework for delegated access using access tokens.
- **API authorization:** Checks whether the authenticated user has permission to access document 123.

For example, Alice should not be able to delete Bob's document simply because she has a valid access token.

---

# Step 3 — High-Level Architecture and Request Flow

## Interviewer question

A customer sends this request:

~~~http
GET /documents/123
Authorization: Bearer <access_token>
~~~

> Walk me through the entire request flow, starting from the user's device until the metadata is retrieved from the database and returned to the user.
>
> Explain the components you would place between the client and the database.
>
> Also explain where you would validate the authentication token.

## Proposed architecture

~~~mermaid
flowchart TD
  C[Client] -. DNS lookup .-> DNS[Route 53]
  C --> G[API Gateway]
  G --> V[VPC Link]
  V --> N[Internal NLB]
  N --> I[EKS Ingress Controller]
  I --> S[Kubernetes Service]
  S --> P[API Pods]
  P --> R[Redis Cache]
  P --> X[RDS Proxy or PgBouncer]
  X --> DB[PostgreSQL Primary]
  DB -. replication .-> RR[Read Replicas]
  P --> S3[S3 File Storage]
~~~

**Corrected architecture for our API**

1. Client resolves api.example.com using Route 53.
2. Client sends the HTTPS request to API Gateway.
3. API Gateway validates the OAuth JWT access token, including signature, expiry, issuer, audience and required scopes.
4. API Gateway forwards valid requests through VPC Link and the internal NLB.
5. The EKS ingress controller forwards traffic to the appropriate Kubernetes Service and API Pod.
6. The application checks whether the user is authorized to read document 123, queries PostgreSQL, and returns the metadata.

**Important:** An authenticated user does not automatically have permission to access every document.

### Precision about each layer

- **Route 53 resolves DNS.** It doesn't forward individual HTTP requests to your load balancer. The client uses the DNS answer to establish the connection.
- **API Gateway and load balancers have different responsibilities.** You don't automatically need a separate public load balancer for /api traffic.
- **Authentication is not authorization.** The JWT may be valid even when a particular document is private.
- **Ingress depends on the architecture.** Depending on your controller and load balancer integration, you may not need every forwarding layer you listed.
- **EKS control plane:** Not in the customer request forwarding path.

**API Gateway vs Direct EKS Ingress**

| Capability | AWS API Gateway | Direct EKS Ingress |
|---|---|---|
| Authentication | Managed JWT/Lambda authorizers | Integrate an authentication component |
| Authorization | Token scopes / policies | Application or external authorization service |
| Rate limiting | Built-in throttling | Configure ingress/gateway rate limiting |
| Monitoring | CloudWatch metrics and logs | Configure monitoring and logging |
| Request validation | Supported, depending on API type | Application or gateway validation |
| API versioning | Routes, stages and deployments | Manage routing and versions |
| TLS | Managed HTTPS support | Configure TLS on ALB/ingress |

**One important correction:** You don't necessarily have to develop all this logic inside a custom Kubernetes Pod.

Without AWS API Gateway, you can use existing components such as an ALB, Kong, Envoy, NGINX or an authentication proxy to implement many of these capabilities.

**Interview-ready statement:**

> I would prefer AWS API Gateway if we need centralized authentication, throttling, API policies and monitoring. Without it, we can implement equivalent capabilities through an ingress or API gateway deployed on EKS, but that increases our operational responsibility. Resource-level authorization remains the responsibility of the application or a dedicated authorization service.

### Follow-up Q&A — Who rejects Alice accessing Bob's document?

**Question:** Alice has a valid JWT, but document 123 belongs to Bob and hasn't been shared with her. Should API Gateway or the application Pod reject her request?

**Answer:**

- **API Gateway:** Validates Alice's JWT token and required OAuth scope.
- **Application Pod:** Checks whether Alice owns document 123 or has permission to access it.
- **Response:** 403 Forbidden if Alice is authenticated but not authorized. Some APIs deliberately return 404 to avoid revealing that the document exists.

---

# Step 4 — Database Design

## 4A — SQL versus NoSQL

**Initial consideration:** Examine the data structure, query patterns, consistency and relationships before choosing the database.

**Important correction:**

**JSON data does not automatically mean we should choose NoSQL.**

Both databases can store JSON-like metadata:

- **PostgreSQL:** Structured columns plus JSONB for flexible metadata.
- **DynamoDB:** Items with flexible attributes, optimized around defined access patterns.

The key architectural question is not just whether the data has a schema. It's also **how the application will query, update, and relate the data**.

The API needs to retrieve a document by ID, list documents by owner, and share documents with users.

**Interview-ready answer:**

> I would select PostgreSQL for this application because we have structured document metadata, relationships between users and documents, and sharing permissions.
>
> We require ACID transactions to maintain data integrity. PostgreSQL also supports JSONB if we need flexible metadata attributes.
>
> DynamoDB is an alternative, especially for known key-based access patterns and large-scale workloads, but PostgreSQL is my initial preference for these requirements.

Two reminders:

1. PostgreSQL is itself a relational SQL database.
2. DynamoDB also supports strong consistency and ACID transactions; PostgreSQL is a natural choice here because of relational data, document-sharing permissions, constraints and transactional updates.

## 4B — Schema and Entities

### Documents table

| Column | Purpose |
|---|---|
| document_id | Primary key, unique document identifier |
| owner_id | Foreign key referencing the user |
| name | Document filename |
| description | What the document is about |
| file_type | PDF, DOCX, etc. |
| size_bytes | Document size |
| metadata | Flexible JSONB attributes |
| created_at | Creation timestamp |
| updated_at | Last modified timestamp |
| parent_document_id | Optional reference to a related/parent document |

**We're designing a document metadata API, so we shouldn't store the actual document contents in this table.** The actual files could reside in S3, with a storage reference included in metadata.

### Document_Permissions table

| document_id | user_id | permission |
|---|---|---|
| 123 | Bob | READ |
| 123 | Charlie | WRITE |

This makes sharing and revoking access easier to manage. The combination of document_id and user_id can uniquely identify each permission record.

### Follow-up Q&A — Grant Bob read access

**Question:** Alice owns document 123 and wants to give Bob read-only access. Update the Documents table or insert a new record into Document_Permissions?

**Answer:**

- **INSERT:** Bob is receiving permission for the first time.
- **UPDATE:** Bob already has READ permission, and Alice changes it to WRITE.
- **DELETE:** Alice revokes Bob's permission.

You don't necessarily need to repeat owner_id in the permissions table because ownership is already recorded in the Documents table.

## 4C — Indexing

Our query:

~~~sql
SELECT *
FROM documents
WHERE owner_id = 'Alice';
~~~

**Important correction:** Index the column being filtered.

~~~sql
CREATE INDEX idx_documents_owner
ON documents(owner_id);
~~~

An index on document_id is correct when retrieving a particular document by its document ID. PostgreSQL automatically creates an index for the primary key.

### Composite index follow-up

~~~sql
SELECT *
FROM documents
WHERE owner_id = 'Alice'
ORDER BY created_at DESC
LIMIT 20;
~~~

Recommended index:

~~~sql
CREATE INDEX idx_owner_created
ON documents(owner_id, created_at DESC);
~~~

PostgreSQL can use this index to:

1. Find documents belonging to Alice.
2. Read them in descending creation order.
3. Stop after retrieving 20 matching records, avoiding a separate sort in a favorable query plan.

| Operation | Impact of indexing |
|---|---|
| SELECT | Usually faster for matching queries |
| INSERT | Additional work to maintain indexes |
| UPDATE | May require index maintenance if indexed values change |
| DELETE | Index entries must be cleaned up |
| Storage | Indexes consume disk space |

**Interview takeaway:** Don't create indexes on every column. Create them based on frequent queries and validate their usefulness with EXPLAIN ANALYZE.

For large result sets, use pagination rather than returning every document in one response.

## 4D — Read replicas and consistency

**User's design choice:** One primary database for writes and two read replicas for eligible reads, since traffic is 80% reads.

~~~mermaid
flowchart TD
  P[EKS API Pods] --> W[PostgreSQL Primary]
  W -. async replication .-> R1[Read Replica 1]
  W -. async replication .-> R2[Read Replica 2]
  P --> R1
  P --> R2
~~~

These aren't three independent databases. They are a primary database and two replicas of its data.

**Important interview point:** Read replicas can experience replication lag. If Alice updates her document and immediately retrieves it, a replica might return the old value. For read-after-write consistency, we may need to route that read to the primary.

### Follow-up Q&A — Alice renames document and immediately reads

**Question:** Alice updates report.pdf to final.pdf. The update succeeds on primary. Read replica is 2 seconds behind. Send GET to primary or replica?

**Interview-ready answer:**

> Since our application requires strong consistency for document updates, I would route immediate read-after-write requests to the primary database. Read replicas may have replication lag, so they could return stale information. I would use replicas for read operations where eventual consistency is acceptable.

We don't need to send every GET request to the primary. We can use replicas for eventually consistent reads and the primary for operations requiring the latest committed data.

## 4E — Database Connection Pooling

**Scenario:**

- 20 application Pods, scaling to 50.
- Each Pod handles multiple concurrent requests.
- PostgreSQL supports a limited number of database connections.
- Traffic may rise from 2,000 to 10,000 RPS.

**What you got right**

- Reuse existing database connections instead of opening new ones for every request.
- Consider the impact of scaling from 20 to 50 Pods.
- Investigate slow queries before increasing pool sizes.
- Recognize that requests may queue while waiting for an available connection.

~~~mermaid
flowchart TD
  A[EKS Application Pods] --> C[Bounded Application Pools]
  C --> RC[Redis Cache]
  C --> PR[RDS Proxy or PgBouncer]
  PR --> D[PostgreSQL]
~~~

**Where does connection pooling happen?**

There are two common approaches:

- Application-side pools, such as HikariCP for Java.
- A shared database connection layer such as RDS Proxy or PgBouncer.

Application-level pooling can be sufficient, but RDS Proxy or PgBouncer can help manage database connections across many Pods. Actual connection reuse depends on pool configuration and session behavior.

**What happens when the pool is exhausted?**

We should never allow requests to wait indefinitely.

We should configure bounded waiting, connection acquisition timeouts and backpressure. If the service is overloaded, it may reject requests with an appropriate error rather than allowing an uncontrolled queue to grow.

### Connection limit follow-up

| Component | Connections |
|---|---:|
| PostgreSQL maximum | 300 |
| 50 Pods × 20 connections | 1,000 potential |
| Proposed 50 Pods × 4 connections | 200 potential |
| Remaining connection capacity | 100 |

I would initially limit each Pod's pool size, keeping headroom for administrative operations and other database clients, then tune it using measured demand.

**Two important improvements:**

1. Even if you introduce RDS Proxy or PgBouncer, the database cannot suddenly execute 1,000 queries concurrently just because a proxy is present.
2. Caching reduces database queries, not necessarily every database connection. If a GET request can be safely served from Redis, it may avoid borrowing a database connection altogether.

**Interview-ready answer:**

> I would use bounded connection pools, check query performance and database saturation before increasing connections, and consider RDS Proxy or PgBouncer for managing connections across Pods. I'd protect PostgreSQL with acquisition timeouts, bounded concurrency, backpressure, and caching for eligible reads. I would avoid uncontrolled retries during saturation.

## 4F — Data Retention and Deletion

**Requirement:** Seven years of retention, in this scenario counted from document creation. Actual document contents are stored separately from PostgreSQL metadata.

**Original design alternatives discussed:** A separate deleted-records table, or a deletion marker in the existing Documents table.

**Prefer soft deletion in the existing table initially.**

| Column | Purpose |
|---|---|
| is_deleted | True or false |
| deleted_at | When the user deleted the document |
| retention_until | When the retention obligation ends |

On deletion, update these fields and hide the document from normal API responses. Revoke ordinary access as appropriate.

**Seven-year retention does not automatically prohibit permanent deletion.** We need to establish whether seven years is a mandatory legal retention period, a business default, or a maximum retention period. Privacy requirements, deletion rights and legal holds may change what we're required or permitted to retain.

When a document becomes eligible for permanent deletion, we must consider PostgreSQL, S3 objects and versions, caches, and backup expiration—not merely its row in the main table.

### Follow-up Q&A — Created two years ago; delete requested today

**Question:** Can we hide the document from the user's view and keep the underlying data for five more years? Is lifecycle alone enough to prevent deletion?

**Answer:**

> I would soft-delete the document from the application's view while retaining the underlying data until its seven-year retention period ends.
>
> For the S3 object, I would use Object Lock in Compliance mode to protect it against premature deletion, and lifecycle or Intelligent-Tiering policies to optimize storage costs.
>
> After the retention period expires, and provided there are no additional legal holds, the cleanup process can permanently delete the eligible data.

**S3 Lifecycle:** Controls transitions between storage classes and eventual expiration.

**S3 Object Lock:** Can protect retained object versions from deletion or modification during a mandatory retention period.

**Storage tiering:** Controls storage cost and retrieval characteristics.

**Retention:** Controls how long data must be preserved.

**Deletion protection:** Prevents premature removal during mandatory retention.

**Important:** S3 Object Lock protects S3 object versions, **not PostgreSQL metadata**. We must enforce metadata retention separately.

### Follow-up Q&A — Move locked object to Standard-IA

**Question:** Can we move an S3 Standard object protected by Object Lock to Standard-IA after 30 days?

**Answer:** Yes, we are not deleting the data; we are moving the data to S3 IA.

Object Lock protection remains in place across supported storage-class transitions. Standard-IA has a 30-day minimum storage duration charge. S3 Intelligent-Tiering is another choice for access-based tiering.

---

# Step 5 — API Security

## 5A — Authentication: OIDC and OAuth 2.0

**Interviewer question:**

> Walk me through how Alice logs in, obtains an access token, and how API Gateway validates that token before forwarding the request to the application Pod.

**Corrected authentication flow**

~~~mermaid
flowchart TD
  A[Alice opens application] --> I[Identity Provider]
  I --> C[Authorization Code]
  C --> T[Token Endpoint with PKCE]
  T --> J[Access and ID Tokens]
  J --> G[API Gateway JWT Validation]
  G --> P[EKS Application Authorization]
~~~

1. Alice opens the application and clicks Login.
2. Identity Provider (Microsoft Entra ID / Okta / Cognito) authenticates Alice.
3. Authorization Code is returned through redirect to client.
4. Token Endpoint exchanges code + PKCE verifier for tokens.
5. API Gateway validates access token.
6. EKS application checks document permissions.

In Step 4, a backend application may perform the code exchange rather than the browser, depending on the client architecture.

### Three corrections to improve the interview answer

**1. Authorization Code + PKCE**

For a modern web or mobile application, I would describe the **Authorization Code flow with PKCE**.

PKCE helps protect the authorization code exchange by requiring proof that the client exchanging the code is the one that initiated the authorization request.

**2. JWT is a token format, not a secure connection**

JWT is a structured token format containing claims about the user or granted access. Its digital signature lets the API verify the token's integrity and issuer.

HTTPS/TLS provides connection security. JWT signatures provide token integrity and authenticity.

**3. OIDC and OAuth 2.0 have different responsibilities**

- **OIDC:** Authenticates Alice and can provide an ID token.
- **OAuth 2.0:** Provides the authorization framework used to obtain an access token for the API.
- **API Gateway:** Validates the access token before allowing the API request.

### What exactly does API Gateway validate?

| Check | Meaning |
|---|---|
| Signature | Was this JWT signed by a trusted issuer? |
| iss | Is it from the expected identity provider? |
| aud | Is it intended for the configured audience? |
| exp | Has the token expired? |
| scope | Does it have the required API permission? |

### Follow-up Q&A — Expired token

**Question:** Alice's access token expired 10 minutes ago. What does API Gateway do? How can the client get another token without asking Alice to log in again?

**Interview-ready answer:**

> API Gateway rejects the expired access token with HTTP 401. The client can then securely send its refresh token to the authorization server to obtain a new access token. Once the new token is issued, the client retries the original API request. This avoids asking the user to log in again, provided the refresh token remains valid.

- **Access token:** Sent to the API Gateway to access protected resources.
- **Refresh token:** Sent to the authorization server's token endpoint to obtain a new access token. It should be stored securely and never sent as the API access credential.
- **Expired refresh token:** The client may require Alice to authenticate again if silent reauthentication isn't possible.

API Gateway generally returns 401; the client application handles token renewal rather than API Gateway automatically redirecting the user.

### Follow-up Q&A — 401, 403 or 404?

| Status | When to use |
|---|---|
| 401 Unauthorized | Expired or invalid authentication token |
| 403 Forbidden | Alice is authenticated but lacks permission |
| 404 Not Found | We deliberately avoid revealing whether Bob's document exists |

For our document API, we can choose 404 for unauthorized document lookups, ensuring that responses don't reveal private document existence through inconsistent behavior.

Both 403 and deliberately chosen 404 can be appropriate. Explain the design decision.

## 5B — Encryption in Transit

**Interviewer question:** How would you secure data traveling through Alice → Route 53 → API Gateway → internal NLB → EKS Pods → PostgreSQL / S3?

**Corrections from our discussion**

1. **Route 53 doesn't terminate TLS.** Route 53 is a DNS service. The actual HTTPS request goes from Alice's client to API Gateway, not through Route 53.
2. **TLS termination doesn't mean we have to send unencrypted traffic afterward.** API Gateway can terminate the client's TLS session and establish a new HTTPS connection to the backend when configured to do so.
3. **Security groups do not encrypt traffic.** Security groups control which connections are permitted. TLS encrypts the data carried over those connections.

~~~mermaid
flowchart TD
  C[Alice Browser] -. DNS lookup .-> R[Route 53]
  C -->|HTTPS| G[API Gateway TLS Termination]
  G -->|New HTTPS connection| V[VPC Link]
  V --> N[Internal NLB TCP 443]
  N -->|TLS passthrough| I[EKS Ingress TLS Termination]
  I -->|Re-encrypted HTTPS or mTLS| P[API Pod]
  P -->|TLS 5432| D[PostgreSQL]
  P -->|HTTPS 443| S[S3]
~~~

For our application handling sensitive document metadata, I would initially use the following configuration:

| Component | What happens |
|---|---|
| Route 53 | DNS only, no application TLS termination |
| API Gateway | Terminates Alice's HTTPS connection |
| VPC Link | Private connectivity to the backend |
| NLB TCP:443 | TLS passthrough |
| EKS Ingress | Terminates backend TLS |
| Application Pod | Receives traffic over re-encrypted TLS/mTLS in our design |
| PostgreSQL | Separate TLS-secured DB connection |
| S3 | Separate HTTPS connection |

For our design, application Pods and the ingress controller must actually be configured to support the chosen encryption method. Simply enabling HTTPS on API Gateway does not automatically encrypt every internal hop.

### Follow-up Q&A — NLB TCP:443 termination

**Question:** If an NLB uses a TCP listener on 443, who terminates TLS, the NLB or EKS ingress controller? Where is the certificate configured?

**Answer:** Option B — EKS Ingress Controller.

When the NLB uses a TCP listener on port 443, it passes encrypted traffic through without terminating TLS.

The **EKS Ingress Controller terminates TLS** using a certificate configured for the requested hostname.

The certificate can be stored in a Kubernetes TLS Secret, referenced by the Ingress configuration, and managed using cert-manager.

**Important detail:** API Gateway must also be configured to establish HTTPS to the private integration. TLS passthrough at the NLB alone doesn't guarantee end-to-end encryption.

## 5C — Encryption at Rest

**AWS KMS manages encryption keys; RDS and S3 use those keys to encrypt data at rest.**

| Component | Encryption approach |
|---|---|
| RDS PostgreSQL | Enable encryption at rest using an AWS KMS key |
| S3 | Enable SSE-KMS (server-side encryption using KMS keys) |
| RDS backups and snapshots | Protected through RDS storage encryption |
| Encryption key management | AWS KMS |

Both services use envelope encryption, involving data encryption keys protected by KMS keys.

**Interview-ready answer:**

> I would enable encryption at rest for RDS PostgreSQL using AWS KMS. For S3, I would use SSE-KMS. I would manage access to the encryption keys through IAM and KMS key policies and enable key rotation where appropriate.

### Follow-up Q&A — KMS key pending deletion

**Question:** Someone schedules deletion of the KMS key encrypting RDS. What happens? What should we do?

- KMS customer-managed keys have a **7–30-day deletion waiting period**, with 30 days as the default.
- The key enters the PendingDeletion state.
- Scheduled deletion can be canceled before the waiting period expires.
- After permanent deletion, data requiring that key to decrypt may become unrecoverable.

**Critical correction:**

**The key becomes unavailable for cryptographic operations as soon as deletion is scheduled—not only after 7–30 days.**

That means applications relying on that key may experience failures before the key is permanently deleted. The exact timing depends on how the AWS service uses and caches data encryption keys.

If you cancel deletion, the KMS key becomes Disabled. You must explicitly enable it again.

~~~bash
aws kms cancel-key-deletion --key-id <key-id>
aws kms enable-key --key-id <key-id>
~~~

**Incident priorities:**

1. Determine whether key deletion was accidental or authorized, but don't let investigation delay safe recovery.
2. Cancel scheduled deletion and re-enable the KMS key.
3. Verify RDS access to encryption credentials and application database health.
4. Restrict deletion permissions and create an EventBridge alert on the CloudTrail ScheduleKeyDeletion event.

Canceling deletion alone does not guarantee database service recovery.

---

# Step 6 — Monitoring, Alerting and Incident Response

## 6A — Customer-facing SLIs

**Start with the four golden signals:**

| Signal | What to measure |
|---|---|
| Latency | p50, p95, p99 response times |
| Traffic | Requests per second, per endpoint |
| Errors | HTTP 5xx rate, unexpected failures |
| Saturation | CPU, memory, connection pool and queue utilization |

Since we've defined an availability SLO of **99.95%**, we should also measure the success rate of eligible API requests.

For example, 401 from an invalid token and 403 for a correctly denied operation do not necessarily indicate application failures. We need to define which responses count against our availability SLO.

**Interview-ready answer:**

> I would first define customer-facing SLIs covering request success rate, p95/p99 latency and request throughput.
>
> I would create dashboards for the API Gateway and application endpoints, followed by EKS, Redis and PostgreSQL infrastructure metrics.
>
> I'd enable distributed tracing to identify latency across API Gateway, ingress, application and database dependencies.
>
> Finally, I'd configure SLO-based alerts, using error-budget burn rates for paging and lower-priority alerts or tickets for noncritical issues.

**Monitoring to include**

- API Gateway HTTP rates, error counts and latency.
- Ingress requests, latency, backend health and status-code breakdown.
- EKS Pod health, readiness, restarts, CPU, memory and Pending Pods.
- Redis cache hit ratio, evictions, memory and failover.
- PostgreSQL CPU, query latency, locks, IOPS and active connections.
- Traces/APM spans through gateway, ingress, app and dependencies.
- CloudWatch/Grafana dashboards and paging/ServiceNow routing.

## 6B — Error-budget burn-rate scenario

**Observed metrics:**

| Metric | Observed | Evaluation |
|---|---|---|
| Traffic | 2,000 RPS | Normal |
| p95 latency | 180 ms | Within 200 ms target |
| HTTP 5xx | 2% | High failure rate |
| Availability SLO | 99.95% | Allows only 0.05% failures |

**Interviewer:** Page on-call or create a ServiceNow ticket?

**Correct answer:** Page on-call.

Our availability SLO allows 0.05% failed requests. But we're currently experiencing 2%.

- 2% actual failure rate ÷ 0.05% allowed = **40× error-budget consumption rate**.
- 2,000 RPS × 2% = **40 failed requests per second**.
- Over 10 minutes = **24,000 failed requests**.

Assuming all 5xx responses count as failures, our API is burning its availability error budget 40 times faster than the sustainable rate.

This doesn't necessarily mean the rolling 30-day SLO has already been breached. But it is a serious customer-impacting incident.

**One more correction:** p95 latency below 200 ms doesn't mean the application is healthy. Successful requests can remain fast while 2% of requests fail.

## 6C — Recent deployment and HTTP 5xx

**Scenario:** Normal traffic, normal p95 latency, 2% HTTP 5xx, new application version deployed 15 minutes ago.

**Preferred first three actions:**

1. **Identify the scope and source of errors.** Check which endpoints return 5xx, which component generates them, and whether the errors affect the new application version specifically.
2. **Correlate errors with the deployment.** Compare error rates before and after deployment. Review deployment health, application logs and traces. If both old and new versions are running, compare their failure rates.
3. **Mitigate quickly.** If the deployment is strongly correlated with the errors and rollback is safe, initiate rollback through the deployment pipeline. Verify the 5xx rate drops and customer requests recover.

**Interview-ready response:**

> First, I would establish the incident scope and identify which component is generating the HTTP 5xx errors.
>
> Since a deployment happened 15 minutes ago, I would correlate the deployment timeline with the increase in errors and compare the old and new application versions.
>
> If the new deployment is the likely cause, I would initiate a safe rollback through our CI/CD pipeline rather than spend excessive time on root-cause investigation.
>
> Once service is restored, I would verify the error rate, latency and request success rate before performing detailed RCA.

### Follow-up Q&A — Old vs new version

| Metric | Old version | New version |
|---|---|---|
| HTTP 5xx error rate | 0.01% | 8% |
| Database | Same | Same |
| CPU / Memory | Normal | Normal |

**Interview-ready answer:**

> Since the new application version has an 8% error rate while the previous version has only 0.01%, I would strongly suspect the new deployment.
>
> I would initiate a safe rollback through the deployment pipeline.
>
> After rollback, I would monitor the HTTP 5xx error rate and request success rate to confirm customer impact has been resolved.

**Important:** We don't have to identify the exact buggy line of code before rolling back. Restoring service takes priority.

### Follow-up Q&A — When is incident resolved?

After rollback, 5xx returns to 0.01% and p95 latency is 170 ms.

> After rollback, the error rate has returned to 0.01%, and p95 latency is within our SLO.
>
> I would verify our critical API endpoints, confirm normal traffic and monitor the application for a period to ensure stability.
>
> Once recovery is confirmed, I would resolve the incident and initiate an RCA to improve our deployment validation, canary rollout and monitoring.

## 6D — Cache failure becomes database overload

**Scenario:**

| Metric | Before | Now |
|---|---:|---:|
| Redis cache hit ratio | 90% | 30% |
| Redis cache miss ratio | 10% | 70% |
| PostgreSQL CPU | Normal | 95% |
| Database connections | Normal | Near maximum |
| HTTP 5xx | Low | Increasing |

Assuming the same volume of cacheable reads, 70% cache misses ÷ 10% cache misses = **7× relative DB read demand from misses**. Actual load depends on query patterns, writes and whether misses cause database calls.

~~~mermaid
flowchart TD
  R[Redis cache hit ratio falls] --> M[More cache misses]
  M --> D[More PostgreSQL reads]
  D --> S[CPU and connections saturate]
  S --> E[Timeouts and HTTP 5xx]
~~~

**Correct troubleshooting sequence**

1. **Protect PostgreSQL.** Apply appropriate rate limiting, bounded concurrency and load shedding to reduce overload. Avoid aggressive retries that could worsen the problem.
2. **Investigate Redis.** Check for evictions, expired keys, memory pressure, Redis errors, connectivity issues or a recent mass invalidation.
3. **Restore the caching layer.** Resolve the underlying Redis issue and carefully warm frequently accessed keys if needed, without creating another database spike.
4. **Verify recovery.** Monitor Redis hit ratio, PostgreSQL CPU, active connections, query latency and the API 5xx error rate.

### Follow-up — Should we scale app Pods and connection pools?

**Question:** Redis is evicting keys; DB at 95% CPU. An engineer suggests scaling 20 → 50 Pods and increasing DB connection pools.

**Answer:** No, this would add more pressure to DB.

**Interview-ready answer:**

> I would not immediately increase application Pods or database connections because PostgreSQL is already saturated. That could make the problem worse. I would first protect the database through rate limiting and backpressure, fix Redis eviction issues, and gradually restore caching. Once the database stabilizes, I would evaluate whether additional capacity is needed.

---

# Step 7 — Scaling and Capacity Planning

## 7A — Application capacity estimation

| Metric | Current | Expected |
|---|---:|---:|
| Traffic | 2,000 RPS | 10,000 RPS |
| Application Pods | 20 | To be calculated |
| RPS per Pod | 100 | Assume 100 |
| Required Pods | 20 | 100 |

If 20 Pods handle 2,000 RPS, then one Pod handles approximately 100 RPS at the current operating point.

Therefore, for 10,000 RPS: **10,000 / 100 = 100 Pods**.

That's a theoretical estimate. Validate it through load testing and provision additional headroom.

## 7B — HPA vs Cluster Autoscaler vs Karpenter

| Component | Responsibility |
|---|---|
| HPA | Increases or decreases application Pods |
| Cluster Autoscaler | Scales existing EC2 Auto Scaling Groups / node groups |
| Karpenter | Provisions appropriately sized EC2 worker nodes based on Pod requirements |

~~~mermaid
flowchart TD
  T[Traffic grows] --> H[HPA increases desired replicas]
  H --> P[Pending Pods if nodes full]
  P --> K[Karpenter or Cluster Autoscaler]
  K --> N[Additional worker nodes]
  N --> R[Pods scheduled and Ready]
~~~

**Interview-ready answer:**

> I would use HPA to scale application replicas based on CPU utilization or suitable application metrics. If the existing nodes cannot schedule additional Pods, I would use Karpenter or Cluster Autoscaler to provision worker nodes. I would distribute capacity across three Availability Zones and maintain sufficient headroom for traffic spikes.

## 7C — Predictable 10 AM traffic spike

**Question:** Would you simply rely on HPA and Karpenter when the 10 AM spike arrives?

**Answer:** No. Prepare additional capacity beforehand.

**Important refinement:** HPA may scale down your manually increased replica count if it still sees low CPU utilization.

For predictable traffic, use scheduled scaling or temporarily adjust the HPA minimum replicas, and ensure worker-node capacity is provisioned in advance.

**Interview-ready answer:**

> Since we know traffic will increase at 10 AM, I would proactively scale application Pods and worker nodes before the event. I would configure scheduled scaling or adjust HPA minimum replicas to prevent premature scale-down. Karpenter would provision additional capacity as required. Finally, I would run load tests and confirm that all Pods and load balancer targets are healthy before the traffic spike.

## 7D — Database scaling: reads, connections and writes

At 10,000 RPS with 80/20 read/write split:

| Traffic | RPS |
|---|---:|
| Total | 10,000 |
| Reads | 8,000 |
| Writes | 2,000 |

Read replicas can help serve the 8,000 read requests per second, and Redis can reduce database reads.

But **the PostgreSQL primary must still handle the 2,000 write requests per second**, assuming each write request requires a database write.

Read replicas don't increase the primary's write capacity.

**What you got right and what to improve**

| Proposal | Evaluation |
|---|---|
| Add read replicas | Correct, for reads that tolerate replication lag |
| Increase connection pool size | Don't increase blindly; measure DB capacity first |
| Backoff and retries | Use bounded retries with jitter only for appropriate transient failures |
| Multiple Redis instances | Consider ElastiCache replication or cluster mode, based on capacity and HA requirements |

Before scaling, load-test the primary; examine slow queries, lock contention, CPU, IOPS, transaction latency and connection utilization.

### Follow-up — 2,000 write RPS saturates primary

**Two cases:**

**1. Synchronous writes**

If the API must confirm that document metadata has been committed before responding, consider scaling up PostgreSQL by increasing CPU, memory or storage IOPS.

But first, check slow queries, expensive indexes, locks and transaction latency to establish the actual bottleneck.

**2. Asynchronous writes**

~~~mermaid
flowchart LR
  A[API Application] --> Q[SQS Queue]
  Q --> W[Write Workers]
  W --> D[PostgreSQL Primary]
~~~

The queue helps absorb temporary spikes, and workers consume messages at a controlled rate.

**One important correction:** SQS doesn't increase PostgreSQL's actual write capacity. It buffers requests so the database isn't overwhelmed.

If the client requires confirmation that a write has been committed, simply putting the request into SQS changes the API's behavior. An asynchronous API could instead return 202 Accepted and provide a way to check completion.

### Temporary versus sustained write pressure

| Scenario | Preferred approach |
|---|---|
| Temporary write spike | SQS buffering if asynchronous writes are acceptable, or sufficient provisioned capacity |
| Sustained write overload | Optimize and increase database write capacity |
| Database connection exhaustion | Bounded connection pools, proxy, backpressure |
| Heavy read traffic | Cache and read replicas |

**Scenario A:** 2,000 writes/sec for 5 minutes. SQS can buffer a temporary spike if asynchronous behavior is acceptable; a bounded pool alone cannot safely absorb arbitrary excess writes.

**Scenario B:** 2,000 writes/sec all day. SQS alone cannot solve this. If messages arrive faster than PostgreSQL can process them indefinitely, the backlog grows. Optimize and increase sustained DB write capacity.

---

# Step 8 — Failure Scenarios and High Availability

## 8A — First identify the failure domain

| Failure | Preferred starting approach |
|---|---|
| One Pod fails | Kubernetes restarts or replaces the Pod |
| One worker node fails | Kubernetes reschedules affected Pods |
| One Availability Zone fails | **Multi-AZ high availability within the same Region** |
| Entire AWS Region fails | **Cross-Region disaster recovery** |

**Important correction from our mock:** The interviewer asked about one AZ failing, but the first answer described a standby in another Region. That is a good DR topic, but it's not required to survive a single AZ failure.

## 8B — One AZ goes down

~~~mermaid
flowchart TD
  U[Customer] --> G[API Gateway]
  G --> L[Internal NLB]
  L -. failed .-> A[AZ A Unavailable]
  L --> B[AZ B Healthy Pods]
  L --> C[AZ C Healthy Pods]
  B --> D[RDS Multi-AZ]
  C --> D
  B --> R[Redis Multi-AZ]
  C --> R
~~~

| Component | Expected behavior |
|---|---|
| Route 53 | No separate regional DNS failover required |
| API Gateway | Regional service remains available |
| NLB | Routes to healthy targets; failed AZ can be removed from zonal DNS responses |
| EKS Pods | Pods in failed AZ become unavailable; replacement Pods can run in healthy AZs |
| EKS control plane | AWS-managed, distributed across multiple AZs |
| PostgreSQL Multi-AZ | Automatic standby promotion if primary AZ fails |
| Redis Multi-AZ | Automatic replica promotion if primary fails and failover is configured |

These behaviors depend on properly configured Multi-AZ resilience.

**Three architecture decisions to mention**

1. **Pod placement:** Use topology spread constraints and enough spare capacity across the surviving AZs.
2. **Database recovery:** RDS Multi-AZ failover is automatic, but it can cause a temporary interruption. Application connections must recover.
3. **Redis recovery:** Enable Multi-AZ replication and automatic failover. Ensure Redis failure doesn't overwhelm PostgreSQL.

One additional point: RDS Multi-AZ standby and read replicas are not necessarily the same thing. A conventional Multi-AZ DB instance uses a standby for HA, whereas separate read replicas help with read scaling.

## 8C — PostgreSQL primary AZ fails

**Question:** Our primary PostgreSQL is in AZ-A and standby in AZ-B. Does AWS promote automatically? What happens to DB connections?

**Correct answer:** AWS RDS Multi-AZ automatically promotes the standby. Connections may fail and must reconnect; SQS is not the mechanism that routes database connections.

~~~mermaid
flowchart TD
  F[AZ A and Primary Fail] --> R[RDS Detects Failure]
  R --> P[Promotes AZ B Standby]
  P --> DNS[RDS Endpoint DNS Updated]
  DNS --> REC[Application Reconnects]
  REC --> OK[Operations Resume]
~~~

- **SQS:** Queues application messages or asynchronous write jobs.
- **Connection pool / RDS Proxy:** Manages database connections.
- **RDS endpoint:** Provides the hostname used to connect to the current primary.

**ACID note:** Transactions are atomic, but a lost connection during commit can leave the client uncertain whether a write committed. Use idempotency or transaction-status verification before retrying writes.

**Interview-ready answer:**

> RDS automatically promotes the standby database in the healthy AZ. Existing database connections may break during failover, so the application must reconnect through the RDS endpoint or use RDS Proxy to reduce disruption. I would configure bounded retries with backoff and jitter, ensure writes are idempotent, and verify database connectivity after failover.

A standard RDS Multi-AZ DB instance typically takes 60–120 seconds to fail over, although actual recovery time varies.

## 8D — Redis primary AZ fails

**Scenario:** Redis primary in AZ-A, replicas in AZ-B and AZ-C, ElastiCache Multi-AZ automatic failover enabled.

With Multi-AZ automatic failover enabled, ElastiCache selects an available replica, normally preferring the one with the least replication lag, and promotes it to primary. Applications reconnect through the configured endpoint. There may be a short interruption, and asynchronous replication can result in some lost recent cache writes.

**You do not need to manually promote the Redis replica.**

~~~mermaid
flowchart TD
  A[Redis Primary AZ A Fails] --> B[ElastiCache Automatic Failover]
  B --> C[Promote Healthy Replica]
  C --> D[Primary Endpoint Reconnect]
  D --> E[Cache Recovery]
~~~

**How do we protect PostgreSQL?**

| Technique | Why we use it |
|---|---|
| Rate limiting | Control incoming request volume |
| Bounded concurrency | Restrict simultaneous DB operations |
| Circuit breaker | Stop repeated calls to a failing Redis or DB dependency |
| Backoff with jitter | Avoid synchronized retries |
| Single-flight loading | Prevent many requests from loading the same missing cache key simultaneously |
| Load shedding | Reject lower-priority requests when DB capacity is exhausted |

**Remember:** A circuit breaker on Redis alone does not protect PostgreSQL. We also need to limit the number of requests allowed to fall back to the database.

**Interview-ready answer:**

> If the Redis primary fails, ElastiCache Multi-AZ automatically promotes a healthy replica. The application reconnects through the Redis endpoint.
>
> During failover, I would protect PostgreSQL using bounded concurrency, rate limiting and load shedding. I'd avoid uncontrolled retries and gradually restore the cache once Redis becomes healthy.

**Quick check:** Redis down for 30 seconds, DB healthy. Allow controlled fallback with rate limiting and bounded concurrency, **not** unlimited direct DB calls.

## 8E — EKS Pod and worker-node recovery

**Scenario:** 90 Pods, 30 in each AZ. AZ-A fails.

| Component | Responsibility |
|---|---|
| Deployment / ReplicaSet | Maintains the desired 90 application replicas |
| Kubernetes scheduler | Assigns Pending Pods to suitable nodes |
| Cluster Autoscaler / Karpenter | Adds worker nodes when Pods cannot be scheduled |
| HPA | Adjusts desired replica count based on configured metrics |

**Important corrections**

1. The EKS control plane remains available. Losing one worker-node AZ should not make the entire control plane unavailable.
2. Cluster Autoscaler doesn't create application Pods.
3. If the Deployment already requires 90 Pods, HPA does not need to increase the replica count merely because an AZ failed.

~~~mermaid
flowchart TD
  A[AZ A Fails] --> B[30 Pods Lost]
  B --> C[Kubernetes Detects Node Failure]
  C --> D[ReplicaSet Replaces Lost Pods]
  D --> E[Scheduler Places Pods]
  E --> F[Karpenter Adds Nodes If Needed]
  F --> G[Ready Pods in AZ B and AZ C]
~~~

Configure topologySpreadConstraints using topology.kubernetes.io/zone to distribute Pods. Constraints should allow scheduling in surviving zones, and Karpenter must be allowed to provision nodes there.

**Interview-ready answer:**

> When AZ-A fails, Kubernetes detects unavailable nodes. The Deployment controller maintains the desired replica count by creating replacement Pods once the failed Pods are removed.
>
> The scheduler attempts to place them in healthy AZs. If capacity is insufficient, Karpenter provisions additional worker nodes. I would configure topology spread constraints and ensure sufficient capacity to run all 90 Pods across the remaining two AZs.
>
> Finally, I would verify that the Pods are Ready, load balancer targets are healthy, and the API meets its availability and latency targets.

## 8F — How much AZ headroom?

**Scenario:** 90 Pods needed to sustain peak traffic; 30 Pods per AZ.

| Metric | Pods |
|---|---:|
| Required for peak traffic | 90 |
| Current Pods per AZ | 30 |
| Remaining after one AZ fails | 60 |
| Additional Pods required | 30 |

If we reserve 30% additional node capacity per AZ, that's approximately 39 Pod slots per AZ.

After one AZ fails, two AZs provide only **78 Pod slots**, less than the required 90.

To run 90 Pods across two surviving AZs, we need capacity for **45 Pods per AZ** — 50% additional node capacity relative to the original 30 Pods per AZ.

**Spare nodes vs spare application Pods**

- **Pre-provisioned worker-node headroom:** Keep 90 application Pods running with enough node capacity for replacements. Recovery still takes time while Kubernetes creates and schedules replacement Pods.
- **Pre-running additional application Pods:** Run 45 Pods per AZ (135 total), appropriately sized for the workload. If an AZ fails, 90 Pods are already running in the other two AZs, reducing recovery delay.

For a strict availability requirement, evaluate pre-running additional application capacity rather than relying entirely on reactive scaling.

### Final interview knowledge check

**Question:** 135 running Pods across 3 AZs; one AZ completely fails. How many remain immediately?

**Answer:** 90 pods available, 45 down.

Since the application needs 90 Pods for peak traffic, the surviving AZs can continue handling peak traffic without waiting for Karpenter to provision nodes, assuming Pods and dependencies remain healthy.

One consideration: We now have **no spare application capacity** after the AZ failure. Restore redundancy while monitoring latency and errors.

---

# Overall Mock Feedback

**Overall practice assessment: 8/10.** This was a guided practice, not yet a timed uninterrupted 25–30-minute interview performance.

| Area | Evaluation |
|---|---|
| Requirements gathering | Strong |
| REST API design | Good, continue practicing |
| High-level architecture | Good, improve request-flow precision |
| Database design | Strong |
| Security and encryption | Strong |
| Monitoring and incident response | Needs improvement |
| Scaling and capacity planning | Good |
| HA, AZ failure and DR | Good, distinguish AZ versus Region failures |

**Three areas to prioritize**

1. **Incident response:** Identify the most likely failure quickly, mitigate customer impact, and investigate root cause afterward.
2. **API design:** Become comfortable explaining endpoints, HTTP methods, authentication and authorization.
3. **Capacity and consistency trade-offs:** Explain why scaling Pods, adding database connections, introducing queues or using read replicas may not always solve the problem.

**One important thing remains:** We've covered the individual decisions, but **we have not yet demonstrated the complete design as one uninterrupted 25–30-minute interview answer**.

---

# Final Reference — Components, Failure, Remediation and Recovery Verification

| Component / layer | Failure or symptom | Immediate mitigation / remediation | How to verify recovery |
|---|---|---|---|
| Client / DNS / Route 53 | DNS lookup fails, wrong record or stale resolution | Check authoritative record, alias/CNAME, TTL and DNS resolution; fix record; distinguish DNS issue from HTTP errors | DNS resolves to intended API endpoint; successful client requests |
| API Gateway | 401/403/429/5xx, JWT authorization or throttling issue | Inspect authorizer configuration, issuer/audience, token expiry, scopes and gateway throttling; restore known-good policies | Authentication works; expected 401/403 behavior; 5xx and 429 metrics return to normal |
| TLS / certificates | TLS handshake failure or expired certificate | Identify termination point, hostname/SNI and trust chain; rotate/restore certificate; verify HTTPS private integration | TLS handshake succeeds at each configured boundary |
| VPC Link | Private integration connectivity failure | Check VPC Link state, target routing, listener, subnets and network access | API Gateway can reach backend; integration errors normalize |
| Internal NLB | Unhealthy targets, wrong TCP listener or zonal impact | Check listeners, target groups, target health, security rules and surviving AZs; restore healthy targets | Healthy target counts and successful forwarding |
| EKS ingress controller | Misrouting, 502/503/504, wrong backend port, TLS termination failure | Inspect ingress rules, controller health/logs, Service endpoints and TLS secret; correct route or rollback config | Routes return expected responses; ingress errors decline |
| Kubernetes Service / endpoints | Service selects no Ready Pods, wrong selector or target port | Check selectors, EndpointSlices, ports, readiness probes and network policy | Ready backend endpoints available; requests reach application |
| EKS application Pod | CrashLoopBackOff, OOM, errors, not Ready | Check Pod events, application logs, CPU/memory, probes, dependencies; restart/replace or safely rollback | Pods Ready; critical API journeys pass; error rate/latency recover |
| Deployment / bad release | New version has high 5xx after rollout | Correlate failure with rollout; stop rollout; safe CI/CD rollback; investigate RCA afterward | Old-version success restored; 5xx near baseline; p95 within SLO |
| EKS worker node | Node NotReady, Pod eviction, insufficient CPU/memory | Replace unhealthy node; use scheduler and Karpenter/Cluster Autoscaler; avoid overpacking | Replacement Pods Running and Ready on healthy nodes |
| Entire AZ | AZ-A unavailable; lost Pods and zonal targets | Shift traffic to healthy AZs; ensure pre-running or pre-provisioned headroom; restore replicas; evaluate DB/cache failovers | Enough Ready Pods and backend targets in surviving AZs; customer SLIs within objectives |
| EKS control plane | Control-plane/API access issue | Verify AWS-managed EKS status; distinguish management-plane impairment from live application data plane | Cluster API/operations recover; workload and customer path observed separately |
| HPA / scheduler | HPA requests more Pods but Pods stay Pending | Inspect HPA metrics, requests, scheduler events, quotas and placement constraints; add node capacity | Pending Pods schedule and become Ready |
| Karpenter / Cluster Autoscaler | No new nodes or insufficient AZ placement | Check NodePool/node group limits, subnet/instance capacity, quotas, IAM, AZ requirements | Nodes join cluster; Pods scheduled in surviving AZs |
| Redis / ElastiCache primary | Primary down during AZ failure | Multi-AZ auto-promote healthy replica; reconnect; bound DB fallback during failover | Primary endpoint usable; hit ratio recovers; DB remains healthy |
| Redis cache | Evictions, hit ratio 90% → 30%, DB overload | Diagnose evictions/TTL/memory and capacity; rate limit DB fallback; warm hot keys gradually; consider cache scaling | Hit ratio, Redis memory/evictions and DB CPU/connection utilization normalize |
| Cache consistency | Stale document metadata after update | Invalidate on DB commit / correct cache key; use primary for strong-consistency reads where required | Read-after-write returns updated document |
| PostgreSQL primary | CPU 90–95%, high write latency | Identify bottleneck via slow queries, locks, IOPS and indexes; tune or scale primary; use controlled backpressure | DB write latency, CPU and transaction success normalize |
| PostgreSQL Multi-AZ failover | Primary AZ fails; broken in-flight DB connections | Managed standby promotion; reconnect via endpoint or RDS Proxy; bounded safe retries and idempotency | Primary writable; critical reads/writes succeed; error rate recovers |
| PostgreSQL read replicas | Replica lag, stale GET response | Route strong read-after-write requests to primary; monitor lag; use replicas only for eligible stale-tolerant reads | Fresh read from primary; acceptable replica lag |
| Connection pool / RDS Proxy | Pool exhaustion, wait timeout, 1,000 potential connections vs 300 DB max | Bound per-Pod pool; verify proxy behavior, acquisition timeout and DB budget; inspect slow queries; apply backpressure | Pool waits and active DB connections under limits; request latency normal |
| SQL indexes / queries | Sequential scan, slow owner listing, excessive writes | Inspect EXPLAIN ANALYZE; add owner_id or composite (owner_id, created_at DESC) index selectively; paginate results | Query plans improve; p95 queries faster; write overhead acceptable |
| SQS / async workers | Backlog, retries or write lag | Ensure writes are truly async; set bounded worker concurrency, DLQ and replay; increase real DB capacity for sustained overload | Queue age/backlog decreases; durable work completes without duplicates |
| OIDC identity provider | Login fails, authorization code/token exchange fails | Inspect IdP availability, redirect URI, PKCE, scopes and token endpoint; restore configuration | Fresh login and token exchange succeed |
| JWT / refresh token | Expired JWT; 401 | API Gateway rejects expired token; client refreshes securely via authorization server; reauthenticate if necessary | New token accepted; legitimate access works |
| Document permissions | Valid JWT but unauthorized access to private document | Enforce per-document permission in application or authorization service; return 403 or deliberate 404 | Authorized users succeed; unauthorized users learn no private information |
| S3 / object access | Document file unavailable or HTTP errors | Check storage reference, bucket policy, permissions, object state and network path; use safe retries | Expected object can be accessed over HTTPS |
| S3 retention / lifecycle | Premature deletion risk; high storage cost | Use compliant Object Lock when mandated; lifecycle transitions/Intelligent-Tiering for costs; controlled expiry | Protected object versions remain; lifecycle and retention effective |
| PostgreSQL metadata retention | Soft-deleted document improperly visible or purged early | Use is_deleted, deleted_at, retention_until; enforce authorization/retention; controlled purge after legal checks | User no longer sees deleted document; mandated retention preserved |
| AWS KMS | PendingDeletion or disabled key; RDS/S3 decryption risk | Confirm unauthorized change; cancel deletion; enable key; validate RDS; restrict IAM/key-policy permissions; alert on ScheduleKeyDeletion | KMS enabled and usable; RDS healthy; encrypted data accessible |
| Observability / alerting | 2% HTTP 5xx while p95 within target | Page on-call for sustained high burn; inspect error breakdown, trace affected endpoint, mitigate | 5xx returns near baseline; critical flows healthy and stable |
| Predictable traffic spike | 2k → 10k RPS at known time | Pre-scale HPA minimums and node capacity; run load tests and readiness checks | 10k-RPS load test passes with headroom |
| Unpredictable overload | Retry storm, overloaded dependencies, rising 5xx | Rate limit, timeout, backoff+jitter only where safe, circuit breakers, bulkheads, bounded concurrency and load shedding | Error budget burn falls; upstream and dependent services stabilize |
| Regional disaster (separate from AZ HA) | Entire Region unavailable | Invoke preplanned cross-Region DR with warm standby, DNS/traffic failover, DB replication/promotion and data-integrity checks; meet RPO/RTO | Customer journeys succeed in DR Region; measured RPO/RTO within targets |
