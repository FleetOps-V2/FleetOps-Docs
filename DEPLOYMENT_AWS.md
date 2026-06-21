# FleetOps V2 — Deployment Guide

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Terraform | >= 1.9 | Infrastructure provisioning |
| AWS CLI | >= 2.x | AWS authentication |
| kubectl | >= 1.28 | Cluster management |
| helm | >= 3.x | Chart management |
| git | any | Source control |

AWS credentials must have sufficient permissions to create EKS, RDS, VPC, IAM, and all related resources. The CI/CD pipeline uses OIDC — no static keys needed in GitHub.

---

## GitHub Secrets Required

These must be set in the GitHub organization or per-repo before any pipeline can run:

| Secret Name | Value |
|---|---|
| `AWS_ACCOUNT_ID` | `538661800892` |
| `AWS_REGION` | `us-east-1` |
| `GITHUB_PAT_FOR_DEPLOYMENTS` | Personal Access Token with `repo` scope (for pushing to fleetops-deployments) |
| `ARGOCD_SERVER` | `argocd.fleetops.website` |
| `ARGOCD_AUTH_TOKEN` | ArgoCD API token for syncing apps from pipeline |

---

## First-Time Deployment (Bootstrap Sequence)

### Step 1 — Bootstrap Terraform Remote State

The S3 bucket and DynamoDB table for Terraform state must exist before anything else.

```bash
cd fleetops-terraform/bootstrap/
terraform init
terraform apply
```

This creates:
- S3 bucket: `fleetops-prod-terraform-state`
- DynamoDB table: `fleetops-prod-terraform-locks`

### Step 2 — Deploy All Infrastructure

```bash
cd fleetops-terraform/environments/prod/
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

This takes approximately 20–30 minutes. It creates:
- VPC, subnets, NAT gateway
- EKS cluster + managed node group
- RDS PostgreSQL (encrypted)
- ElastiCache Redis
- ACM certificate (DNS-validated via Route53)
- CloudFront distribution
- WAF, CloudTrail, Config
- IAM roles (IRSA, GitHub Actions OIDC)
- Secrets Manager secrets
- ArgoCD root Application (via `kubernetes_manifest` resource)

### Step 3 — Update ALB DNS

After apply completes, the ALB DNS name is output. Update `prod.auto.tfvars`:

```hcl
origin_alb_dns = "k8s-fleetops-XXXXXXXX.us-east-1.elb.amazonaws.com"
```

Run `terraform apply` again — this creates the Route53 alias records for `origin.fleetops.website` and `argocd.fleetops.website`.

### Step 4 — Update Domain Nameservers

After Route53 creates the hosted zone, get the nameservers:

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id <ZONE_ID> \
  --query "ResourceRecordSets[?Type=='NS'].ResourceRecords[*].Value" \
  --output text
```

Update your domain registrar to point `fleetops.website` to these 4 nameservers. DNS propagation takes 5–30 minutes.

### Step 5 — Trigger Service Deployments

Push any trivial change to each service repo (or manually trigger the GitHub Actions workflow). This builds Docker images, pushes to ECR, and updates the image tag in `fleetops-deployments`. ArgoCD then deploys each service automatically.

After all pipelines complete:

```bash
kubectl get applications -n argocd
# All 11 apps should be: Synced + Healthy

kubectl get pods -n fleetops-prod
# All pods should be: 1/1 Running
```

---

## Eval Day Apply (Destroy → Recreate)

After a `terraform destroy`, follow this sequence on eval day:

### 1. Apply infrastructure (~25 min)
```bash
cd fleetops-terraform/environments/prod/
terraform apply -auto-approve
```

### 2. Get the new ALB DNS
```bash
terraform output -raw alb_dns_name
# or check: kubectl get ingress -n fleetops-prod
```

### 3. Update origin_alb_dns in prod.auto.tfvars
```hcl
origin_alb_dns = "<new-ALB-dns>"
```

### 4. Apply again to create Route53 DNS records
```bash
terraform apply -auto-approve
```

### 5. Update domain registrar NS records (if new hosted zone)
Get new nameservers from Route53 and update at your registrar.

### 6. Verify everything is up
```bash
kubectl get applications -n argocd
kubectl get pods -n fleetops-prod
curl https://fleetops.website/actuator/health  # should return 200
```

---

## CI/CD Pipeline Details

### Service Pipeline (per microservice)

Located in each service repo under `.github/workflows/`. Uses shared templates from `fleetops-github-workflows/`.

```
Trigger: push to main
Steps:
  1. Checkout
  2. Set up Java 21 / Node 20
  3. Run tests (Maven / npm test)
  4. Configure AWS credentials via OIDC
  5. Login to ECR
  6. Build Docker image
  7. Push image with tags: <git-sha>, latest
  8. Update image.tag in fleetops-deployments/charts/<service>/values.yaml
  9. Commit and push to fleetops-deployments
  10. ArgoCD auto-syncs (automated sync policy with selfHeal)
```

### Terraform Pipeline

Located in `fleetops-terraform/.github/workflows/terraform-apply.yml`.

```
Trigger: push to main (environments/prod/ path)
Steps:
  1. Checkout
  2. Configure AWS credentials via OIDC
  3. terraform init
  4. terraform plan → save tfplan artifact
  5. Manual approval (environment protection rule)
  6. terraform apply tfplan
  7. Extract outputs → write infra-values.yaml to fleetops-deployments
```

---

## Useful Commands

### Check cluster health
```bash
aws eks update-kubeconfig --name fleetops-prod-eks --region us-east-1
kubectl get nodes
kubectl get pods -n fleetops-prod
kubectl get applications -n argocd
```

### Force ArgoCD sync
```bash
kubectl -n argocd patch application <app-name> \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

### Check service logs
```bash
kubectl logs -n fleetops-prod -l app=fleetops-vehicle-service --tail=100
kubectl logs -n fleetops-prod -l app=fleetops-auth-service --tail=100
```

### Check secrets are synced
```bash
kubectl get externalsecrets -n fleetops-prod
kubectl get secrets -n fleetops-prod
```

### Check Terraform state
```bash
cd fleetops-terraform/environments/prod/
terraform state list
terraform output
```

### Access ArgoCD CLI
```bash
argocd login argocd.fleetops.website
argocd app list
argocd app sync fleetops-vehicle-prod
```

---

## Pre-Evaluation Checklist

Before the evaluation session, complete these steps:

- [ ] Rotate GitHub PAT — generate new token, update GitHub Secret + `terraform.tfvars`
- [ ] Remove plaintext secrets from `terraform.tfvars` (move values to Secrets Manager or use env vars)
- [ ] Remove static Bedrock credentials — switch to IRSA (already set up, just remove static keys)
- [ ] Run `terraform apply` and verify all resources are up
- [ ] Confirm `fleetops.website` returns 200
- [ ] Confirm ArgoCD shows all 11 apps Synced + Healthy
- [ ] Confirm all 10 pods are 1/1 Running in `fleetops-prod` namespace
- [ ] Test AI Fleet Advisor via the UI
- [ ] Test login, vehicle CRUD, maintenance tasks, service requests end-to-end

---

## Troubleshooting

### 502 Bad Gateway on fleetops.website
1. Check pods: `kubectl get pods -n fleetops-prod`
2. Check Ingress: `kubectl describe ingress -n fleetops-prod`
3. Check ArgoCD apps are all Synced
4. Check ALB target group health in AWS console
5. Verify `origin_alb_dns` in `prod.auto.tfvars` matches current ALB

### ArgoCD app OutOfSync
```bash
kubectl describe application <app-name> -n argocd | grep -A5 "Conditions"
# Read the sync error message, fix the manifest in fleetops-deployments, push
```

### Pod CrashLoopBackOff
```bash
kubectl logs -n fleetops-prod <pod-name> --previous
kubectl describe pod -n fleetops-prod <pod-name>
# Usually: secret not synced, wrong env var, DB connection issue
```

### Secrets not syncing (ESO)
```bash
kubectl get clustersecretstore
kubectl describe externalsecret fleetops-postgres-secret -n fleetops-prod
# Check IRSA role has correct Secrets Manager permissions
```
