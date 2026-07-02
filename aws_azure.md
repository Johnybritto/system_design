# AWS + Azure Platform Engineering Roadmap

**Audience:** Senior Platform Engineer / SRE / DevOps engineer moving toward AWS + Azure + Kubernetes + AI workload platform ownership.  
**Style:** Kubernetes-style day-wise roadmap with subtopics, practical patterns, review questions, and no intentional topic omissions.  
**Assumption:** 1 hour/day. Adjust pace as needed.  
**Rule:** Include all relevant topics first. Decide what to skip later.

---

## How to Review Every Topic

For every day, answer these eight questions:

1. What problem does this service/topic solve?
2. Why is it important for a platform/SRE engineer?
3. When should we use it?
4. When should we avoid it?
5. What is the simpler alternative?
6. What is the advanced/enterprise alternative?
7. What breaks in real life?
8. How would I explain it in an interview?

---

## Priority Tags

- **Must:** Required for serious cloud/platform/SRE interviews and real work.
- **Good:** Important, but can be learned after the Must layer.
- **Advanced:** Useful for senior/principal/platform architect discussions.
- **AI-Infra:** Required to connect cloud + Kubernetes + AI workloads.
- **Security:** Important for enterprise hardening.
- **FinOps:** Important for cost and platform ownership.

---

## Cloud Service Mapping Quick Reference

| Area | AWS | Azure |
|---|---|---|
| Organization | AWS Organizations, OUs, Accounts | Tenant, Management Groups, Subscriptions, Resource Groups |
| Identity | IAM, IAM Identity Center, STS | Microsoft Entra ID, Azure RBAC, Managed Identity |
| JIT Access | IAM Identity Center temporary elevated access | Entra PIM |
| Network | VPC | VNet |
| Routing | Route Tables, TGW | Route Tables/UDR, Virtual WAN |
| Private Access | PrivateLink, VPC Endpoints | Private Link, Private Endpoint |
| DNS | Route 53, Route 53 Resolver | Azure DNS, Azure Private DNS Resolver |
| Compute | EC2, ASG, Lambda | VM, VMSS, Azure Functions |
| Containers | EKS, ECS, Fargate, ECR | AKS, Container Apps, ACR |
| API Gateway | API Gateway | Azure API Management |
| Storage | S3, EBS, EFS, FSx | Blob, Managed Disk, Azure Files |
| Database | RDS, Aurora, DynamoDB | Azure SQL, PostgreSQL, Cosmos DB |
| AI/ML Platform | SageMaker AI, Bedrock | Azure ML, Azure AI Foundry, Azure OpenAI |
| Observability | CloudWatch, CloudTrail, X-Ray | Azure Monitor, Log Analytics, App Insights |
| IaC | Terraform, OpenTofu, CloudFormation, CDK | Terraform, OpenTofu, Bicep, ARM |
| Platform Control Plane | Crossplane | Crossplane |
| Security | KMS, GuardDuty, Security Hub, Inspector | Key Vault, Defender for Cloud, Sentinel |
| Chaos | AWS Fault Injection Service | Azure Chaos Studio |
| Cost | Cost Explorer, Budgets, Kubecost | Cost Management, Advisor, Kubecost/OpenCost |

---

# Phase 0 — Cloud Foundations

## Day 1 — Cloud Mental Model

**Goal:** Understand cloud as a control plane, not just servers.

**Must**
- Public cloud model
- Shared responsibility model
- Control plane vs data plane
- Region vs Availability Zone
- Edge location
- Account/subscription boundary
- Resource lifecycle
- Billing boundary
- Security boundary

**Good**
- Cloud-native vs lift-and-shift
- Pets vs cattle
- Managed service vs self-managed service

**Practical example**
- Draw how a user request reaches an application through DNS, load balancer, private compute, database, and monitoring.

**Checkpoint**
- Explain why cloud is not just “VMs in someone else’s data center.”

---

## Day 2 — AWS and Azure Resource Hierarchy

**Must**
- AWS Organization
- Organizational Unit
- AWS Account
- AWS Region
- AWS Resource
- Azure Tenant
- Azure Management Group
- Azure Subscription
- Azure Resource Group
- Azure Resource

**Good**
- Dev/test/prod account separation
- Security/shared-services/network account pattern
- Management group hierarchy

**Advanced**
- Landing zone hierarchy
- Centralized logging account/subscription
- Centralized network account/subscription

**Practical example**
- Design a company hierarchy with dev, test, prod, security, network, and shared services.

**Checkpoint**
- AWS account vs Azure subscription: what is similar and what is different?

---

## Day 3 — CLI, Profiles, Contexts, and Safe Access

**Must**
- AWS CLI
- Azure CLI
- AWS profiles
- Azure account/subscription context
- `aws sts get-caller-identity`
- `az account show`
- MFA basics

**Good**
- AWS SSO profile login
- Azure device code login
- Multiple subscriptions
- CLI output filtering

**Security**
- Avoid static keys
- Avoid credentials in shell history
- Avoid committing `.aws/credentials`

**Practical example**
- Switch between dev and prod profile/subscription safely.

**Checkpoint**
- How do you confirm which identity and subscription/account your terminal is using?

---

## Day 4 — Billing Safety and Tagging

**Must**
- AWS Budgets
- Azure Budgets
- Cost Explorer
- Azure Cost Management
- Cost allocation tags
- Resource cleanup
- Free tier limits

**Good**
- Budget alerts
- Forecasting
- Daily cost review
- Mandatory tags: owner, app, env, cost-center, data-classification

**FinOps**
- Idle public IPs
- Idle disks
- NAT gateway cost
- Load balancer cost
- EKS/AKS idle node cost
- GPU idle cost

**Practical example**
- Create a tagging standard for a platform team.

**Checkpoint**
- Which cloud resources can silently generate cost even when no app traffic exists?

---

## Day 5 — Cloud Service Equivalence Review

**Must**
- VPC vs VNet
- IAM vs Entra/RBAC
- EC2 vs Azure VM
- Lambda vs Azure Functions
- EKS vs AKS
- S3 vs Blob
- RDS/Aurora vs Azure SQL/PostgreSQL
- API Gateway vs Azure API Management
- CloudWatch vs Azure Monitor

**Good**
- ALB/NLB vs Application Gateway/Azure Load Balancer
- Transit Gateway vs Virtual WAN
- PrivateLink vs Private Endpoint
- SageMaker vs Azure ML
- Bedrock vs Azure OpenAI Service

**Practical example**
- Build a two-column AWS/Azure mapping sheet.

**Checkpoint**
- For each service pair, explain the problem solved, not just the name.

---

## Day 6 — Weekly Review: Cloud Foundations

**Review topics**
- Cloud hierarchy
- Shared responsibility
- Region/AZ
- Account/subscription design
- Tags and billing
- CLI identity validation

**Practice answer**
- “How would you structure cloud accounts/subscriptions for a platform team supporting multiple application teams?”

---

# Phase 1 — Identity, Governance, Federation, and Access Control

## Day 7 — AWS IAM Fundamentals

**Must**
- IAM user
- IAM group
- IAM role
- IAM policy
- Identity-based policy
- Resource-based policy
- Trust policy
- Permission policy
- STS
- AssumeRole

**Good**
- Permission boundaries
- Session policies
- IAM Access Analyzer
- Service-linked roles

**Security**
- Avoid IAM users for workloads
- Prefer temporary credentials
- Least privilege
- MFA for human users

**Practical example**
- Create a role that allows read-only S3 access and can be assumed by a trusted identity.

**Checkpoint**
- What is the difference between a trust policy and a permission policy?

---

## Day 8 — Azure Identity Fundamentals

**Must**
- Microsoft Entra ID
- User
- Group
- App registration
- Enterprise application
- Service principal
- Managed identity
- Azure RBAC
- Role assignment
- Scope: management group, subscription, resource group, resource

**Good**
- System-assigned managed identity
- User-assigned managed identity
- Entra roles vs Azure RBAC roles
- Conditional Access basics

**Practical example**
- Assign a VM managed identity permission to read a Key Vault secret.

**Checkpoint**
- Service principal vs managed identity: when do you use each?

---

## Day 9 — Least Privilege and Access Boundaries

**Must**
- Least privilege
- Deny by default
- AWS policy evaluation
- Azure RBAC scope inheritance
- Custom roles
- Resource-level access
- Break-glass account

**Good**
- AWS SCP
- Azure Policy deny effect
- Access reviews
- Privileged access review

**Practical example**
- Give a developer read access to prod logs without giving prod admin access.

**Checkpoint**
- How can access be limited by environment, resource type, and team ownership?

---

## Day 10 — AWS Organizations and Azure Management Groups

**Must**
- AWS Organizations
- OUs
- SCPs
- AWS accounts for isolation
- Azure Management Groups
- Azure Subscriptions
- Azure Policy assignment

**Good**
- Central security account
- Log archive account
- Shared network account
- Dev/test/prod subscriptions

**Advanced**
- Account vending
- Subscription vending
- Platform team guardrails

**Practical example**
- Prevent resource creation outside approved regions.

**Checkpoint**
- How does an SCP differ from IAM permissions?

---

## Day 11 — JIT Access, PIM, and Temporary Elevation

**Must**
- Permanent admin anti-pattern
- Just-in-Time access
- Time-bound privilege
- Approval-based elevation
- Azure Privileged Identity Management
- AWS IAM Identity Center temporary elevated access pattern
- Break-glass access

**Good**
- Access review workflow
- Audit evidence for privileged access
- Privileged role activation
- Ticket-linked elevation

**Security**
- No standing production admin
- No shared admin accounts
- Emergency-only break glass

**Practical example**
- Production incident requires DBA admin access for 1 hour with approval and audit trail.

**Checkpoint**
- Why are permanent admin roles risky in cloud environments?

---

## Day 12 — Human Identity Federation

**Must**
- External IdP federation
- Okta / Entra ID as source of truth
- AWS IAM Identity Center federation
- SAML
- SCIM provisioning
- Azure RBAC with Entra groups
- Group-based access

**Good**
- MFA enforcement
- Conditional Access
- Identity lifecycle management
- Joiner/mover/leaver process

**Advanced**
- Multi-cloud identity model
- Central identity provider with AWS and Azure access

**Practical example**
- User joins Platform team group in Entra/Okta and automatically receives AWS read-only access and Azure contributor access to dev.

**Checkpoint**
- Why should user lifecycle not be managed manually in every cloud account?

---

## Day 13 — Workload Identity Federation

**Must**
- OIDC federation
- AWS STS AssumeRoleWithWebIdentity
- AWS IAM OIDC provider
- Azure workload identity federation
- Entra app registration
- Federated credential
- GitHub Actions OIDC
- Azure DevOps OIDC pattern

**Good**
- No static cloud keys in CI/CD
- Short-lived tokens
- Trust conditions
- Repository/environment-based claims

**Practical example**
- Azure DevOps runner deploys to AWS without AWS access keys using OIDC + STS.

**Checkpoint**
- How can a CI/CD pipeline deploy to AWS and Azure without storing long-lived secrets?

---

## Day 14 — Week Review: Identity and Governance

**Review topics**
- IAM vs Azure RBAC
- Human federation
- Workload federation
- JIT/PIM
- SCP/Azure Policy
- Break-glass
- Temporary credentials

**Practice answer**
- “Design identity access for a company using Entra ID, AWS accounts, Azure subscriptions, and GitHub Actions pipelines.”

---

# Phase 2 — Networking, DNS, Hybrid, IPv6, and Multi-Cloud Connectivity

## Day 15 — CIDR, Subnets, and IP Planning

**Must**
- CIDR
- Subnet sizing
- Public subnet
- Private subnet
- Route table
- Default route
- IP exhaustion
- Overlapping CIDR issue

**Good**
- RFC1918 ranges
- Subnet reservation
- Environment-level CIDR planning
- Kubernetes pod/service CIDR planning

**Advanced**
- Multi-cloud CIDR planning
- On-prem overlap handling
- IPAM

**Practical example**
- Allocate non-overlapping CIDRs for AWS dev/prod, Azure dev/prod, and on-prem.

**Checkpoint**
- Why is overlapping CIDR a major problem in hybrid and multi-cloud networks?

---

## Day 16 — AWS VPC Core

**Must**
- VPC
- Subnet
- Route table
- Internet Gateway
- NAT Gateway
- Elastic IP
- Security Group
- NACL
- VPC Flow Logs

**Good**
- VPC endpoints
- Gateway endpoint
- Interface endpoint
- Private subnet routing

**Practical example**
- Public subnet route: `0.0.0.0/0 -> Internet Gateway`. Private subnet route: `0.0.0.0/0 -> NAT Gateway`.

**Checkpoint**
- Why does a private subnet still sometimes need outbound internet?

---

## Day 17 — Azure VNet Core

**Must**
- VNet
- Subnet
- Route table / UDR
- NAT Gateway
- Public IP
- NSG
- ASG
- Network Watcher
- NSG Flow Logs

**Good**
- Azure Firewall
- Private Endpoint
- Service Endpoint
- Private DNS Zone

**Practical example**
- VM with no public IP uses Azure Bastion for admin access and NAT Gateway for outbound internet.

**Checkpoint**
- NSG vs Azure Firewall: when do you need each?

---

## Day 18 — Private Subnet App Pattern

**Must**
- Private app server
- Public load balancer
- Private database
- NAT outbound
- No public IP on app server
- Bastion / SSM / VPN access

**AWS Practical Pattern**
- Internet -> ALB in public subnet -> EC2 in private subnet -> RDS in DB subnet
- EC2 outbound -> NAT Gateway -> Internet Gateway -> Internet
- Admin -> SSM Session Manager or Bastion -> EC2 private IP

**Azure Practical Pattern**
- Internet -> Application Gateway / Azure Load Balancer -> VM in private subnet -> DB private endpoint
- VM outbound -> NAT Gateway -> Internet
- Admin -> Azure Bastion -> VM private IP

**Checkpoint**
- How do you access a private server and still allow it to download patches?

---

## Day 19 — Security Groups, NACLs, NSGs, Firewalls

**Must**
- AWS Security Group
- AWS NACL
- Azure NSG
- Azure ASG
- Stateful vs stateless
- Inbound/outbound rules

**Good**
- Azure Firewall
- AWS Network Firewall
- Central inspection
- Egress filtering

**Advanced**
- L7 filtering
- IDS/IPS
- NVA pattern

**Practical example**
- App subnet allows inbound only from load balancer security group / subnet.

**Checkpoint**
- Security Group vs NACL vs NSG: what is the difference?

---

## Day 20 — PrivateLink, Private Endpoint, and Private Access

**Must**
- AWS PrivateLink
- Interface VPC Endpoint
- Gateway VPC Endpoint
- Azure Private Link
- Azure Private Endpoint
- Private DNS record
- Public network access disabled

**Good**
- Centralized endpoint pattern
- Private SaaS access
- Storage private access
- Database private access

**Advanced**
- Endpoint policy
- Cross-account endpoint service
- Cross-subscription private endpoint

**Practical example**
- App in private subnet accesses S3/Blob or database privately without public internet.

**Checkpoint**
- Why does private endpoint require DNS design, not just network design?

---

## Day 21 — The Transitive Routing Trap

**Must**
- VPC peering non-transitive routing
- VNet peering non-transitive routing
- PrivateLink reachability limits
- Private Endpoint reachability boundaries
- Hub-spoke routing limitation

**Good**
- NVA routing
- Transit Gateway
- Virtual WAN
- Route Server
- Central proxy pattern

**Advanced**
- Centralized PrivateLink pattern
- Centralized Private Endpoint with DNS forwarding
- Route propagation limitations

**Practical example**
- VNet-A peers with Hub, Hub peers with VNet-B. A cannot automatically route through Hub to B unless explicit routing architecture supports it.

**Checkpoint**
- Why is “we have peering” not the same as “everything can route everywhere”?

---

## Day 22 — Hybrid DNS Resolution

**Must**
- Public DNS
- Private DNS
- Split-horizon DNS
- Conditional forwarding
- Route 53 Resolver inbound endpoint
- Route 53 Resolver outbound endpoint
- Azure Private DNS Resolver inbound endpoint
- Azure Private DNS Resolver outbound endpoint
- DNS forwarding ruleset

**Good**
- On-prem to cloud DNS
- Cloud to on-prem DNS
- Cross-cloud private DNS
- Private endpoint DNS

**Advanced**
- Central DNS hub
- DNS firewall
- DNS query logging
- Multi-account DNS sharing

**Practical example**
- On-prem app resolves private RDS endpoint in AWS and private PostgreSQL endpoint in Azure.

**Checkpoint**
- Why are private zones alone not enough in hybrid architecture?

---

## Day 23 — AWS to Azure Interconnect

**Must**
- Multi-cloud VPN
- AWS Site-to-Site VPN
- Azure VPN Gateway
- Direct Connect
- ExpressRoute
- BGP
- Route propagation

**Good**
- Transit Gateway to Virtual WAN
- Colo/carrier interconnect
- Firewall inspection between clouds
- Cross-cloud DNS

**Advanced**
- Active-active VPN
- Dual carrier design
- Latency-aware routing
- Data transfer cost
- Multi-cloud identity and logging

**Practical example**
- Azure workload calls private API hosted in AWS through VPN/Direct Connect/ExpressRoute design.

**Checkpoint**
- How do you connect AWS and Azure privately without exposing services to internet?

---

## Day 24 — IPv6 and Dual-Stack Design

**Must**
- IPv4 exhaustion
- IPv6 CIDR
- Dual-stack VPC/VNet
- IPv6 DNS records
- Egress-only internet gateway
- NAT64 / DNS64 concept

**Good**
- EKS IPv6 mode
- AKS dual-stack
- IPv6 load balancer
- IPv6 security rules

**Advanced**
- IPv6-only pods
- IPv4/IPv6 transition strategy
- Carrier/public IPv4 pricing pressure

**Practical example**
- Design new cloud network with future IPv6 expansion and avoid burning IPv4 ranges.

**Checkpoint**
- Why is IPv6 becoming important for cloud and Kubernetes platforms?

---

## Day 25 — Week Review: Networking

**Review topics**
- VPC/VNet
- Subnets
- NAT
- PrivateLink/Private Endpoint
- DNS Resolver
- Hybrid DNS
- Multi-cloud interconnect
- IPv6
- Transitive routing trap

**Practice answer**
- “Design private connectivity from on-prem to AWS and Azure with private DNS resolution and no public database access.”

---

# Phase 3 — Compute, Serverless, Hardware, and Metadata Security

## Day 26 — Compute Decision Model

**Must**
- EC2 vs VM
- Lambda vs Azure Functions
- Containers
- EKS/AKS
- App Service / App Runner
- Batch jobs
- Stateful vs stateless
- Event-driven vs long-running

**Good**
- Operational burden
- Scaling model
- Cost model
- Runtime control

**Practical example**
- Choose between Lambda, EC2, ECS, EKS, Azure Functions, VM, AKS for a web API, cron job, and ML inference service.

**Checkpoint**
- If Lambda exists, why do we still need EC2?

---

## Day 27 — AWS EC2 Deep Dive

**Must**
- Instance families
- AMI
- EBS
- Instance profile
- Security Group
- User data
- ENI
- Auto Scaling Group
- Launch template

**Good**
- Spot instances
- Reserved Instances
- Savings Plans
- Placement groups

**Security**
- No public SSH by default
- SSM Session Manager
- IMDSv2

**Practical example**
- Launch EC2 in private subnet with instance profile and SSM access.

**Checkpoint**
- How does EC2 get temporary AWS credentials without static keys?

---

## Day 28 — Azure VM Deep Dive

**Must**
- VM size
- Managed Disk
- Image
- VM extension
- Managed identity
- NSG
- Availability Set
- Availability Zone
- VM Scale Set

**Good**
- Azure Bastion
- Azure Update Manager
- Spot VMs
- Custom images

**Practical example**
- Private Azure VM with managed identity, Azure Bastion access, and Key Vault secret read.

**Checkpoint**
- How does Azure VM access Key Vault without storing credentials?

---

## Day 29 — Lambda and Azure Functions

**Must**
- Event-driven compute
- Function runtime
- Trigger
- Cold start
- Timeout
- Memory/CPU sizing
- IAM role / managed identity
- API trigger
- Queue trigger
- Schedule trigger

**Good**
- Concurrency
- Dead-letter queue
- VPC integration
- Durable Functions basics

**Practical example**
- File uploaded to object storage triggers function for metadata extraction.

**Checkpoint**
- When is serverless a bad fit?

---

## Day 30 — API Gateway and Azure API Management

**Must**
- AWS API Gateway
- Azure API Management
- REST API
- HTTP API
- WebSocket API
- API keys
- JWT/OAuth validation
- Rate limiting
- Throttling
- Request/response transformation

**Good**
- Private API
- Internal API gateway
- Developer portal
- API versioning
- Backend integration
- Lambda/Function integration
- EKS/AKS backend integration

**AI-Infra**
- AI model gateway
- Token quota
- Per-team inference throttling
- Bedrock/Azure OpenAI proxy pattern

**Practical example**
- Client -> API Gateway/APIM -> Lambda/Function -> Queue/Database.

**Checkpoint**
- Why do Lambda/Functions usually need API gateway or event gateway around them?

---

## Day 31 — Instance Metadata and Credential Theft Risk

**Must**
- AWS Instance Metadata Service
- IMDSv1 vs IMDSv2
- SSRF risk
- Instance profile credential theft
- Metadata hop limit
- Azure Instance Metadata Service
- Managed identity endpoint

**Good**
- Block metadata from containers where not needed
- Require IMDSv2
- Restrict metadata access
- Pod identity boundaries

**Security**
- SSRF to cloud credential theft
- Metadata token protection
- Least privilege instance role

**Practical example**
- Web app SSRF attempts to query metadata endpoint. Explain how IMDSv2 and least privilege reduce blast radius.

**Checkpoint**
- Why is metadata service security an SRE interview topic?

---

## Day 32 — Image, Bootstrap, and Patching Strategy

**Must**
- Golden image
- AMI
- Azure image
- Packer
- User data
- Cloud-init
- VM extension
- Immutable infrastructure
- Patch management

**Good**
- Rolling replacement
- Blue/green VM deployment
- Drift detection
- Configuration management

**Practical example**
- Bake base image with security agents, bootstrap app config at startup.

**Checkpoint**
- What should be baked into image vs configured at runtime?

---

## Day 33 — ARM64 vs x86 Compute

**Must**
- x86
- ARM64
- AWS Graviton
- Azure Ampere Altra
- Compatibility testing
- Multi-arch container image
- Native dependency risk

**Good**
- Java/Go/Python on ARM
- Helm nodeSelector/affinity for arm64
- Mixed node pools
- CI multi-arch build

**FinOps**
- ARM price/performance
- Workload benchmarking
- Migration decision

**Practical example**
- Build container image for both amd64 and arm64, deploy to mixed-node cluster.

**Checkpoint**
- Why can ARM64 reduce cost, and what can break during migration?

---

## Day 34 — Confidential Computing

**Must**
- Confidential computing
- Encryption in use
- Trusted Execution Environment
- AMD SEV-SNP
- Intel SGX
- Intel TDX
- AWS Nitro Enclaves
- Azure Confidential VMs

**Good**
- Attestation
- Confidential containers
- Regulated workloads
- Data-in-use protection

**Security**
- Runtime memory protection
- High-trust workload isolation

**Practical example**
- Sensitive financial computation runs on confidential VM with attestation and encrypted memory.

**Checkpoint**
- How is confidential computing different from encryption at rest and in transit?

---

## Day 35 — GPU Compute and Hardware for AI Workloads

**Must**
- AWS P5/P5e/P4/G5/G6
- Azure ND/NC series
- NVIDIA H100/A100/L4/T4 concepts
- CUDA
- GPU drivers
- GPU quota
- GPU availability zone constraints

**Good**
- MIG
- GPU sharing
- Spot GPU
- GPU autoscaling
- GPU utilization monitoring

**AI-Infra / FinOps**
- Cost per GPU hour
- Cost per token
- Idle GPU detection
- Reserved GPU capacity

**Practical example**
- Choose GPU instance family for fine-tuning, batch inference, and real-time LLM inference.

**Checkpoint**
- Why is GPU quota planning a platform responsibility?

---

## Day 36 — Week Review: Compute

**Review topics**
- EC2/VM
- Lambda/Functions
- API Gateway/APIM
- Metadata security
- ARM64
- Confidential compute
- GPU compute

**Practice answer**
- “Choose a compute platform for a legacy app, serverless API, Kubernetes workload, and AI inference service.”

---

# Phase 4 — Load Balancing, DNS, Edge, WAF, and Traffic Management

## Day 37 — L4 vs L7 Load Balancing

**Must**
- L4 TCP/UDP load balancing
- L7 HTTP/HTTPS load balancing
- Listener
- Backend/target group
- Health check
- TLS termination
- Internal vs external load balancer

**Good**
- Sticky sessions
- Weighted routing
- Host/path routing
- mTLS basics

**Practical example**
- TCP database proxy uses L4; web app uses L7 with host/path rules.

**Checkpoint**
- ALB vs NLB vs Azure Load Balancer vs Application Gateway.

---

## Day 38 — AWS Load Balancing

**Must**
- Application Load Balancer
- Network Load Balancer
- Gateway Load Balancer
- Target group
- Listener rule
- Health check
- AWS Certificate Manager

**Good**
- Internal ALB/NLB
- Cross-zone load balancing
- TLS policy
- AWS WAF integration

**Practical example**
- Internet -> ALB -> private EC2 target group.

**Checkpoint**
- What causes “targets unhealthy” behind an ALB?

---

## Day 39 — Azure Load Balancing

**Must**
- Azure Load Balancer
- Application Gateway
- Azure Front Door
- Backend pool
- Health probe
- Listener
- Routing rule
- WAF policy

**Good**
- Internal load balancer
- Application Gateway for Containers
- App Gateway Ingress Controller

**Practical example**
- Internet -> Application Gateway WAF -> AKS/private VM backend.

**Checkpoint**
- Azure Load Balancer vs Application Gateway vs Front Door.

---

## Day 40 — DNS and Global Traffic

**Must**
- Route 53
- Azure DNS
- Public DNS zone
- Private DNS zone
- A record
- CNAME
- Alias
- TTL
- DNS caching

**Good**
- Weighted routing
- Failover routing
- Latency routing
- Private DNS forwarding

**Practical example**
- Blue/green release using weighted DNS and load balancer target groups.

**Checkpoint**
- Why does DNS change not immediately reflect for all users?

---

## Day 41 — CDN, WAF, and DDoS

**Must**
- CloudFront
- Azure Front Door
- CDN
- AWS WAF
- Azure WAF
- DDoS protection

**Good**
- Rate limiting
- Bot protection
- Geo restriction
- Cache behavior
- Origin protection

**Practical example**
- Users -> CDN/WAF -> Load balancer -> private app.

**Checkpoint**
- What should be cached at CDN and what should not?

---

## Day 42 — Week Review: Traffic Flow

**Review topics**
- L4/L7
- ALB/NLB
- App Gateway/Front Door
- DNS
- CDN
- WAF
- Health checks

**Practice answer**
- “Troubleshoot a public app that returns 502 behind a load balancer.”

---

# Phase 5 — Storage, Backup Storage, and Data Access Patterns

## Day 43 — Object Storage

**Must**
- S3
- Azure Blob Storage
- Bucket/container
- Object key
- Storage account
- Public access block
- Versioning
- Lifecycle policy
- Encryption

**Good**
- Static website hosting
- Object lock
- Immutability
- Access tiers
- Cross-region replication

**Practical example**
- Store app reports, logs, backups, and images in object storage with lifecycle rules.

**Checkpoint**
- Why is object storage not the same as a mounted disk?

---

## Day 44 — Block Storage

**Must**
- EBS
- Azure Managed Disk
- Disk type
- IOPS
- Throughput
- Snapshot
- Resize
- Encryption

**Good**
- Multi-attach limitations
- Disk performance tuning
- Snapshot backup policy

**Practical example**
- VM database needs high IOPS block disk and snapshot backup.

**Checkpoint**
- Block storage vs object storage.

---

## Day 45 — File Storage

**Must**
- EFS
- Azure Files
- NFS
- SMB
- Shared filesystem
- Mount targets
- Private access

**Good**
- FSx
- Azure NetApp Files
- Performance modes
- Access control

**Practical example**
- Multiple pods/VMs share a file mount for uploaded content.

**Checkpoint**
- When do you need shared file storage instead of object storage?

---

## Day 46 — Secure Storage Access

**Must**
- IAM role access to S3
- Managed identity access to Blob
- Bucket policy
- Storage account firewall
- Private endpoint
- VPC endpoint
- Encryption with KMS/Key Vault

**Good**
- Customer-managed keys
- SAS token risk
- Presigned URL
- Access logging

**Practical example**
- Private app uploads files to object storage without static keys.

**Checkpoint**
- How can an app access storage securely without credentials in config?

---

## Day 47 — Backup and Archive Storage

**Must**
- Backup storage
- Snapshot
- Versioning
- Retention
- Archive tier
- Restore testing
- Immutability

**Good**
- Legal hold
- Object lock
- Cross-region backup
- Ransomware-resistant backup

**Practical example**
- Move logs older than 90 days to archive and keep backup immutable for 1 year.

**Checkpoint**
- Why is backup incomplete until restore is tested?

---

## Day 48 — Week Review: Storage

**Review topics**
- Object/block/file
- Private access
- Encryption
- Lifecycle
- Backup
- Archive

**Practice answer**
- “Choose storage type for VM disk, shared app data, static images, logs, and backups.”

---

# Phase 6 — Database, Messaging, Modern Data Tier, and GenAI Data Foundations

## Day 49 — Managed SQL Databases

**Must**
- RDS
- Aurora
- Azure SQL
- Azure Database for PostgreSQL
- Azure Database for MySQL
- Backup
- Read replica
- Multi-AZ / zone redundancy
- Maintenance window

**Good**
- Parameter groups
- Performance Insights
- Query performance insights
- Private endpoint

**Practical example**
- App in private subnet connects to managed PostgreSQL through private endpoint/security group.

**Checkpoint**
- Managed DB vs self-hosted DB: when do you choose each?

---

## Day 50 — Database Connection Pooling and Proxy Layer

**Must**
- Connection pooling
- RDS Proxy
- PgBouncer
- Connection storm
- Lambda to RDS problem
- Kubernetes app to Postgres problem

**Good**
- Max connections
- Idle connections
- Pool sizing
- Read/write split
- Proxy observability

**Practical example**
- Serverless function burst creates too many DB connections; solve with RDS Proxy or PgBouncer.

**Checkpoint**
- Why can autoscaling apps overload a database even when CPU is low?

---

## Day 51 — NoSQL and Distributed Data

**Must**
- DynamoDB
- Cosmos DB
- Partition key
- Throughput
- Consistency
- TTL
- Global replication
- Backup

**Good**
- Hot partition
- Single-table design basics
- Cosmos DB APIs
- RU/s planning

**Practical example**
- User session metadata stored in globally distributed NoSQL store.

**Checkpoint**
- Why is partition key design critical in NoSQL?

---

## Day 52 — Global Database Topology

**Must**
- Aurora Global Database
- Primary write region
- Secondary read-only regions
- Storage-level replication
- Cross-region failover
- Cosmos DB global distribution
- Multi-region writes
- Conflict handling

**Good**
- RPO/RTO
- Read-local/write-primary pattern
- Write-local multi-master pattern
- Consistency tradeoffs

**Practical example**
- Compare global SaaS app using Aurora Global Database vs Cosmos DB multi-region writes.

**Checkpoint**
- Why is Aurora Global Database not the same as Cosmos DB multi-master writes?

---

## Day 53 — Messaging and Event Services

**Must**
- SQS
- SNS
- EventBridge
- Kinesis basics
- Azure Service Bus
- Event Grid
- Event Hubs
- Storage Queue

**Good**
- Queue vs pub/sub vs event streaming
- Dead-letter queue
- Retry policy
- Ordering
- Backpressure

**Practical example**
- API writes request to queue; worker processes asynchronously; failed messages go to DLQ.

**Checkpoint**
- When do you use queue, pub/sub, or event stream?

---

## Day 54 — Cache and Performance Layer

**Must**
- ElastiCache
- Azure Cache for Redis
- Cache-aside pattern
- TTL
- Eviction
- Session cache
- Rate-limit cache

**Good**
- Redis cluster
- Persistence
- Cache stampede
- Hot key

**Practical example**
- Reduce DB read load using Redis cache-aside pattern.

**Checkpoint**
- What can go wrong when cache and database are inconsistent?

---

## Day 55 — Vector Databases and Embeddings

**Must**
- Embeddings
- Vector search
- Similarity search
- Hybrid search
- pgvector
- RDS PostgreSQL with pgvector
- Aurora PostgreSQL with pgvector
- Azure Database for PostgreSQL pgvector
- Azure AI Search vector index
- Cosmos DB vector search

**Good**
- Metadata filtering
- Chunking strategy
- RAG storage pattern
- Vector index type
- Latency/cost tradeoff

**AI-Infra**
- Embedding pipeline
- Model endpoint + vector DB + retrieval
- Data freshness

**Practical example**
- Build RAG architecture: documents -> chunks -> embeddings -> vector store -> retrieval -> LLM.

**Checkpoint**
- Where do embeddings live, and why is normal SQL search not enough?

---

## Day 56 — Week Review: Data Tier

**Review topics**
- Managed SQL
- Pooling/proxy
- NoSQL
- Global database
- Messaging
- Cache
- Vector DB

**Practice answer**
- “Design the data layer for an AI assistant app using private DB, vector search, queue workers, and global read access.”

---

# Phase 7 — Containers, EKS, AKS, Gateway API, and eBPF

## Day 57 — Container Registry and Image Management

**Must**
- ECR
- ACR
- Image tags
- Immutable tags
- Image scanning
- Pull authentication
- Private registry

**Good**
- Multi-arch images
- Promotion tags
- Vulnerability gates
- Registry replication

**Practical example**
- Build image once, promote from dev to test to prod using immutable digest.

**Checkpoint**
- Why should production deploy by image digest, not mutable tag?

---

## Day 58 — EKS Architecture

**Must**
- EKS control plane
- Managed node group
- Fargate profile
- EKS add-ons
- Cluster endpoint public/private
- IAM Roles for Service Accounts
- EKS Pod Identity
- Cluster Autoscaler
- Karpenter

**Good**
- Private cluster
- Security groups for pods
- Node IAM role vs pod IAM role
- EKS upgrade planning

**Practical example**
- Private EKS cluster with node groups in private subnets and ALB ingress.

**Checkpoint**
- How does a pod access S3 securely from EKS?

---

## Day 59 — AKS Architecture

**Must**
- AKS cluster
- System node pool
- User node pool
- Azure CNI
- Kubenet
- Managed identity
- Azure Workload Identity
- ACR integration
- Cluster Autoscaler

**Good**
- Private AKS cluster
- Node image upgrade
- AKS automatic/standard concepts
- Application routing add-on

**Practical example**
- AKS app accesses Key Vault using workload identity.

**Checkpoint**
- Kubernetes RBAC vs Azure RBAC vs workload identity: how do they differ?

---

## Day 60 — Kubernetes Cloud Load Balancing

**Must**
- Kubernetes Service type LoadBalancer
- Ingress
- AWS Load Balancer Controller
- Azure Application Gateway Ingress Controller
- Internal load balancer
- External load balancer

**Good**
- ALB ingress annotations
- NLB service annotations
- App Gateway backend pool
- Health probe mapping

**Practical example**
- Kubernetes Ingress creates cloud L7 load balancer and routes host/path traffic to services.

**Checkpoint**
- Pod is running, but ingress is not reachable. What do you check?

---

## Day 61 — Kubernetes Gateway API

**Must**
- Gateway API
- GatewayClass
- Gateway
- HTTPRoute
- TCPRoute
- TLSRoute
- GRPCRoute
- Ingress vs Gateway API

**Good**
- Route delegation
- Shared gateway model
- Multi-team routing
- AWS Load Balancer Controller Gateway API
- Azure Application Routing / Application Gateway for Containers

**Advanced**
- Gateway API migration strategy
- Platform-owned Gateway, app-owned routes

**Practical example**
- Platform team owns Gateway; app teams attach HTTPRoutes for their services.

**Checkpoint**
- Why is Gateway API becoming important compared with traditional Ingress?

---

## Day 62 — eBPF, Cilium, and Pod-Level Security

**Must**
- eBPF basics
- Cilium CNI
- Cilium Network Policy
- L3/L4/L7 policy
- Identity-aware policy
- Hubble observability

**Good**
- Service mesh without sidecars concept
- Pod-to-pod visibility
- DNS-aware policy
- HTTP-aware policy
- Cilium on EKS/AKS

**Advanced**
- Cilium Gateway API
- Cilium ClusterMesh
- eBPF-based observability

**Practical example**
- Allow frontend pod to call backend only on `/api`, deny direct DB access from frontend.

**Checkpoint**
- Why can cloud SG/NSG not fully solve Kubernetes pod-to-pod policy?

---

## Day 63 — GPU Nodes in Kubernetes

**Must**
- GPU node pool
- NVIDIA device plugin
- Node labels
- Taints/tolerations
- Resource request: `nvidia.com/gpu`
- GPU scheduling

**Good**
- MIG
- GPU sharing
- Karpenter GPU provisioning
- AKS GPU node pool
- GPU driver management

**AI-Infra / FinOps**
- GPU utilization
- Idle GPU cost
- Inference autoscaling

**Practical example**
- Deploy inference pod requesting one GPU on dedicated GPU node pool.

**Checkpoint**
- Why can a cluster have available CPU but still fail to schedule GPU pods?

---

## Day 64 — Week Review: Kubernetes Cloud Integration

**Review topics**
- EKS
- AKS
- Registry
- Cloud IAM for pods
- Ingress
- Gateway API
- Cilium/eBPF
- GPU node pools

**Practice answer**
- “Design a private EKS/AKS platform exposing apps through Gateway API with secure pod access to cloud services.”

---

# Phase 7.5 — Cloud AI/ML Platform Services and Managed Inference

## Day 65 — AI Platform Decision Model

**Must**
- Managed AI platform vs self-hosted AI platform
- SageMaker vs Azure ML
- Bedrock vs Azure OpenAI
- KServe/Ray/vLLM/Triton on Kubernetes
- Model lifecycle
- Training vs inference
- Batch vs real-time inference

**Good**
- Vendor lock-in
- GPU utilization
- Operational burden
- Compliance requirements

**AI-Infra**
- Platform team responsibilities for AI workloads
- Model serving SLOs
- Cost per inference

**Practical example**
- Decide whether to host a model on SageMaker endpoint, Azure ML endpoint, Bedrock/Azure OpenAI, or vLLM on AKS/EKS.

**Checkpoint**
- When do you use managed AI service vs self-hosted Kubernetes inference stack?

---

## Day 66 — Amazon SageMaker AI

**Must**
- SageMaker Studio
- Training job
- Processing job
- Model Registry
- Model Cards
- Model Monitor
- Real-time endpoint
- Batch Transform
- Asynchronous inference
- Serverless inference
- Multi-model endpoint

**Good**
- SageMaker Pipelines
- Feature Store
- JumpStart
- Private networking
- IAM for model access
- CloudWatch metrics

**Practical example**
- Train model, register version, approve, deploy to real-time endpoint.

**Checkpoint**
- How does SageMaker reduce platform operational burden compared with self-hosted training/inference?

---

## Day 67 — Azure Machine Learning and Azure AI Foundry

**Must**
- Azure ML workspace
- Compute instance
- Compute cluster
- Model registry
- Managed online endpoint
- Batch endpoint
- Environment
- Data asset

**Good**
- Prompt flow basics
- Azure AI Foundry concepts
- Model catalog
- Private endpoint
- Managed identity
- Monitoring endpoints

**Practical example**
- Register a model and deploy it to Azure ML managed online endpoint with private access.

**Checkpoint**
- Azure ML endpoint vs AKS self-hosted model server: what changes operationally?

---

## Day 68 — Foundation Model Services

**Must**
- Amazon Bedrock
- Azure OpenAI Service
- Foundation model access
- Model deployment
- Token limits
- Guardrails
- Content filtering
- Model evaluation

**Good**
- Bedrock Agents
- Bedrock Knowledge Bases
- Azure OpenAI private networking
- Quota management
- Prompt logging controls

**Security / AI-Infra**
- Data privacy
- No direct public exposure
- API gateway in front of model APIs
- Per-team quota and throttling

**Practical example**
- Internal app calls LLM through API Management/API Gateway with auth, rate limit, logging, and quota.

**Checkpoint**
- Bedrock/Azure OpenAI vs hosting open-source LLM on vLLM: what tradeoffs matter?

---

## Day 69 — Managed Inference Endpoints

**Must**
- Real-time inference
- Batch inference
- Async inference
- Serverless inference
- Autoscaling endpoint
- Model versioning
- Canary model rollout
- Blue/green model deployment

**Good**
- Latency SLO
- Cold start
- Payload size
- GPU/CPU endpoint choice
- Model monitoring

**AI-Infra**
- Cost per request
- Token latency
- Endpoint idle cost
- Model rollback

**Practical example**
- Deploy v1 and v2 model endpoints and route 10% traffic to v2.

**Checkpoint**
- How is model deployment different from normal application deployment?

---

## Day 70 — KServe, Ray, vLLM, Triton Comparison

**Must**
- KServe
- Ray Serve
- vLLM
- NVIDIA Triton
- Kubeflow basics
- Model server
- InferenceService
- Autoscaling
- GPU scheduling

**Good**
- Continuous batching
- KV cache
- Tensor parallelism
- Model sharding
- Multi-model serving

**AI-Infra**
- Self-hosted inference on EKS/AKS
- Custom runtime control
- Multi-cloud portability

**Practical example**
- Host open-source LLM on vLLM in AKS/EKS with GPU node pool and expose through internal gateway.

**Checkpoint**
- Why might a platform team choose vLLM on Kubernetes instead of Bedrock/Azure OpenAI?

---

## Day 71 — AI Platform Security and Governance

**Must**
- Model access control
- Data access control
- Prompt/data privacy
- Private endpoints
- VPC/VNet integration
- IAM/managed identity
- Model registry approval
- Audit logs

**Good**
- Model cards
- Responsible AI checks
- Guardrails
- Content filtering
- Prompt injection awareness
- Data exfiltration risk

**Practical example**
- Only approved apps can call production model endpoint; all requests logged with tenant/team metadata.

**Checkpoint**
- What should a platform team control before exposing LLM APIs to application teams?

---

## Day 72 — Week Review: Cloud AI/ML Services

**Review topics**
- SageMaker
- Azure ML
- Bedrock
- Azure OpenAI
- Managed endpoints
- KServe/Ray/vLLM/Triton
- GPU compute
- Model registry
- AI governance

**Practice answer**
- “Design an enterprise AI inference platform across AWS/Azure/Kubernetes with model governance, private access, autoscaling, and cost control.”

---

# Phase 8 — Infrastructure as Code, OpenTofu, Policy Scanning, and Crossplane

## Day 73 — Terraform and OpenTofu Foundations

**Must**
- Terraform
- OpenTofu
- Provider
- Resource
- Data source
- Variable
- Output
- State
- Plan
- Apply
- Destroy

**Good**
- Terraform vs OpenTofu
- Module compatibility
- Provider compatibility
- State compatibility
- License implications

**Practical example**
- Create VPC/VNet using Terraform/OpenTofu.

**Checkpoint**
- Why should OpenTofu be tracked alongside Terraform now?

---

## Day 74 — Terraform/OpenTofu State and Modules

**Must**
- Remote state
- State locking
- State sensitivity
- Module
- Versioning
- Workspace/environment separation

**Good**
- Terragrunt basics
- Module registry
- Import existing resources
- Drift detection

**Security**
- Secrets in state
- State file access control

**Practical example**
- Create reusable network module for AWS and Azure.

**Checkpoint**
- Why is Terraform state a security-sensitive artifact?

---

## Day 75 — AWS IaC Native Tools

**Must**
- CloudFormation
- Stack
- Change set
- Parameters
- Outputs
- StackSets
- CDK basics

**Good**
- CloudFormation drift detection
- Nested stacks
- CDK constructs

**Practical example**
- Compare Terraform module vs CloudFormation stack for VPC creation.

**Checkpoint**
- When would you use CloudFormation/CDK instead of Terraform/OpenTofu?

---

## Day 76 — Azure IaC Native Tools

**Must**
- ARM template
- Bicep
- Parameters
- Variables
- Modules
- Outputs
- Deployment scope

**Good**
- Resource group deployment
- Subscription deployment
- Management group deployment

**Practical example**
- Create resource group, VNet, subnet, and NSG using Bicep.

**Checkpoint**
- Why is Bicep preferred over raw ARM JSON for many Azure-native deployments?

---

## Day 77 — Policy as Code and IaC Scanning

**Must**
- OPA
- Rego basics
- Conftest
- Checkov
- tfsec / Trivy IaC scanning
- Terraform Sentinel
- Terraform plan scanning

**Good**
- Pre-commit hooks
- CI pipeline scanning
- Policy exceptions
- Baseline controls

**Security**
- Deny public S3/Blob
- Deny public DB
- Require encryption
- Require tags
- Require approved regions
- Require IMDSv2

**Practical example**
- CI blocks Terraform plan that creates public storage or disables encryption.

**Checkpoint**
- Why should policy be checked before infrastructure is deployed?

---

## Day 78 — Crossplane and Kubernetes-Native Control Planes

**Must**
- Crossplane
- Kubernetes as cloud control plane
- Managed Resource
- Provider
- Composite Resource
- Composition
- Claim
- Platform API

**Good**
- AWS provider
- Azure provider
- GitOps with Crossplane
- Reconciliation
- Drift correction
- Self-service infra

**Advanced**
- Terraform vs Crossplane
- Platform API design
- Internal developer platform

**Practical example**
- Developer creates `PostgresClaim`; Crossplane provisions managed PostgreSQL in AWS/Azure.

**Checkpoint**
- Why is Crossplane important for platform engineering?

---

## Day 79 — IaC Pipeline Design

**Must**
- Plan stage
- Review stage
- Policy scan
- Security scan
- Approval
- Apply
- Drift check
- Rollback strategy

**Good**
- Environment promotion
- Separate state per environment
- Least privilege deployment role
- OIDC credentials

**Practical example**
- Git PR triggers Terraform plan, Checkov scan, approval, and apply using OIDC role.

**Checkpoint**
- What should happen before infrastructure is changed in production?

---

## Day 80 — Week Review: IaC and Platform Control Plane

**Review topics**
- Terraform
- OpenTofu
- CloudFormation
- Bicep
- Policy as code
- Checkov/tfsec/OPA
- Crossplane
- IaC pipelines

**Practice answer**
- “Design an IaC operating model for a platform team supporting both Terraform/OpenTofu and Crossplane.”

---

# Phase 9 — CI/CD, GitOps, and Supply Chain Security

## Day 81 — CI/CD Fundamentals

**Must**
- Build
- Test
- Scan
- Package
- Artifact
- Deploy
- Rollback
- Promotion
- Environment approval

**Good**
- Blue-green deployment
- Canary deployment
- Rolling deployment
- Feature flags

**Practical example**
- Code commit builds image, runs tests, scans image, pushes artifact, updates deployment manifest.

**Checkpoint**
- CI vs CD vs GitOps: what is the difference?

---

## Day 82 — AWS CI/CD and GitHub Actions to AWS

**Must**
- CodePipeline
- CodeBuild
- CodeDeploy
- ECR deployment
- ECS deployment
- EKS deployment
- GitHub Actions to AWS
- OIDC to AWS

**Good**
- Deployment role
- Least privilege pipeline
- No static AWS keys
- Environment protection

**Practical example**
- GitHub Actions assumes AWS role via OIDC and pushes image to ECR.

**Checkpoint**
- Why should CI/CD not use permanent AWS access keys?

---

## Day 83 — Azure DevOps, GitHub Actions, and Azure Deployments

**Must**
- Azure DevOps
- Azure Pipelines
- Service connection
- GitHub Actions to Azure
- OIDC to Azure
- ACR deployment
- AKS deployment
- App Service deployment

**Good**
- Workload identity federation
- Environment approvals
- Variable groups
- Key Vault integration

**Practical example**
- Azure Pipeline deploys to AKS using federated identity and pulls secrets from Key Vault.

**Checkpoint**
- What is the secure replacement for service principal secrets in pipelines?

---

## Day 84 — GitOps with ArgoCD and Flux

**Must**
- ArgoCD
- FluxCD
- Desired state
- Sync
- Drift detection
- App of Apps
- Helm
- Kustomize
- Environment overlays

**Good**
- Image updater
- Progressive delivery
- GitOps repo structure
- Secret handling

**Practical example**
- CI builds image; GitOps repo updates image tag/digest; ArgoCD syncs cluster.

**Checkpoint**
- Why should CI build artifacts but GitOps deploy desired state?

---

## Day 85 — Supply Chain Security

**Must**
- Secrets scanning
- Dependency scanning
- Container image scanning
- SBOM generation
- SPDX
- CycloneDX
- Cosign
- Sigstore
- Image signing
- Image verification

**Good**
- SLSA basics
- Provenance
- Attestation
- Admission controller for signed images
- Policy exceptions

**Security**
- Block unsigned images
- Block critical CVEs
- Prevent secret leakage

**Practical example**
- Pipeline generates SBOM, signs image with cosign, cluster admits only signed images.

**Checkpoint**
- How do you prove that the image deployed to production is the image built by your pipeline?

---

## Day 86 — Secrets in Pipelines and Runtime

**Must**
- AWS Secrets Manager
- AWS Parameter Store
- Azure Key Vault
- External Secrets Operator
- CSI Secret Store Driver
- OIDC instead of static secrets

**Good**
- Secret rotation
- Secret versioning
- Secret access audit
- Short-lived tokens

**Practical example**
- Kubernetes app reads DB password from Key Vault/Secrets Manager using workload identity.

**Checkpoint**
- Why are Kubernetes Secrets not enough by themselves?

---

## Day 87 — Week Review: CI/CD and Supply Chain

**Review topics**
- CI/CD
- GitOps
- OIDC deployment credentials
- SBOM
- Cosign
- Secrets scanning
- Image scanning
- External secrets

**Practice answer**
- “Design a secure CI/CD and GitOps pipeline for AWS/Azure/Kubernetes with signed images and no static keys.”

---

# Phase 10 — Observability, OpenTelemetry, and Operations

## Day 88 — Observability Foundations

**Must**
- Metrics
- Logs
- Traces
- Events
- Alerts
- Dashboards
- SLI
- SLO
- Error budget
- Runbook
- Postmortem

**Good**
- RED method
- USE method
- Golden signals
- Alert fatigue

**Practical example**
- Define SLO for inference API latency and error rate.

**Checkpoint**
- What is the difference between monitoring and observability?

---

## Day 89 — AWS Observability

**Must**
- CloudWatch Metrics
- CloudWatch Logs
- Logs Insights
- CloudTrail
- X-Ray
- EventBridge
- AWS Config
- VPC Flow Logs
- Container Insights

**Good**
- Metric filters
- Composite alarms
- CloudWatch dashboards
- Centralized logging account

**Practical example**
- Troubleshoot EC2/EKS app using CloudWatch metrics, app logs, ALB target health, and CloudTrail.

**Checkpoint**
- CloudWatch vs CloudTrail: what is each used for?

---

## Day 90 — Azure Observability

**Must**
- Azure Monitor
- Log Analytics Workspace
- KQL
- Application Insights
- Activity Log
- Resource Graph
- Network Watcher
- NSG Flow Logs
- Container Insights

**Good**
- Diagnostic settings
- Action groups
- Workbooks
- Central Log Analytics workspace

**Practical example**
- Troubleshoot AKS app using Container Insights, App Insights, Activity Log, and NSG flow logs.

**Checkpoint**
- Azure Monitor vs Log Analytics vs Application Insights.

---

## Day 91 — OpenTelemetry

**Must**
- OpenTelemetry
- OTel SDK
- OTel Collector
- Traces
- Metrics
- Logs
- Context propagation
- Trace ID
- Span

**Good**
- Export to CloudWatch
- Export to Azure Monitor
- Export to Prometheus/Grafana
- Export to Jaeger/Tempo
- Sampling

**AI-Infra**
- Inference latency traces
- Token latency
- Queue latency
- GPU utilization correlation

**Practical example**
- Instrument API -> queue -> worker -> DB with distributed tracing.

**Checkpoint**
- Why does OpenTelemetry prevent vendor lock-in in observability?

---

## Day 92 — Incident Troubleshooting Flow

**Must**
- DNS check
- CDN/WAF check
- Load balancer health
- Security group/NSG
- Route table
- NAT
- Pod/VM health
- App logs
- DB connectivity
- Recent deployment
- Audit logs

**Good**
- Blast radius
- Mitigation
- Rollback
- Communication
- Postmortem

**Practical example**
- App down after deployment: trace through DNS, LB, backend, logs, DB, and CloudTrail/Activity Log.

**Checkpoint**
- What is your first 10-minute incident checklist?

---

## Day 93 — AI Workload Observability

**Must**
- Model endpoint latency
- Error rate
- Token latency
- Queue depth
- GPU utilization
- GPU memory
- Cold start
- Model version
- Request volume

**Good**
- Prompt logging controls
- PII-safe logging
- Cost per request
- Hallucination/eval signals
- Model drift basics

**Practical example**
- Monitor vLLM endpoint on EKS/AKS and compare with SageMaker/Azure ML endpoint metrics.

**Checkpoint**
- What extra metrics do AI inference workloads need beyond normal web services?

---

## Day 94 — Week Review: Observability

**Review topics**
- CloudWatch
- Azure Monitor
- OpenTelemetry
- Logs/metrics/traces
- Incident flow
- AI inference observability

**Practice answer**
- “Troubleshoot high latency in an AI inference API running on AKS with Azure OpenTelemetry export and GPU metrics.”

---

# Phase 11 — Security, Secrets, Threat Detection, and Compliance Controls

## Day 95 — Cloud Security Foundations

**Must**
- Shared responsibility
- Identity security
- Network security
- Data security
- Runtime security
- Audit logging
- Threat detection
- Vulnerability management

**Good**
- Defense in depth
- Zero trust
- Security baseline
- CIS benchmark

**Practical example**
- Build a minimum security baseline for dev/test/prod cloud accounts.

**Checkpoint**
- What remains your responsibility when using managed cloud services?

---

## Day 96 — Encryption and Key Management

**Must**
- AWS KMS
- Azure Key Vault
- Encryption at rest
- Encryption in transit
- Customer-managed key
- Key rotation
- Certificate management

**Good**
- HSM
- Key access audit
- Envelope encryption
- mTLS basics

**Practical example**
- Encrypt storage, database, and secrets with customer-managed keys.

**Checkpoint**
- When do you need customer-managed keys instead of provider-managed keys?

---

## Day 97 — Cloud Security Services

**Must**
- AWS GuardDuty
- AWS Security Hub
- AWS Inspector
- AWS Config
- AWS WAF
- AWS Shield
- Azure Defender for Cloud
- Azure Policy
- Azure Firewall
- Azure WAF
- Microsoft Sentinel basics

**Good**
- Macie basics
- Vulnerability findings
- Security posture management
- Central security dashboard

**Practical example**
- Central security team monitors findings across all AWS accounts and Azure subscriptions.

**Checkpoint**
- Security Hub/Defender for Cloud vs SIEM: what is the difference?

---

## Day 98 — Secrets and Runtime Security

**Must**
- Secrets Manager
- Parameter Store
- Key Vault
- External Secrets Operator
- CSI Secret Store Driver
- Secret rotation
- Runtime identity

**Good**
- No secrets in images
- No secrets in Git
- Secret access audit
- Secret versioning

**Practical example**
- Pod uses workload identity to fetch DB credentials at runtime.

**Checkpoint**
- How do you rotate secrets without redeploying every application manually?

---

## Day 99 — Network and Data Protection

**Must**
- No public DB
- No public storage
- Private endpoint
- VPC endpoint
- WAF
- Firewall
- Egress control
- Flow logs

**Good**
- DLP basics
- DNS firewall
- Data classification
- Tokenization basics

**Practical example**
- Block public access to storage and allow access only from private workloads.

**Checkpoint**
- How do you prove a database is not reachable from the public internet?

---

## Day 100 — Compliance Mapping

**Must / Good depending on role**
- SOC 2
- ISO 27001
- PCI DSS
- HIPAA basics
- CIS benchmark
- Cloud control mapping
- Evidence collection

**Evidence topics**
- Audit logs
- Access reviews
- Encryption evidence
- Backup evidence
- Change management evidence
- Vulnerability evidence
- Policy compliance evidence

**Practical example**
- Map “encryption at rest required” to KMS/Key Vault settings, policy checks, and audit evidence.

**Checkpoint**
- Why do platform teams need compliance evidence even if they are not auditors?

---

## Day 101 — Week Review: Security

**Review topics**
- KMS/Key Vault
- GuardDuty/Defender
- Secrets
- Private access
- Compliance evidence
- Runtime security

**Practice answer**
- “Design security guardrails for a multi-account AWS and multi-subscription Azure platform.”

---

# Phase 12 — Reliability, Backup, DR, and Chaos Engineering

## Day 102 — Reliability Foundations

**Must**
- High availability
- Fault tolerance
- Resilience
- Multi-AZ
- Health checks
- Retry
- Timeout
- Circuit breaker
- Backpressure

**Good**
- Bulkhead pattern
- Graceful degradation
- Dependency failure design

**Practical example**
- App continues operating when one app node or pod fails.

**Checkpoint**
- HA vs DR: what is the difference?

---

## Day 103 — Backup and Restore

**Must**
- Backup policy
- Snapshot
- Retention
- Point-in-time restore
- Cross-region backup
- Immutable backup
- Restore test

**Good**
- AWS Backup
- Azure Backup
- Backup vault
- Ransomware-resistant backup

**Practical example**
- Restore database from point-in-time backup into isolated environment.

**Checkpoint**
- What is the difference between backup existing and backup working?

---

## Day 104 — Disaster Recovery Patterns

**Must**
- RPO
- RTO
- Backup and restore
- Pilot light
- Warm standby
- Active-passive
- Active-active
- DNS failover

**Good**
- DR drill
- Runbook
- Cross-region replication
- Application dependency mapping

**Practical example**
- Design DR for web app, database, object storage, queue, and DNS.

**Checkpoint**
- How do RPO/RTO drive architecture cost?

---

## Day 105 — AWS and Azure DR Services

**Must**
- AWS Backup
- AWS Elastic Disaster Recovery basics
- Route 53 failover
- RDS/Aurora failover
- Azure Backup
- Azure Site Recovery
- Azure Front Door failover
- Azure SQL HA

**Good**
- Region pair concept
- Cross-region storage replication
- Failback planning

**Practical example**
- Primary region fails; DNS routes users to standby region.

**Checkpoint**
- Why should DR be tested before real disaster?

---

## Day 106 — Chaos Engineering

**Must**
- AWS Fault Injection Service
- Azure Chaos Studio
- Game days
- Failure injection
- Instance termination
- Network latency
- Pod kill
- AZ failure simulation
- DB failover test

**Good**
- Blast radius control
- Stop conditions
- SLO impact measurement
- Experiment hypothesis

**Practical example**
- Terminate one node/pod/instance and confirm autoscaling, health checks, and alerts work.

**Checkpoint**
- What is the difference between chaos engineering and randomly breaking production?

---

## Day 107 — Week Review: Reliability and DR

**Review topics**
- HA
- Backup
- RPO/RTO
- DR patterns
- Chaos engineering
- Failover testing

**Practice answer**
- “Design and test resilience for a multi-AZ app with cross-region DR and chaos experiments.”

---

# Phase 13 — Cost Optimization, Kubernetes FinOps, and AI Cost

## Day 108 — Cloud FinOps Basics

**Must**
- Cost allocation
- Tags
- Budgets
- Forecast
- Rightsizing
- Idle resource cleanup
- Reserved capacity
- Savings Plans
- Spot
- Data transfer cost

**Good**
- Chargeback/showback
- Unit economics
- Cost anomaly detection

**Practical example**
- Build weekly cost review by team/environment/application.

**Checkpoint**
- Why is tagging the foundation of FinOps?

---

## Day 109 — AWS and Azure Cost Tools

**Must**
- AWS Cost Explorer
- AWS Budgets
- Cost and Usage Report
- Compute Optimizer
- Azure Cost Management
- Azure Budgets
- Azure Advisor
- Reservations
- Savings Plans

**Good**
- Azure Hybrid Benefit
- Reserved Instances
- Spot interruption planning

**Practical example**
- Identify idle disks, oversized VMs, and expensive NAT/data transfer.

**Checkpoint**
- Which costs are predictable, and which costs grow with traffic?

---

## Day 110 — Kubernetes Cost Management

**Must**
- Kubecost
- OpenCost
- Namespace cost allocation
- Pod cost allocation
- Node cost allocation
- Idle resource detection
- Over-requested CPU/memory

**Good**
- Karpenter consolidation
- Cluster autoscaler cost
- AKS node pool cost
- EKS managed node group cost
- Spot nodes

**Practical example**
- Show cost per namespace/team and reduce over-requested resources.

**Checkpoint**
- Why can Kubernetes make cloud cost harder to understand?

---

## Day 111 — AI and GPU FinOps

**Must**
- GPU hourly cost
- Idle GPU detection
- GPU utilization
- Cost per token
- Cost per inference
- Endpoint idle cost
- Batch vs real-time cost
- Scale-to-zero

**Good**
- Reserved GPU capacity
- Spot GPU
- Quantization cost impact
- Model size vs infra cost
- Caching AI responses

**Practical example**
- Compare cost of Bedrock/Azure OpenAI API vs self-hosted vLLM on GPU nodes.

**Checkpoint**
- Why can low GPU utilization destroy AI platform economics?

---

## Day 112 — Network and Data Transfer Cost

**Must**
- NAT Gateway cost
- Load balancer cost
- Public IPv4 cost
- Cross-AZ data transfer
- Cross-region data transfer
- Egress cost
- PrivateLink/Private Endpoint cost

**Good**
- CDN savings
- VPC endpoint savings
- Regional placement
- Data locality

**Practical example**
- Reduce NAT cost by using VPC endpoints/private endpoints for cloud service access.

**Checkpoint**
- Why can NAT gateway become a hidden cost problem?

---

## Day 113 — Week Review: FinOps

**Review topics**
- Tags
- Budgets
- Rightsizing
- Kubernetes cost
- GPU cost
- Data transfer
- NAT cost

**Practice answer**
- “Create a cost optimization plan for an EKS/AKS AI inference platform with GPU nodes and private endpoints.”

---

# Phase 14 — Landing Zone, Enterprise Platform, and Multi-Cloud Governance

## Day 114 — Landing Zone Fundamentals

**Must**
- Landing zone
- Account/subscription structure
- Central identity
- Central networking
- Central logging
- Central security
- Policy enforcement
- Tagging enforcement
- Budget enforcement

**Good**
- Shared services
- Platform team operating model
- Application team onboarding
- Guardrails vs gates

**Practical example**
- New application team gets dev/test/prod accounts/subscriptions with standard network, logging, IAM, and budgets.

**Checkpoint**
- Why is a landing zone more than just a VPC/VNet?

---

## Day 115 — AWS Landing Zone

**Must**
- AWS Organizations
- Control Tower
- Account Factory
- SCP
- IAM Identity Center
- Centralized CloudTrail
- Centralized Config
- Shared/network account
- Transit Gateway

**Good**
- Log archive account
- Audit account
- Security tooling account
- Account vending automation

**Practical example**
- Create secure baseline for new AWS account with logging, guardrails, and network connection.

**Checkpoint**
- What should every new AWS account have before app teams use it?

---

## Day 116 — Azure Landing Zone

**Must**
- Management Groups
- Subscriptions
- Resource Groups
- Azure Policy
- Hub-spoke network
- Shared services subscription
- Log Analytics workspace
- Defender for Cloud
- Azure Firewall
- Private DNS

**Good**
- Subscription vending
- Policy initiative
- Management group hierarchy
- Platform landing zone vs application landing zone

**Practical example**
- Onboard a new application subscription with private networking, policies, logging, and budget.

**Checkpoint**
- What should every new Azure subscription inherit from management groups?

---

## Day 117 — Multi-Cloud Governance

**Must**
- AWS + Azure operating model
- Central identity
- Cross-cloud federation
- Multi-cloud network
- Multi-cloud DNS
- Central observability
- Central cost reporting
- Policy consistency

**Good**
- Multi-cloud service catalog
- Common tagging standard
- Common CI/CD standards
- Common security baseline

**Advanced**
- Multi-cloud platform APIs
- Crossplane multi-cloud claims
- Unified developer portal

**Practical example**
- Developer requests “PostgreSQL database” from platform portal and gets AWS RDS or Azure PostgreSQL depending on environment.

**Checkpoint**
- How do you avoid two completely different platform experiences across AWS and Azure?

---

## Day 118 — Self-Service Platform Engineering

**Must**
- Internal developer platform
- Golden paths
- Service catalog
- Templates
- Guardrails
- Self-service infrastructure
- GitOps onboarding
- Platform API

**Good**
- Backstage basics
- Crossplane claims
- Terraform modules as products
- Developer experience metrics

**Practical example**
- Developer creates new service using template: repo, CI/CD, observability, cloud resources, Kubernetes namespace.

**Checkpoint**
- What should platform teams automate so app teams do not raise tickets for everything?

---

## Day 119 — Week Review: Landing Zone and Governance

**Review topics**
- AWS landing zone
- Azure landing zone
- Central identity
- Central network
- Central logging
- Multi-cloud governance
- Self-service platform

**Practice answer**
- “Design an enterprise multi-cloud landing zone for AWS and Azure with Kubernetes and AI workload support.”

---

# Phase 15 — Capstone Architecture and Interview Design Practice

## Day 120 — Capstone 1: Secure 3-Tier Web App

**Design topics**
- DNS
- CDN/WAF
- Public load balancer
- Private app subnet
- Private database subnet
- NAT outbound
- Bastion/SSM access
- Storage
- Monitoring
- Backup
- IaC

**Practice answer**
- Design this in AWS and Azure.

---

## Day 121 — Capstone 2: Private Kubernetes Platform

**Design topics**
- Private EKS/AKS
- Private node pools
- Ingress/Gateway API
- Internal/external load balancers
- Pod identity
- Private registry
- External secrets
- Observability
- Network policies
- Cilium/eBPF

**Practice answer**
- Design a secure multi-team Kubernetes platform on AWS/Azure.

---

## Day 122 — Capstone 3: AI Inference Platform

**Design topics**
- Bedrock/Azure OpenAI
- SageMaker/Azure ML
- vLLM/KServe/Ray on EKS/AKS
- GPU nodes
- API Gateway/APIM
- Auth/quota/throttling
- Vector database
- Observability
- Model registry
- Cost controls

**Practice answer**
- Design a multi-cloud AI inference platform and explain managed vs self-hosted choice.

---

## Day 123 — Capstone 4: Multi-Cloud Hybrid Architecture

**Design topics**
- AWS to Azure connectivity
- On-prem connectivity
- Direct Connect
- ExpressRoute
- VPN
- BGP
- Private DNS resolver
- Private endpoints
- Non-overlapping CIDR
- Central firewall
- Cross-cloud identity

**Practice answer**
- Design private connectivity and DNS across on-prem, AWS, and Azure.

---

## Day 124 — Capstone 5: Secure CI/CD and Supply Chain

**Design topics**
- GitHub/Azure DevOps
- OIDC to AWS/Azure
- Build/test/scan
- SBOM
- Cosign signing
- Image verification
- GitOps deployment
- IaC scanning
- Policy as code
- Secrets management

**Practice answer**
- Design secure deployment pipeline with no static cloud keys.

---

## Day 125 — Capstone 6: Resilience and DR

**Design topics**
- Multi-AZ
- Multi-region
- RPO/RTO
- Backup
- Restore testing
- DNS failover
- Chaos testing
- Game day
- DB failover
- Queue durability

**Practice answer**
- Design and test DR for a production platform.

---

## Day 126 — Final Review Day

**Final checklist**
- Identity/governance
- JIT/PIM/federation
- Networking/private access/DNS/IPv6
- AWS↔Azure interconnect
- Compute/serverless/API gateway
- IMDSv2/metadata security
- ARM/confidential/GPU compute
- Storage/database/vector DB
- Messaging/cache
- EKS/AKS/Gateway API/Cilium
- AI/ML platform services
- Managed vs self-hosted inference
- Terraform/OpenTofu/Crossplane
- CI/CD/GitOps/supply chain
- Observability/OpenTelemetry
- Security/compliance
- Backup/DR/chaos
- FinOps/Kubernetes cost/AI cost
- Landing zone/self-service platform

**Final interview prompt**
- “Explain how you would design a secure, cost-aware, multi-cloud AWS/Azure platform that runs Kubernetes and AI workloads, supports self-service developer onboarding, uses no static credentials, exposes APIs safely, supports private networking, and has observability, DR, and governance built in.”

---

# Final Priority Summary

## Must Master First

1. Identity, RBAC, federation, JIT/PIM
2. VPC/VNet, private subnet, NAT, DNS, private endpoints
3. EC2/VM, Lambda/Functions, API Gateway/APIM decision-making
4. EKS/AKS, pod identity, ingress/Gateway API
5. Terraform/OpenTofu, state, modules, policy scanning
6. Observability with CloudWatch/Azure Monitor/OpenTelemetry
7. Security: KMS/Key Vault, secrets, IMDSv2, no public DB/storage
8. Database: managed SQL, pooling, global database, vector store
9. AI platform: SageMaker, Azure ML, Bedrock, Azure OpenAI, vLLM/KServe
10. Cost: cloud, Kubernetes, GPU, NAT, data transfer

## Advanced but Important

1. Multi-cloud interconnect
2. Hybrid DNS resolvers
3. IPv6 and dual-stack
4. ARM64 migration
5. Confidential computing
6. eBPF/Cilium
7. Crossplane platform APIs
8. Supply chain security
9. Chaos engineering
10. Compliance evidence mapping

---

# Appendix — Decision Questions to Practice

## Compute
- If Lambda exists, why do we need EC2?
- When do we choose Azure Functions vs AKS?
- When does ARM64 make sense?
- When do we need confidential computing?
- When do we need GPU instead of CPU?

## Networking
- How does a private server access internet without public IP?
- Why is VPC/VNet peering not automatically transitive?
- How do Route 53 Resolver and Azure DNS Private Resolver help hybrid DNS?
- How do you connect AWS and Azure privately?
- Why does Private Endpoint need DNS planning?

## Kubernetes
- Why Gateway API over Ingress?
- Why Cilium/eBPF over basic CNI/network policy?
- How do pods access cloud services securely?
- How do you run GPU workloads in EKS/AKS?

## AI Platform
- SageMaker/Azure ML vs KServe/vLLM on Kubernetes?
- Bedrock/Azure OpenAI vs self-hosted open-source LLM?
- Where do embeddings live?
- How do you expose AI model APIs safely?
- How do you monitor cost per inference?

## Security
- How does OIDC remove static cloud credentials?
- What is IMDSv2 protecting against?
- How do you prove no public storage/database exists?
- How do you enforce signed images only?
- How do you collect compliance evidence?

## Reliability and Cost
- What is RPO/RTO?
- How do you test DR?
- Why run chaos experiments?
- Why is NAT Gateway expensive?
- Why can GPU platforms become extremely expensive?
