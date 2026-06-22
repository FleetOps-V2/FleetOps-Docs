# FleetOps V2 — System Architecture

---

## High-Level Architecture

```
                        [ User Browser ]
                               │ HTTPS
                               ▼
                    ┌─────────────────────┐
                    │   Amazon CloudFront  │  WAF v2 (rate limit, geo block)
                    │   fleetops.website   │  TLS via ACM wildcard cert
                    └──────────┬──────────┘
                               │ HTTPS → origin.fleetops.website
                               ▼
                    ┌─────────────────────┐
                    │    AWS ALB (shared)  │  group.name: fleetops
                    │  k8s-fleetops-*      │  host-based routing
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
     [fleetops.website]  [argocd.fleetops.website]  (future services)
              │
    ┌─────────▼──────────────────────────────────────┐
    │              Amazon EKS Cluster                 │
    │           Namespace: fleetops-prod              │
    │                                                 │
    │  ┌────────────┐  ┌────────────┐                │
    │  │  frontend  │  │auth-service│                │
    │  │  (Nginx)   │  │(Spring Boot│                │
    │  └────────────┘  └─────┬──────┘               │
    │                        │ JWT validation         │
    │  ┌────────────┐  ┌─────▼──────┐               │
    │  │  vehicle   │  │maintenance │                │
    │  │  service   │  │  service   │                │
    │  └──────┬─────┘  └─────┬──────┘               │
    │         │               │                      │
    │  ┌──────▼─────┐        │                      │
    │  │  request   │◄───────┘                      │
    │  │  service   │                                │
    │  └────────────┘                                │
    └──────────┬──────────────────────────────────────┘
               │
    ┌──────────┼──────────────────────┐
    │          │                      │
    ▼          ▼                      ▼
[RDS Postgres] [ElastiCache Redis]  [AWS Services]
                                    (Bedrock, SNS,
                                     Step Functions,
                                     Secrets Manager)
```

---

## Infrastructure Layer (Terraform)

All AWS resources are defined in `fleetops-terraform/` with a modular structure. Remote state is stored in S3 with DynamoDB locking.

### Terraform Module Map

| Module | Resources Created |
|---|---|
| `modules/networking` | VPC, public/private subnets, NAT gateway, route tables, VPC flow logs |
| `modules/eks/cluster` | EKS control plane, KMS encryption for secrets, CloudWatch log groups |
| `modules/eks/nodegroup` | Managed node group (m7i-flex.large, 2–5 nodes) |
| `modules/eks/addons` | ArgoCD, ALB Controller, External Secrets Operator, Cluster Autoscaler, Metrics Server |
| `modules/rds` | RDS PostgreSQL 15, multi-AZ subnet group, KMS encryption |
| `modules/redis` | ElastiCache Redis 7 cluster |
| `modules/acm` | ACM wildcard certificate (*.fleetops.website) with DNS validation |
| `modules/cloudfront` | CloudFront distribution pointing to origin.fleetops.website |
| `modules/waf` | WAF v2 web ACL (rate limiting, managed rule groups) |
| `modules/route53` | Hosted zone, A records (fleetops.website → CloudFront, origin → ALB, argocd → ALB) |
| `modules/iam` | IRSA roles (app, ALB controller, external secrets, cluster autoscaler), GitHub Actions OIDC role |
| `modules/secrets-manager` | fleetops/prod/db, fleetops/prod/jwt, fleetops/prod/github-pat, fleetops/prod/lambda-service-credentials |
| `modules/ssm` | /fleetops/prod/redis/endpoint, /fleetops/prod/sns/*, /fleetops/prod/app/* |
| `modules/kms` | KMS keys for RDS, Secrets Manager, S3 |
| `modules/s3` | Vehicle documents bucket (KMS encrypted, versioning enabled) |
| `modules/lambda` | `alert-processor` (Node.js) — daily alert scan: fetches vehicle alerts, publishes to SNS |
| `modules/step-functions` | Service request state machine |
| `modules/sns` | Insurance alert topic, service alert topic |
| `modules/eventbridge` | Daily rule `rate(1 day)` → triggers `alert-processor` Lambda |
| `modules/cloudtrail` | CloudTrail trail → S3 for API audit logging |
| `modules/config` | AWS Config recorder for compliance checks |

### Networking Design

```
VPC: 10.2.0.0/16
├── Public Subnets (ALB, NAT)
│   ├── 10.2.1.0/24  (us-east-1a)
│   └── 10.2.2.0/24  (us-east-1b)
└── Private Subnets (EKS nodes, RDS, Redis)
    ├── 10.2.10.0/24 (us-east-1a)
    └── 10.2.11.0/24 (us-east-1b)
```

EKS nodes, RDS, and Redis run in private subnets. Only the ALB and NAT gateway are in public subnets. All outbound internet traffic from pods goes through the NAT gateway.

---

## GitOps Layer (ArgoCD + fleetops-deployments)

ArgoCD runs inside the EKS cluster. It watches the `fleetops-deployments` GitHub repository and automatically applies any changes to the cluster.

### App-of-Apps Pattern

```
fleetops-root-prod (ArgoCD Application)
├── Points to: argocd/apps/prod/
└── Manages these child apps:
    ├── fleetops-platform-prod    → k8s/platform/     (ServiceAccount, RBAC, ESO stores)
    ├── fleetops-secrets-prod     → k8s/platform/external-secrets.yaml
    ├── fleetops-networkpolicy-prod → k8s/policies/
    ├── fleetops-ingress-prod     → charts/ingress/   (ALB Ingress resource)
    ├── fleetops-auth-prod        → charts/auth-service/
    ├── fleetops-vehicle-prod     → charts/vehicle-service/
    ├── fleetops-maintenance-prod → charts/maintenance-service/
    ├── fleetops-request-prod     → charts/request-service/
    ├── fleetops-frontend-prod    → charts/frontend/
    └── fleetops-db-init-prod     → k8s/prod/db-init/ (one-time DB schema migration)
```

The root app is created by Terraform via `kubernetes_manifest` resource (in `modules/eks/addons/main.tf`). Once Terraform creates it, ArgoCD self-manages everything from that point forward.

### Repository Layout

```
fleetops-deployments/
├── argocd/apps/prod/          ArgoCD Application manifests (child apps)
├── charts/                    Helm charts for each service
│   ├── auth-service/
│   ├── vehicle-service/
│   ├── maintenance-service/
│   ├── request-service/
│   ├── frontend/
│   ├── ingress/               ALB Ingress with ACM cert + WAF SG
│   └── common/                Shared chart helpers
├── k8s/
│   ├── platform/              ServiceAccount (IRSA annotated), ClusterSecretStore, ESO ExternalSecrets
│   ├── policies/              NetworkPolicy (deny-all default, explicit allow)
│   ├── prod/                  Production-specific overrides (RBAC, db-init job)
│   └── base/                  Base manifests shared across environments
└── environments/
    └── prod/infra-values.yaml  certArn, albSgId, hostedZoneId (written by Terraform pipeline)
```

---

## Secret Management (ESO + AWS)

No secrets are stored in Kubernetes manifests or Git. All secrets come from AWS.

```
AWS Secrets Manager                    AWS SSM Parameter Store
├── fleetops/prod/db          ─────►  /fleetops/prod/redis/endpoint
│   username, password, host          /fleetops/prod/sns/insurance-alerts-arn
├── fleetops/prod/jwt                 /fleetops/prod/sns/service-alerts-arn
│   jwt_secret                        /fleetops/prod/app/base-url
├── fleetops/prod/github-pat          /fleetops/prod/app/spring-profile
│   token
└── fleetops/prod/lambda-service-credentials
    password
         │
         │  External Secrets Operator (ClusterSecretStore)
         │  IRSA → fleetops-prod-external-secrets-role
         ▼
Kubernetes Secrets (in fleetops-prod namespace)
├── fleetops-postgres-secret     POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_HOST
├── fleetops-app-secret          JWT_SECRET
├── fleetops-redis-secret        REDIS_HOST
├── fleetops-sns-secret          INSURANCE_SNS_TOPIC_ARN, SERVICE_SNS_TOPIC_ARN
└── fleetops-lambda-service-secret  LAMBDA_SERVICE_PASSWORD
```

---

## IRSA (IAM Roles for Service Accounts)

Pods never use static AWS credentials. Each component has its own IAM role assigned via Kubernetes Service Account annotation.

| Service Account | IAM Role | Permissions |
|---|---|---|
| `fleetops-app` (all app pods) | `fleetops-prod-app-irsa-role` | Secrets Manager read, SSM read, S3 (vehicle docs), Bedrock InvokeModel |
| `external-secrets` | `fleetops-prod-external-secrets-role` | Secrets Manager read, SSM read |
| `aws-load-balancer-controller` | `fleetops-prod-alb-controller-role` | EC2/ELB management |
| `cluster-autoscaler` | `fleetops-prod-cluster-autoscaler-role` | EC2 Auto Scaling |

---

## CI/CD Pipeline (GitHub Actions)

Every service has its own pipeline. Shared workflow templates live in `fleetops-github-workflows/`.

### Pipeline Flow (per microservice)

```
Code Push to main
       │
       ▼
1. Build & Test (Maven / npm)
       │
       ▼
2. Docker Build + Push to ECR
   (tag: SHA + latest)
       │
       ▼
3. Update image tag in fleetops-deployments
   (charts/<service>/values.yaml → image.tag)
       │
       ▼
4. ArgoCD detects change in deployments repo
       │
       ▼
5. ArgoCD applies new Deployment to EKS
   (rolling update, zero downtime)
```

Authentication to AWS uses **OIDC** — GitHub Actions assumes `fleetops-prod-github-actions-role` without any stored AWS keys.

### Terraform Pipeline (fleetops-terraform)

```
Push to main (environments/prod/ changes)
       │
       ▼
1. terraform init (S3 backend)
       │
       ▼
2. terraform plan → saved as tfplan artifact
       │
       ▼
3. Manual approval gate
       │
       ▼
4. terraform apply tfplan
       │
       ▼
5. Extract outputs (ALB DNS, cert ARN, etc.)
   Write to fleetops-deployments/environments/prod/infra-values.yaml
```

---

## Traffic Flow (Request Path)

```
User → fleetops.website
  → CloudFront (WAF check, cache check)
  → origin.fleetops.website (Route53 alias → ALB)
  → ALB (host-based routing → EKS NodePort)
  → Nginx Ingress → Kubernetes Service → Pod

Within the cluster (service-to-service):
  → Pod calls http://fleetops-<service>:8080
  → Kubernetes DNS resolves to ClusterIP Service
  → Routes to healthy pod (liveness/readiness probes)

Auth enforcement:
  → Each backend service validates JWT via Spring Security filter
  → Token issued by auth-service, validated independently (shared JWT_SECRET from ESO)
```

---

## Database Schema

One RDS PostgreSQL instance hosts four schemas — one per service:

| Schema | Owner Service | Key Tables |
|---|---|---|
| `auth_db` | auth-service | users, roles |
| `vehicle_db` | vehicle-service | vehicles, ai_analysis_audit |
| `request_db` | request-service | service_requests |
| `maintenance_db` | maintenance-service | maintenance_tasks |

Each service connects only to its own schema using credentials from Secrets Manager. Cross-service data access goes through REST APIs, never direct DB calls.

---

## Observability

| Tool | What It Monitors |
|---|---|
| EKS Control Plane Logs | API server, audit, scheduler — sent to CloudWatch |
| VPC Flow Logs | All network traffic in/out of VPC |
| AWS CloudTrail | All API calls in the account |
| AWS Config | Resource compliance rules |
| WAF Logs | Blocked/allowed requests → CloudWatch |
| Pod logs | `kubectl logs` / CloudWatch Container Insights |
| ArgoCD UI | Sync status, health status for all apps |
