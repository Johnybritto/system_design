# System Design Learning Plan

## Purpose

This repository is for learning system design step by step for a **Senior Platform Engineer / SRE / Kubernetes / Cloud / DevOps** profile.

The focus is not only generic product design. The plan combines:

- AWS system design
- Azure system design
- Kubernetes platform design
- CI/CD and GitOps
- Observability
- HA, DR, and reliability
- Security and governance
- PCF/TAS to Kubernetes migration
- Product-style system design scenarios
- Gen AI system design basics

---

## Target Interview Positioning

Use this positioning during interviews:

> I design and operate scalable platforms where application teams can deploy securely, reliably, and repeatedly.

The goal is to answer system design questions like a platform/SRE architect, not only as an application developer.

---

## Standard Answer Template

Use this structure for every system design question:

```text
1. Clarify requirements
2. State assumptions
3. Estimate scale
4. Define APIs / users / traffic flow
5. Draw high-level architecture
6. Explain core components
7. Explain data storage
8. Explain scaling
9. Explain reliability and failure handling
10. Explain security
11. Explain observability
12. Explain cost and trade-offs
13. Summarize final design
```

---

## Weekly Plan Overview

| Week | Focus Area | Main Outcome |
|---|---|---|
| Week 1 | Core architecture, traffic flow, scaling | Explain LB, API Gateway, scaling, and three-tier systems across AWS, Azure, and Kubernetes |
| Week 2 | Messaging, events, queues, real-time systems | Design async, event-driven, and WebSocket-based systems |
| Week 3 | Data, storage, caching, hashing, sharding | Choose correct DB/cache/storage and explain trade-offs |
| Week 4 | HA, DR, reliability, distributed systems, cost | Design resilient systems with RPO/RTO and multi-region thinking |
| Week 5 | Security, governance, IAM, encryption, 12-factor apps | Design secure cloud and Kubernetes platforms |
| Week 6 | Full product and platform scenarios | Practice complete interview-style designs |

---

# Week 1 — Core System Design, Traffic Flow, and Scaling

## Topics

- Functional vs non-functional requirements
- API and API Gateway
- Load Balancer vs API Gateway
- ALB vs NLB
- Azure Load Balancer vs Application Gateway vs Front Door
- Azure API Management
- L4 vs L7 load balancing
- Vertical vs horizontal scaling
- VM scaling, serverless scaling, and container scaling
- Real-world scaling interview tips
- Three-tier architecture
- Three-tier architecture on serverless
- Three-tier architecture on Kubernetes

## AWS Angle

- Route 53
- CloudFront
- AWS WAF
- ALB
- NLB
- API Gateway
- EC2 Auto Scaling
- Lambda
- EKS
- RDS / Aurora

## Azure Angle

- Azure DNS
- Azure Front Door
- Azure WAF
- Azure Load Balancer
- Azure Application Gateway
- Azure API Management
- VM Scale Sets
- Azure Functions
- AKS
- Azure SQL / PostgreSQL

## Kubernetes Angle

- Ingress Controller
- Gateway API
- Service type LoadBalancer
- ClusterIP
- NodePort
- HPA
- VPA
- Cluster Autoscaler
- Node pools

## Practice Exercises

1. Design a three-tier web application on AWS.
2. Design a three-tier web application on Azure.
3. Design a three-tier web application on Kubernetes.
4. Explain how external traffic reaches a pod in EKS.
5. Explain how external traffic reaches a pod in AKS.
6. Compare ALB vs NLB vs API Gateway.
7. Compare Azure Load Balancer vs Application Gateway vs API Management.

## Diagram: Three-Tier on Kubernetes

```text
User
 |
 v
DNS
 |
 v
CDN / WAF
 |
 v
Cloud Load Balancer
 |
 v
Ingress Controller / Gateway API
 |
 v
Frontend Service
 |
 v
Backend API Service
 |
 +--> Redis Cache
 +--> Database
 +--> Queue
```

---

# Week 2 — Messaging, Events, Queues, and Real-Time Systems

## Topics

- Synchronous vs event-driven architecture
- Queues vs Pub/Sub
- Streaming vs messaging
- Content-based messaging system
- High-priority queuing / messaging system
- Retry, DLQ, idempotency
- Backpressure
- WebSockets for server-to-client communication
- Chatbot communication patterns
- IoT ingestion basics

## AWS Angle

- SQS
- SNS
- EventBridge
- Kinesis
- MSK / Kafka
- API Gateway WebSocket
- Lambda
- Step Functions

## Azure Angle

- Azure Service Bus Queue
- Azure Service Bus Topic
- Azure Event Grid
- Azure Event Hubs
- Azure Web PubSub
- Azure SignalR Service
- Azure Functions
- Durable Functions

## Kubernetes Angle

- Worker deployments
- HPA based on queue depth
- KEDA autoscaling
- Kafka on Kubernetes
- RabbitMQ on Kubernetes

## Practice Exercises

1. Design a notification system.
2. Design a content-based messaging system.
3. Design a high-priority messaging system.
4. Design WhatsApp / Telegram style messaging.
5. Design a chatbot using WebSockets.
6. Design an IoT ingestion platform.

## Diagram: Notification System

```text
Client / App
 |
 v
API Gateway / APIM
 |
 v
Notification API
 |
 v
Queue / Topic / Event Bus
 |
 +--> Email Worker
 +--> SMS Worker
 +--> Push Worker
 |
 v
External Providers
```

---

# Week 3 — Data, Storage, Caching, Hashing, and Sharding

## Topics

- SQL vs NoSQL
- Aurora vs DynamoDB
- Azure SQL vs Cosmos DB
- CAP theorem
- Hashing
- Challenges of hashing
- Consistent hashing
- Database sharding
- Read replicas
- Partitioning
- Caching
- Redis vs Memcached
- Cache-aside pattern
- Write-through cache
- Write-behind cache
- TTL and cache invalidation
- Hot key problem
- Store and retrieve images
- Object storage patterns

## AWS Angle

- RDS
- Aurora
- DynamoDB
- S3
- ElastiCache Redis
- ElastiCache Memcached
- CloudFront
- OpenSearch

## Azure Angle

- Azure SQL
- Azure Database for PostgreSQL
- Cosmos DB
- Blob Storage
- Azure Cache for Redis
- Azure Front Door / CDN
- Azure AI Search

## Kubernetes Angle

- StatefulSets
- Persistent Volumes
- StorageClass
- CSI drivers
- External managed database access
- Secrets for DB credentials
- Connection pooling

## Practice Exercises

1. Design URL Shortener / TinyURL.
2. Design image upload and retrieval system.
3. Design OneDrive / Dropbox.
4. Design Twitter feed storage.
5. Design product catalog caching for e-commerce.
6. Explain consistent hashing with cache nodes.
7. Explain database sharding for large user tables.

## Diagram: Store and Retrieve Images

```text
Client
 |
 v
API Gateway
 |
 v
Image Metadata API
 |
 +--> Metadata DB
 |
 v
Object Storage: S3 / Blob Storage
 |
 v
CDN: CloudFront / Front Door
```

---

# Week 4 — Reliability, HA, DR, Distributed Systems, and Cost

## Topics

- High availability
- High availability vs fault tolerance
- Distributed computing basics
- Disaster Recovery: RPO vs RTO
- Backup and restore
- Pilot light
- Warm standby
- Active-passive
- Active-active
- Multi-AZ design
- Multi-region design
- Cell-based architecture
- Circuit breaker
- Retry with exponential backoff
- Bulkhead pattern
- Graceful degradation
- Chaos testing
- Performance optimization
- Cost optimization

## AWS Angle

- Multi-AZ RDS
- Route 53 failover
- AWS Global Accelerator
- AWS Backup
- S3 lifecycle policies
- Auto Scaling
- CloudWatch

## Azure Angle

- Availability Zones
- Azure Front Door
- Traffic Manager
- Azure Site Recovery
- Azure Backup
- Paired regions
- Azure Monitor
- Azure Advisor

## Kubernetes Angle

- Multi-AZ node groups
- PodDisruptionBudget
- Topology spread constraints
- Anti-affinity
- HPA
- Cluster Autoscaler
- Readiness and liveness probes
- Multi-cluster failover
- Velero backup

## Practice Exercises

1. Design a highly available payment API.
2. Design active-passive DR architecture.
3. Design active-active global architecture.
4. Design cell-based SaaS architecture.
5. Design cost-optimized Kubernetes platform.
6. Explain RTO and RPO with examples.

---

# Week 5 — Security, Governance, IAM, Encryption, and 12-Factor Apps

## Topics

- Authentication and authorization
- IAM user, role, and group
- AWS IAM vs Azure Entra ID / Azure RBAC
- Encryption at rest
- Client-side encryption
- Server-side encryption
- Encryption in transit with SSL/TLS/mTLS
- TLS vs mTLS
- IDS vs IPS
- Security Group vs NACL
- Azure NSG vs ASG
- WAF
- Secrets management
- Audit logging
- Image scanning
- Runtime security
- Twelve-Factor App
- Twelve-Factor App interview Q&A

## AWS Angle

- IAM
- Cognito
- KMS
- ACM
- WAF
- Shield
- GuardDuty
- Security Groups
- NACLs
- Secrets Manager
- CloudTrail
- ECR image scanning

## Azure Angle

- Microsoft Entra ID
- Azure RBAC
- Managed Identity
- Key Vault
- Azure WAF
- Defender for Cloud
- NSG
- Azure Firewall
- Private Link
- Azure Monitor logs
- Microsoft Defender for Containers

## Kubernetes Angle

- Kubernetes RBAC
- NetworkPolicy
- Pod Security Standards
- OPA Gatekeeper
- Kyverno
- External Secrets Operator
- Secrets Store CSI Driver
- Istio / Linkerd mTLS
- Admission controllers
- Audit logs

## Practice Exercises

1. Design secure login system.
2. Design secure multi-tenant Kubernetes platform for banks.
3. Design mTLS service-to-service platform.
4. Design secrets management for AKS and EKS.
5. Convert a legacy app to 12-factor app for Kubernetes.
6. Explain IDS vs IPS vs SG/NACL/NSG.

---

# Week 6 — Product and Platform System Design Scenarios

## Product Scenarios

1. Design YouTube / Netflix / Prime Video.
2. Design Twitter.
3. Design WhatsApp / Telegram / Snapchat.
4. Design Tinder.
5. Design Uber.
6. Design Fandango / Ticketmaster / LiveNation.
7. Design IoT System.
8. Design Shopify.
9. Design URL Shortener / TinyURL.
10. Design Parking Garage.
11. Design Amazon / Flipkart.
12. Design OneDrive / Dropbox.
13. Design Gen AI Systems.

## Platform Scenarios

1. Design Kubernetes platform for 10,000 developers.
2. Design CI/CD platform for 1,000 microservices.
3. Design observability platform for 1,000 microservices.
4. Design notification system.
5. Design cloud storage system.
6. Design PCF/TAS to Kubernetes migration.
7. Design secure banking Kubernetes platform.
8. Design Gen AI system on Kubernetes/cloud.

---

# Kubernetes Platform Deep Dive

## Recommended Model

For 10,000 developers, use a **hybrid multi-cluster model**:

- Shared platform services cluster
- Multiple workload clusters
- Separate prod and non-prod
- Tenant isolation by namespace/team
- GitOps-based deployment
- Centralized observability
- Centralized security and policy

## Diagram

```text
Developers
 |
 v
Developer Portal / Backstage
 |
 v
Application Git Repo
 |
 v
CI Pipeline: Jenkins / GitHub Actions / Azure DevOps
 |
 v
Container Registry: ECR / ACR
 |
 v
GitOps Repo
 |
 v
ArgoCD / Flux
 |
 v
Workload Kubernetes Clusters
 |
 +--> Dev Cluster
 +--> QA Cluster
 +--> Stage Cluster
 +--> Prod Cluster
```

## Shared Services

```text
Shared Platform Cluster
 |
 +--> ArgoCD / Flux
 +--> Vault / External Secrets / Key Vault / Secrets Manager
 +--> Prometheus / Grafana
 +--> Loki / Splunk / OpenSearch
 +--> OpenTelemetry / Jaeger / Tempo
 +--> Policy Engine: Kyverno / OPA
 +--> Ingress / Gateway API
 +--> Developer Portal
```

---

# CI/CD and GitOps Flow

```text
Developer
 |
 v
App Git Repo
 |
 v
CI Pipeline
 |
 +--> Unit Test
 +--> Build Image
 +--> Scan Image
 +--> Push to Registry
 |
 v
Update GitOps Repo
 |
 v
ArgoCD / Flux
 |
 v
Kubernetes Cluster
```

## Key Points

- CI builds and tests.
- CI pushes image to registry.
- GitOps repo stores desired state.
- ArgoCD/Flux deploys by pull model.
- Policy engine validates before deployment.
- Rollback happens through Git revert.

---

# Observability Platform

```text
Applications / Pods
 |
 +--> Metrics --> Prometheus --> Grafana
 |
 +--> Logs ----> Fluent Bit --> Loki / Splunk / OpenSearch
 |
 +--> Traces --> OpenTelemetry --> Jaeger / Tempo
 |
 +--> Alerts --> Alertmanager / PagerDuty / Teams
```

## Key Points

- Golden signals: latency, traffic, errors, saturation
- RED method: Rate, Errors, Duration
- USE method: Utilization, Saturation, Errors
- SLO-based alerting
- Dashboard per service, namespace, cluster, and platform
- Centralized logging with tenant isolation

---

# PCF/TAS to Kubernetes Migration

## Mapping

```text
PCF Org/Space      -> Kubernetes Namespace / Tenant
Route              -> Ingress / Gateway API
Service Broker     -> Operators / External Secrets
Manifest           -> Helm / Kustomize
Diego Cell         -> Worker Node
Gorouter           -> Ingress Controller / API Gateway
Loggregator        -> Prometheus / Loki / Splunk / Dynatrace
```

## Migration Strategy

1. Application assessment
2. Runtime and dependency mapping
3. Containerization approach
4. Service binding migration
5. Secrets migration
6. Logging and monitoring migration
7. CI/CD migration
8. Routing migration
9. Blue-green or canary cutover
10. Rollback plan

---

# Daily Study Format

Use this 1-hour structure:

```text
10 min  - Revise previous topic
20 min  - Learn one small concept
20 min  - Solve one practical design exercise
10 min  - Review and improve answer
```

---

# First 7 Days

## Day 1 — Load Balancer vs API Gateway

Exercise:

> Design external traffic flow for an application running on EKS and AKS.

## Day 2 — Scaling

Exercise:

> Design scaling strategy for a product catalog API receiving 50,000 requests per minute.

## Day 3 — Three-Tier Architecture

Exercise:

> Design the same three-tier application using AWS, Azure, and Kubernetes.

## Day 4 — Messaging Basics

Exercise:

> Design a notification system that sends email, SMS, and push notifications.

## Day 5 — Database and Caching

Exercise:

> Design URL Shortener with cache and database scaling.

## Day 6 — HA and DR

Exercise:

> Design a payment API with 99.99% availability.

## Day 7 — Security Basics

Exercise:

> Design secure access for a multi-tenant Kubernetes platform.

---

# Progress Files To Add Later

```text
SYSTEM_DESIGN_PROGRESS_LOG.md
SYSTEM_DESIGN_DAILY_EXERCISES.md
SYSTEM_DESIGN_ARCHITECTURE_NOTES.md
SYSTEM_DESIGN_INTERVIEW_SCENARIOS.md
SYSTEM_DESIGN_REVISION_NOTES.md
```

---

# Baseline Question

Before starting Day 1, answer this:

```text
For a Kubernetes platform used by 10,000 developers, would you choose:

A. One large shared Kubernetes cluster
B. Multiple Kubernetes clusters
C. Hybrid model with shared platform services and separate workload clusters

Explain the answer in 5–7 lines.
```
