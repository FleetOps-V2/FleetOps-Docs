# ☁️ AWS Production Provisioning & Deployment Guide

This guide details the step-by-step instructions to deploy the FleetOps microservices platform to Amazon Web Services (AWS). It covers networking, managed database provisioning, container registries, and two deployment pathways:
1.  **Staging Deployments:** A single-instance EC2 runner running Docker Compose.
2.  **Production Deployments:** A highly-available, serverless AWS ECS Fargate cluster behind an Application Load Balancer.

---

## 🏗️ Target AWS Architecture

```text
                             [ Client Browser ]
                                     │ HTTPS (Port 443)
                                     ▼
                      [ Amazon Route 53 (DNS) ]
                                     │
                                     ▼
                  [ AWS Certificate Manager (ACM SSL) ]
                                     │
                                     ▼
                [ Application Load Balancer (ALB) :80/443 ]
                                     │
            ┌────────────────┬───────┴────────┬────────────────┐
            │ /api/auth/*    │ /api/vehicles/*│ /api/tasks/*   │ /api/requests/*
            ▼                ▼                ▼                ▼
   ┌────────────────┐┌────────────────┐┌────────────────┐┌────────────────┐
   │ ECS Task       ││ ECS Task       ││ ECS Task       ││ ECS Task       │
   │ (Auth Service) ││(Vehicle Service││(Maint. Service)││(Request Service│
   └────────┬───────┘└───────┬────────┘└───────┬────────┘└───────┬────────┘
            │                │                 │                 │
            │                │                 │                 │
            │                │                 │                 │
            ▼                ▼                 ▼                 ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                     AWS Private Database Subnets                     │
   │                                                                      │
   │   ┌───────────────────────────────┐     ┌────────────────────────┐   │
   │   │     Amazon RDS PostgreSQL     │     │ Amazon ElastiCache     │   │
   │   │  (Multi-AZ, Port 5432)        │     │ (Redis Cluster, :6379) │   │
   │   └───────────────────────────────┘     └────────────────────────┘   │
   └──────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step 1: Network Infrastructure (VPC)

Create a custom Virtual Private Cloud (VPC) to isolate backend resources from direct internet access.

1.  **Create VPC:**
    *   **Name:** `fleetops-vpc`
    *   **IPv4 CIDR Block:** `10.0.0.0/16`
2.  **Provision Subnets:** Create subnets across two Availability Zones (e.g., `us-east-1a` and `us-east-1b`) for high availability:
    *   **Public Subnet A:** `10.0.1.0/24` (AZ: `us-east-1a` — for ALB / Gateway)
    *   **Public Subnet B:** `10.0.2.0/24` (AZ: `us-east-1b` — for ALB / Gateway)
    *   **Private App Subnet A:** `10.0.10.0/24` (AZ: `us-east-1a` — for EC2 / ECS Tasks)
    *   **Private App Subnet B:** `10.0.11.0/24` (AZ: `us-east-1b` — for EC2 / ECS Tasks)
    *   **Private DB Subnet A:** `10.0.20.0/24` (AZ: `us-east-1a` — for RDS & Redis)
    *   **Private DB Subnet B:** `10.0.21.0/24` (AZ: `us-east-1b` — for RDS & Redis)
3.  **Configure Routing:**
    *   Create an **Internet Gateway (IGW)** and attach it to `fleetops-vpc`.
    *   Create **NAT Gateways** in the Public Subnets to allow private subnet services to pull updates or communicate downstream.
    *   Configure route tables to map public subnets to the IGW, and private subnets to the NAT Gateways.

---

## 🗄️ Step 2: Database Provisioning (Amazon RDS PostgreSQL)

Configure a production-grade relational database instead of running local database containers.

1.  **Create Subnet Group:** Navigate to the RDS Console, select **Subnet Groups**, and create a group named `fleetops-rds-subnet-group` containing `Private DB Subnet A` and `Private DB Subnet B`.
2.  **Create Security Group:** Create `fleetops-rds-sg`:
    *   **Inbound Rule:** Allow TCP on port `5432` from the app security group (`fleetops-app-sg`).
3.  **Launch Database:**
    *   **Engine:** PostgreSQL (version 15.x recommended)
    *   **Template:** Dev/Test (or Production with Multi-AZ enabled)
    *   **DB Instance Identifier:** `fleetops-db-postgres`
    *   **Master Username:** `fleetops_master`
    *   **Master Password:** Generate a secure password.
    *   **DB Subnet Group:** Select `fleetops-rds-subnet-group`.
    *   **Public Access:** No.
    *   **Security Group:** Select `fleetops-rds-sg`.
4.  **Create Isolated Databases:** Once RDS is active, connect via a jump box in the public subnet and run:
    ```sql
    CREATE DATABASE auth_db;
    CREATE DATABASE vehicle_db;
    CREATE DATABASE maintenance_db;
    CREATE DATABASE request_db;
    ```

---

## ⚡ Step 3: Cache Cluster Provisioning (Amazon ElastiCache Redis)

Provision a managed Redis cache to replace the internal Docker Redis container.

1.  **Create Subnet Group:** Create an ElastiCache subnet group named `fleetops-redis-subnet-group` referencing `Private DB Subnet A` and `Private DB Subnet B`.
2.  **Create Security Group:** Create `fleetops-redis-sg`:
    *   **Inbound Rule:** Allow TCP on port `6379` from the app security group (`fleetops-app-sg`).
3.  **Launch Cluster:**
    *   **Engine:** Redis OSS
    *   **Node Type:** `cache.t4g.micro` (or larger depending on traffic)
    *   **Number of Replicas:** 0 (Staging) or 1 (Production Multi-AZ)
    *   **Subnet Group:** Select `fleetops-redis-subnet-group`.
    *   **Security Group:** Select `fleetops-redis-sg`.
    *   Take note of the Primary Endpoint (e.g., `fleetops-redis.xxxx.cache.amazonaws.com`).

---

## 🔒 Step 4: AWS Secrets Manager Setup

Store database passwords and JWT secrets securely to avoid committing plain-text keys to task definitions.

1.  **Create Secret:** Create a secret named `fleetops/production/config`.
2.  **Add Key/Value Pairs:**
    *   `JWT_SECRET` = (Generate a 64-character random string)
    *   `POSTGRES_USER` = `fleetops_master`
    *   `POSTGRES_PASSWORD` = (Your RDS master password)
    *   `DB_HOST` = (Your RDS PostgreSQL Endpoint)
    *   `REDIS_HOST` = (Your ElastiCache Primary Endpoint)

---

## 📦 Step 5: AWS Elastic Container Registry (ECR) Setup

Prepare the cloud container registry to host individual microservice images.

1.  **Create ECR Repositories:** Run these commands using the AWS CLI:
    ```bash
    aws ecr create-repository --repository-name fleetops-auth-service
    aws ecr create-repository --repository-name fleetops-vehicle-service
    aws ecr create-repository --repository-name fleetops-maintenance-service
    aws ecr create-repository --repository-name fleetops-request-service
    aws ecr create-repository --repository-name fleetops-gateway
    ```
2.  **Build & Push Images:** Build the production Docker images locally or via CI/CD, tag them, and push them to ECR:
    ```bash
    # Authenticate Docker to ECR (replace AWS account id and region)
    aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

    # Build and Push Auth Service
    cd fleetops-auth-service
    docker build -t fleetops-auth-service .
    docker tag fleetops-auth-service:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/fleetops-auth-service:latest
    docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/fleetops-auth-service:latest
    ```
    Repeat this build-tag-push flow for all 4 services and the Nginx gateway.

---

## 🛠️ Step 6: EC2 Deployment Guide (Staging / Cost-Effective Option)

To run the system on a single virtual server (EC2 instance) using Docker Compose (ideal for cost-effective development, user testing, or staging):

### 1. Launch the EC2 Instance
*   **AMI:** Ubuntu Server 22.04 LTS or Amazon Linux 2023.
*   **Instance Type:** `t3.medium` (2 vCPUs, 4GB RAM — needed to support 5 Java containers + Nginx).
*   **Network:** `fleetops-vpc` -> `Public Subnet A`. Enable public IP auto-assignment.
*   **Security Group (`fleetops-ec2-sg`):**
    *   Inbound TCP port `22` (SSH) from your IP.
    *   Inbound TCP port `8080` (API Gateway) from Anywhere (`0.0.0.0/0`).
*   **IAM Role:** Attach an IAM Role with the `AmazonEC2ContainerRegistryReadOnly` and `SecretsManagerReadWrite` policy permissions.

### 2. Configure the EC2 Host
Connect to the server via SSH and install Docker and Git:
```bash
sudo apt-get update -y
sudo apt-get install -y docker.io git jq
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
# Log out and log back in to apply group membership
```

### 3. Deploy the Compose Application
Clone the repository, fetch the credentials from Secrets Manager, write them to `.env`, and start the containers:
```bash
# Clone source code
git clone <your-git-repo-url> fleetops
cd fleetops/fleetops-infra

# Authenticate Docker with AWS ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Fetch secrets and write to .env file
aws secretsmanager get-secret-value --secret-id fleetops/production/config --query SecretString --output text | jq -r 'to_entries|map("\(.key)=\(.value)")|.[]' > .env

# Inject cloud database endpoints into .env
echo "DB_HOST=fleetops-db-postgres.xxxxxx.us-east-1.rds.amazonaws.com" >> .env
echo "REDIS_HOST=fleetops-redis.xxxxxx.cache.amazonaws.com" >> .env
echo "DB_PORT=5432" >> .env

# Edit docker-compose.yml to point image fields to ECR instead of local builds:
# Example: image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/fleetops-auth-service:latest

# Build and run
docker compose up -d
```

---

## 🚀 Step 7: Production Deployments using AWS ECS Fargate

For production, deploy the containers serverlessly using Amazon ECS Fargate, removing EC2 cluster management overhead.

### 1. Create ECS Cluster
Navigate to ECS, and click **Create Cluster**:
*   **Cluster Name:** `fleetops-production`
*   **Infrastructure:** AWS Fargate (Serverless)

### 2. Create Task Definitions
Create a Task Definition for each of the 4 microservices. The configuration for **Auth Service** is outlined below:
*   **Task Definition Name:** `fleetops-auth-task`
*   **Infrastructure Requirements:**
    *   Operating System/Architecture: Linux/ARM64 or X86_64
    *   Task Size: `0.5 vCPU` and `1 GB RAM` (sufficient for JVM operation)
*   **Task Role & Task Execution Role:** Select standard ECS role with policies enabling:
    *   Read access to ECR repositories.
    *   Read access to Secrets Manager (`fleetops/production/config`).
    *   Write access to CloudWatch Logs (`logs:CreateLogStream`, `logs:PutLogEvents`).
*   **Container Declarations:**
    *   **Name:** `auth-service`
    *   **Image:** `123456789012.dkr.ecr.us-east-1.amazonaws.com/fleetops-auth-service:latest`
    *   **Port Mappings:** Container Port: `8080`, Protocol: TCP.
    *   **Environment Variables:**
        *   Retrieve values from Secrets Manager keys dynamically:
            *   `SPRING_DATASOURCE_URL` = `jdbc:postgresql://<secrets_manager_DB_HOST>:5432/auth_db`
            *   `SPRING_DATASOURCE_USERNAME` = Ref `POSTGRES_USER`
            *   `SPRING_DATASOURCE_PASSWORD` = Ref `POSTGRES_PASSWORD`
            *   `JWT_SECRET` = Ref `JWT_SECRET`
        *   Hardcoded Variables:
            *   `SPRING_PROFILES_ACTIVE` = `prod` (deactivates developer seeding)
            *   `JAVA_OPTS` = `-Xms256m -Xmx512m`
    *   **Log Configuration:** Enable awslogs utility, mapping output to a CloudWatch log group named `/ecs/fleetops-auth-service`.

Repeat this task configuration for:
*   `vehicle-service` (linking `REDIS_HOST` dynamically).
*   `maintenance-service` (database link).
*   `request-service` (linking `VEHICLE_SERVICE_URL` and `MAINTENANCE_SERVICE_URL` to their internal Service Discovery endpoints).

### 3. Configure Service Discovery (AWS Cloud Map)
To resolve microservice names internally without exposing services to the public internet, enable **ECS Service Discovery** on each Service:
*   Create a private DNS namespace: `fleetops.local`.
*   When provisioning the ECS Services:
    *   Auth service → registers as `auth.fleetops.local`.
    *   Vehicle service → registers as `vehicle.fleetops.local`.
    *   Maintenance service → registers as `maintenance.fleetops.local`.
    *   Request service → registers as `request.fleetops.local`.
*   Update `request-service` task definition configuration URLs:
    *   `VEHICLE_SERVICE_URL` = `http://vehicle.fleetops.local:8080`
    *   `MAINTENANCE_SERVICE_URL` = `http://maintenance.fleetops.local:8080`

### 4. Create ECS Services
Launch the tasks inside the cluster:
*   **Launch Type:** Fargate
*   **Deployment Option:** Service (replica count: 2 for high availability)
*   **Networking:**
    *   VPC: `fleetops-vpc`
    *   Subnets: `Private App Subnet A` and `Private App Subnet B`
    *   Security Group: Allow inbound traffic on port `8080` from the ALB Security Group.
    *   Public IP: Disabled (routes out via NAT Gateway).

### 5. Application Load Balancer Routing Setup
Configure an AWS Application Load Balancer (ALB) to route external user requests to the appropriate Fargate service:

1.  **Launch ALB:**
    *   **Scheme:** Internet-facing
    *   **Subnets:** `Public Subnet A` and `Public Subnet B`
    *   **Security Group:** Allow HTTP (port 80) and HTTPS (port 443) from Anywhere (`0.0.0.0/0`).
2.  **Define Target Groups:** Create 5 Target Groups in ECS, all targeting `IP` addresses on port `8080` (except frontend on port 80):
    *   `tg-auth-service` (Health path: `/actuator/health`)
    *   `tg-vehicle-service` (Health path: `/actuator/health`)
    *   `tg-maintenance-service` (Health path: `/actuator/health`)
    *   `tg-request-service` (Health path: `/actuator/health`)
    *   `tg-frontend` (Health path: `/`)
3.  **Configure Listener Routing Rules:** Setup Path-based Routing on your ALB listener:
    *   Path Pattern `/api/auth/*` → Forward to `tg-auth-service` (strip `/api` or rewrite prefix if necessary, or let Spring controllers map `/api/auth` directly).
    *   Path Pattern `/api/vehicles/*` → Forward to `tg-vehicle-service`.
    *   Path Pattern `/api/requests/*` → Forward to `tg-request-service`.
    *   Path Pattern `/api/tasks/*` → Forward to `tg-maintenance-service`.
    *   Default Rule (matches all other requests `/*`) → Forward to `tg-frontend`.

### 6. Frontend hosting on AWS S3 + CloudFront (Highly Recommended Option)
Instead of serving static files via an Nginx container task inside ECS, deploy the built React assets serverlessly to AWS S3:

1.  **Upload Assets:** Compile the assets using `npm run build` and upload the files to an **Amazon S3 Bucket** configured for static website hosting.
2.  **Configure CloudFront CDN:** Connect an **Amazon CloudFront Distribution** pointing to the S3 bucket as its origin.
3.  **Add Behavior Rules to CDN:**
    *   Behavior matching path `/api/*` → Forward to the Application Load Balancer.
    *   Default Behavior (`*`) → Route directly to S3.
    *   *Custom Error Response:* Map HTTP 404 to `/index.html` with a 200 OK status to support React Router client-side routing.

---

## 📈 Step 8: Health Checks & Validation

Once deployed:
1.  **Monitor Target Groups:** Check that all ECS targets display a `Healthy` status.
2.  **Verify Endpoints:** Execute queries against the public ALB DNS or CloudFront domain:
    *   `https://<your-domain>/health/auth`
    *   `https://<your-domain>/api/vehicles` (should return 401/403 if unauthenticated, proving API connectivity).
3.  **Execute Audit Run:** Deploy a temporary staging test script on an EC2 runner to trigger the 33 API test suite against the target production endpoints to guarantee complete operational compliance.

---
*Next Step: Proceed to [Amazon Bedrock AI Co-Pilot Blueprint](./AI_INTEGRATION.md).*
