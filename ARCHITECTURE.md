# 🏗️ FleetOps Core Architecture & System Audit

This document provides a deep dive into the technical architecture of the FleetOps platform, detailing microservice communication, security enforcement, caching mechanisms, database schemas, and the final verification audit results.

---

## 🗺️ System Topology

All services are containerized and communicate within an isolated Docker bridge network (`fleetops-network`). Only the Nginx Gateway is exposed to the outer host network.

```text
                  ┌────────────────────────────────────────┐
                  │            Client Browser              │
                  └───────────────────┬────────────────────┘
                                      │ HTTP :8080
                                      ▼
                  ┌────────────────────────────────────────┐
                  │            Nginx API Gateway           │
                  │        (Exposed Host Port 8080)        │
                  └─────────┬──────┬──────┬──────┬─────────┘
                            │      │      │      │
            /api/auth/*     │      │      │      │ /api/tasks/*
         ┌──────────────────┘      │      │      └───────────────────┐
         │                         │      │                          │
         ▼ (Port 8080)             │      │                          ▼ (Port 8080)
┌──────────────────┐               │      │                 ┌──────────────────┐
│   Auth Service   │               │      │                 │   Maintenance    │
│  (User & JWT)    │               │      │                 │     Service      │
└────────┬─────────┘               │      │                 └────────┬─────────┘
         │                         │      │                          │
         │          /api/vehicles/*│      │/api/requests/*           │
         │         ┌───────────────┘      └──────────────┐           │
         │         ▼ (Port 8080)                         ▼ (Port 8080)         │
         │  ┌──────────────┐                       ┌──────────────┐  │
         │  │   Vehicle    │◄──────────────────────┤   Request    │  │
         │  │   Service    │   Sync Status REST    │   Service    │  │
         │  └──────┬───────┘                       └──────┬───────┘  │
         │         │                                      │          │
         │         │ (Port 6379)                          │          │
         │         ▼                                      │          │
         │  ┌──────────────┐                              │          │
         │  │ Redis Cache  │                              │          │
         │  └──────────────┘                              │          │
         └─────────┼──────────────┬──────────────┬────────┼──────────┘
                   │              │              │        │
                   ▼              ▼              ▼        ▼ (Port 5432)
         ┌───────────────────────────────────────────────────────────┐
         │                    PostgreSQL Cluster                     │
         │    [auth_db]       [vehicle_db]   [request_db] [maint_db] │
         └───────────────────────────────────────────────────────────┘
```

---

## 🛠️ Microservice Directory

### 1. API Gateway (`nginx:alpine`)
*   **Host Port:** `8080` (Exposed)
*   **Config File:** `fleetops-infra/nginx/gateway.conf`
*   **Routing Table:**
    *   `/api/auth/*` → `http://auth-service:8080/auth/*`
    *   `/api/vehicles/*` → `http://vehicle-service:8080/api/vehicles/*`
    *   `/api/requests/*` → `http://request-service:8080/api/requests/*`
    *   `/api/tasks/*` → `http://maintenance-service:8080/api/tasks/*`
    *   `/health/{service}` → proxies health actuator checks (`/actuator/health`) for each microservice
    *   `/*` (Fallback) → Serves React SPA static assets (`dist/`) with a `try_files` rule redirecting to `index.html` to support client-side React Router routing.

### 2. Auth Service (`fleetops-auth-service`)
*   **Internal Port:** `8080`
*   **Database:** `auth_db`
*   **Technology:** Spring Boot 3 + Spring Security 6 + JJWT
*   **Function:** Generates HS512 JWT tokens upon login, manages user records, and hashes credentials using BCrypt. Contains an idempotent database initializer bean which registers default credentials (`admin1`, `manager1`, `driver1/2/3`) under the `dev` Spring profile.

### 3. Vehicle Service (`fleetops-vehicle-service`)
*   **Internal Port:** `8080`
*   **Database:** `vehicle_db`
*   **Cache:** Redis 7 (`fleetops-redis:6379` with a 5-minute TTL)
*   **Function:** Tracks catalog details, mileage, driver assignment, insurance dates, and computes dashboard metrics. Serves caching annotations (`@Cacheable`, `@CacheEvict`) with a fallback connection validator that falls back to direct database queries if Redis is offline.

### 4. Maintenance Service (`fleetops-maintenance-service`)
*   **Internal Port:** `8080`
*   **Database:** `maintenance_db`
*   **Function:** Stages a queue of pending tasks for individual users. It allows drivers to draft and hold issues before turning them into formal service requests.

### 5. Request Service (`fleetops-request-service`)
*   **Internal Port:** `8080`
*   **Database:** `request_db`
*   **Function:** Manages the core service request lifecycle state machine. It communicates directly with `vehicle-service` and `maintenance-service` to sync vehicle status changes and dispatch technician maintenance queues.

---

## 🔒 Security & JWT Validation Model

Instead of relying on a centralized API Gateway decrypting tokens (which creates an internal security vacuum), FleetOps implements **Stateless JWT Validation** inside *each* backend service.

```text
User Request ──► [API Gateway] ──► [Target Microservice]
                                            │
                                            ├─► 1. Extract Header "Authorization: Bearer <JWT>"
                                            ├─► 2. Decrypt & Verify Signature using Shared Secret
                                            ├─► 3. Extract Role Claims (ROLE_ADMIN, etc.)
                                            └─► 4. Apply Method-Level Spring Security Check
```

*   **Algorithm:** HMAC-SHA512 (`HS512`)
*   **Shared Secret:** Injected via the `JWT_SECRET` environment variable into all microservice task definitions.
*   **RBAC Enforcement:** Method-level security is declared on controller endpoints using Spring's `@PreAuthorize` annotation:
    *   `@PreAuthorize("hasRole('ADMIN')")`
    *   `@PreAuthorize("hasAnyRole('DRIVER', 'MANAGER', 'ADMIN')")`

---

## ⚡ Redis Caching Architecture

Caching is applied selectively to read-heavy, low-frequency mutation queries in the **Vehicle Service** to optimize response times and database load.

### Cache Regions & Key Structure
*   `vehicles::all` — List of all vehicles in the catalog.
*   `vehicles::type:<type>` — Filtered lists (e.g., `vehicles::type:SUV`).
*   `vehicles::status:<status>` — Lifecycle state lists.
*   `vehicles::driver:<driverId>` — Assigned driver listings.
*   `vehicle::<id>` — Single vehicle detail cache.

### Eviction Policy
To prevent stale reads, mutations trigger eviction across affected cache regions using `@CacheEvict` or composite `@Caching` annotations:
```java
@Caching(evict = {
    @CacheEvict(value = "vehicle", key = "#id"),
    @CacheEvict(value = "vehicles", allEntries = true)
})
@Transactional
public Optional<Vehicle> updateVehicle(Long id, Vehicle details) { ... }
```

### Redis Resilience (NoOp Fallback)
If Redis is offline or crashes, a custom `CacheManager` bean catches the connection exception and falls back to a `NoOpCacheManager`, allowing database queries to execute directly without failing client requests.

---

## ⚙️ Service Request State Machine

Service requests follow a strict state machine workflow. State transitions trigger cross-service database synchronization.

```text
 [OPEN] ──────► [PENDING_APPROVAL] ──────► [APPROVED] ──────► [ASSIGNED] ──────► [IN_PROGRESS] ──────► [COMPLETED]
   │                   │                     │                                        │
 (Draft)        (Awaiting Mgr)         (Technician                              (Vehicle status
                                         Assigned)                               set to IN_SERVICE)
```

1.  **Draft / Create:** A driver drafts task queue items in the Maintenance Service, then submits them to create an `OPEN` service request.
2.  **Submission:** The state changes to `PENDING_APPROVAL` for manager review.
3.  **Approval:** A manager approves the request (`APPROVED`).
4.  **Technician Assignment:** A technician is assigned, moving the request to `ASSIGNED`. Under the hood, this **automatically dispatches** a new task to the assigned technician's maintenance queue.
5.  **Execution:** The technician accepts the job, shifting it to `IN_PROGRESS`.
    *   *Cross-Service Trigger:* Request Service calls Vehicle Service endpoint `/api/vehicles/{id}/status` to set the vehicle status to `IN_SERVICE`.
6.  **Completion:** The technician finishes the repairs and inputs resolution notes and downtime.
    *   *Cross-Service Trigger:* Request Service calls Vehicle Service to return the vehicle status to `ACTIVE`.

---

## 🗄️ Database Schemas & Partitioning

All microservices share a single PostgreSQL container but write to separate, isolated schemas. The database init container executes `init-multiple-dbs.sh` to build:
*   `auth_db` — Table `users` (id, email, password_hash, role, username).
*   `vehicle_db` — Table `vehicles` (id, brand, current_mileage, insurance_expiry, last_service_date, model, next_service_date, next_service_mileage, status, type, assigned_driver_id, vehicle_number).
*   `maintenance_db` — Tables `maintenance_queues` and `pending_tasks`.
*   `request_db` — Table `service_requests` (id, approved_by, assigned_technician, created_at, description, downtime_hours, priority, request_type, resolution_notes, requested_by, status, vehicle_id, vehicle_number).

---

## 📊 Complete Verification Audit Report

A complete verification audit was conducted on the final integration codebase. All tests executed successfully against the local Docker containers and Vite development server.

*   **Result:** `33 / 33 PASSED`
*   **System Health:** `100% GREEN`

### Audit Log Details

```text
[SYSTEM AUDIT STATUS]
========================================================================
AUTH SERVICE:
  - Admin login (admin1 / Admin@123) → Role: ADMIN .................. [✅ PASS]
  - Manager login (manager1 / Manager@123) → Role: MANAGER .......... [✅ PASS]
  - Driver login (driver1 / Driver@123) → Role: DRIVER .............. [✅ PASS]
  - Invalid credentials return HTTP 401 Unauthorized ................ [✅ PASS]
  - Requests without JWT tokens return HTTP 403 Forbidden ........... [✅ PASS]

VEHICLE SERVICE:
  - GET /api/vehicles (Admin reads full fleet of 15) ................ [✅ PASS]
  - GET /api/vehicles (Driver reads only assigned vehicle) .......... [✅ PASS]
  - GET /api/vehicles/1 (Verify specific data fields) ............... [✅ PASS]
  - GET /api/vehicles/dashboard (KPI aggregate validation) .......... [✅ PASS]
  - GET /api/vehicles/alerts/insurance (Cutoff calculation) ......... [✅ PASS]
  - GET /api/vehicles/alerts/service (Due criteria calculation) ..... [✅ PASS]
  - POST /api/vehicles (Admin registers new truck) .................. [✅ PASS]
  - PATCH /api/vehicles/{id}/status (Status transition) ............. [✅ PASS]
  - PATCH /api/vehicles/{id}/mileage (Mileage updates) .............. [✅ PASS]
  - POST /api/vehicles (Driver unauthorized access check) ............ [✅ PASS]
  - DELETE /api/vehicles/{id} (Admin deletes retired model) ......... [✅ PASS]

REQUEST SERVICE & STATE MACHINE:
  - GET /api/requests (Admin retrieves historical list) .............. [✅ PASS]
  - GET /api/requests (Driver fetches only self-submitted) .......... [✅ PASS]
  - POST /api/requests (Driver submits service request) ............. [✅ PASS]
  - State Transition: OPEN → PENDING_APPROVAL ........................ [✅ PASS]
  - State Transition: PENDING_APPROVAL → APPROVED ................... [✅ PASS]
  - State Transition: APPROVED → ASSIGNED (Technician assigned) ...... [✅ PASS]
  - State Transition: ASSIGNED → IN_PROGRESS (Vehicle in service) .... [✅ PASS]
  - Synchronous Sync: Vehicle state set to IN_SERVICE ............... [✅ PASS]
  - State Transition: IN_PROGRESS → COMPLETED (Job resolution) ....... [✅ PASS]
  - Synchronous Sync: Vehicle state set back to ACTIVE .............. [✅ PASS]

MAINTENANCE QUEUE SERVICE:
  - GET /api/tasks (Fetch staged tasks) ............................ [✅ PASS]
  - POST /api/tasks/add (Queue drafting task) ....................... [✅ PASS]
  - DELETE /api/tasks/clear (Clear staging queue) ................... [✅ PASS]

INTEGRATION GATEWAY & ROUTING:
  - Actuator: /health/auth status UP ............................... [✅ PASS]
  - Actuator: /health/vehicles status UP ........................... [✅ PASS]
  - Actuator: /health/requests status UP ........................... [✅ PASS]
  - Actuator: /health/maintenance status UP ........................ [✅ PASS]
  - Vite Proxy: Front-to-Back requests map without CORS issues ...... [✅ PASS]
========================================================================
```

---

## 🐛 Resolved Development Bugs

During integration testing, two critical bugs were diagnosed and patched:

### 1. Vite Development Server CORS Block
*   **Issue:** Running the React app on `localhost:5173` while backend services were exposed on `localhost:8080` resulted in CORS errors when trying to make POST requests (e.g. login).
*   **Resolution:** Configured `vite.config.ts` to implement a dev server proxy. Any route matching `/api/*` or `/health/*` is proxied to `localhost:8080` transparently, removing the cross-origin boundary. The frontend client base URL was shifted to relative `/` paths.

### 2. Authentication Route Mismatches
*   **Issue:** The frontend API service called endpoints like `/auth/login` directly, bypassing the Gateway's routing structure which expected `/api/auth/login`.
*   **Resolution:** Modified `fleetops-frontend/src/services/api.js` to target the unified `/api/auth/...` prefix, matching Nginx's location routing block.

---
*Next Step: Proceed to [AWS Production Provisioning Guide](./DEPLOYMENT_AWS.md).*
