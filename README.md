# 🚛 FleetOps — Microservices Vehicle Maintenance Platform

FleetOps is a production-style, containerized vehicle maintenance and fleet tracking platform built with a **microservices architecture**. It features a React SPA frontend served via NGINX, four independent Spring Boot backend services, a shared PostgreSQL instance with isolated databases per service, and JWT-based stateless authentication — all orchestrated with Docker Compose.

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        User's Browser                          │
└─────────────────────────┬───────────────────────────────────┘
                           │  HTTP :8080
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   fleetops-frontend                            │
│              React SPA  +  NGINX Reverse Proxy                  │
│                                                                 │
│  /api/auth/*   ──────────► auth-service:8080                    │
│  /api/vehicles/* ────────► vehicle-service:8080                 │
│  /api/tasks/*    ────────► maintenance-service:8080             │
│  /api/requests/* ────────► request-service:8080                 │
│  /*            ──────────► index.html  (React SPA)              │
└──────────┬──────────┬──────────┬──────────┬─────────────────┘
           │          │          │          │
           ▼          ▼          ▼          ▼
      ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
      │  auth   │ │ vehicle │ │ maint.  │ │ request │
      │service  │ │ service │ │ service │ │ service │
      │:8080    │ │:8080    │ │:8080    │ │:8080    │
      └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
           │           │  ↕ cache  │            │
           │      ┌────┴────┐      │            │
           │      │  Redis  │      │            │
           │      │  :6379  │      │            │
           │      └─────────┘      │            │
           └───────────┴─────┬─────┴────────────┘
                             ▼
                 ┌───────────────────────┐
                 │   PostgreSQL :5432     │
                 │  auth_db               │
                 │  vehicle_db            │
                 │  maintenance_db        │
                 │  request_db            │
                 └───────────────────────┘
```

All services communicate on an internal **Docker bridge network** (`fleetops-network`). The frontend is the **only service exposed to the host** on port `8080`. NGINX acts as the API gateway, proxying requests to the appropriate backend service.

---

## 📦 Repository Structure

```
Final Project/
├── FleetOps-Docs/               ← Root README (this file)
├── fleetops-frontend/           ← React + Vite SPA
├── fleetops-auth-service/       ← Spring Boot: JWT auth + user management
├── fleetops-vehicle-service/    ← Spring Boot: fleet tracking + KPIs
├── fleetops-maintenance-service/← Spring Boot: pending tasks queue
├── fleetops-request-service/    ← Spring Boot: service request workflows
└── fleetops-infra/              ← Docker Compose + DB init scripts + seeds
```

---

## 🚀 Quick Start

### Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (v4+)
- Docker Compose v2

### 1. Configure environment

```bash
cd fleetops-infra
cp .env.example .env
# Edit .env — at minimum set JWT_SECRET to any long random string
```

**Minimum `.env` content:**
```env
JWT_SECRET=your-super-secret-key-minimum-32-chars
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

### 2. Build and run

```bash
cd fleetops-infra
docker compose up --build -d
```

Docker Compose will:
1. Start PostgreSQL and wait for it to be healthy
2. Start `auth-service`, `vehicle-service`, `maintenance-service` in parallel (each depends on Postgres)
3. Start `request-service` (depends on vehicle + maintenance being healthy)
4. Start `frontend` last (depends on all four services being healthy)

### 3. Database Seeding

The database is seeded **automatically** the first time the PostgreSQL container starts. You do not need to run any manual seed scripts. It creates 21 test vehicles (Sedans, SUVs, Trucks) and 3 test users (`driver1`, `manager1`, `admin1`).

### 4. Open the application

```
http://localhost:8080
```

### 👑 Testing the Accounts

The automatic seed script creates three users for testing role-based access:
- **Admin**: `admin1 / admin123` (Full Fleet CRUD)
- **Manager**: `manager1 / manager123` (KPI Dashboard & Request Approvals)
- **Driver**: `driver1 / driver123` (View assigned vehicles & log maintenance)

---

## 🔑 Service Details

---

### 1. `fleetops-auth-service` — Authentication & Authorization

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3 + Spring Security 6 |
| Port (internal)| 8080                          |
| Database       | `auth_db` (PostgreSQL)        |
| Exposed via    | `/api/auth/*`                 |

#### Responsibilities
- **User registration** — stores BCrypt-hashed passwords
- **User login** — validates credentials, issues HS512 JWT tokens
- **Role management** — `DRIVER`, `MANAGER`, and `ADMIN`

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/auth/register` | Public | Register a new user |
| `POST` | `/auth/login` | Public | Login, returns JWT |
| `GET` | `/auth/me` | JWT | Get current username |

---

### 2. `fleetops-vehicle-service` — Fleet Lifecycle & Alerts

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3.4 + Spring Data JPA |
| Port (internal)| 8080                          |
| Database       | `vehicle_db` (PostgreSQL)     |
| Cache          | Redis 7 (5-min TTL)           |
| Exposed via    | `/api/vehicles/*`             |

#### Responsibilities
- CRUD operations on the fleet catalog
- Computes real-time KPIs (Total active vehicles, breakdowns, expiring insurance, due service)
- Dynamic search by driver assignment and status
- Handles transition alerts

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/vehicles` | Public | List all vehicles (optional `?driverId=` or `?status=`) |
| `GET` | `/vehicles/dashboard` | MANAGER/ADMIN | Fetch fleet KPIs |
| `POST` | `/vehicles` | ADMIN only | Register new vehicle |
| `PATCH` | `/vehicles/{id}/status` | Authenticated | Update vehicle lifecycle state |

---

### 3. `fleetops-maintenance-service` — Pending Task Staging

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3 + Spring Data JPA |
| Port (internal)| 8080                          |
| Database       | `maintenance_db` (PostgreSQL)        |
| Exposed via    | `/api/tasks/*`                 |

#### Responsibilities
- Maintains a **persistent** staging queue of maintenance tasks per authenticated user
- Allows drivers to draft service issues before formal submission
- Queue is cleared upon successful formalization into a Service Request

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/tasks` | JWT | Get current user's queued tasks |
| `POST` | `/tasks/add` | JWT | Add task to queue `{vehicleId, description}` |
| `DELETE` | `/tasks/clear` | JWT | Clear all items from queue |

---

### 4. `fleetops-request-service` — Service Request Workflow

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3 + Spring Data JPA + RestClient |
| Port (internal)| 8080                          |
| Database       | `request_db` (PostgreSQL)       |
| Exposed via    | `/api/requests/*`               |
| Depends on     | `vehicle-service`, `maintenance-service` |

#### Responsibilities
- Service Request state machine: `OPEN` -> `PENDING_APPROVAL` -> `IN_PROGRESS` -> `COMPLETED`
- Synchronizes vehicle states (e.g., setting a vehicle to `IN_SERVICE` when a request is approved)
- Clears pending tasks once formal requests are filed
- Maintains historical maintenance logs per vehicle

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/requests/place` | JWT | Formalize queued tasks into Service Requests |
| `GET` | `/requests` | JWT | Get current user's request history or all requests (Manager/Admin) |
| `PATCH` | `/requests/{id}/status` | MANAGER/ADMIN | Progress request state |

---

### 5. `fleetops-frontend` — React SPA

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | React 18 + Vite                |
| Styling        | Vanilla CSS (custom design system) |
| State          | React Context + useReducer     |
| API Client     | Axios (with interceptors)      |
| Routing        | React Router v6               |
| Server         | NGINX (production build)       |
| Port (host)    | **8080**                       |

#### Frontend Routes

| Route | Access | Who sees it | Description |
|-------|--------|-------------|-------------|
| `/` | Public | Everyone | Home / landing page |
| `/vehicles` | Public | Everyone | Fleet vehicle grid |
| `/login` | Public | Everyone | Login form |
| `/dashboard` | MANAGER/ADMIN | Managers & Admins | Fleet KPIs and Alerts |
| `/requests` | Authenticated | Logged-in users | Service Request history |
| `/admin` | ADMIN only | Admin users | Vehicle CRUD management |

**Role-based UX rules:**
- **Admin**: Auto-redirects to `/admin`. Has full CRUD capabilities.
- **Manager**: Auto-redirects to `/dashboard`. Can approve/manage requests.
- **Driver**: Auto-redirects to `/vehicles`. Can create tasks for their assigned vehicles.

#### Global State (`AppContext`)
```js
{
  isAuthenticated: boolean,
  username: string,
  role: "DRIVER" | "MANAGER" | "ADMIN",
  taskItemsCount: number
}
```

---

## 🗄️ Database Strategy

All four services share **one PostgreSQL 15 container** but use completely **isolated databases**. The init script runs on first container startup:

```bash
# fleetops-infra/database/init-multiple-dbs.sh
CREATE DATABASE auth_db;
CREATE DATABASE vehicle_db;
CREATE DATABASE maintenance_db;
CREATE DATABASE request_db;
```

Each service connects to its own database via `SPRING_DATASOURCE_URL`. Hibernate manages schema creation with `ddl-auto: update`.

---

## 🔒 Security Model

**JWT secret is shared** across all services via the `JWT_SECRET` environment variable. Each service independently validates tokens — no dedicated API gateway or token introspection service is needed. `hasRole()` checks are placed on all mutating backend endpoints.

---

## 🛠️ Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Vite, React Router v6, Axios |
| Styling | Vanilla CSS with CSS variables (dark theme, glassmorphism) |
| Backend | Spring Boot 3.4, Spring Security 6, Spring Data JPA |
| Auth | JWT (HS512 via jjwt library) + BCrypt |
| Cache | Redis 7-alpine |
| Database | PostgreSQL 15 |
| ORM | Hibernate 6 |
| Containerization | Docker, Docker Compose |
| Web Server | NGINX Alpine (reverse proxy + SPA host) |
| Build Tools | Maven (Java), Vite (JS) |

---

*FleetOps — A modern microservices vehicle tracking and maintenance platform.*
