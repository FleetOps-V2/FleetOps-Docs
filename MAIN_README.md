# 🚚 FleetOps Enterprise — Vehicle Maintenance & Fleet Tracking

FleetOps is a production-grade, containerized vehicle maintenance and fleet tracking platform built with a **microservices architecture**. It features a modern React SPA frontend served via Nginx, four independent Spring Boot backend services, a shared PostgreSQL instance with service-isolated databases, Redis-based fleet lifecycle caching, and JWT stateless authentication.

This repository represents a fully operational development and staging environment that has passed all verification audits.

---

## 📂 Repository Structure

```text
d:\UST Training\Fleetops-V2/
├── README.md                      <-- Root portal (this file)
├── FleetOps-Docs/                 <-- Detailed Documentation Hub
│   ├── ARCHITECTURE.md            <-- Core Architecture, Schemas & Audit Report
│   ├── DEPLOYMENT_AWS.md          <-- Step-by-Step EC2 & ECS AWS Provisioning Guide
│   ├── AI_INTEGRATION.md          <-- Amazon Bedrock AI Co-Pilot Blueprint
│   └── ROADMAP.md                 <-- Multi-Phase Delivery Roadmap
├── fleetops-frontend/             <-- React SPA + Vite Dev Proxy
├── fleetops-auth-service/         <-- Spring Boot: User Auth & JWT Issuer
├── fleetops-vehicle-service/      <-- Spring Boot: Fleet Management & Redis Cache
├── fleetops-maintenance-service/  <-- Spring Boot: Pending Task Queue Staging
├── fleetops-request-service/      <-- Spring Boot: Service Request State Machine
└── fleetops-infra/                <-- Docker Compose orchestration & Nginx Gateway
```

---

## ⚡ Platform Status & Audit Results

Following the final system audit, the FleetOps platform is **fully operational** with all components healthy and communicating.

*   **Status:** `✅ FULLY OPERATIONAL`
*   **Total Audit Tests:** `33 / 33 PASSED`
*   **CORS & Routing:** Resolved via local dev proxy configuration and Nginx gateway.

| Service | Container Name | Technology | Port (Internal) | Status |
| :--- | :--- | :--- | :--- | :--- |
| **API Gateway** | `fleetops-gateway` | Nginx Alpine | `8080` (Host Exposure) | ✅ Healthy |
| **Auth Service** | `fleetops-auth-service` | Spring Boot 3 + Security 6 | `8080` | ✅ Healthy |
| **Vehicle Service** | `fleetops-vehicle-service` | Spring Boot 3 + Redis Cache | `8080` | ✅ Healthy |
| **Request Service** | `fleetops-request-service` | Spring Boot 3 + RestTemplate | `8080` | ✅ Healthy |
| **Maintenance Service** | `fleetops-maintenance-service` | Spring Boot 3 | `8080` | ✅ Healthy |
| **Database** | `fleetops-postgres` | PostgreSQL 15 | `5432` | ✅ Healthy |
| **Cache Store** | `fleetops-redis` | Redis 7 | `6379` | ✅ Healthy |

---

## 🔑 Demo Access Credentials

The database seeds automatically on first container launch. The platform enforces **Role-Based Access Control (RBAC)** across the following preconfigured test accounts:

| Role | Username | Password | Default Landing Page | Permissions |
| :--- | :--- | :--- | :--- | :--- |
| **Admin** | `admin1` | `Admin@123` | `/admin` | Full CRUD operations, Fleet Admin catalog, cache monitor. |
| **Manager** | `manager1` | `Manager@123` | `/dashboard` | View fleet KPI dashboard, review and approve service requests. |
| **Driver** | `driver1` | `Driver@123` | `/vehicles` | View assigned vehicle status, draft maintenance items. |
| **Driver** | `driver2` | `Driver@123` | `/vehicles` | View assigned vehicle status, draft maintenance items. |
| **Driver** | `driver3` | `Driver@123` | `/vehicles` | View assigned vehicle status, draft maintenance items. |

---

## 🚀 Local Quick Start (Docker Compose)

### Prerequisites
*   Docker Desktop (v4+)
*   Docker Compose v2

### 1. Initialize Configuration
Navigate to the infrastructure directory and copy the environment template:
```bash
cd fleetops-infra
cp .env.example .env
```
Edit the `.env` file and set the JWT secret (must be a secure string of at least 32 characters):
```env
JWT_SECRET=mySuperSecureJWTSecretMustBeAtLeast32CharactersLong
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

### 2. Build & Launch Containers
Spin up the entire microservices network in background mode:
```bash
docker-compose up --build -d
```
Docker Compose will automatically resolve service dependencies:
1. Start `fleetops-postgres` and `fleetops-redis` databases.
2. Spin up `auth-service`, `vehicle-service`, and `maintenance-service` once databases report healthy.
3. Boot `request-service` once both vehicle and maintenance services are healthy.
4. Execute `frontend` builder and launch Nginx `fleetops-gateway` to expose the app.

### 3. Access the Application
*   **Web Portal:** [http://localhost:8080](http://localhost:8080)
*   **Health Dashboard:** Navigate to [http://localhost:8080/health/auth](http://localhost:8080/health/auth) to check service health actuator responses (replace `/auth` with `/vehicles`, `/requests`, or `/maintenance` to inspect specific services).

---

## 📖 Deep-Dive Manuals

To continue onto production deployment and advanced features, please refer to our dedicated documentation modules:

1.  **[Core Architecture & System Audit](./FleetOps-Docs/ARCHITECTURE.md)**
    *   Detailed service communication maps and JSON request flows.
    *   Database schemas and relationships.
    *   Complete list of the 33 audit test specs and details of bugs resolved.
2.  **[AWS Production Provisioning Guide](./FleetOps-Docs/DEPLOYMENT_AWS.md)**
    *   Step-by-step setup for AWS RDS PostgreSQL (isolated schemas), ElastiCache Redis, and Secrets Manager.
    *   EC2 Docker Compose single-instance staging guide.
    *   ECS Fargate multi-service production architecture guide.
3.  **[Amazon Bedrock AI Co-Pilot Blueprint](./FleetOps-Docs/AI_INTEGRATION.md)**
    *   Architecture flow for integrating LLM diagnostics.
    *   Spring Boot integration code using the AWS Bedrock runtime SDK.
    *   Prompt templates, IAM permission definitions, and React SPA chat UI mockups.
4.  **[Multi-Phase Project Roadmap](./FleetOps-Docs/ROADMAP.md)**
    *   Structured gantt roadmap mapping subsequent sprint goals.
    *   Step-by-step phases covering staging refinement, ECS production deployments, Bedrock AI releases, and real-time operations.

---
*FleetOps Enterprise — A Modern Vehicle Lifecycle Management Suite.*
