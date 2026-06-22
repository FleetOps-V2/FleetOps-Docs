# FleetOps V2 — Enterprise Fleet Management Platform

FleetOps V2 is a production-grade fleet maintenance and vehicle tracking platform deployed on AWS EKS using a full GitOps pipeline. It consists of four Spring Boot microservices, a React frontend, AI-powered maintenance advisory via Amazon Bedrock, and a fully automated CI/CD pipeline from code push to production.

**Live URL:** https://fleetops.website  
**ArgoCD Dashboard:** https://argocd.fleetops.website  
**AWS Account:** 538661800892 (us-east-1)

---

## Repository Map

```
FleetOps-V2/
├── fleetops-frontend/              React SPA (Vite + TypeScript)
├── fleetops-auth-service/          Spring Boot — JWT auth, user management
├── fleetops-vehicle-service/       Spring Boot — fleet management, Bedrock AI advisor
├── fleetops-maintenance-service/   Spring Boot — scheduled maintenance, SNS alerts
├── fleetops-request-service/       Spring Boot — service requests, Step Functions
├── fleetops-terraform/             All AWS infrastructure as code
├── fleetops-deployments/           Kubernetes manifests + ArgoCD app definitions
├── fleetops-github-workflows/      Shared reusable CI/CD workflow templates
└── FleetOps-Docs/                  This documentation hub
```

---

## Documentation Index

| Document | What It Covers |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Full system architecture — EKS, GitOps, AWS services, data flow |
| [DEPLOYMENT_AWS.md](DEPLOYMENT_AWS.md) | How to deploy, CI/CD pipeline, day-of-eval apply steps |
| [AI_INTEGRATION.md](AI_INTEGRATION.md) | Amazon Bedrock Fleet Maintenance Advisor — live implementation |

---

## Platform at a Glance

| Layer | Technology |
|---|---|
| Frontend | React 18 + Vite + TypeScript, served via Nginx on EKS |
| Backend | 4 x Spring Boot 3.x microservices |
| Database | Amazon RDS PostgreSQL 15 (multi-schema, one instance) |
| Cache | Amazon ElastiCache Redis 7 |
| Container Orchestration | Amazon EKS 1.31 (managed node groups, m7i-flex.large) |
| GitOps | ArgoCD v2 (App-of-Apps pattern) |
| Infrastructure as Code | Terraform (modular, remote state on S3 + DynamoDB lock) |
| CI/CD | GitHub Actions with OIDC — no static AWS keys in pipelines |
| Secret Management | AWS Secrets Manager + SSM via External Secrets Operator (ESO) |
| AI Advisory | Amazon Bedrock (Claude 3 Haiku via Converse API) |
| CDN & Security | Amazon CloudFront + AWS WAF v2 |
| DNS & TLS | Route 53 + ACM wildcard certificate (*.fleetops.website) |
| Workflow Automation | AWS Step Functions (service request state machine) |
| Alerting | Amazon SNS (insurance expiry + service overdue notifications) |
| Scheduled Jobs | Amazon EventBridge → Lambda (daily alert scan → SNS) |
| Audit & Compliance | AWS CloudTrail, AWS Config, VPC Flow Logs, KMS encryption |

---

## Services

| Service | Key Responsibility |
|---|---|
| `fleetops-auth-service` | JWT issuance, user registration/login, ROLE_ADMIN / ROLE_MANAGER / ROLE_DRIVER |
| `fleetops-vehicle-service` | Vehicle CRUD, fleet health scoring, Redis cache, Bedrock AI advisor |
| `fleetops-maintenance-service` | Maintenance task lifecycle, SNS alert broadcasts |
| `fleetops-request-service` | Service request creation and tracking, Step Functions state machine |
| `fleetops-frontend` | React SPA — dashboard, fleet view, AI advisor panel, request management |

---

## Key Design Decisions

- **No static AWS credentials anywhere** — all pods use IRSA (IAM Roles for Service Accounts); CI/CD uses OIDC
- **No secrets in Git** — all secrets live in AWS Secrets Manager and are injected at runtime via ESO
- **GitOps-only deployments** — no `kubectl apply` by hand; ArgoCD owns cluster state
- **Single ALB shared** — all Ingresses use `group.name: fleetops` so one ALB serves all services with host-based routing
- **CloudFront in front of everything** — users never hit the ALB directly; CloudFront adds WAF, caching, and HTTPS termination
