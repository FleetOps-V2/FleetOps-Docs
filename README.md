# FleetOps V2

**FleetOps V2** is a production-grade fleet maintenance and vehicle tracking platform built on AWS. It provides fleet managers with real-time vehicle tracking, maintenance scheduling, AI-powered fleet health analysis, and an automated service request workflow — all served from a secured Kubernetes cluster on Amazon EKS.

| | |
|---|---|
| **Live URL** | https://fleetops.website |
| **ArgoCD Dashboard** | https://argocd.fleetops.website |
| **AWS Region** | us-east-1 |

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Repository Structure](#2-repository-structure)
3. [Architecture](#3-architecture)
   - [Request Flow](#request-flow)
   - [Networking Design](#networking-design)
4. [Services](#4-services)
   - [Auth Service](#auth-service)
   - [Vehicle Service](#vehicle-service)
   - [Maintenance Service](#maintenance-service)
   - [Request Service](#request-service)
   - [Service-to-Service Communication](#service-to-service-communication)
   - [Frontend](#frontend)
5. [Security and Authentication](#5-security-and-authentication)
   - [JWT Authentication](#jwt-authentication)
   - [IRSA — Pod-Level AWS Access](#irsa--pod-level-aws-access)
   - [GitHub Actions OIDC](#github-actions-oidc)
6. [Secret Management](#6-secret-management)
7. [AI Fleet Advisor](#7-ai-fleet-advisor)
8. [Database](#8-database)
9. [Infrastructure](#9-infrastructure)
10. [GitOps with ArgoCD](#10-gitops-with-argocd)
11. [CI/CD Pipeline](#11-cicd-pipeline)
12. [Daily Alert Automation](#12-daily-alert-automation)
13. [Observability](#13-observability)
14. [Technology Stack](#14-technology-stack)
15. [Local Development](#15-local-development)
16. [Deployment Guide](#16-deployment-guide)
17. [Operations Reference](#17-operations-reference)
18. [Troubleshooting](#18-troubleshooting)

---

## 1. System Overview

FleetOps V2 is composed of four independent Spring Boot microservices, a React frontend, and a full AWS infrastructure managed entirely by Terraform. Every component — from secret rotation to rolling deployments — is automated.

**Key capabilities:**

| Capability | Implementation |
|---|---|
| Fleet management & GPS tracking | `fleetops-vehicle-service` |
| AI fleet health analysis | Amazon Bedrock (Amazon Nova Lite) |
| Maintenance task queues | `fleetops-maintenance-service` |
| Service request orchestration | `fleetops-request-service` + AWS Step Functions |
| Secure file storage (photos, PDFs, videos) | Amazon EFS (shared across pods) |
| Daily maintenance & insurance alerts | AWS Lambda + EventBridge + SNS |
| Zero-downtime deployments | ArgoCD (GitOps) + EKS rolling updates |
| No static AWS credentials | IRSA + GitHub Actions OIDC |
| No secrets in Git | External Secrets Operator + AWS Secrets Manager |

---

## 2. Repository Structure

The platform spans nine repositories under the `FleetOps-V2` GitHub organization:

```
FleetOps-V2/
├── fleetops-auth-service/          Spring Boot — authentication, JWT issuance, user management
├── fleetops-vehicle-service/       Spring Boot — fleet catalog, tracking, AI health advisor
├── fleetops-maintenance-service/   Spring Boot — task queues, media uploads, SNS alerts
├── fleetops-request-service/       Spring Boot — service request lifecycle, Step Functions
├── fleetops-frontend/              React SPA — fleet dashboard, role-based UI
├── fleetops-infra/                 Local development — Docker Compose stack + Nginx gateway
├── fleetops-terraform/             Terraform — all AWS infrastructure as modules
├── fleetops-deployments/           Kubernetes manifests, Helm charts, ArgoCD apps
└── fleetops-github-workflows/      Shared reusable GitHub Actions workflow templates
```

Each repository has an independent CI/CD pipeline. Shared workflow logic (Docker build, ECR push, ArgoCD sync) is defined once in `fleetops-github-workflows/` and called from each service repo using workflow_call.

---

## 3. Architecture

### Architecture Diagrams

| File | Description |
|---|---|
| [FleetOps-Architecture.drawio](FleetOps-Architecture.drawio) | Full AWS architecture (AWS icon set, 2026) |
| [FleetOps-Architecture.png](FleetOps-Architecture.png) | Static PNG export |
| [FleetOps-AWS-Architecture.drawio](FleetOps-AWS-Architecture.drawio) | Infrastructure-focused view |
| [FleetOps-Application-Architecture.drawio](FleetOps-Application-Architecture.drawio) | Application and service interaction view |
| [ARCHITECTURE.mmd](ARCHITECTURE.mmd) | Mermaid source (text-based diagram) |

### Request Flow

A user request travels through five distinct layers before reaching a backend pod:

```
Browser
  │
  │  HTTPS (port 443)
  ▼
Amazon CloudFront
  ├── WAF v2 — inspects for SQLi, XSS, known bad inputs, rate limits
  ├── TLS terminated here using ACM wildcard cert (*.fleetops.website)
  └── Forwards to → origin.fleetops.website
  │
  ▼
AWS Application Load Balancer   (internet-facing, public subnets)
  ├── Terminates TLS (second time — CloudFront→ALB is also HTTPS)
  ├── Routes by path using rules from the Kubernetes Ingress resource:
  │     /api/auth      →  fleetops-auth-service:8080
  │     /api/vehicles  →  fleetops-vehicle-service:8080
  │     /api/tasks     →  fleetops-maintenance-service:8080
  │     /api/media     →  fleetops-maintenance-service:8080
  │     /api/requests  →  fleetops-request-service:8080
  │     /api/tracking  →  fleetops-vehicle-service:8080
  │     /              →  fleetops-frontend:80
  └── Connects directly to pod IP (target-type: ip, bypasses kube-proxy)
  │
  ▼
EKS Pod (fleetops-prod namespace)
  ├── Spring Security reads JWT from HttpOnly cookie
  ├── Validates JWT locally (shared JWT_SECRET from Secrets Manager)
  └── @PreAuthorize enforces role-based access per endpoint
```

> **Note on the ALB and Ingress:** The AWS Load Balancer Controller (running as a pod in the cluster) watches for Kubernetes Ingress resources and translates them into real ALB listener rules and target groups via AWS APIs. The ALB itself is what routes traffic — the Ingress YAML is just configuration. There is no Nginx or proxy pod in the traffic path.

### Networking Design

```
VPC: 10.2.0.0/16
│
├── Public Subnets           (ALB, NAT Gateway — internet-reachable)
│   ├── 10.2.1.0/24          us-east-1a
│   └── 10.2.2.0/24          us-east-1b
│
├── Private Subnets          (EKS worker nodes — no inbound from internet)
│   ├── 10.2.10.0/24         us-east-1a
│   └── 10.2.11.0/24         us-east-1b
│
└── Database Subnets         (RDS, Redis — isolated, no route to NAT)
    ├── 10.2.20.0/24         us-east-1a
    └── 10.2.21.0/24         us-east-1b
```

EKS nodes sit in private subnets. All outbound internet traffic (e.g., Docker Hub pulls during rare bootstrap) routes through a single NAT Gateway in us-east-1a.

**VPC Endpoints** keep all AWS API traffic on the AWS private backbone, never leaving the VPC:

| Type | Endpoints |
|---|---|
| Interface (private link) | `ecr-api`, `ecr-dkr`, `sts`, `logs`, `secretsmanager`, `kms`, `sns`, `bedrock-runtime`, `eks`, `ec2`, `ssm`, `step-functions` |
| Gateway (free) | `s3`, `dynamodb` |

---

## 4. Services

### Auth Service

**Repository:** `fleetops-auth-service` | **Port:** 8080 | **Schema:** `auth_db`

Handles user registration, login, and JWT issuance. This is the only service that issues tokens — all other services only validate them.

**User Roles:**

| Role | Capabilities |
|---|---|
| `ROLE_DRIVER` | View own vehicle, update own mileage, create service requests |
| `ROLE_MANAGER` | View all vehicles and requests, assign technicians, approve requests |
| `ROLE_ADMIN` | Full access including user management and vehicle creation |

**API Endpoints:**

| Method | Path | Access | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | Public | Register new user (defaults to DRIVER role) |
| `POST` | `/api/auth/login` | Public | Login and receive JWT as HttpOnly cookie |
| `POST` | `/api/auth/logout` | Authenticated | Clear JWT cookie |
| `GET` | `/api/auth/me` | Authenticated | Get current user profile |
| `POST` | `/api/audit/events` | Authenticated | Write an audit event to application log |

---

### Vehicle Service

**Repository:** `fleetops-vehicle-service` | **Port:** 8080 | **Schema:** `vehicle_db`

The core fleet catalog. Manages all vehicles, GPS location tracking, service and insurance alerts, and the AI-powered fleet health advisor.

**Vehicle Lifecycle:**

```
ACTIVE  →  IN_SERVICE  →  ACTIVE
       ↘  BREAKDOWN   →  ACTIVE
       ↘  RETIRED
```

**API Endpoints:**

| Method | Path | Access | Description |
|---|---|---|---|
| `GET` | `/api/vehicles` | Authenticated | List vehicles (DRIVER sees own vehicle only) |
| `GET` | `/api/vehicles/{id}` | Authenticated | Get vehicle detail |
| `POST` | `/api/vehicles` | ADMIN | Add vehicle to fleet |
| `PUT` | `/api/vehicles/{id}` | ADMIN | Update vehicle details |
| `DELETE` | `/api/vehicles/{id}` | ADMIN | Remove vehicle from fleet |
| `PATCH` | `/api/vehicles/{id}/status` | MANAGER, ADMIN | Change vehicle status |
| `PATCH` | `/api/vehicles/{id}/mileage` | DRIVER (own), ADMIN | Update current mileage |
| `GET` | `/api/vehicles/alerts/insurance` | MANAGER, ADMIN | Vehicles with insurance expiring within 30 days |
| `GET` | `/api/vehicles/alerts/service` | MANAGER, ADMIN | Vehicles with overdue or due-soon service |
| `GET` | `/api/vehicles/dashboard` | MANAGER, ADMIN | Fleet KPI summary |
| `GET` | `/api/vehicles/ai/fleet-analysis` | MANAGER, ADMIN | AI fleet health report |
| `POST` | `/api/documents/presigned-url` | Authenticated | Generate S3 presigned URL for document upload |
| `GET` | `/api/documents/vehicle/{vehicleNumber}` | Authenticated | List all documents stored in S3 for a vehicle |
| `POST` | `/api/tracking/ping` | Authenticated | Submit a GPS telemetry ping (from device or simulator) |
| `GET` | `/api/tracking/live` | MANAGER, ADMIN | Latest position for every vehicle (frontend polls every 3s) |
| `GET` | `/api/tracking/vehicle/{id}` | Authenticated | Latest GPS position for a specific vehicle |
| `GET` | `/api/tracking/vehicle/{id}/history` | MANAGER, ADMIN | Last 50 telemetry pings for a vehicle |

---

### Maintenance Service

**Repository:** `fleetops-maintenance-service` | **Port:** 8080 | **Schema:** `maintenance_db`

Manages per-technician maintenance task queues and vehicle media (inspection photos, repair documents, dashcam clips). Media files are stored on a shared Amazon EFS volume mounted in every pod.

**API Endpoints:**

| Method | Path | Access | Description |
|---|---|---|---|
| `GET` | `/api/tasks` | Authenticated | List tasks in own queue |
| `POST` | `/api/tasks/add` | Authenticated | Add task — to own queue; MANAGER/ADMIN can target any user via `?username=` |
| `DELETE` | `/api/tasks/remove/{taskId}` | Authenticated | Remove a task from own queue |
| `DELETE` | `/api/tasks/clear` | Authenticated | Clear own task queue |
| `POST` | `/api/tasks/alarms/broadcast` | MANAGER, ADMIN | Broadcast SNS alerts for all overdue vehicles |
| `GET` | `/api/media/catalog` | Authenticated | List media files for a vehicle |
| `GET` | `/api/media/file/{filename}` | Authenticated | Download a media file |
| `POST` | `/api/media/upload` | Authenticated | Upload a media file |

**Media storage:**

- EFS mount path: `/var/www/fleetops/shared-media`
- Allowed file types: `jpg`, `jpeg`, `png`, `pdf`, `mp4`
- File naming: `<vehicleNumber>_<timestamp>.<ext>`
- Path traversal protection: upload path is validated against the mount directory
- The EFS volume is shared across all maintenance-service pods on all nodes — files uploaded to any pod are instantly available from all others

---

### Request Service

**Repository:** `fleetops-request-service` | **Port:** 8080 | **Schema:** `request_db`

Manages the full lifecycle of vehicle service requests. When a driver reports a breakdown or requests scheduled maintenance, this service orchestrates the workflow via AWS Step Functions.

**Request Lifecycle:**

```
OPEN  →  PENDING_APPROVAL  →  APPROVED  →  ASSIGNED  →  IN_PROGRESS  →  COMPLETED
                           ↘  REJECTED
```

**On request creation:**
1. Validates the vehicle is in `ACTIVE` status (via REST call to vehicle-service)
2. Acquires a pessimistic lock (`SELECT FOR UPDATE`) to prevent duplicate requests per vehicle
3. Updates vehicle status to `IN_SERVICE` or `BREAKDOWN`
4. Starts an AWS Step Functions execution to track the workflow

**On technician assignment:**
- Creates a maintenance task in the assigned technician's queue (REST call to maintenance-service)

**On completion or rejection:**
- Sets the vehicle back to `ACTIVE`

**API Endpoints:**

| Method | Path | Access | Description |
|---|---|---|---|
| `GET` | `/api/requests` | Authenticated | List requests (DRIVER sees own only) |
| `GET` | `/api/requests/{id}` | Authenticated | Get request detail |
| `GET` | `/api/requests/vehicle/{vehicleId}` | MANAGER, ADMIN | All requests for a vehicle |
| `POST` | `/api/requests` | DRIVER, ADMIN | Create a new service request |
| `PATCH` | `/api/requests/{id}/status` | MANAGER, ADMIN | Approve or reject a request |
| `PATCH` | `/api/requests/{id}/assign` | MANAGER, ADMIN | Assign to a technician |
| `PATCH` | `/api/requests/{id}/complete` | MANAGER, ADMIN | Mark as completed |

---

### Service-to-Service Communication

Backend services call each other via REST over the internal network. No service calls another's database directly.

| Caller | Called | Purpose |
|---|---|---|
| `request-service` | `vehicle-service` | Validate vehicle is ACTIVE before creating a request |
| `request-service` | `vehicle-service` | Update vehicle status to IN_SERVICE / BREAKDOWN / ACTIVE |
| `request-service` | `maintenance-service` | Create a task in technician's queue on assignment |

**In Kubernetes (production):** services resolve via K8s DNS:
```
http://fleetops-vehicle-service:8080
http://fleetops-maintenance-service:8080
```

**Locally (Docker Compose):** services resolve via Docker bridge network:
```
http://vehicle-service:8080
http://maintenance-service:8080
```

The JWT from the original request is forwarded in the `Authorization` header on all inter-service calls so the downstream service can authenticate the caller.

---

### Frontend

**Repository:** `fleetops-frontend` | **Port:** 80

A React single-page application served from an Nginx pod inside EKS. The frontend has no direct database access — it communicates entirely through the ALB using path-based routing.

**Views:**

| View | Access |
|---|---|
| Login | Public |
| Dashboard (fleet KPIs, alerts) | All roles |
| Fleet list and vehicle detail | DRIVER sees own; MANAGER/ADMIN see all |
| Maintenance task queue | All roles |
| Service requests | DRIVER sees own; MANAGER/ADMIN see all |
| AI Fleet Advisor | MANAGER, ADMIN |
| Media catalog and upload | All roles |

JWT is stored in an `HttpOnly` cookie set by the auth service on login. Axios is configured to send cookies on every request automatically — no JavaScript can read the token.

---

## 5. Security and Authentication

### JWT Authentication

```
POST /api/auth/login
  │
  ▼
auth-service validates username + password against auth_db
  │
  ▼
Issues JWT:
  {
    "sub": "username",
    "role": "ROLE_MANAGER",
    "exp": <24 hours from now>
  }
  Signed with JWT_SECRET (HMAC-SHA256)
  Set as: HttpOnly; Secure; SameSite=Strict cookie
  │
  ▼
Every subsequent request (to any service):
  │
  ├── JwtAuthenticationFilter reads JWT from cookie (or Authorization: Bearer header)
  ├── Verifies signature using same JWT_SECRET from Secrets Manager (via ESO)
  ├── Populates Spring SecurityContext with username and role
  └── @PreAuthorize("hasRole('MANAGER')") enforces per-method access control
```

Each service validates JWT **independently** — there is no call back to auth-service on every request. All services share the same `JWT_SECRET` value, which is pulled from AWS Secrets Manager via External Secrets Operator at pod startup.

### IRSA — Pod-Level AWS Access

Pods never have static AWS credentials. Instead, EKS uses **IRSA (IAM Roles for Service Accounts)** to give each pod an IAM role scoped to exactly what it needs.

**How it works:**

1. EKS has an OIDC provider endpoint registered with AWS IAM
2. Each Kubernetes ServiceAccount is annotated with an IAM role ARN
3. When a pod starts, the EKS mutating webhook injects two environment variables:
   - `AWS_WEB_IDENTITY_TOKEN_FILE` — path to a short-lived JWT (rotated automatically by kubelet)
   - `AWS_ROLE_ARN` — the IAM role this pod should assume
4. The AWS SDK reads these automatically and calls `STS:AssumeRoleWithWebIdentity`
5. STS verifies the JWT signature against the EKS OIDC provider's public key
6. STS returns temporary credentials (1-hour TTL, auto-refreshed)

**IRSA role assignments:**

| Kubernetes Service Account | IAM Role | Permissions |
|---|---|---|
| `fleetops-app` | `fleetops-prod-app-irsa-role` | Secrets Manager read, SSM read, S3 vehicle docs, Bedrock InvokeModel, SNS publish, Step Functions start |
| `external-secrets` | `fleetops-prod-external-secrets-role` | Secrets Manager read, SSM read, KMS decrypt |
| `aws-load-balancer-controller` | `fleetops-prod-alb-controller-role` | EC2 describe, ELB create/modify/delete, target group management |
| `cluster-autoscaler` | `fleetops-prod-cluster-autoscaler-role` | EC2 Auto Scaling describe and set capacity |
| `cloudwatch-agent` | `fleetops-prod-cloudwatch-agent-role` | CloudWatch metrics and logs write |
| `efs-csi-controller` | `fleetops-prod-efs-csi-role` | EFS mount management |

### GitHub Actions OIDC

GitHub Actions CI/CD pipelines authenticate to AWS without any stored AWS access keys. GitHub mints a short-lived OIDC JWT for each workflow run. The workflow exchanges it for temporary AWS credentials using `STS:AssumeRoleWithWebIdentity`.

Two IAM roles are available to CI/CD:

| Role | Used By | Permissions |
|---|---|---|
| `fleetops-prod-github-actions-role` | Terraform pipeline | AdministratorAccess (full infra management) |
| `fleetops-prod-github-actions-ecr-role` | Service build pipelines | ECR push to specific repos only |

Trust condition on both roles: `"repo:FleetOps-V2/fleetops-*:*"` — only workflows from within this GitHub organization can assume them.

---

## 6. Secret Management

No secret values are stored in Git, Kubernetes manifests, or environment variable files. All secrets originate in AWS and flow into pods at runtime via External Secrets Operator (ESO).

```
AWS Secrets Manager                        AWS SSM Parameter Store
├── fleetops/prod/db                       /fleetops/prod/redis/endpoint
│     username, password, host             /fleetops/prod/sns/insurance-alerts-arn
├── fleetops/prod/jwt                      /fleetops/prod/sns/service-alerts-arn
│     jwt_secret                           /fleetops/prod/app/base-url
├── fleetops/prod/github-pat               /fleetops/prod/app/spring-profile
│     token (GitHub PAT for CI/CD)
└── fleetops/prod/lambda-service-credentials
      password (Lambda → ALB auth)

              │
              │   External Secrets Operator
              │   IRSA → fleetops-prod-external-secrets-role
              │   Refreshes every 1 hour
              ▼

Kubernetes Secrets (namespace: fleetops-prod)
├── fleetops-postgres-secret        POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_HOST
├── fleetops-app-secret             JWT_SECRET
├── fleetops-redis-secret           REDIS_HOST
├── fleetops-sns-secret             INSURANCE_SNS_TOPIC_ARN, SERVICE_SNS_TOPIC_ARN
└── fleetops-lambda-service-secret  LAMBDA_SERVICE_PASSWORD

              │
              │   Mounted as environment variables
              ▼

Spring Boot application reads values at startup via @Value("${POSTGRES_PASSWORD}") etc.
```

---

## 7. AI Fleet Advisor

The AI fleet advisor provides fleet managers with a health score (0–100) and per-vehicle maintenance recommendations. It runs inside `fleetops-vehicle-service` and is powered by Amazon Bedrock.

### How It Works

1. **Data collection** — Queries all `ACTIVE` vehicles from `vehicle_db`
2. **Filtering** — Keeps only vehicles where at least one is true:
   - Service date is overdue, or current mileage ≥ next service mileage
   - Insurance expires within 30 days
3. **Prompt construction** — Builds a structured plain-English prompt with exact figures (km overdue, days until service, insurance expiry date)
4. **Bedrock call** — Sends to `amazon.nova-lite-v1:0` via the Converse API (max 8 vehicles, max 2500 tokens)
5. **Response parsing** — Strips any markdown code fences, parses JSON into `FleetAnalysisResponse`
6. **Caching** — Result is cached in Redis for 5 minutes (`@Cacheable`)
7. **Audit logging** — Every call (cached or not) is written to `ai_analysis_audit` with timestamp, requester, score, recommendation count, and latency

### Response Structure

```json
{
  "fleetHealthScore": 72,
  "summary": "Fleet is in acceptable condition but 3 vehicles require immediate attention.",
  "recommendations": [
    {
      "vehicleId": 3,
      "vehicleNumber": "TRK-001",
      "priority": "HIGH",
      "taskType": "ROUTINE_SERVICE",
      "action": "Schedule immediate service for TRK-001",
      "reasoning": "TRK-001 is 200 km past its service interval and 12 days overdue. Continued operation risks engine damage and unplanned breakdown, with estimated repair cost 4–8x a routine service. Complete within 48 hours.",
      "confidence": 95
    }
  ]
}
```

### Caching Design

The caching uses Spring `@Cacheable` with a self-injected proxy to ensure the audit log is always written, even when the response is served from cache:

```java
// Outer method — never cached. Writes audit on every call.
public FleetAnalysisResponse analyseFleet(String requestedBy) {
    FleetAnalysisResponse result = self.invokeBedrockCached();  // goes through AOP proxy
    auditRepository.save(audit);
    return result;
}

// Inner method — cached. Only calls Bedrock on cache miss.
@Cacheable(value = "fleetAnalysis", key = "'current'")
public FleetAnalysisResponse invokeBedrockCached() { ... }
```

Cache TTL is configured in `application.yml`:

```yaml
spring:
  cache:
    redis:
      time-to-live: 300000   # 5 minutes in milliseconds
```

---

## 8. Database

A single RDS PostgreSQL 15.7 instance (`db.t3.micro`) hosts four schemas — one per microservice. Services never share a schema or query each other's tables directly.

| Schema | Owner Service | Key Tables |
|---|---|---|
| `auth_db` | fleetops-auth-service | `users` |
| `vehicle_db` | fleetops-vehicle-service | `vehicles`, `vehicle_telemetry`, `ai_analysis_audit` |
| `request_db` | fleetops-request-service | `service_requests` |
| `maintenance_db` | fleetops-maintenance-service | `maintenance_queue`, `pending_task` |

Each service uses its own DB user with access restricted to its own schema. Cross-service data needs are met by REST API calls, not direct DB queries.

**Schema migrations** are handled by Flyway, which runs automatically on service startup and applies any pending migration scripts in version order.

---

## 9. Infrastructure

All AWS infrastructure is managed by Terraform in `fleetops-terraform/`. Remote state is stored in S3 (`fleetops-terraform-state-johan`) with DynamoDB locking (`fleetops-terraform-locks`).

### Terraform Module Map

| Module | What It Creates |
|---|---|
| `modules/networking` | VPC, public/private/database subnets, NAT gateway, route tables, VPC interface and gateway endpoints, VPC Flow Logs (ALL traffic, 365-day retention) |
| `modules/kms` | KMS customer-managed keys for RDS, Secrets Manager, S3, and events |
| `modules/iam` | IRSA roles for all cluster components, EKS node role, GitHub Actions OIDC roles |
| `modules/rds` | RDS PostgreSQL 15.7, KMS-encrypted, multi-AZ subnet group, automated backups |
| `modules/redis` | ElastiCache Redis 7.1 single-node cluster |
| `modules/s3` | Vehicle documents bucket — KMS encrypted, versioning enabled |
| `modules/efs` | EFS filesystem + access point for shared maintenance media |
| `modules/secrets-manager` | Secret placeholders for DB credentials, JWT secret, GitHub PAT, Lambda credentials |
| `modules/ssm` | Parameter Store entries for Redis endpoint, SNS ARNs, app config |
| `modules/eks/cluster` | EKS control plane (v1.31), KMS-encrypted etcd, CloudWatch control plane logs |
| `modules/eks/nodegroup` | Managed node group: m7i-flex.large, AL2023, 2–5 nodes |
| `modules/eks/oidc` | OIDC provider for IRSA |
| `modules/eks/addons` | AWS Load Balancer Controller, ArgoCD, External Secrets Operator, Cluster Autoscaler, Metrics Server, EFS CSI Driver, CloudWatch Observability |
| `modules/acm` | ACM wildcard certificate (`*.fleetops.website`), Route 53 DNS validation |
| `modules/route53` | Hosted zone, A records: `fleetops.website → CloudFront`, `origin.fleetops.website → ALB`, `argocd.fleetops.website → ALB` |
| `modules/cloudfront` | CloudFront distribution, HTTPS redirect, custom origin pointing to ALB |
| `modules/waf` | WAF v2 WebACL: AWSManagedRulesCommonRuleSet, SQLiRuleSet, KnownBadInputsRuleSet |
| `modules/sns` | Insurance alert topic, service alert topic (email subscriptions) |
| `modules/lambda` | `alert-processor` function (Node.js 22.x), X-Ray tracing, IAM execution role |
| `modules/eventbridge` | Daily EventBridge rule `rate(1 day)` → Lambda trigger |
| `modules/step-functions` | Service request state machine |
| `modules/cloudtrail` | Multi-region CloudTrail trail → S3 + CloudWatch Logs |
| `modules/config` | AWS Config recorder + delivery channel |
| `modules/cloudwatch` | Alarms for RDS CPU, RDS connections, ALB 5xx rate, Lambda errors |

### Key Design Decisions

**ALB over NLB** — The Application Load Balancer (Layer 7) can inspect HTTP headers and route by URL path. AWS WAF only attaches to ALB. NLB (Layer 4) operates on raw TCP packets and cannot do path-based routing or WAF integration.

**Ingress + AWS ALB Controller over Kubernetes Gateway API** — The AWS Load Balancer Controller v1.8.1 has mature, production-tested Ingress support. Gateway API support only arrived in v2.x+. For path-based routing with a single service entry point, Ingress requires one YAML resource versus Gateway API's minimum of three (GatewayClass, Gateway, HTTPRoute).

**m7i-flex.large over t3/t4g** — Burstable instances (t-family) can throttle under sustained load when CPU credits are exhausted. The m7i-flex.large provides consistent baseline performance without credit limits — important for a production fleet platform running multiple JVM services.

**Single NAT Gateway** — One NAT Gateway covers both AZs. For a true multi-AZ production setup, each AZ would have its own NAT Gateway to eliminate cross-AZ NAT traffic. The current design accepts this trade-off.

---

## 10. GitOps with ArgoCD

ArgoCD runs inside the EKS cluster (`argocd` namespace) and watches the `fleetops-deployments` repository. Any change pushed to that repository is automatically applied to the cluster — no `kubectl apply` is ever run manually.

### App-of-Apps Pattern

A single root ArgoCD Application manages all child applications:

```
fleetops-root-prod                   (bootstrapped by Terraform)
  │  Points to: argocd/apps/prod/
  │
  ├── fleetops-platform-prod         ServiceAccount (IRSA-annotated), RBAC, ClusterSecretStore
  ├── fleetops-secrets-prod          ExternalSecret resources (triggers ESO to create K8s Secrets)
  ├── fleetops-networkpolicy-prod    NetworkPolicies (deny-all default, explicit allow per service)
  ├── fleetops-ingress-prod          ALB Ingress resource (path routing, TLS, WAF)
  ├── fleetops-auth-prod             auth-service Helm chart
  ├── fleetops-vehicle-prod          vehicle-service Helm chart
  ├── fleetops-maintenance-prod      maintenance-service Helm chart
  ├── fleetops-request-prod          request-service Helm chart
  ├── fleetops-frontend-prod         frontend Helm chart
  └── fleetops-db-init-prod          one-time DB schema seeding Job (runs once on first deploy)
```

**Sync policy:** `automated` with `prune: true` and `selfHeal: true`. If a resource drifts from the desired state in Git, ArgoCD corrects it automatically within minutes.

### deployments Repository Layout

```
fleetops-deployments/
├── argocd/apps/prod/               Child ArgoCD Application manifests
├── charts/
│   ├── auth-service/               Helm chart (Deployment, Service, HPA, PDB)
│   ├── vehicle-service/
│   ├── maintenance-service/
│   ├── request-service/
│   ├── frontend/
│   ├── ingress/                    ALB Ingress with TLS, WAF, and path rules
│   └── common/                     Shared chart helpers (_helpers.tpl)
├── k8s/
│   ├── platform/                   ServiceAccount, ClusterSecretStore, ExternalSecrets
│   ├── policies/                   NetworkPolicy (default-deny, allow lists)
│   └── prod/                       Production RBAC, db-init Job
└── environments/
    └── prod/infra-values.yaml      cert ARN, ALB SG ID, hosted zone ID — written by Terraform CI/CD
```

---

## 11. CI/CD Pipeline

### Service Build and Deploy Pipeline

Every microservice and the frontend has its own GitHub Actions pipeline. The actual Docker build, ECR push, and image tag update steps are defined once in `fleetops-github-workflows/` and called as reusable workflows.

```
Developer pushes to main branch
  │
  ▼
GitHub Actions pipeline starts
  │
  ├── Step 1: Build and test
  │     Maven (services): mvn verify
  │     npm (frontend): npm run build
  │
  ├── Step 2: Authenticate to AWS
  │     GitHub Actions OIDC → STS:AssumeRoleWithWebIdentity
  │     No AWS_ACCESS_KEY_ID stored in secrets
  │
  ├── Step 3: Docker build and push to ECR
  │     Image tagged: <git-sha>  and  latest
  │
  ├── Step 4: Update image tag in fleetops-deployments
  │     Writes new SHA to charts/<service>/values.yaml → image.tag
  │     Commits and pushes to fleetops-deployments (using GITHUB_PAT_FOR_DEPLOYMENTS)
  │
  └── Step 5: ArgoCD detects the change
        Applies new Deployment to EKS
        Kubernetes performs a rolling update (zero downtime)
```

### Terraform Infrastructure Pipeline

```
Push to fleetops-terraform main (environments/prod/ changes)
  │
  ▼
GitHub Actions pipeline starts
  │
  ├── Step 1: terraform init  (S3 backend, DynamoDB lock)
  │
  ├── Step 2: terraform plan  (saved as artifact)
  │
  ├── Step 3: Manual approval gate
  │     Reviewer inspects plan output before apply
  │
  ├── Step 4: terraform apply
  │
  └── Step 5: Extract Terraform outputs
        Writes ALB DNS name, cert ARN, SG IDs to
        fleetops-deployments/environments/prod/infra-values.yaml
```

### GitHub Secrets Required

These secrets must be configured at the GitHub organization level (or per repo):

| Secret | Purpose |
|---|---|
| `AWS_ACCOUNT_ID` | Your AWS account ID — used to construct the ECR registry URL |
| `AWS_REGION` | `us-east-1` |
| `GITHUB_PAT_FOR_DEPLOYMENTS` | PAT with `repo` scope — allows CI to push to `fleetops-deployments` |
| `ARGOCD_SERVER` | `argocd.fleetops.website` |
| `ARGOCD_AUTH_TOKEN` | ArgoCD API token for programmatic sync trigger |

---

## 12. Daily Alert Automation

An automated daily scan identifies vehicles with overdue maintenance or expiring insurance and sends email alerts to fleet managers via Amazon SNS.

```
EventBridge Scheduler
  rule: rate(1 day)
  │
  ▼
alert-processor Lambda (Node.js 22.x)
  │
  ├── Retrieves Lambda service credentials from Secrets Manager
  │
  ├── POST origin.fleetops.website/api/auth/login
  │     (service account with MANAGER role)
  │
  ├── GET /api/vehicles/alerts/insurance
  │     → vehicles with insurance expiring within 30 days
  │
  ├── GET /api/vehicles/alerts/service
  │     → vehicles with overdue or due-soon service
  │
  ├── SNS publish → fleetops-insurance-alerts topic
  │     → email subscribers (fleet managers)
  │
  └── SNS publish → fleetops-service-alerts topic
        → email subscribers (fleet managers)
```

The Lambda runs outside the VPC. It reaches the backend through the public DNS (`origin.fleetops.website`), which maps to the ALB. WAF rules allow its requests through. Lambda has X-Ray active tracing enabled.

---

## 13. Observability

| Layer | Tool | What It Captures |
|---|---|---|
| Pod logs and metrics | CloudWatch Observability EKS addon | Container stdout/stderr, CPU, memory, network — visible in Container Insights |
| EKS control plane | EKS Control Plane Logs | API server, audit, scheduler, controller-manager |
| Network traffic | VPC Flow Logs | Every accepted and rejected connection in the VPC (ALL type, 365-day retention) |
| API activity | AWS CloudTrail | Every AWS API call in the account, multi-region, logged to S3 |
| Compliance | AWS Config | Resource configurations and drift detection |
| Web traffic | AWS WAF Logs | All allowed and blocked requests, rule match details → CloudWatch |
| Application tracing | AWS X-Ray | Request traces for Lambda (active tracing enabled) |
| Alerting | CloudWatch Alarms | RDS CPU > 80%, RDS connections > 80 threshold, ALB 5xx rate spike, Lambda error rate |
| Deployment health | ArgoCD UI | Sync status, resource health, rollout history for all applications |

---

## 14. Technology Stack

### Application

| Component | Technology | Version |
|---|---|---|
| Backend language | Java | 21 (LTS) |
| Backend framework | Spring Boot | 3.5.14 |
| Security | Spring Security | 6.5.9 |
| JWT | JJWT | 0.12.3 |
| DB driver | PostgreSQL JDBC | 42.7.11 |
| DB migrations | Flyway | (Spring Boot BOM) |
| JSON | Jackson | 2.21.2 |
| Frontend framework | React | 19.2.5 |
| Frontend language | TypeScript | 6.0.2 |
| Frontend build | Vite | 8.0.9 |
| Frontend routing | React Router DOM | 7.15.0 |
| HTTP client | Axios | 1.16.0 |
| UI icons | Lucide React | 1.17.0 |

### Infrastructure

| Component | Technology | Version |
|---|---|---|
| Container orchestration | Amazon EKS | 1.31 |
| Node AMI | Amazon Linux 2023 | AL2023_x86_64_STANDARD |
| Node instance type | m7i-flex.large | min=2, desired=3, max=5 |
| IaC | Terraform | >= 1.6.0 |
| AWS provider | hashicorp/aws | 5.100.0 |
| Database | RDS PostgreSQL | 15.7 (db.t3.micro) |
| Cache | ElastiCache Redis | 7.1 (cache.t3.micro) |
| GitOps | ArgoCD | 6.7.11 (Helm chart) |
| Load balancer controller | AWS Load Balancer Controller | 1.8.1 |
| Secret sync | External Secrets Operator | 0.9.19 |
| Cluster autoscaler | cluster-autoscaler | 9.37.0 (image v1.33.0) |
| Metrics server | metrics-server | 3.12.1 |
| Observability | CloudWatch Observability addon | v6.2.0-eksbuild.1 |
| Shared storage driver | EFS CSI Driver | v2.0.7-eksbuild.1 |
| CDN | Amazon CloudFront | — |
| WAF | AWS WAF v2 | — |
| AI | Amazon Bedrock | amazon.nova-lite-v1:0 |
| Serverless | AWS Lambda | Node.js 22.x |
| Embedded server | Apache Tomcat | 10.1.55 |
| Networking | Netty | 4.1.135.Final |

---

## 15. Local Development

The full platform runs locally via Docker Compose using the `fleetops-infra` repository. No AWS account is needed for local development — AWS-specific features (Bedrock, Step Functions, SNS) are absent from the local stack.

### Prerequisites

- Docker and Docker Compose
- JDK 21 (to build service images)
- Node.js 22 (to build the frontend image)

### Start the Stack

**Step 1 — Create a `.env` file in `fleetops-infra/`:**

```bash
JWT_SECRET=local-dev-secret-key-minimum-32-chars
```

**Step 2 — Build and start all services:**

```bash
cd fleetops-infra/
docker compose up --build
```

Services start in dependency order: PostgreSQL and Redis must be healthy before the Spring Boot services start. Request-service waits for vehicle-service and maintenance-service. The gateway starts last.

### Local Endpoints

Everything is accessible through the Nginx gateway on port 8080, which mirrors the AWS ALB path routing:

| URL | Routes to |
|---|---|
| `http://localhost:8080` | Frontend (React SPA) |
| `http://localhost:8080/api/auth` | Auth Service |
| `http://localhost:8080/api/vehicles` | Vehicle Service |
| `http://localhost:8080/api/tracking` | Vehicle Service (TrackingController) |
| `http://localhost:8080/api/requests` | Request Service |
| `http://localhost:8080/api/tasks` | Maintenance Service |
| `http://localhost:8080/api/media` | Maintenance Service |
| `http://localhost:8080/api/audit` | Auth Service (AuditController) |
| `http://localhost:5432` | PostgreSQL (direct access) |
| `http://localhost:6379` | Redis (direct access) |

Individual service health checks are exposed via the gateway:

```
GET http://localhost:8080/health/auth
GET http://localhost:8080/health/vehicles
GET http://localhost:8080/health/requests
GET http://localhost:8080/health/maintenance
```

### Local vs Production Differences

| Aspect | Local (Docker Compose) | Production (EKS) |
|---|---|---|
| Routing | Nginx gateway (`nginx/gateway.conf`) | AWS ALB (via K8s Ingress) |
| Secrets | `.env` file | AWS Secrets Manager via ESO |
| AI Advisor | Not available (no Bedrock) | Amazon Bedrock |
| Step Functions | Not wired | AWS Step Functions |
| SNS Alerts | Not wired | AWS SNS → email |
| Spring profile | `dev` (seeds default users) | `prod` |
| Media storage | Local filesystem | Amazon EFS |
| Document storage (`/api/documents`) | Not routed — no nginx block for this path | Amazon S3 via presigned URL |

### Default Users (dev profile)

The `dev` Spring profile activates `DataInitializer`, which seeds these accounts on first startup:

| Username | Password | Role |
|---|---|---|
| `admin1` | `Admin@123` | ADMIN |
| `manager1` | `Manager@123` | MANAGER |
| `driver1` | `Driver@123` | DRIVER |
| `driver2` | `Driver@123` | DRIVER |
| `driver3` | `Driver@123` | DRIVER |

Seeding is idempotent — safe to run on every restart. Users already in the database are skipped.

### Stopping the Stack

```bash
docker compose down          # stop containers, keep data
docker compose down -v       # stop containers and delete postgres volume
```

---

## 16. Deployment Guide

### Prerequisites

| Tool | Minimum Version | Purpose |
|---|---|---|
| Terraform | 1.6.0 | Provision AWS infrastructure |
| AWS CLI | 2.x | EKS kubeconfig, CLI operations |
| kubectl | 1.28 | Kubernetes cluster management |
| helm | 3.x | Helm chart operations (optional, ArgoCD manages this) |

Ensure the AWS CLI is authenticated as a user with sufficient permissions to create IAM roles, EKS clusters, VPCs, and all other resources listed in the module map.

### Step 1 — Bootstrap Terraform Remote State

This creates the S3 bucket and DynamoDB table that all subsequent Terraform runs use for state storage and locking.

```bash
cd fleetops-terraform/bootstrap/
terraform init
terraform apply
```

Creates:
- S3 bucket: `fleetops-terraform-state-johan`
- DynamoDB table: `fleetops-terraform-locks`

### Step 2 — Provision Full Infrastructure

```bash
cd fleetops-terraform/environments/prod/
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

This takes approximately 25 minutes. It provisions everything in one run: VPC, EKS cluster, RDS, Redis, ECR repos, ACM certificate, CloudFront, WAF, Route53 zone, all IAM roles, ESO, ArgoCD, and the AWS Load Balancer Controller.

### Step 3 — Configure Domain Nameservers

After Terraform completes, retrieve the Route53 nameservers:

```bash
terraform output route53_nameservers
```

Go to your domain registrar and update the nameservers for `fleetops.website` to the four values returned above. DNS propagation typically takes 10–30 minutes.

### Step 4 — Configure GitHub Secrets

Add the five secrets listed in the [GitHub Secrets Required](#github-secrets-required) section to the GitHub organization secrets (or to each repo individually).

### Step 5 — Confirm SNS Email Subscriptions

Terraform creates SNS topics and subscribes the email addresses in `alert_emails` (set in `terraform.tfvars`). AWS sends a confirmation email to each address immediately after `terraform apply`. You must click **Confirm subscription** in that email before daily alerts will be delivered.

### Step 6 — Deploy All Services

Trigger the CI/CD pipeline for each service by pushing to their `main` branch, or run the GitHub Actions workflow manually from the Actions tab. ArgoCD will pick up each image tag update and deploy the services automatically.

### Step 7 — Get the ArgoCD Admin Password

ArgoCD generates an initial admin password on first install and stores it in a Kubernetes Secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Log in at https://argocd.fleetops.website with username `admin` and the password above. Change the password after first login (`argocd account update-password`).

### Step 8 — Verify Everything Is Running

```bash
# Connect to the cluster
aws eks update-kubeconfig --name fleetops-prod-eks --region us-east-1

# All nodes should be Ready
kubectl get nodes

# All pods should be 1/1 Running
kubectl get pods -n fleetops-prod

# All ArgoCD apps should be Synced + Healthy
kubectl get applications -n argocd

# Secrets should be synced
kubectl get externalsecrets -n fleetops-prod
```

Open https://fleetops.website — the login page should load.

---

## 17. Operations Reference

### Cluster Access

```bash
# Authenticate kubectl to the cluster
aws eks update-kubeconfig --name fleetops-prod-eks --region us-east-1
```

### Pod and Node Commands

```bash
# Node status
kubectl get nodes -o wide

# All pods in the application namespace
kubectl get pods -n fleetops-prod

# Pod logs (live tail)
kubectl logs -n fleetops-prod -l app=fleetops-vehicle-service -f --tail=100

# Describe a pod (events, resource limits, env vars)
kubectl describe pod -n fleetops-prod <pod-name>
```

### Secret and Config Commands

```bash
# Check all ExternalSecret sync status
kubectl get externalsecrets -n fleetops-prod

# Describe a specific secret sync (shows last sync time and any errors)
kubectl describe externalsecret fleetops-postgres-secret -n fleetops-prod

# List all Kubernetes Secrets (names only)
kubectl get secrets -n fleetops-prod
```

### ArgoCD Commands

```bash
# List all applications and their sync status
kubectl get applications -n argocd

# Force sync a specific app
kubectl -n argocd patch application fleetops-vehicle-prod \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'

# ArgoCD CLI (if installed)
argocd login argocd.fleetops.website
argocd app list
argocd app sync fleetops-vehicle-prod
argocd app get fleetops-vehicle-prod
```

### Terraform Commands

```bash
cd fleetops-terraform/environments/prod/

# List all managed resources
terraform state list

# Show current outputs (ALB DNS, cert ARN, etc.)
terraform output

# Refresh state against actual AWS resources
terraform refresh
```

---

## 18. Troubleshooting

### 502 Bad Gateway from the browser

The ALB reached a pod but got no valid response, or no healthy target exists.

1. Check pod status — look for `CrashLoopBackOff` or `Error`:
   ```bash
   kubectl get pods -n fleetops-prod
   ```
2. Check the Ingress resource for ALB provisioning errors:
   ```bash
   kubectl describe ingress -n fleetops-prod
   ```
3. Check ALB target group health in the AWS console (EC2 → Target Groups → filter by `fleetops`). Unhealthy targets indicate failed health checks.
4. Check if ExternalSecrets have synced — a missing secret causes a pod to fail to start:
   ```bash
   kubectl get externalsecrets -n fleetops-prod
   ```

---

### Pod in CrashLoopBackOff

The pod starts and crashes repeatedly. Common causes: DB connection failure, missing environment variable, or failed ESO secret sync.

```bash
# View crash logs
kubectl logs -n fleetops-prod <pod-name> --previous

# Check events (scheduling issues, image pull failures)
kubectl describe pod -n fleetops-prod <pod-name>
```

---

### ExternalSecret not syncing

The Kubernetes Secret is not being created or updated from AWS.

```bash
# Check ClusterSecretStore is connected
kubectl get clustersecretstore

# Check specific ExternalSecret for errors
kubectl describe externalsecret fleetops-postgres-secret -n fleetops-prod
```

Common causes:
- The IRSA role on the `external-secrets` ServiceAccount does not have `secretsmanager:GetSecretValue` permission for that secret ARN
- The secret path in the ExternalSecret YAML does not match the actual path in Secrets Manager
- KMS key policy does not allow the IRSA role to decrypt

---

### ArgoCD app shows OutOfSync

The cluster state does not match what is in Git.

```bash
kubectl describe application <app-name> -n argocd | grep -A 10 "Conditions"
```

If the diff shows a resource that should not exist, check if `prune: true` is set on the app's sync policy. If the diff is due to a label or annotation injected by another controller, add the field to the app's `ignoreDifferences` list.

To force a full resync:
```bash
argocd app sync <app-name> --force
```

---

### Terraform plan shows unexpected changes

Before applying, review whether the change is expected:

- `aws_eks_node_group` `desired_size` changes are expected — Cluster Autoscaler owns this value and Terraform is configured to ignore it
- `aws_secretsmanager_secret_version` `secret_string` changes indicate someone manually updated a secret — Terraform is configured to ignore these too
- `aws_iam_openid_connect_provider` `thumbprint_list` changes are AWS-managed certificate rotations — also ignored

If a change is unexpected, investigate in the AWS console before applying.
