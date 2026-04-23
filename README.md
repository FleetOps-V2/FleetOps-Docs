# 🛒 CloudCart — Microservices E-Commerce Platform

CloudCart is a production-style, containerized e-commerce platform built with a **microservices architecture**. It features a React SPA frontend served via NGINX, four independent Spring Boot backend services, a shared PostgreSQL instance with isolated databases per service, and JWT-based stateless authentication — all orchestrated with Docker Compose.

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        User's Browser                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │  HTTP :8080
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   cloudcart-frontend                            │
│              React SPA  +  NGINX Reverse Proxy                  │
│                                                                 │
│  /api/auth/*   ──────────► auth-service:8080                    │
│  /api/products/* ────────► product-service:8080                 │
│  /api/cart/*   ──────────► cart-service:8080                    │
│  /api/orders/* ──────────► order-service:8080                   │
│  /*            ──────────► index.html  (React SPA)              │
└──────────┬──────────┬──────────┬──────────┬────────────────────┘
           │          │          │          │
           ▼          ▼          ▼          ▼
      ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
      │  auth   │ │product  │ │  cart   │ │  order  │
      │service  │ │service  │ │service  │ │service  │
      │:8080    │ │:8080    │ │:8080    │ │:8080    │
      └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
           │           │           │            │
           └───────────┴─────┬─────┴────────────┘
                             ▼
                 ┌────────────────────────┐
                 │   PostgreSQL :5432     │
                 │  auth_db               │
                 │  product_db            │
                 │  cart_db               │
                 │  order_db              │
                 └────────────────────────┘
```

All services communicate on an internal **Docker bridge network** (`cloudcart-network`). The frontend is the **only service exposed to the host** on port `8080`. NGINX acts as the API gateway, proxying requests to the appropriate backend service.

---

## 📦 Repository Structure

```
Final Project/
├── Cloud-Cart/               ← Root README (this file)
├── cloudcart-frontend/       ← React + Vite SPA
├── cloudcart-auth-service/   ← Spring Boot: JWT auth + user management
├── cloudcart-product-service/← Spring Boot: product catalog + stock
├── cloudcart-cart-service/   ← Spring Boot: shopping cart (per user)
├── cloudcart-order-service/  ← Spring Boot: order placement + history
└── cloudcart-infra/          ← Docker Compose + DB init scripts + seeds
```

---

## 🚀 Quick Start

### Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (v4+)
- Docker Compose v2

### 1. Clone all repositories

```bash
git clone https://github.com/Cloud-Cart-Project/cloudcart-infra
# Also clone each service repo into the same parent directory
```

### 2. Configure environment

```bash
cd cloudcart-infra
cp .env.example .env
# Edit .env — at minimum set JWT_SECRET to any long random string
```

**Minimum `.env` content:**
```env
JWT_SECRET=your-super-secret-key-minimum-32-chars
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

### 3. Build and run

```bash
cd cloudcart-infra
docker compose up --build -d
```

Docker Compose will:
1. Start PostgreSQL and wait for it to be healthy
2. Start `auth-service`, `product-service`, `cart-service` in parallel (each depends on Postgres)
3. Start `order-service` (depends on product + cart being healthy)
4. Start `frontend` last (depends on all four services being healthy)

### 4. Seed the database (first run only)

```bash
# From cloudcart-infra directory:
Get-Content database/seed.sql | docker exec -i cloudcart-postgres psql -U postgres -d product_db
```

### 5. Open the application

```
http://localhost:8080
```

### Default Admin Account

The seed script does **not** create an admin user automatically. Create one via the register API, then promote via SQL:

```bash
# 1. Register a new user
Invoke-RestMethod -Uri "http://localhost:8080/api/auth/register" `
  -Method POST -ContentType "application/json" `
  -Body '{"username":"cloudadmin","email":"admin@example.com","password":"Admin@123"}'

# 2. Promote to ADMIN role
docker exec cloudcart-postgres psql -U postgres -d auth_db `
  -c "UPDATE users SET role='ADMIN' WHERE username='cloudadmin';"
```

---

## 🔑 Service Details

---

### 1. `cloudcart-auth-service` — Authentication & Authorization

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3 + Spring Security 6 |
| Port (internal)| 8080                          |
| Database       | `auth_db` (PostgreSQL)        |
| Exposed via    | `/api/auth/*`                 |

#### Responsibilities
- **User registration** — stores BCrypt-hashed passwords
- **User login** — validates credentials, issues HS512 JWT tokens
- **Role management** — `CUSTOMER` (default) and `ADMIN`

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/auth/register` | Public | Register a new user |
| `POST` | `/auth/login` | Public | Login, returns JWT |
| `GET` | `/auth/me` | JWT | Get current username |

#### JWT Token Structure
```json
{
  "sub": "cloudadmin",
  "role": "ROLE_ADMIN",
  "iat": 1714000000,
  "exp": 1714086400
}
```

- Token expiry: **24 hours** (86400000 ms, configurable)
- Algorithm: **HS512**
- All other services validate the **same JWT secret** — no central auth gateway needed

#### Security Config
- `/auth/register` and `/auth/login` are fully **public**
- All other routes require a valid JWT bearer token
- Passwords stored with **BCrypt** (strength 10)

---

### 2. `cloudcart-product-service` — Product Catalog & Stock

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3 + Spring Data JPA |
| Port (internal)| 8080                          |
| Database       | `product_db` (PostgreSQL)     |
| Exposed via    | `/api/products/*`             |

#### Responsibilities
- CRUD operations on the product catalog
- Atomic, race-condition-safe stock management
- Category-based product filtering
- Product image URL storage

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/products` | Public | List all products (optional `?category=`) |
| `GET` | `/products/{id}` | Public | Get single product |
| `POST` | `/products` | ADMIN only | Create new product |
| `PUT` | `/products/{id}` | ADMIN only | Update product |
| `DELETE` | `/products/{id}` | ADMIN only | Delete product |
| `PATCH` | `/products/{id}/stock` | Authenticated | Adjust stock by `quantity` delta |

#### Atomic Stock Update
The stock `PATCH` endpoint uses a custom JPA `@Modifying` query:
```sql
UPDATE products SET stock = stock + :quantity
WHERE id = :id AND stock + :quantity >= 0
```
This ensures:
- No stock goes negative (atomic guard)
- No race conditions between concurrent orders
- Returns `409 Conflict` if insufficient stock

#### Product Schema
```
products
  id          BIGINT PK
  name        VARCHAR
  description VARCHAR
  category    VARCHAR
  price       DECIMAL(10,2)
  stock       INTEGER
  image_url   VARCHAR
  created_at  TIMESTAMP
  updated_at  TIMESTAMP
```

---

### 3. `cloudcart-cart-service` — Shopping Cart

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3 + Spring Data JPA |
| Port (internal)| 8080                          |
| Database       | `cart_db` (PostgreSQL)        |
| Exposed via    | `/api/cart/*`                 |

#### Responsibilities
- Maintains a **persistent** shopping cart per authenticated user
- Supports adding, updating, and removing items
- Cart is cleared upon successful cart-mode checkout

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/cart` | JWT | Get current user's cart with all items |
| `POST` | `/cart/add` | JWT | Add item to cart `{productId, quantity}` |
| `DELETE` | `/cart/clear` | JWT | Clear all items from cart |

#### Cart Schema
```
carts
  id          BIGINT PK
  username    VARCHAR (unique per user)

cart_items
  id          BIGINT PK
  cart_id     FK → carts.id
  product_id  BIGINT
  quantity    INTEGER
```

> **Note:** The cart service stores only `productId` and `quantity`. Product details (name, price, image) are fetched on-demand from the product service by the frontend and order service.

---

### 4. `cloudcart-order-service` — Order Placement & History

| Property       | Value                          |
|----------------|-------------------------------|
| Framework      | Spring Boot 3 + Spring Data JPA + RestClient |
| Port (internal)| 8080                          |
| Database       | `order_db` (PostgreSQL)       |
| Exposed via    | `/api/orders/*`               |
| Depends on     | `product-service`, `cart-service` |

#### Responsibilities
- Two checkout modes: **Buy Now** (single product) and **Cart** (multi-item)
- Stock validation and deduction before order confirmation
- Preserves price at time of purchase (`priceAtTime`)
- Clears cart only on cart-mode checkout
- Per-user order history

#### Key Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/orders/place` | JWT | Place an order |
| `GET` | `/orders` | JWT | Get current user's order history |
| `GET` | `/orders/{id}` | JWT | Get a specific order (owner only) |

#### Order Request Payload

**Buy Now:**
```json
{
  "type": "BUY_NOW",
  "items": [
    { "productId": 2, "quantity": 1 }
  ]
}
```

**Cart Checkout:**
```json
{
  "type": "CART"
}
```
*(Cart items are fetched from the cart-service automatically)*

#### Order Placement Flow (Backend)
```
1. Receive JWT → extract username
2. If CART mode → fetch items from cart-service
3. For each item:
   a. Fetch product details from product-service
   b. Verify stock >= quantity → 409 if not
   c. PATCH /products/{id}/stock  (decrement stock atomically)
   d. Record OrderItem with priceAtTime
4. Save Order to order_db
5. If CART mode → DELETE /cart/clear
6. Return saved Order object
```

#### Order Schema
```
orders
  id            BIGINT PK
  username      VARCHAR
  total_amount  DECIMAL(10,2)
  created_at    TIMESTAMP

order_items
  id            BIGINT PK
  order_id      FK → orders.id
  product_id    BIGINT
  quantity      INTEGER
  price_at_time DECIMAL(10,2)   ← snapshot of price at purchase time
```

---

### 5. `cloudcart-frontend` — React SPA

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

| Route | Access | Description |
|-------|--------|-------------|
| `/` | Public | Home / landing page |
| `/products` | Public | Product catalog grid |
| `/login` | Public | Login + Register form |
| `/checkout` | Auth required | Checkout (Buy Now or Cart mode) |
| `/orders` | Auth required | Order history + Reorder |
| `/admin` | ADMIN role only | Product management dashboard |

#### Global State (`AppContext`)
```js
{
  isAuthenticated: boolean,
  username: string,
  role: "CUSTOMER" | "ADMIN",
  cartItemsCount: number
}
```

#### Axios Interceptors
- **Request:** Automatically attaches `Authorization: Bearer <token>` to every API request
- **Response:** On `401 Unauthorized`, fires `auth-expired` DOM event → context logs the user out automatically

#### NGINX API Routing (inside container)
```nginx
/api/auth/*     → http://auth-service:8080/auth/*
/api/products/* → http://product-service:8080/products/*
/api/cart/*     → http://cart-service:8080/cart/*
/api/orders/*   → http://order-service:8080/orders/*
/*              → index.html  (React SPA fallback)
```

#### Build Pipeline
```dockerfile
# Stage 1 — Build
FROM node:20-alpine AS build
RUN npm install && npm run build   # outputs /app/dist

# Stage 2 — Serve
FROM nginx:alpine
COPY --from=build /app/dist  /usr/share/nginx/html
COPY nginx.conf               /etc/nginx/conf.d/default.conf
```

---

## 🗄️ Database Strategy

All four services share **one PostgreSQL 15 container** but use completely **isolated databases**. The init script runs on first container startup:

```bash
# cloudcart-infra/database/init-multiple-dbs.sh
CREATE DATABASE auth_db;
CREATE DATABASE product_db;
CREATE DATABASE cart_db;
CREATE DATABASE order_db;
```

Each service connects to its own database via `SPRING_DATASOURCE_URL`. Hibernate manages schema creation with `ddl-auto: update` — tables are created/migrated automatically.

---

## 🔒 Security Model

```
Browser
  │
  ├─ POST /api/auth/login ─────────────────────► auth-service
  │                                               └─ validates password (BCrypt)
  │                                               └─ returns JWT (HS512, 24h)
  │
  ├─ GET /api/products (public) ───────────────► product-service (no auth check)
  │
  ├─ POST /api/cart/add ──────────────────────► cart-service
  │   Authorization: Bearer <JWT>               └─ JwtAuthFilter validates token
  │                                             └─ extracts username from claims
  │
  └─ POST /api/orders/place ──────────────────► order-service
      Authorization: Bearer <JWT>               └─ validates same JWT secret
                                                └─ calls product-service (no auth)
                                                └─ calls cart-service (forwards JWT)
```

**JWT secret is shared** across all services via the `JWT_SECRET` environment variable. Each service independently validates tokens — no dedicated API gateway or token introspection service is needed.

---

## 🛍️ End-to-End Order Flows

### Flow A — Cart Checkout

```
User → Add to Cart → Cart Drawer → Proceed to Checkout
  → /checkout page loads (fetches cart items + enriches with product details)
  → User clicks "Place Order"
  → POST /api/orders/place { type: "CART" }
  → order-service fetches cart, validates stock, deducts stock, saves order, clears cart
  → Success screen with Order ID
```

### Flow B — Buy Now

```
User → Click "Buy Now" on product card
  → navigate('/checkout', { state: { product } })
  → /checkout page shows single product
  → User clicks "Place Order"
  → POST /api/orders/place { type: "BUY_NOW", items: [{productId, quantity: 1}] }
  → order-service validates stock, deducts, saves order (cart NOT cleared)
  → Success screen with Order ID
```

### Flow C — Reorder

```
User → /orders page → click "Reorder" on past order item
  → navigate('/checkout', { state: { product: { id, price: priceAtTime, name } } })
  → Standard Buy Now checkout flow with that product
```

---

## 🛠️ Admin Capabilities

Admin users (role = `ADMIN`) have access to:

| Feature | Location | Behavior |
|---------|----------|---------|
| View all products | `/admin` | Table with stock, price, last updated |
| Stock adjustment | `/admin` → +10 / +50 buttons | Optimistic UI update + PATCH API call |
| Add new product | `/admin` → "+ Add Product" | Modal form |
| Edit product | `/admin` → Edit button | Inline edit modal |
| Delete product | `/admin` → Del button | Soft confirmation then DELETE API |

> Admin access is guarded both on the **frontend** (ProtectedRoute with `requireAdmin`) and the **backend** (Spring Security `hasRole("ADMIN")` on mutating product endpoints).

---

## ⚙️ Environment Variables

| Variable | Service | Default | Description |
|----------|---------|---------|-------------|
| `JWT_SECRET` | all | *(required)* | Shared HS512 signing secret |
| `JWT_EXPIRATION` | auth | `86400000` | Token TTL in milliseconds |
| `POSTGRES_USER` | infra/all | `postgres` | Database superuser |
| `POSTGRES_PASSWORD` | infra/all | `postgres` | Database password |
| `DB_HOST` | all | `postgres` | Postgres container hostname |
| `DB_PORT` | all | `5432` | Postgres port |
| `PRODUCT_SERVICE_URL` | order | `http://product-service:8080` | Internal product URL |
| `CART_SERVICE_URL` | order | `http://cart-service:8080` | Internal cart URL |
| `CORS_ALLOWED_ORIGINS` | all | `*` | CORS policy |

---

## 🐳 Docker Compose Startup Order

```
postgres  ──healthy──►  auth-service   ──────────────────────────────────────────►┐
                   ──►  product-service ──healthy──►  order-service ──healthy──►  frontend
                   ──►  cart-service   ──healthy──►┘
```

Each service uses `depends_on` with `condition: service_healthy`. Spring Boot services expose `/actuator/health` which is polled by Docker's healthcheck.

---

## 📋 Useful Commands

```bash
# Start everything
docker compose up -d

# Rebuild a single service (after code change)
docker compose build --no-cache auth-service
docker compose up --force-recreate -d auth-service

# View logs
docker compose logs -f order-service

# Check all container health
docker compose ps

# Seed products (first run)
Get-Content database/seed.sql | docker exec -i cloudcart-postgres psql -U postgres -d product_db

# Access the database directly
docker exec -it cloudcart-postgres psql -U postgres -d auth_db

# Stop everything
docker compose down

# Wipe volumes (full reset)
docker compose down -v
```

---

## 🧪 API Testing Examples

```powershell
# Register
Invoke-RestMethod -Uri "http://localhost:8080/api/auth/register" `
  -Method POST -ContentType "application/json" `
  -Body '{"username":"testuser","email":"test@example.com","password":"Test@123"}'

# Login (save token)
$res = Invoke-RestMethod -Uri "http://localhost:8080/api/auth/login" `
  -Method POST -ContentType "application/json" `
  -Body '{"username":"testuser","password":"Test@123"}'
$token = $res.token

# Get products
Invoke-RestMethod -Uri "http://localhost:8080/api/products"

# Add to cart
Invoke-RestMethod -Uri "http://localhost:8080/api/cart/add" `
  -Method POST -ContentType "application/json" `
  -Headers @{Authorization="Bearer $token"} `
  -Body '{"productId":1,"quantity":2}'

# Place a Buy Now order
Invoke-RestMethod -Uri "http://localhost:8080/api/orders/place" `
  -Method POST -ContentType "application/json" `
  -Headers @{Authorization="Bearer $token"} `
  -Body '{"type":"BUY_NOW","items":[{"productId":1,"quantity":1}]}'

# Get order history
Invoke-RestMethod -Uri "http://localhost:8080/api/orders" `
  -Headers @{Authorization="Bearer $token"}
```

---

## 📁 Key Source Files

| File | Purpose |
|------|---------|
| `cloudcart-infra/docker-compose.yml` | Entire container orchestration |
| `cloudcart-infra/database/init-multiple-dbs.sh` | Creates all 4 databases on first boot |
| `cloudcart-infra/database/seed.sql` | Seeds 21 products across 6 categories |
| `cloudcart-frontend/nginx.conf` | NGINX API gateway routing rules |
| `cloudcart-frontend/src/App.jsx` | All frontend routes + route guards |
| `cloudcart-frontend/src/context/AppContext.jsx` | Global auth + cart state |
| `cloudcart-frontend/src/services/api.js` | Axios instance + all API functions |
| `cloudcart-auth-service/.../AuthController.java` | Login + Register endpoints |
| `cloudcart-auth-service/.../AuthConfig.java` | BCrypt + DaoAuthProvider setup |
| `cloudcart-product-service/.../SecurityConfig.java` | Public GET, ADMIN write guard |
| `cloudcart-order-service/.../OrderController.java` | Dual-mode order logic |

---

## 🏗️ Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Vite, React Router v6, Axios |
| Styling | Vanilla CSS with CSS variables (dark theme, glassmorphism) |
| Backend | Spring Boot 3.4, Spring Security 6, Spring Data JPA |
| Auth | JWT (HS512 via jjwt library) + BCrypt |
| Database | PostgreSQL 15 |
| ORM | Hibernate 6 |
| Containerization | Docker, Docker Compose |
| Web Server | NGINX Alpine (reverse proxy + SPA host) |
| Build Tools | Maven (Java), Vite (JS) |

---

*CloudCart — built as a full-stack microservices training project demonstrating containerized service decomposition, JWT security, atomic data operations, and modern React SPA patterns.*
