# System Design Revision Notes

## Purpose

This file will store the final interview-ready answers after each daily deep-dive session.

Every day, once the topic is completed, the final answer, diagram, trade-offs, failure scenarios, AWS/Azure/Kubernetes comparison, and interview Q&A must be written here.

---

# Revision Note Template

Use this template for every day.

```markdown
## Day X — Topic Name

### Date

YYYY-MM-DD

### Topic

Topic name here.

### Why This Topic Matters

Explain why this topic is important for system design interviews and for Senior Platform Engineer / SRE / Kubernetes roles.

### Key Concepts

- Concept 1
- Concept 2
- Concept 3

### AWS Implementation

Explain the AWS version, including relevant services and design choices.

### Azure Implementation

Explain the Azure version, including relevant services and design choices.

### Kubernetes / Platform Engineering Implementation

Explain how this maps to Kubernetes, EKS, AKS, OpenShift, platform engineering, GitOps, or SRE operations.

### Architecture Diagram

Add mobile-friendly text diagram here.

### Data Flow / Request Flow

Step-by-step flow of how the system works.

### Scaling Strategy

Explain vertical scaling, horizontal scaling, autoscaling, queue scaling, database scaling, cache scaling, or cluster scaling as relevant.

### Failure Scenarios

List likely production failure modes and how to handle them.

### Reliability / HA / DR

Explain availability, fault tolerance, disaster recovery, RPO/RTO, multi-AZ, or multi-region design as relevant.

### Security Design

Explain authentication, authorization, IAM/RBAC, encryption, network controls, secrets management, and audit logging.

### Observability Design

Explain metrics, logs, traces, dashboards, alerts, SLOs, and operational checks.

### Cost and Operational Trade-Offs

Explain cost impact, complexity, operational burden, and when to avoid over-engineering.

### Interview Questions and Answers

#### Q1. Question?

Answer.

#### Q2. Question?

Answer.

#### Q3. Question?

Answer.

### Final Interview-Ready Answer

Write the polished answer here in the standard interview structure.

### Weak Areas to Revise

- Item 1
- Item 2
```

---

# Day 1 — Load Balancer vs API Gateway, External API Gateway vs Internal Kubernetes Gateway

## Date

2026-05-12

## Status

In progress. Core explanation captured. Practical exercise answer still pending.

## Topic

Load Balancer vs API Gateway, ALB vs NLB, Azure equivalents, Kubernetes Gateway API, and when to use an external API Gateway vs an internal Kubernetes gateway.

---

## Why This Topic Matters

This topic is important because platform engineers are often responsible for designing how traffic enters a platform and how APIs are exposed securely and reliably.

In interviews, it is not enough to say “I will use a load balancer.” A senior answer must explain which layer is used, why it is used, where API governance happens, where Kubernetes routing happens, and how the design behaves during failure.

---

## Key Concepts

- A load balancer distributes traffic to healthy backend targets.
- A Layer 4 load balancer routes TCP/UDP/TLS traffic without understanding HTTP paths.
- A Layer 7 load balancer understands HTTP/HTTPS and can route by host, path, headers, or methods.
- An external API Gateway manages API consumers, policies, authentication, throttling, API keys, quotas, and versioning.
- An internal Kubernetes gateway manages routing to Kubernetes Services and Pods.
- Kubernetes Gateway API is not the same as Apigee, AWS API Gateway, or Azure API Management.

---

## Load Balancer vs API Gateway

A load balancer is mainly a traffic distribution component.

An API Gateway is an API governance and control layer.

Simple memory line:

```text
Load Balancer = distributes traffic to healthy backends.
API Gateway = manages, secures, governs, and routes APIs.
```

A load balancer is used when the main requirement is traffic distribution, health checks, and high availability.

An API Gateway is used when the main requirement is API management, such as authentication, authorization, rate limiting, throttling, API keys, request validation, response transformation, API versioning, monitoring, and developer onboarding.

---

## AWS Implementation

### ALB — Application Load Balancer

Use ALB when the traffic is HTTP/HTTPS and the platform needs Layer 7 routing.

Good use cases:

- Host-based routing
- Path-based routing
- TLS termination
- Web application routing
- Exposing multiple microservices
- EKS ingress for HTTP/HTTPS services

Example:

```text
/app1 -> app1-service
/app2 -> app2-service
/api  -> api-service
```

### NLB — Network Load Balancer

Use NLB when the traffic is Layer 4 TCP/UDP/TLS and the platform needs high performance, static IPs, or TLS passthrough.

Good use cases:

- TCP services
- UDP services
- TLS passthrough
- Kafka-style endpoints
- Internal platform services
- Very low latency workloads
- Exposing an in-cluster gateway through a stable L4 entry point

### AWS API Gateway

Use AWS API Gateway when the API requires API-level governance.

Good use cases:

- Public APIs
- Partner APIs
- API keys
- Usage plans
- Throttling
- Request validation
- API versioning
- Lambda/serverless APIs
- WebSocket APIs

---

## Azure Implementation

### Azure Load Balancer

Use Azure Load Balancer for Layer 4 TCP/UDP load balancing.

Good use cases:

- Internal services
- Private AKS exposure
- TCP/UDP traffic
- Simple network load balancing

### Azure Application Gateway

Use Azure Application Gateway for Layer 7 HTTP/HTTPS routing.

Good use cases:

- Path-based routing
- Host-based routing
- TLS termination
- WAF integration
- Regional web application routing
- AKS ingress through Application Gateway Ingress Controller

### Azure Front Door

Use Azure Front Door when the application is global or internet-facing and needs edge routing.

Good use cases:

- Global entry point
- CDN-style acceleration
- WAF at edge
- Multi-region routing
- TLS termination at edge

### Azure API Management

Use Azure API Management when the API needs governance and lifecycle management.

Good use cases:

- API keys
- JWT validation
- Quotas
- Rate limits
- API versioning
- Developer portal
- Partner API onboarding
- Request/response transformation

---

## Kubernetes / Platform Engineering Implementation

In Kubernetes, the term “API Gateway” can be confusing.

There are two different ideas:

1. External API Gateway product
2. Internal Kubernetes gateway

Examples of external API gateway products:

- AWS API Gateway
- Azure API Management
- Apigee
- MuleSoft API Manager

Examples of internal Kubernetes gateway technologies:

- Kubernetes Gateway API
- Kong Gateway
- Envoy Gateway
- Istio Ingress Gateway
- NGINX Ingress / NGINX Gateway
- Traefik

Simple memory line:

```text
External API Gateway manages API consumers.
Internal Kubernetes Gateway manages Kubernetes service traffic.
```

---

## Ingress vs Kubernetes Gateway API

Ingress is the older Kubernetes resource for exposing HTTP/HTTPS services using host and path-based routing.

Gateway API is the newer and more expressive Kubernetes traffic routing model.

Ingress is fine for simple applications.

Gateway API is better for large multi-tenant platforms because it separates platform ownership from application routing.

### Ingress Model

```text
User
 |
 v
Load Balancer
 |
 v
Ingress Controller
 |
 v
Ingress Resource
 |
 v
Kubernetes Service
 |
 v
Pods
```

### Gateway API Model

```text
User
 |
 v
Load Balancer / Gateway Implementation
 |
 v
GatewayClass
 |
 v
Gateway
 |
 v
HTTPRoute / TCPRoute / GRPCRoute
 |
 v
Kubernetes Service
 |
 v
Pods
```

### Platform Ownership Model

Platform team owns:

- GatewayClass
- Gateway
- Listener configuration
- TLS policy
- Which namespaces can attach routes
- External or internal gateway exposure

Application team owns:

- HTTPRoute
- Service routing
- Path rules
- Canary weights
- Service-level routing rules

This model is better for platform engineering because it supports governance and self-service without giving every team full control over the shared entry point.

---

## Can We Use NLB / Azure Load Balancer with an Internal Kubernetes API Gateway?

Yes. This is a valid production pattern.

The cloud load balancer provides the external or internal Layer 4 entry point. The in-cluster gateway handles Layer 7 API routing and policies.

### AWS Pattern

```text
Internet / Internal Network
 |
 v
Route 53
 |
 v
AWS NLB
 |
 v
Kubernetes Service type LoadBalancer
 |
 v
Kong / Envoy / Istio / NGINX Gateway inside EKS
 |
 v
Kubernetes Services
 |
 v
Pods
```

### Azure Pattern

```text
Internet / Internal Network
 |
 v
Azure DNS
 |
 v
Azure Load Balancer
 |
 v
AKS Service type LoadBalancer
 |
 v
Kong / Envoy / Istio / NGINX Gateway inside AKS
 |
 v
Kubernetes Services
 |
 v
Pods
```

Use this pattern when:

- You want a simple Layer 4 cloud entry point.
- You want API routing controlled inside Kubernetes.
- You want portability across AWS, Azure, and on-prem.
- You use Kong, Envoy, Istio, NGINX, or Traefik as the standard platform gateway.
- You want GitOps-managed gateway configuration.
- You need TLS passthrough or TCP-level entry before Layer 7 processing inside the cluster.

For public internet-facing workloads, add edge protection before the Layer 4 load balancer:

```text
CloudFront + AWS WAF -> NLB -> In-cluster gateway
Azure Front Door + WAF -> Azure Load Balancer -> In-cluster gateway
```

---

## When to Use External API Gateway

Use an external API Gateway such as AWS API Gateway, Azure API Management, or Apigee when the API is exposed to:

- External customers
- Mobile apps
- Partner systems
- Third-party consumers
- Public internet users
- Multiple business units

Use it when you need:

- API keys
- OAuth/JWT validation
- Subscription plans
- Rate limiting
- Quotas
- API versioning
- Developer portal
- API documentation
- Request/response transformation
- Centralized API governance
- Partner onboarding

### External API Gateway Pattern

```text
External Client / Partner / Mobile App
 |
 v
External API Gateway: AWS API Gateway / Azure APIM / Apigee
 |
 v
Load Balancer / Private Link / VPC Link
 |
 v
EKS / AKS Services
```

---

## When to Use Internal Kubernetes Gateway

Use an internal Kubernetes gateway such as Kong, Envoy, Istio Gateway, NGINX, Traefik, or Gateway API when the main need is:

- Routing traffic to Kubernetes services
- Host/path/header-based routing
- Canary routing
- Blue-green routing
- mTLS inside the cluster
- Service-to-service traffic control
- GitOps-managed routing
- Tenant/team-level route ownership
- Platform-controlled ingress

### Internal Kubernetes Gateway Pattern

```text
Internal User / Internal App / External LB
 |
 v
Internal NLB / Azure Internal Load Balancer
 |
 v
Kubernetes Gateway / Kong / Istio / Envoy
 |
 v
Kubernetes Services
 |
 v
Pods
```

---

## When to Use Both

Use both only when each layer has a clear purpose.

Example enterprise pattern:

```text
Internet / Partner / Mobile
 |
 v
WAF / CDN / Edge Protection
 |
 v
External API Gateway: AWS API Gateway / Azure APIM / Apigee
 |
 v
Cloud Load Balancer
 |
 v
Internal Kubernetes Gateway: Kong / Envoy / Istio / NGINX
 |
 v
Kubernetes Services
 |
 v
Pods
```

External API Gateway responsibilities:

- API consumer authentication
- API keys
- Quotas
- Rate limits
- API versioning
- Developer portal
- Partner governance

Internal Kubernetes Gateway responsibilities:

- Routing to Kubernetes services
- Canary / blue-green traffic splitting
- mTLS / service mesh integration
- Internal retries and timeouts
- Service-level observability
- GitOps-managed routing

Avoid using both if the application only needs simple routing, because it creates two places to manage routing, TLS, timeouts, policies, logs, and troubleshooting.

---

## Architecture Diagram — Recommended Enterprise Pattern

```text
Users / Partners / Mobile Apps
 |
 v
DNS
 |
 v
CDN + WAF
 |
 v
External API Gateway / APIM / Apigee
 |
 v
Cloud Load Balancer: ALB/NLB/App Gateway/Azure LB
 |
 v
Internal Kubernetes Gateway: Kong/Envoy/Istio/NGINX
 |
 v
Kubernetes Services
 |
 v
Pods
 |
 +--> Cache
 +--> Database
 +--> Queue
```

---

## Scaling Strategy

External gateway scaling:

- Use managed gateway autoscaling where available.
- Use regional and multi-region deployment for high availability.
- Apply throttling and quotas to protect backend systems.

Cloud load balancer scaling:

- Use managed ALB/NLB/Application Gateway/Azure Load Balancer capacity.
- Spread targets across multiple Availability Zones.
- Use health checks to remove unhealthy backends.

Internal Kubernetes gateway scaling:

- Run multiple gateway replicas.
- Use HPA based on CPU, memory, request rate, or custom metrics.
- Use PodDisruptionBudget.
- Spread replicas across nodes and zones.
- Use resource requests and limits.

Backend service scaling:

- Use HPA for application workloads.
- Use queue-based autoscaling with KEDA where relevant.
- Use caching to reduce backend load.

---

## Failure Scenarios

| Failure Scenario | Impact | Mitigation |
|---|---|---|
| External API Gateway is down or misconfigured | API consumers cannot reach backend APIs | Multi-AZ/managed gateway, staged rollout, config validation |
| API Gateway throttling too strict | Valid users receive 429 errors | Correct quota planning and monitoring |
| Cloud load balancer target unhealthy | Traffic may fail or reduce capacity | Health checks and multi-AZ targets |
| In-cluster gateway pod fails | Routes through that pod fail | Multiple replicas, HPA, PDB, readiness probes |
| Wrong route configuration | Traffic goes to wrong service | GitOps review, route validation, staged rollout |
| TLS certificate expired | HTTPS traffic fails | Certificate automation and expiry alerts |
| WAF rule blocks valid traffic | Users receive blocked requests | Test WAF in monitor mode first |
| Backend pod unhealthy | Service may return errors | Readiness probes, retries, circuit breakers |

---

## Security Design

Security controls to mention:

- WAF at edge for public-facing workloads.
- TLS termination at edge, API Gateway, or in-cluster gateway depending on requirements.
- mTLS for service-to-service communication where high security is required.
- OAuth/JWT validation at external API Gateway or APIM.
- API keys and subscription plans for partner APIs.
- Kubernetes RBAC for route ownership.
- NetworkPolicy to restrict namespace-to-namespace traffic.
- Secrets stored in AWS Secrets Manager, Azure Key Vault, Vault, or External Secrets.
- Audit logs for gateway configuration changes.

---

## Observability Design

Monitor these metrics:

- Request count
- Latency p50/p95/p99
- 4xx errors
- 5xx errors
- 429 throttling errors
- Gateway upstream errors
- Target health
- TLS handshake errors
- WAF blocked requests
- Ingress/gateway controller errors
- Pod restarts
- HPA scaling events
- Saturation: CPU, memory, connections

Logging and tracing:

- Enable external API Gateway/APIM access logs.
- Enable ALB/NLB/Application Gateway logs.
- Enable in-cluster gateway access logs.
- Correlate request IDs across gateway and backend services.
- Use OpenTelemetry for distributed tracing.
- Send logs to CloudWatch, Azure Monitor, Splunk, Loki, or OpenSearch.

---

## Cost and Operational Trade-Offs

Using only external API Gateway:

- Good for API governance.
- Can become expensive for very high-volume simple traffic.
- Less Kubernetes-native for service routing.

Using only internal Kubernetes gateway:

- Good for platform-owned service routing.
- More portable across AWS, Azure, and on-prem.
- Requires the platform team to operate the gateway.
- May lack enterprise API lifecycle features like developer portal and usage plans.

Using both:

- Best for large enterprise API platforms.
- Provides both API governance and Kubernetes-native routing.
- Adds complexity because routing, TLS, timeouts, logs, and policies exist in multiple layers.
- Should be used only when responsibilities are clearly separated.

---

## Interview Questions and Answers

### Q1. Is API Gateway a specific product like Apigee?

API Gateway is both an architecture pattern and a product category. Apigee, AWS API Gateway, and Azure API Management are product implementations of the API Gateway/API management pattern.

### Q2. What is Kubernetes Gateway API?

Kubernetes Gateway API is a newer Kubernetes traffic routing model that uses resources like GatewayClass, Gateway, and HTTPRoute. It is mainly used to route traffic into Kubernetes Services. It is not the same as Apigee or Azure API Management.

### Q3. What is the difference between Ingress and Gateway API?

Ingress is the older and simpler Kubernetes resource for HTTP/HTTPS routing. Gateway API is newer, more expressive, and better suited for multi-tenant platforms because it separates infrastructure ownership from application routing.

### Q4. Can we use NLB or Azure Load Balancer and then use an API gateway inside Kubernetes?

Yes. NLB or Azure Load Balancer can act as a Layer 4 entry point, while an in-cluster gateway like Kong, Envoy, Istio, NGINX, or Traefik handles Layer 7 routing, authentication, rate limiting, canary routing, and observability.

### Q5. When should we use external API Gateway vs internal Kubernetes gateway?

Use external API Gateway for API consumers, partner APIs, API keys, quotas, throttling, versioning, and developer portal. Use internal Kubernetes gateway for routing traffic to Kubernetes services, canary routing, mTLS, GitOps-managed routing, and platform-level traffic control.

---

## Final Interview-Ready Answer

I would separate external API governance from internal Kubernetes service routing.

For external consumers such as mobile apps, partners, or public clients, I would use an external API Gateway such as AWS API Gateway, Azure API Management, or Apigee. This layer handles authentication, authorization, API keys, throttling, quotas, API versioning, developer onboarding, and centralized governance.

For Kubernetes service exposure, I would use an internal Kubernetes gateway such as Kong, Envoy Gateway, Istio Gateway, NGINX, Traefik, or Kubernetes Gateway API. This layer handles routing to Kubernetes Services, host/path/header-based rules, canary traffic splitting, blue-green routing, mTLS, retries, timeouts, and service-level observability.

In a simple internal platform, I may only use an internal Kubernetes gateway behind an internal NLB or Azure Load Balancer. For a large public enterprise API platform, I may use both: external API Gateway for API governance and internal Kubernetes gateway for service routing. I would avoid using both layers blindly because it increases operational complexity around TLS, routing, logging, timeouts, and troubleshooting.

The simple rule is: external API Gateway manages API consumers; internal Kubernetes gateway manages Kubernetes service traffic.

---

## Weak Areas to Revise

- Exact difference between ALB, NLB, Application Gateway, Azure Load Balancer, Front Door, and APIM.
- Gateway API YAML structure: GatewayClass, Gateway, HTTPRoute.
- When to use Ingress vs Gateway API in a multi-tenant platform.
- How TLS termination should be designed across edge, API gateway, load balancer, and in-cluster gateway.
- How to avoid duplicate policies when both external and internal gateways are used.

---

# Day 2 — To Be Filled

## Topic

Scaling: vertical vs horizontal, VM vs serverless vs container scaling.

## Status

Pending.

---

# Day 3 — To Be Filled

## Topic

Three-tier architecture on AWS, Azure, serverless, and Kubernetes.

## Status

Pending.

---

# Day 4 — To Be Filled

## Topic

Messaging basics: synchronous vs async, queue vs pub/sub, streaming vs messaging.

## Status

Pending.

---

# Day 5 — To Be Filled

## Topic

Database and caching: SQL vs NoSQL, Aurora vs DynamoDB, Azure SQL vs Cosmos DB, Redis vs Memcached.

## Status

Pending.

---

# Day 6 — To Be Filled

## Topic

High availability and disaster recovery: HA vs fault tolerance, RPO vs RTO, DR options.

## Status

Pending.

---

# Day 7 — To Be Filled

## Topic

Security basics: authentication, authorization, TLS, mTLS, encryption, SG/NACL/NSG.

## Status

Pending.
