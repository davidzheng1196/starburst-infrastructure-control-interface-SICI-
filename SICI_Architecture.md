# Starburst Infrastructure & Control Interface (SICI)

## Federated Query Control Plane
Enterprise-Managed Starburst Platform

---

# 1. Overview

Starburst Infrastructure & Control Interface (SICI) transforms decentralized Starburst deployments into a centrally governed, enterprise-managed federated query platform.

Instead of application teams deploying and managing Starburst clusters within their own AWS accounts, SICI provides:

- Dedicated cluster per team
- Centralized AWS account ownership
- Strict configuration standardization
- Centralized governance enforcement
- Automated provisioning via REST API

SICI acts as the **control plane** for all Starburst deployments.
By centralizing deployment, configuration, and lifecycle management, SICI transforms Starburst from an IaaS offering into a SaaS-like, fully managed service—delivering standardized, self-service query capabilities for teams.

---

# 2. Problem Statement

## Current State

Application teams today:

- Deploy Starburst in their own AWS accounts
- Manage their own EKS clusters
- Handle:
    - EKS upgrades
    - IAM policies
    - Networking
    - Starburst image updates
    - Monitoring
    - Usage reporting

## Issues

### Operational Burden
- Duplicate infrastructure across teams
- Inconsistent upgrade cadence
- High onboarding friction

### Governance Gaps
- No centralized RBAC enforcement
- Inconsistent IAM configurations
- No standardized connector approval process

### Compliance & Risk Exposure
- Limited visibility into cluster usage
- Manual billing reporting
- Potential data access misconfiguration
- No authoritative fleet control

---

# 3. Target State

Starburst becomes a centrally managed SaaS-style service.

Application teams:

- Submit cluster request via REST API
- Request datasource access
- Consume query endpoint
- View usage metrics

Application teams do NOT:

- Manage infrastructure
- Modify Starburst config
- Adjust IAM policies
- Install plugins
- Manage reverse proxy and http request body modification

The Data Platform Team:

- Owns infrastructure(clusters and related resources)
- Owns IAM templates
- Owns configuration
- Owns upgrades
- Owns compliance posture
- Owns role based access control
- Owns usage telemetry for billing purposes

---

# 4. Architecture Overview

## Control Plane

Runs in central AWS account.

Responsibilities:

- Cluster lifecycle management
- Configuration generation
- IAM template generation
- VPC provisioning
- EKS deployment
- Starburst deployment
- Fleet upgrade orchestration
- Usage aggregation
- Audit logging
- Policy enforcement

Primary interface: REST API

---

## Data Plane (Per Team)

For each team:

- Dedicated VPC
- Dedicated subnets
- Dedicated EKS cluster
- Dedicated Starburst deployment
- Dedicated IAM role
- Central logging export
- No manual access

Strict isolation between clusters.

No cross-tenant connectivity.

---

## Control Plane / Data Plane Model

Application Teams

↓

SICI REST API (Control Plane)

↓

Central AWS Account

↓

Per-Team VPC

↓

Per-Team EKS

↓

Starburst Cluster


---

# 5. Governance Philosophy

SICI follows a **Strict Standardization Model**.

Zero configuration override by application teams.

## Not Allowed

- Custom Starburst configuration
- Custom plugin installation
- Arbitrary connector parameter override
- Direct IAM modification
- Direct Kubernetes access

## Allowed

- Select compute tier (S/M/L)
- Submit datasource request
- Create/delete cluster
- View metrics
- Query endpoint

All configuration is generated and enforced by the control plane.

---

# 6. REST API (V1 Surface)

## Clusters

POST /clusters


GET /clusters/{id}

GET /clusters

DELETE /clusters/{id}

GET /clusters/{id}/events


Cluster fields:

- clusterId
- teamId
- environment
- tier
- status (PROVISIONING | ACTIVE | FAILED | DELETING)
- endpoint
- createdAt

Cluster creation is asynchronous.

---

## Datasources
POST /clusters/{id}/datasources

GET /clusters/{id}/datasources

DELETE /clusters/{id}/datasources/{dsId}


Supported in V1:

- Approved S3 connectors (via cross-account assume role)
- Approved internal sources only

Control plane generates:

- IAM policies
- Connector configuration
- Access validation rules

---

## Usage
GET /clusters/{id}/usage

GET /clusters/{id}/queries


Metrics include:

- Query count
- CPU usage
- Memory usage
- Data scanned
- Billing estimate

---

# 7. Provisioning Workflow

Cluster creation executes via orchestrated workflow:

1. Validate team entitlement
2. Allocate cluster metadata and tags
3. Create per-team VPC
4. Create subnets and security groups
5. Generate IAM roles and policies
6. Create EKS cluster
7. Install baseline addons
8. Deploy Starburst (immutable image)
9. Register endpoint
10. Mark cluster ACTIVE

All steps must be idempotent.

Provisioning is asynchronous (job-based model).

---

# 8. AWS Service Architecture (Direct AWS Management)

## Core AWS Services

- API Gateway / Internal Load Balancer
- ECS or EKS (Control Plane Service)
- AWS Step Functions (Workflow Orchestration)
- Lambda (Provisioning Steps)
- DynamoDB or RDS (State Store)
- EKS (Per-Team Clusters)
- IAM (Scoped Roles)
- VPC/Subnets/Security Groups
- CloudWatch (Logging & Metrics)
- ALB/NLB (Cluster Endpoint Exposure)

---

## Orchestration Model

REST API:

- Writes provisioning job to state store
- Starts Step Function execution
- Returns 202 Accepted + jobId

Step Function:

- Executes AWS resource creation steps
- Handles retries
- Maintains state transitions
- Ensures idempotency

---

# 9. Internal Domain Model

## Core Entities

### Cluster

- clusterId
- teamId
- environment
- tier
- status
- vpcId
- eksClusterArn
- iamRoleArn
- endpoint
- createdAt
- updatedAt

---

### ProvisioningJob

- jobId
- clusterId
- operationType (CREATE | DELETE | UPDATE | ADD_DATASOURCE)
- status
- startedAt
- completedAt
- errorDetails

---

### Datasource

- datasourceId
- clusterId
- type
- externalAccountId
- bucketArn
- region
- iamPolicyReference

---

### UsageRecord

- clusterId
- timeWindow
- cpuUsage
- memoryUsage
- queryCount
- dataScanned
- estimatedCost

---

# 10. Security & Compliance Model

## Network Isolation

- Dedicated VPC per team
- No VPC peering between tenants
- Private subnets for worker nodes
- Controlled ingress only
- Restricted egress where possible

## IAM Isolation

- Dedicated IRSA role per cluster
- Scoped S3 bucket access only
- Cross-account STS assume role
- Least privilege enforced via template generator

## Operational Lockdown

- No SSH access
- No kubectl access for app teams
- Immutable Starburst image
- Centralized logging required
- Mandatory resource tagging

---

# 11. Security Review Readiness

## Questions Security Will Ask

1. How do you prevent cross-tenant data access?
2. Who controls connector configuration?
3. How are IAM roles scoped?
4. Can application teams modify infrastructure?
5. Is all access auditable?
6. How do you perform emergency shutdown?

## Prepared Answers

- Dedicated VPC per tenant
- Zero configuration override
- IAM policies generated by control plane
- No direct infra access
- Full audit logging of all provisioning and query activity
- Fleet-level shutdown capability via control plane

---

# 12. Non-Goals (V1)

- Custom plugin installation
- Arbitrary Starburst tuning
- Cross-team shared clusters
- Manual IAM configuration
- Multi-tenant pooled compute
- UI portal

---

# 13. Implementation Stack (Suggested)

## Control Plane

- Language: Java (Spring Boot) or Kotlin
- REST: Spring Web
- Persistence: DynamoDB or Postgres
- Orchestration: AWS Step Functions
- Provisioning: AWS SDK v2
- Deployment: ECS or EKS

## Data Plane

- EKS per team
- Helm-based Starburst deployment
- Immutable container image
- Centralized logging sidecar

---

# 14. Success Criteria

## Governance

- 100% centralized provisioning
- Zero unmanaged Starburst deployments
- Standardized IAM policies
- Full audit traceability

## Operational

- Cluster creation under 20 minutes
- Fleet-wide upgrade capability
- No configuration drift

## Financial

- Accurate per-cluster usage reporting
- Automated chargeback capability
- Reduced infra duplication

---

# 15. Roadmap

## Phase 1 – Governance Foundation
- Cluster provisioning
- Datasource onboarding
- Usage telemetry
- Strict enforcement model

## Phase 2 – Operational Maturity
- Fleet upgrade orchestration
- Enhanced monitoring dashboards
- Policy abstraction layer

## Phase 3 – Platform Expansion
- UI portal
- Dev/Test isolation models
- Connector expansion process
- Automated compliance scanning

---

# 16. Strategic Positioning

SICI establishes:

- Centralized governance boundary
- Enterprise-grade federated data control
- Standardized Starburst deployment model
- Auditable and compliant query platform

It transforms Starburst from:

Infrastructure product  
into  
Enterprise-managed data service