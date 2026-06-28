# Realistic System Design Micro-Lesson Plan

## Why This File Exists

The earlier system design plan had the right topics, but the sessions became too heavy. One “day” contained too many concepts, examples, cloud mappings, Kubernetes details, HA, security, observability, and interview answers.

That creates cognitive overload and makes the project difficult to continue consistently.

This file replaces the broad day-based approach with a **micro-lesson based plan**.

The rule is simple:

> One session = one small concept fully closed.

It does not matter how many calendar days it takes. What matters is that each session is realistic, retained, and written into revision notes.

---

## New Learning Rule

Do not try to complete a broad topic in one sitting.

Each session must cover only:

1. One concept
2. One small diagram
3. One AWS mapping
4. One Azure mapping
5. One Kubernetes/platform mapping
6. One production failure case
7. One interview answer
8. One revision note update

If a topic cannot be closed in one sitting, split it further.

---

## Session Completion Checklist

A micro-lesson is complete only when all of the following are done:

- Concept explained clearly
- One mobile-friendly diagram created
- AWS equivalent covered
- Azure equivalent covered
- Kubernetes/platform equivalent covered
- One failure scenario covered
- One security or observability point covered
- One small exercise answered
- Final interview-ready answer written
- `SYSTEM_DESIGN_REVISION_NOTES.md` updated

If the revision file is not updated, the session is not complete.

---

## Recommended Session Size

Each session should fit within practical learning time.

Target:

```text
5 min  - Recap previous session
15 min - Learn one concept
10 min - AWS/Azure/Kubernetes mapping
10 min - Failure/security/observability angle
10 min - Small exercise
10 min - Final answer and revision note
```

If the topic is still too large, split it again.

---

# Module 1 — Traffic Entry and API Exposure

## Goal

Understand how traffic enters cloud and Kubernetes platforms, and explain load balancer, API gateway, ingress, and gateway choices clearly in interviews.

## Micro-Lessons

### 1A — Load Balancer vs API Gateway Basics

Scope:

- What a load balancer does
- What an API gateway does
- Why they are not the same
- One basic architecture diagram

Exercise:

> Explain the difference between a load balancer and an API gateway in 5 lines.

Output:

- Final 2-minute interview answer
- Revision note update

---

### 1B — AWS ALB vs NLB

Scope:

- ALB Layer 7 routing
- NLB Layer 4 routing
- When to use ALB
- When to use NLB
- Common EKS usage

Exercise:

> Choose ALB or NLB for HTTP microservices, Kafka/TCP workload, and TLS passthrough.

Output:

- ALB vs NLB comparison table
- Revision note update

---

### 1C — Azure Load Balancer vs Application Gateway vs Front Door vs APIM

Scope:

- Azure Load Balancer for Layer 4
- Application Gateway for regional Layer 7
- Front Door for global edge/WAF
- API Management for API governance

Exercise:

> Map AWS ALB, NLB, CloudFront/WAF, and API Gateway to Azure equivalents.

Output:

- AWS/Azure mapping table
- Revision note update

---

### 1D — Kubernetes Ingress vs Gateway API

Scope:

- Ingress basics
- Gateway API basics
- GatewayClass, Gateway, HTTPRoute
- Why Gateway API is better for multi-tenant platforms

Exercise:

> Explain when you would use Ingress and when you would use Gateway API.

Output:

- Ingress vs Gateway API table
- Revision note update

---

### 1E — Internal Kubernetes Gateway vs External API Gateway

Scope:

- External API Gateway: AWS API Gateway, Azure APIM, Apigee
- Internal Kubernetes Gateway: Kong, Envoy, Istio, NGINX, Traefik
- Consumer governance vs service routing

Exercise:

> Explain when to use external API Gateway and when to use internal Kubernetes gateway.

Output:

- Final interview answer
- Revision note update

---

### 1F — HA for API Gateway and In-Cluster Gateway

Scope:

- HA for managed external API gateways
- HA for Kong/Envoy/Istio/NGINX in Kubernetes
- Multi-AZ, multi-region, replicas, PDB, HPA

Exercise:

> If the API gateway fails, what breaks and how do you make it highly available?

Output:

- Failure table
- HA design diagram
- Revision note update

---

### 1G — Final Traffic Entry Interview Answer

Scope:

- Combine 1A to 1F into one polished answer
- AWS design
- Azure design
- Kubernetes design
- Failure, security, observability, and trade-offs

Exercise:

> Design external traffic flow for an application running on both EKS and AKS.

Output:

- Final 5-minute interview answer
- Revision note update

---

# Module 2 — Scaling

## Goal

Understand scaling at compute, application, Kubernetes, serverless, database, cache, and queue layers without overloading one session.

## Micro-Lessons

### 2A — Vertical vs Horizontal Scaling

Scope:

- Scale up vs scale out
- When vertical scaling is acceptable
- Why horizontal scaling is preferred for stateless services

Exercise:

> Explain vertical vs horizontal scaling with one example each.

Output:

- Simple comparison table
- Revision note update

---

### 2B — VM Scaling: AWS ASG vs Azure VMSS

Scope:

- EC2 Auto Scaling Group
- Azure Virtual Machine Scale Sets
- Health checks
- Scaling policies
- Use cases for VM scaling

Exercise:

> When would you still use VM scaling instead of Lambda or Kubernetes?

Output:

- ASG vs VMSS comparison
- Revision note update

---

### 2C — Lambda vs EC2

Scope:

- Lambda use cases
- EC2 use cases
- Lambda max runtime
- Cold start, concurrency, timeout, and workload limits

Exercise:

> Why can’t we use Lambda for everything?

Output:

- Lambda vs EC2 decision table
- Revision note update

---

### 2D — Lambda HA and Serverless Reliability

Scope:

- Managed HA
- Reserved concurrency
- Provisioned concurrency
- Retry and DLQ
- Idempotency
- Multi-region serverless architecture

Exercise:

> How do you make Lambda-based APIs highly available?

Output:

- Serverless HA diagram
- Revision note update

---

### 2E — Kubernetes Pod Scaling: HPA and VPA

Scope:

- HPA for pod replicas
- VPA for resource recommendation/adjustment
- Resource requests and limits
- Metrics Server

Exercise:

> When will HPA help and when will it not help?

Output:

- HPA/VPA decision table
- Revision note update

---

### 2F — Kubernetes Node Scaling: Cluster Autoscaler, Karpenter, AKS Autoscaler

Scope:

- Pending pods
- Node capacity
- Cluster Autoscaler
- Karpenter
- AKS node pool autoscaling

Exercise:

> HPA scaled pods, but pods are pending. What happened?

Output:

- Pod scaling vs node scaling diagram
- Revision note update

---

### 2G — Event-Driven Scaling with KEDA

Scope:

- Queue depth
- Kafka lag
- SQS, Service Bus, Event Hubs
- Scale-to-zero workers

Exercise:

> How would workers scale when queue depth increases?

Output:

- KEDA event scaling diagram
- Revision note update

---

### 2H — Scaling Product Catalog API

Scope:

- 50,000 requests/minute
- Average RPS
- Peak RPS
- Cache
- HPA
- Node autoscaling
- DB protection

Exercise:

> Design a scaling strategy for a product catalog API receiving 50,000 requests per minute.

Output:

- Final scaling interview answer
- Revision note update

---

# Module 3 — Three-Tier Architecture

## Goal

Understand classic, cloud, serverless, and Kubernetes three-tier design step by step.

## Micro-Lessons

### 3A — What Is Three-Tier Architecture?

Scope:

- Presentation tier
- Application tier
- Data tier
- Why separation matters

Exercise:

> Explain three-tier architecture in 5 lines.

Output:

- Basic diagram
- Revision note update

---

### 3B — Three-Tier on AWS

Scope:

- Route 53
- CloudFront/WAF
- ALB
- EC2/ECS/EKS/Lambda
- RDS/Aurora/DynamoDB/S3/ElastiCache

Exercise:

> Map presentation, application, and data tiers to AWS services.

Output:

- AWS three-tier diagram
- Revision note update

---

### 3C — Three-Tier on Azure

Scope:

- Azure DNS
- Front Door/WAF
- Application Gateway
- App Service/AKS/VMSS/Functions
- Azure SQL/Cosmos DB/Blob Storage/Redis

Exercise:

> Map presentation, application, and data tiers to Azure services.

Output:

- Azure three-tier diagram
- Revision note update

---

### 3D — Three-Tier on Kubernetes

Scope:

- Ingress/Gateway API
- Frontend Service
- Backend Service
- Managed DB
- Cache
- Queue

Exercise:

> Design a Kubernetes three-tier application with frontend, backend, and data tier.

Output:

- Kubernetes diagram
- Revision note update

---

### 3E — Three-Tier on Serverless

Scope:

- Static frontend
- API Gateway/APIM
- Lambda/Azure Functions
- DynamoDB/Cosmos DB/S3/Blob

Exercise:

> When is serverless three-tier better than VM/Kubernetes three-tier?

Output:

- Serverless three-tier comparison
- Revision note update

---

### 3F — HA, Security, and Observability for Three-Tier

Scope:

- Multi-AZ
- Private subnets
- WAF
- TLS
- DB HA
- Metrics/logs/traces

Exercise:

> What can fail in a three-tier app and how do you protect it?

Output:

- Failure table
- Revision note update

---

### 3G — Final Three-Tier Interview Answer

Scope:

- Complete AWS/Azure/Kubernetes comparison
- Final polished answer

Exercise:

> Design the same three-tier application using AWS, Azure, and Kubernetes.

Output:

- Final 5-minute interview answer
- Revision note update

---

# Module 4 — Messaging and Event-Driven Architecture

## Goal

Understand messaging without mixing queues, pub/sub, streaming, retries, DLQ, and KEDA all in one sitting.

## Micro-Lessons

### 4A — Synchronous vs Asynchronous Communication

Exercise:

> When should a service call be synchronous and when should it be asynchronous?

---

### 4B — Queue vs Pub/Sub

Exercise:

> Use queue or pub/sub for notification, invoice generation, and audit logging.

---

### 4C — Streaming vs Messaging

Exercise:

> Choose Kafka/Event Hubs/Kinesis or SQS/Service Bus for different workloads.

---

### 4D — Retry, DLQ, and Idempotency

Exercise:

> Why must consumers be idempotent in messaging systems?

---

### 4E — Content-Based Messaging

Exercise:

> Route payment_failed, payment_success, and high_priority events.

---

### 4F — High-Priority Queuing

Exercise:

> Design queues for OTP SMS, order confirmation, and marketing email.

---

### 4G — Notification System Final Design

Exercise:

> Design a notification system that sends email, SMS, and push notifications.

---

# Module 5 — Databases, Storage, and Caching

## Micro-Lessons

### 5A — SQL vs NoSQL
### 5B — Aurora/RDS vs DynamoDB
### 5C — Azure SQL/PostgreSQL vs Cosmos DB
### 5D — CAP Theorem Basics
### 5E — Caching Basics
### 5F — Redis vs Memcached
### 5G — Cache Strategies: Cache-Aside, Write-Through, Write-Behind
### 5H — Hashing and Consistent Hashing
### 5I — Database Sharding
### 5J — Store and Retrieve Images
### 5K — URL Shortener Final Design

Each lesson must end with one exercise and revision note update.

---

# Module 6 — Reliability, HA, and DR

## Micro-Lessons

### 6A — High Availability vs Fault Tolerance
### 6B — RPO vs RTO
### 6C — Backup and Restore
### 6D — Pilot Light DR
### 6E — Warm Standby DR
### 6F — Active-Passive DR
### 6G — Active-Active DR
### 6H — Circuit Breaker, Retry, Timeout, Bulkhead
### 6I — Cell-Based Architecture
### 6J — Payment API Final Design

---

# Module 7 — Security and Governance

## Micro-Lessons

### 7A — Authentication vs Authorization
### 7B — IAM User, Group, Role vs Azure Entra/RBAC
### 7C — TLS vs mTLS
### 7D — Encryption at Rest vs In Transit
### 7E — Client-Side vs Server-Side Encryption
### 7F — Security Group vs NACL vs NSG
### 7G — IDS vs IPS
### 7H — Secrets Management in AWS, Azure, Kubernetes
### 7I — Kubernetes RBAC and NetworkPolicy
### 7J — 12-Factor App
### 7K — Secure Banking Kubernetes Platform Final Design

---

# Module 8 — Platform Engineering and Kubernetes System Design

## Micro-Lessons

### 8A — Single Cluster vs Multi-Cluster vs Hybrid Model
### 8B — Namespace and Tenant Isolation
### 8C — RBAC and NetworkPolicy Design
### 8D — ResourceQuota and LimitRange
### 8E — GitOps Onboarding Flow
### 8F — Developer Portal / Backstage Flow
### 8G — Secrets Management Platform
### 8H — Observability Platform
### 8I — Multi-Cluster Upgrade and Failure Strategy
### 8J — Kubernetes Platform for 10,000 Developers Final Design

---

# Module 9 — PCF/TAS to Kubernetes Migration

## Micro-Lessons

### 9A — PCF Org/Space to Kubernetes Namespace/Tenant
### 9B — PCF Route/Gorouter to Ingress/Gateway API
### 9C — Service Broker to Operators/External Secrets
### 9D — Manifest to Helm/Kustomize
### 9E — Loggregator to Observability Stack
### 9F — CI/CD Migration
### 9G — Blue-Green Cutover and Rollback
### 9H — 200-App Migration Final Design

---

# Module 10 — Product System Design Scenarios

These should be attempted only after the foundation modules are strong.

Each product design should be split into smaller sessions:

1. Requirements and scale
2. APIs and data model
3. High-level architecture
4. Data/storage/cache/messaging
5. Scaling and HA
6. Security and observability
7. Final interview answer

## Scenarios

- URL Shortener / TinyURL
- Notification System
- YouTube / Netflix / Prime Video
- Twitter
- WhatsApp / Telegram
- Uber
- Ticketmaster / BookMyShow
- Amazon / Flipkart
- OneDrive / Dropbox
- Shopify
- IoT System
- Parking Garage
- Gen AI System

---

# Backlog Cleanup Plan

Current broad sessions must be converted into micro-lessons.

## Already Partially Covered

- 1A Load Balancer vs API Gateway Basics
- 1D Ingress vs Gateway API
- 1E Internal vs External API Gateway
- 1F API Gateway HA
- 2C Lambda vs EC2
- 2D Lambda HA

## Still Needs Closure

- 1B ALB vs NLB exercise and revision
- 1C Azure LB/App Gateway/Front Door/APIM exercise and revision
- 1G Final traffic-entry answer
- 2A Vertical vs horizontal scaling
- 2B VM scaling ASG vs VMSS
- 2E HPA/VPA
- 2F Node autoscaling
- 2G KEDA
- 2H Final scaling answer
- 3A to 3G Three-tier architecture split properly
- 4A to 4G Messaging split properly

---

# Practical Rule Going Forward

Do not ask “which day are we in?”

Ask:

```text
Which micro-lesson are we closing?
```

The project should move like this:

```text
Learn small concept
Answer small exercise
Review answer
Write revision note
Close micro-lesson
Move to next micro-lesson
```

This is the realistic path for retention and interview readiness.
