# FleetOps V2 — Complete Technical Mastery Guide
> Live site: https://fleetops.website | ArgoCD: https://argocd.fleetops.website
> AWS Account: 538661800892 | Region: us-east-1 | EKS: v1.31 | Terraform: ≥1.6.0

---

## HOW TO USE THIS GUIDE
Each section covers a phase of the evaluation. Read it front-to-back once to build the mental model, then use the Q&A sections to drill. Every answer here is traceable to an actual file in this repository.

---

# PHASE 1 — REPOSITORY STRUCTURE & PURPOSE

## Repository Map

```
Fleetops-V2/
├── fleetops-auth-service/          Spring Boot JWT auth microservice
├── fleetops-vehicle-service/       Spring Boot fleet catalog + Bedrock AI
├── fleetops-maintenance-service/   Spring Boot maintenance tasks + SNS + EFS
├── fleetops-request-service/       Spring Boot service requests + Step Functions
├── fleetops-frontend/              React 18 + Vite + TypeScript SPA
├── fleetops-terraform/             All AWS infrastructure as Terraform modules
├── fleetops-deployments/           Helm charts + K8s manifests + ArgoCD apps
├── fleetops-github-workflows/      Reusable CI/CD workflow templates
├── fleetops-infra/                 Local dev Docker Compose stack
└── FleetOps-Docs/                  Architecture docs + this guide
```

### Why each repository exists and what breaks without it

| Repo | Purpose | Breaks without it |
|------|---------|-------------------|
| `fleetops-auth-service` | Issues and validates JWT tokens; all other services trust its tokens | Every API call fails — no auth |
| `fleetops-vehicle-service` | Fleet catalog, KPIs, Bedrock AI advisor, S3 document presigned URLs | No vehicle data, no AI features |
| `fleetops-maintenance-service` | Maintenance tasks, EFS file uploads, SNS insurance/service alerts | No maintenance workflows, no file storage |
| `fleetops-request-service` | Orchestrates service requests via AWS Step Functions state machine | No request lifecycle management |
| `fleetops-frontend` | React SPA served via Nginx, accessed through CloudFront | Users have no UI |
| `fleetops-terraform` | Provisions every AWS resource (VPC, EKS, RDS, Redis, S3, SNS, Lambda…) | No cloud infrastructure to run anything on |
| `fleetops-deployments` | Helm charts + ArgoCD Applications — defines desired Kubernetes state | ArgoCD has nothing to sync; services can't deploy |
| `fleetops-github-workflows` | Reusable CI/CD templates used by all 5 service repos | Each service would have duplicated, diverging pipelines |
| `fleetops-infra` | Docker Compose for local dev | Developers can't run the stack locally |

---

## Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | Java | 21 (LTS) |
| Framework | Spring Boot | 3.5.14 |
| Frontend | React + Vite + TypeScript | 18 / 5 / 5 |
| Container runtime | Docker (multi-stage builds) | — |
| Orchestration | Amazon EKS | 1.31 |
| GitOps | ArgoCD | 6.7.11 (Helm chart) |
| IaC | Terraform | ≥ 1.6.0 |
| CI/CD | GitHub Actions | — |
| Database | PostgreSQL on RDS | 15.7 |
| Cache | Redis on ElastiCache | 7 |
| Secret sync | External Secrets Operator | 0.9.19 |
| Load balancer | AWS ALB (via LBC controller) | 1.8.1 |
| CDN | CloudFront + WAFv2 | — |
| AI | Amazon Bedrock (Nova Lite) | amazon.nova-lite-v1:0 |
| Autoscaling | Cluster Autoscaler + HPA | 9.37.0 / built-in |
| Observability | CloudWatch Observability addon | v6.2.0 |
| Shared storage | EFS (CSI driver) | v2.0.7 |
| Code quality | SonarQube + Snyk + Trivy | — |

---

# PHASE 2 — SYSTEM ARCHITECTURE DEEP DIVE

## High-Level Request Path (user → backend)

```
Browser
  │  HTTPS request to fleetops.website
  ▼
Route53  (A record → CloudFront distribution)
  │  Anycast DNS resolution
  ▼
CloudFront  (WAFv2 web ACL enforced here)
  │  Cache hit → serve from edge
  │  Cache miss / API → forward to origin
  ▼
ALB  (origin.fleetops.website A record → ALB DNS)
  │  HTTPS 443 listener; path-based routing rules
  ▼
EKS Node (target group → pod IP:port directly — target-type: ip)
  │  ALB → pod bypasses kube-proxy
  ▼
Spring Boot Container (port 8080 / 80)
  │  Spring Security JWT filter validates Bearer token
  ▼
RDS PostgreSQL / ElastiCache Redis / AWS SDK calls
```

---

## Route53

**WHAT:** AWS's managed authoritative DNS service. Hosts the `fleetops.website` hosted zone.

**WHY:** Native integration with CloudFront, ALB, ACM (DNS validation). Alias records resolve to AWS resource IPs without TTL drift. Alternative (external registrar) would require manual IP management.

**WHY NOT Cloudflare DNS:** Cloudflare DNS is excellent but adds external dependency; Route53 stays in-account, enabling IAM-controlled zone management and ACM DNS validation without leaving AWS.

**HOW IT WORKS:**
- Hosted zone created by `modules/route53`
- NS records delegated from domain registrar to Route53 name servers
- Three A records:
  - `fleetops.website` → CloudFront distribution (Alias, zero TTL)
  - `origin.fleetops.website` → ALB DNS name (Alias, resolves to ALB IPs)
  - `argocd.fleetops.website` → same ALB (Alias)
- ACM certificate validated via CNAME record auto-created in this zone

**FAILURE MODE:** If the hosted zone is deleted, DNS resolution fails globally for all endpoints. CloudFront and ALB continue to work but are unreachable by hostname.

**COST:** ~$0.50/month per hosted zone + $0.40 per million queries.

---

## CloudFront

**WHAT:** AWS's global CDN with 400+ edge locations. Sits in front of the ALB for the main domain.

**WHY:** Static assets (React JS/CSS bundles) are cached at edge, drastically reducing ALB and EKS load. Also the mandatory attachment point for WAFv2 in global scope.

**HOW IT WORKS:**
- Distribution created by `modules/cloudfront`
- Origin: `origin.fleetops.website` (the ALB)
- Cache behaviors:
  - `/api/*` → no cache (forward all headers, no caching)
  - `/*` → cache static assets (JS/CSS/images by query string + headers)
- WAFv2 web ACL attached at the CloudFront level (CLOUDFRONT scope)
- ACM wildcard cert (`*.fleetops.website`) attached — must be in `us-east-1` for CloudFront even if resources are in another region
- HTTPS-only; HTTP → HTTPS redirect

**FAILURE MODE:** If CloudFront goes down (extremely rare — it's a global service), `fleetops.website` becomes unreachable. The ALB remains reachable directly via `origin.fleetops.website` for bypass access.

**SECURITY:** WAFv2 runs at the edge — blocks malicious traffic before it hits the ALB or EKS. Rate limiting rules prevent brute-force login attacks.

---

## WAF (WAFv2)

**WHAT:** Web Application Firewall. Inspects HTTP(S) requests and allows/blocks based on rules.

**WHY:** Protects against OWASP Top 10: SQL injection, XSS, rate-based DoS, bad bots. Required for PCI/compliance posture.

**HOW IT WORKS:**
- Created by `modules/waf` as CLOUDFRONT scope (must be us-east-1)
- Attached to CloudFront distribution
- Rules include:
  - AWS Managed Rule Groups (AWSManagedRulesCommonRuleSet)
  - Rate-based rule: block IPs exceeding threshold requests/5min
  - Bot control (optional)
- Request is evaluated → scored → allowed or blocked with 403

**FAILURE MODE:** If WAF rules are misconfigured, legitimate traffic can be blocked (false positives). WAF has a Count mode for testing rules before enforcing.

---

## ALB (Application Load Balancer)

**WHAT:** Layer 7 load balancer. Terminates TLS, routes HTTP requests to EKS pods by URL path.

**WHY:** Native integration with EKS via the AWS Load Balancer Controller. Supports path-based routing (required for multiple microservices on one domain), health checks, and ACM TLS certificates.

**WHY NOT NLB:** NLB is Layer 4 (TCP). It can't do path-based routing or HTTP header inspection. We need `/api/auth/` → auth-service, `/api/vehicles/` → vehicle-service.

**HOW IT WORKS:**
- The ALB is NOT provisioned by Terraform directly
- The AWS Load Balancer Controller (running in EKS) watches Ingress resources with `kubernetes.io/ingress.class: alb`
- When ArgoCD syncs `ingress.yaml`, the LBC controller reads its annotations and calls AWS APIs to create the ALB
- Key annotations on the Ingress:
  - `alb.ingress.kubernetes.io/scheme: internet-facing` — public ALB in public subnets
  - `alb.ingress.kubernetes.io/target-type: ip` — routes directly to pod IP (bypasses kube-proxy/NodePort)
  - `alb.ingress.kubernetes.io/group.name: fleetops` — ArgoCD ingress and app ingress share ONE ALB
  - `alb.ingress.kubernetes.io/certificate-arn` — ACM cert for HTTPS
  - `alb.ingress.kubernetes.io/security-groups` — pre-created ALB security group
- Routing rules:
  ```
  /api/auth/*      → auth-service:8080
  /api/audit/*     → auth-service:8080
  /api/vehicles/*  → vehicle-service:8080
  /api/tracking/*  → vehicle-service:8080
  /api/tasks/*     → maintenance-service:8080
  /api/media/*     → maintenance-service:8080
  /api/requests/*  → request-service:8080
  /*               → frontend:80
  ```

**INTERNALS:** ALB target groups use the pod IP directly (target-type: ip). When a pod is deleted, the LBC deregisters it. When a new pod passes readiness probe, the LBC registers it. Health checks use `/actuator/health`.

**FAILURE MODE:** If the LBC controller pod crashes, the ALB continues routing existing traffic. New Ingress changes won't be applied until LBC recovers. The ALB itself has AWS-managed HA across AZs.

---

## VPC

**WHAT:** Virtual Private Cloud. Isolated network in AWS with CIDR `10.2.0.0/16`.

**WHY:** All FleetOps resources live in this VPC. Network-level isolation from other AWS accounts and from the internet (for private resources).

**Subnet Design:**

| Subnet Type | CIDR | AZs | Used For |
|-------------|------|-----|----------|
| Public | 10.2.1.0/24, 10.2.2.0/24 | us-east-1a, 1b | ALB, NAT Gateway |
| Private | 10.2.10.0/24, 10.2.11.0/24 | us-east-1a, 1b | EKS nodes |
| DB (private) | Separate CIDRs | us-east-1a, 1b | RDS, Redis, EFS |

**WHY this subnet split:**
- Public subnets have a route to the Internet Gateway (IGW) — required for ALB to receive traffic from the internet
- Private subnets have a route to NAT Gateway only — EKS nodes can pull container images and call AWS APIs but are not directly reachable from the internet
- DB subnets have NO internet route — databases are completely isolated. Only resources inside the VPC with the right security group can connect

**NAT Gateway:** Sits in a public subnet. EKS nodes in private subnets route outbound internet traffic through NAT Gateway. This allows nodes to pull ECR images, call AWS APIs (STS, Secrets Manager, Bedrock), etc. A single NAT Gateway is used (cost optimization) — production HA would use one per AZ.

---

## Security Groups

Security groups are stateful firewalls at the resource level. FleetOps uses separate SGs:

| SG | Inbound | Outbound |
|----|---------|----------|
| ALB SG | 443 from 0.0.0.0/0 (internet) | 8080 to EKS cluster SG, 80 to EKS cluster SG |
| EKS cluster SG | 8080 from ALB SG, 80 from ALB SG | All outbound |
| RDS SG | 5432 from EKS cluster SG | None |
| Redis SG | 6379 from EKS cluster SG | None |
| EFS SG | 2049 (NFS) from EKS nodes | None |

The critical SG rules added in `environments/prod/main.tf`:
```hcl
# ALB → pods on port 8080 (Spring Boot)
resource "aws_vpc_security_group_ingress_rule" "alb_to_pods_8080"

# ALB → pods on port 80 (Nginx frontend)
resource "aws_vpc_security_group_ingress_rule" "alb_to_pods_80"

# EKS → RDS
resource "aws_vpc_security_group_ingress_rule" "rds_from_eks_cluster_sg"

# EKS → Redis
resource "aws_vpc_security_group_ingress_rule" "redis_from_eks_cluster_sg"
```

These rules are expressed as Security Group references (not CIDRs) — any pod in the EKS cluster SG can reach RDS, but nothing outside the VPC can.

---

## EKS (Elastic Kubernetes Service)

**WHAT:** AWS-managed Kubernetes control plane. AWS runs etcd, kube-apiserver, scheduler, controller-manager. You manage worker nodes.

**WHY EKS over self-managed K8s:** AWS manages control plane availability (99.95% SLA), patches, and upgrades. Self-managed K8s on EC2 requires managing etcd backups, HA, and upgrades manually.

**WHY NOT ECS/Fargate:** EKS gives full Kubernetes API compatibility — Helm, ArgoCD, HPA, custom resource definitions (CRDs like ExternalSecret, Application) all work natively. ECS is simpler but less portable and lacks GitOps ecosystem.

**Cluster version:** 1.31 (set in `eks_cluster_version` variable)

**Node Group:** Managed node group on `m7i-flex.large` (4 vCPU, 16 GB RAM). Min 2, max 5 nodes. Nodes run in private subnets.

**Why m7i-flex.large:** "flex" instance type is ~10% cheaper than standard m7i with no sacrifice — AWS can use either Intel or AMD chips. 16 GB RAM fits multiple Spring Boot pods (each using 256–512 Mi).

### Kubernetes Control Plane Components (AWS-managed)

**kube-apiserver:** The REST API front door. Every kubectl command, every controller, every kubelet talks to the API server. In EKS it runs in AWS-managed infrastructure — you access it via the cluster endpoint URL. Public access is restricted to CIDRs specified in `eks_public_access_cidrs`.

**etcd:** Distributed key-value store for all cluster state (pod specs, secrets, deployments). In EKS, AWS runs a managed etcd with automatic backups. If etcd is lost, the cluster state is gone (that's why AWS manages it).

**kube-scheduler:** Watches for unscheduled pods and assigns them to nodes based on: resource requests/limits, node selectors, taints/tolerations, affinity rules. It does NOT start the pod — it just writes the node assignment to etcd.

**kube-controller-manager:** Runs control loops. ReplicaSet controller ensures desired replicas match actual. Deployment controller manages rolling updates. Endpoint controller keeps Service → Pod IP mappings current.

### Worker Node Components

**kubelet:** Agent running on every node. Watches etcd for pods assigned to its node. Calls the container runtime (containerd) to pull images and start containers. Reports pod status back to API server. Runs liveness/readiness probes.

**kube-proxy:** Runs on every node. Programs iptables/IPVS rules to route Service ClusterIP traffic to the correct pod IPs. In FleetOps with ALB target-type: ip, kube-proxy is bypassed for inbound traffic — ALB routes directly to pod IP.

**CoreDNS:** Cluster DNS. Resolves `fleetops-auth-service.fleetops-prod.svc.cluster.local` → ClusterIP. Runs as a Deployment in kube-system with 2 replicas. Every pod gets CoreDNS as its DNS server via `/etc/resolv.conf`.

### EKS Addons installed via Terraform

| Addon | Version | Purpose |
|-------|---------|---------|
| aws-load-balancer-controller | 1.8.1 (Helm) | Provisions ALB from Ingress resources |
| external-secrets | 0.9.19 (Helm) | Syncs AWS Secrets Manager/SSM → K8s Secrets |
| cluster-autoscaler | 9.37.0 (Helm) | Scales node group up/down based on pending pods |
| metrics-server | 3.12.1 (Helm) | Provides CPU/memory metrics for HPA |
| argocd | 6.7.11 (Helm) | GitOps continuous deployment |
| amazon-cloudwatch-observability | v6.2.0 | CloudWatch Container Insights + Fluent Bit log shipping |
| aws-efs-csi-driver | v2.0.7 | Mounts EFS volumes into pods |

**All addon images are mirrored to ECR** (pull-through cache) so nodes never pull from public registries. This prevents rate limiting and improves security.

---

## IRSA (IAM Roles for Service Accounts)

IRSA is the mechanism that allows individual Kubernetes pods to assume specific AWS IAM roles — without using static access keys or shared node instance profiles.

**Full mechanics (step by step):**

1. **OIDC Provider created** by `modules/eks/oidc`. The EKS control plane acts as an OIDC identity provider. Its URL looks like `oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE`

2. **Terraform registers this OIDC URL** with AWS IAM as a trusted identity provider

3. **IAM role created** with a trust policy that says: "Allow `sts:AssumeRoleWithWebIdentity` from this OIDC provider, but ONLY if the sub claim equals `system:serviceaccount:fleetops-prod:fleetops-app`"

   ```json
   "Condition": {
     "StringEquals": {
       "oidc.eks.us-east-1.amazonaws.com/id/XXX:sub": "system:serviceaccount:fleetops-prod:fleetops-app",
       "oidc.eks.us-east-1.amazonaws.com/id/XXX:aud": "sts.amazonaws.com"
     }
   }
   ```

4. **Kubernetes ServiceAccount annotated:**
   ```yaml
   annotations:
     eks.amazonaws.com/role-arn: arn:aws:iam::538661800892:role/fleetops-prod-app-irsa-role
   ```

5. **Pod spec references the ServiceAccount.** When the pod starts, the EKS token controller mounts a projected ServiceAccount token at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` — this is a short-lived OIDC JWT (valid 86400s by default, rotated every 24h).

6. **AWS SDK in the Spring Boot pod** reads the `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN` environment variables (injected automatically by EKS), then calls `sts:AssumeRoleWithWebIdentity`, presenting the projected token.

7. **STS validates** the token against the OIDC provider, checks the sub/aud conditions in the trust policy, and issues temporary credentials (AccessKeyId, SecretAccessKey, SessionToken) valid for 1 hour.

8. **SDK caches** the temporary credentials and refreshes them before expiry. The pod calls Bedrock, Secrets Manager, S3, Step Functions using these credentials.

**Why IRSA is superior:**
- vs. access keys: No static credentials to rotate or leak. Credentials are temporary and auto-renewed.
- vs. node instance profile: A node profile gives ALL pods on that node the same permissions. IRSA is pod-level — auth-service cannot call Bedrock, only vehicle-service can.
- vs. Kubernetes Secrets: Storing AWS keys in K8s Secrets (even encrypted) is an attack surface. IRSA has no secret to steal.

**IRSA roles in this project:**
- `fleetops-prod-app-irsa-role` — for the `fleetops-app` ServiceAccount in `fleetops-prod` namespace (all app pods). Permissions: Secrets Manager read, S3 CRUD, Step Functions, EFS mount, Bedrock invoke, SNS publish, CloudWatch logs.
- `fleetops-prod-alb-controller-role` — for `aws-load-balancer-controller` ServiceAccount in `kube-system`
- `fleetops-prod-external-secrets-role` — for `external-secrets` ServiceAccount in `external-secrets` namespace
- `fleetops-prod-cluster-autoscaler-role` — for `cluster-autoscaler` ServiceAccount in `kube-system`
- `fleetops-prod-cloudwatch-agent-role` — for `cloudwatch-agent` ServiceAccount in `amazon-cloudwatch`
- `fleetops-prod-efs-csi-driver-role` — for `efs-csi-controller-sa` ServiceAccount in `kube-system`

---

## RDS (PostgreSQL 15.7)

**WHAT:** Managed relational database. Single instance, db.t3.micro, 20 GB gp2 storage.

**WHY PostgreSQL:** ACID-compliant, excellent Spring Data JPA support, Flyway migrations, supports JSONB for flexible data.

**WHY NOT Aurora:** Aurora Serverless v2 would be ideal for variable load, but db.t3.micro costs ~$15/month vs Aurora minimum ~$45/month. Project is cost-optimized for training/evaluation.

**Database schema:**
Each service has its own database (not separate instances — same RDS instance, multiple DBs):
- `auth_db` — users, roles, audit logs
- `vehicle_db` — vehicles, tracking events, GPS data
- `maintenance_db` — maintenance tasks, media records
- `request_db` — service requests, request states

**Flyway migrations:** Each Spring Boot service has `src/main/resources/db/migration/V*.sql` scripts. On startup, Flyway connects to the DB and applies any pending migrations in order. This ensures schema is always in sync with code.

**Security:**
- In DB subnet (no internet route)
- KMS encryption at rest (`kms_rds_key_arn`)
- Credentials stored in Secrets Manager (`fleetops/prod/db`)
- Security group: only EKS cluster SG can connect on port 5432

**Connection pool:** HikariCP (Spring Boot default). Configured via `spring.datasource.hikari.*` in `application.yml`. Important: each pod has its own pool. With 2 replicas per service and 4 services, that's 8 HikariCP pools all connecting to the same RDS instance. db.t3.micro max connections = ~85. Each pool default is 10. 8 × 10 = 80 connections — fits, but tight. Pool size tuned down to ~5-7 to leave headroom.

**FAILURE MODE:** RDS goes down → all services that need DB calls fail with connection errors. Spring Boot returns 503. Redis-cached responses (vehicle-service) continue to serve until TTL. Auth validation (which reads the JWT secret from memory) continues to work. Single-AZ means no automatic failover — production would use Multi-AZ.

---

## Redis (ElastiCache 7)

**WHAT:** In-memory key-value store used as a cache layer by vehicle-service.

**WHY Redis for caching:** Sub-millisecond reads. Vehicle fleet data (list of all vehicles, KPI aggregates) is expensive to compute from PostgreSQL on every request. Redis caches the result for 5 minutes (TTL = 300,000ms).

**How vehicle-service uses Redis:**
```
GET /api/vehicles/fleet
  → Check Redis key "fleet:all"
  → Cache HIT: return Redis value (fast, no DB query)
  → Cache MISS: query RDS, write result to Redis with TTL, return result
```

**Configuration:** Redis endpoint injected via ExternalSecret from SSM Parameter Store `/fleetops/prod/redis/endpoint`. Spring Boot auto-configures `spring.data.redis.host` from `REDIS_HOST` env var.

**Security:**
- In DB subnet
- Security group: only EKS cluster SG can connect on port 6379
- Encryption in transit (TLS) and at rest

**FAILURE MODE:** Redis goes down → vehicle-service falls back to direct DB queries. Performance degrades but functionality is maintained. Spring `@Cacheable` methods will simply always miss.

**cache.t3.micro:** Single-node ElastiCache (no Multi-AZ). Adequate for caching use case in this project.

---

## EFS (Elastic File System)

**WHAT:** Managed NFS (Network File System). Provides a shared filesystem accessible by multiple pods simultaneously.

**WHY EFS for maintenance-service:** Maintenance tasks include media file uploads (photos of vehicles, maintenance evidence). With multiple replicas of maintenance-service, a local volume would not work — a file uploaded to Pod A would not be visible to Pod B. EFS is a shared filesystem accessible by all pods concurrently (ReadWriteMany).

**WHY NOT S3 directly:** S3 is object storage — there's no filesystem path. The application uses standard file I/O at path `/var/www/fleetops/shared-media`. EFS presents a POSIX-compliant filesystem.

**HOW IT WORKS:**
- EFS file system created by `modules/efs`
- EFS CSI driver addon (`aws-efs-csi-driver`) installed in EKS via Terraform
- Helm chart creates a PersistentVolume and PersistentVolumeClaim referencing the EFS filesystem ID
- Pod mounts the PVC at `/var/www/fleetops/shared-media`
- All maintenance-service replicas share the same EFS mount point simultaneously

**Security:**
- EFS security group allows NFS (port 2049) only from EKS nodes
- KMS encryption at rest
- IRSA policy on `fleetops-app` role grants `elasticfilesystem:ClientMount`, `ClientWrite`

---

## S3

**WHAT:** Object storage. Used for vehicle document uploads (contracts, registrations, inspection reports).

**WHY S3 for vehicle documents:** Binary files don't belong in PostgreSQL. S3 is infinitely scalable, durable (11 9s), cheap, and supports presigned URLs for direct browser-to-S3 uploads.

**HOW presigned URL upload works:**
1. Frontend calls `POST /api/vehicles/{id}/documents/upload-url`
2. vehicle-service calls `S3.generatePresignedUrl()` with the target key and PUT method, expiry 15 minutes
3. S3 returns a presigned URL containing the signed AWS credentials embedded in query params
4. vehicle-service returns the presigned URL to the frontend
5. Browser uploads the file directly to S3 using PUT to the presigned URL — no data passes through the EKS pods
6. After upload, frontend calls `POST /api/vehicles/{id}/documents` to register the document metadata in PostgreSQL

**Security:** Bucket is private (no public access). Only presigned URL holders can access objects. IRSA grants vehicle-service `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:ListBucket` on `fleetops-prod-vehicle-docs`. KMS encryption.

---

## SNS (Simple Notification Service)

**WHAT:** Managed pub/sub messaging. Maintenance-service publishes alerts to SNS topics; subscribers (SQS, email, Lambda) receive them.

**Two topics:**
- `fleetops-prod-insurance-alerts` — triggered when a vehicle's insurance is nearing expiry
- `fleetops-prod-service-alerts` — triggered when a maintenance task is overdue or critical

**HOW maintenance-service publishes:**
```java
snsClient.publish(PublishRequest.builder()
    .topicArn(insuranceSnsTopicArn)
    .message(alertMessage)
    .subject("Insurance Alert")
    .build());
```
The `insuranceSnsTopicArn` is injected from Kubernetes Secret `fleetops-sns-secret` (synced from SSM Parameter Store by External Secrets Operator).

**IRSA:** The `fleetops-app` IRSA role has `sns:Publish` permission on `arn:aws:sns:us-east-1:538661800892:fleetops-*`.

**Subscribers:** Email subscription (configured via `alert_emails` Terraform variable). In production, SQS would be added for durable message processing.

**FAILURE MODE:** If SNS publish fails, Spring Boot logs an error but the maintenance task CRUD operation succeeds — alert delivery is best-effort, not transactional.

---

## Lambda

**WHAT:** Serverless function. The `alert-processor` function is a Node.js Lambda that runs daily.

**PURPOSE:** Daily scan for vehicles with expiring insurance, overdue maintenance, or critical alerts. Lambda calls the vehicle-service and maintenance-service REST APIs, then publishes alerts to SNS topics.

**WHY Lambda for daily alerts:** The alert scan is not user-initiated — it's scheduled. Running it in a pod 24/7 wastes resources. Lambda runs for seconds per day and costs fractions of a cent.

**Execution lifecycle:**
1. EventBridge rule fires at `rate(1 day)`
2. Lambda service invokes the `alert-processor` function
3. **Cold start** (if no warm execution environment): Lambda creates a new execution environment, initializes Node.js runtime, loads the function code. Takes 100–500ms extra.
4. Function handler runs: calls auth-service to get a service token (using `LAMBDA_SERVICE_PASSWORD` from Secrets Manager), calls vehicle/maintenance service endpoints, publishes to SNS
5. Function returns → execution environment kept warm for ~15min

**IAM:** Lambda role (`fleetops-prod-lambda-role`) has:
- `AWSLambdaBasicExecutionRole` (CloudWatch Logs)
- `AWSLambdaVPCAccessExecutionRole` (ENI creation in VPC — if VPC-deployed)
- Custom policy: `sns:Publish` + `secretsmanager:GetSecretValue` for `fleetops/*` secrets

**LAMBDA_SERVICE_PASSWORD:** The Lambda authenticates to auth-service with a dedicated service account (username: `lambda-service`). The password is stored in Secrets Manager at `fleetops/prod/lambda-service-credentials`. Auth-service seeds this account on startup via `LambdaServiceInitializer`, reading the password from the `LAMBDA_SERVICE_PASSWORD` env var (synced by External Secrets Operator from the `fleetops-lambda-service-secret` K8s Secret).

---

## EventBridge

**WHAT:** Serverless event bus and scheduler. Used to trigger the Lambda on a schedule.

**Rule:** `rate(1 day)` — fires once every 24 hours.
**Target:** Lambda `alert-processor` ARN.
**Permission:** EventBridge has `lambda:InvokeFunction` permission on the alert-processor function.

**WHY EventBridge over cron in a pod:** EventBridge is managed, zero-infrastructure, and guaranteed to fire even if EKS is down for maintenance. A Kubernetes CronJob would miss its schedule if the node running the scheduler is unavailable.

---

## Step Functions

**WHAT:** Serverless workflow orchestration. A state machine that manages the lifecycle of a service request from PENDING through APPROVED/REJECTED to COMPLETED.

**WHY Step Functions for service request lifecycle:** Service requests go through multiple states with branching logic, approval steps, and conditional transitions. Encoding this in a database + application code is fragile. Step Functions provides:
- Visual workflow definition
- Guaranteed state transitions (exactly-once semantics)
- Built-in retry/error handling
- Execution history and logging
- Long-running workflows (months-long executions possible)

**State machine states:**
```
PENDING → [MANAGER_REVIEW] → APPROVED → [TECHNICIAN_ASSIGNED] → IN_PROGRESS → COMPLETED
                           → REJECTED
```

**HOW request-service uses Step Functions:**
1. User creates a service request via `POST /api/requests`
2. request-service calls `StepFunctions.startExecution()` with the request payload
3. Step Functions creates an execution and begins the state machine
4. Each state transition in Step Functions triggers a task — these call back to request-service endpoints or the application logic
5. The execution ARN is stored in the request record for status tracking
6. `GET /api/requests/{id}/status` calls `StepFunctions.describeExecution()` to get current state

**IRSA:** `fleetops-app` role has `states:StartExecution`, `states:DescribeExecution`, `states:StopExecution` on `arn:aws:states:us-east-1:538661800892:stateMachine:fleetops-*`.

---

## Bedrock

**WHAT:** AWS's managed AI/ML model inference service. FleetOps uses it for an AI Fleet Maintenance Advisor.

**Model:** `amazon.nova-lite-v1:0` — Amazon's lightweight language model, cost-optimized for text generation tasks.

**WHY Bedrock over OpenAI API:** All data stays within the AWS account. No data sent to third parties. IAM-controlled access. Pay-per-token pricing with no subscription.

**HOW vehicle-service calls Bedrock:**
1. User calls `POST /api/vehicles/{id}/ai-analysis`
2. vehicle-service fetches vehicle data from PostgreSQL
3. Builds a prompt: "Given this vehicle's maintenance history and current status: {data}, provide maintenance recommendations..."
4. Calls Bedrock `InvokeModel` (or `Converse` API) with model ID `amazon.nova-lite-v1:0`
5. Bedrock returns generated text
6. vehicle-service returns the AI analysis to the frontend

**Converse API structure:**
```json
{
  "modelId": "amazon.nova-lite-v1:0",
  "messages": [
    { "role": "user", "content": [{ "text": "..." }] }
  ],
  "inferenceConfig": {
    "maxTokens": 1024,
    "temperature": 0.7
  }
}
```

**Authentication:** Bedrock credentials stored as a Kubernetes Secret `fleetops-bedrock-secret` (created directly by Terraform, not via External Secrets Operator, because Bedrock keys are IAM user keys — not in Secrets Manager). Alternatively, IRSA on `fleetops-app` role has `bedrock:InvokeModel` permission — the cleaner approach.

**Redis caching of AI responses:** AI calls are expensive (~50–200ms, charged per token). vehicle-service caches Bedrock responses in Redis with a longer TTL (analysis for the same vehicle doesn't change in 5 minutes).

---

## Secrets Manager vs SSM Parameter Store

**Secrets Manager:** Used for sensitive credentials:
- `fleetops/prod/db` — `{username, password, host}`
- `fleetops/prod/jwt` — `{jwt_secret}`
- `fleetops/prod/lambda-service-credentials` — `{password}`
- `fleetops/prod/github-pat` — `{username, token}` (used by ArgoCD to clone the deployments repo)

**SSM Parameter Store:** Used for non-secret configuration (but marked SecureString for good measure):
- `/fleetops/prod/redis/endpoint` — ElastiCache hostname
- `/fleetops/prod/sns/insurance-alerts-arn` — SNS topic ARN
- `/fleetops/prod/sns/service-alerts-arn` — SNS topic ARN
- `/fleetops/prod/cors-allowed-origins`

**WHY two services:** Secrets Manager costs $0.40/secret/month + $0.05/10K API calls. SSM SecureString (standard tier) is free. Use Secrets Manager for high-value credentials, SSM for config values. Both are encrypted with KMS.

**HOW External Secrets Operator syncs them:**
- Two `ClusterSecretStore` objects in `k8s/platform/external-secrets.yaml`:
  - `aws-secrets-manager` → Secrets Manager
  - `aws-ssm` → SSM Parameter Store
- Each `ExternalSecret` object references a store, maps remote keys to Kubernetes Secret keys
- ESO polls every `refreshInterval: 1h` and updates the K8s Secret if the value changed
- ESO uses its own IRSA role to authenticate — not the app's role

---

## CloudWatch

**WHAT:** AWS observability platform — metrics, logs, alarms, dashboards.

**In FleetOps:**
- EKS control plane logs → CloudWatch Log Groups
- Container logs (stdout/stderr from pods) → CloudWatch via Fluent Bit (bundled in CloudWatch Observability addon)
- CloudWatch Alarms:
  - RDS CPU > threshold → SNS notification
  - Lambda error rate > threshold → SNS notification
  - ALB 5xx error rate → SNS notification
- Container Insights: Node and pod CPU/memory metrics

**Log path:** Pod stdout → Fluent Bit DaemonSet → CloudWatch Log Group `/aws/containerinsights/{cluster}/application`

---

## CloudTrail

**WHAT:** Records every AWS API call made in the account. Who called what API, when, from where.

**WHY:** Compliance, forensics, security auditing. If credentials are compromised, CloudTrail shows exactly what was done.

**Configuration:** `modules/cloudtrail` creates a trail that logs to an S3 bucket, encrypted with KMS. All management events captured. Data events (S3 object-level) are optional.

---

## AWS Config

**WHAT:** Continuously evaluates AWS resource configurations against compliance rules.

**WHY:** Drift detection. If someone manually changes a security group rule outside of Terraform, AWS Config flags it. Provides a compliance dashboard.

---

## ECR (Elastic Container Registry)

**WHAT:** Private Docker registry in AWS. All container images are pushed here.

**Repositories (created by `bootstrap/main.tf`):**
- `fleetops-dev/auth-service`
- `fleetops-dev/vehicle-service`
- `fleetops-dev/maintenance-service`
- `fleetops-dev/request-service`
- `fleetops-dev/frontend`
- (same for prod)

**Image naming convention:**
- `develop` branch: `develop-{SHA:7}` e.g., `develop-a3f92c1`
- `main` branch: semver `v1.2.3`
- feature branches: `feature-name-{SHA:7}`

**Pull-through cache:** EKS nodes pull addon images (ArgoCD, metrics-server, etc.) from ECR pull-through cache rather than directly from public registries. Configured in Terraform: nodes have `ecr:CreateRepository` and `ecr:BatchImportUpstreamImage` permissions.

---

## ArgoCD

**WHAT:** GitOps continuous deployment tool. Watches a Git repository and ensures the Kubernetes cluster matches the desired state defined in that repository.

**WHY ArgoCD:** Git becomes the single source of truth for cluster state. Every deployment is a Git commit (auditable). Self-healing — if someone manually changes a K8s resource, ArgoCD reverts it. Rollback = `git revert`.

**WHY NOT Flux:** ArgoCD has a richer UI, better multi-cluster support, and the App-of-Apps pattern maps well to this project's structure. Both are valid; this is a judgment call.

**Installed via:** `helm_release.argocd` in `modules/eks/addons/main.tf`. ArgoCD runs in the `argocd` namespace. Exposed via ALB Ingress at `argocd.fleetops.website`.

**App-of-Apps Pattern:**
```
fleetops-root-prod (root Application)
  → reads: argocd/apps/prod/*.yaml
  → manages:
      fleetops-platform-prod   (wave 0 — namespace, RBAC, SA)
      fleetops-secrets-prod    (wave 0 — ExternalSecrets)
      fleetops-networkpolicy   (wave 1)
      fleetops-ingress-prod    (wave 2 — creates ALB)
      fleetops-auth-prod       (wave 3)
      fleetops-vehicle-prod    (wave 4)
      fleetops-maintenance-prod (wave 4)
      fleetops-request-prod    (wave 4)
      fleetops-frontend-prod   (wave 5)
      fleetops-db-init-prod    (wave 1 — one-time schema migration Job)
```

**Sync waves** control deployment order. Wave N must reach Healthy before wave N+1 starts. This ensures secrets exist before pods try to read them, and the ALB exists before services register as targets.

**selfHeal: true** — If someone `kubectl edit`s a deployment, ArgoCD detects the drift within 3 minutes and reverts to the Git state.

**prune: true** — If a resource is removed from Git, ArgoCD deletes it from the cluster.

**Repository access:** ArgoCD is given a GitHub App token stored as a Kubernetes Secret `argocd-repo-fleetops-deployments` (created by Terraform, reading the PAT from Secrets Manager).

---

# PHASE 3 — TERRAFORM MASTERCLASS

## Module Architecture

```
fleetops-terraform/
├── bootstrap/          One-time: S3 state bucket, DynamoDB lock, ECR repos
├── environments/
│   └── prod/
│       ├── main.tf     Composes all modules for the prod environment
│       ├── variables.tf
│       └── outputs.tf
└── modules/
    ├── networking/     VPC, subnets, IGW, NAT, route tables, SGs
    ├── eks/
    │   ├── cluster/    EKS cluster resource
    │   ├── oidc/       OIDC provider registration
    │   ├── nodegroup/  Managed node group
    │   └── addons/     Helm-installed addons + IRSA roles
    ├── iam/            IRSA roles: app, lambda, devops-agent
    ├── kms/            KMS keys (RDS, Secrets, S3, Events/SNS/CloudTrail)
    ├── rds/            RDS PostgreSQL instance
    ├── redis/          ElastiCache Redis cluster
    ├── s3/             Vehicle documents bucket
    ├── efs/            Elastic File System
    ├── secrets-manager/ Secrets for DB, JWT, Lambda, GitHub PAT
    ├── ssm/            SSM parameters for Redis endpoint, SNS ARNs, CORS
    ├── route53/        Hosted zone
    ├── acm/            ACM wildcard certificate
    ├── cloudfront/     CDN distribution
    ├── waf/            WAFv2 web ACL
    ├── alb/            (used for direct ALB provisioning if needed)
    ├── sns/            Insurance + service alert topics
    ├── lambda/         alert-processor function
    ├── eventbridge/    Daily trigger rule
    ├── step-functions/ Service request state machine
    ├── dynamodb/       Document metadata (optional)
    ├── cloudwatch/     Alarms, log groups
    ├── cloudtrail/     Audit trail
    ├── config/         AWS Config rules
    └── bedrock/        Bedrock model access config
```

## Remote State

**WHY remote state:** Terraform state stores the mapping between Terraform resources and real AWS resources. It must be shared across team members and CI/CD runs. Storing it locally means only one developer can apply.

**S3 backend configuration** (in `environments/prod/main.tf`):
```hcl
backend "s3" {
  bucket         = "fleetops-terraform-state-johan"
  key            = "environments/prod/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "fleetops-terraform-locks"
  encrypt        = true
}
```

- **S3 bucket:** Stores the `terraform.tfstate` JSON file. Versioning enabled (in bootstrap) — you can recover previous state if corrupted.
- **DynamoDB table:** `fleetops-terraform-locks`. When `terraform apply` starts, it writes a lock item to this table. If another `apply` tries to run concurrently, it reads the lock and fails. This prevents two simultaneous applies from corrupting state.
- **encrypt: true:** S3 server-side encryption (KMS) for the state file, which contains sensitive data (DB passwords in plaintext in state — known issue, addressed in security backlog).

**Bootstrap sequence:**
```bash
# First: create the state bucket itself (bootstrapped with local state)
cd bootstrap/
terraform init   # uses local state
terraform apply  # creates S3, DynamoDB, ECR repos

# Then: use that bucket for all other environments
cd environments/prod/
terraform init   # downloads state from S3 bucket
terraform apply
```

## Terraform State Mechanics

**Dependency graph:** Terraform parses all `.tf` files in a directory, builds a directed acyclic graph (DAG) of resources. Edges represent dependencies (explicit via `depends_on`, implicit via reference: `module.rds.db_endpoint` used in another module creates an edge from rds to that module).

**Plan phase:**
1. `terraform init` downloads providers and modules
2. `terraform plan` reads the state file, calls AWS APIs to get current resource state (refresh), compares desired config to current state, generates an execution plan: CREATE/UPDATE/DESTROY for each resource

**Apply phase:**
1. Terraform walks the dependency graph in topological order (leaf nodes first)
2. Resources with no dependencies run in parallel (Terraform default parallelism = 10)
3. When a resource completes, its outputs are available for dependent resources
4. State is updated incrementally — if apply fails mid-way, partial state is saved

**Drift detection:** `terraform plan` after manual changes shows the drift. `terraform refresh` updates the state to match reality without changing resources (deprecated in 0.15+, use `terraform apply -refresh-only`).

**State locking:** DynamoDB lock prevents concurrent applies. Lock is released after apply completes (or fails). If a process is killed, the lock may remain — `terraform force-unlock <lock-id>` clears it (dangerous, manual step).

## Provider Configuration

```hcl
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Project     = "fleetops"
      Environment = var.environment
      ManagedBy   = "terraform"
      Owner       = "FleetOps-Team"
    }
  }
}
```
`default_tags` applies to every AWS resource created by this provider — no need to specify tags on each resource block.

```hcl
provider "helm" {
  kubernetes {
    host                   = module.eks_cluster.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks_cluster.cluster_ca_data)
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args        = ["eks", "get-token", "--cluster-name", module.eks_cluster.cluster_name]
    }
  }
}
```
The `exec` block calls `aws eks get-token` dynamically to authenticate to EKS. This means the machine running Terraform must have AWS credentials. In CI/CD, this is the OIDC-authenticated GitHub Actions role.

## Module: KMS

Creates four KMS keys:
- `rds_key_arn` — encrypts RDS storage at rest
- `secrets_key_arn` — encrypts Secrets Manager secrets at rest; also encrypts EKS secrets (etcd encryption)
- `s3_key_arn` — encrypts S3 bucket objects + EFS
- `events_key_arn` — encrypts CloudTrail S3, SNS, Step Functions, CloudWatch logs

**WHY separate keys:** Blast radius reduction. If the RDS key is compromised, S3 data is unaffected. Separate keys also allow separate key policies per service.

**Key rotation:** `enable_key_rotation = true` on all keys. AWS automatically rotates the backing key material annually.

## Module: Networking

Creates the entire network layer. Dependencies: none (no other module needed before networking).

Outputs consumed by every other module:
- `vpc_id` → EKS, networking rules
- `public_subnet_ids` → EKS cluster (for ALB), NAT Gateway
- `private_subnet_ids` → EKS node group
- `database_subnet_ids` → RDS, Redis, EFS
- `rds_sg_id`, `redis_sg_id`, `alb_sg_id`, `eks_control_plane_sg_id`, `efs_sg_id` → used in resource-specific SG rules

## Module: EKS (cluster + oidc + nodegroup + addons)

Split into 4 sub-modules to control dependency order:

1. `eks/cluster` → creates the EKS cluster (control plane). Outputs: `cluster_endpoint`, `cluster_ca_data`, `cluster_name`, `oidc_issuer_url`, `cluster_sg_id`
2. `eks/oidc` → registers the OIDC issuer URL with IAM. Depends on: cluster. Outputs: `oidc_provider_url`
3. `eks/nodegroup` → creates managed node group. Depends on: cluster, IAM (node role). 
4. `eks/addons` → installs Helm charts (ArgoCD, ESO, ALB controller, etc.) and creates IRSA roles. Depends on: nodegroup (nodes must exist before Helm installs), secrets_manager (ArgoCD needs GitHub PAT)

**Why `depends_on = [module.eks_nodegroup, module.secrets_manager]` on addons:**
Helm charts deploy pods. Pods need worker nodes. If addons run before nodegroup is ready, pods stay Pending forever.

## GitHub Actions OIDC for Terraform

GitHub Actions assumes an AWS IAM role via OIDC — no static AWS credentials stored in GitHub Secrets:

```hcl
# In modules/iam, there's a GitHub OIDC provider + IAM role
# Trust policy allows GitHub Actions from the FleetOps-V2 org to assume the role
Condition: {
  StringLike: {
    "token.actions.githubusercontent.com:sub": "repo:FleetOps-V2/*:*"
  }
}
```

The terraform-apply workflow:
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-east-1
```
GitHub exchanges the OIDC token for temporary STS credentials. No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in GitHub Secrets.

## Terraform CI/CD Workflows

**terraform-plan.yml (PR validation):**
- Trigger: PR touching `environments/**` or `modules/**`
- Steps: AWS OIDC auth → `terraform init` → `terraform validate` → `terraform plan`
- Posts plan output as a PR comment (truncated to 60 KB)

**terraform-apply.yml (main branch):**
- Job 1 (Validate): `terraform fmt -check`, `terraform validate`, Checkov IaC security scan
- Job 2 (Plan): `terraform plan -out=tfplan` → uploaded as artifact
- Job 3 (Apply): Manual approval gate for prod → downloads `tfplan` → `terraform apply tfplan` → extracts outputs (ACM cert ARN, ALB SG, EFS ID, Step Functions ARN) → writes `environments/prod/infra-values.yaml` → pushes to deployments repo

**Why save plan as artifact and apply the saved plan:** This ensures the apply executes exactly the plan that was reviewed. If `terraform apply` is run without a plan file, it re-plans first — and infrastructure could have changed between review and apply.

## Checkov IaC Security Scan

Checkov scans Terraform code for security misconfigurations:
- S3 buckets without versioning or encryption → flagged
- Security groups open to 0.0.0.0/0 → flagged
- RDS without deletion protection → flagged (suppressed for dev with `#checkov:skip`)

Some Checkov checks are intentionally skipped with explanations:
```hcl
#checkov:skip=CKV_AWS_260:Source is scoped to ALB SG via referenced_security_group_id, not 0.0.0.0/0
```
This is the correct approach — suppress false positives with documented justification, not blanket disables.

---

## TERRAFORM Q&A — Evaluator Questions

**Q: What is the difference between `terraform plan` and `terraform apply`?**
A: `plan` is a dry run — it reads state, refreshes from AWS, and shows what would change. `apply` executes those changes. In this project, CI saves the plan as an artifact (`-out=tfplan`) and apply executes that exact plan, ensuring no drift between what was reviewed and what runs.

**Q: What happens if two engineers run `terraform apply` simultaneously?**
A: The second apply fails immediately. The DynamoDB `fleetops-terraform-locks` table uses a conditional write — when the first apply acquires the lock, the second sees the lock item exists and errors: "Error acquiring the state lock." The lock is released when the first apply finishes.

**Q: What is state drift and how does Terraform detect it?**
A: Drift is when real AWS resource state differs from the Terraform state file. This can happen from manual console changes or external automation. `terraform plan` calls AWS APIs to get current state (refresh step), compares to the state file, and surfaces changes as modifications in the plan output.

**Q: Why is the Terraform state encrypted?**
A: The state file contains ALL resource attributes — including secrets like DB passwords, JWT secrets. At rest encryption with KMS means an attacker who accesses the S3 bucket gets encrypted data they cannot read without the KMS key.

**Q: What are Terraform modules and why use them?**
A: Modules are reusable, parameterized groups of resources. In FleetOps, `modules/networking` encapsulates all VPC resources. This means:
1. The networking config can be reused across dev/staging/prod environments with different variables
2. Each module has defined inputs (variables) and outputs — other modules reference outputs, not internal resource IDs
3. Changes are isolated — modifying the RDS module doesn't risk accidentally changing networking

**Q: What is `depends_on` and when is it needed?**
A: Terraform automatically infers dependencies when one resource references another's attribute. `depends_on` is for hidden dependencies — cases where a resource must exist before another, but there's no attribute reference. Example: `module.eks_addons` depends on `module.eks_nodegroup` because Helm pods need worker nodes, but no attribute of the nodegroup is directly referenced in the addons module inputs.

**Q: What is `terraform destroy` and when would you use it?**
A: `terraform destroy` deletes all resources managed by the current configuration. Used for: tearing down temporary dev environments, cleaning up after evaluation. Dangerous for production — the `enable_deletion_protection = true` on RDS prevents accidental deletion even if `destroy` is run.

**Q: What is the `lifecycle` block used for in this project?**
A: The `kubernetes_namespace.fleetops_prod` resource has:
```hcl
lifecycle {
  ignore_changes = [metadata[0].labels, metadata[0].annotations]
}
```
ArgoCD adds its own labels/annotations to the namespace. Without `ignore_changes`, every `terraform plan` would show a diff trying to remove ArgoCD's labels, causing perpetual drift. `ignore_changes` tells Terraform to stop tracking those fields.

---

# PHASE 4 — EKS & KUBERNETES MASTERCLASS

## Every Kubernetes Object in FleetOps

### Deployment
Declares the desired state for a set of identical pods. From `charts/auth-service/templates/deployment.yaml`:
- `serviceAccountName: fleetops-app` — activates IRSA; the SA annotation triggers EKS to mount the OIDC token
- `securityContext: runAsNonRoot: true, runAsUser: 1000` — container cannot run as root even if the image allows it
- `envFrom: configMapRef` — loads all ConfigMap keys as env vars (non-secret config)
- Individual `secretKeyRef` entries — pulls specific keys from K8s Secrets (POSTGRES_HOST, JWT_SECRET, LAMBDA_SERVICE_PASSWORD)
- `SPRING_DATASOURCE_URL: "jdbc:postgresql://$(POSTGRES_HOST):5432/auth_db"` — Kubernetes variable substitution: `$(POSTGRES_HOST)` is replaced at pod start with the value of the `POSTGRES_HOST` env var from the Secret

The **Deployment controller** runs a reconciliation loop: if actual pod count < desired, it creates pods. If actual > desired (e.g., after HPA scale-down), it deletes pods. During rolling updates, it creates new pods before terminating old ones.

### Service (ClusterIP)
Stable virtual IP for routing to a set of pods matching a selector.

```yaml
kind: Service
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: fleetops-auth-service
  ports:
    - port: 8080
      targetPort: 8080
```

When maintenance-service calls `http://fleetops-auth-service:8080`, CoreDNS resolves `fleetops-auth-service.fleetops-prod.svc.cluster.local` → ClusterIP. kube-proxy iptables rules then DNAT the packet to one of the healthy pod IPs (round-robin).

**Why ClusterIP not LoadBalancer:** `LoadBalancer` type creates a new AWS ELB per Service — expensive and unnecessary when an ALB with path routing already exists. ClusterIP is internal-only; external traffic enters via the shared ALB Ingress.

### Ingress
Defines HTTP routing rules interpreted by the AWS Load Balancer Controller.

**Dual-host design** in `k8s/prod/apps/ingress.yaml`:
- `fleetops.website` — browser requests (arriving from CloudFront)
- `origin.fleetops.website` — used by CloudFront when forwarding to origin (CloudFront sends `Host: origin.fleetops.website` to the ALB) AND by the Lambda function (which calls backend APIs directly, bypassing CloudFront)

Both hosts need identical routing rules. Without `origin.fleetops.website`, CloudFront forwarded requests would hit the ALB with the wrong Host header and return 404.

**Key annotations:**
- `alb.ingress.kubernetes.io/group.name: fleetops` — ArgoCD ingress and app ingress share ONE ALB (group merge). Without this, each Ingress creates its own ALB at ~$18/month each.
- `alb.ingress.kubernetes.io/target-type: ip` — routes directly to pod IP, bypassing kube-proxy, removing a network hop
- `alb.ingress.kubernetes.io/ssl-redirect: "443"` — ALB listener rule redirects HTTP 80 → HTTPS 443 automatically

### ConfigMap
Non-secret configuration data mounted as env vars. Example from auth-service:
```yaml
data:
  SPRING_PROFILES_ACTIVE: "prod"
  JWT_EXPIRATION: "86400000"
  CORS_ALLOWED_ORIGINS: "https://fleetops.website"
  AWS_REGION: "us-east-1"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://$(POSTGRES_HOST):5432/auth_db"
```
ConfigMaps are not encrypted. They are appropriate for non-sensitive configuration. Sensitive values (passwords, keys, endpoints that reveal infrastructure) go in Secrets.

### Secret (synced by External Secrets Operator)
In FleetOps, K8s Secrets are NOT created manually — ESO creates and maintains them:
- `fleetops-postgres-secret` — POSTGRES_HOST, POSTGRES_USER, POSTGRES_PASSWORD (from Secrets Manager `fleetops/prod/db`)
- `fleetops-app-secret` — JWT_SECRET (from Secrets Manager `fleetops/prod/jwt`)
- `fleetops-redis-secret` — REDIS_HOST (from SSM `/fleetops/prod/redis/endpoint`)
- `fleetops-sns-secret` — INSURANCE_SNS_TOPIC_ARN, SERVICE_SNS_TOPIC_ARN (from SSM)
- `fleetops-lambda-service-secret` — LAMBDA_SERVICE_PASSWORD (from Secrets Manager)

EKS etcd is encrypted with KMS (`kms_secrets_key_arn`) — so K8s Secrets are encrypted at rest.

### ServiceAccount
```yaml
metadata:
  name: fleetops-app
  namespace: fleetops-prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::538661800892:role/fleetops-prod-app-irsa-role
```
The annotation is what activates IRSA. The EKS Pod Identity Webhook (built into EKS) sees this annotation and modifies the pod spec to inject the projected token volume and AWS env vars.

### HPA (HorizontalPodAutoscaler)
```yaml
spec:
  scaleTargetRef:
    kind: Deployment
    name: fleetops-auth-service
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```
HPA queries metrics-server every 15 seconds. If avg CPU across all pods > 70%, it calculates `desiredReplicas = ceil(currentReplicas * currentCPU / targetCPU)` and scales up. Scale-down has a 5-minute stabilization window to prevent thrashing.

**Why min 2:** Single replica = SPOF. During rolling update, the one old pod terminates while the new pod is starting — brief outage. Two replicas ensures one is always serving.

---

## Pod Startup Sequence (auth-service, complete)

1. **ArgoCD detects** new image tag in `values-prod.yaml`, syncs the Deployment
2. **Deployment controller** creates new ReplicaSet, marks pods as desired
3. **kube-scheduler** assigns pod to a worker node (scores by available CPU/memory)
4. **kubelet** on that node receives the pod spec
5. **Image pull:** containerd checks local cache for `538661800892.dkr.ecr.us-east-1.amazonaws.com/fleetops-dev/auth-service:v1.2.3`. If absent, authenticates to ECR via node IAM role, pulls layers. `imagePullPolicy: IfNotPresent` — skips pull if cached.
6. **EKS Pod Identity Webhook** injects:
   - Projected token volume at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
   - Env vars: `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`, `AWS_REGION`
7. **K8s Secrets and ConfigMap** injected as env vars (POSTGRES_HOST, POSTGRES_USER, POSTGRES_PASSWORD, JWT_SECRET, LAMBDA_SERVICE_PASSWORD, SPRING_PROFILES_ACTIVE, etc.)
8. **Container starts:** JVM launches, Spring Boot auto-configuration runs:
   - HikariCP pool opens N connections to RDS at `$(POSTGRES_HOST):5432/auth_db`
   - Flyway executes pending `V*.sql` migrations in version order
   - Spring Security JwtFilter registered with the `JWT_SECRET`
   - `LambdaServiceInitializer` @PostConstruct: seeds `lambda-service` account if absent
   - Embedded Tomcat starts listening on port 8080
9. **Startup probe** (`initialDelaySeconds: 30`, `periodSeconds: 10`, `failureThreshold: 10`): kubelet waits 30s, then calls `GET /actuator/health` up to 10 times. Spring Boot Actuator returns `{"status":"UP"}` when all health indicators pass (DB connection, disk space). Pod allowed to start receiving traffic.
10. **Readiness probe passes:** kubelet marks pod `Ready=True`. LBC registers pod IP in ALB target group. ALB begins routing requests to this pod.
11. **Liveness probe** (`periodSeconds: 15`, `failureThreshold: 3`): ongoing health check. Three consecutive failures → kubelet restarts container.

---

## How Traffic Reaches a Pod (complete packet flow)

```
1.  User opens https://fleetops.website/api/auth/login
2.  Browser DNS: queries OS resolver → Route53 → returns CloudFront IP (anycast)
3.  TLS handshake with CloudFront edge PoP (ACM cert for *.fleetops.website)
4.  CloudFront: WAFv2 inspection → not blocked → check cache → /api/* = no cache
5.  CloudFront forwards to origin: origin.fleetops.website, Host header = origin.fleetops.website
6.  Route53: origin.fleetops.website A record → ALB DNS → ALB node IPs
7.  TLS handshake with ALB (same ACM cert)
8.  ALB evaluates listener rules:
    - Host: origin.fleetops.website, Path: /api/auth → target group: fleetops-auth-service:8080
9.  ALB picks healthy pod IP from target group (health check: /actuator/health returns 200)
10. ALB sends HTTP/1.1 request directly to pod IP:8080 (target-type: ip, intra-VPC routing)
11. Spring Security JwtFilter: /api/auth/login is public → no JWT required → pass through
12. AuthController.login() → HikariCP pool → PostgreSQL auth_db → validate credentials
13. Generate JWT → return 200 {token: "eyJ..."}
14. Response: pod → ALB → CloudFront → edge PoP → browser
```

---

## Service-to-Service Communication

Example: maintenance-service calling vehicle-service:
```yaml
# maintenance-service application.yml
app:
  vehicle-service:
    url: http://fleetops-vehicle-service:8080
```

1. Java HTTP client calls `http://fleetops-vehicle-service:8080/api/vehicles/{id}`
2. JVM DNS → CoreDNS (at VPC DNS IP, 169.254.169.253 via `/etc/resolv.conf`)
3. CoreDNS resolves with search domain: `fleetops-vehicle-service.fleetops-prod.svc.cluster.local` → ClusterIP
4. kube-proxy iptables DNAT: ClusterIP → one of the vehicle-service pod IPs (random, consistent)
5. Packet routed within VPC to the pod IP
6. vehicle-service Spring Boot receives request, validates JWT if endpoint requires auth, responds

---

## Rolling Update Process

```
Deployment controller creates new ReplicaSet (new image tag)
    ↓
Scale up new RS by 1 (maxSurge: 1 by default)
New pod: schedule → pull image → startup probe → readiness probe PASS
    ↓
LBC registers new pod in ALB target group
    ↓
Scale down old RS by 1 (maxUnavailable: 0 by default)
Old pod: kubelet sends SIGTERM → Spring Boot graceful shutdown (finish in-flight requests) → exit
LBC deregisters old pod from ALB target group
    ↓
Repeat until all old pods replaced
    ↓
Old ReplicaSet kept at 0 replicas (rollback capability: kubectl rollout undo)
```

Zero-downtime because `maxUnavailable: 0` ensures at least `replicas` pods are always Ready and registered in the ALB.

---

## Kubernetes Q&A

**Q: Why does auth-service use `initialDelaySeconds: 30` on the startup probe?**
A: Spring Boot with Flyway takes 15–25s to start. 30s delay prevents kubelet from probing before the JVM is ready. failureThreshold: 10 × periodSeconds: 10 = 100s max startup window. If Spring Boot fails to start within 100s, kubelet kills and restarts the container.

**Q: What is the difference between liveness and readiness probes?**
A: Readiness controls traffic routing — a not-ready pod is removed from Service endpoints and ALB target group but keeps running (useful for temporary overload). Liveness controls container health — a pod failing liveness is restarted by kubelet. You might want a pod to be alive but not ready during startup.

**Q: What happens to in-flight requests when a pod terminates?**
A: ALB has a connection draining / deregistration delay (default 300s, configurable via annotation `alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30`). During this window, the ALB stops sending new requests to the terminating pod but allows existing connections to complete. Combined with Spring Boot's graceful shutdown (`server.shutdown: graceful`), in-flight requests complete before the pod exits.

**Q: What is target-type: ip vs target-type: instance?**
A: `instance` mode: ALB routes to EC2 node IP:NodePort, kube-proxy forwards NodePort → pod. Two network hops, SNAT hides source IP. `ip` mode: ALB routes directly to pod IP, one hop, source IP preserved. FleetOps uses `ip` for lower latency and accurate source IP logging in ALB access logs.

**Q: What is an ExternalSecret and how does it work?**
A: ExternalSecret is a CRD installed by External Secrets Operator. It declares a mapping: `fleetops/prod/db` in Secrets Manager → Kubernetes Secret `fleetops-postgres-secret`. ESO uses IRSA to authenticate to AWS, reads the secret, and creates/updates the K8s Secret. Every `refreshInterval: 1h`, ESO re-reads and reconciles. The app pod reads `fleetops-postgres-secret` as env vars — it never calls Secrets Manager directly.

**Q: Why does ArgoCD use sync waves?**
A: Waves enforce deployment order for dependent resources. Secrets (wave 0) must exist before pods (wave 3) try to mount them. The Ingress (wave 2) must exist before services register as targets. DB init job (wave 1) must complete before app pods start making DB queries. Without waves, ArgoCD deploys everything simultaneously and pods fail to start because secrets don't exist yet.

---

# PHASE 5 — IRSA & SECURITY MASTERCLASS

## IRSA — Complete Internal Flow

### Step 1: OIDC Provider (Terraform, one-time)
```hcl
resource "aws_iam_openid_connect_provider" "eks" {
  url            = "https://oidc.eks.us-east-1.amazonaws.com/id/XXXXX"
  client_id_list = ["sts.amazonaws.com"]
  thumbprint_list = [sha1_of_oidc_cert]
}
```
IAM now trusts JWT tokens signed by this EKS cluster.

### Step 2: IAM Trust Policy (per role)
The `fleetops-prod-app-irsa-role` trust policy from `modules/iam/main.tf`:
```json
"Condition": {
  "StringEquals": {
    "oidc.eks.us-east-1.amazonaws.com/id/XXXXX:sub":
      "system:serviceaccount:fleetops-prod:fleetops-app",
    "oidc.eks.us-east-1.amazonaws.com/id/XXXXX:aud":
      "sts.amazonaws.com"
  }
}
```
- `sub` check: only the `fleetops-app` SA in `fleetops-prod` namespace. No other pod can assume this role.
- `aud` check: token must be scoped to STS, not reusable for other services.

### Step 3: Token Projection
Pod starts with `serviceAccountName: fleetops-app`. EKS Pod Identity Webhook intercepts the pod creation and injects:
- Volume: projected ServiceAccount token at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` (OIDC JWT, default 24h expiry)
- Env vars automatically set on the container:
  ```
  AWS_ROLE_ARN=arn:aws:iam::538661800892:role/fleetops-prod-app-irsa-role
  AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
  AWS_REGION=us-east-1
  ```

### Step 4: SDK AssumeRoleWithWebIdentity
The AWS SDK credential provider chain checks for `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN`. When found:
```
SDK reads token from file
SDK calls sts.amazonaws.com/AssumeRoleWithWebIdentity(
  RoleArn = AWS_ROLE_ARN,
  WebIdentityToken = <content of token file>,
  RoleSessionName = "aws-sdk-java-session"
)
STS: validate JWT signature against OIDC public keys
STS: check sub == system:serviceaccount:fleetops-prod:fleetops-app ✓
STS: check aud == sts.amazonaws.com ✓
STS returns: AccessKeyId, SecretAccessKey, SessionToken (valid 1 hour)
```

### Step 5: Credential Use and Refresh
SDK caches temporary credentials. Auto-refreshes 5 minutes before expiry by calling AssumeRoleWithWebIdentity again. The OIDC token file is rotated by EKS before its 24h expiry. No human intervention required.

---

## IRSA vs Alternatives

**vs. Static IAM User Access Keys:**
- Static keys don't expire → unlimited exploit window after theft
- Must be stored in a K8s Secret or env var → attack surface
- Rotation is manual → operational burden
- IRSA credentials are ephemeral (1h), auto-rotated, never stored

**vs. EC2 Node Instance Profile:**
- Node profile permissions apply to ALL pods on that node
- If vehicle-service is compromised, it has auth-service's, maintenance-service's permissions too
- IRSA is pod-level, namespace-scoped. The trust policy's `sub` condition ensures only the exact service account can assume the role
- In FleetOps: `fleetops-app` SA can invoke Bedrock. If we used node instance profile, ALL pods (including frontend, which shouldn't call Bedrock) would have Bedrock access.

**vs. Kubernetes Secrets storing AWS credentials:**
- K8s Secrets are base64-encoded in etcd (encrypted with KMS in EKS, but readable by anyone with `kubectl get secret` RBAC)
- Appear in kubectl output, log aggregation systems, cluster backups
- IRSA credentials are never at rest anywhere — generated on demand

---

## IRSA Roles in This Project

| Role Name | SA:Namespace | Permissions |
|-----------|-------------|-------------|
| fleetops-prod-app-irsa-role | fleetops-app:fleetops-prod | SM read, S3 CRUD, Step Functions, EFS, Bedrock, SNS publish, CloudWatch logs |
| fleetops-prod-alb-controller-role | aws-load-balancer-controller:kube-system | EC2, ELB, ACM, WAF APIs for ALB provisioning |
| fleetops-prod-external-secrets-role | external-secrets:external-secrets | SM GetSecretValue, SSM GetParameter, KMS Decrypt |
| fleetops-prod-cluster-autoscaler-role | cluster-autoscaler:kube-system | ASG describe/modify, EC2 describe |
| fleetops-prod-cloudwatch-agent-role | cloudwatch-agent:amazon-cloudwatch | CloudWatch put metrics/logs |
| fleetops-prod-efs-csi-driver-role | efs-csi-controller-sa:kube-system | EFS describe/create access points |

---

## JWT Authentication Deep Dive

**Token issuance (auth-service):**
```java
Jwts.builder()
    .subject(username)
    .claim("roles", List.of("ROLE_MANAGER"))
    .claim("userId", user.getId())
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + jwtExpiration))  // 24h
    .signWith(Keys.hmacShaKeyFor(jwtSecret.getBytes()), Jwts.SIG.HS256)
    .compact();
// Result: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIi4uLn0.SIGNATURE
```

**Token validation (every service, Spring Security filter):**
```java
Jwts.parser()
    .verifyWith(Keys.hmacShaKeyFor(jwtSecret.getBytes()))
    .build()
    .parseSignedClaims(token);
// Throws JwtException if: signature invalid, token expired, malformed
```

All four services share `JWT_SECRET` from `fleetops-app-secret`. They validate locally — no auth-service call per request.

**Token structure:** `header.payload.signature` — base64url encoded.
- Header: `{"alg":"HS256","typ":"JWT"}`
- Payload: `{"sub":"user@example.com","roles":["ROLE_MANAGER"],"iat":1234567890,"exp":1234654290}`
- Signature: HMAC-SHA256 of `header.payload` using `JWT_SECRET`

---

## Defense in Depth Summary

| Layer | Control | Implementation |
|-------|---------|----------------|
| DNS | HTTPS-only, no HTTP access | Route53 + CloudFront |
| Edge | WAFv2 OWASP rules, rate limiting | modules/waf |
| Transport | TLS 1.2+ everywhere | ACM + ALB + RDS SSL + Redis TLS |
| Network | Private subnets, SGs, NetworkPolicies | modules/networking, k8s/policies/ |
| Identity (AWS) | IRSA — no static keys | modules/iam, OIDC |
| Identity (users) | JWT HS256, 24h expiry | auth-service |
| Secrets | Secrets Manager + KMS, ESO sync | modules/secrets-manager, external-secrets.yaml |
| Container | Non-root, read-only where possible | securityContext in all Helm charts |
| Images | Trivy scan blocks HIGH/CRITICAL CVEs | java-ci-ecr.yml step 4 |
| Dependencies | Snyk OSS vulnerability scan | java-ci-ecr.yml step 2 |
| IaC | Checkov misconfig scan | terraform-apply.yml job 1 |
| Code quality | SonarQube quality gate | java-ci-ecr.yml step 2 |
| Audit | CloudTrail all API calls, AWS Config | modules/cloudtrail, modules/config |
| Data at rest | KMS encryption: RDS, S3, EFS, etcd, SNS, SM | modules/kms |

---

## Security Q&A

**Q: What is the blast radius of a compromised app pod?**
A: The `fleetops-app` IRSA role gives: SM read for `fleetops/*`, S3 CRUD on vehicle-docs bucket, Bedrock invoke, Step Functions start, EFS mount, SNS publish on `fleetops-*` topics, CloudWatch log write. Impact: read all app secrets, exfiltrate vehicle documents, run up Bedrock costs, publish false alerts. Cannot: modify IAM, access other buckets, touch EKS infrastructure, access other AWS accounts.

**Q: How would you rotate the JWT secret with zero downtime?**
A: Current setup cannot do zero-downtime rotation (single secret). The production-grade approach: switch to RS256 asymmetric signing. auth-service signs with private key. All other services verify with public key. No shared secret to rotate. During rotation: issue new tokens with new private key, serve both old and new public keys for the token TTL window (24h), then remove old public key.

**Q: Why does External Secrets Operator need its own IRSA role instead of using the app role?**
A: Least privilege. ESO only needs SM read + SSM read. The app role has additional permissions (S3, Bedrock, Step Functions) that ESO has no need for. If ESO is compromised, the blast radius is limited to reading secrets — it cannot upload to S3 or invoke Bedrock.

**Q: What is Checkov and why is it in the CI pipeline?**
A: Checkov is a static analysis tool for Terraform. It checks for misconfigurations like: S3 buckets without encryption, security groups open to 0.0.0.0/0, RDS without deletion protection, unencrypted volumes. It runs in `terraform-apply.yml` Job 1 (Validate) — blocking infrastructure deployment if critical misconfigs are found. Some rules are suppressed with documented justifications (e.g., `#checkov:skip=CKV_AWS_260:Source scoped to ALB SG not 0.0.0.0/0`).

---

# PHASE 6 — MICROSERVICES MASTERCLASS

## Auth Service — Complete Reference

**Port:** 8080 | **Database:** `auth_db` | **ArgoCD sync wave:** 3

**Business rules:**
- Username must be unique
- Password hashed with BCrypt (strength 10)
- JWT expires in 86,400,000ms (24 hours)
- Roles: ADMIN, MANAGER, TECHNICIAN, USER
- Audit events logged for: login, logout, failed login, user creation, role change
- Lambda service account auto-seeded on startup via `LambdaServiceInitializer`

**Key endpoints:**
| Method | Path | Auth Required | Roles | Description |
|--------|------|---------------|-------|-------------|
| POST | /api/auth/register | No | — | Register new user |
| POST | /api/auth/login | No | — | Authenticate, receive JWT |
| POST | /api/auth/logout | Bearer | Any | Invalidate session |
| GET | /api/auth/me | Bearer | Any | Get own profile |
| PUT | /api/auth/users/{id}/role | Bearer | ADMIN | Change user role |
| GET | /api/audit/logs | Bearer | ADMIN | Retrieve audit log |

**Failure modes:**
- DB down → 503 on login (HikariCP pool exhausted)
- JWT_SECRET missing → application fails to start (Spring context fails)
- Flyway migration fails → application fails to start (prevents serving traffic with stale schema)

---

## Vehicle Service — Complete Reference

**Port:** 8080 | **Database:** `vehicle_db` | **ArgoCD sync wave:** 4

**Business rules:**
- License plate must be unique per fleet
- Vehicle status: ACTIVE, INACTIVE, UNDER_MAINTENANCE, DECOMMISSIONED
- AI analysis cached in Redis for 5 minutes per vehicle
- Fleet list cached in Redis with TTL 5 minutes, evicted on any CRUD operation
- S3 presigned URL valid for 15 minutes; document metadata stored in `vehicle_db`

**Key endpoints:**
| Method | Path | Auth | Roles | Description |
|--------|------|------|-------|-------------|
| GET | /api/vehicles | Bearer | Any | List vehicles (Redis cached) |
| POST | /api/vehicles | Bearer | MANAGER, ADMIN | Create vehicle |
| GET | /api/vehicles/{id} | Bearer | Any | Vehicle details |
| PUT | /api/vehicles/{id} | Bearer | MANAGER, ADMIN | Update vehicle |
| DELETE | /api/vehicles/{id} | Bearer | ADMIN | Delete vehicle |
| POST | /api/vehicles/{id}/ai-analysis | Bearer | Any | Bedrock analysis |
| GET | /api/vehicles/{id}/documents/upload-url | Bearer | Any | S3 presigned PUT URL |
| POST | /api/tracking | Bearer | TECHNICIAN, MANAGER | Record GPS event |
| GET | /api/tracking/{vehicleId} | Bearer | Any | Tracking history |

**Bedrock call sequence:**
1. `POST /api/vehicles/{id}/ai-analysis`
2. Check Redis cache key `ai-analysis:{vehicleId}` — if HIT, return cached
3. Fetch vehicle data from `vehicle_db`
4. Build Converse API request with vehicle data as user message
5. Call `bedrockClient.converse(request)` using IRSA credentials
6. Extract text from response: `response.output().message().content().get(0).text()`
7. Store in Redis with 5-min TTL
8. Return analysis text to client

**S3 presigned URL sequence:**
1. `GET /api/vehicles/{id}/documents/upload-url?filename=contract.pdf&contentType=application/pdf`
2. vehicle-service generates presigned URL: `s3Client.utilities().getPresignedPutObjectRequest(...)` with 15-min expiry
3. Returns `{uploadUrl: "https://fleetops-prod-vehicle-docs.s3.amazonaws.com/vehicles/{id}/contract.pdf?X-Amz-Signature=..."}`
4. Frontend PUTs the file directly to S3 (no proxy through pod)
5. Frontend calls `POST /api/vehicles/{id}/documents {filename, contentType, s3Key}` to register metadata

---

## Maintenance Service — Complete Reference

**Port:** 8080 | **Database:** `maintenance_db` | **ArgoCD sync wave:** 4

**Business rules:**
- Tasks have statuses: PENDING, IN_PROGRESS, COMPLETED, CANCELLED
- Insurance alerts triggered when vehicle insurance expires in ≤ 30 days
- Service alerts triggered when a task is overdue (past due date and still PENDING/IN_PROGRESS)
- File uploads: max 20MB per file, max 25MB per request
- Files stored on EFS at `/var/www/fleetops/shared-media/{taskId}/{filename}`
- All replicas of maintenance-service can read/write the same EFS directory (ReadWriteMany)

**Key endpoints:**
| Method | Path | Auth | Roles | Description |
|--------|------|------|-------|-------------|
| POST | /api/tasks | Bearer | Any | Create maintenance task |
| GET | /api/tasks | Bearer | Any | List tasks |
| GET | /api/tasks/{id} | Bearer | Any | Task details |
| PUT | /api/tasks/{id}/assign | Bearer | MANAGER | Assign technician |
| PUT | /api/tasks/{id}/start | Bearer | TECHNICIAN | Start work |
| PUT | /api/tasks/{id}/complete | Bearer | TECHNICIAN | Complete task |
| POST | /api/media/{taskId}/upload | Bearer | Any | Upload file to EFS |
| GET | /api/media/{taskId}/{filename} | Bearer | Any | Download from EFS |

**SNS publish (AlarmBroadcastService):**
```java
@Service
public class AlarmBroadcastService {
    public void publishInsuranceAlert(VehicleInsuranceAlert alert) {
        snsClient.publish(PublishRequest.builder()
            .topicArn(insuranceSnsTopicArn)  // from env var INSURANCE_SNS_TOPIC_ARN
            .message(objectMapper.writeValueAsString(alert))
            .subject("Vehicle Insurance Expiry Alert")
            .build());
    }
}
```

The SNS topic ARNs come from `fleetops-sns-secret` K8s Secret, synced by ESO from SSM parameters.

---

## Request Service — Complete Reference

**Port:** 8080 | **Database:** `request_db` | **ArgoCD sync wave:** 4

**Business rules:**
- Every service request creates a Step Functions execution
- Execution ARN stored in DB for status tracking
- Only MANAGER can approve/reject/assign
- Only TECHNICIAN can start/complete work
- Step Functions state transitions are the authoritative state source

**Step Functions state machine transitions:**
```
SUBMITTED  →  [startExecution()]  →  PENDING
PENDING    →  [manager approves]  →  APPROVED
PENDING    →  [manager rejects]   →  REJECTED
APPROVED   →  [manager assigns]   →  ASSIGNED
ASSIGNED   →  [technician starts] →  IN_PROGRESS
IN_PROGRESS → [technician done]   →  COMPLETED
```

**Execution tracking:**
```java
// On request create:
StartExecutionResponse sfnResp = sfnClient.startExecution(
    StartExecutionRequest.builder()
        .stateMachineArn(stateMachineArn)
        .name(request.getId().toString())  // unique execution name
        .input(toJson(requestPayload))
        .build());
request.setExecutionArn(sfnResp.executionArn());
requestRepository.save(request);

// On status query:
DescribeExecutionResponse exec = sfnClient.describeExecution(
    DescribeExecutionRequest.builder()
        .executionArn(request.getExecutionArn())
        .build());
return exec.status();  // RUNNING, SUCCEEDED, FAILED, ABORTED
```

---

## Microservices Q&A

**Q: Why four separate databases instead of one?**
A: Database-per-service enforces bounded context isolation. auth-service's `users` table schema can change without affecting vehicle-service. Each service scales its DB independently. The tradeoff: no cross-service JOINs — services must call each other's APIs (maintenance-service calls vehicle-service to get vehicle details for alerts). For consistency across services, this system uses eventual consistency rather than distributed transactions.

**Q: How does `$(POSTGRES_HOST)` substitution work?**
A: This is Kubernetes dependent variable substitution in the pod spec `env` array. `POSTGRES_HOST` is set first (from secretKeyRef). Later in the list, `SPRING_DATASOURCE_URL` value contains `$(POSTGRES_HOST)` — Kubernetes substitutes the value inline at pod start. This avoids hardcoding the RDS hostname in the ConfigMap, which must work across dev and prod environments.

**Q: Why is Redis host stored in SSM Parameter Store instead of Secrets Manager?**
A: The Redis endpoint is infrastructure configuration — not a secret credential. It does not grant access by itself; the security is provided by the Redis security group (only EKS nodes can connect on port 6379). SSM Parameter Store standard tier is free; Secrets Manager costs $0.40/secret/month. Using SSM for non-credential config is a cost-optimization best practice.

**Q: What happens if Step Functions execution fails?**
A: The execution moves to FAILED state with an error and cause. `describeExecution()` returns `ExecutionStatus.FAILED`. The request-service can detect this and surface an error state to the user. Step Functions has built-in retry and catch configurations in the state machine definition — retries can be configured per state for transient failures.

**Q: How is the Lambda service account password kept secure?**
A: Flow: 1) Terraform creates the secret in Secrets Manager at `fleetops/prod/lambda-service-credentials`. 2) ESO syncs it to K8s Secret `fleetops-lambda-service-secret`. 3) auth-service pod reads `LAMBDA_SERVICE_PASSWORD` from the K8s Secret. 4) `LambdaServiceInitializer` uses it to seed the DB user. 5) Lambda reads the same Secrets Manager secret directly (Lambda IAM role has `secretsmanager:GetSecretValue`). The password is never in Git, never in plaintext in any configuration file.

---

# PHASE 7 — AWS INTEGRATIONS MASTERCLASS

## Bedrock — Internal Flow

**Model:** `amazon.nova-lite-v1:0` — Amazon's cost-optimized language model for text generation.

**Converse API (used by vehicle-service):**
```
POST https://bedrock-runtime.us-east-1.amazonaws.com/model/amazon.nova-lite-v1:0/converse

Request body:
{
  "messages": [
    {
      "role": "user",
      "content": [{"text": "Vehicle ID: V-001, Make: Toyota, Model: Land Cruiser, Year: 2020, Last service: 6 months ago, Mileage: 45000km, Status: ACTIVE. Provide maintenance recommendations."}]
    }
  ],
  "inferenceConfig": {
    "maxTokens": 1024,
    "temperature": 0.7,
    "topP": 0.9
  }
}

Response:
{
  "output": {
    "message": {
      "role": "assistant",
      "content": [{"text": "Based on the vehicle data provided, here are maintenance recommendations..."}]
    }
  },
  "usage": {"inputTokens": 85, "outputTokens": 342}
}
```

**Authentication:** IRSA. The `fleetops-app` IRSA role has `bedrock:InvokeModel` on `arn:aws:bedrock:us-east-1::foundation-model/*`. The SDK uses the projected OIDC token to get STS temporary credentials and signs the Bedrock API request with SigV4.

**Caching strategy:** Redis key `ai-analysis:{vehicleId}` with 5-minute TTL. This reduces Bedrock cost (charged per token) and latency for repeated dashboard loads.

**Cost:** Nova Lite costs approximately $0.00006 per 1K input tokens and $0.00024 per 1K output tokens — very low for this use case (~$0.01 per 100 analyses).

---

## SNS — Publish and Delivery Flow

**Two topics:**
- `fleetops-prod-insurance-alerts` — insurance expiry warnings
- `fleetops-prod-service-alerts` — maintenance overdue alerts

**Publish path (maintenance-service → SNS):**
1. `AlarmBroadcastService.publishInsuranceAlert()` called
2. SNS client (using IRSA credentials) calls `SNS:Publish`
3. SNS receives message, stores durably, fans out to all subscriptions
4. Email subscription: SNS delivers to SES/SMTP for the `alert_emails` list
5. Retry: SNS retries failed deliveries with exponential backoff (3 immediate, 2 pre-backoff, 10 post-backoff)
6. Dead letter: undeliverable messages after retry limit go to DLQ (if configured)

**Message format sent to SNS:**
```json
{
  "vehicleId": "V-001",
  "licensePlate": "ABC-1234",
  "insuranceExpiryDate": "2026-07-15",
  "daysUntilExpiry": 23,
  "alertType": "INSURANCE_EXPIRY",
  "timestamp": "2026-06-22T09:00:00Z"
}
```

**KMS encryption:** SNS topic encrypted with `events_key_arn`. Messages at rest in SNS are encrypted.

---

## Lambda — Complete Lifecycle

**Function:** `fleetops-prod-alert-processor` (Node.js runtime)

**Cold start sequence:**
1. EventBridge fires `rate(1 day)` rule
2. Lambda service finds no warm execution environment
3. Lambda creates new execution environment: provisions microVM, loads Node.js runtime, downloads function code from S3
4. Initializes the function module (requires, global setup) — cold start overhead: 100–500ms
5. Invokes the handler function

**Warm invocation (subsequent same-day runs would be warm):** Lambda reuses the execution environment — handler runs in ~50ms without the cold start overhead.

**Handler logic:**
```javascript
exports.handler = async (event) => {
  // 1. Authenticate to auth-service
  const tokenResp = await fetch(`${AUTH_SERVICE_URL}/api/auth/login`, {
    method: 'POST',
    body: JSON.stringify({ username: 'lambda-service', password: LAMBDA_SERVICE_PASSWORD })
  });
  const { token } = await tokenResp.json();

  // 2. Fetch vehicles with expiring insurance
  const vehiclesResp = await fetch(`${VEHICLE_SERVICE_URL}/api/vehicles?insuranceExpiringSoon=true`, {
    headers: { Authorization: `Bearer ${token}` }
  });
  const vehicles = await vehiclesResp.json();

  // 3. Fetch overdue maintenance tasks
  const tasksResp = await fetch(`${MAINTENANCE_SERVICE_URL}/api/tasks?status=overdue`, {
    headers: { Authorization: `Bearer ${token}` }
  });
  const tasks = await tasksResp.json();

  // 4. Publish alerts to SNS
  for (const vehicle of vehicles) {
    await snsClient.send(new PublishCommand({
      TopicArn: INSURANCE_SNS_ARN,
      Message: JSON.stringify(vehicle)
    }));
  }
  // ... similar for tasks
};
```

**Environment variables** injected by Terraform (`modules/lambda`):
- `VEHICLE_SERVICE_URL` — `https://origin.fleetops.website` (bypasses CloudFront, uses ALB directly)
- `AUTH_SERVICE_URL` — same
- `LAMBDA_SERVICE_CREDENTIALS_SECRET_ARN` — reads password from Secrets Manager at runtime
- `INSURANCE_SNS_ARN`, `SERVICE_SNS_ARN`

**IAM:** Lambda role (`fleetops-prod-lambda-role`) has:
- `AWSLambdaBasicExecutionRole` — CloudWatch Logs
- `sns:Publish` on `fleetops-*` topics
- `secretsmanager:GetSecretValue` on `fleetops/*` secrets

---

## Step Functions — State Machine Internals

**State machine type:** Standard (exactly-once execution, long-running, up to 1 year)

**States definition (conceptual — defined in `modules/step-functions`):**
```json
{
  "StartAt": "PendingApproval",
  "States": {
    "PendingApproval": {
      "Type": "Wait",
      "Comment": "Waiting for manager to approve or reject",
      "Next": "CheckDecision"
    },
    "CheckDecision": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.decision", "StringEquals": "APPROVED", "Next": "Approved" },
        { "Variable": "$.decision", "StringEquals": "REJECTED", "Next": "Rejected" }
      ]
    },
    "Approved": { "Type": "Pass", "Next": "WaitForAssignment" },
    "Rejected": { "Type": "Succeed" },
    "WaitForAssignment": { "Type": "Wait", "Next": "InProgress" },
    "InProgress": { "Type": "Wait", "Next": "Completed" },
    "Completed": { "Type": "Succeed" }
  }
}
```

**Execution lifecycle:**
1. request-service calls `startExecution()` → execution created with unique name (request UUID)
2. Execution enters first state
3. State transitions are triggered by `sendTaskSuccess()` / `sendTaskFailure()` calls from request-service when events occur (approval, rejection, assignment)
4. Each state transition is logged to CloudWatch Logs
5. `describeExecution()` returns current status and state output at any time

**Exactly-once guarantee:** If request-service crashes after `startExecution()` but before saving the execution ARN, it can look up the execution by its name (request UUID) on restart.

**KMS:** Step Functions execution history encrypted with `events_key_arn`.

---

## Redis — Cache Operations in Detail

**Cache hit flow (vehicle fleet list):**
```
GET /api/vehicles
  → Spring @Cacheable("vehicles", key="fleet:all")
  → CacheManager: GET "vehicles::fleet:all" from Redis
  → Redis: key exists, not expired
  → Return deserialized List<Vehicle> from Redis bytes
  → No DB query. Response in ~5ms.
```

**Cache miss flow:**
```
GET /api/vehicles (cache empty or expired)
  → Redis: key absent (TTL expired or first request)
  → Execute method: SELECT * FROM vehicles in vehicle_db
  → Spring: serialize result to bytes, SET "vehicles::fleet:all" PX 300000 (5min TTL)
  → Return result. Response in ~50-200ms (DB query).
```

**Cache eviction on write:**
```
POST /api/vehicles (create vehicle)
  → Spring @CacheEvict(value="vehicles", allEntries=true)
  → Redis: DEL all keys in "vehicles" cache
  → Next GET will be a cache miss, re-fetches from DB
```

**TTL expiry:** Redis automatically removes keys after 5 minutes. The application doesn't need to manage expiry.

**Connection:** ElastiCache Redis endpoint injected as `REDIS_HOST` env var. Spring Boot Redis auto-configuration creates a Lettuce connection pool to `{REDIS_HOST}:6379`.

---

# PHASE 8 — CI/CD MASTERCLASS

## CI Workflow — java-ci-ecr.yml (Reusable)

Called by each service's `ci.yml`. 7 jobs, runs on push to any branch and on PRs.

### Job 1: prepare
**Purpose:** Determine the image tag for this build.
**Logic (from actual workflow code):**
- `main` branch: inspect commit messages since last git tag. If contains `BREAKING CHANGE` → major bump. If `feat:` → minor bump. If `fix:` → patch bump. Result: `v1.2.3`. If no bump-worthy commit, uses `main-{SHA:7}`.
- `develop` branch: always `develop-{SHA:7}` (e.g., `develop-a3f92c1`)
- feature branches: `{branch-name}-{SHA:7}` (e.g., `feature/login-a3f92c1`)

**Output:** `image-tag`, `full-image` (ECR URL + tag), `should-release`, `version`

**Why semantic versioning from commit messages?** Conventional Commits standard (feat/fix/BREAKING CHANGE) makes version bumps automatic and auditable. No manual tagging. The tag is immutable — `v1.2.3` always refers to the same code.

### Job 2: quality
**Runs on:** ubuntu-latest with a real PostgreSQL 15 service container (port 5432)

**Steps:**
1. `actions/setup-java@v4` with Temurin JDK 21 and Maven cache
2. `mvn -B clean verify` — compiles, runs unit and integration tests against the real PostgreSQL service
3. SonarQube scan: `mvn sonar:sonar -Dsonar.projectKey={key} -Dsonar.token={token}`
4. Quality gate check: polls SonarQube API until the analysis task completes, then checks the quality gate status. Fails if `ERROR` (blocks merge/push).
5. Snyk OSS scan: checks all Maven dependencies for known CVEs with `--severity-threshold=high`

**Why a real PostgreSQL in CI?** Mocking the database was intentionally avoided — integration tests must prove that SQL queries, Flyway migrations, and JPA mappings work against a real database engine.

### Job 3: build
**Dependencies:** prepare, quality (both must pass)
- Docker Buildx builds the multi-stage image
- Image saved as tar artifact (`docker save`) — not pushed yet
- `load: true` loads image into Docker daemon for Trivy scanning

### Job 4: trivy-scan
- Downloads the image tar artifact
- Runs Trivy: `severity: HIGH,CRITICAL`, `exit-code: 1` → blocks if vulnerabilities found
- `ignore-unfixed: true` — ignores CVEs with no available fix (no false blocking)
- Results uploaded to GitHub Security tab (SARIF format)

### Job 5: push
**Only runs on:** push events to `main` or `develop` (NOT on PRs)
- Downloads image artifact
- OIDC authentication: `configure-aws-credentials@v4` with `role-to-assume: AWS_ECR_ROLE_ARN`
  - GitHub Actions exchanges its OIDC token for temporary AWS credentials
  - No `AWS_ACCESS_KEY_ID` stored in GitHub Secrets
- `amazon-ecr-login@v2` → authenticates Docker to ECR
- `docker push {full-image}`

### Job 6: release
**Only runs when:** `should-release == 'true'` AND branch is `main` AND push succeeded
- Creates git tag `v1.2.3` and pushes it
- Creates GitHub Release with auto-generated notes
- Release artifacts: the tagged image in ECR is the release artifact

### Job 7: notify
- Runs `always()` on push events
- Sends success/failure email via SMTP using `dawidd6/action-send-mail@v3`
- Contains service name, tag, branch, triggering actor, and link to workflow run on failure

---

## CD Workflow — cd-ecr.yml (Reusable)

Triggered by the CI workflow completion (via `workflow_run` event in service `cd.yml`).

### Dev deployment job:
1. Resolve tag: `develop-{HEAD_SHA:7}` from the CI run's `head-sha` input
2. Generate GitHub App token using `APP_ID` + `APP_PRIVATE_KEY` — this gives write access to `fleetops-deployments` repo
3. Checkout `FleetOps-V2/fleetops-deployments`
4. Update `charts/{service}/values-dev.yaml`: `yq e ".image.tag = \"${TAG}\"" -i ${VALUES_FILE}`
5. Git commit: `chore({service}): bump dev image tag to {tag} [skip ci]`
6. `git pull --rebase origin main && git push`

**Why GitHub App token instead of PAT?** GitHub Apps have fine-grained repository permissions and their tokens expire in 1 hour. A PAT has broad user-level permissions and doesn't expire unless manually rotated. `[skip ci]` in the commit message prevents the deployments repo's CI from triggering (no CI on that repo).

### Prod deployment job:
- Requires `environment: production` — this is a GitHub Environments gate (can require manual approval)
- Resolves the latest semver tag from git tags (the one created by the CI release job)
- Updates `charts/{service}/values-prod.yaml`

### Smoke test job:
- Waits 90 seconds for ArgoCD to detect and sync the change
- curl with `--retry 5 --retry-delay 15` to the service's health URL
- `HTTP 2xx or 4xx` = deployment succeeded (service is up, may require auth)
- `HTTP 5xx or 000` = deployment failed → job fails → team notified

---

## Terraform CI/CD Workflows

### terraform-plan.yml:
```
Trigger: PR modifying environments/** or modules/**
Steps:
  1. AWS OIDC auth (role from AWS_ROLE_ARN secret)
  2. terraform init (uses S3 backend, DynamoDB lock)
  3. terraform validate (syntax check)
  4. terraform plan -var DB_PASSWORD=$TF_VAR_DB_PASSWORD ...
  5. Post plan output as PR comment (truncated to 60KB)
```
Engineers review the plan before merging. ALB ARN suffix, subnet IDs, etc. are all visible in the plan.

### terraform-apply.yml:
```
Trigger: push to main, or manual workflow_dispatch
Job 1 - Validate:
  - terraform fmt -check (formatting gate)
  - terraform validate
  - Checkov security scan
Job 2 - Plan:
  - terraform plan -out=tfplan (saved as artifact)
Job 3 - Apply (needs manual approval for prod):
  - Download tfplan artifact
  - terraform apply tfplan
  - Extract outputs (ACM cert ARN, ALB SG ID, EFS filesystem ID, Step Functions ARN)
  - Write environments/prod/infra-values.yaml
  - Push infra-values.yaml to fleetops-deployments repo
  - Mirror external-secrets image to ECR
Job 4 - Print outputs:
  - Print GitHub Actions role ARN, ECR role ARN, DevOps agent role ARN
```

**Why extract outputs to infra-values.yaml?** Helm charts need the ACM certificate ARN (for the ALB Ingress annotation) and EFS filesystem ID (for the EFS PVC). These are known only after Terraform runs. Terraform writes them to `environments/prod/infra-values.yaml`, commits to the deployments repo, ArgoCD syncs the updated values.

---

## OIDC Authentication (GitHub Actions → AWS)

No static AWS credentials in GitHub Secrets. Here's how it works:

1. Workflow has `permissions: id-token: write` — GitHub generates an OIDC token for the job
2. `configure-aws-credentials` action calls GitHub OIDC endpoint to get the JWT
3. Action calls `sts:AssumeRoleWithWebIdentity` with the JWT
4. AWS IAM verifies the JWT against GitHub's OIDC provider (`token.actions.githubusercontent.com`)
5. Trust policy condition checks: `repo:FleetOps-V2/fleetops-auth-service:ref:refs/heads/main` — only this specific repo+branch can assume the role
6. STS returns temporary credentials (1-hour expiry)
7. GitHub Actions runner environment has `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` set for the rest of the job

---

## CI/CD Q&A

**Q: Why does the push job only run on push events, not PRs?**
A: PRs come from forks or feature branches. Pushing an image to ECR for every PR would pollute the ECR repository with unreviewed images and waste ECR storage costs. PRs get quality checks (test, SonarQube, Trivy scan) but no deployment. Only reviewed and merged code on `main`/`develop` gets pushed to ECR.

**Q: What is `[skip ci]` in the CD commit message and why is it needed?**
A: GitHub Actions will re-trigger on any push to a repo, including automated commits. When the CD workflow pushes to `fleetops-deployments`, adding `[skip ci]` (a GitHub-recognized special string) tells GitHub Actions to skip workflow triggers for that commit. Without it, pushing the image tag update would trigger another CI run on the deployments repo, creating an infinite loop.

**Q: What is the ArgoCD sync latency from git push to pod running?**
A: Total time: CI pipeline (~5-8 min) + CD pipeline (~2 min) + ArgoCD sync delay (up to 3 min polling, instant with webhook) + rolling update (~2-3 min). Total: ~12-16 minutes from `git push` to new pod serving traffic. With ArgoCD webhook configuration, the sync delay drops to ~30 seconds.

**Q: Why is `force_update = true` set on the ArgoCD Helm release in Terraform?**
A: ArgoCD's Helm chart sometimes has resources that can't be updated in-place (e.g., immutable fields in CustomResourceDefinitions). `force_update = true` allows Terraform to force-replace the Helm release if a normal update fails. This is acceptable for the ArgoCD installation since it manages other apps declaratively and can be reinstalled without data loss.

**Q: How does semantic versioning work in the CI pipeline?**
A: The `prepare` job fetches all git tags, finds the latest `v*` tag, and inspects commit messages since that tag. Following Conventional Commits: `BREAKING CHANGE` or `!:` suffix → major bump; `feat:` prefix → minor bump; `fix:` prefix → patch bump. No bump-worthy commit since last tag → no release created. This automates versioning without human intervention.

---

# PHASE 9 — END-TO-END FLOWS

## Flow 1: User Login

```
1. Browser: POST https://fleetops.website/api/auth/login
   Body: { "username": "manager@fleet.com", "password": "SecurePass123" }

2. Route53 → CloudFront → WAF inspection (no block) → origin.fleetops.website
3. ALB: /api/auth → fleetops-auth-service:8080
4. Spring Security: /api/auth/login is permittAll() — no JWT required
5. AuthController.login():
   a. UserDetailsService.loadUserByUsername("manager@fleet.com") → query auth_db.users
   b. BCryptPasswordEncoder.matches(rawPassword, hashedPassword) → true
   c. Generate JWT: sub=manager@fleet.com, roles=[ROLE_MANAGER], exp=now+24h, sign with HS256
   d. Record audit event: LOGIN_SUCCESS to audit_log table
6. Response 200: { "token": "eyJ...", "expiresIn": 86400000, "roles": ["ROLE_MANAGER"] }
7. CloudFront cache: /api/* not cached → pass through to browser
8. Browser stores token in memory (React state / sessionStorage)
```

**Logs generated:** ALB access log (IP, path, response code, latency), Spring Boot log (INFO: login attempt, result), CloudTrail (not applicable — no AWS API call for login).

---

## Flow 2: Vehicle Creation

```
1. Manager: POST https://fleetops.website/api/vehicles
   Headers: Authorization: Bearer eyJ...
   Body: { "make": "Toyota", "model": "Hilux", "year": 2023, "licensePlate": "KBZ-001" }

2. CloudFront → ALB → /api/vehicles → fleetops-vehicle-service:8080
3. Spring Security JwtFilter:
   a. Extract Bearer token
   b. Parse and verify signature using JWT_SECRET
   c. Extract claims: sub=manager@fleet.com, roles=[ROLE_MANAGER]
   d. Set SecurityContext
4. @PreAuthorize("hasRole('MANAGER')") → ROLE_MANAGER present → allow
5. VehicleController.createVehicle():
   a. Validate: licensePlate unique? → SELECT FROM vehicles WHERE license_plate = 'KBZ-001'
   b. Persist: INSERT INTO vehicles (...) → vehicle_db
   c. @CacheEvict("vehicles", allEntries=true) → DELETE vehicles::fleet:all in Redis
6. Response 201: { "id": "V-045", "make": "Toyota", ... }
```

**Side effects:** Redis cache evicted for fleet list. Next GET /api/vehicles will be a cache miss and re-query the DB (correct behavior — new vehicle must appear).

---

## Flow 3: AI Fleet Analysis

```
1. Manager: POST https://fleetops.website/api/vehicles/V-045/ai-analysis
   Headers: Authorization: Bearer eyJ...

2. ALB → fleetops-vehicle-service:8080
3. JWT validated → ROLE_MANAGER authorized
4. VehicleController.getAiAnalysis("V-045"):
   a. Check Redis: GET "vehicles::ai-analysis:V-045" → MISS
   b. Fetch vehicle from vehicle_db: SELECT * FROM vehicles WHERE id = 'V-045'
   c. Fetch maintenance history from maintenance-service:
      GET http://fleetops-maintenance-service:8080/api/tasks?vehicleId=V-045
      (internal service call via CoreDNS → ClusterIP → pod)
   d. Build Bedrock prompt with vehicle + maintenance data
   e. AWS SDK: AssumeRoleWithWebIdentity (IRSA) → STS temporary credentials
   f. Call Bedrock Converse API:
      POST bedrock-runtime.us-east-1.amazonaws.com/model/amazon.nova-lite-v1:0/converse
   g. Response: AI-generated maintenance recommendations text
   h. Cache: SET "vehicles::ai-analysis:V-045" <analysis_json> PX 300000
5. Response 200: { "vehicleId": "V-045", "analysis": "Based on the vehicle data...", "generatedAt": "..." }
```

**AWS calls:** STS AssumeRoleWithWebIdentity (IRSA refresh), Bedrock InvokeModel.

---

## Flow 4: Service Request Creation

```
1. Technician: POST https://fleetops.website/api/requests
   Body: { "vehicleId": "V-045", "type": "OIL_CHANGE", "priority": "HIGH", "description": "Overdue oil change" }

2. ALB → fleetops-request-service:8080
3. JWT validated → any authenticated user can create requests
4. RequestController.createRequest():
   a. Validate vehicleId exists: GET http://fleetops-vehicle-service:8080/api/vehicles/V-045
   b. Persist: INSERT INTO requests (id=UUID, vehicle_id, type, status='PENDING', ...) → request_db
   c. Start Step Functions execution:
      sfnClient.startExecution(stateMachineArn, name=requestId, input=requestJson)
      → returns executionArn
   d. UPDATE requests SET execution_arn = ? WHERE id = requestId
5. Response 201: { "id": "REQ-101", "status": "PENDING", "executionArn": "arn:aws:states:..." }
```

**AWS calls:** STS (IRSA), Step Functions startExecution.

---

## Flow 5: Request Approval

```
1. Manager: PUT https://fleetops.website/api/requests/REQ-101/approve
   Body: { "comment": "Approved for next week" }

2. ALB → fleetops-request-service:8080
3. JWT validated → @PreAuthorize("hasRole('MANAGER')") → allow
4. RequestController.approveRequest("REQ-101"):
   a. Fetch request from request_db → get executionArn
   b. sfnClient.sendTaskSuccess(taskToken, output=approvalJson)
      → Step Functions transitions: PENDING → APPROVED
   c. UPDATE requests SET status = 'APPROVED' WHERE id = 'REQ-101'
5. Response 200: { "id": "REQ-101", "status": "APPROVED" }
```

---

## Flow 6: Technician Assignment

```
1. Manager: PUT https://fleetops.website/api/requests/REQ-101/assign
   Body: { "technicianId": "T-007" }

2. ALB → request-service:8080
3. JWT + ROLE_MANAGER
4. RequestController.assignTechnician():
   a. Validate technicianId: GET http://fleetops-auth-service:8080/api/auth/users/T-007
   b. Step Functions: sendTaskSuccess(token, {assignedTo: "T-007"})
      → State machine: APPROVED → ASSIGNED
   c. UPDATE requests SET technician_id = 'T-007', status = 'ASSIGNED'
   d. SNS notify: maintenance-service publishes SERVICE_SNS_TOPIC alert about new assignment
5. Response 200: { "id": "REQ-101", "status": "ASSIGNED", "technicianId": "T-007" }
```

---

## Flow 7: Vehicle Document Upload

```
1. Manager: GET https://fleetops.website/api/vehicles/V-045/documents/upload-url
   Params: ?filename=insurance.pdf&contentType=application/pdf

2. ALB → vehicle-service:8080
3. JWT validated
4. VehicleController.getUploadUrl():
   a. IRSA: SDK uses projected token to get STS temp creds
   b. S3 presigned URL: PUT s3://fleetops-prod-vehicle-docs/vehicles/V-045/insurance.pdf
      Expiry: 900s (15 min)
5. Response 200: { "uploadUrl": "https://fleetops-prod-vehicle-docs.s3.amazonaws.com/vehicles/V-045/insurance.pdf?X-Amz-Algorithm=...&X-Amz-Expires=900&..." }

6. Browser: PUT {uploadUrl} (direct S3 upload, NO proxy through pod)
   Body: PDF binary data
   Result: S3 stores object, returns 200

7. Manager: POST https://fleetops.website/api/vehicles/V-045/documents
   Body: { "filename": "insurance.pdf", "contentType": "application/pdf", "s3Key": "vehicles/V-045/insurance.pdf" }
8. VehicleController.registerDocument(): INSERT into vehicle_documents table
9. Response 201: document metadata
```

**AWS calls:** S3 GeneratePresignedUrl (vehicle-service pod). S3 PutObject (browser directly to S3 — no pod involved).

---

## Flow 8: Daily Alert Processing (Lambda)

```
EventBridge: rate(1 day) fires at 00:00 UTC
  ↓
Lambda: fleetops-prod-alert-processor invoked
  ↓
Lambda reads LAMBDA_SERVICE_CREDENTIALS_SECRET_ARN
→ secretsmanager:GetSecretValue → { password: "..." }
  ↓
Lambda: POST https://origin.fleetops.website/api/auth/login
  { username: "lambda-service", password: "..." }
→ Receives JWT token
  ↓
Lambda: GET https://origin.fleetops.website/api/vehicles?insuranceExpiringSoon=true
  Authorization: Bearer {token}
→ vehicle-service queries DB, returns vehicles with insurance expiring in ≤30 days
  ↓
For each vehicle: SNS Publish to fleetops-prod-insurance-alerts topic
→ Email subscription receives alert
  ↓
Lambda: GET https://origin.fleetops.website/api/tasks?overdue=true
→ maintenance-service returns overdue tasks
  ↓
For each overdue task: SNS Publish to fleetops-prod-service-alerts topic
  ↓
Lambda logs: CloudWatch Log Group /aws/lambda/fleetops-prod-alert-processor
```

**Why `origin.fleetops.website` not `fleetops.website`?** Lambda calls from outside the VPC go through CloudFront at `fleetops.website`. CloudFront adds latency and may cache GET responses. `origin.fleetops.website` bypasses CloudFront and hits the ALB directly — faster, no caching issues.

---

## Flow 9: Alarm Broadcast (Maintenance Service SNS)

```
Maintenance task created with status OVERDUE detected by scheduled check:
  ↓
MaintenanceScheduler @Scheduled(cron="0 0 8 * * ?") runs at 8am daily
  ↓
AlarmBroadcastService.scanAndBroadcast():
  a. Query maintenance_db: SELECT * FROM tasks WHERE status='PENDING' AND due_date < NOW()
  b. For each overdue task:
     - Fetch vehicle details: GET http://fleetops-vehicle-service:8080/api/vehicles/{vehicleId}
     - Publish SNS: PublishRequest(topicArn=SERVICE_SNS_TOPIC_ARN, message=json)
     - IRSA STS credentials used for SNS Publish
  c. Query maintenance_db: SELECT * FROM vehicles WHERE insurance_expiry <= NOW() + 30 days
     (requires cross-service data: maintenance-service joins with vehicle-service data)
     - Publish SNS: PublishRequest(topicArn=INSURANCE_SNS_TOPIC_ARN, message=json)
  ↓
SNS fans out to email subscribers (configured via alert_emails variable)
```

---

## Flow 10: Live GPS Tracking

```
1. Vehicle GPS device (or frontend simulator):
   POST https://fleetops.website/api/tracking
   Authorization: Bearer {technicianToken}
   Body: { "vehicleId": "V-045", "lat": -1.2921, "lng": 36.8219, "speed": 65, "timestamp": "..." }

2. CloudFront → ALB → vehicle-service:8080
3. JWT validated (TECHNICIAN or MANAGER role)
4. TrackingController.recordLocation():
   a. Validate vehicleId exists
   b. INSERT INTO tracking_events (vehicle_id, lat, lng, speed, timestamp)
   c. Optional: @CacheEvict for vehicle's latest position cache
5. Response 201

6. Frontend polls: GET https://fleetops.website/api/tracking/V-045?last=100
7. vehicle-service: SELECT * FROM tracking_events WHERE vehicle_id = 'V-045' ORDER BY timestamp DESC LIMIT 100
8. Returns GPS track history
```

---

# PHASE 10 — FINAL EVALUATOR DEFENSE

## Top 100 General Evaluator Questions

### Architecture

**Q1: Why microservices instead of a monolith?**
- Basic: Each service can be developed, deployed, and scaled independently. A bug in maintenance-service doesn't take down auth-service.
- Deeper: Database-per-service enables independent schema evolution. CI/CD pipelines are per-service, so teams can release without coordinating. In FleetOps, vehicle-service needs Bedrock + Redis; auth-service needs none of that — separate deployments allow separate dependency management.
- Expert: The tradeoffs are real. Distributed systems introduce latency (internal HTTP calls), consistency challenges (no distributed transactions — eventual consistency via Saga pattern), and operational complexity (4 CI pipelines, 4 deployments, 4 databases to monitor). A monolith would have been simpler to start; the microservices architecture is justified by the independent scaling and deployment requirements of each service.

**Q2: How does CloudFront improve the system?**
- Basic: Caches static assets at edge PoPs close to users, reducing latency and ALB load.
- Deeper: JavaScript/CSS bundles for the React SPA are served from edge locations (400+ globally). A user in Tokyo gets sub-10ms asset load time from the nearest PoP instead of cross-Pacific latency to us-east-1. API calls (/api/*) bypass cache and route to origin.
- Expert: CloudFront also provides WAFv2 attachment point — WAF must run at CLOUDFRONT scope for global protection. The ALB becomes an internal origin, reducing its attack surface. CloudFront absorbs DDoS at layer 7 via WAF rate limiting and at layer 3/4 via AWS Shield Standard (free, always on).

**Q3: Why is the ALB NOT provisioned by Terraform?**
- Basic: The AWS Load Balancer Controller running in EKS creates the ALB automatically when it sees an Ingress resource with the `alb` ingress class.
- Deeper: This is the Kubernetes-native approach — infra follows app declarations. If we provisioned the ALB in Terraform, we'd need to manually update Terraform every time routing rules change (new service, new path). With the LBC, adding a new path rule is just updating the Ingress YAML in the deployments repo — ArgoCD syncs it, LBC provisions the listener rule. Zero Terraform changes.
- Expert: The shared `group.name: fleetops` allows multiple Ingress objects (app + ArgoCD) to share one ALB. LBC merges the rules. This saves ~$36/month in ALB costs vs. separate ALBs.

**Q4: Explain the complete secret lifecycle from creation to pod consumption.**
- Basic: Secrets Manager stores credentials → ESO reads them → K8s Secrets created → pods read as env vars.
- Deeper: Terraform creates the secret in Secrets Manager with KMS encryption. ESO (running with its own IRSA role) reads the secret using `secretsmanager:GetSecretValue`. ESO creates a K8s Secret with `creationPolicy: Owner` — ESO owns the Secret and updates it every `refreshInterval: 1h`. The pod spec references the K8s Secret via `secretKeyRef`. The kubelet injects the value as an env var at pod start. The env var is in the pod's environment memory — never on disk (unless heap dump occurs).
- Expert: The `refreshInterval: 1h` means there's up to 1 hour of staleness in K8s Secrets after a rotation in Secrets Manager. To propagate immediately: force-refresh ESO or restart pods. EKS etcd encryption (KMS `kms_secrets_key_arn`) means the K8s Secret bytes are encrypted at rest in etcd. The full chain: Secrets Manager (KMS encrypted) → ESO (IRSA — no stored credentials) → K8s Secret (etcd KMS encrypted) → pod env var (in-memory).

**Q5: What would happen if ArgoCD is deleted?**
- Basic: GitOps sync stops. Existing pods continue running — Kubernetes doesn't need ArgoCD to run pods. New deployments won't happen automatically.
- Deeper: Since `selfHeal: true`, if ArgoCD is gone, manual kubectl changes would persist (no revert). To recover: reinstall ArgoCD, re-apply the root app. ArgoCD reads the Git state and reconciles — all apps return to the Git-desired state. No data loss.
- Expert: This is the beauty of GitOps. The Git repository IS the system state. ArgoCD is just the reconciliation engine. Any other GitOps tool (Flux, Jenkins X) could replace it. The state in Git is authoritative, not the state in ArgoCD's own database.

---

## Top 50 AWS Questions

**Q6: What is the difference between Secrets Manager and SSM Parameter Store?**
Secrets Manager: $0.40/secret/month, automatic rotation hooks, cross-account sharing, version history, JSON value support. Best for: credentials, API keys.
SSM Parameter Store: free (standard tier), SecureString uses KMS, hierarchical naming `/fleetops/prod/redis/endpoint`, no auto-rotation. Best for: configuration values, ARNs, endpoints.
FleetOps uses both: SM for DB creds/JWT/PAT (need rotation capability), SSM for Redis endpoint/SNS ARNs (config, not credentials).

**Q7: What is an OIDC provider and why does EKS need one?**
OIDC (OpenID Connect) is a standard for identity federation. EKS issues OIDC tokens to pods via projected service accounts. AWS IAM must trust these tokens to allow pods to assume IAM roles. The OIDC provider registration (`aws_iam_openid_connect_provider`) tells IAM: "tokens signed by this EKS cluster's OIDC issuer URL are valid for trust evaluation." Without this, IRSA cannot work — IAM would reject the projected tokens.

**Q8: Why is the VPC CIDR 10.2.0.0/16?**
10.2.0.0/16 provides 65,534 IP addresses. Sufficient for EKS pods (each pod gets its own VPC IP via the VPC CNI), RDS, Redis, EFS mount targets across AZs. The /16 was chosen to not conflict with 10.0.x.x or 10.1.x.x ranges commonly used for VPN or peering scenarios. In AWS, EKS nodes get IPs from the node subnet, and pods get IPs from the same subnet (VPC CNI mode) — a /16 gives ample room.

**Q9: Why is there only one NAT Gateway?**
Cost optimization. A NAT Gateway costs ~$0.045/hour ($32/month) plus data transfer. Multi-AZ HA would require one NAT per AZ (~$64/month). For a training/evaluation project, single NAT is sufficient. Production recommendation: one NAT per AZ to avoid cross-AZ traffic charges and AZ failure blast radius.

**Q10: What is CloudTrail and what does it NOT capture?**
CloudTrail records management events (API calls to AWS APIs — who created a security group, who launched an EC2 instance). It does NOT capture: application-level logs (Spring Boot logs), database query logs, SSH sessions (use Systems Manager Session Manager logging for that), network traffic content (use VPC Flow Logs for IP-level, WAF logs for HTTP content).

**Q11: What is AWS Config and how does it differ from CloudTrail?**
CloudTrail = what happened (event log). AWS Config = what exists NOW and how it changed over time (configuration history). Config evaluates compliance rules: "All S3 buckets must have versioning enabled." If someone disables versioning, Config marks the resource as NON_COMPLIANT and can trigger automatic remediation. CloudTrail tells you who disabled it; Config tells you it's currently disabled.

**Q12: How does the ALB health check work?**
ALB health check configured via Ingress annotation: `alb.ingress.kubernetes.io/healthcheck-path: /actuator/health`, interval 30s, healthy threshold 2, unhealthy threshold 3. The ALB sends GET /actuator/health to each registered pod every 30s. Spring Boot Actuator returns `{"status":"UP"}`. If a pod returns non-2xx 3 consecutive times, it's marked unhealthy and removed from routing.

**Q13: What is a KMS CMK (Customer Managed Key) vs AWS Managed Key?**
AWS Managed Key: auto-created by AWS services, you can see it but can't manage rotation schedule or key policy. Customer Managed Key (CMK): you create, you define the key policy, you control rotation. FleetOps creates CMKs for all sensitive resources (`modules/kms`) with `enable_key_rotation = true`. CMKs allow: restricting who can use/manage the key, cross-account sharing, audit via CloudTrail (every decrypt logged).

**Q14: What happens if the RDS instance runs out of connections?**
HikariCP pool gets exhausted → new connection requests wait for `connectionTimeout` (default 30s) → if no connection freed, throws `SQLTransientConnectionException`. Spring Boot returns 500. Mitigation: tune pool size (FleetOps reduced from 10 to ~5-7 per pod), use RDS Proxy (connection pooling at the RDS level — scales connections across many application instances), or use Aurora Serverless (scales connections independently).

**Q15: Why is ElastiCache Redis used instead of a local Spring Boot cache (Caffeine)?**
Local in-memory cache (Caffeine) is per-pod. With 2+ replicas of vehicle-service, each pod has its own cache. Pod A caches fleet data. Pod B has a cache miss (cold start). A new vehicle is added — Pod A evicts its cache, but Pod B's cache is stale for up to 5 minutes. Redis is shared across all pods — one cache invalidation propagates to all pod reads instantly.

---

## Top 50 Kubernetes Questions

**Q16: What is the Kubernetes control plane and who manages it in EKS?**
Control plane = kube-apiserver + etcd + scheduler + controller-manager. In EKS, AWS manages these components — they run in AWS-owned infrastructure, not on your nodes. You manage worker nodes (node groups). AWS provides a 99.95% SLA for the EKS control plane. You're responsible for the worker nodes (OS patches, instance health, node group scaling).

**Q17: What is etcd and what does it store?**
etcd is a distributed key-value store. Kubernetes stores ALL cluster state in etcd: pod specs, Deployments, Services, Secrets, ConfigMaps, ExternalSecrets (CRD instances), ArgoCD Application resources, RBAC rules, Node registrations. If etcd is lost without backup, the cluster state is gone (pods keep running, but the API has no knowledge of them). EKS automatically backs up etcd.

**Q18: What happens when you run `kubectl apply -f deployment.yaml`?**
1. kubectl serializes YAML to JSON, sends to kube-apiserver via HTTPS
2. kube-apiserver authenticates (your IAM user → `aws eks get-token` → STS token → K8s auth)
3. kube-apiserver authorizes (RBAC: does your user have `create/update` on deployments in this namespace?)
4. kube-apiserver validates the object (admission controllers: ValidatingWebhookConfiguration from ALB controller)
5. kube-apiserver writes to etcd
6. Deployment controller (in controller-manager) watches etcd, sees new/updated Deployment, reconciles ReplicaSets
7. Scheduler watches for unscheduled pods, assigns them to nodes
8. kubelet on assigned nodes pulls images and starts containers

**Q19: What is a DaemonSet and does FleetOps use one?**
DaemonSet ensures exactly one pod runs on every node. FleetOps uses DaemonSets implicitly: the CloudWatch Observability addon installs a Fluent Bit DaemonSet that ships container logs from every node to CloudWatch. The VPC CNI plugin is also a DaemonSet (runs on every node to manage pod networking).

**Q20: How does the Cluster Autoscaler decide to add a node?**
When a pod is in `Pending` state because no existing node has sufficient CPU/memory, the scheduler marks it unschedulable. Cluster Autoscaler watches for unschedulable pods, simulates which node group can accommodate them, calls AWS Auto Scaling API to increase the desired count. A new EC2 instance joins the node group, kubelet registers with the cluster, the scheduler places the pending pod.

**Q21: What is a CRD (Custom Resource Definition)?**
CRDs extend the Kubernetes API with custom object types. `ExternalSecret`, `ClusterSecretStore`, `Application` (ArgoCD) are all CRDs. Once a CRD is installed (via Helm for ESO, via ArgoCD's own Helm chart), you can use `kubectl get externalsecrets` just like `kubectl get pods`. The CRD controller watches these custom objects and acts on them.

**Q22: What would happen if the metrics-server pod crashed?**
HPA stops receiving CPU metrics → HPA logs errors but continues at current replica count (doesn't scale to 0). After metrics-server restarts, HPA resumes normal scaling. This is a graceful degradation — loss of metrics-server doesn't stop pods from running, it stops autoscaling decisions.

**Q23: How does Kubernetes Secret base64 encoding differ from encryption?**
Base64 is encoding, not encryption — anyone with `kubectl get secret -o yaml` can decode it with `base64 -d`. In EKS, etcd is encrypted with KMS (`kms_secrets_key_arn`), so the raw bytes on disk are encrypted. Access control is via RBAC — restricting who can `get` secrets in the namespace. The security comes from KMS encryption at rest + RBAC access control, not from base64.

**Q24: What is `PrunePropagationPolicy=foreground` in ArgoCD syncOptions?**
When ArgoCD prunes a resource (removes it because it's no longer in Git), foreground propagation means the parent resource waits for all dependent resources to be deleted before the parent is deleted. Default is background (parent deleted immediately, children garbage collected asynchronously). Foreground is safer for resources with dependents (e.g., a Deployment deletion waits for pods to terminate).

**Q25: How would you roll back a bad deployment?**
Option 1 (GitOps preferred): `git revert` the commit that changed the image tag → CD pipeline updates `values-prod.yaml` back to old tag → ArgoCD syncs → rolling update to old image.
Option 2 (immediate): `kubectl rollout undo deployment/fleetops-auth-service` — reverts to previous ReplicaSet. But ArgoCD selfHeal will detect drift from Git state and roll it forward again within 3 minutes. For lasting rollback, fix it in Git.

---

## Top 50 Terraform Questions

**Q26: What is a Terraform provider and why are there three providers in this project?**
A provider is a plugin that translates Terraform configuration into API calls. Three providers:
- `aws` — provisions AWS resources (VPC, EKS, RDS, etc.)
- `helm` — installs Helm charts into Kubernetes (ArgoCD, ESO, ALB controller, etc.)
- `kubernetes` — manages K8s resources directly (Secrets, Namespaces, Ingresses via `kubernetes_*` resources)

`helm` and `kubernetes` providers both use the EKS cluster endpoint — configured with `exec` blocks that call `aws eks get-token` for authentication.

**Q27: What is the purpose of `bootstrap/main.tf`?**
It creates the prerequisites for all other Terraform state:
- S3 bucket: stores the `terraform.tfstate` files for all environments
- DynamoDB table: provides state locking
- ECR repositories: stores Docker images for all services

Bootstrap uses local state (state stored locally, not in S3) because S3 doesn't exist yet when bootstrap runs. It's run once manually. After bootstrap, all subsequent `terraform init` in `environments/*/` uses the S3 backend.

**Q28: Explain `terraform import` and when you'd use it.**
`terraform import` brings an existing AWS resource under Terraform management without recreating it. Use case: a security group was manually created before Terraform was adopted. Run `terraform import aws_security_group.my_sg sg-12345` — Terraform writes the resource to state. Next `plan` shows the desired config vs. actual; `apply` reconciles any differences. In FleetOps, import was not needed (greenfield project, everything created by Terraform from the start).

**Q29: What is `terraform workspace` and is it used here?**
Workspaces allow multiple state files in the same configuration directory (e.g., dev/staging/prod in one directory with one set of .tf files). FleetOps uses separate directories (`environments/dev/`, `environments/prod/`) instead of workspaces — clearer separation, each environment's main.tf can differ. Workspaces are simpler but lead to "workspace sprawl" if configuration between envs diverges significantly.

**Q30: Why does `module.eks_addons` have `depends_on = [module.eks_nodegroup, module.secrets_manager]`?**
Two reasons:
1. `module.eks_nodegroup`: Helm charts deploy pods. If addons run before worker nodes exist, pods stay Pending forever. Terraform's implicit dependency detection can't infer this because no addons module input directly references a nodegroup attribute.
2. `module.secrets_manager`: The ArgoCD Helm install reads the GitHub PAT from Secrets Manager (`kubernetes_secret.argocd_repo` resource reads it). If secrets_manager hasn't run yet, the PAT secret doesn't exist and the read fails.

---

## Top 50 CI/CD Questions

**Q31: What is the difference between CI and CD?**
CI (Continuous Integration): automated build, test, and quality check on every code push. Ensures code is always in a releasable state. CD (Continuous Delivery/Deployment): automated deployment of validated artifacts to environments. In FleetOps: CI = java-ci-ecr.yml (test→scan→build→push). CD = cd-ecr.yml (update Helm values) + ArgoCD (sync to cluster).

**Q32: Why does the CD workflow use a GitHub App token instead of a PAT?**
GitHub Apps have:
- Fine-grained permissions (only write to `fleetops-deployments` repo, nothing else)
- Short-lived tokens (1 hour expiry) — automatically generated per job run
- Machine identity (not tied to a human user whose account could be deactivated)
- Audit trail shows "github-actions[bot]" not a human username
PATs are tied to a human user, have broad scopes, don't expire by default, and create a human single point of failure.

**Q33: What is Snyk and what does it scan?**
Snyk is a dependency vulnerability scanner. In FleetOps, it scans `pom.xml` for Maven dependencies with known CVEs (using the Snyk vulnerability database). `--severity-threshold=high` means only HIGH and CRITICAL findings fail the build. It also monitors for newly published CVEs — a dependency that was safe today may get a CVE tomorrow, surfaced on next build.

**Q34: What is SonarQube quality gate?**
A quality gate is a pass/fail threshold on code quality metrics: code coverage %, new bugs, new security hotspots, code duplications, maintainability rating. In FleetOps, the quality gate is polled after the analysis task completes. If status is `ERROR` (gate failed), the CI workflow exits with code 1, blocking the merge/push. `OK` or `WARN` allows the pipeline to continue.

**Q35: How does semantic versioning (SemVer) work in the CI pipeline?**
The `prepare` job inspects commit messages since the last `v*` git tag:
- `BREAKING CHANGE` anywhere in message, or `!:` suffix → bump major (1.x.x → 2.0.0)
- `feat:` prefix → bump minor (1.2.x → 1.3.0)
- `fix:` prefix → bump patch (1.2.3 → 1.2.4)
- No conventional commit → no version bump, no GitHub Release, tag stays `main-{SHA:7}`
This follows the Conventional Commits specification. Version bumps are fully automatic.

---

## Top 50 Security Questions

**Q36: What is OIDC and how is it used in two different places in this project?**
OIDC (OpenID Connect) is used in two distinct contexts:
1. **IRSA:** EKS acts as an OIDC provider. Pods get OIDC tokens (projected service account tokens). AWS IAM trusts these tokens to grant pods temporary AWS credentials (AssumeRoleWithWebIdentity).
2. **GitHub Actions OIDC:** GitHub Acts as an OIDC provider. GitHub Actions workflows get OIDC tokens. AWS IAM trusts GitHub's OIDC provider to allow workflows to assume IAM roles (no static AWS credentials in GitHub Secrets).
Same protocol, two different identity providers for two different use cases.

**Q37: What is a security group and how is it different from a Network Policy?**
Security Group: AWS-level network filter attached to ENIs (elastic network interfaces). Stateful — if you allow inbound port 5432, the return traffic is automatically allowed. Operates on IP + port at the EC2/ENI level.
Network Policy: Kubernetes-level network filter. Controls pod-to-pod and pod-to-external traffic using label selectors. Requires a CNI plugin that supports NetworkPolicy (VPC CNI does with `NetworkPolicy` addon, or Calico). Both work together: SG controls traffic at the EC2 level, NetworkPolicy controls it at the pod level.

**Q38: What is WAF rate limiting and how is it configured?**
WAF rate limiting blocks or counts requests from a source IP that exceed a threshold within a 5-minute window. In `modules/waf`, a rate-based rule (type: RATE_BASED) is configured with an aggregate key type (IP) and limit. If an IP sends more than `limit` requests in 5 minutes, WAF blocks subsequent requests with 403. This mitigates brute-force login attempts and credential stuffing attacks.

**Q39: What are Checkov's findings on this project and how are they handled?**
Several Checkov rules are suppressed with justifications:
- `CKV_AWS_260` on ALB SG ingress rule: "0.0.0.0/0 source" — suppressed because the actual source is scoped to the ALB SG via `referenced_security_group_id`, not a CIDR. Checkov can't detect SG-to-SG rules as scoped.
- `CKV_AWS_288/289/290/355` on ALB controller IAM policy: ALB provisioning requires broad ELB and EC2 describe permissions as per AWS's official recommended policy. Suppressed with documentation.
The right approach is: never suppress without justification, document every suppression.

**Q40: What is the security risk of `jwt_secret` stored in Terraform state?**
Terraform state is a JSON file that stores all resource attributes in plaintext. `fleetops/prod/jwt.jwt_secret` value appears in the state file as a string. The S3 state bucket has KMS encryption — the file is encrypted at rest. But anyone with `s3:GetObject` on the state bucket + KMS decrypt permission can read the JWT secret. This is in the security backlog. Mitigation: use `sensitive = true` on the variable, and consider rotating the JWT secret out-of-band (not via Terraform) — store only the ARN in Terraform, not the value.

---

## Top 50 Project-Specific Questions

**Q41: Why does the frontend use Nginx instead of Node.js to serve files in production?**
The React app is a SPA — after `npm run build`, it's static HTML/JS/CSS. Node.js is only needed for the build step. Nginx is a battle-tested static file server with better performance, smaller footprint, and simpler configuration for serving SPA with proper cache headers. The Dockerfile uses a multi-stage build: Node 20 (build stage) → Nginx Alpine (runtime). The final image contains no Node.js.

**Q42: What is the `fleetops-app` ServiceAccount and why do all services share it?**
`fleetops-app` is a single Kubernetes ServiceAccount in `fleetops-prod` namespace annotated with the IRSA role ARN. All four backend services use it. This means all services share the same IRSA role and its permissions. The more secure design would be one ServiceAccount per service, each with only the permissions it needs (auth-service doesn't need Bedrock; maintenance-service doesn't need Step Functions). Consolidated into one for simplicity at this project stage.

**Q43: Why is the Git repository for deployments separate from the application code?**
GitOps separation of concerns:
1. Application repo: tracks business logic changes (commit = code change, not deployment)
2. Deployments repo: tracks desired cluster state (commit = deployment, not code)
ArgoCD watches only the deployments repo. A code commit in auth-service doesn't trigger ArgoCD directly — the CD pipeline commits the new image tag to the deployments repo, which ArgoCD detects. This means: you can deploy without code changes (configuration update), roll back by reverting the deployments repo commit, and have a clear deployment history separate from code history.

**Q44: How would you add a new microservice to this platform?**
1. Create service repo with code + Dockerfile + `.github/workflows/ci.yml` + `cd.yml` (calling reusable workflows)
2. Create ECR repository (add to bootstrap/main.tf, run `terraform apply`)
3. Create Helm chart in `fleetops-deployments/charts/new-service/` (copy from auth-service chart template)
4. Create ArgoCD Application in `fleetops-deployments/argocd/apps/prod/new-service-prod.yaml` with appropriate sync wave
5. Add Ingress rule for the new service's paths in `k8s/prod/apps/ingress.yaml`
6. Configure ExternalSecrets if the service needs DB credentials (add to `external-secrets.yaml`)
7. Add GitHub secrets to the new repo (AWS_ECR_ROLE_ARN, SONAR_TOKEN, etc.)

**Q45: What is the `db-init` Job in ArgoCD sync wave 1?**
A Kubernetes Job runs once, completes, and is not restarted. The `db-init` Job creates the application databases (auth_db, vehicle_db, maintenance_db, request_db) in RDS if they don't exist. It runs at wave 1 — before the application pods (wave 3-5). Without this Job, Flyway migrations would fail because the databases don't exist yet. The Job runs a PostgreSQL client and executes `CREATE DATABASE IF NOT EXISTS` for each database.

**Q46: Why does ArgoCD use `allowEmpty: false` on app sync?**
`allowEmpty: false` prevents ArgoCD from applying an empty set of resources (which would delete all currently deployed resources). This is a safety guard — if the Helm chart renders to empty YAML (e.g., values file is incorrect), ArgoCD won't apply the empty state and inadvertently destroy the deployment.

**Q47: What is the `infra-values.yaml` file and how is it populated?**
`environments/prod/infra-values.yaml` is generated by the `terraform-apply.yml` workflow after `terraform apply` completes. It contains Terraform output values needed by the Helm charts:
```yaml
acmCertificateArn: arn:aws:acm:us-east-1:538661800892:certificate/XXX
albSecurityGroupId: sg-XXXXX
efsFileSystemId: fs-XXXXX
stepFunctionsArn: arn:aws:states:us-east-1:538661800892:stateMachine:fleetops-prod
```
ArgoCD Helm applications reference this file as an additional values file. This bridges Terraform outputs (infrastructure facts) into Kubernetes deployments.

---

## Weak Areas, Tradeoffs, and Improvements

### Known Limitations

**1. Single shared ServiceAccount (IRSA):**
All four services share `fleetops-app` SA. Auth-service has unnecessary Bedrock permissions; frontend has no AWS access at all. Fix: one SA per service with scoped permissions.

**2. JWT in localStorage (Security Backlog):**
Frontend stores JWT in localStorage — vulnerable to XSS attacks. A malicious script on the page can `localStorage.getItem('token')`. Fix: store in httpOnly cookie (not accessible to JavaScript). Requires backend Set-Cookie header changes.

**3. Terraform state contains plaintext secrets:**
`db_password`, `jwt_secret` values appear in the S3 state file. Fix: use random_password resource managed outside Terraform and stored only in Secrets Manager; Terraform references the ARN, not the value.

**4. Single NAT Gateway:**
One NAT Gateway in one AZ. If that AZ has issues, outbound internet access from all private subnets fails (pods can't call AWS APIs). Fix: one NAT per AZ (~$32/month extra).

**5. Single-AZ RDS:**
`db.t3.micro` single-AZ. If the AZ fails, RDS is unavailable. Fix: Multi-AZ RDS (synchronous standby in another AZ, automatic failover in ~60s). Cost: 2x instance cost.

**6. No Resilience4j circuit breakers:**
Synchronous service-to-service calls (maintenance → vehicle, request → vehicle/auth) have no retry or circuit breaker. If vehicle-service is slow, maintenance-service requests pile up. Fix: add Resilience4j with `@CircuitBreaker` and `@Retry` annotations.

**7. Redis single-node:**
`cache.t3.micro` single-node. No Multi-AZ. Cache loss = cache miss storm (all requests hit DB). Fix: Redis cluster mode or Multi-AZ replication group.

### Architectural Tradeoffs

**GitOps vs. direct kubectl:** GitOps is more auditable and declarative but adds ~3-minute sync latency. Direct kubectl is faster but untracked. Project chose GitOps for correctness over speed.

**Helm vs. raw manifests:** Both are used (Helm for services, raw manifests for platform setup). Helm provides templating and values files for multi-env config. Raw manifests are simpler for one-off platform resources.

**Bedrock vs. OpenAI:** Bedrock stays in-account (data privacy), but has fewer model options and higher latency than OpenAI. Cost is comparable at low volumes.

**EFS vs. S3 for media:** EFS behaves as a filesystem (POSIX), which is what the Spring Boot file upload code expects. S3 would require rewriting to object storage semantics. EFS costs more (~$0.30/GB/month vs S3 ~$0.023/GB) but required no code changes.

### Cost Optimizations Already Made
- `m7i-flex.large` nodes (~10% cheaper than standard m7i)
- `db.t3.micro` + `cache.t3.micro` (smallest instances)
- Single NAT Gateway (vs. one per AZ)
- Single-AZ RDS (vs. Multi-AZ)
- Shared ALB via `group.name` (vs. one ALB per service)
- SSM for config values (free) vs. Secrets Manager ($0.40/secret)
- ECR pull-through cache (avoid public registry rate limits)
- Nova Lite model for Bedrock (cheapest Amazon model)
- Redis caching reduces RDS query load
- CloudFront caches static assets (reduces ALB + EKS compute load)

### Future Improvements
- Implement RS256 JWT (asymmetric) for better secret hygiene
- Add Resilience4j circuit breakers on inter-service calls
- Implement distributed tracing (AWS X-Ray or OpenTelemetry → Jaeger)
- Add Karpenter for more intelligent node provisioning (vs. Cluster Autoscaler)
- Implement Vertical Pod Autoscaler (VPA) for right-sizing resource requests
- Add Redis Sentinel or Cluster Mode for HA cache
- Implement RDS Multi-AZ for production durability
- Per-service ServiceAccounts with minimal IRSA permissions
- Move JWT to httpOnly cookies (XSS protection)
- Implement Terraform secret encryption (avoid plaintext in state)
- Add SQS dead letter queues for SNS delivery failures
- Implement API rate limiting at the application layer (Spring Boot + Redis token bucket)
- Set up PodDisruptionBudgets to ensure availability during node drains

---

# SUPPLEMENT â€” INTERNALS, MISSING COMPONENTS & COMPLETE FLOWS

> This supplement adds every section missing from the main guide. Read after Phase 2.

---

## PHASE 2 SUPPLEMENT â€” INTERNALS FOR EVERY COMPONENT

### Route53 â€” INTERNALS

**How a DNS query resolves step by step:**
1. User types `fleetops.website` â€” browser checks local DNS cache (TTL-gated)
2. OS stub resolver sends UDP query to the recursive resolver (usually ISP or 8.8.8.8)
3. Recursive resolver checks its cache. On miss:
   - Queries root nameservers (`.`) â†’ returns TLD nameservers for `.website`
   - Queries `.website` TLD nameservers â†’ returns Route53's 4 nameservers for `fleetops.website`
   - Queries Route53's nameservers â†’ returns the A record (Alias â†’ CloudFront IP)
4. Route53 Alias record: resolves CloudFront's IP at AWS-internal resolution time (no separate DNS TTL for Alias records â€” TTL is always 60s for Alias)
5. Recursive resolver caches the result and returns to the client
6. Browser connects to the resolved IP

**Why Alias records cost nothing for Route53 queries to AWS resources:** Alias records targeting CloudFront, ALB, or other AWS services are free â€” Route53 does not charge for Alias queries to AWS endpoints. Standard A record queries are charged at $0.40 per million.

**Health check integration:** Route53 can do its own health checks on endpoints and do DNS failover. In FleetOps, this is not configured â€” CloudFront and ALB handle health checking at their own layers.

---

### CloudFront â€” INTERNALS

**How a cache hit/miss works:**
```
Request arrives at edge PoP
  â†“
CloudFront checks the cache key: URI path + (optionally) query strings + headers
  â†“
Cache HIT: CloudFront returns the cached response from SSD storage at edge PoP
            Time: 1-5ms. Origin never contacted.
  â†“
Cache MISS: CloudFront opens a persistent TCP connection to the origin (origin.fleetops.website)
            If connection already exists (connection reuse), adds request to the pipeline
            CloudFront fetches the response, stores it with Cache-Control headers respected
            Returns to client. Subsequent requests for same cache key = HIT
```

**Cache key for the React SPA:**
- `fleetops.website/` â†’ `index.html` â€” cached with `Cache-Control: no-cache` (always revalidated â€” ensures users get new index.html on deployment)
- `fleetops.website/assets/main-AbCdEf.js` â†’ hashed bundle â€” cached with `max-age=31536000` (1 year, because the hash in filename changes on every build)

**Edge PoP â†’ Origin connection:** CloudFront maintains warm TCP (and HTTP/2) connections to the origin ALB. When a miss occurs, CloudFront does NOT do a new TCP handshake per request â€” it uses a persistent connection pool. This means origin latency is only the HTTP round-trip, not TCP setup.

**WAF evaluation order at CloudFront:**
```
Request â†’ IP reputation check â†’ Rate-based rule check â†’ AWS Managed Rules (SQLi, XSS) â†’ Allow/Block
```
WAF rules are evaluated in priority order (lower number = higher priority). First matching rule's action applies. If no rule matches â†’ default action (allow).

**SCALING:** CloudFront scales automatically. It's a global anycast network â€” traffic is absorbed by whichever PoP is closest to the user. No capacity planning needed. AWS manages the underlying infrastructure.

**COST:** ~$0.0085 per 10,000 HTTPS requests + $0.009/GB data transfer (first 10 TB/month to internet). Static asset caching dramatically reduces origin data transfer cost.

---

### WAF â€” INTERNALS

**Rule evaluation engine:**
Each WAF Web ACL has an ordered list of rules. For each incoming request:
```
For each rule in priority order:
  1. Match the request against the rule's statement
     - AWSManagedRulesCommonRuleSet: checks 10 OWASP categories
       Each rule checks specific request components (headers, URI, body, query string)
     - Rate-based: counts requests per IP in a 5-min sliding window
     - Custom: user-defined patterns
  2. If rule matches AND action is BLOCK â†’ return 403, stop evaluation
  3. If rule matches AND action is COUNT â†’ increment counter, continue
  4. If rule matches AND action is ALLOW â†’ immediately allow, stop evaluation
  5. If no rule matches â†’ apply default action (usually ALLOW)
```

**AWSManagedRulesCommonRuleSet contents (key rules):**
- `NoUserAgent_HEADER`: blocks requests missing User-Agent header (typical bot behavior)
- `SizeRestrictions`: blocks requests > 8KB body (SQL injection attempts tend to be large)
- `SQLi_QUERYARGUMENTS`: checks for SQL injection patterns in query strings
- `CrossSiteScripting_BODY`: checks for `<script>` patterns in request body

**Rate-based rule mechanics:**
AWS WAF maintains a distributed counter per IP address. The window is a 5-minute sliding window. When the counter exceeds the threshold, the IP is automatically added to a temporary blocklist. The block persists until the request rate drops below the threshold.

**SCALING:** WAF is a fully managed service â€” scales with CloudFront automatically.

**COST:** $5/month per Web ACL + $1/month per rule + $0.60/million requests evaluated. The AWSManagedRulesCommonRuleSet is free.

---

### ALB â€” INTERNALS (AWS Load Balancer Controller Reconciliation Loop)

**How the LBC controller works internally:**
```
LBC controller pod (in kube-system) starts
  â†“
Registers watch on Kubernetes API for:
  - Ingress resources (networking.k8s.io/v1)
  - Services (core/v1)
  - Endpoints (core/v1)
  - Nodes (core/v1)
  â†“
Event: Ingress fleetops-ingress created in fleetops-prod
  â†“
LBC reads Ingress annotations â†’ builds desired ALB state:
  - scheme: internet-facing
  - subnets: public subnets from VPC (auto-discovered via subnet tags kubernetes.io/role/elb=1)
  - security groups: from annotation
  - listeners: [{port:80,protocol:HTTP,redirectâ†’443}, {port:443,protocol:HTTPS,cert:ACM_ARN}]
  - target groups: one per {service:port} combination
  - listener rules: path-based rules â†’ target groups
  â†“
LBC calls AWS API: CreateLoadBalancer, CreateTargetGroup, CreateListener, CreateRule
  â†“
ALB created â†’ LBC writes ALB DNS name back to Ingress.status.loadBalancer.ingress[0].hostname
  â†“
When a pod becomes Ready:
  Endpoint controller updates the Endpoints object for the Service
  LBC watches Endpoints â†’ calls RegisterTargets(pod_ip, port) on the target group
  â†“
When a pod is deleted/not-ready:
  LBC calls DeregisterTargets
  Target group draining: ALB stops sending new requests to that pod IP
  Existing connections finish (deregistration_delay, default 300s)
```

**Why subnet auto-discovery uses tags:**
LBC looks for subnets tagged `kubernetes.io/cluster/{cluster-name}=owned` and `kubernetes.io/role/elb=1` (public) or `kubernetes.io/role/internal-elb=1` (private). These tags are added by the EKS cluster module in Terraform. This is why the annotation `alb.ingress.kubernetes.io/subnets` is not explicitly set â€” LBC discovers them via tags.

**Target group health check:**
ALB sends `GET /actuator/health` every 30s to each registered pod IP. The pod must return HTTP 200. Spring Boot Actuator provides this endpoint automatically â€” it checks DB connection, disk space, and custom health indicators.

**SCALING:** The ALB scales automatically â€” it adds/removes load balancer nodes (EC2 instances behind the ALB DNS name) as traffic increases. This is fully transparent. AWS manages it. No configuration needed.

**COST:** $0.018/hour ($~13/month) + $0.008/LCU-hour (Load Balancer Capacity Unit â€” scales with throughput). Sharing one ALB via `group.name: fleetops` saves one ALB cost vs. two separate ALBs.

---

### VPC â€” CNI INTERNALS (How Pod IPs are Assigned)

**VPC CNI (Amazon VPC Container Network Interface)** is a DaemonSet running on every EKS node. It gives every pod a real VPC IP address from the node's subnet.

**How pod IP allocation works:**
```
Node starts, VPC CNI DaemonSet pod starts
  â†“
CNI plugin calls EC2 API: CreateNetworkInterface
  Assigns a secondary ENI (Elastic Network Interface) to the EC2 node
  Each secondary ENI can hold N secondary private IPs (depends on instance type)
  m7i-flex.large: up to 3 ENIs, 10 secondary IPs per ENI = 30 pod IPs per node
  â†“
CNI maintains a warm IP pool (pre-allocated IPs ready for pods)
  â†“
Pod scheduled to node:
  Kernel creates a network namespace (netns) for the pod
  CNI plugin takes one IP from the warm pool
  Creates a veth pair: one end in pod netns (eth0), one end in node netns (eniX)
  Routes pod's IP through the ENI's IP slot
  Pod eth0 gets the VPC IP â†’ pod has a real, routable VPC IP address
  â†“
ALB can route directly to pod IP (no NAT) because the pod IP is in the VPC CIDR
```

**Why this matters for ALB target-type: ip:** Because pod IPs are real VPC IPs, the ALB can route directly to them without going through a NodePort on the EC2 instance. This is only possible with VPC CNI â€” it does NOT work with Calico or Flannel (which use overlay networks with NAT).

---

### EKS â€” INTERNALS

**kube-apiserver internal flow for a kubectl apply:**
```
kubectl serializes YAML â†’ JSON â†’ sends HTTPS POST to api.eks.us-east-1.amazonaws.com
  â†“
Authentication: kube-apiserver calls aws-iam-authenticator
  aws eks get-token returns a pre-signed STS URL
  kube-apiserver fetches the URL â†’ STS returns TokenReview (your IAM identity)
  IAM identity mapped to K8s user/group via aws-auth ConfigMap
  â†“
Authorization: RBAC check
  Does this K8s user have the `create` verb on `deployments` in `fleetops-prod` namespace?
  Check ClusterRoleBinding/RoleBinding rules
  â†“
Admission controllers run (in order):
  1. Mutating: PodIdentityWebhook injects IRSA env vars + volume
  2. Validating: ALB controller's ValidatingWebhookConfiguration checks Ingress fields
  3. Others: LimitRanger, ResourceQuota, etc.
  â†“
Object written to etcd (base64-encoded, KMS-encrypted)
  â†“
etcd watch notification â†’ controller-manager sees new/updated resource
  Deployment controller creates/updates ReplicaSet
  ReplicaSet controller creates Pods (writes pod spec to etcd with node = "")
  â†“
kube-scheduler watch: sees Pod with no node assigned
  Scores nodes: resource availability, affinity, taints/tolerations
  Writes node assignment to pod spec in etcd
  â†“
kubelet watch on assigned node: sees pod spec
  Pulls image (or uses cached), creates containers, mounts volumes, sets env vars
  Starts containers, begins probing
```

**etcd write guarantee:** etcd uses Raft consensus. In EKS, AWS runs a 3-node etcd cluster across 3 AZs. A write is committed only when 2 out of 3 nodes acknowledge it (quorum = (3/2)+1 = 2). This means etcd can tolerate one AZ failure without data loss.

**How HPA works internally:**
```
metrics-server aggregates CPU/memory from kubelet every 15s
  â†“
HPA controller (in kube-controller-manager) queries metrics-server every 15s
  GET /apis/metrics.k8s.io/v1beta1/namespaces/fleetops-prod/pods
  â†’ returns current CPU per pod
  â†“
HPA calculates: desiredReplicas = ceil(currentReplicas Ã— (currentCPU / targetCPU))
  Example: 2 pods, 90% avg CPU, target 70%: ceil(2 Ã— 90/70) = ceil(2.57) = 3 replicas
  â†“
HPA writes new replica count to the Deployment spec
  â†“
Deployment controller creates/deletes pods to match
  â†“
Scale-down stabilization: HPA will not scale down for 5 minutes after last scale-up
  (prevents thrashing: scale up at load spike, don't immediately scale down)
```

---

### RDS â€” INTERNALS

**HikariCP Connection Pool (per pod):**
```
Spring Boot starts:
  HikariCP creates a pool with minimumIdle connections (usually 2-5)
  Each connection: TCP handshake â†’ PostgreSQL auth (MD5 or SCRAM) â†’ set session params
  Pool ready
  â†“
Request comes in (e.g., SELECT * FROM vehicles):
  Application calls dataSource.getConnection()
  HikariCP: is there an idle connection in the pool?
    YES: return it immediately (microseconds)
    NO: wait up to connectionTimeout (30s) for one to become available
       If none available after timeout: HikariConnectionTimeoutException â†’ 503
  â†“
Application executes SQL via JDBC driver
  JDBC driver serializes the query as PostgreSQL wire protocol messages
  TCP â†’ RDS security group â†’ RDS endpoint â†’ PostgreSQL process â†’ execute â†’ return result
  â†“
Application calls connection.close() â†’ returns to pool (not actually closed)
```

**Why HikariCP over other pools (C3P0, DBCP):** HikariCP is the fastest Java connection pool â€” it uses lock-free data structures for connection acquisition. Spring Boot auto-configures it as the default since Spring Boot 2.0. Zero-overhead connection health checks (uses TCP keepalive rather than test queries).

**Max connections on db.t3.micro:**
- Formula: `LEAST({DBInstanceClassMemory/9531392}, 5000)`
- db.t3.micro: 1GB RAM â†’ ~85 max connections
- With 4 services Ã— 2 pods Ã— 5 connections = 40 connections used, leaving 45 headroom
- The `db-init` Job, Lambda, and Flyway migrations also briefly use connections

**Flyway internals:**
```
Spring Boot startup â†’ FlywayAutoConfiguration runs
  â†“
Flyway connects to auth_db (using same datasource credentials)
  â†“
Checks for flyway_schema_history table
  If absent: CREATE TABLE flyway_schema_history (...)
  â†“
Reads all V*.sql files from classpath:/db/migration/
  Sorts by version number (V1 < V2 < V3 ...)
  â†“
For each migration not in flyway_schema_history:
  BEGIN TRANSACTION
  Execute the SQL statements in the migration file
  INSERT INTO flyway_schema_history (version, description, checksum, ...) VALUES (...)
  COMMIT
  â†“
If any migration fails: ROLLBACK, Flyway throws FlywayException
  Spring Boot startup fails â†’ pod never becomes Ready
  Old pod continues serving traffic (no bad schema deployed)
  â†“
If all migrations succeed: Spring context continues loading
```

**Checksum validation:** Flyway stores a CRC32 checksum of each applied migration script. If a previously applied script is modified (common mistake), Flyway detects the checksum mismatch and FAILS startup with `FlywayException: Validate failed`. This prevents silent schema corruption.

---

### Redis â€” INTERNALS

**How Spring Cache abstraction maps to Redis:**
```java
@Cacheable(value = "vehicles", key = "'fleet:all'")
public List<Vehicle> getAllVehicles() { ... }
```

Spring's caching abstraction translates this to Redis commands:
```
On method call:
  REDIS GET "vehicles::fleet:all"
  â†’ NULL (miss): execute method, then:
    REDIS SET "vehicles::fleet:all" <serialized_bytes> PX 300000
    (PX = milliseconds expiry)
  â†’ value (hit): deserialize and return without calling method

@CacheEvict(allEntries=true):
  REDIS SCAN cursor MATCH "vehicles::*"  (finds all keys in the "vehicles" cache)
  REDIS DEL vehicles::fleet:all vehicles::ai-analysis:V-001 ...
```

**Serialization:** Spring uses JdkSerializationRedisSerializer by default (Java serialization). Better practice: Jackson2JsonRedisSerializer (stores as JSON â€” human-readable, compatible with other clients). The key is a String; the value is binary.

**Redis memory management:**
- `maxmemory-policy: allkeys-lru` (default if not configured) â€” when Redis is full, evict the least-recently-used key
- ElastiCache `cache.t3.micro`: 512 MB RAM
- Each cached vehicle list entry: ~10KB for 100 vehicles
- AI analysis entries: ~2KB each
- No risk of memory exhaustion for this fleet size

**Redis Persistence:** ElastiCache Redis uses RDB (snapshot) persistence by default. On restart, Redis loads the last snapshot. For a cache use case, this is acceptable â€” a cold cache after restart just means more DB queries for 5 minutes.

**Connection:** Spring Boot uses Lettuce (non-blocking Netty-based Redis client). One connection pool per application instance. Connection to ElastiCache uses TLS (SSL enabled in ElastiCache cluster).

---

### S3 â€” INTERNALS

**How presigned URL works cryptographically:**
```
vehicle-service calls S3 SDK generatePresignedUrl():
  SDK creates a canonical request:
    Method: PUT
    Bucket: fleetops-prod-vehicle-docs
    Key: vehicles/V-045/insurance.pdf
    Content-Type: application/pdf
    X-Amz-Date: 20260622T090000Z
    X-Amz-Expires: 900 (15 minutes)
    X-Amz-Credential: {AccessKey}/20260622/us-east-1/s3/aws4_request
  â†“
  SDK signs with SigV4: HMAC-SHA256 of canonical request using secret key
  â†’ produces X-Amz-Signature value
  â†“
Returns URL: https://fleetops-prod-vehicle-docs.s3.amazonaws.com/vehicles/V-045/insurance.pdf
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20260622%2Fus-east-1%2Fs3%2Faws4_request
  &X-Amz-Date=20260622T090000Z
  &X-Amz-Expires=900
  &X-Amz-SignedHeaders=content-type%3Bhost
  &X-Amz-Signature=XXXXX
```

When the browser uses this URL for PUT:
- S3 receives the request
- S3 reconstructs the canonical request using the query params
- S3 recomputes the signature using the IAM credentials in `X-Amz-Credential`
- Signatures match â†’ allow the PUT
- If current time > (X-Amz-Date + X-Amz-Expires) â†’ reject with `RequestExpired` error

**S3 durability:** Objects are stored across at least 3 AZs within us-east-1 automatically. 99.999999999% (11 9s) durability. No configuration needed.

**S3 server-side encryption (SSE-KMS):** When vehicle-service uploads (or presigned URL is used for PUT), S3 encrypts the object with the KMS `s3_key_arn` before writing to disk. On GET, S3 decrypts automatically (if the caller has `kms:Decrypt` permission).

---

### SNS â€” INTERNALS

**Message delivery architecture:**
```
maintenance-service â†’ SNS:Publish(topicArn, message)
  â†“
SNS receives message â†’ assigns MessageId â†’ stores durably in multiple AZs
  â†“
SNS checks subscriptions for the topic:
  - Email subscription (email@example.com)
  - (Future) SQS queue subscription
  â†“
For email subscription:
  SNS calls SES (Simple Email Service) or SMTP to deliver the email
  Retry policy: 3 immediate retries, then 2 pre-backoff (1s delay), then 10 post-backoff (20s delay)
  Total retry window: ~1 hour
  If all retries fail: message sent to DLQ (if configured) or discarded
  â†“
For SQS subscription:
  SNS calls SQS:SendMessage with the notification payload wrapped in SNS envelope JSON
  SQS stores message durably (can be consumed by any number of workers)
```

**SNS message envelope (received by SQS subscriber):**
```json
{
  "Type": "Notification",
  "MessageId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TopicArn": "arn:aws:sns:us-east-1:538661800892:fleetops-prod-service-alerts",
  "Subject": "Vehicle Insurance Expiry Alert",
  "Message": "{\"vehicleId\":\"V-001\",\"daysUntilExpiry\":23,...}",
  "Timestamp": "2026-06-22T09:00:00.000Z",
  "SignatureVersion": "1",
  "Signature": "XXXXX"
}
```

**KMS encryption:** SNS topic encrypted with `events_key_arn`. Messages are encrypted at rest in SNS. SQS subscribers must have `kms:Decrypt` permission.

**Fan-out pattern:** One SNS publish â†’ multiple subscribers all receive the same message simultaneously. This is the key benefit over direct SQS publish: one publish, many consumers.

---

### Lambda â€” INTERNALS (Deep Dive)

**Execution environment lifecycle:**
```
INIT phase (cold start, ~100-500ms for Node.js):
  AWS creates a microVM (Firecracker) â€” lightweight, sub-second boot
  Downloads function code from S3 (zip archive)
  Initializes Node.js runtime (Node 20)
  Runs module-level code (requires, AWS SDK initialization, env var reads)
  â†’ execution environment WARM

INVOKE phase (function execution, repeatable):
  Lambda service calls the handler function
  handler(event, context) executes
  Returns response
  â†’ execution environment stays WARM for ~15 minutes (configurable)
  
WARM invocation (same execution environment):
  handler() called directly â€” no cold start overhead
  Module-level variables (SDK clients, cached values) persist between invocations
  
SHUTDOWN phase:
  After inactivity timeout: environment destroyed
  Next invocation: new cold start
```

**Node.js Lambda advantages over Java Lambda:**
- Cold start: Node.js 100-300ms vs. Java 2-5 seconds (JVM initialization)
- Memory: Node.js uses ~50-100MB vs. Java ~200-512MB
- For this daily alerting use case, cold start doesn't matter much (runs once/day), but Node.js was chosen for simplicity

**Concurrency model:**
- Lambda creates one execution environment per concurrent request
- EventBridge triggers once/day â†’ 1 concurrent execution
- No concurrency limits needed for this use case
- If SNS triggers Lambda directly as a subscriber â†’ concurrency scales with message volume

**Environment variables injected by Terraform (modules/lambda):**
```
VEHICLE_SERVICE_URL   = "https://origin.fleetops.website"
AUTH_SERVICE_URL      = "https://origin.fleetops.website"
INSURANCE_SNS_ARN     = "arn:aws:sns:us-east-1:538661800892:fleetops-prod-insurance-alerts"
SERVICE_SNS_ARN       = "arn:aws:sns:us-east-1:538661800892:fleetops-prod-service-alerts"
LAMBDA_SERVICE_CREDENTIALS_SECRET_ARN = "arn:aws:secretsmanager:..."
```
Lambda reads `LAMBDA_SERVICE_CREDENTIALS_SECRET_ARN` at runtime and calls `secretsmanager:GetSecretValue` to get the actual password (not stored as plaintext env var).

---

### Step Functions â€” INTERNALS (Logging & Tracing)

**Execution persistence:**
Step Functions stores every state transition in its own managed database. Executions are durable â€” if the Lambda/service that's processing a task fails, the execution continues from where it was. This is fundamentally different from an in-memory state machine in the application code.

**Step Functions logging to CloudWatch:**
```hcl
# In modules/step-functions/main.tf
resource "aws_sfn_state_machine" "service_requests" {
  logging_configuration {
    level                  = "ALL"   # ALL, ERROR, FATAL, OFF
    include_execution_data = true    # Log input/output of each state
    log_destination        = "${aws_cloudwatch_log_group.sfn.arn}:*"
  }
}
```
With `level = "ALL"`, every state entry/exit, input/output, and error is logged to CloudWatch. This creates a complete audit trail: "Request REQ-101 entered PENDING state at 09:00:00, manager approved at 09:15:32, technician assigned at 09:20:00."

**X-Ray tracing with Step Functions:**
```hcl
resource "aws_sfn_state_machine" "service_requests" {
  tracing_configuration {
    enabled = true
  }
}
```
When X-Ray tracing is enabled, Step Functions creates trace segments for each state transition. These are stitched together in the X-Ray service map, showing the full execution graph of a service request workflow.

**Task token pattern (used for long-running approvals):**
In Wait states, Step Functions generates a task token and passes it to an activity/Lambda. The application stores the token in its DB. When the manager approves, the application calls `SendTaskSuccess(taskToken, output)`. Step Functions resumes the execution. This is how workflows can wait days or weeks for human approval without consuming compute resources.

---

### X-Ray â€” COMPLETE COVERAGE

**WHAT:** AWS distributed tracing service. Traces requests across multiple services â€” from ALB through EKS pods through AWS services (RDS, Bedrock, S3, Step Functions).

**WHY X-Ray:** In a microservices architecture, a single user request may touch auth-service â†’ vehicle-service â†’ Bedrock â†’ Redis. Without distributed tracing, identifying which service introduced latency is nearly impossible from logs alone. X-Ray gives a visual service map and per-request trace showing exact latency at each hop.

**HOW IT WORKS â€” trace propagation:**
```
1. Browser â†’ ALB â†’ auth-service pod
   X-Ray SDK (or CloudWatch agent) generates a trace ID: X-Amzn-Trace-Id: Root=1-xxx-yyy
   SDK creates a trace segment for the incoming request
   â†“
2. auth-service â†’ vehicle-service (internal call)
   SDK injects X-Amzn-Trace-Id header into the outgoing HTTP request
   vehicle-service SDK reads the header, creates a child subsegment with the same trace ID
   â†“
3. vehicle-service â†’ Bedrock (AWS SDK call)
   AWS SDK automatically propagates the trace context to the Bedrock API call
   Bedrock creates its own subsegment
   â†“
4. All segments/subsegments sent to X-Ray daemon (agent) â†’ X-Ray service
   X-Ray assembles the full trace from all segments sharing the same trace ID
   Service map built: shows all services + latency between them
```

**In FleetOps:** X-Ray is partially enabled via the CloudWatch Observability addon. The addon includes the CloudWatch agent which can collect X-Ray traces. Full application-level tracing requires adding the AWS X-Ray SDK to each Spring Boot service or using OpenTelemetry auto-instrumentation.

**Service Map:** X-Ray generates a visual graph: `ALB â†’ auth-service â†’ RDS` showing average latency, error rate, and trace count. Anomalies (sudden latency spike in vehicle-service â†’ Bedrock) are immediately visible.

**Sampling:** X-Ray samples requests to reduce cost. Default: 5% of requests sampled (1 per second + 5% of additional). All errors are sampled. Sampling rates configurable per rule.

**Cost:** $5 per million traces recorded + $0.50 per million traces retrieved. At low traffic, effectively free.

---

### DynamoDB â€” COMPLETE COVERAGE

**WHAT:** AWS-managed NoSQL key-value and document database. In FleetOps, used in two ways:
1. **State locking** (`fleetops-terraform-locks` table) â€” Terraform uses DynamoDB to prevent concurrent state file modifications
2. **Document metadata** (`modules/dynamodb`) â€” optional document metadata store for vehicle documents

**State locking DynamoDB table:**
```hcl
# In bootstrap/main.tf
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "fleetops-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"  # no capacity planning needed
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**How Terraform locking works with DynamoDB:**
```
terraform apply starts
  â†“
Terraform: DynamoDB.PutItem({LockID: "fleetops-terraform-state-johan/environments/prod/terraform.tfstate"}, 
                              ConditionExpression: "attribute_not_exists(LockID)")
  â†’ Conditional write: only succeeds if LockID does not exist
  â†“
If PutItem succeeds â†’ lock acquired â†’ apply proceeds
If PutItem fails (item exists) â†’ another apply is running â†’ Terraform exits with lock error
  â†“
After apply completes (success or failure):
  Terraform: DynamoDB.DeleteItem({LockID: "..."}) â†’ lock released
```

The conditional write is DynamoDB's distributed locking primitive. It's atomic â€” two concurrent `PutItem` calls with `attribute_not_exists` can never both succeed.

**Document metadata table (optional):**
Stores metadata for vehicle documents (S3 key, filename, content type, upload timestamp, vehicleId). This allows querying "all documents for vehicle V-045" without listing S3 objects.

**WHY DynamoDB for metadata over PostgreSQL:** DynamoDB scales to millions of rows with consistent sub-millisecond reads. For document metadata (high read volume, simple key-value access patterns), DynamoDB is more appropriate than a relational join. Cost: $0.25/million reads for on-demand pricing.

---

### ACM (AWS Certificate Manager) â€” COMPLETE COVERAGE

**WHAT:** AWS-managed TLS/SSL certificate service. Issues and auto-renews certificates for `*.fleetops.website`.

**WHY ACM:** Free certificates for AWS services (CloudFront, ALB). Auto-renewal 60 days before expiry. No private key management â€” AWS stores the private key and you never see it.

**WHY NOT Let's Encrypt:** Let's Encrypt requires deploying a certificate renewal agent on the server and managing certificate rotation. ACM handles renewal automatically with zero operational overhead. Also, ACM certificates cannot be exported â€” they can only be used with AWS services (security feature â€” no key theft possible).

**Wildcard certificate `*.fleetops.website`:**
- Covers: `fleetops.website`, `origin.fleetops.website`, `argocd.fleetops.website`
- A single wildcard cert for all subdomains
- Created by `modules/acm`

**DNS validation process:**
```hcl
resource "aws_acm_certificate" "fleetops" {
  domain_name               = "*.${var.domain_name}"
  subject_alternative_names = [var.domain_name]
  validation_method         = "DNS"
}

resource "aws_route53_record" "cert_validation" {
  zone_id = module.route53.zone_id
  name    = each.value.resource_record_name   # e.g., _abc123.fleetops.website
  type    = each.value.resource_record_type   # CNAME
  records = [each.value.resource_record_value]
  ttl     = 60
}
```

1. ACM generates a unique CNAME record name and value (e.g., `_abc123.fleetops.website. CNAME _def456.acm-validations.aws.`)
2. Terraform creates this CNAME in Route53 automatically
3. ACM queries DNS for the CNAME every few minutes
4. Once ACM sees the CNAME â†’ validates domain ownership â†’ issues certificate
5. CNAME stays in Route53 permanently (enables future auto-renewal)

**Auto-renewal:** ACM uses the same CNAME validation record for renewal. 60 days before expiry, ACM re-validates by re-checking the CNAME and issues a new certificate. The ALB and CloudFront pick up the new cert automatically â€” zero downtime.

**CloudFront requirement:** ACM certificates for CloudFront distributions MUST be created in `us-east-1`, even if all other resources are in a different region. CloudFront is a global service that references certificates only from us-east-1.

---

### CloudWatch â€” INTERNALS (Complete Observability Stack)

**Complete log flow from pod to CloudWatch:**
```
Spring Boot pod (auth-service):
  logger.info("User {} logged in", username)
  â†“ writes to stdout (System.out via Logback/SLF4J)
  
Container runtime (containerd):
  Captures stdout â†’ writes to /var/log/pods/{namespace}_{pod}_{uid}/{container}/*.log
  (JSON-formatted log lines on disk)
  
Fluent Bit DaemonSet (runs on every node, part of CloudWatch Observability addon):
  Tails /var/log/pods/fleetops-prod_*/*.log
  Parses JSON log lines (timestamp, stream, log content)
  Adds Kubernetes metadata: pod name, namespace, container name, node name
  Batches log records
  â†“
Fluent Bit output plugin: CloudWatch Logs
  Calls CloudWatch:PutLogEvents
  Log group: /aws/containerinsights/fleetops-prod-eks/application
  Log stream: {pod_name}/{container_name}
  â†“
CloudWatch Logs stores log records (encrypted with KMS events_key_arn)
  Retention: configured in modules/cloudwatch (e.g., 30 days)
```

**CloudWatch Alarms â†’ SNS â†’ Email chain:**
```
CloudWatch monitors: RDS CPUUtilization metric (from RDS)
  Every 5 minutes: get metric value
  Threshold: CPU > 80% for 2 consecutive 5-min periods (10 minutes sustained high CPU)
  â†“ threshold breached
CloudWatch: publish message to SNS topic (fleetops-prod-service-alerts)
  â†“
SNS: deliver email to alert_emails list
  "ALARM: RDS CPUUtilization exceeded 80%"
```

**CloudWatch Container Insights:**
The CloudWatch Observability addon installs the CloudWatch agent as a DaemonSet. It collects:
- Node-level metrics: CPU, memory, disk, network (from Linux kernel)
- Pod-level metrics: CPU, memory, network per pod
- Container-level metrics: per-container within a pod
These appear in CloudWatch as custom namespace `ContainerInsights`. Pre-built dashboards in CloudWatch show cluster health, pod performance, and node utilization.

**EKS Control Plane Logs:**
```hcl
# In modules/eks/cluster
resource "aws_eks_cluster" "main" {
  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
}
```
All five control plane log types sent to CloudWatch Log Group `/aws/eks/{cluster-name}/cluster`:
- `api`: every API server request (high volume, use with care)
- `audit`: Kubernetes audit log (who did what â€” critical for security forensics)
- `authenticator`: IAM authentication events
- `controllerManager`: reconciliation loop events
- `scheduler`: scheduling decisions

---

### ACM Certificate â€” SCALING
Certificates don't scale â€” they're static configurations. ACM auto-renews, ALB handles certificate rotation with zero downtime. Up to 2,500 certificates per account per region (free tier: 1,000).

---

## AUTHENTICATION & AUTHORIZATION FLOW PER SERVICE

### Complete Auth+Authz Flow (every service, every request)

**Step 1: Token validation (every protected endpoint):**
```java
// Spring Security JwtAuthenticationFilter (runs on every request)
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String header = request.getHeader("Authorization");
        // 1. Extract Bearer token
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response); // no token â†’ pass to next filter
            return;
        }
        String token = header.substring(7);
        
        // 2. Parse and verify JWT
        Claims claims = Jwts.parser()
            .verifyWith(secretKey)  // JWT_SECRET from env var
            .build()
            .parseSignedClaims(token)  // throws JwtException if invalid/expired
            .getPayload();
        
        // 3. Extract identity
        String username = claims.getSubject();
        List<String> roles = claims.get("roles", List.class);
        
        // 4. Set SecurityContext (for @PreAuthorize to read)
        UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
            username, null, roles.stream().map(SimpleGrantedAuthority::new).collect(toList())
        );
        SecurityContextHolder.getContext().setAuthentication(auth);
        
        chain.doFilter(request, response);
    }
}
```

**Step 2: Role authorization (on each endpoint):**
```java
@RestController
@RequestMapping("/api/vehicles")
public class VehicleController {

    @GetMapping  // GET /api/vehicles â€” any authenticated user
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<List<Vehicle>> getAll() { ... }

    @PostMapping  // POST /api/vehicles â€” MANAGER or ADMIN only
    @PreAuthorize("hasRole('MANAGER') or hasRole('ADMIN')")
    public ResponseEntity<Vehicle> create(@RequestBody VehicleRequest req) { ... }

    @DeleteMapping("/{id}")  // DELETE â€” ADMIN only
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> delete(@PathVariable String id) { ... }
}
```

`@PreAuthorize` uses Spring Expression Language (SpEL) evaluated against the SecurityContext. If the expression evaluates to false â†’ Spring Security throws `AccessDeniedException` â†’ 403 Forbidden response.

**Service-to-service authentication:**
When maintenance-service calls `http://fleetops-vehicle-service:8080/api/vehicles/{id}`, it can pass the user's JWT token forward in the `Authorization` header (user context propagation). If the endpoint is internal-only (not user-facing), it may be configured as `permitAll()` for same-namespace calls â€” mitigated by Kubernetes NetworkPolicy (only pods in `fleetops-prod` can reach vehicle-service).

---

## DATA FLOW PER SERVICE

### auth-service Data Flow
```
Request â†’ Spring Security Filter (JWT validate) â†’ AuthController
  â†’ UserRepository (JPA) â†’ HikariCP pool â†’ PostgreSQL auth_db
  â†’ AuditLogRepository (JPA) â†’ INSERT audit_log
  â†’ JWT library (in-memory) â†’ sign token with JWT_SECRET
  â†’ Response
```

### vehicle-service Data Flow
```
Request â†’ JWT Filter â†’ VehicleController
  â”œâ”€â”€ GET /vehicles:
  â”‚     â†’ Redis: GET vehicles::fleet:all
  â”‚     â†’ (miss) VehicleRepository (JPA) â†’ vehicle_db
  â”‚     â†’ Redis: SET vehicles::fleet:all PX 300000
  â”‚
  â”œâ”€â”€ POST /vehicles/{id}/ai-analysis:
  â”‚     â†’ Redis: GET ai-analysis:{id}
  â”‚     â†’ (miss) VehicleRepository â†’ vehicle_db
  â”‚     â†’ HTTP client â†’ maintenance-service (internal) â†’ maintenance_db
  â”‚     â†’ Bedrock SDK â†’ STS (IRSA) â†’ Bedrock API (us-east-1)
  â”‚     â†’ Redis: SET ai-analysis:{id} PX 300000
  â”‚
  â””â”€â”€ GET /vehicles/{id}/documents/upload-url:
        â†’ S3 SDK â†’ STS (IRSA) â†’ presigned URL generation (local computation)
        â†’ Return URL (no S3 API call yet â€” signing is local)
```

### maintenance-service Data Flow
```
Request â†’ JWT Filter â†’ MaintenanceController
  â”œâ”€â”€ POST /tasks:
  â”‚     â†’ TaskRepository (JPA) â†’ maintenance_db
  â”‚     â†’ (if alert condition) SNS SDK â†’ STS (IRSA) â†’ SNS:Publish
  â”‚
  â”œâ”€â”€ POST /media/{taskId}/upload:
  â”‚     â†’ Multipart file â†’ Java FileOutputStream â†’ EFS NFS mount
  â”‚       (/var/www/fleetops/shared-media/{taskId}/{filename})
  â”‚
  â””â”€â”€ GET /media/{taskId}/{filename}:
        â†’ Java FileInputStream â†’ EFS NFS mount â†’ stream bytes â†’ Response
```

### request-service Data Flow
```
Request â†’ JWT Filter â†’ RequestController
  â”œâ”€â”€ POST /requests:
  â”‚     â†’ HTTP client â†’ vehicle-service (validate vehicleId exists)
  â”‚     â†’ RequestRepository (JPA) â†’ request_db
  â”‚     â†’ Step Functions SDK â†’ STS (IRSA) â†’ StartExecution â†’ request_db (store executionArn)
  â”‚
  â”œâ”€â”€ PUT /requests/{id}/approve:
  â”‚     â†’ RequestRepository â†’ get executionArn
  â”‚     â†’ Step Functions SDK â†’ SendTaskSuccess(taskToken, approval)
  â”‚     â†’ RequestRepository â†’ UPDATE status=APPROVED
  â”‚
  â””â”€â”€ GET /requests/{id}/status:
        â†’ RequestRepository â†’ get executionArn
        â†’ Step Functions SDK â†’ DescribeExecution
        â†’ Return execution status
```

---

## LOGGING FLOW (Complete)

```
Application logs (Spring Boot â†’ Logback):
  All services use SLF4J + Logback
  Log format: JSON structured logging (timestamp, level, logger, message, MDC context)
  MDC (Mapped Diagnostic Context): thread-local request metadata added per-request
    - requestId: UUID for correlating all log lines within one request
    - username: authenticated user (from SecurityContext)
    - traceId: X-Ray trace ID (if X-Ray agent is attached)
  â†“
Container stdout (Logback appender â†’ System.out)
  â†“
containerd: captured to node filesystem
  /var/log/pods/fleetops-prod_{pod-name}_{uid}/{container}/0.log
  â†“
Fluent Bit DaemonSet (CloudWatch Observability addon):
  INPUT: tail /var/log/pods/fleetops-prod_*/*.log
  FILTER: kubernetes (adds pod metadata: name, namespace, labels, annotations)
  FILTER: parser (parses JSON log line â†’ structured record)
  OUTPUT: cloudwatch_logs
    log_group_name: /aws/containerinsights/fleetops-prod-eks/application
    log_stream_name: {namespace}.{pod-name}.{container-name}
    auto_create_group: true
  â†“
CloudWatch Logs: stored encrypted (KMS events_key_arn)
  Retention: 30 days (configurable in modules/cloudwatch)
  â†“
CloudWatch Logs Insights (query):
  fields @timestamp, @message
  | filter kubernetes.namespace_name = "fleetops-prod"
  | filter @message like "ERROR"
  | sort @timestamp desc
  | limit 100
```

**Audit logs** (auth-service â†’ PostgreSQL):
Sensitive events (login, logout, role change, failed login) are written to `audit_log` table in `auth_db`. These are independent of the Fluent Bit pipeline â€” they're in the relational database for structured querying by admins via `GET /api/audit/logs`.

---

## MONITORING FLOW (Complete)

```
Metrics sources:
  â”œâ”€â”€ Spring Boot Actuator â†’ /actuator/prometheus
  â”‚     Exposes: JVM heap, GC, HTTP request count/latency, HikariCP pool metrics
  â”‚     (if Spring Actuator + Micrometer + Prometheus registry on classpath)
  â”‚
  â”œâ”€â”€ kubelet â†’ /metrics (cAdvisor) â†’ metrics-server
  â”‚     Exposes: CPU, memory per pod/container
  â”‚     Used by: HPA for scaling decisions
  â”‚
  â””â”€â”€ CloudWatch Agent DaemonSet â†’ CloudWatch
        Collects: node CPU/memory/disk/network (from /proc, /sys)
        Collects: pod CPU/memory (from cAdvisor metrics)
        Namespace: ContainerInsights/{cluster-name}

CloudWatch Alarms watch ContainerInsights + RDS + ALB metrics:
  RDS CPUUtilization > 80% for 2 periods â†’ ALARM â†’ SNS â†’ email
  Lambda Errors > 0 â†’ ALARM â†’ SNS â†’ email
  ALB HTTPCode_Target_5XX_Count > 10 per 5min â†’ ALARM â†’ SNS â†’ email

Dashboard:
  CloudWatch dashboards (configured in modules/cloudwatch) show:
    - ALB request rate, error rate, latency (P50, P99)
    - EKS node CPU/memory utilization
    - RDS connections, CPU, IOPS
    - Lambda invocations, errors, duration
    - Redis hit rate, memory usage
```

---

## TERRAFORM INTERNALS â€” DEEP DIVE

### How the Dependency Graph Is Built

```
Terraform reads all .tf files in the directory
  â†“
Parser: builds AST (Abstract Syntax Tree) of all resource, module, local, output blocks
  â†“
Graph builder: creates a node for every resource + module + data source
  â†“
Edge creation (two types):
  1. IMPLICIT (attribute reference): 
     module.secrets_manager.db_host is referenced in module.lambda's inputs
     â†’ edge: secrets_manager â†’ lambda (lambda DEPENDS ON secrets_manager)
     
  2. EXPLICIT (depends_on):
     module.eks_addons { depends_on = [module.eks_nodegroup, module.secrets_manager] }
     â†’ edges: nodegroup â†’ addons, secrets_manager â†’ addons
  â†“
Graph validation:
  - Check for cycles (A depends on B depends on A â†’ error)
  - No cycles = valid DAG (Directed Acyclic Graph)
  â†“
Topological sort:
  Resources with no inbound edges (leaf nodes = no dependencies) â†’ can run first
  Example in FleetOps: module.kms has no dependencies â†’ runs first
  After kms completes: module.networking, module.rds, module.s3 (all depend on kms) â†’ run in parallel
  After networking: module.eks_cluster â†’ runs
  ...and so on
```

**Actual dependency order in FleetOps:**
```
Wave 0 (parallel): kms
Wave 1 (parallel): networking, route53
Wave 2 (parallel): rds, redis, s3, efs, secrets_manager, sns, acm, waf, cloudfront
Wave 3 (parallel): eks_cluster, iam
Wave 4: eks_oidc (needs eks_cluster)
Wave 5 (parallel): eks_nodegroup, ssm (after secrets_manager + iam)
Wave 6: eks_addons (needs nodegroup + secrets_manager)
Wave 7 (parallel): lambda, cloudwatch, cloudtrail, step_functions, eventbridge, config_aws
Wave 8: Route53 records (needs eks_addons for ALB data source)
```

### State Reconciliation Process

```
terraform plan:
  â†“
1. Read current state file (S3: fleetops-terraform-state-johan/environments/prod/terraform.tfstate)
   State file contains: resource ID â†’ attributes mapping
   Example: aws_vpc.main â†’ {id: "vpc-12345", cidr_block: "10.2.0.0/16", ...}
  â†“
2. Refresh (optional, default enabled in older Terraform, disabled by default in 1.x):
   For each resource in state: call AWS API to get current attributes
   Compare current attributes to state file attributes
   If they differ: update the in-memory state
   â†’ This detects manual changes (drift)
  â†“
3. Diff: compare refreshed current state to desired state (your .tf files)
   For each resource:
     - In .tf AND in state AND no diff â†’ no-op
     - In .tf AND in state AND diff â†’ update
     - In .tf AND NOT in state â†’ create
     - NOT in .tf AND in state â†’ destroy (with prune/lifecycle settings)
  â†“
4. Plan output: shows all creates/updates/destroys
   + symbols: create
   ~ symbols: update in-place
   - symbols: destroy
   -/+ symbols: destroy and recreate (for attributes that can't be updated in-place)
```

### State File Structure

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "resources": [
    {
      "module": "module.rds",
      "mode": "managed",
      "type": "aws_db_instance",
      "name": "main",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "fleetops-prod-postgresql",
            "address": "fleetops-prod-postgresql.xxxx.us-east-1.rds.amazonaws.com",
            "username": "fleetops",
            "password": "ACTUAL_PASSWORD_HERE",   â† plaintext in state!
            "db_name": null,
            "engine": "postgres",
            "engine_version": "15.7",
            ...
          }
        }
      ]
    }
  ]
}
```

This is why state encryption matters â€” the password appears in plaintext in the JSON. S3 KMS encryption protects it at rest.

---

## ARGOCD â€” INTERNALS

**ArgoCD reconciliation loop:**
```
ArgoCD Application controller (runs in argocd namespace):
  â†“
Every 3 minutes (or on webhook):
  1. Clone/fetch the Git repo (fleetops-deployments) â€” uses token from argocd-repo-fleetops-deployments secret
  2. Render the Helm chart with the specified values files (values.yaml + values-prod.yaml)
     For charts/auth-service: helm template with values â†’ list of K8s objects
  3. Compare rendered objects to live cluster objects (kubectl get)
     Uses server-side diff: sends rendered objects to kube-apiserver with dryRun=true
     kube-apiserver returns what the applied diff would be
  4. If diff exists â†’ Application status = OutOfSync
  
If automated sync enabled (syncPolicy.automated):
  5. Apply the rendered objects: kubectl apply (server-side apply with field manager=argocd)
  6. Sync waves: apply wave 0 â†’ wait for all wave 0 resources to be Healthy â†’ apply wave 1 â†’ ...
  7. Update Application status = Synced + Healthy
  
If selfHeal:
  8. If someone kubectl-edits a resource â†’ ArgoCD detects diff â†’ re-applies Git state â†’ reverts
```

**How ArgoCD determines "Healthy":**
Each resource type has a health check:
- Deployment: Healthy when `availableReplicas == desiredReplicas`
- Job: Healthy when `completionTime != null` (completed)
- Ingress: Healthy when `loadBalancer.ingress[0].hostname != ""` (ALB DNS assigned)
- ExternalSecret: Healthy when `Ready=True` condition (secret synced)

**ArgoCD App-of-Apps sync wave timing:**
Wave N resources must ALL be Healthy before wave N+1 begins. If platform secrets (wave 0) never become Healthy (e.g., ESO can't sync because IAM role is wrong), ALL subsequent waves are blocked. This is the most common deployment failure mode.

---

## DOCKER MULTI-STAGE BUILD â€” INTERNALS

**auth-service Dockerfile:**
```dockerfile
# Stage 1: Build (Maven + JDK 21)
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline    # Pre-download deps (cache layer)
COPY src/ src/
RUN mvn -B -DskipTests clean package    # Build JAR

# Stage 2: Runtime (JRE only, no build tools)
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S fleetops && adduser -S fleetops -G fleetops    # non-root user
USER fleetops
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar    # Only the JAR, no Maven, no source
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Why multi-stage:** The build stage needs Maven, JDK 21, all test dependencies (~800MB). The runtime only needs the JRE and the JAR (~200MB). Multi-stage builds produce a final image with only the runtime artifacts â€” no build tools, no source code, smaller attack surface, smaller image to push/pull.

**Docker layer caching:** Each `RUN`, `COPY`, `ADD` instruction creates a layer. Layers are content-addressed (SHA256 hash). If the content of a layer's inputs hasn't changed, Docker reuses the cached layer. `COPY pom.xml` + `RUN mvn dependency:go-offline` is its own layer â€” if only Java source changes (not pom.xml), the dependency download layer is reused, dramatically speeding up the build.

**Image layer structure (runtime image):**
```
Layer 0: eclipse-temurin:21-jre-alpine base (JRE binaries, Alpine Linux)
Layer 1: RUN addgroup/adduser (new /etc/passwd entry)
Layer 2: COPY --from=build app.jar (the Spring Boot fat JAR ~50MB)
```
Total image size: ~180-250 MB.

---

## DEVOPS AGENT ROLE

From `modules/iam/main.tf`:
```hcl
resource "aws_iam_role" "devops_agent" {
  name = "${local.name_prefix}-devops-agent-role"
  assume_role_policy = jsonencode({
    Principal = { Service = "aidevops.amazonaws.com" }
    Action    = "sts:AssumeRole"
  })
}
# Policies: CloudWatchReadOnlyAccess, AmazonEKSClusterPolicy, AmazonRDSReadOnlyAccess
```

**Purpose:** This role is created for an AWS AI DevOps agent (if used with Amazon Q Developer or similar AWS AI services). The service principal `aidevops.amazonaws.com` allows AWS's AI-powered DevOps tools to assume this role and perform read-only diagnostics: read CloudWatch logs/metrics, describe EKS cluster state, query RDS metrics â€” without write access.

**Why read-only:** An AI diagnostic agent should never modify production infrastructure. Read-only access ensures it can investigate issues (high CPU, pod failures, DB connection errors) but cannot accidentally trigger changes.

---

## NGINX FRONTEND CONFIGURATION

**Dockerfile (frontend):**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json .
RUN npm ci --prefer-offline    # Install from lockfile (deterministic)
COPY . .
RUN npm run build              # Vite: outputs to dist/

FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

**nginx.conf** (SPA routing requirement):
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Static assets (hashed filenames) â€” long cache
    location ~* \.(js|css|png|jpg|gif|ico|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # API proxy (local dev) or pass-through (production: CloudFront handles this)
    location /api/ {
        proxy_pass http://backend:8080;
    }

    # SPA fallback â€” ALL unknown routes return index.html
    # This is critical for React Router: /dashboard, /vehicles/V-001, etc.
    # Without this, refreshing /dashboard returns Nginx 404 (no file named 'dashboard')
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

**Why `try_files $uri $uri/ /index.html`:** React Router handles routing client-side. The server only has one HTML file (`index.html`). If a user refreshes `https://fleetops.website/vehicles/V-001`, Nginx must return `index.html` (not a 404) â€” React Router then reads the URL and renders the correct component. Without this, any hard refresh on a non-root URL returns 404.

---

## SPRING BOOT ACTUATOR â€” INTERNALS

**What /actuator/health returns:**
```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 10737418240,
        "free": 8589934592,
        "threshold": 10485760,
        "exists": true
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.0.12"
      }
    }
  }
}
```

**Health indicator evaluation:**
- `db`: Spring sends a JDBC validation query (SELECT 1) to each configured DataSource. If it returns within `spring.datasource.hikari.connectionTimeout`, status = UP.
- `redis`: Spring pings the Redis connection. If PONG received, status = UP.
- `diskSpace`: checks free disk space > threshold (10MB). If EFS is mounted, it checks EFS free space.

**Readiness vs Liveness distinction in Spring Boot:**
- `/actuator/health/liveness` â†’ checks: is the application alive? (JVM running, not in deadlock). Never checks external dependencies. If DB is down, liveness is still UP â€” the pod should not be restarted just because DB is temporarily unavailable.
- `/actuator/health/readiness` â†’ checks: is the application ready to serve traffic? Includes DB, Redis health checks. If DB is down, readiness = DOWN â†’ pod removed from ALB target group.

In FleetOps Helm charts, both probes use `/actuator/health`. A more precise configuration would use the split endpoints. This is a known improvement.

---

## MISSING Q&A â€” ADDED

**Q: What is X-Ray and how would traces flow through FleetOps?**
A: X-Ray is AWS's distributed tracing service. A trace starts at the ALB (or first instrumented service), generates a `X-Amzn-Trace-Id` header, and propagates through every downstream call. Each service creates segments and subsegments. For FleetOps: a single `/api/vehicles/{id}/ai-analysis` request would show segments for: auth-service JWT validation, vehicle-service DB query, internal HTTP call to maintenance-service, Bedrock API call. The service map shows average latency per hop and error rates. Currently partially enabled via CloudWatch Observability addon; full tracing requires AWS X-Ray SDK or OpenTelemetry in each Spring Boot service.

**Q: Why does DynamoDB use PAY_PER_REQUEST for the Terraform lock table?**
A: The lock table has extremely low and sporadic throughput â€” it only gets written to when `terraform apply` runs (maybe a few times per day). Provisioned capacity (even the minimum 1 RCU/1 WCU) would cost ~$0.59/month for capacity that's almost never used. PAY_PER_REQUEST charges per actual read/write operation â€” at this usage, effectively free (~$0.01/month).

**Q: What happens during ACM certificate renewal?**
A: AWS ACM auto-renews 60 days before expiry. It re-checks the CNAME validation record in Route53 (which was created by Terraform and stays permanently). Once re-validated, ACM issues a new certificate. The ALB and CloudFront automatically start using the new certificate â€” no downtime, no manual action. The old certificate remains valid until its expiry date.

**Q: How does Fluent Bit avoid losing logs during pod restart?**
A: Fluent Bit maintains a position file (`/var/log/flb_kube.db`) tracking how far it has read each log file. If Fluent Bit restarts, it resumes from where it stopped. During pod restart, container logs continue to be written to the node filesystem (containerd writes to `/var/log/pods/...`), and Fluent Bit picks them up on its next poll. Brief log gaps are possible if Fluent Bit is also restarting at the same time.

**Q: What is the `aws-auth` ConfigMap and why does it matter?**
A: The `aws-auth` ConfigMap in `kube-system` maps IAM users/roles to Kubernetes RBAC users/groups. Without this mapping, even the IAM user who created the EKS cluster cannot run kubectl commands. In FleetOps, `admin_iam_user_arns` is passed to the EKS cluster module and those IAM users are added to `aws-auth` as `system:masters` (cluster admin). GitHub Actions IAM role is also mapped for CI/CD kubectl access.

**Q: What is an EFS Access Point?**
A: EFS Access Points enforce a specific POSIX user/group and root directory for each client. The maintenance-service Helm chart uses an EFS access point with `posixUser.uid: 1000` (matching `runAsUser: 1000` in the pod securityContext) and a `rootDirectory.path: /fleetops/media` with automatic permissions. This ensures maintenance-service pods write as UID 1000 and are isolated to the `/fleetops/media` directory of the EFS filesystem â€” other applications cannot read/write this path even if they mount the same EFS.

**Q: How does the Cluster Autoscaler interact with Pending pods?**
A: When a pod enters Pending state with reason `Insufficient cpu` or `Insufficient memory`, the Cluster Autoscaler identifies which node group can accommodate it, calls AWS AutoScaling to increment the desired count, and waits for the new node to register. Once the node is Ready, the scheduler places the pending pod. Scale-down: Cluster Autoscaler identifies underutilized nodes (all pods fit on fewer nodes), cordons the node (no new pods), drains pods (respecting PodDisruptionBudgets), and terminates the EC2 instance.



---

# PHASE 4 SUPPLEMENT â€” KUBERNETES INTERNALS (DEEP DIVE)

---

## Container Runtime Internals â€” containerd â†’ runc â†’ Linux primitives

When kubelet decides to start a container, the chain is:

```
kubelet
  â†“ CRI (Container Runtime Interface) gRPC call
containerd (container runtime, runs as systemd service on every EKS node)
  â†“ OCI spec generation
runc (low-level container runtime â€” actually creates the container)
  â†“ Linux syscalls
Kernel: namespaces + cgroups + overlay filesystem
```

**What runc actually does to create a container:**

1. **Linux namespaces** â€” isolation per container:
   - `pid`: container sees only its own processes (PID 1 = the entrypoint)
   - `net`: container gets its own network stack (eth0, routing table, iptables)
   - `mnt`: container gets its own mount namespace (overlayFS)
   - `uts`: container can have its own hostname
   - `ipc`: isolated IPC (message queues, semaphores)
   - `user`: optional UID/GID remapping (rootless containers)

2. **cgroups v2** â€” resource enforcement:
   - `cpu`: limits CPU time using the CFS scheduler. `resources.limits.cpu: 500m` = 500 millicores = 0.5 CPU cores = the cgroup is limited to 50% of one CPU second per second.
   - `memory`: limits physical + swap memory. `resources.limits.memory: 512Mi` = if the process tries to allocate more, the Linux OOM killer is invoked inside the cgroup.
   - `pids`: limits number of processes inside the container.

3. **OverlayFS** â€” the container filesystem:
   ```
   Lower layers (read-only):  eclipse-temurin:21-jre-alpine base + app.jar layer
                               â†‘ these are the image layers pulled from ECR
   Upper layer (read-write):   container's own writable layer
                               â†‘ any files written at runtime go here
   Merged view:                what the process sees as its filesystem
   ```
   Image layers are shared across all containers using that image on the same node â€” one ECR pull serves all pods of the same image on that node.

**OOMKilled â€” what actually happens:**
1. Spring Boot JVM exceeds the container's memory limit (cgroup memory.max)
2. Linux kernel's OOM killer (inside the cgroup) selects the process with highest `oom_score`
3. OOM killer sends SIGKILL to the JVM process (not SIGTERM â€” immediate kill, no graceful shutdown)
4. containerd reports container exit code 137 (128 + SIGKILL = 137)
5. kubelet sees container exited with 137 â†’ marks container as OOMKilled
6. kubelet restarts the container (respecting restartPolicy: Always)
7. Pod stays Running but `kubectl describe pod` shows OOMKilled in Last State

**Why OOMKilled matters in production:** If you see OOMKilled in metrics, the JVM heap is hitting the limit. Solutions: increase `resources.limits.memory` or tune JVM flags: `-XX:MaxRAMPercentage=75.0` (limits heap to 75% of container memory, leaving headroom for off-heap).

---

## How kube-proxy Programs iptables (Packet Path)

When a pod calls `http://fleetops-auth-service:8080`:

```
1. DNS resolution:
   Pod's /etc/resolv.conf: nameserver 172.20.0.10 (CoreDNS ClusterIP)
   Lookup: fleetops-auth-service.fleetops-prod.svc.cluster.local
   CoreDNS returns: 172.20.xx.yy (the Service ClusterIP)

2. Packet leaves pod eth0 â†’ enters node network namespace via veth pair

3. Node kernel's iptables chain (programmed by kube-proxy):

   PREROUTING â†’ KUBE-SERVICES chain:
     Match: destination 172.20.xx.yy:8080 (auth-service ClusterIP)
     â†’ Jump to KUBE-SVC-AUTH chain

   KUBE-SVC-AUTH chain (random load balancing):
     Rule 1: 1/3 probability â†’ jump to KUBE-SEP-AUTH-POD1 (pod 1 DNAT)
     Rule 2: 1/2 probability â†’ jump to KUBE-SEP-AUTH-POD2 (pod 2 DNAT)
     Rule 3: default â†’ jump to KUBE-SEP-AUTH-POD3 (pod 3 DNAT)

   KUBE-SEP-AUTH-POD1:
     DNAT: rewrite destination to 10.2.x.y:8080 (actual pod IP:port)

4. Packet now has real pod IP as destination
5. VPC routing: pod IPs are real VPC IPs â†’ packet routed directly to pod
   (same node: stays local; different node: sent via VPC routing to target node's ENI)
```

**Why ALB target-type: ip bypasses kube-proxy for inbound traffic:** When the ALB routes to pod IP directly, packets arrive at the pod with ALB as source â€” kube-proxy iptables rules are NOT involved in the inbound path. kube-proxy is only involved in east-west traffic (pod-to-pod via ClusterIP).

**iptables vs IPVS:** IPVS is a kernel module for load balancing with O(1) lookup (hash table) vs iptables O(n) (linear scan of rules). For large clusters (10,000+ services), IPVS is significantly faster. FleetOps uses iptables (default for EKS) â€” perfectly adequate for this scale.

---

## CoreDNS Internals

CoreDNS runs as a Deployment in `kube-system` with 2 replicas. It's the DNS server for every pod in the cluster.

**CoreDNS Corefile (configuration):**
```
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf    # forward external queries to upstream DNS
    cache 30
    loop
    reload
    loadbalance
}
```

**Resolution chain for `fleetops-auth-service`:**
```
Pod's /etc/resolv.conf:
  search fleetops-prod.svc.cluster.local svc.cluster.local cluster.local ec2.internal
  nameserver 172.20.0.10
  options ndots:5

Query: fleetops-auth-service (ndots < 5)
  â†’ Try: fleetops-auth-service.fleetops-prod.svc.cluster.local
  â†’ CoreDNS: checks Kubernetes service store
  â†’ Found: Service fleetops-auth-service in namespace fleetops-prod, ClusterIP = 172.20.xx.yy
  â†’ Return A record: 172.20.xx.yy TTL 30

Full FQDN (cross-namespace, e.g., from kube-system to fleetops-prod):
  Query: fleetops-auth-service.fleetops-prod.svc.cluster.local
  â†’ CoreDNS resolves directly
```

**Why `ndots: 5`:** If the query has fewer than 5 dots, the search list is tried first (so `fleetops-auth-service` â†’ tries `fleetops-auth-service.fleetops-prod.svc.cluster.local` before sending as a raw hostname). This is why short service names work without full FQDNs.

**CoreDNS caching:** TTL of 30 seconds. DNS resolution results are cached in CoreDNS memory. High-frequency services (called thousands of times/minute) don't cause DNS overhead.

---

## ExternalSecrets Operator (ESO) â€” Reconcile Loop Internals

ESO is a Kubernetes controller (runs as a Deployment in `kube-system`). It watches `ExternalSecret` and `ClusterSecretStore` CRDs.

```
ESO controller starts â†’ registers watch on ExternalSecret CRDs (all namespaces)
  â†“
Event: ExternalSecret fleetops-postgres-secret created in fleetops-prod
  â†“
ESO reconcile loop fires:
  1. Read ExternalSecret spec:
     - secretStoreRef: aws-secrets-manager (ClusterSecretStore)
     - remoteRef.key: fleetops/prod/db-credentials
  â†“
  2. Read ClusterSecretStore: aws-secrets-manager
     - provider.aws.service: SecretsManager
     - auth.jwt.serviceAccountRef: external-secrets-sa (IRSA)
  â†“
  3. Get IRSA credentials:
     external-secrets-sa has annotation: eks.amazonaws.com/role-arn: fleetops-prod-external-secrets-role
     ESO reads projected ServiceAccount token from /var/run/secrets/eks.amazonaws.com/serviceaccount/token
     Calls STS: AssumeRoleWithWebIdentity(roleArn, webIdentityToken)
     Gets temporary credentials for fleetops-prod-external-secrets-role
  â†“
  4. Call Secrets Manager API (with IRSA credentials):
     secretsmanager:GetSecretValue(SecretId: "fleetops/prod/db-credentials")
     Returns JSON: {"host":"xxx.rds.amazonaws.com","username":"fleetops","password":"xxx"}
  â†“
  5. Map secret keys per ExternalSecret spec:
     POSTGRES_HOST â†’ .host
     POSTGRES_USERNAME â†’ .username
     POSTGRES_PASSWORD â†’ .password
  â†“
  6. Create or update K8s Secret:
     kubectl create secret generic fleetops-postgres-secret \
       --from-literal=POSTGRES_HOST=xxx.rds.amazonaws.com \
       --from-literal=POSTGRES_USERNAME=fleetops \
       --from-literal=POSTGRES_PASSWORD=xxx
     (values are base64-encoded in etcd, encrypted with KMS secrets_key_arn)
  â†“
  7. Set ownerReference: ExternalSecret â†’ K8s Secret
     If ExternalSecret is deleted â†’ K8s Secret is garbage-collected (creationPolicy: Owner)
  â†“
  8. Update ExternalSecret.Status: Ready=True, syncedAt=now
  â†“
  9. Schedule next reconcile after refreshInterval: 1h
```

**What happens if Secrets Manager is temporarily unavailable:** ESO retries with exponential backoff. The existing K8s Secret is NOT deleted â€” it stays from the last successful sync. Pods continue to use the cached secret value.

**What happens if the secret value changes in Secrets Manager:** After `refreshInterval: 1h`, ESO re-fetches and updates the K8s Secret. Pods that have the secret mounted as env vars do NOT get the update automatically â€” they must restart. Pods with secrets mounted as files get the update within ~1 minute (kubelet watches the Secret and updates the file on disk).

---

## RBAC Evaluation â€” How Authorization Works

Every API request to kube-apiserver goes through the RBAC authorizer:

```
Request: GET /apis/apps/v1/namespaces/fleetops-prod/deployments
User: github-actions-role (from aws-auth ConfigMap mapping)
  â†“
RBAC authorizer builds the question:
  Can user github-actions-role PERFORM GET ON deployments IN fleetops-prod?
  â†“
Walk all RoleBindings in fleetops-prod:
  Found: RoleBinding deploy-access binds ClusterRole deployer to github-actions-role
  â†“
Walk rules in ClusterRole deployer:
  rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "patch", "update"]
  â†“
Match: apiGroup "apps" âœ“, resource "deployments" âœ“, verb "get" âœ“
  â†’ ALLOW
```

**Deny by default:** Kubernetes RBAC is deny-by-default. If no Rule grants the verb+resource+apiGroup combination, the request is denied. There's no DENY rule â€” absence of ALLOW = DENY.

**ClusterRole vs Role:**
- `Role`: namespaced â€” grants permissions within one namespace
- `ClusterRole`: cluster-scoped â€” grants permissions across all namespaces (when bound via ClusterRoleBinding) OR within one namespace (when bound via RoleBinding in that namespace)

**In FleetOps:**
- ArgoCD uses a ClusterRole (needs to manage resources in `fleetops-prod`, `kube-system`, `argocd` namespaces)
- ESO uses a ClusterRole (needs to read ExternalSecrets in all namespaces)
- Individual service pods don't have RBAC permissions (they don't call the K8s API)

---

## NetworkPolicy Enforcement

NetworkPolicy resources define firewall rules at the pod level. In EKS, enforcement is done by the VPC CNI (which supports network policies via Kubernetes Network Policy Controller mode).

**Example NetworkPolicy for auth-service:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: auth-service-netpol
  namespace: fleetops-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: fleetops-auth-service
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: {}          # any pod in fleetops-prod namespace
    ports: [{port: 8080}]
  egress:
  - to:
    - ipBlock: {cidr: 10.2.0.0/16}   # VPC CIDR (RDS, Redis)
    ports: [{port: 5432}, {port: 6379}]
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system   # CoreDNS
    ports: [{port: 53, protocol: UDP}]
```

**How enforcement works (VPC CNI eBPF mode):** The VPC CNI network policy controller programs eBPF maps (or iptables rules per pod netns) that enforce the allow/deny decisions at the kernel level. A packet from an unauthorized source is dropped at the kernel before it reaches the container. This is NOT enforced at the application layer.

**Important:** If no NetworkPolicy selects a pod, ALL traffic is allowed (default permissive). If at least one NetworkPolicy selects a pod, ONLY traffic matching that policy's rules is allowed.

---

## Rolling Update â€” maxSurge and maxUnavailable Mechanics

Deployment spec:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # can temporarily have current+1 pods
    maxUnavailable: 0  # must always have current count available
```

**With 2 replicas, maxSurge: 1, maxUnavailable: 0:**

```
Initial state: [old-pod-1 Ready] [old-pod-2 Ready]

Step 1: Deployment controller creates new-pod-1 (total: 3 pods, maxSurge=1 allows it)
  [old-pod-1 Ready] [old-pod-2 Ready] [new-pod-1 Starting]

Step 2: new-pod-1 becomes Ready (passes startup + readiness probes)
  [old-pod-1 Ready] [old-pod-2 Ready] [new-pod-1 Ready]

Step 3: maxUnavailable=0 satisfied (2 ready pods still serving). Terminate old-pod-1
  [old-pod-1 Terminating] [old-pod-2 Ready] [new-pod-1 Ready]
  During termination: kubelet sends SIGTERM to pod. PreStop hook runs.
  Spring Boot graceful shutdown: drains in-flight requests.
  ALB deregisters pod IP (deregistration delay 30s).

Step 4: old-pod-1 fully gone. Create new-pod-2
  [old-pod-2 Ready] [new-pod-1 Ready] [new-pod-2 Starting]

Step 5: new-pod-2 Ready. Terminate old-pod-2.
  [old-pod-2 Terminating] [new-pod-1 Ready] [new-pod-2 Ready]

Final state: [new-pod-1 Ready] [new-pod-2 Ready]
Zero downtime: at least 2 pods were always Ready throughout.
```

**maxUnavailable: 1 (faster but allows brief capacity reduction):**
Steps 1+3 happen simultaneously â€” one old pod is terminated as soon as the new pod is created (before it's Ready). Faster deployment but briefly runs at 1 replica capacity.

---

## How a Node Registers with EKS

```
EC2 instance (EKS node) starts (from Auto Scaling Group launch template)
  â†“
User data script runs:
  /etc/eks/bootstrap.sh fleetops-prod-eks-cluster \
    --kubelet-extra-args '--node-labels=nodegroup=main' \
    --b64-cluster-ca ${CA_DATA}
  â†“
bootstrap.sh:
  1. Configures kubelet: sets --cloud-provider=aws, --cluster-dns, --kubeconfig
  2. Gets cluster endpoint from EKS API (using EC2 instance profile / node role)
  3. Starts kubelet systemd service
  â†“
kubelet starts:
  1. Calls kube-apiserver: POST /apis/certificates.k8s.io/v1/certificatesigningrequests
     (kubelet bootstrap: requests a client certificate for itself)
  2. kube-apiserver approves the CSR (auto-approved for EKS nodes using the node role)
  3. kubelet gets a signed certificate â†’ stores in /var/lib/kubelet/pki/
  4. kubelet calls kube-apiserver: POST /api/v1/nodes (registers itself)
     Node name = EC2 private DNS name: ip-10-2-x-y.ec2.internal
  5. Node enters NotReady state (no pods assigned yet)
  â†“
VPC CNI DaemonSet pod starts on the new node:
  1. Calls EC2 API: creates secondary ENI, assigns secondary IPs
  2. Node condition NetworkReady=True
  â†“
Node enters Ready state
  â†“
kube-scheduler can now assign pods to this node
  â†“
Any pending pods with `Unschedulable` reason are now re-evaluated by the scheduler
```

---

## How a Pod Gets Its IRSA Token (Projected ServiceAccount Volume)

```
Pod spec:
  serviceAccountName: fleetops-app

Admission webhook (Pod Identity Webhook) intercepts pod creation:
  â†“
Checks: does ServiceAccount fleetops-app have annotation?
  eks.amazonaws.com/role-arn: arn:aws:iam::538661800892:role/fleetops-prod-app-service-account-role
  â†“
Webhook mutates the pod spec (adds volumes + env vars):

volumes:
  - name: aws-iam-token
    projected:
      sources:
      - serviceAccountToken:
          audience: sts.amazonaws.com
          expirationSeconds: 86400   # 24 hours
          path: token

env:
  - name: AWS_WEB_IDENTITY_TOKEN_FILE
    value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
  - name: AWS_ROLE_ARN
    value: arn:aws:iam::538661800892:role/fleetops-prod-app-service-account-role
  - name: AWS_REGION
    value: us-east-1
  â†“
kubelet: mounts the projected volume
  kube-apiserver generates a bound ServiceAccount token for the pod:
    Signed by the cluster's OIDC private key
    audience: sts.amazonaws.com
    sub: system:serviceaccount:fleetops-prod:fleetops-app
    Written to /var/run/secrets/eks.amazonaws.com/serviceaccount/token
  â†“
Pod starts. AWS SDK in the Spring Boot container:
  Credential provider chain detects AWS_WEB_IDENTITY_TOKEN_FILE env var
  Reads the token from the projected volume
  Calls STS: AssumeRoleWithWebIdentity(roleArn, webIdentityToken=token)
  STS: verifies token against OIDC issuer URL (public keys from EKS OIDC endpoint)
  Sub matches trust policy condition: system:serviceaccount:fleetops-prod:fleetops-app â†’ ALLOW
  Returns temporary credentials (access key + secret + session token, 1h expiry)
  â†“
SDK caches credentials. Before 1h expiry:
  Re-reads the projected token (kubelet auto-refreshes it)
  Re-calls STS â†’ gets new temporary credentials
  Seamless rotation â€” no application restart needed
```

---

## ConfigMap Hot Reload (and Why Env Vars Don't Update)

**Environment variables from ConfigMap: NO hot reload**
```yaml
envFrom:
  - configMapRef:
      name: fleetops-config
```
These env vars are set at pod startup. If the ConfigMap changes, the pod does NOT see the new value. The pod must restart.

**Volumes from ConfigMap: YES hot reload (up to 1 minute delay)**
```yaml
volumes:
  - name: config
    configMap:
      name: fleetops-config
```
kubelet polls ConfigMaps every 60 seconds (configurable: `--sync-frequency`). When a ConfigMap changes, kubelet atomically replaces the files in the volume mount (symlink swap). Applications that watch the file (e.g., Nginx, Prometheus, Fluent Bit) pick up the new config automatically.

**In FleetOps:** All configuration uses environment variables (via ConfigMap â†’ envFrom). Config changes require a pod restart. This is acceptable â€” rolling restart via `kubectl rollout restart deployment` causes zero downtime with 2+ replicas.

---

## What Happens When a Secret Rotates

```
Secrets Manager: secret fleetops/prod/db-credentials password rotated
  â†“ (after up to 1 hour)
ESO refreshInterval fires: ESO re-fetches from Secrets Manager
  â†’ Updates K8s Secret fleetops-postgres-secret with new password value
  â†“
Existing pods: env vars already set at startup = OLD password
  â†’ Pods continue to use old password until restarted
  â†’ HikariCP pool connections: established with old credentials
  â†’ DB: if old credentials are immediately revoked â†’ connection errors begin
  â†“ solution
ArgoCD or CI/CD triggers rolling restart:
  kubectl rollout restart deployment/fleetops-auth-service
  â†’ New pods start â†’ read new env vars â†’ HikariCP connects with new password âœ“
```

**Best practice for secret rotation:** Use a dual-password rotation strategy (Secrets Manager supports this):
1. Create new password in Secrets Manager
2. Add the new password to the DB (without revoking the old one)
3. Wait for pods to restart with the new password
4. Revoke the old password

This prevents the window where old pods can't connect.

---

# PHASE 8 SUPPLEMENT â€” CI/CD INTERNALS (DEEP DIVE)

---

## GitHub Actions Runner Internals

**How a job gets picked up:**

```
Developer runs: git push origin main
  â†“
GitHub receives push â†’ evaluates workflow triggers
  on: push: branches: [main] â†’ matches â†’ queues workflow run
  â†“
GitHub assigns the workflow run to a runner:
  ubuntu-latest = GitHub-hosted runner
  GitHub has a pool of pre-created runner VMs (Ubuntu 22.04 on Azure VMs)
  A runner process is started â†’ connects to GitHub's job assignment queue via HTTPS long-polling
  â†“
Runner receives job (e.g., job: prepare)
  1. Downloads the repo: git clone https://x-access-token:${GITHUB_TOKEN}@github.com/repo.git
  2. Checks out the commit that triggered the workflow
  3. Reads the steps from the workflow YAML
  4. For each step:
     - `uses: actions/checkout@v4` â†’ downloads the action from github.com/actions/checkout, runs index.js
     - `run: echo "hello"` â†’ spawns a bash process on the runner
  5. Collects outputs (echo "key=value" >> $GITHUB_OUTPUT)
  6. Reports job status to GitHub API
  â†“
Job completes â†’ runner VM is discarded (ephemeral)
Next job: fresh runner VM from the pool
```

**Why ephemeral runners matter for security:** Each job starts with a clean slate. Secrets from Job 1 cannot leak to Job 2 (different VM). No persistent files between jobs (artifacts are the explicit handoff mechanism).

**Job parallelism:** Jobs without `needs:` run in parallel on separate runners simultaneously. Jobs with `needs: [prepare, quality]` wait for both to complete before starting on a fresh runner.

---

## OIDC Token Structure and Verification â€” Deep Dive

**The GitHub OIDC JWT (what GitHub issues to the Actions runner):**

```json
// Header
{
  "alg": "RS256",
  "typ": "JWT",
  "x5t": "sha1-thumbprint-of-signing-cert",
  "kid": "key-id-1234"
}

// Payload
{
  "jti": "unique-token-id",
  "sub": "repo:FleetOps-V2/fleetops-auth-service:ref:refs/heads/main",
  "aud": "sts.amazonaws.com",
  "ref": "refs/heads/main",
  "sha": "a3f92c1d...",
  "repository": "FleetOps-V2/fleetops-auth-service",
  "repository_owner": "FleetOps-V2",
  "run_id": "12345678",
  "run_number": "42",
  "actor": "johannabyvannilam",
  "workflow": "CI Pipeline",
  "job_workflow_ref": "FleetOps-V2/fleetops-github-workflows/.github/workflows/java-ci-ecr.yml@refs/heads/main",
  "iss": "https://token.actions.githubusercontent.com",
  "nbf": 1719043200,
  "exp": 1719046800,
  "iat": 1719043200
}

// Signature: RS256(header.payload, GitHub's private key)
```

**What AWS STS does to verify this JWT:**

```
configure-aws-credentials action calls:
  sts:AssumeRoleWithWebIdentity(
    RoleArn: "arn:aws:iam::538661800892:role/fleetops-prod-github-ecr-role",
    WebIdentityToken: "eyJhbGciOiJSUzI1NiI...",
    RoleSessionName: "GitHubActions-push-12345"
  )
  â†“
STS:
  1. Decode JWT header â†’ find kid (key ID)
  2. Fetch OIDC provider's public keys: GET https://token.actions.githubusercontent.com/.well-known/jwks
     â†’ Returns JSON Web Key Set (public RSA keys)
     â†’ Find key with matching kid
  3. Verify JWT signature using the public key (RS256)
  4. Verify iss = "https://token.actions.githubusercontent.com"
  5. Verify aud = "sts.amazonaws.com" (the OIDC provider in IAM was registered with this audience)
  6. Verify exp > now (not expired)
  7. Evaluate trust policy conditions:
     StringLike: "token.actions.githubusercontent.com:sub" = "repo:FleetOps-V2/fleetops-auth-service:ref:refs/heads/main"
     â†’ Checks JWT payload sub field matches
  â†“
All checks pass â†’ STS returns:
  AccessKeyId, SecretAccessKey, SessionToken (valid for 1 hour)
  â†“
configure-aws-credentials action sets env vars:
  AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN
  Subsequent aws CLI or SDK calls use these automatically
```

**Why the trust policy uses `StringLike` with wildcards:** The condition `repo:FleetOps-V2/*:ref:refs/heads/main` allows any FleetOps-V2 repo's main branch to assume the ECR push role. `repo:FleetOps-V2/fleetops-auth-service:ref:refs/heads/main` restricts it to exactly one repo. More restrictive = more secure.

---

## Docker Buildx and BuildKit â€” Layer Caching Internals

**BuildKit vs legacy Docker build:**

Legacy `docker build`:
- Sequential: Dockerfile instructions run one-by-one
- Single-platform only
- Limited caching (local only)

BuildKit (docker buildx):
- Parallel: independent stages run concurrently
- Multi-platform: can build for `linux/amd64` + `linux/arm64` simultaneously
- Advanced caching: `--cache-from`, `--cache-to` with multiple backends (local, inline, registry, gha)

**GHA (GitHub Actions) cache backend:**

```yaml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
    load: true
```

```
First run (cache miss):
  BuildKit builds all layers from scratch
  After build: exports all layer blobs to GitHub Actions cache
  (GitHub provides a cache API at runner level, backed by Azure Blob Storage)
  Cache key: based on Dockerfile + build context content hash

Second run (cache hit):
  BuildKit checks GitHub Actions cache API for matching layers
  Downloads cached layer blobs â†’ skips re-executing those Dockerfile instructions
  Only instructions with changed inputs are re-executed
  
Typical savings:
  Maven dependency layer (RUN mvn dependency:go-offline):
    If pom.xml hasn't changed â†’ this layer is a cache HIT â†’ no Maven download
    This is the most expensive step (~2-3 min) â†’ saved on every build that doesn't change deps
  
  Source compilation (COPY src/ + RUN mvn package):
    ALWAYS a cache MISS (source code changed with every commit)
    â†’ Maven recompiles (30-60s for a Spring Boot app)
```

**`mode=max` vs `mode=min`:**
- `mode=min`: caches only the final stage layers (smaller cache, less useful for multi-stage builds)
- `mode=max`: caches ALL stage layers including intermediate (build stage + runtime stage). For multi-stage builds, this means the Maven build stage is cached separately â€” saving the most time.

---

## Maven Cache in GitHub Actions

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'   # â† enables Maven cache
```

**What this does:**
```
actions/setup-java with cache: 'maven':
  1. Computes cache key: hash of all pom.xml files in the repo
     key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
  2. Looks up GitHub Actions cache for this key
  â†“
Cache HIT (pom.xml unchanged from last run):
  Downloads ~/.m2/repository from GitHub Actions cache
  Maven finds all dependencies locally â†’ no network downloads
  
Cache MISS (pom.xml changed â€” new/updated dependency):
  Maven downloads dependencies from Maven Central â†’ stores in ~/.m2/repository
  After job: uploads ~/.m2/repository to GitHub Actions cache under the new key
  
Savings: ~2-3 minutes of Maven Central downloads on every build
```

---

## workflow_run Event â€” How CD is Triggered

```yaml
# In each service's cd.yml (service-specific file in service repo):
on:
  workflow_run:
    workflows: ["CI Pipeline"]   # exact name of the CI workflow
    types: [completed]
    branches: [main, develop]
```

**How this works:**

```
CI workflow "CI Pipeline" completes on branch main
  â†“
GitHub: evaluates all workflows in the repo with trigger: workflow_run
  Found: cd.yml matches (workflow name "CI Pipeline", type completed, branch main)
  â†“
GitHub queues the CD workflow run
  - The workflow_run event payload contains: workflow_run.conclusion (success/failure)
  - The workflow_run event contains: workflow_run.head_sha (the commit SHA)
  â†“
CD workflow starts:
  if: ${{ github.event.workflow_run.conclusion == 'success' }}
  â†’ Only runs if CI passed (no deployment on CI failure)
  â†“
CD workflow uses the head_sha from the CI run to build the image tag:
  TAG=develop-${HEAD_SHA:0:7}
  (This matches exactly what the CI push job pushed to ECR)
```

**Why `workflow_run` instead of `on: push`:** The CD workflow needs to know the CI workflow succeeded AND needs the image tag from CI. `workflow_run` provides the conclusion and can access CI artifacts/outputs. A direct `push` trigger would race with CI â€” deploying before CI finishes.

---

## GitHub App Token Exchange Internals

The CD workflow uses a GitHub App to write to `fleetops-deployments` repo. A GitHub App token is more secure than a PAT.

**How the GitHub App token is generated:**

```yaml
- uses: tibdex/github-app-token@v2
  with:
    app_id: ${{ secrets.APP_ID }}
    private_key: ${{ secrets.APP_PRIVATE_KEY }}
```

**What this action does internally:**

```
Step 1: Create a JWT signed with the GitHub App's private key (RS256):
  {
    "iss": APP_ID,          // GitHub App ID (numeric)
    "iat": now,
    "exp": now + 600        // JWT valid for 10 minutes
  }
  Sign with APP_PRIVATE_KEY (RSA private key stored in GitHub Secret)
  â†’ App JWT

Step 2: Call GitHub API with App JWT:
  GET https://api.github.com/app/installations
  Authorization: Bearer {App JWT}
  â†’ Returns: installation_id (the ID for the org/user where the app is installed)

Step 3: Exchange for installation access token:
  POST https://api.github.com/app/installations/{installation_id}/access_tokens
  Authorization: Bearer {App JWT}
  Body: {"repositories": ["fleetops-deployments"], "permissions": {"contents": "write"}}
  â†’ Returns: installation_token (valid 1 hour, scoped to fleetops-deployments write only)

Step 4: Configure git with this token:
  git remote set-url origin https://x-access-token:{installation_token}@github.com/FleetOps-V2/fleetops-deployments.git
  git commit + git push â†’ authenticated with installation token
```

**Why GitHub App over PAT:**
- PAT: user-level token, inherits all user's permissions, never expires unless manually rotated
- GitHub App installation token: scoped to exactly the repos and permissions declared, expires in 1 hour, auditable (shows as "github-actions[bot]" in commit history), revokable independently of any user

---

## yq YAML Editing â€” How values-prod.yaml is Updated

```bash
yq e ".image.tag = \"${TAG}\"" -i charts/auth-service/values-prod.yaml
```

**What yq does:**
```yaml
# Before:
image:
  repository: 538661800892.dkr.ecr.us-east-1.amazonaws.com/fleetops-prod/auth-service
  tag: v1.2.3
  pullPolicy: IfNotPresent

# yq -e ".image.tag = \"v1.3.0\"" -i values-prod.yaml

# After:
image:
  repository: 538661800892.dkr.ecr.us-east-1.amazonaws.com/fleetops-prod/auth-service
  tag: v1.3.0
  pullPolicy: IfNotPresent
```

yq (`mikefarah/yq`) reads the YAML file, evaluates the expression (`.image.tag = "v1.3.0"`), writes the modified YAML back to the same file (`-i` = in-place). It preserves comments, ordering, and formatting (within YAML spec constraints).

**The git push with rebase:**
```bash
git pull --rebase origin main && git push
```
Why `--rebase`: If another service's CD workflow pushed to `fleetops-deployments` in the same second (race condition), `git push` would fail with "non-fast-forward". `git pull --rebase` re-applies the local commit on top of the latest main, then pushes cleanly. `--rebase` avoids a merge commit, keeping history linear.

---

## Trivy Image Scanning Internals

```
trivy-scan job:
  1. Download image tar from GitHub Actions artifact
  2. docker load < image.tar â†’ image loaded into runner's Docker daemon
  3. trivy image --severity HIGH,CRITICAL \
                  --ignore-unfixed \
                  --exit-code 1 \
                  --format sarif \
                  --output trivy-results.sarif \
                  538661800892.dkr.ecr.us-east-1.amazonaws.com/fleetops-dev/auth-service:develop-a3f92c1
```

**How Trivy detects CVEs:**

```
Trivy downloads vulnerability databases:
  - OS packages DB: NVD (NIST National Vulnerability Database) + distro-specific (Alpine secdb)
  - Language DB: GitHub Advisory Database (Java/Maven CVEs)
  Stored locally in ~/.cache/trivy/

Trivy scans the image layers:
  1. OS package scan: reads /lib/apk/db/installed (Alpine), /var/lib/dpkg/status (Debian)
     Compares package name+version against OS CVE database
     Example: libssl3 3.0.8-r0 â†’ CVE-2023-5678 HIGH (fixed in 3.0.8-r1)
     
  2. Application dependency scan: reads /app/app.jar
     Trivy unpacks the JAR â†’ reads BOOT-INF/lib/*.jar (Spring Boot fat JAR)
     Each nested JAR: name+version compared against GitHub Advisory DB
     Example: spring-webmvc-6.0.12.jar â†’ CVE-2024-xxxx HIGH
     
  --ignore-unfixed: if no fixed version exists in the OS/language ecosystem yet, skip
  (prevents failing builds for CVEs where no mitigation is available)
  
  --exit-code 1: if any HIGH or CRITICAL unfixed CVE found â†’ Trivy exits with code 1
  â†’ GitHub Actions step fails â†’ job fails â†’ push job never runs
```

**SARIF output:** Security Analysis Results Interchange Format. GitHub Security tab parses SARIF to show CVEs as Code Scanning alerts, linked to the exact file/dependency causing the issue.

---

## SonarQube Quality Gate â€” What Metrics Are Evaluated

```
mvn sonar:sonar -Dsonar.projectKey=fleetops-auth-service \
                -Dsonar.token=${SONAR_TOKEN} \
                -Dsonar.host.url=${SONAR_HOST_URL}
```

**What SonarQube collects:**

```
SonarQube Scanner analyzes:
  - Java source files (src/main/java/**/*.java)
  - Test coverage report from JaCoCo: target/site/jacoco/jacoco.xml
  - Compiled bytecode for bug detection

Metrics evaluated for the Quality Gate:
  1. Coverage: percentage of lines executed by tests
     Default threshold: Coverage on New Code >= 80%
     (FleetOps must have 80%+ of any new code covered by tests)
  
  2. Duplications: percentage of duplicated code blocks
     Default threshold: Duplicated Lines on New Code <= 3%
  
  3. Maintainability: code smells (naming, method length, complexity)
     A rating (best) to E rating (worst) for new code
     Default threshold: Maintainability Rating on New Code = A
  
  4. Reliability: bugs detected by static analysis
     Null pointer risks, resource leaks, incorrect API usage
     Default threshold: Reliability Rating on New Code = A (zero bugs)
  
  5. Security: security hotspots and vulnerabilities
     Hardcoded credentials, SQL injection risks, cryptographic issues
     Default threshold: Security Rating on New Code = A

Quality Gate result: PASSED (all metrics within thresholds) or FAILED (any metric outside)
```

**Quality gate check step:**
```bash
# Poll SonarQube API until the analysis task is complete
TASK_ID=$(cat .sonar/report-task.txt | grep ceTaskId | cut -d= -f2)
while true; do
  STATUS=$(curl -s -u ${SONAR_TOKEN}: ${SONAR_HOST_URL}/api/ce/task?id=${TASK_ID} | jq -r '.task.status')
  if [ "$STATUS" = "SUCCESS" ]; then break; fi
  sleep 5
done
# Check the quality gate result
QG=$(curl -s -u ${SONAR_TOKEN}: ${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=fleetops-auth-service | jq -r '.projectStatus.status')
if [ "$QG" != "OK" ]; then exit 1; fi
```

---

## Snyk OSS Scan â€” How It Checks Maven Dependencies

```
snyk test --severity-threshold=high --file=pom.xml
```

**What Snyk does:**

```
1. Parse pom.xml â†’ resolve full dependency tree (including transitive dependencies)
   Same as: mvn dependency:tree
   
2. Send dependency list to Snyk API:
   POST https://snyk.io/api/v1/test/maven
   Body: {dependencies: [{name: "org.springframework.boot:spring-boot-starter-web", version: "3.2.0"}, ...]}
   
3. Snyk API: looks up each dependency in Snyk's vulnerability database
   Database: aggregated from NVD, GitHub Advisory Database, npm/Maven advisories
   Returns: list of CVEs per dependency with severity level
   
4. --severity-threshold=high: only fail on HIGH or CRITICAL CVEs
   (MEDIUM and LOW are reported but don't block the build)
   
5. If HIGH CVEs found â†’ exit code 1 â†’ CI job fails â†’ image never built
```

**Snyk vs Trivy:** Snyk catches vulnerable Maven dependencies in `pom.xml` BEFORE the image is built (in the quality job). Trivy catches CVEs in the OS packages AND the final JAR layers AFTER the image is built (in the trivy-scan job). Two different layers of defense: Snyk = dependency audit, Trivy = image audit.

---

## ECR Internals â€” Image Lifecycle and Scanning

**Image push flow:**
```
CI push job:
  1. aws ecr get-login-password --region us-east-1 | docker login \
     --username AWS --password-stdin 538661800892.dkr.ecr.us-east-1.amazonaws.com
     â†’ ECR returns a temporary Docker registry password (12-hour token)
     â†’ docker login stores it in ~/.docker/config.json
  
  2. docker push 538661800892.dkr.ecr.us-east-1.amazonaws.com/fleetops-dev/auth-service:develop-a3f92c1
     â†’ Docker daemon: pushes layers not already in ECR (delta push)
     â†’ Each layer: Content-Addressed Storage (SHA256 hash) â†’ if same layer exists, skip upload
     â†’ Saves bandwidth: base image layers shared across all service images
  
  3. ECR stores image manifest + layer blobs in S3 (AWS managed)
     ECR basic scanning: automatic CVE scan on push (uses Clair scanner)
     Results visible in ECR console â†’ Security tab
```

**ECR lifecycle policy (defined in modules/ecr):**
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 develop images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["develop-"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "Keep all semver images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 20
      },
      "action": {"type": "expire"}
    }
  ]
}
```

Without lifecycle policies, ECR accumulates images indefinitely. At $0.10/GB/month, 100 Docker images Ã— 200MB = 20GB = $2/month â€” manageable but multiplied across 5 repos with frequent deploys it grows quickly.

---

## End-to-End CI/CD Flow â€” Complete Internal Trace

```
Developer: git push origin main (commit: "feat: add vehicle insurance expiry check")

--- GITHUB SIDE ---
t+0s:  GitHub receives push. Evaluates triggers:
       CI workflow: on.push.branches=[main] â†’ match â†’ queue CI run
       
t+5s:  GitHub-hosted runner (ubuntu-latest) starts, clones repo

--- JOB 1: prepare (t+30s) ---
       git fetch --tags --unshallow
       LAST_TAG=$(git describe --tags --abbrev=0)  # e.g., v1.3.2
       git log ${LAST_TAG}..HEAD --oneline --format="%s"
         â†’ "feat: add vehicle insurance expiry check"
       â†’ feat: prefix â†’ minor bump
       NEW_VERSION=v1.4.0
       FULL_IMAGE=538661800892.dkr.ecr.us-east-1.amazonaws.com/fleetops-prod/vehicle-service:v1.4.0
       Output: image-tag=v1.4.0, should-release=true

--- JOB 2: quality (parallel with prepare) ---
       Service container: postgres:15 starts on port 5432
       mvn -B clean verify â†’ compiles + runs all tests against real PostgreSQL
       mvn sonar:sonar â†’ uploads to SonarQube
       Wait for SonarQube quality gate â†’ PASSED
       snyk test â†’ no HIGH/CRITICAL CVEs â†’ PASSED

--- JOB 3: build (t+5m, needs: prepare, quality) ---
       docker buildx build --cache-from=type=gha --cache-to=type=gha,mode=max
         Stage 1 (builder): maven:21 â†’ dependency download (CACHE HIT from pom.xml hash)
                                     â†’ mvn package (CACHE MISS - source changed)
         Stage 2 (runtime): eclipse-temurin:21-jre-alpine (CACHE HIT - base unchanged)
                            COPY --from=builder app.jar â†’ built JAR
       docker save â†’ image.tar artifact (200MB uploaded to GitHub artifact storage)

--- JOB 4: trivy-scan (t+7m) ---
       docker load < image.tar
       trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ...
       â†’ 0 vulnerabilities found â†’ PASSED
       Upload SARIF to GitHub Security tab

--- JOB 5: push (t+9m) ---
       configure-aws-credentials (OIDC):
         GitHub OIDC endpoint â†’ JWT
         STS:AssumeRoleWithWebIdentity â†’ 1-hour AWS credentials for ECR push role
       aws ecr get-login-password | docker login
       docker push :v1.4.0
       â†’ ECR receives image layers â†’ stores in S3 backend

--- JOB 6: release (t+10m) ---
       gh release create v1.4.0 --generate-notes
       git tag v1.4.0 && git push origin v1.4.0
       â†’ GitHub Release created

--- JOB 7: notify (t+10m) ---
       Email: "CI Pipeline SUCCEEDED: vehicle-service v1.4.0 on main"

--- CD WORKFLOW triggered (t+10m, workflow_run: completed) ---
       if: CI conclusion == success â†’ true
       
       Dev deployment job:
         TAG=develop-${HEAD_SHA:0:7}  â† for develop. For main prod: TAG=v1.4.0
         
       Prod deployment job:
         tibdex/github-app-token â†’ RS256 JWT â†’ GitHub App token (write to deployments repo)
         git clone fleetops-deployments
         yq e ".image.tag = \"v1.4.0\"" -i charts/vehicle-service/values-prod.yaml
         git commit -m "chore(vehicle-service): bump prod image tag to v1.4.0 [skip ci]"
         git pull --rebase && git push â†’ commit in fleetops-deployments

--- ARGOCD SIDE (t+12m) ---
       ArgoCD polls every 3 minutes (or webhook if configured):
         Fetch latest fleetops-deployments git state
         Render charts/vehicle-service with values.yaml + values-prod.yaml (tag=v1.4.0)
         Diff: current Deployment image = v1.3.2, desired = v1.4.0 â†’ OutOfSync
         automated.selfHeal â†’ trigger sync
         
         Apply: kubectl apply (server-side apply, field manager=argocd)
         Deployment updated â†’ Deployment controller creates new ReplicaSet
         Rolling update begins (see K8s rolling update internals above)

--- KUBERNETES SIDE (t+13m) ---
       kubelet: imagePullPolicy=IfNotPresent â†’ check local cache for v1.4.0
       Cache MISS â†’ pull from ECR:
         EC2 node role â†’ ECR:BatchGetImage + ECR:GetDownloadUrlForLayer
         Layer download (only changed layers â€” base JRE layers already cached)
       Container starts â†’ JVM initializes â†’ Flyway runs (no new migrations â†’ instant)
       Startup probe: /actuator/health â†’ UP after ~20s
       Readiness probe passes â†’ ALB target group registration
       
--- SMOKE TEST (t+16m) ---
       curl --retry 5 --retry-delay 15 https://origin.fleetops.website/api/vehicles/health
       HTTP 200 â†’ deployment SUCCEEDED

Total time: ~16 minutes from git push to new pods serving production traffic
```

---

## Additional CI/CD Q&A

**Q: How does Docker know which layers to skip when pushing to ECR?**
A: Each Docker image layer has a content-addressed SHA256 digest. Before uploading, Docker calls ECR's `BatchCheckLayerAvailability` API with all layer digests. ECR returns which layers already exist. Docker only uploads the missing layers. For the FleetOps Java images: the base JRE layer (~150MB) is shared across all 4 services â€” it's only pushed once and then reused.

**Q: What is the difference between `docker build` and `docker buildx build`?**
A: `docker build` uses the legacy builder (non-BuildKit). `docker buildx build` uses BuildKit with parallel stage execution, advanced caching backends (GHA cache), and multi-platform support. For multi-stage Dockerfiles, BuildKit can execute independent stages in parallel (e.g., testing stage and linting stage running concurrently). In FleetOps, `build-push-action@v5` automatically uses Buildx.

**Q: Why does the CD workflow need a separate GitHub App instead of using GITHUB_TOKEN?**
A: `GITHUB_TOKEN` (the automatically provided token) only has write access to the repository that triggered the workflow â€” in this case, `fleetops-auth-service`. The CD workflow needs to push to `fleetops-deployments` â€” a completely different repository. `GITHUB_TOKEN` cannot be used cross-repository. Options: PAT (user-level, never expires) or GitHub App installation token (scoped, short-lived). GitHub App is preferred for production.

**Q: What triggers ArgoCD to sync faster than its 3-minute polling cycle?**
A: A webhook. GitHub can be configured to send a push event webhook to ArgoCD's API server (`https://argocd.fleetops.website/api/webhook`). When the CD workflow pushes the image tag update, GitHub fires the webhook to ArgoCD within seconds. ArgoCD immediately starts a sync cycle. This reduces the sync latency from 3 minutes to ~10 seconds. FleetOps does not currently have this webhook configured (it's a pending improvement).

**Q: How does the `environment: production` gate work in GitHub Actions?**
A: GitHub Environments can be configured with protection rules: required reviewers, deployment branches, wait timers. When a job has `environment: production`, GitHub checks if the protection rules are satisfied before running the job. If "Required reviewers" is set, the workflow pauses and emails the reviewers to approve. The job runs only after approval. This implements a manual approval gate for production deployments without any external tooling.

**Q: What happens if the CD workflow's `git push` fails due to a concurrent push?**
A: The `git pull --rebase && git push` pattern handles this. If another service's CD workflow pushes simultaneously, `git push` fails with "rejected: non-fast-forward". The step fails and the CD workflow fails. To make it more robust, a retry loop can be added:
```bash
for i in 1 2 3; do
  git pull --rebase origin main && git push && break || sleep $((i * 5))
done
```
This retries 3 times with increasing delay.

**Q: What is Checkov and what does it check in Terraform CI?**
A: Checkov is an open-source IaC static analysis tool. In the `terraform-apply.yml` workflow, it scans all `.tf` files against a database of security rules. Example checks:
- `CKV_AWS_18`: S3 bucket should have access logging enabled
- `CKV_AWS_260`: Security group should not allow ingress from 0.0.0.0/0 on port 22
- `CKV_AWS_111`: IAM policy should not allow wildcard * resource
Rules that are intentionally violated have `#checkov:skip=CKV_AWS_260:Reason` comments in the Terraform code. Checkov exits 1 if any unskipped failing check is found.

**Q: How does the Terraform OIDC role differ from the ECR push OIDC role?**
A: Two separate IAM roles with different permissions and trust policies:
- `ECR push role` (`AWS_ECR_ROLE_ARN`): trusted by service repos' CI workflows on main/develop branches. Permissions: ECR GetAuthorizationToken, BatchGetImage, PutImage â€” image push only.
- `Terraform role` (`AWS_ROLE_ARN`): trusted by the infra repo's workflows only. Permissions: broad AWS permissions (EC2, EKS, RDS, IAM, etc.) needed to create infrastructure. This role is more powerful and restricted to the infra repository.

**Q: Why are there both `java-ci-ecr.yml` (reusable workflow) and service-specific `ci.yml` files?**
A: The reusable workflow pattern (`.github/workflows/java-ci-ecr.yml` in `fleetops-github-workflows`) centralizes the build logic. Each service repo has a thin `ci.yml` that calls the reusable workflow with service-specific inputs (service name, SonarQube project key, ECR repo name). Benefits: one change to the build logic (e.g., add a new scanning tool) applies to all 5 services simultaneously. Without this pattern, each service would have a copy of the CI logic â€” diverging over time with inconsistent quality gates.



---

# PHASE 2 SUPPLEMENT â€” COST, SCALING, FAILURE MODES (All Services)

## Quick Cost Reference â€” Full Stack

| Service | Config | Monthly Cost (approx) |
|---------|--------|----------------------|
| EKS control plane | 1 cluster | $73 |
| EC2 nodes | m7i-flex.large Ã— 2 (min) | $120 |
| RDS PostgreSQL | db.t3.micro, 20GB | $15 |
| ElastiCache Redis | cache.t3.micro | $14 |
| EFS | ~10GB standard | $3 |
| NAT Gateway | 1, ~10GB/month | $36 |
| ALB | 1 shared, low LCU | $16 |
| CloudFront | low traffic | $1â€“5 |
| WAFv2 | 1 ACL, 2 rules | $8 |
| Route53 | 1 hosted zone | $0.50 |
| ACM | wildcard cert | $0 |
| S3 | state + vehicle docs | $2 |
| Secrets Manager | ~6 secrets | $2.40 |
| SSM Parameter Store | 4 params (free tier) | $0 |
| Lambda | 1 invocation/day | <$0.01 |
| EventBridge | 1 rule | <$0.01 |
| Step Functions | 6 state transitions/execution | <$1 |
| SNS | ~1000 emails/month | $0.10 |
| Bedrock | Nova Lite ~$0.003/1K tokens | Variable |
| ECR | 5 repos Ã— ~500MB | $0.25 |
| CloudTrail | management events | $0 (free tier) |
| CloudWatch | logs, metrics | ~$5 |
| KMS | 4 CMKs + API calls | $4 |
| DynamoDB | lock table, PAY_PER_REQUEST | <$0.01 |
| **Total estimate** | | **~$310â€“350/month** |

---

## SCALING â€” Every Service

**EKS:** Control plane scales automatically (AWS managed). Worker nodes: Cluster Autoscaler scales the ASG from min 2 to max 5 m7i-flex.large nodes. For higher scale: increase max, change instance type, add multiple node groups (spot vs on-demand). EKS supports up to 5,000 nodes per cluster.

**RDS:** Vertical scale: `terraform apply` with `db_instance_class = "db.t3.small"` triggers a maintenance window resize. Horizontal read scale: create read replicas, point read-heavy services at the replica endpoint. Production-grade: migrate to Aurora PostgreSQL (auto-scales storage, supports read replicas + Global Database for multi-region). Maximum connections on db.t3.micro: ~85.

**Redis:** Scale vertically: change `cache.t3.micro` to `cache.r6g.large`. Horizontal: enable Cluster Mode (sharding across multiple nodes). For session management: cluster mode + multi-AZ. Current single-node: adequate for caching fleet data at this scale.

**EFS:** Automatically scales â€” no provisioning needed. Standard Throughput mode scales with active usage. For heavy parallel I/O: switch to Provisioned Throughput. EFS can grow to petabytes. Multiple pods across multiple AZs can mount simultaneously (ReadWriteMany).

**S3:** Infinitely scalable, no configuration. Automatically handles 5,500 GET/HEAD requests per second per prefix, 3,500 PUT/POST/DELETE per second per prefix. For high throughput: use multiple key prefixes (avoid sequential key names that hash to the same partition).

**Lambda:** Scales concurrently: each invocation gets its own execution environment. Default concurrency limit: 1,000 per region (soft limit, can increase). For this daily-trigger use case: concurrency = 1. For high-frequency Lambda: configure Reserved Concurrency to guarantee capacity.

**SNS:** Scales automatically to millions of messages per second. No configuration needed. SQS subscribers: add SQS queue to spread processing load. Fan-out: one SNS topic â†’ many SQS queues, each processed independently.

**Step Functions:** Standard Workflows: up to 2,000 executions/second per account. Express Workflows: up to 100,000/second. Each execution is isolated. FleetOps uses Standard (default) â€” appropriate for long-running (days/weeks) approval workflows.

**Bedrock:** Managed by AWS. Throttling: Nova Lite has per-minute token limits (varies by account tier). For production: request limit increase via AWS support. For high concurrency: Bedrock on-demand pricing is per-request â€” no capacity planning needed.

**EventBridge:** Fully managed. Rules scale to millions of events. Rate rules: guaranteed delivery. For high volume: EventBridge Pipes or SQS buffering before Lambda.

**Secrets Manager:** Auto-scales. Rate limits: 10,000 GetSecretValue requests per second per account. ESO calls once per hour per secret â€” effectively no concern.

**SSM Parameter Store:** Rate limits: 40 GetParameter requests per second (standard). ESO calls once per hour. No concern.

**CloudWatch:** Auto-scales. Log ingestion: unlimited. Custom metrics: 150 per account free, then $0.30/metric/month. Alarms: 10 free, then $0.10/alarm/month.

**ECR:** Auto-scales. Image storage: $0.10/GB/month. Push/pull bandwidth within us-east-1: free.

**ArgoCD:** Single-instance Deployment in ArgoCD namespace. For large clusters (100+ apps): increase ArgoCD controller replicas and sharding. For FleetOps scale: default 1 replica is fine.

---

## FAILURE MODE â€” Every Service

**EFS â€” failure modes:**
- EFS becomes unavailable: maintenance-service pods fail file upload/download endpoints (500 error). Task CRUD operations continue normally (only media ops fail).
- NFS mount hangs: pod can get stuck waiting for I/O. Mitigation: mount option `timeo=600` (timeout) and `retrans=2`. Pod restart resolves.
- EFS access point misconfigured (wrong UID): pods get permission denied writing files. Diagnosis: `kubectl exec` into pod and `ls -la /var/www/fleetops/shared-media`.

**S3 â€” failure modes:**
- S3 unavailable: presigned URL generation still works (local computation) but the presigned URL itself fails when browser tries to PUT. User gets upload failure.
- KMS key disabled: `s3:PutObject` fails with `KMS key is disabled`. All existing objects can't be downloaded either. Mitigation: never disable KMS key for S3 bucket, use key deletion 30-day pending period.
- Bucket policy misconfigured: `403 Forbidden` on presigned URL PUT. Diagnosis: check bucket policy, CORS configuration (browser PUT requires CORS headers).

**Lambda â€” failure modes:**
- Lambda function timeout (15-min max): if vehicle-service or auth-service APIs are slow, Lambda times out. EventBridge retries 2 times by default. After 3 failures: Lambda DLQ (if configured).
- Cold start delay: first daily invocation may take 100-500ms extra. No user impact (async background job).
- Secrets Manager throttled: Lambda `GetSecretValue` fails â†’ Lambda can't authenticate to backend â†’ all alerts for that day missed. Mitigation: Lambda caches the secret for the execution duration (reads once on cold start, reuses on warm invocations).
- Lambda VPC vs non-VPC: FleetOps Lambda calls `origin.fleetops.website` (public URL). If Lambda is inside the VPC, it needs a NAT Gateway to reach the internet. If outside VPC (current config), it has internet access but cannot access VPC-internal resources. Trade-off is explicit.

**EventBridge â€” failure modes:**
- Rule disabled: Lambda never fires. Alerts are missed until rule is re-enabled. Mitigation: CloudWatch Alarm on Lambda invocation count dropping to 0.
- Lambda IAM permission missing: EventBridge fires but Lambda returns 403 `AccessDenied`. `aws lambda get-policy` shows if EventBridge has invoke permission. Terraform ensures `lambda:InvokeFunction` is granted.

**Step Functions â€” failure modes:**
- State machine definition invalid (JSON syntax error): `startExecution` throws `StateMachineDoesNotExist` or `InvalidDefinition`. Caught during Terraform apply (Step Functions validates definition). Request creation fails with 500.
- Execution timeout: Standard executions can run for 1 year. For approval workflows, the manager simply hasn't approved â€” the execution stays in `WAITING_FOR_TASK_TOKEN` state indefinitely until timeout. Configurable per state with `TimeoutSeconds`.
- Task token lost: if request_db loses the taskToken, the execution is permanently stuck. Mitigation: store taskToken in the Step Functions execution input/output (retrievable via `DescribeExecution`).

**Bedrock â€” failure modes:**
- Model unavailable: Bedrock returns 503. vehicle-service should implement a circuit breaker (Resilience4j) â€” on 3 consecutive Bedrock failures, short-circuit and return a "AI analysis temporarily unavailable" message.
- Token limit exceeded: Nova Lite has context window limits. Very long vehicle history prompts may fail with `ValidationException: Input too long`. Mitigation: truncate the prompt to the most recent N maintenance records.
- IRSA credentials expired: AWS SDK auto-refreshes credentials. If STS is unavailable for the refresh, the SDK retries with exponential backoff. No action needed.

**Secrets Manager â€” failure modes:**
- Throttling: `GetSecretValue` throttled (>10,000/s). Very unlikely for this project. ESO caches secrets and only re-fetches every hour.
- Secret deleted: ESO can no longer sync â†’ K8s Secret goes stale (last-synced value persists). Pods continue to use old value until restart. New pods may fail to start if ESO hasn't synced yet.
- KMS key unavailable: All `GetSecretValue` calls fail (SM uses KMS to decrypt). Critical failure â€” all secrets unavailable. Mitigation: never delete KMS keys for active secrets.

**ECR â€” failure modes:**
- ECR unavailable: new image pulls fail â†’ new pods fail to start. Existing pods continue running (already pulled). Mitigation: `imagePullPolicy: IfNotPresent` on EKS nodes (images cached after first pull).
- ECR image not found (tag deleted by lifecycle policy): rolling update fails â€” new pod can't pull image. Mitigation: lifecycle policy keeps enough `v*` tagged images. Never delete the currently-deployed tag.
- IAM role missing ECR permissions: pull fails with `401 Unauthorized`. Node IAM role needs `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`.

**CloudWatch â€” failure modes:**
- Fluent Bit pod crash: logs stop shipping. Application continues running. Logs buffer on node filesystem until Fluent Bit restarts (DaemonSet: immediately restarted by kubelet). Position file ensures no duplicates.
- CloudWatch API throttled: Fluent Bit backs off and retries. Log ingestion may be delayed by seconds.
- CloudWatch Alarm not triggering: check if alarm state is `INSUFFICIENT_DATA` (metric not being reported). Diagnosis: verify CloudWatch agent is running, check for IAM issues with `cloudwatch:PutMetricData`.

---

# PHASE 10 SUPPLEMENT â€” COMPLETE Q&A

## Top 100 General Evaluator Questions (Q48â€“Q100 + remaining)

**Q48: What is the difference between synchronous and asynchronous communication in this system?**
- Synchronous (blocking): HTTP calls between services (request-service â†’ vehicle-service to validate vehicleId). Caller waits for response. Simple but creates coupling â€” if vehicle-service is slow, request-service is slow.
- Asynchronous (non-blocking): SNS publish (maintenance-service â†’ SNS topic). Caller fires and continues â€” does not wait for email delivery. EventBridge â†’ Lambda trigger. Step Functions waiting for task token.
- Expert: In a production microservices system, you'd prefer async wherever possible. FleetOps mixes both deliberately: synchronous for data that's needed immediately (vehicle validation), async for side effects (alerts, workflows).

**Q49: What is the CAP theorem and how does FleetOps address it?**
- Basic: CAP = Consistency, Availability, Partition Tolerance. A distributed system can guarantee at most 2 of 3 during a network partition.
- Deeper: FleetOps is primarily AP (Available + Partition Tolerant). The Redis cache can return stale fleet data for up to 5 minutes if the DB is being updated â€” it chooses availability over strict consistency. PostgreSQL is strongly consistent within a single instance (ACID) but there's no cross-service transaction consistency.
- Expert: The eventual consistency gap is acknowledged: if vehicle-service updates a vehicle record and maintenance-service reads it 1 second later via its own API call, it may get the old value if caches haven't been invalidated. For the use cases here (fleet management, not financial transactions), eventual consistency is acceptable.

**Q50: How does the system handle a rolling EKS upgrade (Kubernetes version 1.31 â†’ 1.32)?**
- Basic: `terraform apply` with updated `eks_cluster_version`. EKS upgrades the control plane in-place (no downtime). Then node groups are upgraded by creating new nodes with the new AMI and draining old ones.
- Deeper: Control plane is upgraded first. Then node group upgrade: Terraform changes the `ami_type` or `launch_template` to the new version AMI, sets force update â†’ EKS creates new node with new K8s version, cordons the old node, drains pods (graceful termination), terminates old node. Services continue during this because HPA ensures enough replicas.
- Expert: Before upgrading: check all Helm chart compatibility with new K8s API versions (e.g., if a CRD uses a deprecated API, it breaks). Run `kubectl convert` or check Helm chart release notes. Upgrade strategy: control plane first, node groups after, always check AWS EKS release notes for deprecated APIs.

**Q51: What is the "thundering herd" problem and does FleetOps have it?**
- Vehicle-service has all pods starting simultaneously (fresh deployment). All pods have a cold Redis cache. Every request goes to PostgreSQL. This sudden DB hit is the thundering herd.
- Mitigation: Redis TTL is set so not all cache entries expire simultaneously (slight variation in write time staggers expiry). HPA warm-up stabilization window (5 min) prevents over-scaling during startup traffic. Also: `initialDelaySeconds: 30` on readiness probe means pods don't receive traffic until warmed up.

**Q52: Why does Spring Boot use Tomcat embedded rather than an external Tomcat server?**
- Embedded Tomcat eliminates the "it works on my machine" problem â€” the server is part of the artifact. The JAR is self-contained: `java -jar app.jar`. No separate server installation, no deployment step, no WAR file. Docker images are simpler. This is the standard Spring Boot production pattern.

**Q53: What is the difference between `kubectl apply` and `kubectl create`?**
- `create`: fails if resource already exists.
- `apply`: creates if absent, updates if present (declarative). ArgoCD exclusively uses server-side apply (`kubectl apply --server-side`) â€” it sends the desired state and the server reconciles.
- Expert: Server-side apply uses "field management" â€” ArgoCD "owns" the fields it sets. If you manually set a field ArgoCD also manages, ArgoCD's next sync overwrites it. `selfHeal: true` enforces this continuously.

**Q54: What is a Kubernetes Init Container and does FleetOps use one?**
- Init containers run before the main container and must complete successfully. Used for: waiting for a dependency (wait for DB to be ready), data seeding, file permission setup.
- FleetOps uses the `db-init` Job (wave 1) instead of init containers for DB creation. Could also use an init container in each service pod to verify `nc -z postgres-host 5432` before starting Spring Boot. Currently the startup probe handles this â€” if DB isn't ready, Spring Boot fails startup probe and kubelet retries.

**Q55: What is the Spring Boot `@PostConstruct` annotation and how is it used in FleetOps?**
- `@PostConstruct` methods run after Spring dependency injection completes but before the application starts serving requests. In FleetOps: `LambdaServiceInitializer.init()` is annotated `@PostConstruct`. It checks if the `lambda-service` user exists in auth_db, creates it if not. This runs once per pod startup. Idempotent â€” checks before inserting.

**Q56: Why does the CI pipeline use a service container for PostgreSQL instead of an in-memory H2 database?**
- H2 is not PostgreSQL. Flyway migrations may use PostgreSQL-specific SQL (e.g., `CREATE INDEX CONCURRENTLY`, PostgreSQL-specific data types). H2 might accept or reject these differently. JDBC URL formats differ. Spring Boot auto-configuration behaves differently with H2 vs PostgreSQL dialects.
- Real PostgreSQL in CI guarantees: if it works in CI, it works in prod. This caught real bugs â€” H2 would have silently accepted invalid migrations.

**Q57: What is the React SPA architecture and how does it interact with the backend?**
- React 18 + Vite + TypeScript: Vite builds a static bundle (HTML + hashed JS/CSS). At runtime, all routing is client-side (React Router). The browser downloads `index.html` once, then JavaScript handles navigation without page reloads.
- API calls: React components call the backend via `fetch()` or Axios with `Authorization: Bearer {token}`. No server-side rendering.
- Token storage: JWT stored in browser memory (React state) or `sessionStorage` â€” cleared on tab close. Alternatively `localStorage` â€” persists across sessions but vulnerable to XSS.

**Q58: What is Vite and why use it over Create React App?**
- Vite is a next-generation build tool using ES modules natively in development (no bundling â†’ instant hot module replacement). Build uses Rollup with tree shaking. CRA uses Webpack which is slower.
- TypeScript 5 support, CSS modules, ESBuild for fast transpilation. CRA is effectively unmaintained since 2023.

**Q59: How does the frontend handle 401 Unauthorized responses?**
- An Axios interceptor (or fetch wrapper) checks the response status. On 401: clear the JWT from memory, redirect to login page, display "Session expired" message.
- Why: JWT expires after 24 hours. The user may have left a tab open overnight. Silent redirect prevents the user from seeing a confusing raw error.

**Q60: What would you change if this were going to handle 10,000 concurrent users?**
- RDS: Move to Aurora PostgreSQL with read replicas. Enable RDS Proxy for connection pooling.
- Redis: Enable Cluster Mode, add Read Replicas.
- EKS: Increase node group max to 10-20 nodes. Add dedicated node groups (application + addons).
- ALB: Scale automatically â€” no config change needed.
- Bedrock: Request concurrency limit increase from AWS.
- HPA: Lower CPU target (50%) for more aggressive scale-up.
- CDN: CloudFront already handles this at edge â€” no change needed.
- Monitoring: Enable AWS X-Ray sampling at 100% for load testing, reduce to 5% for production.

---

## Top 50 AWS Questions (continued â€” Q61â€“Q110)

**Q61: What is AWS Shield Standard and does FleetOps use it?**
AWS Shield Standard is automatically enabled on all AWS accounts at no charge. It protects against layer 3/4 DDoS attacks (SYN floods, UDP floods, reflection attacks). CloudFront is a Shield Standard protected endpoint â€” global AWS infrastructure absorbs volumetric attacks before they reach the ALB. FleetOps does not explicitly configure Shield Standard (it's automatic). Shield Advanced ($3,000/month) adds DDoS response team, cost protection, layer 7 protection â€” not used in this project.

**Q62: What is a VPC Endpoint and should FleetOps use one?**
VPC Endpoints let resources in the VPC reach AWS services (S3, Secrets Manager, STS, Bedrock, etc.) without going through the internet or NAT Gateway. Currently EKS nodes reach these services via NAT Gateway â€” costs NAT bandwidth fees and adds latency.
- Gateway endpoint (S3, DynamoDB): free. Should be added.
- Interface endpoint (Secrets Manager, STS, ECR): ~$0.01/hour per endpoint (~$7/month). Reduces NAT costs and removes internet dependency.
- In this project: not implemented (cost optimization â€” NAT Gateway data transfer is minimal at this scale).

**Q63: What is Multi-AZ for RDS and why isn't it used here?**
Multi-AZ RDS creates a synchronous standby replica in a second AZ. AWS automatically fails over in 60-120 seconds if the primary becomes unavailable (DB engine crash, AZ failure). Cost: doubles RDS cost (~$30/month vs $15/month for db.t3.micro).
Not used in FleetOps: cost optimization for a training project. Production recommendation: enable Multi-AZ. Single-AZ means if the AZ goes down, the DB is unavailable and Spring Boot returns 503 until AWS recovers it (~5 min typically, up to 30 min in severe cases).

**Q64: What is RDS Proxy and when would you add it?**
RDS Proxy is a fully managed database proxy that sits between application pods and RDS. It pools connections at the proxy level (one pool shared across all app instances) instead of one HikariCP pool per pod. Benefits: handles connection bursts without exhausting max_connections, automatic failover (cuts failover time to <30s from 60-120s), IAM authentication support. Cost: 1/10 the price of the database ($1.50/month for t3.micro). Add when: connection exhaustion errors appear in logs, or enabling Multi-AZ.

**Q65: What is AWS CloudFront Functions vs Lambda@Edge?**
CloudFront Functions: JavaScript running at all 400+ edge PoPs, sub-millisecond execution, very limited runtime (no network calls, max 2MB code). Use for: URL rewrites, header manipulation, A/B testing.
Lambda@Edge: Full Lambda runtime at a subset of PoPs (regional edge caches), up to 30s execution, can make network calls. Use for: auth at the edge, request transformation, personalization.
FleetOps doesn't use either â€” WAF handles security filtering. If geo-blocking or auth at edge were needed, CloudFront Functions would be the starting point.

**Q66: What is the difference between KMS Decrypt and KMS GenerateDataKey?**
KMS uses envelope encryption: 
- `GenerateDataKey`: KMS creates a data encryption key (DEK). Returns plaintext DEK (for use) + encrypted DEK (for storage alongside the data). Service uses plaintext DEK to encrypt data locally. Discards plaintext, stores encrypted DEK with data.
- `Decrypt`: when reading data, service sends the encrypted DEK to KMS. KMS decrypts and returns the plaintext DEK. Service uses it to decrypt the data.
S3, RDS, EFS all use this pattern internally â€” you never call these APIs directly. IRSA role has `kms:Decrypt` to allow ESO to decrypt Secrets Manager values.

**Q67: What is AWS IAM Access Analyzer?**
Access Analyzer automatically identifies resources in your account that are shared with external entities (public, cross-account). Example: if the fleetops-prod-vehicle-docs S3 bucket accidentally got a public bucket policy, Access Analyzer would flag it immediately.
Not explicitly configured in FleetOps (`modules/config` uses AWS Config rules instead). Adding Access Analyzer is a security improvement.

**Q68: What are SQS, and why is it not used in FleetOps?**
SQS (Simple Queue Service) is a managed message queue for decoupling services. Messages are stored durably and consumed by one consumer at a time (point-to-point). SNS fan-out to SQS (SNS â†’ multiple SQS queues â†’ multiple consumers) is a common pattern.
FleetOps uses SNS â†’ email directly. No SQS because: there's no background processing consumer â€” alerts are fire-and-forget. Adding SQS would make sense if Lambda needed to process alerts reliably (retry on failure, DLQ for failed messages). A future improvement: SNS â†’ SQS â†’ Lambda (more reliable than SNS â†’ Lambda directly, enables DLQ).

**Q69: What is VPC Flow Logs and why is it not enabled?**
VPC Flow Logs captures every network flow (source IP, dest IP, port, protocol, bytes, packets, allow/reject) from ENIs in the VPC. Stored in CloudWatch or S3.
Not enabled in FleetOps: cost (PB of data for busy systems) and noise. Useful for security analysis (detecting unusual traffic patterns, port scans, data exfiltration). Production recommendation: enable for the EKS cluster SG at minimum, retain for 30 days in CloudWatch.

**Q70: What is the difference between CloudFront HTTPS with ALB HTTPS vs CloudFront HTTPS with ALB HTTP?**
CloudFront â†’ ALB HTTPS (end-to-end encryption): Traffic is encrypted in transit from browser to CloudFront edge AND from CloudFront to ALB. Requires ACM cert on ALB. More secure.
CloudFront â†’ ALB HTTP: Traffic is encrypted from browser to CloudFront. CloudFront to ALB is unencrypted. Still secure from attacker's perspective (CloudFront to ALB is within AWS backbone, not internet). Slightly simpler (no cert on ALB).
FleetOps uses HTTPS on both: CloudFront uses the ACM wildcard cert, and the ALB HTTPS listener uses the same cert (annotated in Ingress). Full end-to-end TLS.

**Q71: What is AWS Compute Optimizer and what would it recommend for FleetOps?**
Compute Optimizer analyzes CloudWatch metrics (CPU, memory, network) and recommends right-sizing for EC2 instances. For m7i-flex.large nodes: if pods consistently use < 2 vCPU, Compute Optimizer might recommend m7i-flex.medium (2 vCPU, 8GB, ~$0.03/hour cheaper). For db.t3.micro: if CPU consistently < 5%, it's right-sized; if > 80%, upgrade to db.t3.small.

**Q72: What is AWS Savings Plans vs Reserved Instances for this project?**
On-demand pricing for EKS nodes (m7i-flex.large): ~$0.19/hour = ~$137/month for 2 nodes.
1-year Compute Savings Plan: up to 37% discount = ~$86/month for 2 nodes. Saves $51/month.
Reserved Instance (3-year, all upfront): up to 57% off. Better savings but 3-year commitment.
For this training project: on-demand is appropriate. Production: 1-year Savings Plan after usage stabilizes.

**Q73: What is AWS CloudFormation and how does it compare to Terraform for this project?**
CloudFormation is AWS's native IaC service. YAML/JSON templates, managed by AWS, native integration with AWS services.
Terraform advantages over CloudFormation: multi-cloud support (not relevant here), more intuitive HCL syntax, better module ecosystem, Terraform state can be inspected/manipulated directly, Checkov supports both. 
CloudFormation advantages: no state file to manage (AWS manages it), native rollback on failure, StackSets for multi-account deployment.
FleetOps chose Terraform: better Kubernetes provider support (Helm, kubernetes providers), better module reuse patterns, team familiarity.

**Q74: What is the AWS Well-Architected Framework and which pillars does FleetOps address?**
5 pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization.
- Operational Excellence: GitOps (all changes auditable, reversible), CI/CD automation, structured logging to CloudWatch.
- Security: WAF, IRSA (no static credentials), KMS encryption for all data at rest, least-privilege IAM.
- Reliability: Multi-AZ subnets (EKS spans 2 AZs), HPA (auto-scaling), rolling deployments (zero downtime).
- Performance Efficiency: CloudFront CDN, Redis caching, Bedrock on-demand (no idle AI infrastructure).
- Cost Optimization: Single NAT Gateway, shared ALB, t3.micro for DB/Redis (right-sized for training scale), Lambda for infrequent tasks.

**Q75: What is Bedrock's token pricing for Nova Lite?**
Amazon Nova Lite pricing (us-east-1): ~$0.00006 per 1,000 input tokens, ~$0.00024 per 1,000 output tokens. A typical fleet analysis prompt: ~500 input tokens + ~500 output tokens â‰ˆ $0.00015 per analysis. If the AI endpoint is called 100 times per day: $0.015/day = $0.45/month. The Redis cache reduces this by 80-90% (repeated calls for same vehicle return cached response).

---

## Top 50 Kubernetes Questions (continued â€” Q76â€“Q125)

**Q76: What is a PersistentVolume (PV) and PersistentVolumeClaim (PVC)?**
PV: Cluster-scoped resource representing a piece of storage (EFS, EBS, NFS). It's the actual storage resource.
PVC: Namespace-scoped request for storage. Pod references the PVC. Kubernetes binds PVC to a matching PV.
FleetOps EFS: The EFS CSI driver creates a PV pointing to the EFS filesystem + access point. The maintenance-service Helm chart creates a PVC with `storageClass: efs-sc`. The driver binds them. Pod mounts PVC at `/var/www/fleetops/shared-media`.

**Q77: What is a StorageClass?**
StorageClass defines how storage is dynamically provisioned. The EFS CSI driver registers a StorageClass `efs-sc` with the cluster. When a PVC references this StorageClass, the EFS CSI driver automatically creates an EFS Access Point and PV. Without dynamic provisioning, PVs would need to be created manually.

**Q78: What is a Kubernetes Job vs a CronJob?**
Job: runs one or more pods to completion, then stops. Used for one-time tasks (database migrations, data imports). The `db-init` Job in ArgoCD wave 1 is a Kubernetes Job â€” it creates databases, then exits. 
CronJob: creates Jobs on a schedule (cron expression). FleetOps does not use a CronJob in K8s â€” instead EventBridge â†’ Lambda triggers the daily alert job outside Kubernetes. This is intentional: Lambda is more reliable (no node dependency), cheaper, and fully managed.

**Q79: What is a Kubernetes Namespace and what is isolated between namespaces?**
Namespace: a logical partition of cluster resources. Resource names must be unique within a namespace but can repeat across namespaces.
What IS isolated: names, RBAC scopes, NetworkPolicy (can scope to namespace), resource quotas, LimitRanges.
What is NOT isolated: nodes (pods from all namespaces share the same nodes â€” unless node affinity/taints), cluster DNS (cross-namespace calls work), network traffic (without NetworkPolicy, pods from any namespace can reach any pod).
FleetOps namespaces: `fleetops-prod` (app pods), `argocd` (ArgoCD), `kube-system` (cluster addons), `amazon-cloudwatch` (CloudWatch agent), `external-secrets` (ESO).

**Q80: What is a Kubernetes LimitRange?**
LimitRange sets default and maximum resource requests/limits for containers in a namespace. If a pod spec omits `resources.requests`, LimitRange applies defaults.
Without LimitRange: a pod with no resource requests can consume all node resources (starving other pods). The HPA also needs resource requests to calculate utilization percentages.
FleetOps: LimitRange not explicitly configured â€” each service's Helm chart specifies resources explicitly. Production: add LimitRange as a safety net.

**Q81: What is a Kubernetes ResourceQuota?**
ResourceQuota limits total resource consumption within a namespace. Example: `requests.cpu: 4000m` (namespace gets max 4 vCPU total). Prevents a single namespace from monopolizing cluster resources.
FleetOps: ResourceQuota not configured. Production multi-tenant: add quotas to prevent one team's services from starving another's.

**Q82: What is the difference between a Helm value override file (`values-prod.yaml`) and the base `values.yaml`?**
`values.yaml`: default values â€” work for any environment. Contains relative defaults (image tag: latest, replicas: 1).
`values-prod.yaml`: environment-specific overrides. Contains: prod image tag (v1.3.0), replicas: 2, prod RDS endpoint, prod EFS filesystem ID.
ArgoCD applies both: base `values.yaml` first, then `values-prod.yaml` overrides on top. Anything in `values-prod.yaml` wins.

**Q83: What happens if a pod is evicted?**
Eviction occurs when a node is under memory pressure. kubelet kills pods with `BestEffort` QoS (no resource requests set) first, then `Burstable` (requests < limits), then `Guaranteed` (requests == limits).
FleetOps pods: `Burstable` QoS (limits higher than requests). If the node OOMs, FleetOps pods may be evicted.
Evicted pod: rescheduled by the scheduler on another node. If all nodes are full, the pod goes Pending until Cluster Autoscaler adds a node.
Mitigation: set `requests == limits` for critical services (Guaranteed QoS, never evicted). Trade-off: less efficient bin packing.

**Q84: What is the difference between `kubectl delete pod` and `kubectl rollout restart`?**
`kubectl delete pod auth-service-abc123`: deletes one specific pod. ReplicaSet controller immediately creates a replacement. Useful for: forcing a specific pod to restart. Does NOT guarantee new pods use a new image.
`kubectl rollout restart deployment/auth-service`: triggers a rolling update with a new `kubectl.kubernetes.io/restartedAt` annotation. ALL pods are replaced in rolling update order (maxSurge/maxUnavailable). Used for: config reload, forcing a fresh IRSA token, applying updated Secrets (ESO just synced).

**Q85: What is the purpose of `kubernetes.io/cluster/{cluster-name}: owned` tag on subnets?**
EKS uses this tag to discover which subnets belong to the cluster. The AWS Load Balancer Controller uses it to auto-discover subnets for ALB placement: `kubernetes.io/role/elb=1` = public (internet-facing ALB), `kubernetes.io/role/internal-elb=1` = private (internal ALB). Without these tags, the LBC cannot create ALBs automatically and the Ingress would stay in a broken state.

**Q86: What is a Kubernetes Admission Controller?**
Admission controllers intercept API requests AFTER authentication and authorization but BEFORE objects are persisted to etcd. Types:
- Mutating: modify the request (e.g., inject sidecar containers, add default values)
- Validating: allow or reject based on custom rules

FleetOps uses admission controllers:
- EKS Pod Identity Webhook (MutatingAdmissionWebhook): injects IRSA env vars + projected token volume into pods with annotated ServiceAccounts
- ALB Controller's ValidatingWebhookConfiguration: validates Ingress resources before accepting

**Q87: What is `kubectl port-forward` and when would you use it for debugging?**
`kubectl port-forward pod/auth-service-abc123 8080:8080`: forwards local port 8080 to the pod's port 8080 via the Kubernetes API server tunnel. No ALB, no CloudFront â€” direct access to the pod. Used for: debugging a specific pod, testing actuator endpoints, checking DB connection from inside a pod.
`kubectl port-forward service/fleetops-auth-service 8080:8080`: load-balances across all service pods.

**Q88: What is the EKS node bootstrap script and what does it configure?**
EKS managed nodes run the EKS-optimized Amazon Linux 2 AMI. The AMI includes: containerd, kubelet, kubectl, the AWS VPC CNI DaemonSet. The bootstrap script (`/etc/eks/bootstrap.sh`) configures:
- Kubelet arguments (`--node-labels`, `--max-pods` based on instance type)
- Cluster endpoint (so kubelet knows where to register)
- CA data (for TLS verification of API server)
- `--container-runtime containerd` (replaces Docker daemon)

**Q89: What is the difference between horizontal and vertical pod autoscaling?**
HPA (Horizontal Pod Autoscaler): adds more pod replicas when load increases. Scale-out. Requires stateless pods (FleetOps pods are stateless â€” sessions in Redis/DB, not pod memory).
VPA (Vertical Pod Autoscaler): adjusts resource requests/limits based on actual usage. Scale-up. Pod must be restarted to apply new requests. Not installed in FleetOps â€” HPA is sufficient.
For stateful workloads (databases): vertical scaling is typically preferred (more resources on same instance) vs horizontal (data sharding is complex).

**Q90: What is a Pod Disruption Budget (PDB)?**
PDB limits how many pods can be voluntarily disrupted simultaneously. `minAvailable: 1` means Kubernetes won't drain/evict more pods than would leave fewer than 1 available.
Used by: Cluster Autoscaler scale-down (won't evict the last pod of a deployment), `kubectl drain` (respects PDB).
FleetOps: PDB not explicitly configured. With HPA `minReplicas: 2`, there are always 2 pods â€” the Cluster Autoscaler won't evict both simultaneously. Adding `minAvailable: 1` PDB would enforce this formally.

---

## Top 50 Terraform Questions (continued â€” Q91â€“Q140)

**Q91: What is `terraform output` and when is it useful?**
`terraform output` prints values declared in `outputs.tf` to stdout. Used in CI/CD:
```bash
CERT_ARN=$(terraform output -raw acm_certificate_arn)
EFS_ID=$(terraform output -raw efs_filesystem_id)
```
In `terraform-apply.yml`, outputs are extracted and written to `infra-values.yaml` in the deployments repo. Helm charts read this file to get infrastructure-specific values (ACM cert ARN for ALB annotation).

**Q92: What is a Terraform `data` source?**
Data sources read existing (pre-existing) AWS resources without creating them. Example in FleetOps:
```hcl
data "aws_lb" "fleetops" {
  tags = { "elbv2.k8s.aws/cluster" = module.eks_cluster.cluster_name }
}
```
After the ALB is created by the LBC controller, this data source fetches the ALB's DNS name to create Route53 alias records. It reads, never creates. `depends_on = [module.eks_addons]` ensures the ALB exists before the data source tries to read it.

**Q93: What is the Terraform `try()` function and why is it used in the providers?**
```hcl
host = try(module.eks_cluster.cluster_endpoint, "")
```
`try()` evaluates the expression and returns the fallback (`""`) if the first argument would cause an error (e.g., if `module.eks_cluster` hasn't run yet and the output doesn't exist).
This prevents Terraform from failing during `terraform init` or `terraform plan` before EKS exists. The helm/kubernetes providers need an endpoint, but during first apply the cluster doesn't exist yet â€” `try()` returns `""` and the provider skips validation.

**Q94: What is the Terraform `for_each` meta-argument?**
Creates multiple instances of a resource from a map or set. Example:
```hcl
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.fleetops.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  name    = each.value.name
  records = [each.value.record]
  type    = each.value.type
}
```
One `aws_route53_record` per domain validation option in the ACM certificate. More idiomatic than `count` because instances are addressed by key (`each.key`) not index â€” adding/removing a domain doesn't shift indices.

**Q95: What is `terraform validate` vs `terraform plan`?**
`validate`: static analysis only â€” checks syntax, type compatibility, variable references. Does NOT call AWS APIs. Fast (~1s). Run before plan to catch obvious errors.
`plan`: calls AWS APIs to get current state, evaluates the full diff. Requires valid AWS credentials. Much slower but gives complete picture.
In CI: `validate` runs first (fast gate), then `plan` (requires OIDC auth, slower).

**Q96: What is `terraform fmt` and why does CI enforce it?**
`terraform fmt` reformats .tf files to canonical HCL style (consistent indentation, aligned equals signs). `-check` mode exits non-zero if any file needs reformatting. In CI, this enforces consistent code style without needing code review comments. Pre-commit hooks can also run `terraform fmt` automatically before commit.

**Q97: What is `terraform taint` and when was it used?**
`terraform taint` (deprecated in Terraform 1.x, replaced by `-replace`) marks a resource for forced recreation on next apply. Use case: an EC2 instance is stuck in a bad state that Terraform's update can't fix, but a fresh instance would work. In Terraform 1.x: `terraform apply -replace=module.eks_nodegroup.aws_launch_template.main`.
FleetOps: would use this if a node group launch template gets corrupted and needs fresh recreation.

**Q98: What is the `random` provider used for in FleetOps?**
The `random` provider generates random values. Typical use: suffix for globally unique resource names:
```hcl
resource "random_string" "bucket_suffix" {
  length  = 8
  special = false
  upper   = false
}
resource "aws_s3_bucket" "state" {
  bucket = "fleetops-terraform-state-${random_string.bucket_suffix.result}"
}
```
S3 bucket names are globally unique â€” appending a random suffix prevents naming conflicts across accounts. The random value is stored in state and doesn't change on re-apply.

**Q99: What is the `archive` provider used for?**
The `archive` provider creates ZIP files from source directories. Used by the Lambda module:
```hcl
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/src"
  output_path = "${path.module}/lambda.zip"
}
resource "aws_lambda_function" "alert_processor" {
  filename      = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
}
```
`source_code_hash` ensures Lambda updates when source code changes (Terraform detects hash mismatch and triggers a Lambda update).

**Q100: What is `terraform state mv` and when would you use it?**
`terraform state mv` moves a resource from one state address to another without recreating it. Use cases:
- Rename a resource in code: `aws_security_group.alb` â†’ `aws_security_group.load_balancer`
- Move a resource into a module: `aws_vpc.main` â†’ `module.networking.aws_vpc.main`
- Split a module into sub-modules
Without `state mv`, Terraform would destroy the old resource and create a new one â€” catastrophic for resources like RDS.

**Q101: What is a Terraform `moved` block (Terraform 1.1+)?**
A safer alternative to `terraform state mv` â€” declarative in .tf code:
```hcl
moved {
  from = aws_security_group.alb
  to   = module.networking.aws_security_group.alb
}
```
This is committed to the repository and applied automatically on next `terraform apply`. Prevents teammates from accidentally recreating moved resources.

**Q102: What is `terraform refresh-only` vs the old `terraform refresh`?**
`terraform refresh` (deprecated) updated the state to match reality without showing changes. `terraform apply -refresh-only` achieves the same but shows a plan of state changes first (review before committing). Use when: someone manually modified an AWS resource and you want to accept the change into state without reverting it.
Example: manually added a tag to an RDS instance. `apply -refresh-only` accepts the tag in state. Next `terraform plan` will still show the tag as a diff to remove (because your .tf code doesn't have it) â€” you must also add it to .tf to make it stable.

**Q103: What is the `tls` provider used for in FleetOps?**
The `tls` provider can generate TLS certificates and keys. In the EKS OIDC module, it may be used to fetch the OIDC endpoint's TLS certificate thumbprint:
```hcl
data "tls_certificate" "eks_oidc" {
  url = module.eks_cluster.oidc_issuer_url
}
resource "aws_iam_openid_connect_provider" "eks" {
  thumbprint_list = [data.tls_certificate.eks_oidc.certificates[0].sha1_fingerprint]
}
```
The thumbprint tells IAM which CA issued the OIDC endpoint's certificate â€” used to verify JWT signatures.

**Q104: What is `terraform graph` and what does it show?**
`terraform graph | dot -Tsvg > graph.svg`: generates a DOT-format dependency graph visualization. Shows all resources as nodes, dependencies as directed edges. Useful for: understanding why an apply fails (which dependency is blocking), visualizing the module structure, identifying circular dependencies.
For FleetOps, the graph would show: kms â†’ (networking, rds, redis, s3, efs, secrets_manager) â†’ eks_cluster â†’ eks_oidc â†’ (iam, eks_nodegroup) â†’ eks_addons â†’ (lambda, cloudwatch, etc.).

**Q105: What happens during `terraform init` in CI?**
```
terraform init:
  1. Downloads providers (aws, helm, kubernetes, random, tls, archive) from Terraform Registry
     Cached in .terraform/providers/ (GitHub Actions can cache this directory)
  2. Downloads modules (all local modules are relative paths, no download needed)
  3. Configures the S3 backend:
     - Reads bucket/key/region from backend block
     - Authenticates with AWS (OIDC credentials from configure-aws-credentials)
     - Verifies bucket exists and is accessible
     - Checks/creates the state file if absent
  4. Generates .terraform.lock.hcl with provider version hashes
     (ensures reproducible builds â€” same versions every run)
```

---

## Top 50 CI/CD Questions (continued â€” Q106â€“Q155)

**Q106: What is a GitHub Actions composite action?**
A composite action bundles multiple steps into a reusable unit. The `configure-aws-credentials@v4` action is a composite action â€” it internally handles OIDC token fetching, STS call, and env var setting. FleetOps could extract common steps (like semantic version calculation) into a composite action stored in the `fleetops-github-workflows` repo. Currently, reusable workflows cover most of this.

**Q107: What is the GitHub Actions `concurrency` key?**
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false
```
If two pushes to the same branch trigger CI, `concurrency` decides whether the second run waits for the first (`cancel-in-progress: false`) or cancels the first (`cancel-in-progress: true`). 
For CI (build, test, push): `cancel-in-progress: true` â€” no point testing the old commit if a new one is ready.
For CD (deploy): `cancel-in-progress: false` â€” never cancel a deployment mid-way (could leave cluster in inconsistent state).

**Q108: What is the difference between `actions/checkout@v4` with `fetch-depth: 0` vs `fetch-depth: 1`?**
`fetch-depth: 1` (default): shallow clone, only the latest commit. Fast but no git history.
`fetch-depth: 0`: full history, all tags. Required for semantic versioning â€” `git describe --tags --abbrev=0` needs to find the last tag, which may be many commits back. The CI `prepare` job uses `fetch-depth: 0` to correctly calculate version bumps since the last tag.

**Q109: How does the GitHub Actions `needs` context work?**
```yaml
jobs:
  build:
    needs: [prepare, quality]
    steps:
    - name: Get tag from prepare job
      run: echo "TAG=${{ needs.prepare.outputs.image-tag }}"
```
`needs` creates a dependency and provides access to the outputs of the specified jobs via `needs.<job>.outputs.<output-name>`. The `build` job waits for both `prepare` and `quality` to succeed, then reads the `image-tag` output from `prepare`.

**Q110: What is `docker/setup-buildx-action` and why is it needed?**
Docker Buildx is not enabled by default on GitHub Actions runners. `docker/setup-buildx-action@v3` installs the Buildx driver (Docker containerd driver or docker-container driver). Required to use `docker/build-push-action@v5` with advanced features (GHA cache, multi-platform, BuildKit). Without it, only basic `docker build` works.

**Q111: What is the `GITHUB_OUTPUT` environment variable?**
The mechanism for passing data between steps in the same job:
```bash
echo "image-tag=${TAG}" >> $GITHUB_OUTPUT
```
Then in a later step or dependent job:
```yaml
run: echo "${{ steps.compute-tag.outputs.image-tag }}"
```
Old method (deprecated): `::set-output name=tag::value` (vulnerable to injection if value contained newlines). `GITHUB_OUTPUT` writes to a file, eliminating injection risk.

**Q112: How does the `workflow_run` download artifact mechanism work?**
CI workflow uploads the image tar as an artifact:
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: docker-image
    path: /tmp/image.tar
```
CD or trivy-scan job downloads it:
```yaml
- uses: actions/download-artifact@v4
  with:
    name: docker-image
    run-id: ${{ github.event.workflow_run.id }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
```
The `run-id` references the specific CI run that produced the artifact. `GITHUB_TOKEN` authenticates the download (artifacts are private to the repo).

**Q113: What is branch protection and how does it enforce the CI pipeline?**
Branch protection rules on the `main` branch:
- Require status checks to pass before merging: `CI Pipeline / quality`, `CI Pipeline / trivy-scan`
- Require branches to be up to date before merging
- Dismiss stale pull request approvals when new commits are pushed

This means a PR cannot be merged to `main` unless CI passes. Direct pushes to `main` are blocked. This enforces: all code on `main` has passed tests, SonarQube, and Trivy scan.

**Q114: What is a GitHub Environment and how is it different from a branch?**
A GitHub Environment (Settings â†’ Environments â†’ "production") is a named deployment target with optional protection rules:
- Required reviewers: specific users/teams must approve before the job runs
- Wait timer: delay N minutes after trigger before running
- Deployment branches: only allow deployments from specific branches
The CD workflow's `environment: production` job is gated by the production environment's protection rules. If "Required reviewers" is set, the job pauses and emails the reviewers.

**Q115: What is `actions/cache@v4` and how does it work?**
GitHub Actions cache stores files between workflow runs for the same branch. Cache key determines when the cache is used vs. refreshed:
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: maven-
```
On hit: downloads cached ~/.m2. On miss: `restore-keys: maven-` finds the most recent maven-* cache (partial match) â€” still useful, just slightly stale.
Cache is stored in GitHub's Azure Blob Storage backend. Max 10GB per repo, 7-day expiry on unused caches.

---

## Top 50 Security Questions (continued â€” Q116â€“Q165)

**Q116: What is BCrypt and why use strength 10?**
BCrypt is an adaptive password hashing function. Strength (work factor) 10 means 2^10 = 1,024 iterations of the internal KDF. This makes brute-force and dictionary attacks expensive.
At strength 10: hashing one password takes ~100ms on modern hardware. An attacker can only try 10 hashes/second per CPU core. At strength 12: ~400ms (more secure, but login takes 400ms).
BCrypt advantages over MD5/SHA: designed specifically for passwords, salted (no rainbow table attacks), adaptive (increase strength as hardware improves). Spring Security BCryptPasswordEncoder default strength is 10.

**Q117: What is JWT and why use HS256 vs RS256 for signing?**
JWT (JSON Web Token) is a compact, signed token format. Header.Payload.Signature.
HS256 (HMAC-SHA256): symmetric â€” uses one secret key for both signing and verification. Simple, fast. Requires every service that validates JWTs to have the secret.
RS256 (RSA-SHA256): asymmetric â€” private key signs, public key verifies. Services can verify tokens with the public key without access to the private key (more secure for multi-service architectures).
FleetOps uses HS256 with a shared `JWT_SECRET`. Acceptable for this architecture since all services are internal. Production improvement: use RS256 â€” auth-service holds the private key, other services verify with the public key (no secret sharing needed).

**Q118: What is XSS (Cross-Site Scripting) and how does FleetOps prevent it?**
XSS: attacker injects malicious JavaScript into the page. If JWT is in `localStorage`, the malicious script can steal it: `localStorage.getItem('token')`.
Prevention:
- ContentSecurityPolicy (CSP) header: whitelist which scripts are allowed to run
- Spring Boot response headers: `X-XSS-Protection`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`
- React's JSX: automatically escapes dynamic values (`{userInput}` is rendered as text, not HTML)
- JWT in httpOnly cookie: inaccessible to JavaScript
FleetOps security backlog item: move JWT from localStorage to httpOnly cookie.

**Q119: What is CSRF (Cross-Site Request Forgery) and does FleetOps have it?**
CSRF: malicious website tricks the user's browser into making an authenticated request to fleetops.website.
FleetOps mitigation: JWT in Authorization header. Browsers never automatically send Authorization headers (they auto-send cookies). So CSRF is not applicable to JWT-based auth. If FleetOps moved to httpOnly cookie, CSRF protection (SameSite=Strict, CSRF token) would be needed.

**Q120: What is SQL injection and how does Spring JPA prevent it?**
SQL injection: attacker injects SQL code via user input: `' OR '1'='1`.
Spring Data JPA with parameterized queries:
```java
// SAFE (Spring JPA)
vehicleRepository.findByLicensePlate(licensePlate);  
// Generates: SELECT * FROM vehicles WHERE license_plate = ? with params

// UNSAFE (would be):
entityManager.createNativeQuery("SELECT * FROM vehicles WHERE plate = '" + plate + "'");
```
JPA uses PreparedStatements automatically â€” values are always parameters, never interpolated into the SQL string. Injection is impossible.

**Q121: What is HTTPS and why is HTTP disabled entirely in FleetOps?**
HTTPS = HTTP over TLS. Encrypts the connection between browser and server. Without HTTPS:
- Passwords sent in plaintext (anyone on the network can see them)
- JWT tokens interceptable (session hijacking)
- Man-in-the-middle attacks possible

FleetOps: CloudFront enforces HTTPS-only. The ALB listener redirect (80 â†’ 443) via `ssl-redirect: "443"` annotation. Spring Boot itself doesn't need SSL termination (CloudFront/ALB handle it).

**Q122: What is a security group's "stateful" behavior and why does it matter?**
Stateful: if you allow inbound traffic on port 8080, the reply traffic is automatically allowed outbound â€” you don't need an explicit outbound rule for the return traffic.
Example: ALB SG allows inbound 443 from 0.0.0.0/0. When ALB makes a connection to a pod on 8080, the pod's response (on ephemeral ports 1024-65535) is automatically allowed back through the ALB SG.
Contrast with NACLs (Network Access Control Lists): stateless â€” must explicitly allow both directions.

**Q123: What is container image scanning and what does Trivy find that Snyk misses?**
Snyk (in CI quality job): scans Maven pom.xml for dependency CVEs. Only checks Java dependencies declared in pom.xml.
Trivy (in CI trivy-scan job): scans the BUILT IMAGE. Finds:
- OS-level CVEs: vulnerabilities in Alpine Linux packages (openssl, glibc, musl) â€” not detectable from pom.xml
- All JAR CVEs: including transitive dependencies not explicitly in pom.xml
- Java runtime CVEs: JDK/JRE vulnerabilities in the base image
Trivy catches what Snyk misses: OS packages and base image vulnerabilities.

**Q124: What is the principle of least privilege and how is it implemented in FleetOps?**
Least privilege: grant only the minimum permissions required.
- IRSA `fleetops-app` role: `secretsmanager:GetSecretValue` only on `fleetops/*` ARNs (not all secrets in the account)
- SNS publish: only on `arn:aws:sns:us-east-1:538661800892:fleetops-*` (not all topics)
- S3: only `fleetops-prod-vehicle-docs` bucket (not `s3:*`)
- ECR push role: only `ecr:PutImage`, `ecr:InitiateLayerUpload` etc (not `iam:*` or `ec2:*`)
Known violation: `fleetops-app` ServiceAccount is shared across all services â€” auth-service has Bedrock permissions it doesn't need. Fix: separate IRSA roles per service.

**Q125: What is the AWS Security Token Service (STS) and when does FleetOps call it?**
STS issues temporary security credentials. FleetOps calls STS in three places:
1. IRSA: pods call `sts:AssumeRoleWithWebIdentity` to get 1-hour credentials for AWS SDK calls
2. GitHub Actions OIDC: CI/CD runner calls `sts:AssumeRoleWithWebIdentity` to get credentials for ECR push or Terraform apply
3. kubectl auth: GitHub Actions runs `aws eks get-token` which calls `sts:GetCallerIdentity` to generate a pre-signed STS URL used as a K8s bearer token
All three use OIDC identity tokens â€” no static credentials anywhere.

**Q126: What is AWS Config's `restricted-ssh` rule?**
AWS Config Managed Rule: checks that no security group allows unrestricted SSH access (port 22, 0.0.0.0/0). FleetOps EKS nodes never have port 22 open (managed nodes, no SSH). AWS Config would report these as COMPLIANT.
Access to EKS nodes: use AWS Systems Manager Session Manager (SSM) â€” creates a secure shell session through the SSM agent without port 22. The node IAM role has `AmazonSSMManagedInstanceCore` policy for this.

**Q127: What would a security penetration tester look for in FleetOps?**
- JWT localStorage storage: XSS vulnerability
- Shared `fleetops-app` IRSA role: over-privileged pods
- Plaintext secrets in `terraform.tfvars`: plaintext db_password, jwt_secret (security backlog)
- Single NAT Gateway: not HA but not a security issue
- Single-AZ RDS: availability issue, not security
- No WAF logging: WAF blocks requests but doesn't log blocked requests to CloudWatch (improvement: enable WAF logging to S3)
- No VPC Flow Logs: can't investigate network anomalies post-incident
- ArgoCD exposed publicly (argocd.fleetops.website): properly secured with admin password, but publicly reachable (improvement: put behind VPN or restrict to office IPs)

---

## Top 50 Project-Specific Questions (continued â€” Q128â€“Q177)

**Q128: How does vehicle-service know the maintenance history when calling Bedrock?**
vehicle-service makes an internal HTTP call to maintenance-service to fetch maintenance records for that vehicle before building the Bedrock prompt:
```java
List<MaintenanceTask> history = maintenanceServiceClient.getTasksByVehicleId(vehicleId);
String prompt = buildPrompt(vehicle, history);
bedrockClient.converse(buildConverseRequest(prompt));
```
This is a synchronous service-to-service call using the vehicle's Bearer token (or a service account token) for auth. The call goes through CoreDNS â†’ ClusterIP â†’ maintenance-service pod (not through the ALB or internet).

**Q129: What is the `LambdaServiceInitializer` and why is it needed?**
Lambda cannot access Kubernetes-internal services (it's outside the cluster). It must call the auth-service via the public URL (`origin.fleetops.website`). To authenticate, it needs a service account in auth-service.
`LambdaServiceInitializer` is a Spring `@Component` with `@PostConstruct`. On every auth-service pod startup:
1. Check: `SELECT COUNT(*) FROM users WHERE username = 'lambda-service'`
2. If 0: create the user with role TECHNICIAN and password from `LAMBDA_SERVICE_PASSWORD` env var
3. If exists: skip
This is idempotent â€” no-op if the user already exists.

**Q130: Why is the db-init Job in sync wave 1 and not wave 0?**
Wave 0: platform (namespace, RBAC, ServiceAccount) and secrets (ExternalSecrets) must exist first.
The db-init Job needs:
- The `fleetops-prod` namespace (wave 0) to run in
- The `fleetops-postgres-secret` K8s Secret (wave 0) for DB credentials
- PostgreSQL must be reachable (RDS always reachable â€” not an ArgoCD concern)
Wave 1 ensures namespace and secrets exist before db-init tries to connect and create databases. If wave 0 is skipped, db-init can't read the secret and fails.

**Q131: How does the frontend React app make authenticated API calls?**
```typescript
// axios interceptor pattern (in api.ts or a custom hook)
const axiosInstance = axios.create({
  baseURL: 'https://fleetops.website',
});

axiosInstance.interceptors.request.use((config) => {
  const token = getToken(); // from React state / sessionStorage
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

axiosInstance.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      clearToken();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```
Every API call automatically has the JWT. 401 responses redirect to login.

**Q132: What is the difference between `ROLE_MANAGER` and `MANAGER` in Spring Security?**
Spring Security convention: `GrantedAuthority` objects prefixed with `ROLE_` are treated as roles. When you use `hasRole('MANAGER')`, Spring Security checks for authority `ROLE_MANAGER`.
In the JWT, roles are stored as: `["ROLE_MANAGER", "ROLE_TECHNICIAN"]`. When Spring Security parses the JWT and creates `SimpleGrantedAuthority` objects, the `ROLE_` prefix is already there.
`@PreAuthorize("hasRole('MANAGER')")` â†’ Spring looks for `ROLE_MANAGER` âœ“
`@PreAuthorize("hasAuthority('ROLE_MANAGER')")` â†’ also works (more explicit)

**Q133: How does the CloudFront distribution know to NOT cache `/api/*` responses?**
In `modules/cloudfront`, the distribution has a cache behavior:
```hcl
ordered_cache_behavior {
  path_pattern = "/api/*"
  forwarded_values {
    query_string = true
    headers      = ["Authorization", "Content-Type"]
  }
  min_ttl     = 0
  default_ttl = 0
  max_ttl     = 0
  viewer_protocol_policy = "redirect-to-https"
}
```
TTL of 0 = never cache. The Authorization header is forwarded to the origin (ALB) â€” if CloudFront cached API responses, it would cache per Authorization header (one cache entry per user), which is both expensive and incorrect.

**Q134: What is the ArgoCD `repoURL` and how does ArgoCD authenticate to it?**
`repoURL: https://github.com/FleetOps-V2/fleetops-deployments.git`
ArgoCD authenticates using the secret `argocd-repo-fleetops-deployments` created by Terraform (`modules/eks/addons`). This secret contains the GitHub PAT stored in Secrets Manager. Type: `repository` secret, which ArgoCD recognizes and uses for Git operations:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-repo-fleetops-deployments
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
data:
  type: Z2l0  # "git" base64
  url: aHR0cHM6Ly9naXRodWIuY29tL0ZsZWV0T3BzLVYyL2ZsZWV0b3BzLWRlcGxveW1lbnRzLmdpdA==
  username: dXNlcm5hbWU=  # "username"
  password: <PAT base64>
```

**Q135: What is Helm's `helm template` command and how does ArgoCD use it?**
`helm template release-name ./chart -f values.yaml -f values-prod.yaml` renders the chart templates (Go templates) to plain Kubernetes YAML. ArgoCD runs this internally (via the Helm library) before comparing rendered YAML to live cluster state. This is why ArgoCD can do diffing even for Helm-deployed apps â€” it always works with rendered YAML, not Helm release state.

**Q136: Why does FleetOps not use ArgoCD's Application health for RDS/Redis?**
ArgoCD only manages Kubernetes resources. It has no concept of AWS resources. RDS, Redis, EFS health are checked by:
- Spring Boot Actuator: `/actuator/health` checks DB and Redis from inside the pod
- ALB health check: uses `/actuator/health` to determine pod readiness
- CloudWatch Alarms: monitors RDS CPU, Redis memory directly
ArgoCD sees the pods as Healthy (Deployment has desired replicas Ready). The DB health is implicit â€” if DB is down, pods fail startup probes and are not Ready, so ArgoCD shows Deployment as Degraded.

**Q137: What is the `@Scheduled` annotation in maintenance-service and how does it work?**
```java
@Scheduled(cron="0 0 8 * * ?")  // 8am every day, server timezone
public void scanAndBroadcastAlerts() {
    alarmBroadcastService.scanAndBroadcast();
}
```
Spring's `@EnableScheduling` starts a scheduled task executor. `@Scheduled` registers the method in the executor's cron scheduler. The pod where this task runs depends on which pod the scheduler happens to be on.
Critical issue with replicas: with 2+ maintenance-service replicas, BOTH pods will execute this cron at 8am. This causes duplicate SNS alerts!
Mitigation: implement distributed locking (ShedLock, using the DB as a lock store), or use a Kubernetes CronJob (only one pod runs), or use the EventBridge Lambda approach (external scheduler triggers one Lambda).

**Q138: How does the infra-values.yaml bridge Terraform and ArgoCD?**
Chicken-and-egg problem: ArgoCD needs the ACM cert ARN for the Ingress annotation, but ACM cert ARN is only known after `terraform apply`. Solution:
1. Terraform applies â†’ extracts outputs: `terraform output -raw acm_certificate_arn`
2. Terraform CI writes `environments/prod/infra-values.yaml` with these values
3. Terraform CI commits and pushes this file to `fleetops-deployments` repo
4. ArgoCD syncs `fleetops-deployments` â†’ reads `infra-values.yaml` as an additional Helm values file
5. Helm chart substitutes `{{ .Values.infrastructure.acmCertificateArn }}` into the Ingress annotation

**Q139: What is the `allowEmpty: false` argocd syncOption?**
If a Helm chart renders to ZERO resources (e.g., all resources gated by `if .Values.enabled` and all are false), ArgoCD by default would apply the empty set â€” effectively deleting everything. `allowEmpty: false` prevents this: ArgoCD refuses to sync if the rendered output is empty. This catches template errors before they cause an outage.

**Q140: What monitoring would immediately alert you if a deployment went bad?**
1. ALB 5xx error rate CloudWatch Alarm: if new pods are crashing, ALB gets 502 (bad gateway) or 503 (service unavailable). Alarm fires within 5 minutes.
2. Spring Boot Actuator + ALB health check: pods failing health checks are removed from target group â†’ ALB continues routing to healthy pods. If ALL pods fail â†’ ALB 503 â†’ CloudWatch alarm.
3. CD workflow smoke test: 90s after deployment, curl the health endpoint. `HTTP 5xx` or connection refused fails the CD job and sends a notification.
4. ArgoCD sync status: if pods never reach Healthy state, ArgoCD shows app as Degraded.

**Q141: How would you implement blue-green deployment for FleetOps instead of rolling?**
Current: rolling (old and new pods mix during update).
Blue-Green: two identical environments (blue = current, green = new).
Implementation with ArgoCD + ALB:
1. Deploy green version alongside blue (separate Deployment with `-green` suffix)
2. Test green with a subset of traffic or internal testing
3. Switch ALB target group from blue â†’ green (instant switchover, zero overlap)
4. If green is healthy: delete blue
ArgoCD doesn't natively support blue-green â€” need Argo Rollouts (a separate project). FleetOps uses rolling deploy (simpler, adequate for this use case).

**Q142: What is the `@CacheEvict` annotation and when does it fire in FleetOps?**
```java
@CacheEvict(value = "vehicles", allEntries = true)
public Vehicle createVehicle(CreateVehicleRequest request) { ... }
```
When `createVehicle()` returns (after the vehicle is persisted to DB), Spring's caching proxy evicts ALL entries in the "vehicles" cache from Redis. This ensures the fleet list cache is invalidated whenever a vehicle is added/updated/deleted â€” next GET reads fresh data from DB.
Cache eviction is synchronous with the method â€” the vehicle is in the DB AND the cache is cleared before the HTTP response is returned to the client.

**Q143: What is the `@Transactional` annotation and is it used in FleetOps?**
`@Transactional` wraps a method in a database transaction. If the method throws an exception, the transaction is rolled back (all DB changes undone).
In FleetOps request-service: `createRequest()` does two things:
1. INSERT into request_db
2. Call StepFunctions.startExecution()
If Step Functions call fails, the INSERT should be rolled back. With `@Transactional`: if an unchecked exception escapes, the INSERT is rolled back. Without it: INSERT is permanent even if Step Functions fails.
Problem: AWS SDK calls inside @Transactional are not part of the DB transaction â€” Step Functions is a separate system. Solution: use the Saga pattern (compensating transactions) or event-driven approach.

**Q144: What is Spring Boot's `spring.jpa.hibernate.ddl-auto` and why is it set to `validate`?**
`ddl-auto` controls what Hibernate does with the database schema on startup:
- `create`: drops and recreates schema every start (destroys data â€” NEVER in production)
- `update`: tries to ALTER schema to match entities (dangerous â€” can lose data)
- `validate`: checks that DB schema matches entity mappings, throws error if mismatch (CORRECT for production with Flyway)
- `none`: Hibernate doesn't touch schema

FleetOps uses `validate`. Flyway manages schema creation/migration. Hibernate validates that what Flyway created matches the JPA entity definitions. If they mismatch â†’ startup fails (caught early, before users see errors).

**Q145: How does CORS work in FleetOps and where is it configured?**
CORS (Cross-Origin Resource Sharing) allows the React frontend (served from `https://fleetops.website`) to call the backend API (also `https://fleetops.website` â€” same origin, no CORS issue!).
But during local development: frontend runs on `http://localhost:3000`, backend on `http://localhost:8080` â€” different origins â†’ CORS required.
Spring Boot CORS config:
```java
@Configuration
public class CorsConfig {
    @Value("${cors.allowed-origins}")  // from ConfigMap: CORS_ALLOWED_ORIGINS
    private String allowedOrigins;
    
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins(allowedOrigins.split(","))
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .allowedHeaders("Authorization", "Content-Type");
            }
        };
    }
}
```
`CORS_ALLOWED_ORIGINS` is injected from the `fleetops-config` ConfigMap, set by Terraform via SSM to `https://fleetops.website`.

**Q146: What is Helm's `{{ .Release.Name }}` and how are resource names generated?**
Every resource in a Helm chart uses the release name to avoid naming conflicts:
```yaml
metadata:
  name: {{ .Release.Name }}-auth-service
  # When ArgoCD deploys with release name "fleetops-auth-prod":
  # â†’ fleetops-auth-prod-auth-service
```
The ArgoCD Application's `releaseName` field sets this. If not set, defaults to the application name. This allows multiple instances of the same chart in different namespaces (e.g., dev and prod) without name collisions.

**Q147: What is the difference between ArgoCD's `Synced` and `Healthy` status?**
`Synced`: the live cluster state matches what's in Git (ArgoCD applied it successfully).
`Healthy`: the resources are functioning correctly (Deployment has desired replicas Ready, Ingress has ALB assigned, etc.).
These are independent:
- `Synced + Healthy`: deployment succeeded âœ“
- `OutOfSync + Healthy`: cluster was manually changed (selfHeal will fix it)
- `Synced + Degraded`: deployment applied but pods not starting (check pod logs)
- `OutOfSync + Degraded`: deployment pending AND pods failing (worst case)

**Q148: What is the `sync-wave` annotation and who processes it?**
`argocd.argoproj.io/sync-wave: "3"` is processed by ArgoCD's Application controller. It's not a standard Kubernetes annotation â€” ArgoCD reads it from the Application resource before syncing. Resources with lower wave numbers are applied first and must reach Healthy status before higher wave resources are applied.
The annotation is on the ArgoCD Application resources (`argocd/apps/prod/*.yaml`), not on the K8s resources themselves.

**Q149: What happens if a Helm chart template has a rendering error?**
ArgoCD runs `helm template` internally. If template rendering fails (e.g., accessing a nil value in a template), ArgoCD reports the Application as `SyncError` with the template error. No resources are applied. The error is visible in the ArgoCD UI and CLI.
Example error: `error calling include: template: chart/templates/deployment.yaml:42:22: executing "chart/templates/deployment.yaml" at <.Values.image.tag>: nil pointer evaluating interface {}.tag`
Fix: ensure `values-prod.yaml` has the required key.

**Q150: How does the maintenance-service EFS mount survive a pod restart?**
PersistentVolume and PersistentVolumeClaim are cluster resources (not pod resources). When the pod is deleted and recreated:
1. New pod spec references the same PVC by name
2. Kubernetes binds the same PVC to the new pod
3. kubelet mounts the same EFS volume at the same path
4. Files written by the old pod are still there

EFS is an NFS filesystem â€” data persists independently of pods. The NFS mount is re-established on each pod start. Existing files are immediately accessible.

**Q151: What is `fleetops.website` vs `www.fleetops.website`?**
The hosted zone is `fleetops.website` (apex/root domain). Route53 has an A record (Alias) at the apex â†’ CloudFront. There is no `www.` record configured. Users must access `https://fleetops.website` directly.
Production improvement: add `www.fleetops.website` CNAME â†’ `fleetops.website`, or CloudFront alternate domain names include `www.fleetops.website` with a redirect to the apex.

**Q152: How would you add database encryption in transit to FleetOps?**
RDS PostgreSQL supports SSL. JDBC URL change:
```properties
spring.datasource.url=jdbc:postgresql://${POSTGRES_HOST}:5432/auth_db?ssl=true&sslmode=require
```
This forces TLS on the JDBC connection. RDS provides a CA certificate bundle â€” the JVM truststore must include it (or `sslmode=verify-full` for certificate validation).
Currently: traffic between EKS pods and RDS is within the VPC (private subnet) but not encrypted in transit. Security group restriction provides network-level isolation, but not encryption. Adding TLS in transit is a security improvement.

**Q153: What is the Step Functions `Express Workflow` vs `Standard Workflow`?**
Standard: exactly-once execution, up to 1-year duration, execution history stored for 90 days. Used for: long-running workflows (approval processes, business transactions). Cost: $0.025/1,000 state transitions.
Express: at-least-once execution (idempotency required), max 5-minute duration, high throughput (100K/s). Used for: IoT event processing, streaming data. Cost: much cheaper per invocation.
FleetOps uses Standard: service request workflows may take days (waiting for manager approval). Exactly-once is critical â€” don't start two executions for one request.

**Q154: How does `kubectl rollout status` work and how is it used in CI?**
`kubectl rollout status deployment/fleetops-auth-service --timeout=120s`:
- Watches the Deployment's rollout progress
- Exits 0 when: all pods are updated AND ready (readiness probe passing)
- Exits 1 if timeout exceeded
Used in CI/CD post-deploy verification:
```bash
kubectl rollout status deployment/fleetops-auth-service -n fleetops-prod --timeout=180s
```
If the new pods fail to become Ready within 180s (DB down, image pull failure, startup probe timeout), this command fails and the CD workflow fails, alerting the team.

**Q155: What is the `@Api` annotation in Spring Boot and does FleetOps document its APIs?**
`@Api`, `@ApiOperation`, `@ApiResponse` are Springfox/Swagger annotations for auto-generating OpenAPI documentation. When present, Spring Boot serves an OpenAPI spec at `/v3/api-docs` and Swagger UI at `/swagger-ui.html`.
FleetOps: API documentation is a known gap (not in the current implementation). Production recommendation: add `springdoc-openapi-starter-webmvc-ui` dependency, annotate controllers, expose swagger UI (protected behind ADMIN role or removed in prod for security).

**Q156: What is the purpose of `kubectl describe` vs `kubectl get`?**
`kubectl get pod auth-service-abc123`: shows status summary (phase, ready, restarts, age).
`kubectl describe pod auth-service-abc123`: shows everything â€” events (why pod is stuck), env vars, volumes, probe configurations, resource requests/limits, node assignment, QoS class.
For debugging: always start with `kubectl describe` to see events (e.g., "Back-off pulling image: 404 Not Found" â†’ wrong image tag, ECR lifecycle policy deleted it).

**Q157: What is `kubectl exec` and when would you use it?**
`kubectl exec -it auth-service-abc123 -- /bin/sh`: opens a shell inside the running container.
Debug use cases:
- `nc -z postgres-host 5432`: verify DB connectivity from inside the pod
- `env | grep AWS`: verify IRSA env vars are set
- `cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token`: verify projected OIDC token exists
- `df -h /var/www/fleetops/shared-media`: verify EFS is mounted and has space
Note: JRE-only images may not have `/bin/sh` (Alpine has it). Distroless images have no shell â€” use `kubectl debug` instead.

**Q158: What is the EKS `aws-auth` ConfigMap and what happens if it's misconfigured?**
`aws-auth` in `kube-system` maps IAM entities to Kubernetes RBAC users:
```yaml
mapUsers:
- userarn: arn:aws:iam::538661800892:user/johan
  username: cluster-admin
  groups: [system:masters]
mapRoles:
- rolearn: arn:aws:iam::538661800892:role/fleetops-prod-github-actions-role
  username: github-actions
  groups: [system:masters]
```
Misconfiguration risk: if this ConfigMap is deleted or corrupted, ALL kubectl access is lost (even the cluster creator). Recovery: use the EKS API directly (AWS console) or `aws eks update-kubeconfig` with the node role (as a workaround). Production: enable EKS Access Entries (new EKS feature) instead of `aws-auth` for safer management.

**Q159: How does Helm handle secrets in values files?**
Helm values files are plaintext YAML â€” never store secrets in them. The `values-prod.yaml` in FleetOps contains NO secrets: image tags, replica counts, resource limits, config values. All secrets come from K8s Secrets (ESO-synced from Secrets Manager).
Pattern used: chart templates reference secrets by name:
```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: fleetops-postgres-secret  # ESO creates this
        key: POSTGRES_PASSWORD
```
The Helm chart never touches the secret value â€” only its reference.

**Q160: What is `kubectl top` and what does it require?**
`kubectl top pods -n fleetops-prod`: shows real-time CPU and memory usage per pod.
`kubectl top nodes`: shows per-node usage.
Requires: metrics-server running in kube-system. FleetOps installs metrics-server (Helm chart 3.12.1) via Terraform. Also used by: HPA (to get CPU metrics for scaling decisions).
Output:
```
NAME                                    CPU(cores)   MEMORY(bytes)
fleetops-auth-service-7d8b9c-abc12      45m          256Mi
fleetops-vehicle-service-6f9d-def34     120m         384Mi
```

**Q161: What is the Kubernetes `Terminating` namespace issue and how is it fixed?**
If a namespace is stuck in `Terminating` state, it means a resource with a finalizer is preventing deletion. Common cause: ArgoCD Application resources have a finalizer that waits for the app to be deleted cleanly. If ArgoCD is already gone, nothing removes the finalizer.
Fix: manually remove the finalizer:
```bash
kubectl get namespace fleetops-prod -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/fleetops-prod/finalize" -f -
```
This force-removes the namespace regardless of finalizers. Use only when confident about what's being deleted.

**Q162: What is Kustomize and how does it compare to Helm?**
Kustomize: patch-based YAML customization. Uses base + overlays (patch files, prefix/suffix generators, secret generators). Part of kubectl since 1.14.
Helm: template-based. Go templates + values files. Package manager with versioning, rollback, hooks.
FleetOps uses Helm: better for parameterized multi-environment deployments (values-dev.yaml, values-prod.yaml). Kustomize would work but requires explicit patch files for every environment difference.

**Q163: What monitoring would you set up to know FleetOps is healthy before an evaluation?**
```bash
# 1. All pods Running and Ready
kubectl get pods -n fleetops-prod
# 2. ArgoCD all apps Synced and Healthy  
kubectl get applications -n argocd
# 3. ALB health checks passing (check target groups in AWS console)
# 4. Smoke test all endpoints
curl -s https://fleetops.website/api/auth/health  # or /actuator/health
curl -s -H "Authorization: Bearer $TOKEN" https://fleetops.website/api/vehicles
# 5. CloudWatch: no active alarms
aws cloudwatch describe-alarms --state-value ALARM
# 6. ESO secrets synced
kubectl get externalsecrets -n fleetops-prod
# 7. ArgoCD UI: all green
open https://argocd.fleetops.website
```

**Q164: How would you debug a 500 Internal Server Error from a specific endpoint?**
```
1. kubectl logs -n fleetops-prod -l app.kubernetes.io/name=fleetops-vehicle-service --tail=100
   â†’ Find the stack trace

2. kubectl describe pod {failing-pod}
   â†’ Check for OOMKilled, resource pressure, recent events

3. Check CloudWatch Log Insights:
   fields @timestamp, @message | filter kubernetes.pod_name = "{pod}" | filter @message like "ERROR"

4. Check HikariCP pool:
   kubectl logs ... | grep "HikariPool\|connection\|timeout"
   
5. Check IRSA creds:
   kubectl exec -it {pod} -- env | grep AWS_
   
6. Check Bedrock/S3/SFN call:
   Look for AWS SDK exception in the stack trace
   Common: ThrottlingException, ResourceNotFoundException, AccessDeniedException
```

**Q165: What is the `spring.datasource.hikari.maximum-pool-size` setting and what value does FleetOps use?**
HikariCP maximum pool size limits the number of concurrent connections this pod can hold to RDS.
db.t3.micro max connections â‰ˆ 85. With 4 services Ã— 2 replicas = 8 pods.
Safe allocation: (85 - 10 for maintenance) / 8 = ~9 per pod. FleetOps sets ~5-7 to be conservative.
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
      connection-timeout: 30000  # 30s before giving up
      idle-timeout: 600000       # 10min idle before closing
```
On HPA scale to 5 replicas: 4 services Ã— 5 replicas Ã— 5 connections = 100 > 85. Connection exhaustion. Mitigation: reduce pool size to 3 when deploying at max scale, or use RDS Proxy.



---

## PHASE 10 â€” Q&A CONTINUED (Q166â€“Q260)

### GENERAL ARCHITECTURE â€” continued

**Q166. Why does FleetOps use four separate microservices instead of a monolith?**
Each service owns a distinct bounded context: identity/audit (auth), asset registry (vehicle), work orders (maintenance), and service requests (request). Independent deployability means a bug in maintenance-service doesn't require redeploying auth-service. Each service scales independently â€” vehicle-service handles heavy GPS polling while request-service is low-traffic. Different data schemas are isolated: no cross-service foreign keys; communication is via REST or event (SNS/EventBridge).

**Q167. How does a React SPA handle page refresh without returning 404?**
Nginx is configured with `try_files $uri $uri/ /index.html` â€” any path that doesn't match a static file falls through to index.html. React Router then interprets the URL client-side. Without this, a refresh on `/vehicles/42` hits Nginx, which looks for a file at that path, finds nothing, and returns 404.

**Q168. Why is the frontend served through CloudFront rather than directly from the ALB?**
CloudFront provides: global edge caching (static assets served from PoP nearest to user), TLS termination at the edge, WAF integration, custom cache behaviors (long TTL for `/assets/*`, no-cache for `/index.html`), and DDoS protection via AWS Shield Standard. The ALB only handles origin requests for API paths.

**Q169. Explain the dual-host ingress design (fleetops.website vs origin.fleetops.website).**
CloudFront rewrites `Host: fleetops.website` to `Host: origin.fleetops.website` when forwarding to the ALB origin. The ALB Ingress Controller creates one listener with two host-based routing rules. Lambda functions (non-VPC) also call `origin.fleetops.website` directly to bypass CloudFront and reach backend pods. Without this second host rule, CloudFrontâ†’ALB routing would return 404.

**Q170. What happens when HikariCP exhausts its connection pool?**
HikariCP waits up to `connectionTimeout` (30s default) for a connection to become available. If no connection is freed in that window, it throws `SQLTransientConnectionException`. Under sustained load this surfaces as HTTP 500s. Mitigation: increase pool size, add read replicas, implement circuit breakers, or reduce query latency to return connections faster.

**Q171. How does Flyway know which migrations have already run?**
Flyway maintains a `flyway_schema_history` table in the target database. Each applied migration is recorded with its version, description, checksum, and execution timestamp. On startup, Flyway compares pending migration files against this table and only applies unapplied scripts in version order. If a previously-applied script's checksum changes, Flyway fails with a validation error.

**Q172. Why does the auth-service use HS256 for JWT signing instead of RS256?**
HS256 uses a shared symmetric key â€” simpler for a single-issuer/single-verifier pattern where only auth-service signs and only auth-service verifies. RS256 would add value if other services needed to verify tokens independently without sharing the secret. In FleetOps, downstream services pass tokens to auth-service's `/api/auth/validate` endpoint for verification, so a shared secret is sufficient and simpler.

**Q173. What is the purpose of the audit trail in auth-service?**
Every login, logout, failed attempt, role change, and token validation is written to an audit log table. This satisfies compliance requirements (who accessed what, when), enables forensic investigation after a security incident, and feeds CloudWatch metrics for anomaly detection. Exposed at `/api/audit` â€” only accessible to ADMIN role.

**Q174. How does Redis serve as both a cache and a session store in FleetOps?**
For caching: frequently-read vehicle status data is stored with a TTL so RDS isn't hammered by GPS polling. For session: JWT blacklist entries (on logout) are stored in Redis with TTL matching the token's remaining expiry. When auth-service validates a token, it checks Redis for a blacklist entry before trusting the JWT signature â€” this enables stateless JWT with revocation capability.

**Q175. What is the data flow when a maintenance request is created?**
Client POST `/api/requests` â†’ ALB â†’ request-service pod â†’ validates JWT via auth-service â†’ writes ServiceRequest to its RDS schema â†’ publishes `ServiceRequested` event to SNS â†’ EventBridge rule matches event â†’ triggers alert-processor Lambda â†’ Lambda calls back to vehicle-service and auth-service to enrich event â†’ publishes to insurance SNS or service SNS â†’ Step Functions workflow starts for multi-step approval if needed.

**Q176. How does the S3 presigned URL mechanism work for media uploads?**
Client requests upload URL from maintenance-service â†’ service calls `s3:PutObject` presigned URL generation (SigV4-signed, 15-min expiry, scoped to specific key) â†’ returns URL to client â†’ client uploads directly to S3 (no traffic through backend) â†’ S3 stores object â†’ service stores the S3 key reference in RDS. This avoids streaming large files through application pods.

**Q177. What monitoring signals would indicate the system is unhealthy?**
Key signals: ALB 5xx rate >1% (SLO breach), P99 latency >2s, HikariCP active connections near pool max, RDS CPU >70%, Redis memory >80%, EKS node CPU/memory >80%, pod restart count >3 in 5 minutes, Flyway migration failure on startup, ESO sync failures (secrets not refreshed), ArgoCD sync health degraded.

**Q178. How does the application handle a Redis failure gracefully?**
Spring Boot services with `spring.cache.type=redis` should be configured with `cache-null-values=true` and a fallback to read-through from RDS when Redis is unavailable. If Redis is completely down, the circuit breaker (if implemented) opens and all cache reads fall through to the database. This increases RDS load significantly â€” hence the importance of Redis cluster mode for HA.

**Q179. What is the role of EventBridge in the architecture versus SNS?**
SNS is a push-based fan-out to subscribers (email, Lambda, SQS). EventBridge is a rules-based event bus with schema registry, content-based filtering, and cross-account routing. In FleetOps: SNS delivers alerts to end users (email subscriptions). EventBridge filters specific event patterns (e.g., `ServiceRequested` with `priority=HIGH`) and routes to Lambda â€” decoupling event producers from consumers without point-to-point Lambda invocations.

**Q180. Explain the Step Functions Standard Workflow for service request approval.**
Standard Workflow: durable execution state persisted externally, exactly-once execution, supports waits up to 1 year. States: (1) ValidateRequest â€” Lambda validates fields, (2) CheckVehicleAvailability â€” Lambda calls vehicle-service, (3) Choice â€” if available, go to AssignTechnician; if not, go to QueueRequest, (4) AssignTechnician â€” Lambda calls maintenance-service, (5) NotifyParties â€” Lambda publishes SNS, (6) End. Each state transition is logged to CloudWatch.

**Q181. Why use Bedrock's Nova Lite model specifically?**
amazon.nova-lite-v1:0 is AWS's lowest-latency, lowest-cost multimodal model in Bedrock â€” appropriate for real-time fleet analysis queries where response speed matters more than maximum reasoning depth. It supports text and image inputs, enabling future maintenance-image analysis. Bedrock eliminates the need to manage model infrastructure and provides VPC endpoint support for private access.

**Q182. How does CloudFront cache invalidation work when the frontend is redeployed?**
The CI/CD pipeline hashes asset filenames (Vite content-hash: `main.abc123.js`). Since filenames change with each build, CloudFront automatically serves new assets â€” no invalidation needed for `/assets/*`. Only `/index.html` needs explicit invalidation (`aws cloudfront create-invalidation --paths /index.html`) because it has a short TTL and is the entry point that references the new hashed assets.

**Q183. What is the VPC architecture and why are services in private subnets?**
Three subnet tiers: public (ALB, NAT Gateway), private (EKS nodes, pods via VPC CNI), database (RDS, ElastiCache â€” no internet route). EKS nodes are in private subnets â€” they reach the internet via NAT Gateway (for ECR pulls, AWS API calls) but are not directly reachable from the internet. RDS and Redis are in database subnets with SG rules allowing only traffic from the EKS cluster SG.

**Q184. How does the ALB perform health checks on pods?**
ALB is configured with `healthcheck-path: /actuator/health` and `healthcheck-interval-seconds: 30`. For each registered pod IP (target-type: ip), the ALB sends GET requests to port 8080 (or 80 for frontend) at /actuator/health. Two consecutive successes (healthy-threshold: 2) mark a target healthy; three consecutive failures (unhealthy-threshold: 3) remove it from rotation. Spring Actuator returns 200 with `{"status":"UP"}` when all health indicators pass.

**Q185. What happens to in-flight requests during a rolling deployment?**
With `maxUnavailable: 0`, old pods stay in service until new pods pass readiness probes. The ALB drains connections from terminating pods (deregistration delay: 30s default) â€” existing requests complete before the pod dies. New requests go to new pods. `preStop` hook adds a sleep to give the ALB time to deregister before the container process exits. Zero downtime is achievable but requires correct probe and deregistration configuration.

**Q186. How are database credentials rotated without application downtime?**
Secrets Manager supports automatic rotation via Lambda. When rotation fires: (1) Lambda creates new DB credentials in RDS, (2) stores them in Secrets Manager, (3) ESO detects changed secret version within its reconcile interval (1h default or triggered), (4) updates the K8s Secret, (5) pods read the updated secret from the mounted volume (file-based secrets are live-updated by kubelet without pod restart). HikariCP re-authenticates new connections with the updated credentials.

**Q187. What observability stack is used and what does each component provide?**
CloudWatch: metrics (EKS, RDS, Lambda, ALB), log groups (Fluent Bit â†’ CloudWatch Logs), alarms, dashboards. X-Ray: distributed tracing â€” traces span from ALB through Spring Boot services (via AWS X-Ray SDK) to RDS/Redis calls, showing P50/P99 latency per segment. CloudTrail: API audit log for all AWS control-plane calls. AWS Config: compliance drift detection (e.g., security group opens port 22 to 0.0.0.0/0 â€” triggers non-compliant finding).

**Q188. Why use KMS customer-managed keys (CMK) instead of AWS-managed keys?**
CMK provides: key rotation control (annual), key usage audit via CloudTrail (`kms:Decrypt` events show exactly which principal decrypted what), key policy enforcement (deny Decrypt to specific principals), and cross-account sharing. AWS-managed keys (`aws/rds`, `aws/s3`) cannot be deleted, have no custom key policy, and AWS controls rotation. For compliance (GDPR data erasure via key deletion, audit requirements), CMK is essential.

**Q189. Explain how WAF protects the application.**
WAFv2 WebACL is associated with the CloudFront distribution. Rule groups: AWS Managed Rules (common attack patterns â€” SQLi, XSS, known bad IPs), rate-based rule (limit to 2000 requests/5min per IP to prevent brute force), size constraint (reject bodies >10MB to prevent large payload attacks). WAF evaluates rules top-down; first match wins. Blocked requests get 403. WAF logs to S3 for forensic analysis.

**Q190. How does ACM certificate auto-renewal work?**
ACM monitors certificate expiry. For DNS-validated certificates (used in FleetOps), ACM automatically renews ~60 days before expiry by re-verifying the CNAME record in Route53. Since the CNAME was added during initial validation and never removed, renewal is fully automatic â€” no manual action needed. The new certificate is automatically deployed to CloudFront and ALB.

---

### AWS DEEP DIVE â€” continued

**Q191. What is the EKS control plane and who manages it?**
AWS manages the control plane: API server, etcd, scheduler, controller manager â€” all run in AWS-managed VPC, multi-AZ, with automated patching and HA. You pay $0.10/hour per cluster. You manage: node groups, add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI), and Kubernetes objects. The cluster endpoint is exposed publicly (restricted to `eks_public_access_cidrs` in FleetOps) or privately.

**Q192. How does EKS authenticate kubectl commands?**
`aws eks get-token --cluster-name fleetops-prod` generates a presigned STS `GetCallerIdentity` URL, base64-encoded as a Bearer token. The EKS API server's webhook authenticator (`aws-iam-authenticator`) calls STS to resolve the IAM identity from the token. It then maps the IAM ARN to a Kubernetes RBAC username via the `aws-auth` ConfigMap. The RBAC binding determines what the identity can do.

**Q193. What is the aws-auth ConfigMap and how does it work?**
`aws-auth` in `kube-system` maps IAM roles/users to K8s RBAC users/groups. Example entry: `rolearn: arn:aws:iam::ACCOUNT:role/eks-node-role, username: system:node:{{EC2PrivateDNSName}}, groups: [system:bootstrappers, system:nodes]`. Node group role is mapped here so kubelets can authenticate. Admin IAM users are mapped to `system:masters` group. Misconfiguration locks out all cluster access.

**Q194. Explain EKS managed node groups versus self-managed nodes.**
Managed node groups: AWS provisions and manages EC2 instances, auto-scaling group, AMI updates, graceful node draining during updates â€” all via EKS API. Self-managed: you provision EC2s, configure kubelet, manage AMI, handle drain/cordon manually. FleetOps uses managed node groups (`module.eks_nodegroup`) â€” simpler lifecycle management, automatic security patches, EKS-optimized AMI (Amazon Linux 2 with containerd).

**Q195. How does the EBS CSI driver work in EKS?**
EBS CSI driver runs as a DaemonSet (node plugin) + Deployment (controller). When a PVC with `storageClass: gp3` is created, the controller calls EC2 `CreateVolume` API (using IRSA credentials), then `AttachVolume` to the node hosting the pod. The node plugin then formats and mounts the volume at the pod's requested path. On pod delete/move, the driver calls `DetachVolume` and optionally `DeleteVolume` (if reclaim policy is Delete).

**Q196. What is the difference between EFS and EBS in the FleetOps context?**
EFS: NFS-based, multi-AZ, multiple pods can mount simultaneously (ReadWriteMany) â€” used for maintenance-service shared media storage. EBS: block storage, single-AZ, one node at a time (ReadWriteOnce) â€” appropriate for databases but not used in FleetOps pods directly. EFS costs more per GB (~$0.30/GB vs ~$0.08/GB for EBS gp3) but provides the shared access pattern needed for media files across multiple pod replicas.

**Q197. How does RDS Multi-AZ work and what does it protect against?**
RDS Multi-AZ synchronously replicates all writes from the primary instance to a standby in another AZ using Amazon's internal replication protocol (not PostgreSQL streaming replication). The standby is not readable. On primary failure (hardware, AZ outage, maintenance), RDS automatically updates the CNAME DNS record to point to the standby â€” typically within 60-120 seconds. Applications reconnect after TCP timeout; HikariCP reconnects automatically.

**Q198. What is ElastiCache Redis cluster mode and why isn't it enabled in FleetOps?**
Cluster mode partitions data across multiple primary shards (up to 500), each with up to 5 read replicas â€” enables horizontal scaling of both reads and writes. FleetOps uses a single-node Redis (`redis_node_type: cache.t3.micro`) â€” appropriate for a training project. In production, you'd enable cluster mode with at least 3 shards and 1 replica per shard for HA. Without cluster mode, Redis failure means total cache unavailability.

**Q199. How does SNS fanout work with multiple subscribers?**
When a message is published to an SNS topic, SNS delivers copies to ALL subscriptions concurrently. In FleetOps: `insurance_alerts_topic` has email subscription(s) and optionally a Lambda. SNS retries failed deliveries (with exponential backoff for Lambda, with delivery policy for HTTP). Email subscriptions require opt-in confirmation. SNS is not ordered and has at-least-once delivery â€” idempotent consumers are important.

**Q200. What is the Lambda cold start problem and how is it mitigated?**
Cold start: when Lambda has no warm execution environment, it must: provision a micro-VM, load the runtime (Node.js 18), download and initialize the function code, run the handler. For Node.js this takes 200-500ms typically. Mitigation options: Provisioned Concurrency (pre-warms N environments â€” costs money), keep functions small (fast init), avoid VPC placement (adds ENI attachment time ~500ms). FleetOps Lambda is non-VPC â€” avoids the ENI penalty.

**Q201. How does Lambda access FleetOps backend services if it's not in the VPC?**
Lambda calls `origin.fleetops.website` â€” a public DNS name pointing to the ALB. Traffic path: Lambda â†’ internet â†’ Route53 resolves origin.fleetops.website â†’ ALB â†’ pod. The ALB has an internet-facing scheme, so it's reachable from the internet. WAF and ALB security groups control what traffic is allowed. Lambda uses stored credentials (from Secrets Manager via `lambda_service_credentials_secret_arn`) to authenticate as a service account.

**Q202. What is the X-Ray service map and what does it show?**
X-Ray aggregates trace segments from instrumented services into a service map â€” a directed graph showing: nodes (services/AWS resources), edges (calls between them), latency histograms per edge, error rates per node. In FleetOps: ALB â†’ auth-service â†’ RDS, ALB â†’ vehicle-service â†’ Redis â†’ RDS, Lambda â†’ vehicle-service â†’ RDS. Helps identify which service is the bottleneck (longest segment) or highest error source.

**Q203. How does CloudTrail differ from CloudWatch Logs?**
CloudTrail: records AWS API calls (control plane) â€” who called what AWS API, when, from where, with what parameters. Stored in S3, queryable via Athena. Example: `ec2:CreateSecurityGroup`, `secretsmanager:GetSecretValue`, `sts:AssumeRoleWithWebIdentity`. CloudWatch Logs: application and infrastructure logs â€” stdout/stderr from containers (via Fluent Bit), Lambda execution logs, VPC flow logs. CloudTrail is for "who did what to AWS infrastructure"; CloudWatch Logs is for "what happened inside the application."

**Q204. What does AWS Config actually check in FleetOps?**
AWS Config evaluates managed rules against resource configurations. Example rules: `restricted-ssh` (flags SGs allowing port 22 from 0.0.0.0/0), `s3-bucket-public-read-prohibited` (flags public S3 buckets), `encrypted-volumes` (flags unencrypted EBS), `rds-storage-encrypted` (flags unencrypted RDS), `iam-root-access-key-check` (flags root access keys). Non-compliant findings appear in Config console and can trigger SNS notifications.

**Q205. How does ECR image scanning work?**
ECR supports two scan types: Basic (uses Clair, free) and Enhanced (uses Inspector, additional cost). On push, ECR scans the image layers against a CVE database. Results include findings by severity (CRITICAL, HIGH, MEDIUM, LOW, INFORMATIONAL). In CI/CD, Trivy performs a pre-push scan â€” images with CRITICAL CVEs fail the pipeline. ECR lifecycle policies automatically delete old untagged images (keeping last 10 tagged images) to reduce storage cost and attack surface.

**Q206. What is the purpose of the KMS events_key in FleetOps?**
`kms.events_key_arn` is used by: SNS topic encryption (messages at rest), Step Functions execution history encryption, CloudTrail log file encryption, CloudWatch Log group encryption, EventBridge archive encryption. Centralizing these under one CMK simplifies key management and audit â€” all `kms:Decrypt` calls for event data appear under one key in CloudTrail.

**Q207. How does Secrets Manager secret rotation Lambda work internally?**
The rotation Lambda follows a 4-step protocol: (1) `createSecret` â€” generates new credentials, stores as `AWSPENDING` version, (2) `setSecret` â€” applies the new password to the RDS instance via `rds:ModifyDBInstance`, (3) `testSecret` â€” connects with `AWSPENDING` credentials to verify they work, (4) `finishSecret` â€” promotes `AWSPENDING` to `AWSCURRENT`, demotes old version to `AWSPREVIOUS`. ESO then picks up the new `AWSCURRENT` on next reconcile.

**Q208. What is the SSM Parameter Store used for vs Secrets Manager?**
SSM Parameter Store (Standard tier, free): non-sensitive configuration â€” Redis endpoint, CORS origins, SNS topic ARNs, base URL. These are injected via ESO as K8s ConfigMaps. Secrets Manager (paid, ~$0.40/secret/month): sensitive credentials â€” DB password, JWT secret, GitHub PAT. The distinction: SSM for config that's inconvenient to hardcode; Secrets Manager for data that must be audited, rotated, and encrypted.

**Q209. Explain the Route53 DNS resolution path for fleetops.website.**
Route53 hosts the `fleetops.website` public hosted zone. `fleetops.website` A record â†’ CloudFront distribution IP (alias record, not a real IP â€” AWS manages the IP pool). `origin.fleetops.website` A record â†’ ALB DNS name (alias). `argocd.fleetops.website` A record â†’ same ALB. When a client resolves `fleetops.website`, Route53 returns CloudFront IPs; CloudFront proxies to `origin.fleetops.website` for dynamic content.

**Q210. How does CloudFront determine which requests go to the ALB vs. serve cached content?**
CloudFront behaviors (ordered by path pattern, most specific first): `/api/*` â†’ forward to ALB origin (no caching, forward all headers + cookies), `/assets/*` â†’ cache at edge (TTL: 1 year, content-hash filenames), `/index.html` â†’ cache with low TTL (no-store or 5min), `/*` â†’ cache with medium TTL. API requests always hit origin; static assets are served from edge caches worldwide.

---

### KUBERNETES â€” continued

**Q211. What is a Kubernetes Operator and does FleetOps use any?**
An Operator is a custom controller that encodes operational knowledge about a stateful application. It watches Custom Resources (CRDs) and reconciles actual state to desired state. FleetOps uses: External Secrets Operator (watches `ExternalSecret` CRDs â†’ reconciles K8s Secrets from Secrets Manager), ArgoCD (watches `Application` CRDs â†’ reconciles cluster state from Git). Both are installed via Helm during Terraform `eks_addons` provisioning.

**Q212. How does Horizontal Pod Autoscaler decide when to scale?**
HPA watches a metric source (typically `cpu` from metrics-server) at `--horizontal-pod-autoscaler-sync-period` (15s default). Desired replicas = ceil(currentReplicas Ã— currentMetric / targetMetric). Example: 2 replicas at 80% CPU, target 50% â†’ ceil(2 Ã— 80/50) = ceil(3.2) = 4 replicas. Scale-up is immediate when threshold is exceeded; scale-down waits `--horizontal-pod-autoscaler-downscale-stabilization` (5min default) to avoid thrashing.

**Q213. What is the metrics-server and how does it feed HPA?**
metrics-server is a Deployment that scrapes `kubelet/metrics/resource` from each node (via the Summary API) every 15s. It aggregates CPU and memory per pod and exposes them via the `metrics.k8s.io` API. HPA queries this API. metrics-server is NOT a long-term metrics store â€” it holds only the current sample. For historical metrics, Prometheus + Adapter is needed (not in FleetOps scope).

**Q214. Explain pod affinity and anti-affinity â€” why would you use them?**
Pod affinity: schedule pods near other pods (same node or AZ). Pod anti-affinity: spread pods away from each other. In FleetOps: `topologySpreadConstraints` or anti-affinity rules would ensure 2 replicas of auth-service don't both land on the same node â€” if that node fails, one replica survives. Without anti-affinity, the scheduler may colocate both replicas on the cheapest-fitting node, creating a single point of failure.

**Q215. What is a PodDisruptionBudget and why is it important during node upgrades?**
PDB specifies the minimum number of pods that must remain available during voluntary disruptions (node drain, rolling update). Example: `minAvailable: 1` for a 2-replica deployment means draining a node that hosts one replica is allowed (1 remains), but draining a node that hosts both is blocked. During EKS managed node group updates (which drain nodes), PDBs prevent all replicas of a service from being evicted simultaneously.

**Q216. How does the Kubernetes scheduler place pods?**
Scheduling is two phases: (1) Filtering â€” eliminate nodes that don't satisfy hard constraints: resource requests (CPU/memory), nodeSelector, affinity/anti-affinity, taints/tolerations, volume topology. (2) Scoring â€” rank remaining nodes by priority functions: LeastAllocated (prefer nodes with most free resources), spreading scores, image locality (prefer nodes that already have the image). Pod goes to the highest-scoring node.

**Q217. What is a taint and toleration? Give a practical example.**
Taint: mark a node to repel pods. `kubectl taint nodes node1 dedicated=gpu:NoSchedule` â€” no pod lands on node1 unless it has a matching toleration. Toleration: allow a pod to be scheduled on a tainted node. Use case: reserve GPU nodes for ML workloads. In EKS, nodes tainted `eks.amazonaws.com/nodegroup=monitoring:NoSchedule` would only accept monitoring pods that tolerate the taint.

**Q218. What happens when a pod's OOMKilled limit is hit repeatedly?**
CrashLoopBackOff: kubelet applies exponential backoff between restarts (10s, 20s, 40s... up to 5min). The pod's `restartCount` increments. `kubectl describe pod` shows `OOMKilled` as the last termination reason and exit code 137. Mitigation: increase memory limit, fix memory leak, or reduce memory-intensive operations. Check JVM heap settings â€” Spring Boot on Java 21 should use `-XX:MaxRAMPercentage=75` to cap heap at 75% of the container limit.

**Q219. What is the difference between a liveness probe and a readiness probe?**
Liveness: "Is this container alive?" Failure â†’ kubelet kills and restarts the container. Use for detecting deadlocks or hung processes. Readiness: "Is this container ready to serve traffic?" Failure â†’ pod removed from Service endpoints (no traffic sent). Use during startup or when a downstream dependency is unavailable. In FleetOps: `/actuator/health/liveness` and `/actuator/health/readiness` are separate Spring Boot endpoints â€” readiness can fail when RDS is unreachable without killing the pod.

**Q220. How does a Kubernetes Service of type ClusterIP work internally?**
ClusterIP is a virtual IP (VIP) assigned from the service CIDR (e.g., 10.100.0.0/16). No process actually listens on this IP â€” it exists only in iptables rules managed by kube-proxy. When a pod sends a packet to the ClusterIP:port, iptables intercepts in PREROUTING, traverses KUBE-SERVICES â†’ KUBE-SVC-xxx â†’ KUBE-SEP-xxx (one per endpoint, probabilistic selection) â†’ DNAT rewrites destination to a pod IP:port. The packet then routes normally to the pod.

**Q221. Why doesn't FleetOps use NodePort or LoadBalancer Service types for backend services?**
NodePort exposes a static port on every node â€” bypasses the ingress layer, harder to secure, not path-based. LoadBalancer creates one AWS ELB per service â€” expensive (one ELB ~$16-18/month Ã— 4 services = $64-72/month extra) and unmanageable at scale. ClusterIP + ALB Ingress Controller provides one shared ALB with path-based routing to all services â€” one ELB, one ACM cert, unified access logs.

**Q222. Explain ConfigMap volume mounts vs. environment variable injection.**
Volume mount: ConfigMap data is mounted as files in the container filesystem. Kubelet watches the ConfigMap and updates the file without pod restart (within ~1min). Useful for config files (application.yml, nginx.conf). Environment variable: `envFrom: configMapRef`. Injected at pod startup, NOT updated on ConfigMap change â€” requires pod restart. FleetOps uses env vars for most Spring Boot properties; a config change requires a rollout.

**Q223. What is RBAC in Kubernetes? Name the four resource types.**
RBAC (Role-Based Access Control) controls who can do what to which K8s resources. Four types: (1) Role â€” namespace-scoped set of rules (verbs on resources), (2) ClusterRole â€” cluster-scoped rules, (3) RoleBinding â€” binds a Role to a subject (user, group, ServiceAccount) within a namespace, (4) ClusterRoleBinding â€” binds a ClusterRole cluster-wide. ESO uses a ClusterRole to read ClusterSecretStore and create Secrets in any namespace.

**Q224. How does ArgoCD detect configuration drift?**
ArgoCD's application controller polls Git every 3 minutes (default) or via Git webhook. It compares the desired state (rendered manifests from Git) against live state (K8s API server). Diff is computed using a three-way merge (like `kubectl apply`). Differences are reported as `OutOfSync`. With `syncPolicy: automated`, ArgoCD applies the diff automatically. Drift can also be detected for resources modified directly with `kubectl` (bypassing Git).

**Q225. What are ArgoCD sync waves and why does FleetOps use them?**
Sync wave is `argocd.argoproj.io/sync-wave` annotation on resources. ArgoCD applies resources in ascending wave order, waiting for each wave's resources to be healthy before proceeding. FleetOps waves: 0 (namespaces, CRDs), 1 (ClusterSecretStore), 2 (ExternalSecrets), 3 (ConfigMaps), 4 (Deployments), 5 (Ingress). This ensures secrets exist before pods start, and pods exist before ingress routes to them â€” preventing startup race conditions.

**Q226. What is the App-of-Apps pattern in ArgoCD?**
A root Application points to a directory of Application manifests in Git. Each child Application manages a subset of the cluster (e.g., auth-service, vehicle-service, infrastructure). The root app is bootstrapped manually; it then syncs and creates all child apps, which then sync their own resources. Benefits: single ArgoCD installation manages arbitrary numbers of apps; adding a new service means adding one Application manifest to Git â€” no manual ArgoCD configuration.

**Q227. How does the External Secrets Operator authenticate to AWS Secrets Manager?**
ESO pod has a ServiceAccount annotated with `eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/fleetops-eso-role`. The Pod Identity Webhook mutates the pod spec to inject the OIDC token at a projected volume path. When ESO calls `secretsmanager:GetSecretValue`, the AWS SDK automatically exchanges the projected token for temporary credentials via `sts:AssumeRoleWithWebIdentity`. The IAM role policy grants `secretsmanager:GetSecretValue` on `fleetops/prod/*` resources only.

**Q228. What happens if an ExternalSecret fails to sync?**
The ExternalSecret resource's status shows `SecretSyncedError` condition. ESO retries with exponential backoff. The existing K8s Secret is NOT deleted â€” it retains the last successfully synced value. Pod startup that depends on the secret may fail if the Secret doesn't exist yet (first sync). If ESO is down entirely, existing Secrets persist but won't be refreshed â€” stale credentials for up to ~1h before rotation is missed.

**Q229. How does NetworkPolicy enforce pod-to-pod traffic rules?**
NetworkPolicy is implemented by the CNI plugin (VPC CNI uses eBPF or iptables, or requires a separate policy engine like Calico). A NetworkPolicy selects pods via `podSelector` and specifies allowed ingress/egress. Without a policy, all traffic is allowed. With a deny-all policy plus explicit allows, only permitted paths work. Example: allow ingress to auth-service:8080 only from pods with label `component: frontend` â€” drops all other ingress silently.

**Q230. What is a DaemonSet and what does FleetOps run as a DaemonSet?**
DaemonSet ensures exactly one pod runs on every node (or a selected subset). In FleetOps: Fluent Bit runs as a DaemonSet â€” it needs to read `/var/log/containers/*.log` from the node filesystem, so one instance per node is required. Other DaemonSets in EKS: `aws-node` (VPC CNI), `kube-proxy`. When a new node joins, the DaemonSet controller automatically schedules the pod on it.

---

### TERRAFORM â€” continued

**Q231. What is the Terraform dependency graph and how is it constructed?**
Terraform builds a DAG where each resource/module is a node. Edges come from explicit references (`module.rds.db_endpoint` creates edge networkingâ†’rds), `depends_on` annotations, and provider dependencies. During plan/apply, Terraform traverses the DAG in topological order â€” resources with no dependencies are provisioned first (in parallel), dependents follow. `terraform graph | dot -Tpng` renders the visual graph.

**Q232. What happens during `terraform plan` vs `terraform apply`?**
Plan: reads current state from S3 backend, queries AWS APIs to get actual resource attributes, computes diff (create/update/replace/destroy), outputs the execution plan without making changes. Apply: executes the plan â€” calls AWS APIs to create/modify/delete resources, updates state file on success. The state file is locked (DynamoDB) during apply to prevent concurrent modifications from another operator.

**Q233. What does `terraform refresh` do and when is it useful?**
`terraform refresh` (or `terraform apply -refresh-only`) queries AWS APIs to update the state file with actual resource attributes, without making changes to infrastructure. Useful when: someone manually modified a resource (drift detection), a resource attribute changed outside Terraform (e.g., security group rule added via console), or after importing existing resources. After refresh, `terraform plan` shows the correct drift.

**Q234. Explain the Terraform lifecycle meta-argument `create_before_destroy`.**
By default, Terraform destroys the old resource before creating the replacement (for resources that must be replaced). `create_before_destroy = true` reverses this: new resource is created first, then the old is destroyed. Critical for: ACM certificates (new cert must be valid before old is removed from ALB), RDS instances, any resource where the replacement must be healthy before the original is deleted.

**Q235. What is `terraform import` and when would you use it in FleetOps?**
`terraform import RESOURCE_ADDRESS REAL_ID` adds an existing AWS resource to Terraform state without creating it. Use case: an S3 bucket or RDS instance was created manually before Terraform was set up. After import, Terraform knows about it and can manage updates/destroy. You must write the corresponding resource block manually â€” import only updates state, doesn't generate config.

**Q236. How does Checkov integrate into the FleetOps CI/CD pipeline?**
Checkov runs `checkov -d fleetops-terraform/ --skip-check CKV_AWS_xxx` in the Terraform CI workflow (`.github/workflows/terraform.yml`) after `terraform validate` but before `terraform plan`. It scans all `.tf` files for 500+ security/compliance checks (encryption at rest, public access, logging enabled, etc.). Failures block the pipeline. `#checkov:skip=CKV_AWS_260:reason` inline comments suppress known false positives.

**Q237. What is `terraform taint` and has it been replaced?**
`terraform taint RESOURCE` marked a resource for forced replacement on next apply â€” useful for forcing recreation without changing config. It was deprecated in Terraform 1.0 and replaced by `terraform apply -replace=RESOURCE`. In FleetOps: `terraform apply -replace=module.eks_nodegroup.aws_eks_node_group.main` would force the node group to be recreated.

**Q238. How does Terraform handle provider version conflicts between modules?**
Each module can specify `required_providers` with version constraints. Terraform resolves a single provider version that satisfies all constraints using the dependency lock file (`.terraform.lock.hcl`). If two modules require incompatible versions (`>= 5.0` and `< 4.0`), Terraform errors with "no valid version." The lock file pins exact versions â€” `terraform providers lock` updates it after changing constraints.

**Q239. What is the difference between `terraform.tfvars` and `prod.auto.tfvars`?**
`terraform.tfvars` is automatically loaded by Terraform (must be named exactly this). Files matching `*.auto.tfvars` are also automatically loaded â€” `prod.auto.tfvars` is used in FleetOps to supply environment-specific values. Multiple `.auto.tfvars` files are all loaded (alphabetical order). `terraform.tfvars` is typically gitignored for secrets; `prod.auto.tfvars` in FleetOps contains secrets in plaintext â€” a known security debt to be addressed.

**Q240. What is a Terraform workspace and does FleetOps use them?**
Workspaces maintain separate state files for the same configuration â€” `terraform workspace new staging` creates a parallel state. FleetOps does NOT use workspaces â€” instead uses separate directories (`environments/prod/`, `environments/staging/` if added) with separate state keys. Directory-per-environment is preferred for strong isolation; workspaces share backend config and are easier to accidentally destroy cross-environment.

**Q241. How does `terraform destroy` know what to delete?**
`terraform destroy` reads the state file (from S3 backend) to get all managed resources and their real IDs, then calls the appropriate AWS API to delete each in reverse dependency order. It ONLY deletes resources tracked in state â€” manually created resources are unknown to Terraform. Resources with `deletion_protection` (like `enable_deletion_protection = true` on RDS) will fail â€” protection must be disabled first.

**Q242. What does `terraform validate` check vs `terraform plan`?**
`validate`: syntax check and type-checking of Terraform configuration files â€” no AWS API calls, no state access. Catches: undefined variables, invalid resource arguments, type mismatches, module interface violations. `plan`: calls AWS APIs to read current state, resolves references, computes actual changes. Catches: missing permissions, resource not found, value doesn't match constraint. Both run in CI â€” validate first (fast), then plan (slower, needs credentials).

---

### CI/CD â€” continued

**Q243. What is a reusable workflow in GitHub Actions and how does FleetOps use it?**
A reusable workflow (`workflow_call` trigger) is called from another workflow using `uses: ./.github/workflows/build.yml`. It accepts inputs and secrets from the caller. FleetOps: `build-push.yml` (reusable) handles Docker build + ECR push; called from service-specific workflows with `image-name`, `dockerfile-path` as inputs. This avoids duplicating 50+ lines of Docker build logic across 5 service workflows.

**Q244. Explain the GitHub Actions OIDC flow step by step.**
(1) Workflow step requests OIDC token from GitHub's token endpoint (automatic â€” `id-token: write` permission). (2) GitHub mints a JWT signed with its OIDC private key, containing `sub: repo:ORG/REPO:ref:refs/heads/main`, `aud: sts.amazonaws.com`. (3) `aws-actions/configure-aws-credentials` sends the JWT to STS `AssumeRoleWithWebIdentity`. (4) STS verifies JWT signature against GitHub's public JWKS endpoint. (5) STS checks the IAM role's trust policy â€” `Condition: StringLike: token.actions.githubusercontent.com:sub: repo:FleetOps-V2/*`. (6) STS returns temporary credentials (AccessKeyId, SecretAccessKey, SessionToken, expiry 1h). (7) Subsequent AWS CLI/SDK calls use these credentials.

**Q245. Why does the CI workflow use `workflow_run` trigger for deployment?**
`workflow_run` triggers a workflow when another workflow completes â€” enables sequential workflows across repositories or jobs. In FleetOps: the image-build workflow runs on push to main; deployment workflow triggers on `workflow_run: completed` of the build workflow, only if `conclusion: success`. This ensures deployment only happens after a successful build+push. `push` trigger on the same event would run both workflows simultaneously without ordering guarantee.

**Q246. How does semantic versioning via Conventional Commits work in the CI pipeline?**
The pipeline reads the commit message: `feat:` â†’ bumps minor (1.2.0 â†’ 1.3.0), `fix:` â†’ bumps patch (1.2.0 â†’ 1.2.1), `BREAKING CHANGE` in footer â†’ bumps major (1.2.0 â†’ 2.0.0). Tools like `semantic-release` or `standard-version` automate this. The computed version tags the Docker image in ECR (`1.3.0` and `latest`) and creates a GitHub Release. This enables rollback to any semantic version without tracking build numbers.

**Q247. What is Docker layer caching and why does layer order matter?**
Docker builds layers top-to-bottom. A layer is rebuilt when its instruction OR any preceding layer changes. Best practice: copy dependency files first (`pom.xml`), run dependency download (`mvn dependency:go-offline`), then copy source (`COPY src/ ./`), then build. If only source changes, the dependency layer is cached â€” build skips downloading all Maven dependencies. Wrong order (COPY all â†’ build) invalidates cache on every source change.

**Q248. Explain multi-stage Docker builds for the Spring Boot services.**
Stage 1 (`AS builder`): `FROM maven:3.9-eclipse-temurin-21` â€” copies pom.xml, downloads dependencies, copies src, runs `mvn package -DskipTests`. Produces a fat JAR. Stage 2 (`FROM eclipse-temurin:21-jre`): copies ONLY the fat JAR from stage 1. Final image has no Maven, no source code, no test dependencies â€” much smaller (typically 200-300MB vs 600MB+ for a full Maven image). Attack surface reduced significantly.

**Q249. What does `mode=max` in Docker Buildx GHA cache backend do?**
`cache-to: type=gha,mode=max` caches ALL layers including intermediate build stages (e.g., the Maven build stage). `mode=min` (default) caches only the final stage layers. With `mode=max`, a subsequent build that changes only `src/` can reuse the cached Maven dependency download layer from stage 1 â€” even though stage 1 is intermediate and not in the final image. This is the key difference that dramatically speeds up CI builds.

**Q250. How does Trivy scan Docker images in CI and what does it report?**
Trivy scans: OS packages (Alpine, Debian apt/dpkg), language packages (Maven pom.xml, npm package-lock.json), and application files for known CVEs. It pulls the CVE database from GitHub (`aquasecurity/trivy-db`). Reports findings by severity: CRITICAL, HIGH, MEDIUM, LOW, UNKNOWN. Pipeline config: `--exit-code 1 --severity CRITICAL,HIGH` fails the build on any critical or high severity CVE. Output formats: table, JSON, SARIF (for GitHub Security tab upload).

**Q251. What is SonarQube and what five quality gate metrics does FleetOps enforce?**
SonarQube performs static code analysis. The quality gate in FleetOps enforces: (1) Coverage â‰¥80% (unit test line coverage), (2) Duplications â‰¤3% (duplicate code blocks), (3) Maintainability Rating = A (technical debt ratio <5%), (4) Reliability Rating = A (no bugs), (5) Security Rating = A (no vulnerabilities). Gate failure blocks merge to main. Coverage data is uploaded via `sonar-maven-plugin` after `mvn test`.

**Q252. How does Snyk scan Maven dependencies?**
`snyk test --file=pom.xml` reads the Maven dependency tree (including transitive dependencies), resolves all artifact coordinates, and queries Snyk's vulnerability database. It reports: CVE ID, affected package, vulnerable version range, fixed version, severity. CI config: `snyk monitor` uploads results to Snyk dashboard for continuous monitoring. `--fail-on=upgradable` only fails on CVEs with available fixes â€” avoids blocking on non-fixable vulnerabilities.

**Q253. What is a GitHub Environment and how does it protect production deployments?**
GitHub Environments (Settings â†’ Environments) apply protection rules to specific deployment jobs. Rules: `required_reviewers` (at least N approvals before deployment runs), `wait-timer` (delay N minutes after approval), `deployment_branches` (only allow from `main` or release/*). In FleetOps: `production` environment requires 1 reviewer. The deployment job is paused until an authorized user approves in the GitHub UI â€” preventing accidental pushes to prod.

**Q254. How does the ArgoCD image updater work with ECR?**
ArgoCD Image Updater watches ECR for new image tags matching a pattern (e.g., semver `^1\.[0-9]+\.[0-9]+`). When a new tag is found, it updates the image tag in the Git repository (via `git commit` to a dedicated branch or directly to the values file), which ArgoCD then syncs. This closes the GitOps loop: CI pushes image â†’ image updater commits tag change to Git â†’ ArgoCD syncs â†’ rolling deployment. No CI system needs kubectl access.

**Q255. What is the difference between `git push --force` and `git push --force-with-lease`?**
`--force` overwrites the remote ref regardless of its current state â€” can silently discard colleagues' pushed commits. `--force-with-lease` checks that the remote ref is at the expected SHA before pushing â€” fails if someone else pushed since your last fetch, preventing accidental overwrites. In CI/CD for tag pushes or Git-ops commits, `--force-with-lease` is the safe default when rewriting history is unavoidable.

---

### SECURITY â€” continued

**Q256. What is the principle of least privilege and how is it applied in FleetOps?**
Least privilege: grant only the minimum permissions needed for a function to operate. In FleetOps: each microservice's IRSA role allows only the specific Secrets Manager secrets it needs (e.g., auth-service role can read `fleetops/prod/db-credentials` and `fleetops/prod/jwt-secret` â€” not vehicle-service secrets). Lambda role allows only the specific SNS topics it publishes to. ESO role allows `secretsmanager:GetSecretValue` on `fleetops/prod/*` only.

**Q257. How does BCrypt password hashing work and why is it chosen over SHA-256?**
BCrypt applies a cost factor (strength 10 in FleetOps = 2^10 = 1024 iterations of the Blowfish key setup) per hash computation. Each hash includes a random 128-bit salt, so identical passwords produce different hashes â€” rainbow tables are useless. Cost factor is configurable â€” as hardware gets faster, increase it to maintain ~100ms+ hash time. SHA-256 is deterministic, fast (nanoseconds), and without a salt is trivially brute-forced. BCrypt is purpose-built for password storage.

**Q258. What is a JWT and what are its three components?**
JWT (JSON Web Token) has three base64url-encoded parts separated by dots: (1) Header: `{"alg":"HS256","typ":"JWT"}`, (2) Payload: claims â€” `{"sub":"user-id","role":"ADMIN","iat":1700000000,"exp":1700003600}`, (3) Signature: HMAC-SHA256 of `header.payload` using the secret key. The signature is verified server-side â€” tampering with any claim invalidates the signature. JWTs are stateless â€” no server-side session store needed (except for the Redis blacklist for revocation).

**Q259. What is CSRF and how does FleetOps mitigate it?**
CSRF (Cross-Site Request Forgery): attacker tricks an authenticated user's browser into making a state-changing request to your site (using the user's session cookie). Mitigation: FleetOps uses JWT in Authorization header (not cookies) â€” browser does not automatically send headers to cross-origin requests. CORS configuration (`cors_allowed_origins` from SSM) restricts which origins can make cross-origin requests. `SameSite=Strict` cookie attribute (if cookies were used) would also prevent CSRF.

**Q260. What is SQL injection and how does Spring Data JPA prevent it?**
SQL injection: attacker inserts SQL into user input that gets executed as part of a query. Example: `username = "admin'; DROP TABLE users; --"`. Spring Data JPA uses parameterized queries: `findByUsername(String username)` generates `SELECT * FROM users WHERE username = ?` with the input bound as a parameter â€” the driver sends data and SQL separately, so the database never interprets user input as SQL. `@Query` with JPQL also uses positional parameters â€” never string concatenation.

---

### PROJECT-SPECIFIC â€” continued

**Q261. Walk through what happens when `terraform apply` is run for the first time.**
(1) S3 backend initialization â€” creates state bucket if needed, acquires DynamoDB lock. (2) Parallel: KMS, networking, route53 (no dependencies). (3) Parallel: IAM (needs OIDC â€” must wait for eks_oidc, which needs eks_cluster), RDS (needs networking, KMS), Redis, S3, EFS, SNS, SSM. (4) EKS cluster â†’ OIDC â†’ node group â†’ addons (depends on nodegroup + secrets_manager). (5) ECR, CloudWatch, CloudTrail, Config, WAF, ACM (needs route53 for DNS validation). (6) CloudFront (needs ACM, WAF). (7) Lambda, EventBridge, Step Functions. (8) Inline SG rules, Route53 alias records (need ALB â€” created by addons). Full apply: ~45-60 minutes.

**Q262. Why does `data.aws_lb.fleetops` have `depends_on = [module.eks_addons]`?**
The ALB is created by the ALB Ingress Controller (installed in `eks_addons`) when ArgoCD syncs the Ingress resource. It does not exist in Terraform state â€” Terraform discovers it via `data.aws_lb` using tags. `depends_on` ensures Terraform doesn't attempt to read the ALB data source before `eks_addons` finishes (which provisions the controller and bootstraps ArgoCD, which syncs the Ingress). Without it, the data source would fail with "no matching load balancer found."

**Q263. What would you do if a pod is stuck in `Pending` state?**
`kubectl describe pod PODNAME -n fleetops-prod` â†’ Events section shows why. Common causes: (1) Insufficient resources â€” `0/2 nodes are available: 2 Insufficient cpu` â†’ scale node group or reduce requests, (2) Volume not available â€” `waiting for a volume to be created` â†’ check PVC/PV status, (3) ImagePullBackOff â€” wrong ECR URI or missing IRSA permission for `ecr:GetAuthorizationToken`, (4) Taint/toleration mismatch â†’ check node taints, (5) NodeSelector mismatch â†’ check labels.

**Q264. How would you roll back a bad deployment in FleetOps?**
Option 1 (GitOps â€” preferred): revert the offending commit in Git â†’ ArgoCD detects â†’ re-syncs to previous manifest version â†’ rolling update back to previous image. Option 2 (ArgoCD UI): History â†’ select previous revision â†’ Rollback â†’ ArgoCD applies previous state. Option 3 (kubectl fallback): `kubectl rollout undo deployment/fleetops-auth-service -n fleetops-prod` â†’ K8s reverts to previous ReplicaSet. GitOps rollback is preferred â€” it keeps Git as the source of truth.

**Q265. What is the fleetops-prod namespace and what does it contain?**
`fleetops-prod` is the primary application namespace. Contains: 4 backend Deployments, 1 frontend Deployment, 5 ClusterIP Services, 1 Ingress (ALB), 5 ServiceAccounts (one per service with IRSA annotation), ConfigMaps (app config from SSM via ESO), Secrets (DB creds, JWT secret â€” synced from Secrets Manager via ESO), HPAs for each backend service, PVC for EFS (maintenance-service media storage).

**Q266. How does auth-service validate a JWT from a downstream request?**
Other services forward the `Authorization: Bearer <token>` header to auth-service's `/api/auth/validate` endpoint. Auth-service: (1) extracts the token, (2) parses the header+payload (base64url decode), (3) recomputes HMAC-SHA256 of `header.payload` using the JWT secret from Secrets Manager, (4) compares with the token's signature â€” if mismatch, returns 401, (5) checks `exp` claim â€” if expired, returns 401, (6) checks Redis blacklist â€” if found (logged out), returns 401, (7) returns user ID + role in response body.

**Q267. What is the request-service and what external integrations does it trigger?**
Request-service manages fleet service requests (repairs, inspections, insurance claims). On POST `/api/requests`: validates JWT, stores ServiceRequest in RDS, publishes to SNS â†’ EventBridge rule matches â†’ Lambda enriches and routes to: insurance SNS topic (for insurance-related requests) or service SNS topic (for maintenance requests). Lambda also calls Step Functions to start approval workflow for high-priority requests. Separate from maintenance-service which handles the actual work orders.

**Q268. Why is the vehicle-service also responsible for GPS tracking (`/api/tracking`)?**
Vehicle tracking data (GPS coordinates, speed, status) is logically part of the vehicle domain â€” the same aggregate that owns the vehicle's identity, registration, and assignment. Putting tracking in a separate service would require cross-service joins for common queries like "show all vehicles with their current locations." The Ingress routes both `/api/vehicles` and `/api/tracking` to vehicle-service on port 8080.

**Q269. What is the fleetops-system namespace used for?**
`fleetops-system` hosts infrastructure/platform components: ArgoCD (Argo CD server, repo-server, application-controller, redis, dex), External Secrets Operator (controller + webhook), ClusterSecretStore (cluster-scoped ESO resource). These components serve all application namespaces â€” keeping them in a separate namespace enables RBAC isolation (only platform team has write access to fleetops-system).

**Q270. How would you debug a 502 Bad Gateway from the ALB?**
502 means ALB received an invalid response from the backend pod. Steps: (1) Check ALB access logs in S3/CloudWatch â€” which target IP returned the error, (2) `kubectl logs POD -n fleetops-prod` â€” look for startup errors or exceptions, (3) Check readiness probe â€” if pod is marked unhealthy, it may be receiving traffic briefly during state transition, (4) Check `alb.ingress.kubernetes.io/healthcheck-path` â€” if /actuator/health returns non-200, ALB marks target unhealthy, (5) Check security groups â€” ALB SG must allow egress to pod IPs on 8080.

---

### AWS DEEP DIVE â€” continued

**Q271. What is an IAM instance profile vs. an IAM role vs. IRSA?**
IAM instance profile: attaches an IAM role to an EC2 instance â€” all processes on the instance share that role via the IMDS endpoint. IAM role for nodes (instance profile): used by kubelet and node-level processes. IRSA (IAM Roles for Service Accounts): maps a K8s ServiceAccount to an IAM role via OIDC â€” pod-level granularity. Each pod only gets credentials for its own role, not the node role. IRSA is strictly more secure â€” compromising one pod doesn't expose other pods' permissions.

**Q272. How does EKS handle Kubernetes version upgrades?**
Process: (1) Update control plane: `aws eks update-cluster-version` â€” AWS performs rolling update of control plane components, no downtime, takes ~20min. (2) Update add-ons: VPC CNI, CoreDNS, kube-proxy must be updated to versions compatible with the new K8s version. (3) Update node group: EKS launches new nodes with the new K8s-version AMI, cordons and drains old nodes (respecting PDBs), terminates them. Applications continue serving traffic on old nodes until new nodes are ready.

**Q273. What are EKS add-ons vs. self-managed add-ons?**
EKS managed add-ons (VPC CNI `aws-node`, CoreDNS, kube-proxy, EBS CSI): installed/updated via EKS API â€” AWS manages compatibility with the cluster version, provides one-click updates, reports health status. Self-managed (ALB Ingress Controller, External Secrets Operator, ArgoCD, Fluent Bit, metrics-server): installed via Helm, you manage version compatibility and updates. FleetOps uses Terraform Helm releases for self-managed add-ons in `module.eks_addons`.

**Q274. What is the AWS VPC CNI plugin and why is it important for FleetOps?**
VPC CNI allocates real VPC IP addresses to pods from the node's ENIs (secondary private IPs). Each node gets 1 primary ENI + N secondary ENIs (depending on instance type). Each ENI gets M IPs. `m5.large`: 3 ENIs Ã— 10 IPs = 30 pod IPs per node. Pods with VPC IPs are directly routable within the VPC â€” enables ALB `target-type: ip` (register pod IPs directly, not node IPs) and allows RDS/ElastiCache to apply SG rules to pod traffic by source IP.

**Q275. What happens when an ALB target group has no healthy targets?**
ALB returns 503 Service Unavailable to all incoming requests. This happens when: all pod readiness probes fail (e.g., RDS is down, so /actuator/health returns DOWN), during deployment when old pods terminate before new pods pass probes (prevented by `maxUnavailable: 0`), or when all pods OOMKill and restart simultaneously. Mitigation: proper PDB, correct probe thresholds, min-replicas â‰¥ 2, HPA to add capacity under load.

**Q276. How does CloudFront origin failover work if ALB is unavailable?**
CloudFront supports origin groups with primary + secondary origins and failover criteria (HTTP status codes). If the primary origin (ALB) returns 5xx, CloudFront retries the request on the secondary origin. FleetOps doesn't currently implement origin failover â€” a simpler mitigation is ALB multi-AZ (ALB is already multi-AZ by default) and EKS nodes across 3 AZs so pod failures in one AZ don't take down the service.

**Q277. What is an SQS dead-letter queue and could FleetOps benefit from one?**
DLQ: after N failed delivery attempts on an SQS queue, messages are moved to the DLQ for inspection/retry. FleetOps doesn't use SQS directly â€” SNS delivers to Lambda and email. But if Lambda fails to process an alert event N times (configurable), SNS moves it to a DLQ. This prevents event loss during Lambda errors. An SQS subscription with DLQ would be a resilience improvement over direct Lambda subscription.

**Q278. What is the difference between synchronous and asynchronous Lambda invocation?**
Synchronous: caller waits for Lambda response â€” e.g., API Gatewayâ†’Lambda, ALBâ†’Lambda. Errors returned to caller. Asynchronous: caller returns immediately (202 Accepted) â€” e.g., EventBridgeâ†’Lambda, SNSâ†’Lambda. Lambda retries on failure (up to 2 retries). After retries exhausted, event goes to DLQ (if configured) or is lost. FleetOps: EventBridge invokes alert-processor Lambda asynchronously â€” caller (EventBridge) doesn't wait for Lambda result.

**Q279. How does Step Functions handle Lambda execution failures?**
State machine definition includes `Catch` and `Retry` fields per Task state. `Retry`: on `Lambda.ServiceException`, retry up to 3 times with exponential backoff. `Catch`: if all retries fail, transition to a `Fail` state or `Fallback` handler. Standard Workflows store all execution history â€” you can inspect each state transition, input/output, and error in the console. Failed executions can be re-driven from the point of failure.

**Q280. What is the Bedrock invocation model for FleetOps?**
FleetOps calls `bedrock:InvokeModel` (synchronous, on-demand) for the Nova Lite model. Request: `POST /model/amazon.nova-lite-v1:0/invoke` with JSON body containing `messages` array and `inferenceConfig`. Response: generated text. No streaming in the basic integration. Cost: per-token pricing (input + output). For production, consider Bedrock Provisioned Throughput (reserved capacity, predictable latency) for consistent SLA.

---

### KUBERNETES â€” continued

**Q281. What is pod topology spread and why would you configure it?**
`topologySpreadConstraints` distributes pods evenly across topology domains (zones, nodes). Example: `maxSkew: 1, topologyKey: topology.kubernetes.io/zone, whenUnsatisfiable: DoNotSchedule` â€” ensures pods are spread across AZs with no more than 1 extra pod in any zone vs. the minimum. With 2 pods across 3 AZs, pods land in 2 different AZs â€” AZ failure takes down at most 1 pod. Without constraints, scheduler may colocate both in the same AZ.

**Q282. How does kubelet manage container startup with init containers?**
Init containers run sequentially before app containers start. Each init container must exit 0 before the next starts. Use cases: wait for a dependency to be ready (`until curl http://db:5432; do sleep 2; done`), run database migrations (Flyway), seed config. If an init container fails, kubelet restarts it according to the pod's `restartPolicy`. App containers don't start until all init containers succeed.

**Q283. What is the containerd runtime and how does it relate to Docker?**
containerd is the industry-standard container runtime (CNCF graduated). Docker itself uses containerd internally. EKS nodes use containerd directly (Docker was deprecated as a K8s runtime in 1.24). kubelet calls containerd via CRI (Container Runtime Interface). containerd calls runc (OCI runtime) to actually create Linux namespaces and cgroups. Image format is OCI-compliant â€” Docker images work without modification.

**Q284. What is a Kubernetes Admission Controller?**
Admission controllers intercept API server requests after authentication/authorization but before persistence. Two types: Mutating (can modify the request â€” e.g., Pod Identity Webhook adds projected volume for OIDC token) and Validating (can approve/reject â€” e.g., reject pods without resource limits). Implemented as webhooks: API server sends AdmissionReview JSON to a webhook server, receives approve/deny/patch response.

**Q285. How does the Pod Identity Webhook work for IRSA?**
The webhook is a MutatingAdmissionWebhook. When a pod is created with a ServiceAccount annotated with `eks.amazonaws.com/role-arn`, the webhook mutates the PodSpec: adds a projected volume (ServiceAccountToken with audience `sts.amazonaws.com`, expiry 86400s) at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`, and sets env vars `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`. The AWS SDK automatically uses these to call STS.

**Q286. What happens when a Secret referenced by a pod doesn't exist?**
If the Secret is referenced as an environment variable (`envFrom: secretRef`) or volume mount, the pod stays in `Pending` state with event: `couldn't find key X in Secret namespace/name`. The pod won't start until the Secret exists. This is why ArgoCD sync waves matter â€” ExternalSecret must reconcile and create the K8s Secret (wave 2) before Deployments start (wave 4). Without ordering, pods fail to start and ESO races to create secrets.

**Q287. How does Kubernetes garbage collection work for completed pods?**
Completed pods (Job pods that finished, or failed pods) accumulate unless cleaned up. `ttlSecondsAfterFinished` on Jobs auto-deletes the pod after N seconds. For Deployment pods: old ReplicaSets are kept up to `revisionHistoryLimit` (default 10). Kubernetes also has a garbage collection controller that removes orphaned resources â€” when a Deployment is deleted, its ReplicaSets and pods are garbage collected.

**Q288. What is etcd and what data does it store for FleetOps?**
etcd is the distributed key-value store that backs the Kubernetes API server â€” all cluster state is persisted here: every Pod, Deployment, Service, ConfigMap, Secret, Ingress, CRD, etc. EKS manages etcd â€” it runs in the control plane (multi-AZ, managed by AWS, not visible to users). Data is encrypted at rest using the KMS key specified in `kms_secrets_key_arn`. etcd uses the Raft consensus algorithm â€” a quorum (majority) of etcd members must agree on each write.

**Q289. How does Kubernetes handle a node failure?**
Node controller detects node as `NotReady` after `node-monitor-grace-period` (40s default). After `pod-eviction-timeout` (5min default), pods on the failed node are evicted (marked for deletion). The ReplicaSet controller sees fewer replicas than desired and schedules replacements on healthy nodes. With PDBs: ensures minimum available replicas during voluntary disruptions. For node failure (involuntary): PDB doesn't block eviction â€” evictions proceed regardless to restore desired replica count.

**Q290. What is a StatefulSet and when would FleetOps need one?**
StatefulSet provides: stable pod names (pod-0, pod-1), stable network identity (DNS: pod-0.service.namespace.svc.cluster.local), ordered deployment/scaling, persistent volume claims per pod (not shared). FleetOps doesn't currently use StatefulSets â€” all services are stateless (Deployments). If running a database (PostgreSQL, Redis) inside K8s, StatefulSets would be required for stable storage. FleetOps offloads statefulness to managed AWS services (RDS, ElastiCache).

---

### TERRAFORM â€” continued

**Q291. What is `for_each` in Terraform and when would you use it over `count`?**
`for_each = var.subnet_map` creates one resource instance per map entry, keyed by the map key. If you remove an item from the middle of a `count` list, Terraform renumbers all subsequent instances and recreates them. `for_each` instances are keyed by the map key â€” removing one entry only affects that specific instance, leaving others untouched. Use `for_each` for any resource where stable identity matters (subnets, IAM policies, security groups).

**Q292. How does Terraform handle sensitive values in state?**
Terraform marks values from `sensitive = true` variables and outputs as `(sensitive)` in CLI output (plan/apply) â€” they're not printed. However, they ARE stored in plaintext in the state file (S3 in FleetOps). The state file must be protected: S3 bucket with encryption (KMS), versioning, block-public-access, access logging, and strict IAM bucket policies. This is the known FleetOps security debt â€” secrets in `prod.auto.tfvars` and state need rotation before evaluation.

**Q293. What is a Terraform data source and give an example from FleetOps?**
Data sources read existing infrastructure without managing it. `data "aws_lb" "fleetops"` looks up the ALB created by the K8s ALB Ingress Controller using tag filters â€” Terraform didn't create it, but reads its DNS name and zone ID to create Route53 alias records. Other examples: `data "aws_caller_identity" "current"` reads the current AWS account ID (used in ARN construction), `data "aws_availability_zones" "available"` reads AZs without hardcoding them.

**Q294. How does Terraform provider `exec` block work for EKS authentication?**
The `helm` and `kubernetes` providers need to authenticate to the EKS API server. The `exec` block runs an external command to generate credentials: `aws eks get-token --cluster-name CLUSTER_NAME` outputs a kubeconfig-compatible token. Terraform runs this command at plan/apply time and uses the output as the Bearer token for API calls. This avoids storing cluster credentials in Terraform config.

**Q295. What is the purpose of `try()` in the provider configuration?**
`try(module.eks_cluster.cluster_endpoint, "")` returns the first argument if it's not an error, otherwise returns the fallback. During initial `terraform plan` (before eks_cluster exists), `module.eks_cluster.cluster_endpoint` would be null/unknown â€” `try()` returns `""` and the provider initializes without crashing. Without `try()`, the plan would fail because the provider can't initialize with a null endpoint.

---

### CI/CD â€” continued

**Q296. What is a GitHub Actions matrix strategy?**
`strategy.matrix` runs a job multiple times with different variable combinations. Example: `matrix: service: [auth, vehicle, maintenance, request, frontend]` â€” runs 5 parallel jobs, each with a different `service` value. Used in FleetOps to build and push all 5 services in parallel rather than sequentially â€” reduces total CI time from ~NÃ—M minutes to ~M minutes (longest single build).

**Q297. How does `actions/cache` differ from Docker Buildx GHA cache?**
`actions/cache`: caches arbitrary files between workflow runs (Maven `.m2` repository, node_modules). Keyed by a cache key (e.g., hash of `pom.xml`). Restored at job start, saved at job end. Docker Buildx GHA cache: caches Docker layer blobs in the GitHub Actions cache backend â€” not individual files but BuildKit content-addressed layers. Both use the same underlying cache storage but serve different purposes.

**Q298. What is a GitHub Actions self-hosted runner and does FleetOps use one?**
Self-hosted runners run on your own infrastructure (EC2, EKS pod, on-prem server). Advantages: access to private VPC resources, persistent cache, custom hardware (GPU), no GitHub-hosted runner time limits. FleetOps uses GitHub-hosted runners (`ubuntu-latest`) â€” simpler, no infrastructure to manage, but no direct VPC access. Lambda and EKS deployments use OIDC credentials to reach AWS APIs â€” no VPC access needed.

**Q299. How would you implement a canary deployment for FleetOps?**
Option 1 (ArgoCD Rollouts): install Argo Rollouts, use `Rollout` resource instead of `Deployment`, specify canary strategy: send 10% traffic to new version, pause, increment to 50%, pause, 100%. Traffic split via ALB weighted target groups. Option 2 (manual): deploy new version as separate Deployment with 1 replica alongside existing (10 replicas), monitor errors, gradually shift traffic by adjusting replica counts. ArgoCD Rollouts is the GitOps-native approach.

**Q300. What happens if the GitHub Actions OIDC token is stolen?**
The token expires in 10 minutes (GitHub OIDC token lifetime) â€” usability window is very short. The IAM role's trust policy has conditions: `StringLike token.actions.githubusercontent.com:sub: repo:FleetOps-V2/*` â€” the stolen token is only valid if used before expiry AND from a role the attacker can assume (they'd need to make STS calls). STS logs show the AssumeRoleWithWebIdentity call with the caller's IP â€” easily detectable in CloudTrail. No long-lived credentials exist to steal.

---

### SECURITY â€” continued

**Q301. What is the OWASP Top 10 and which are most relevant to FleetOps?**
OWASP Top 10 (2021): A01 Broken Access Control, A02 Cryptographic Failures, A03 Injection, A04 Insecure Design, A05 Security Misconfiguration, A06 Vulnerable Components, A07 Auth Failures, A08 Software Integrity Failures, A09 Logging Failures, A10 SSRF. Most relevant to FleetOps: A01 (JWT role enforcement â€” ADMIN endpoints), A02 (plaintext secrets in tfvars â€” known debt), A03 (parameterized JPA queries), A06 (Snyk/Trivy scanning), A07 (BCrypt, JWT expiry, Redis blacklist).

**Q302. How does CORS work and how is it configured in FleetOps?**
CORS (Cross-Origin Resource Sharing): browser blocks cross-origin requests unless the server includes `Access-Control-Allow-Origin` header. Spring Boot services have `@CrossOrigin` or global `CorsConfiguration` with `allowedOrigins: [https://fleetops.website]` (from SSM `cors_allowed_origins`). Preflight OPTIONS requests return `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`. Without CORS config, browser blocks API calls from the React frontend.

**Q303. What is certificate pinning and is it used in FleetOps?**
Certificate pinning: client hardcodes the expected certificate fingerprint or public key for a server â€” rejects connections even if a technically-valid (but unexpected) cert is presented. Protects against CA compromise or MITM with a fraudulent cert. FleetOps does NOT implement pinning â€” it relies on standard TLS certificate validation via ACM and Route53. Mobile apps sometimes pin; web applications typically don't (breaks on cert renewal).

**Q304. How does FleetOps protect against brute force login attempts?**
WAFv2 rate-based rule (2000 requests per 5-minute window per IP) blocks IPs making too many requests â€” limits brute force login attempts. Auth-service could additionally implement account lockout (lock after N failed attempts) or exponential delay. CloudWatch alarm on high 4xx rate to `/api/auth/login` can trigger SNS alert. BCrypt's intentional slowness (~100ms per hash) also limits brute force to ~10 attempts/second per attacker.

**Q305. What is TLS termination and where does it happen in FleetOps?**
TLS termination: decrypt HTTPS at a termination point, then forward as HTTP internally. In FleetOps: first termination at CloudFront edge (TLS from client â†’ CloudFront). CloudFront then connects to ALB via HTTPS (TLS from CloudFront â†’ ALB, using the same ACM cert on the ALB). ALB terminates TLS and forwards HTTP to pod port 8080. Pod-to-pod communication is HTTP (within the cluster private network). End-to-end: client â†” CloudFront â†” ALB â†” Pod.

**Q306. What is least-privilege IAM and how would you audit it?**
Audit steps: (1) CloudTrail â†’ Athena query to find actual API calls per IAM role over 90 days, (2) Compare against policy statements â€” any Action never called is over-permissive, (3) IAM Access Analyzer identifies resources shared externally, (4) IAM policy simulator tests specific actions. Tools: `aws iam generate-service-last-accessed-details` shows last-used date per service. Remove permissions for services not used in 90 days. For FleetOps: verify each IRSA role only calls what it needs.

**Q307. What is a supply chain attack and how does FleetOps defend against it?**
Supply chain attack: compromise a dependency or build tool to inject malicious code into legitimate software. Example: `event-stream` npm package compromise (2018). FleetOps defenses: Snyk scans Maven transitive dependencies for known-compromised packages, Trivy scans the final Docker image for OS-level vulnerabilities, ECR image scanning runs after push, `pom.xml` pins exact versions (no floating `LATEST`), GitHub Actions pins `uses: actions/checkout@v4` (fixed SHA). SigStore/cosign image signing would add provenance verification.

**Q308. What is the difference between authentication and authorization?**
Authentication (AuthN): verify identity â€” "who are you?" JWT token contains `sub` (user ID) and is validated by checking signature + expiry. Authorization (AuthZ): verify permission â€” "are you allowed to do this?" Spring Security `@PreAuthorize("hasRole('ADMIN')")` checks the `role` claim in the JWT. FleetOps: auth-service handles AuthN (issue/validate tokens); each backend service handles AuthZ (check role before executing business logic). These are deliberately separated.

**Q309. What are Kubernetes Secrets security concerns?**
K8s Secrets are base64-encoded (NOT encrypted) by default in etcd â€” easily decoded. Anyone with `kubectl get secret` permission can read values. EKS addresses this with `kms_secrets_key_arn` â€” EKS encrypts Secret values in etcd using KMS (envelope encryption: data encrypted with a data key, data key encrypted with CMK). External Secrets Operator (IRSA) further reduces risk: secrets stay in Secrets Manager; K8s Secret is a temporary copy with an ownerReference that ESO controls.

**Q310. What is the AWS Shared Responsibility Model?**
AWS manages security OF the cloud (physical infrastructure, hypervisor, managed service internals â€” e.g., RDS OS patches, EKS control plane). Customer manages security IN the cloud (IAM policies, security groups, OS patches on EC2/EKS nodes, application code, data encryption, network ACLs). For EKS: AWS manages control plane (API server, etcd); you manage node AMI updates, pod security, RBAC, network policies, application vulnerabilities.

---

### PROJECT-SPECIFIC â€” continued

**Q311. How would you add a new microservice to FleetOps?**
(1) Create GitHub repo with Spring Boot app, Dockerfile, Helm chart or raw manifests. (2) Add ECR repository in Terraform (`module.ecr`). (3) Add IAM role for IRSA with required permissions. (4) Add Secrets Manager secrets and SSM parameters for the service. (5) Add K8s manifests: Deployment, Service, ServiceAccount, HPA, ExternalSecret, ConfigMap. (6) Add Ingress path in `ingress.yaml`. (7) Add GitHub Actions workflow for CI/CD. (8) Add ArgoCD Application manifest. (9) Update ALB security group if new port needed.

**Q312. What would you monitor to detect a security incident in FleetOps?**
CloudTrail: unusual IAM API calls (new user created, policy attached, role assumed from unknown IP). CloudWatch: spike in 401/403 responses (brute force or stolen token reuse), unusual Secrets Manager GetSecretValue calls. WAF: sudden IP block increase (scanning). GuardDuty (not in FleetOps but recommended): ML-based threat detection â€” DNS exfiltration, EC2 cryptocurrency mining, IAM credential compromise. X-Ray: unusual service graph changes (SSRF attempts hitting internal IPs).

**Q313. How would you handle a database migration that requires downtime?**
FleetOps uses Flyway â€” migrations run at application startup before traffic is served (Spring Boot auto-apply). For zero-downtime migrations: (1) Deploy migration that only adds columns/tables (backward compatible), (2) Deploy new code that writes to both old and new columns, (3) Backfill data, (4) Deploy code that reads from new column only, (5) Deploy migration that drops old column. Each step is a separate deployment â€” no downtime. Flyway ensures migrations run exactly once per environment.

**Q314. What is the fleetops-terraform state file and what's the risk if it's deleted?**
State file at `s3://fleetops-terraform-state-johan/environments/prod/terraform.tfstate` tracks all 100+ managed resources with their real AWS IDs and current attribute values. If deleted: Terraform loses knowledge of what it manages. `terraform plan` thinks all resources need to be created â€” running `apply` would try to create duplicates (most would fail with "already exists") or worse, create parallel resources. Recovery: `terraform import` for each resource (tedious). Protection: S3 versioning (enabled) allows restoration of the state file.

**Q315. Describe the end-to-end latency budget for a typical API request.**
DNS: ~1ms (Route53 cached). TLS handshake: ~20ms (CloudFront edge). CloudFrontâ†’ALB: ~5ms (within AWS). ALB target registration: ~1ms. Network to pod: ~1ms (VPC routing). Spring Boot processing: ~10-50ms (typical CRUD with HikariCP + SQL). SQL query: ~2-10ms (RDS, indexed). Redis cache hit: ~1ms. Response back: ~5ms. Total: ~40-90ms P50 for a cache-miss CRUD operation. X-Ray shows the breakdown per segment. SLO target: P99 < 500ms.

**Q316. Why are there four separate GitHub repositories (one per service)?**
Independent deployability: each service has its own CI/CD pipeline triggered by its own commits. A change to vehicle-service triggers only the vehicle-service build â€” not auth-service or frontend builds. Independent versioning: each service has its own semantic version (auth-service v1.5.2, vehicle-service v2.1.0). Blast radius: a broken build in maintenance-service doesn't block vehicle-service deployments. Trade-off: cross-service changes require coordinated PRs; no atomic commits across services.

**Q317. What is the difference between the `fleetops-deployments` repo and the service repos?**
Service repos (`fleetops-auth-service`, etc.): contain application source code, Dockerfile, Maven config, unit tests, CI workflow. `fleetops-deployments`: contains only Kubernetes manifests and Helm values â€” no application code. ArgoCD watches `fleetops-deployments` for changes and syncs to the cluster. This separation is the GitOps pattern: build artifacts (images) come from CI; deployment config comes from a separate repo controlled by platform/ops team. Changes to deploy config don't require rebuilding the application.

**Q318. How does the frontend communicate with backend services?**
React app (running in browser) makes fetch/axios calls to `https://fleetops.website/api/*`. CloudFront routes `/api/*` to the ALB origin. ALB matches the path and forwards to the appropriate backend service pod. The React app sets `Authorization: Bearer <token>` header on authenticated requests. No direct pod IPs or internal DNS â€” everything goes through the CloudFrontâ†’ALBâ†’pod path. During local development, a proxy in `vite.config.ts` forwards `/api/*` to localhost:808x.

**Q319. What would happen if the JWT secret was rotated?**
All existing issued JWTs would immediately become invalid â€” signature verification would fail because the secret no longer matches. All logged-in users would be forced to re-authenticate. In FleetOps: JWT secret is in Secrets Manager â†’ ESO syncs to K8s Secret â†’ pods read via env var (injected at startup). To rotate without mass logout: (1) Support two valid secrets simultaneously (current + previous), (2) Deploy code update, (3) After all old JWTs expire (~1h), remove previous secret. Spring Security's `JwtDecoder` can accept multiple keys.

**Q320. What is the purpose of the `bedrock_access_key` and `bedrock_secret_key` variables in Terraform?**
These are static AWS credentials for Bedrock access â€” passed from `prod.auto.tfvars` into `module.eks_addons` and stored in a K8s Secret or ConfigMap for the service that calls Bedrock's `InvokeModel` API. This is a known security debt â€” static credentials should be replaced with IRSA (add `bedrock:InvokeModel` to the relevant service's IAM role). Static credentials in `tfvars` violate the least-privilege and no-long-lived-credentials principles.

---

### GENERAL ARCHITECTURE â€” continued

**Q321. What is the CAP theorem and how does it apply to FleetOps?**
CAP: distributed systems can guarantee at most two of three: Consistency (all nodes see same data), Availability (every request gets a response), Partition Tolerance (system works despite network splits). RDS (single writer): CP â€” strong consistency, sacrifices availability during failover. ElastiCache Redis (single node): CA â€” consistent and available, but no partition tolerance. S3: highly available and partition-tolerant, eventual consistency for overwrite PUTs. FleetOps accepts brief unavailability (CP) for the database in exchange for data correctness.

**Q322. What is eventual consistency and where does it appear in FleetOps?**
Eventual consistency: all nodes will eventually agree on a value, but there may be a window of inconsistency. In FleetOps: ESO secret sync (Secrets Manager value â†’ K8s Secret) has up to ~1h lag. CloudFront cache (static assets): a deployed frontend update may not be visible at all edge locations instantly (short TTL + cache invalidation mitigates). S3 object listing after a delete may briefly still show the object. RDS read replica (if added): replica lags primary by milliseconds to seconds.

**Q323. What is idempotency and why does it matter for Lambda and Step Functions?**
Idempotency: calling the same operation multiple times produces the same result as calling it once. SNS delivers at-least-once â†’ Lambda may be invoked multiple times with the same event. If Lambda inserts a DB record without idempotency check, duplicates result. Mitigation: use a unique event ID as a primary key or deduplication ID (SNS `MessageDeduplicationId`), check for existing record before insert. Step Functions Standard Workflow execution is idempotent by execution name â€” retrying with the same name returns the existing execution.

**Q324. What is blue-green deployment and how would you implement it on FleetOps?**
Blue-green: maintain two identical production environments; switch traffic between them for zero-downtime releases. On EKS: run blue (current) and green (new) Deployments simultaneously, update the Service selector or Ingress to point to green, verify, then scale blue to 0. Or use ALB weighted target groups (via Argo Rollouts). More expensive (double infrastructure during transition) than rolling updates, but provides instant rollback: just switch the selector back to blue.

**Q325. What is service mesh and does FleetOps use one?**
A service mesh (Istio, Linkerd, AWS App Mesh) adds a sidecar proxy to every pod that handles: mTLS between services, fine-grained traffic control (canary, circuit breaker, retry), distributed tracing, and telemetry â€” without changing application code. FleetOps does NOT use a service mesh â€” it's an additional complexity layer appropriate for large-scale microservice deployments with many teams. X-Ray SDK provides tracing without a mesh. mTLS between pods is not implemented â€” pods communicate over plain HTTP within the cluster.

**Q326. What is observability's three pillars and how are they covered in FleetOps?**
(1) Metrics: CloudWatch metrics â€” ALB request count/latency/5xx, EKS CPU/memory, RDS connections/CPU, custom application metrics via CloudWatch SDK. (2) Logs: Fluent Bit DaemonSet â†’ CloudWatch Logs â€” all container stdout/stderr, structured JSON. (3) Traces: AWS X-Ray â€” distributed traces from ALB through Spring Boot services to RDS/Redis, showing per-service latency. Together: metrics alert you something is wrong, logs show what happened, traces show where in the call chain the issue is.

**Q327. What is a circuit breaker pattern and how would you add it to FleetOps?**
Circuit breaker: when a downstream service is failing repeatedly, stop calling it for a period (open circuit) instead of continuing to fail slowly. Three states: Closed (normal), Open (calls rejected immediately), Half-Open (test call â€” if succeeds, close; if fails, stay open). Resilience4j is the standard Java library. Example: vehicle-service calls Redis; if Redis is down, circuit opens, calls return cached-null or default, preventing thread pool exhaustion from slow Redis timeouts.

**Q328. What would change in the architecture if FleetOps needed to serve 100x more traffic?**
Database: RDS â†’ Aurora Global Database (multi-region read replicas, writes to primary region). Cache: Redis â†’ ElastiCache cluster mode (sharded, auto-scaling). EKS: larger instance types, cluster autoscaler, multi-region EKS clusters behind Route53 latency routing. CDN: CloudFront already scales automatically. Lambda: provisioned concurrency to eliminate cold starts at scale. ALB: already auto-scales. Application: review connection pool sizes (HikariCP), add Redis caching for read-heavy paths, introduce message queues (SQS) to absorb write spikes.

**Q329. What is infrastructure as code and what are its benefits over manual provisioning?**
IaC: infrastructure defined in code (Terraform HCL in FleetOps) â€” versioned, reviewed, and applied reproducibly. Benefits: (1) Reproducibility â€” `terraform apply` creates identical environments, (2) Version control â€” every change is a Git commit with author, reason, and diff, (3) Drift detection â€” `terraform plan` shows if manual changes were made, (4) Disaster recovery â€” destroy and recreate entire infrastructure in 45min from code, (5) Code review â€” infrastructure changes go through PR review like application code, (6) Automation â€” CI/CD pipeline for infrastructure changes.

**Q330. How does cost optimization align with the architecture choices in FleetOps?**
t3.micro/small instances (RDS, Redis): low-cost for training environment. Single-AZ ElastiCache (no replication): saves ~50% vs multi-AZ. EKS managed nodes with spot instances (optional): 70-90% savings over on-demand. S3 intelligent tiering: automatically moves infrequently accessed objects to cheaper storage class. ECR lifecycle policies: delete old images to reduce storage cost. Lambda (pay-per-invocation): zero cost when no alerts fire. CloudFront: reduces origin load (fewer requests to ALB = lower EC2 compute cost).

---

### AWS DEEP DIVE â€” continued

**Q331. What is an ENI (Elastic Network Interface) and how does EKS use multiple ENIs?**
ENI: virtual network card attached to an EC2 instance. Each ENI has a primary private IP and can have secondary private IPs. EKS VPC CNI attaches multiple ENIs to each node: ENI 1 (primary, node IP), ENI 2-N (secondary, allocated for pods). Each secondary IP is assigned to one pod. `m5.large` supports up to 3 ENIs Ã— 10 IPs = 30 pods. When ENI capacity is exhausted, VPC CNI allocates a new ENI. The IPAMD daemon on each node manages IP allocation.

**Q332. What is AWS PrivateLink and could FleetOps use it?**
PrivateLink creates a private endpoint for an AWS service inside your VPC â€” traffic never leaves the AWS network. Example: EKS pods calling Secrets Manager via a VPC endpoint don't traverse the internet or NAT Gateway. Benefits: no NAT Gateway data processing cost for AWS API calls, reduced attack surface, consistent latency. FleetOps could add VPC endpoints for: Secrets Manager, SSM, ECR (especially for large image pulls), S3, STS â€” each saves NAT Gateway data processing charges (~$0.045/GB).

**Q333. What is an SQS FIFO queue vs. standard queue?**
Standard: at-least-once delivery, best-effort ordering, unlimited throughput. FIFO: exactly-once processing, strict ordering within a message group, 300 messages/second (3000 with batching). FleetOps uses SNS (not SQS) for event delivery to Lambda â€” Lambda handles at-least-once with idempotency. If FleetOps needed guaranteed ordering for service request processing (e.g., status updates must apply in sequence), an SQS FIFO queue subscribed to SNS would be appropriate.

**Q334. How does AWS auto-scaling of Lambda concurrency work?**
Lambda scales by adding concurrent execution environments. Burst limit: 3000 initial burst, then +500/minute. If all environments are busy (no available slot within 1 second), Lambda returns a throttle error (429). Reserved concurrency: cap the maximum concurrent executions for a function (protects downstream services from being overwhelmed). Provisioned concurrency: pre-warm N environments â€” they're always ready (no cold start), billed per hour even when idle.

**Q335. What is DynamoDB and how does FleetOps use it?**
DynamoDB is AWS's serverless key-value/document database. FleetOps uses it exclusively as the Terraform state lock table (`fleetops-terraform-locks`). The table has a `LockID` primary key (String). Terraform writes a lock record at the start of plan/apply and deletes it on completion â€” prevents concurrent Terraform operations from corrupting the state file. Not used for application data â€” all application data is in RDS PostgreSQL.

**Q336. What is the ALB connection draining (deregistration delay)?**
When a target is deregistered from an ALB target group (pod being terminated during rolling update), ALB stops sending new requests to it but continues processing in-flight requests for `deregistration_delay.timeout_seconds` (default 300s, typically set to 30s for fast rolling updates). After the delay, ALB closes connections. The `preStop` hook in the pod spec should sleep for deregistration delay duration to ensure ALB has deregistered before the process exits.

**Q337. What is CloudWatch Logs Insights and how would you use it for FleetOps?**
CloudWatch Logs Insights: interactive query language for log groups. Example queries:
- `fields @timestamp, level, message | filter level = "ERROR" | sort @timestamp desc | limit 50` â€” find recent errors.
- `fields @timestamp, traceId | filter @message like /auth-service/ | stats count(*) by bin(5m)` â€” request rate per 5 minutes.
- `filter @message like /CRITICAL/ | stats count(*) by bin(1m)` â€” critical alert rate.
Useful for debugging incidents: correlate timestamps across services by filtering all log groups in a time window.

**Q338. How does Route53 health checking work for failover routing?**
Route53 can perform HTTP/HTTPS health checks against an endpoint. Failover routing: primary record has an associated health check; if health check fails, Route53 serves the secondary record (failover). FleetOps doesn't implement Route53 failover â€” CloudFront and ALB provide HA. If added: `fleetops.website` primary â†’ CloudFront; secondary â†’ static S3 maintenance page. Health check interval: 30s; failure threshold: 3 consecutive failures.

**Q339. What is AWS GuardDuty and how would it complement FleetOps security?**
GuardDuty: ML-based threat detection service. Analyzes CloudTrail, VPC Flow Logs, DNS logs. Detects: EC2 instance making DNS queries to known malicious domains (malware C2), unusual IAM API calls from new geolocation (credential theft), cryptocurrency mining activity, port scanning from pods. Not in FleetOps currently â€” adding it would cost ~$4/month for the data volume. Findings trigger CloudWatch Events â†’ SNS alert.

**Q340. How does the ALB handle HTTP to HTTPS redirect?**
`alb.ingress.kubernetes.io/ssl-redirect: "443"` annotation causes the ALB Ingress Controller to create an HTTP:80 listener with a redirect action to HTTPS:443. Any HTTP request gets a 301 redirect response with `Location: https://...`. The HTTPS:443 listener terminates TLS (ACM cert) and forwards to pods. This is handled entirely at the ALB â€” no application code involved.

---

### SECURITY â€” continued

**Q341. What is the principle of defense in depth?**
Defense in depth: multiple independent security layers so that breaching one layer doesn't give full access. In FleetOps: (1) WAF filters malicious HTTP patterns, (2) CloudFront origin header (`Host: origin.fleetops.website`) prevents direct ALB access, (3) ALB security group allows only CloudFront IP ranges, (4) Pod security groups restrict pod-to-pod and pod-to-RDS traffic, (5) IRSA limits each pod's AWS permissions, (6) JWT validates user identity, (7) Spring `@PreAuthorize` enforces role-based access, (8) RDS in private subnet with SG. Attacker must breach all layers.

**Q342. What is a penetration test and would you recommend one for FleetOps?**
A pentest is an authorized simulated attack to find exploitable vulnerabilities before malicious actors do. For a production FleetOps deployment: yes â€” especially before external launch. Scope: web app (OWASP Top 10), API (auth bypass, injection, broken object level auth), infrastructure (exposed ports, SG misconfigurations), container escape (pod privilege escalation). AWS permits pentesting for most services without prior approval, except DDoS simulation (needs explicit approval).

**Q343. How does TLS 1.3 differ from TLS 1.2 and which does CloudFront support?**
TLS 1.3: fewer round trips (0-RTT or 1-RTT handshake vs 2 in TLS 1.2), removed weak cipher suites (RC4, DES, 3DES, SHA-1), forward secrecy mandatory (ECDHE key exchange always), faster. CloudFront supports TLS 1.3 with `TLSv1.2_2021` security policy (also supports TLS 1.2 for compatibility). FleetOps should use at minimum `TLSv1.2_2021` â€” this drops TLS 1.0 and 1.1 (vulnerable to POODLE, BEAST attacks).

**Q344. What is secrets sprawl and how does ESO help?**
Secrets sprawl: credentials scattered across config files, environment variables, CI/CD variables, developer laptops, Slack messages. Hard to audit, rotate, or revoke. ESO with Secrets Manager centralizes all secrets: single authoritative source, automatic rotation, CloudTrail audit on every access (who called GetSecretValue, when, from which pod via IRSA role), easy revocation (delete the secret or revoke the IRSA role). K8s Secrets are ephemeral copies â€” the canonical value is always in Secrets Manager.

**Q345. What is image signing and why is it important?**
Image signing (Sigstore/cosign, Notary): cryptographically signs a container image so consumers can verify it came from a trusted builder and wasn't tampered with. Workflow: CI builds image, signs it with a private key, pushes signature to OCI registry alongside image. Admission controller (policy engine) verifies signature before allowing pod to start. FleetOps doesn't implement image signing â€” a gap to address for production hardening. Without it, a compromised ECR registry could serve malicious images.

---

### PROJECT-SPECIFIC â€” final questions

**Q346. What is the first thing you would do after a successful `terraform apply` to verify the environment is healthy?**
(1) `kubectl get nodes -n fleetops-prod` â€” verify nodes are Ready, (2) `kubectl get pods -n fleetops-prod` â€” verify all pods Running, (3) `kubectl get pods -n fleetops-system` â€” verify ArgoCD, ESO pods Running, (4) Check ArgoCD UI at argocd.fleetops.website â€” all Applications Synced + Healthy, (5) curl `https://fleetops.website` â€” verify 200 response and frontend loads, (6) curl `https://fleetops.website/api/auth/health` â€” verify backend responds, (7) Check CloudWatch â€” no alarms firing.

**Q347. How would you explain the FleetOps architecture to a non-technical stakeholder?**
"FleetOps is a web application that manages a vehicle fleet. Users access it via a web browser â€” we use Amazon's global network (CloudFront) to serve the website fast worldwide. The app is split into small, specialized programs (microservices) that run on Amazon's managed Kubernetes service (EKS) â€” like shipping containers for software that can scale up automatically when usage spikes. Data is stored in Amazon's managed database (RDS) so we don't have to manage servers. Everything is automated â€” code changes go through automated testing and deploy themselves without manual steps."

**Q348. What is the biggest technical risk in the current FleetOps architecture?**
Single RDS instance with no read replica: if it goes down, the entire application loses data persistence â€” all write operations fail. Mitigation priority: (1) Enable Multi-AZ for automatic failover (~2min downtime vs total outage), (2) Add a read replica for vehicle-service and request-service (read-heavy). Second risk: plaintext secrets in `terraform.tfvars` â€” if the file is committed to a public repo, all credentials are exposed. Third risk: no Redis HA â€” cache failure increases RDS load significantly.

**Q349. How does the team collaborate on infrastructure changes?**
Terraform changes follow the GitOps pattern: (1) Feature branch with Terraform changes, (2) PR triggers `terraform plan` in CI â€” plan output posted as PR comment, (3) Checkov security scan must pass, (4) PR reviewed and approved by a team member, (5) Merge to main triggers `terraform apply` in CI with OIDC credentials. No developer has `terraform apply` permissions on prod directly â€” all changes go through CI. This provides audit trail, review process, and prevents manual drift.

**Q350. What would you change about FleetOps V2 if starting fresh?**
(1) IRSA for Bedrock from day one â€” no static credentials, (2) Secrets Manager for all sensitive values â€” no plaintext in tfvars, (3) Redis cluster mode with at least one replica â€” HA by default, (4) RDS Multi-AZ enabled from the start â€” the cost is worth the reliability, (5) Image signing with Sigstore/cosign â€” supply chain security, (6) VPC endpoints for ECR/Secrets Manager/S3 â€” eliminate NAT Gateway cost for AWS API calls, (7) Terraform workspaces or separate state per environment â€” cleaner multi-env management, (8) mTLS between pods via service mesh (Linkerd) â€” zero-trust networking.

---

*End of Q166â€“Q350 supplement. Total Q&A count: approximately 350 questions.*

