# Phase 1 Technical Design

## 1. Goal

Phase 1 delivers a working, authenticated API surface for the four foundational services: `auth-service`, `gateway-service`, `product-service`, and `order-service` (basic CRUD only — no saga). By the end of Phase 1, a user can register, log in, browse products, and place an order that is persisted in `PENDING` status. All requests flow through the gateway with JWT validation enforced.

This phase proves the core infrastructure patterns — JWT auth, circuit-breaker-wrapped inter-service REST calls, the outbox pattern, Flyway migrations, and Testcontainers-based integration tests — before the full saga is wired in Phase 2.

---

## 2. Scope

| Service | What is built in Phase 1 |
|---|---|
| `auth-service` | Register, login, logout, refresh token, `/me` endpoint, `user.registered.v1` outbox event |
| `gateway-service` | JWT validation, routing to all four Phase 1 services, rate limiting, `X-Request-ID` injection |
| `product-service` | Create, update, deactivate, list, and get products; `product.created.v1` outbox event |
| `order-service` | Create order (persisted as `PENDING`), list orders, get order, get order timeline, cancel order |

---

## 3. Out of Scope for Phase 1

- `inventory-service`, `payment-service`, `shipment-service`, `invoice-service`, `notification-service`
- SNS/SQS event publishing and consumption (outbox rows are written but not polled/published)
- The fulfillment saga and all compensation flows
- The outbox poller scheduler
- AWS deployment, Terraform, CloudWatch alarms
- Admin retry endpoint for failed orders
- Stock quantity display in product listings (requires inventory-service — returns `null` with `meta.stockDataAvailable: false`)

---

## 4. Service Responsibilities

### 4.1 auth-service

Owns user identity and token lifecycle.

- Stores users (`auth_users`) and hashed passwords (bcrypt, cost factor 12).
- Issues RS256 JWT access tokens (15-minute expiry) and opaque refresh tokens (7-day expiry, stored as SHA-256 hash in `auth_refresh_tokens`).
- Exposes the RS256 public key via `GET /api/v1/auth/public-key` for gateway startup.
- Rate-limits login attempts: 5 per 15 minutes per IP using Resilience4j `RateLimiter`.
- Writes `user.registered.v1` to `auth_outbox` within the same registration transaction (outbox poller is Phase 2).
- JWT claims: `sub` (userId), `email`, `role`, `iat`, `exp`.

### 4.2 gateway-service

Stateless ingress proxy. Owns no business data.

- Validates JWT on every request except `/api/v1/auth/register`, `/api/v1/auth/login`, and `/api/v1/auth/refresh`.
- RS256 public key is loaded from the `JWT_PUBLIC_KEY` environment variable (PEM-encoded) at startup — no runtime call to auth-service.
- Injects `X-Request-ID` header (generates UUID if absent) on every forwarded request.
- Enforces rate limits: 100 req/s per authenticated user, 10 req/s per IP for unauthenticated endpoints (Spring Cloud Gateway `RequestRateLimiter` filter with in-memory token bucket).
- Routes by path prefix (see Section 6.1).
- Propagates `Authorization` header downstream unchanged.

### 4.3 product-service

Owns the product catalog.

- Admins create, update, and deactivate products.
- All authenticated users can list and get products.
- Writes `product.created.v1` to `prd_outbox` on product creation (outbox poller is Phase 2).
- In Phase 1, `availableStock` is always `null` and `meta.stockDataAvailable` is always `false` in list/get responses (inventory-service not yet available). The circuit-breaker fallback is wired but the downstream call is a no-op stub returning empty.
- Price changes apply to future orders only — no event required in Phase 1.
- Deactivated products are excluded from customer-facing listings.

### 4.4 order-service

Owns the order lifecycle. In Phase 1, orders are created and persisted in `PENDING` status only — no saga events are consumed.

- Resolves product name and unit price from `product-service` via a synchronous REST call (circuit-breaker wrapped; fails with `503` if product-service is unavailable after retries).
- Writes `order.placed.v1` to `ord_outbox` within the same create-order transaction (outbox poller is Phase 2).
- Supports idempotent order creation via `Idempotency-Key` header.
- Exposes list, get, timeline, and cancel endpoints.
- Cancel is limited to `PENDING` status in Phase 1 (no saga states exist yet).
- Order timeline records `ORDER_PLACED` and `ORDER_CANCELLED` events in Phase 1.


---

## 5. Database Tables

All tables use `TIMESTAMPTZ` for timestamps, `UUID` primary keys, and a `version` column for optimistic locking unless noted. Flyway manages all schema changes.

### 5.1 auth-service (`auth_` prefix)

**`auth_users`**

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `email` | VARCHAR(255) | UNIQUE NOT NULL |
| `password_hash` | VARCHAR(255) | NOT NULL |
| `first_name` | VARCHAR(100) | NOT NULL |
| `last_name` | VARCHAR(100) | NOT NULL |
| `role` | VARCHAR(20) | NOT NULL — `CUSTOMER`, `ADMIN` |
| `active` | BOOLEAN | NOT NULL DEFAULT true |
| `version` | BIGINT | NOT NULL DEFAULT 0 |
| `created_at` | TIMESTAMPTZ | NOT NULL |
| `updated_at` | TIMESTAMPTZ | NOT NULL |

**`auth_refresh_tokens`**

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `user_id` | UUID | NOT NULL — logical FK to `auth_users` |
| `token_hash` | VARCHAR(255) | UNIQUE NOT NULL — SHA-256 of the raw token |
| `expires_at` | TIMESTAMPTZ | NOT NULL |
| `revoked` | BOOLEAN | NOT NULL DEFAULT false |
| `created_at` | TIMESTAMPTZ | NOT NULL |

**`auth_outbox`**

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `event_type` | VARCHAR(100) | NOT NULL |
| `aggregate_id` | UUID | NOT NULL |
| `payload` | JSONB | NOT NULL |
| `correlation_id` | UUID | NOT NULL |
| `status` | VARCHAR(20) | NOT NULL — `PENDING`, `PUBLISHED` |
| `created_at` | TIMESTAMPTZ | NOT NULL |
| `published_at` | TIMESTAMPTZ | nullable |

Flyway scripts: `V1__create_auth_users.sql`, `V2__create_auth_refresh_tokens.sql`, `V3__create_auth_outbox.sql`

---

### 5.2 product-service (`prd_` prefix)

**`prd_products`**

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `name` | VARCHAR(255) | NOT NULL |
| `description` | TEXT | nullable |
| `category` | VARCHAR(100) | NOT NULL |
| `price` | NUMERIC(10,2) | NOT NULL — must be > 0 |
| `status` | VARCHAR(20) | NOT NULL — `ACTIVE`, `INACTIVE` |
| `version` | BIGINT | NOT NULL DEFAULT 0 |
| `created_at` | TIMESTAMPTZ | NOT NULL |
| `updated_at` | TIMESTAMPTZ | NOT NULL |

**`prd_outbox`** — same structure as `auth_outbox`.

Flyway scripts: `V1__create_prd_products.sql`, `V2__create_prd_outbox.sql`

---

### 5.3 order-service (`ord_` prefix)

**`ord_orders`**

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `customer_id` | UUID | NOT NULL — logical FK to auth-service |
| `status` | VARCHAR(30) | NOT NULL — `PENDING`, `CANCELLED` in Phase 1 |
| `total_amount` | NUMERIC(10,2) | NOT NULL |
| `retry_count` | INT | NOT NULL DEFAULT 0 |
| `idempotency_key` | UUID | UNIQUE nullable |
| `version` | BIGINT | NOT NULL DEFAULT 0 |
| `created_at` | TIMESTAMPTZ | NOT NULL |
| `updated_at` | TIMESTAMPTZ | NOT NULL |

**`ord_order_items`**

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `order_id` | UUID | NOT NULL FK → `ord_orders` |
| `product_id` | UUID | NOT NULL |
| `product_name` | VARCHAR(255) | NOT NULL — snapshot |
| `quantity` | INT | NOT NULL — must be > 0 |
| `unit_price` | NUMERIC(10,2) | NOT NULL — snapshot |

**`ord_order_timeline`**

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `order_id` | UUID | NOT NULL FK → `ord_orders` |
| `event_type` | VARCHAR(100) | NOT NULL |
| `description` | TEXT | NOT NULL |
| `metadata` | JSONB | nullable |
| `occurred_at` | TIMESTAMPTZ | NOT NULL |

**`ord_outbox`** — same structure as `auth_outbox`.

Flyway scripts: `V1__create_ord_orders.sql`, `V2__create_ord_order_items.sql`, `V3__create_ord_order_timeline.sql`, `V4__create_ord_outbox.sql`


---

## 6. APIs

All endpoints are under `/api/v1/`. All request and response bodies are `application/json`. All timestamps are ISO 8601 UTC. All IDs are UUIDs. Authentication is `Authorization: Bearer {jwt}` unless marked **Public**.

### 6.1 Gateway Routing Table

| Path Prefix | Downstream Service | Port |
|---|---|---|
| `/api/v1/auth/**` | auth-service | 8081 |
| `/api/v1/products/**` | product-service | 8082 |
| `/api/v1/orders/**` | order-service | 8083 |

### 6.2 auth-service Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/auth/register` | Public | Register a new user |
| POST | `/api/v1/auth/login` | Public | Authenticate and receive tokens |
| POST | `/api/v1/auth/logout` | Authenticated | Revoke refresh token |
| POST | `/api/v1/auth/refresh` | Public | Exchange refresh token for new access token |
| GET | `/api/v1/auth/me` | Authenticated | Get current user profile |
| GET | `/api/v1/auth/public-key` | Public | Retrieve RS256 public key (PEM) |

**POST /api/v1/auth/register** — `201 Created`
```json
// Request
{ "firstName": "Jane", "lastName": "Doe", "email": "jane@example.com", "password": "S3cur3P@ss" }

// Response
{ "data": { "userId": "uuid", "email": "jane@example.com", "firstName": "Jane", "lastName": "Doe", "role": "CUSTOMER", "createdAt": "2024-01-15T10:30:00Z" } }
```
Errors: `400` (validation), `409` (email already registered)

**POST /api/v1/auth/login** — `200 OK`
```json
// Request
{ "email": "jane@example.com", "password": "S3cur3P@ss" }

// Response
{ "data": { "accessToken": "eyJ...", "refreshToken": "eyJ...", "expiresIn": 900, "tokenType": "Bearer" } }
```
Errors: `400`, `401` (invalid credentials), `429` (rate limited — 5 attempts per 15 min per IP)

**POST /api/v1/auth/logout** — `200 OK`
```json
// Request
{ "refreshToken": "eyJ..." }
// Response
{ "data": { "message": "Logged out successfully." } }
```
Errors: `401` (token not found or already revoked)

**POST /api/v1/auth/refresh** — `200 OK`
```json
// Request
{ "refreshToken": "eyJ..." }
// Response
{ "data": { "accessToken": "eyJ...", "expiresIn": 900, "tokenType": "Bearer" } }
```
Errors: `401` (invalid, expired, or revoked token)

**GET /api/v1/auth/me** — `200 OK`
```json
{ "data": { "userId": "uuid", "email": "jane@example.com", "firstName": "Jane", "lastName": "Doe", "role": "CUSTOMER", "createdAt": "2024-01-15T10:30:00Z" } }
```

---

### 6.3 product-service Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/products` | Authenticated | List products (paginated, filterable) |
| GET | `/api/v1/products/{productId}` | Authenticated | Get a single product |
| POST | `/api/v1/products` | Admin | Create a product |
| PATCH | `/api/v1/products/{productId}` | Admin | Update product details |
| POST | `/api/v1/products/{productId}/deactivate` | Admin | Deactivate a product |

**GET /api/v1/products** — `200 OK`

Query params: `page` (default 0), `size` (default 20, max 100), `category`, `search` (name prefix, case-insensitive), `sort` (e.g. `price,asc`)

```json
{
  "data": [{ "productId": "uuid", "name": "Wireless Keyboard", "description": "...", "category": "ELECTRONICS", "price": 49.99, "availableStock": null, "status": "ACTIVE", "createdAt": "2024-01-10T08:00:00Z" }],
  "pagination": { "page": 0, "size": 20, "totalElements": 1, "totalPages": 1 },
  "meta": { "stockDataAvailable": false, "requestId": "uuid", "timestamp": "2024-01-15T10:30:00Z" }
}
```

**POST /api/v1/products** — `201 Created` with `Location: /api/v1/products/{productId}`
```json
// Request
{ "name": "Wireless Keyboard", "description": "Compact wireless keyboard", "category": "ELECTRONICS", "price": 49.99, "initialStock": 200 }
// Response
{ "data": { "productId": "uuid", "name": "Wireless Keyboard", "category": "ELECTRONICS", "price": 49.99, "status": "ACTIVE", "createdAt": "2024-01-15T10:30:00Z" } }
```
Errors: `400` (validation), `403` (not admin)

**PATCH /api/v1/products/{productId}** — `200 OK` (all fields optional)
```json
// Request
{ "name": "Wireless Keyboard Pro", "price": 59.99, "description": "Updated description" }
```
Errors: `400`, `403`, `404`

**POST /api/v1/products/{productId}/deactivate** — `200 OK`
Returns updated product with `"status": "INACTIVE"`. Errors: `403`, `404`


---

### 6.4 order-service Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/orders` | Authenticated | Create a new order |
| GET | `/api/v1/orders` | Authenticated | List orders (own orders for customers, all for admins) |
| GET | `/api/v1/orders/{orderId}` | Authenticated | Get a single order |
| GET | `/api/v1/orders/{orderId}/timeline` | Authenticated | Get order event timeline |
| POST | `/api/v1/orders/{orderId}/cancel` | Authenticated | Cancel an order |

**POST /api/v1/orders** — `201 Created` with `Location: /api/v1/orders/{orderId}`

Optional header: `Idempotency-Key: {uuid}`

```json
// Request
{ "lineItems": [{ "productId": "uuid", "quantity": 2 }, { "productId": "uuid", "quantity": 1 }] }

// Response
{
  "data": {
    "orderId": "uuid", "customerId": "uuid", "status": "PENDING",
    "lineItems": [{ "productId": "uuid", "productName": "Wireless Keyboard", "quantity": 2, "unitPrice": 49.99, "subtotal": 99.98 }],
    "totalAmount": 99.98, "createdAt": "2024-01-15T10:30:00Z", "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```
Errors: `400` (validation), `422` (product not found or inactive), `503` (product-service unavailable)

If the same `Idempotency-Key` is reused within 24 hours, returns `200` with the original order.

**GET /api/v1/orders** — `200 OK`

Query params: `page`, `size`, `status`, `sort`. Customers see only their own orders; admins see all.

**GET /api/v1/orders/{orderId}** — `200 OK`

Errors: `403` (customer accessing another customer's order), `404`

**GET /api/v1/orders/{orderId}/timeline** — `200 OK`
```json
{
  "data": {
    "orderId": "uuid",
    "events": [
      { "eventType": "ORDER_PLACED", "description": "Order placed by customer", "occurredAt": "2024-01-15T10:30:00Z", "metadata": {} }
    ]
  }
}
```
Errors: `403`, `404`

**POST /api/v1/orders/{orderId}/cancel** — `200 OK`

In Phase 1, only `PENDING` orders can be cancelled (no saga states exist). Returns updated order with `"status": "CANCELLED"`. Idempotent: if already `CANCELLED`, returns `200`.

Errors: `403` (customer cancelling another customer's order), `404`, `409` (order not in cancellable state)

---

## 7. Security Model

### 7.1 JWT Structure

- Algorithm: RS256
- Private key: loaded from `JWT_PRIVATE_KEY` environment variable (PEM-encoded PKCS#8) in auth-service
- Public key: loaded from `JWT_PUBLIC_KEY` environment variable in gateway-service
- Access token expiry: 900 seconds (15 minutes)
- Claims: `sub` (userId UUID), `email`, `role` (`CUSTOMER` or `ADMIN`), `iat`, `exp`

### 7.2 Endpoint Access Rules

| Endpoint | Rule |
|---|---|
| `POST /api/v1/auth/register` | Public |
| `POST /api/v1/auth/login` | Public |
| `POST /api/v1/auth/refresh` | Public |
| `GET /api/v1/auth/public-key` | Public |
| `GET /api/v1/auth/me` | Any authenticated user |
| `POST /api/v1/auth/logout` | Any authenticated user |
| `GET /api/v1/products`, `GET /api/v1/products/{id}` | Any authenticated user |
| `POST /api/v1/products`, `PATCH /api/v1/products/{id}`, `POST /api/v1/products/{id}/deactivate` | `ADMIN` role only |
| `POST /api/v1/orders` | Any authenticated user (`CUSTOMER` or `ADMIN`) |
| `GET /api/v1/orders` | Customers see own orders; admins see all |
| `GET /api/v1/orders/{id}`, `GET /api/v1/orders/{id}/timeline` | Owner or `ADMIN` |
| `POST /api/v1/orders/{id}/cancel` | Owner (PENDING only) or `ADMIN` |

### 7.3 Gateway JWT Validation

The gateway validates the JWT signature using the RS256 public key on every non-whitelisted request. Invalid or expired tokens return `401`. The gateway does not inspect roles — role-based access control is enforced at the service layer using `@PreAuthorize`.

### 7.4 Secrets

- `JWT_PRIVATE_KEY` — injected into auth-service at runtime. Never in source code or config files.
- `JWT_PUBLIC_KEY` — injected into gateway-service at runtime.
- Database credentials — injected via `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD` environment variables.
- In local development, these are set in `docker-compose.yml`. In production (Phase 4), they come from AWS Secrets Manager.


---

## 8. Validation Rules

All request DTOs are Java `record` types with Bean Validation constraints. `@Valid` is applied to all `@RequestBody` parameters. Validation failures return `400 Bad Request` with field-level error details.

### 8.1 auth-service

**RegisterRequest**

| Field | Constraint |
|---|---|
| `firstName` | `@NotBlank`, max 100 characters |
| `lastName` | `@NotBlank`, max 100 characters |
| `email` | `@NotBlank`, `@Email`, max 255 characters |
| `password` | `@NotBlank`, min 8 characters, must contain at least one uppercase letter, one digit, and one special character |

**LoginRequest**

| Field | Constraint |
|---|---|
| `email` | `@NotBlank`, `@Email` |
| `password` | `@NotBlank` |

**RefreshTokenRequest / LogoutRequest**

| Field | Constraint |
|---|---|
| `refreshToken` | `@NotBlank` |

### 8.2 product-service

**CreateProductRequest**

| Field | Constraint |
|---|---|
| `name` | `@NotBlank`, max 255 characters |
| `description` | optional, max 2000 characters |
| `category` | `@NotBlank`, max 100 characters |
| `price` | `@NotNull`, `@DecimalMin("0.01")` |
| `initialStock` | `@NotNull`, `@Min(0)` |

**UpdateProductRequest** (all fields optional, at least one must be present)

| Field | Constraint |
|---|---|
| `name` | max 255 characters if provided |
| `description` | max 2000 characters if provided |
| `price` | `@DecimalMin("0.01")` if provided |

### 8.3 order-service

**CreateOrderRequest**

| Field | Constraint |
|---|---|
| `lineItems` | `@NotEmpty`, max 50 items |
| `lineItems[].productId` | `@NotNull`, valid UUID |
| `lineItems[].quantity` | `@NotNull`, `@Min(1)`, `@Max(1000)` |

Additional business validation (returns `422`, not `400`):
- Each `productId` must exist and be `ACTIVE` in product-service.
- Duplicate `productId` entries in the same request are rejected.

---

## 9. Error Response Format

All error responses follow the standard envelope defined in `common-api`.

```json
{
  "error": {
    "code": "SCREAMING_SNAKE_CASE_ERROR_CODE",
    "message": "Human-readable description.",
    "details": [],
    "requestId": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

For validation errors (`400`), `details` contains field-level errors:
```json
"details": [
  { "field": "email", "message": "must be a well-formed email address" },
  { "field": "password", "message": "must be at least 8 characters" }
]
```

### Error Code Reference (Phase 1)

| HTTP Status | Error Code | Trigger |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Bean Validation failure |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 401 | `TOKEN_REVOKED` | Refresh token has been revoked |
| 401 | `TOKEN_EXPIRED` | Refresh token has expired |
| 403 | `FORBIDDEN` | Authenticated but insufficient role |
| 404 | `USER_NOT_FOUND` | User does not exist |
| 404 | `PRODUCT_NOT_FOUND` | Product does not exist |
| 404 | `ORDER_NOT_FOUND` | Order does not exist |
| 409 | `EMAIL_ALREADY_REGISTERED` | Duplicate registration |
| 409 | `ORDER_NOT_CANCELLABLE` | Order is not in a cancellable state |
| 422 | `PRODUCT_INACTIVE` | Order references a deactivated product |
| 422 | `DUPLICATE_LINE_ITEMS` | Same productId appears more than once in an order |
| 429 | `RATE_LIMIT_EXCEEDED` | Login rate limit hit |
| 503 | `PRODUCT_SERVICE_UNAVAILABLE` | product-service unreachable during order creation |

A single `@RestControllerAdvice` class per service maps domain exceptions to these codes. Stack traces are never included in responses. 5xx errors are logged at `ERROR` level with full stack trace and `requestId`. 4xx errors are logged at `WARN` level without stack trace.


---

## 10. Inter-Service Communication

### 10.1 order-service → product-service

At order creation time, order-service calls product-service to resolve product name and unit price for each line item.

- Endpoint called: `GET /api/v1/products/{productId}`
- Wrapped in a Resilience4j circuit breaker (`productServiceCircuitBreaker`)
- Retry policy: 3 attempts, exponential backoff starting at 500 ms, jitter enabled
- Timeouts: connect 2 s, read 5 s
- Fallback: throw `ProductServiceUnavailableException` → controller maps to `503`
- If product is `INACTIVE` or returns `404`, order creation fails with `422`

### 10.2 product-service → inventory-service (Phase 1 stub)

product-service has a circuit-breaker-wrapped call to inventory-service for stock display. In Phase 1, inventory-service does not exist, so the circuit breaker fallback fires immediately and returns `availableStock: null` with `meta.stockDataAvailable: false`. The fallback is the correct production behaviour for degraded mode — no special Phase 1 code path is needed.

### 10.3 RestClient Configuration

Each service that makes outbound HTTP calls defines a `@Configuration` class that creates a `RestClient` bean with:
- Base URL from `@ConfigurationProperties`
- Connect and read timeouts set via `ClientHttpRequestFactory`
- Resilience4j circuit breaker and retry decorators applied via `RestClient` interceptor

---

## 11. Outbox Pattern (Phase 1 — Write Only)

In Phase 1, services write to their outbox tables within the same database transaction as the domain state change. The outbox poller (which reads `PENDING` rows and publishes to SNS) is implemented in Phase 2.

**Outbox write contract** (same for all services):

```
BEGIN TRANSACTION
  1. Persist domain entity (user, product, order)
  2. INSERT INTO {prefix}_outbox (id, event_type, aggregate_id, payload, correlation_id, status, created_at)
     VALUES (uuid, 'event.type.v1', aggregateId, jsonb, correlationId, 'PENDING', now())
COMMIT
```

The `correlation_id` is taken from the `X-Request-ID` header propagated by the gateway. If absent (e.g., direct service call in tests), a new UUID is generated.

This guarantees that an event row exists for every committed domain change, ready for Phase 2 publishing.

---

## 12. Local Development Plan

### 12.1 Prerequisites

- Docker Desktop (or equivalent) with at least 4 GB RAM allocated
- Java 21 JDK
- Maven 3.9+

### 12.2 docker-compose.yml Services (Phase 1)

```
auth-db       PostgreSQL 15 — port 5432
product-db    PostgreSQL 15 — port 5433
order-db      PostgreSQL 15 — port 5434
auth-service  port 8081
gateway-service port 8080
product-service port 8082
order-service   port 8083
```

LocalStack is included in `docker-compose.yml` (port 4566) but is not actively used in Phase 1 — it starts cleanly and the init script creates the SNS/SQS resources so Phase 2 can use them without changes to the compose file.

### 12.3 Environment Variables (docker-compose)

```yaml
# auth-service
JWT_PRIVATE_KEY: <RSA private key PEM — generated once, stored in .env file, gitignored>
SPRING_DATASOURCE_URL: jdbc:postgresql://auth-db:5432/authdb
SPRING_DATASOURCE_USERNAME: authuser
SPRING_DATASOURCE_PASSWORD: authpass

# gateway-service
JWT_PUBLIC_KEY: <RSA public key PEM — matches auth-service private key>
AUTH_SERVICE_URL: http://auth-service:8081
PRODUCT_SERVICE_URL: http://product-service:8082
ORDER_SERVICE_URL: http://order-service:8083

# product-service
SPRING_DATASOURCE_URL: jdbc:postgresql://product-db:5433/productdb
SPRING_DATASOURCE_USERNAME: productuser
SPRING_DATASOURCE_PASSWORD: productpass
ORDER_SERVICE_URL: http://order-service:8083

# order-service
SPRING_DATASOURCE_URL: jdbc:postgresql://order-db:5434/orderdb
SPRING_DATASOURCE_USERNAME: orderuser
SPRING_DATASOURCE_PASSWORD: orderpass
PRODUCT_SERVICE_URL: http://product-service:8082
```

A `.env.example` file is committed to the repository with placeholder values. The actual `.env` file is gitignored.

### 12.4 Starting the Stack

```bash
# Generate RSA key pair (one-time setup)
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem

# Copy .env.example to .env and fill in the key values
cp .env.example .env

# Start all Phase 1 services
docker-compose up --build
```

Swagger UI is available at:
- `http://localhost:8081/swagger-ui.html` — auth-service
- `http://localhost:8082/swagger-ui.html` — product-service
- `http://localhost:8083/swagger-ui.html` — order-service

### 12.5 Spring Profiles

- `local` — used in docker-compose. Points AWS SDK to LocalStack (`http://localhost:4566`). Enables Swagger UI.
- `test` — used by Testcontainers integration tests. Datasource and AWS endpoint URLs are injected via `@DynamicPropertySource`.
- `prod` — used in AWS (Phase 4). Disables Swagger UI. Reads secrets from environment variables injected by Secrets Manager.


---

## 13. Testing Plan

Coverage target: ≥ 80% line coverage on `service/` and `domain/` packages, enforced by Jacoco in the Maven build.

### 13.1 Unit Tests

Framework: JUnit 5 + Mockito + AssertJ. No Spring context. No I/O.

**auth-service**

| Class | Scenarios |
|---|---|
| `AuthServiceImpl` | Register success; duplicate email throws `EmailAlreadyRegisteredException`; login success; login with wrong password throws `InvalidCredentialsException`; refresh with valid token; refresh with revoked token; refresh with expired token; logout revokes token |
| `JwtService` | Token generated with correct claims; token expiry is correct; invalid signature rejected; expired token rejected |
| `RateLimiterService` | Allows 5 attempts; blocks 6th attempt within window; resets after window |

**product-service**

| Class | Scenarios |
|---|---|
| `ProductServiceImpl` | Create product success; create product with duplicate name (if unique constraint); update price; update name; deactivate active product; deactivate already-inactive product (idempotent); get active product; get inactive product returns correctly |
| `ProductMapper` | Entity → response DTO mapping; request DTO → entity mapping |

**order-service**

| Class | Scenarios |
|---|---|
| `OrderServiceImpl` | Create order success; create order with idempotency key (returns existing); create order with inactive product throws `ProductInactiveException`; create order when product-service unavailable throws `ProductServiceUnavailableException`; cancel PENDING order; cancel already-CANCELLED order (idempotent); cancel non-PENDING order throws `OrderNotCancellableException`; get own order; get another customer's order throws `ForbiddenException` |
| `OrderMapper` | Entity → response DTO; line item mapping with subtotal calculation |
| `TotalAmountCalculator` | Correct total for single item; correct total for multiple items; rounding to 2 decimal places |

### 13.2 Integration Tests

Framework: JUnit 5 + Testcontainers + `@SpringBootTest` + AssertJ + Awaitility. All integration test classes extend `AbstractIntegrationTest` and are tagged `@Tag("integration")`.

**auth-service integration tests**

| Test | Approach |
|---|---|
| `POST /register` happy path | `@SpringBootTest` + `TestRestTemplate`; assert `201`, response body, outbox row written |
| `POST /register` duplicate email | Assert `409`, no outbox row |
| `POST /register` validation errors | `@WebMvcTest`; assert `400` with field details |
| `POST /login` success | Assert `200`, JWT claims correct, refresh token stored |
| `POST /login` wrong password | Assert `401` |
| `POST /login` rate limit | 6 rapid requests; assert 6th returns `429` |
| `POST /logout` revokes token | Assert token `revoked = true` in DB; subsequent refresh returns `401` |
| `POST /refresh` with revoked token | Assert `401` |
| Flyway migrations | `@DataJpaTest`; assert all tables exist with correct columns |

**product-service integration tests**

| Test | Approach |
|---|---|
| `POST /products` admin creates product | Assert `201`, product in DB, outbox row written with `product.created.v1` |
| `POST /products` customer forbidden | Assert `403` |
| `GET /products` pagination | Assert correct page/size/totalElements |
| `GET /products` category filter | Assert only matching category returned |
| `GET /products` search | Assert prefix match, case-insensitive |
| `GET /products` excludes inactive | Assert deactivated product not in results |
| `POST /products/{id}/deactivate` | Assert `status = INACTIVE` in DB |
| `PATCH /products/{id}` price update | Assert new price in DB |
| Flyway migrations | `@DataJpaTest` |

**order-service integration tests**

| Test | Approach |
|---|---|
| `POST /orders` success | Assert `201`, order in DB with `PENDING` status, outbox row written, timeline entry `ORDER_PLACED` |
| `POST /orders` idempotency key reuse | Assert `200` with original order, no duplicate DB row |
| `POST /orders` product-service unavailable | Mock product-service with WireMock returning 503; assert `503` |
| `POST /orders` inactive product | Mock product-service returning inactive product; assert `422` |
| `GET /orders` customer sees own orders only | Two customers; assert each sees only their own |
| `GET /orders/{id}` wrong owner | Assert `403` |
| `GET /orders/{id}/timeline` | Assert events in chronological order |
| `POST /orders/{id}/cancel` PENDING order | Assert `200`, status `CANCELLED`, timeline entry `ORDER_CANCELLED` |
| `POST /orders/{id}/cancel` already cancelled | Assert `200` (idempotent) |
| `POST /orders/{id}/cancel` non-PENDING order | Assert `409` |
| Flyway migrations | `@DataJpaTest` |

**gateway-service integration tests**

| Test | Approach |
|---|---|
| Request without JWT | Assert `401` |
| Request with expired JWT | Assert `401` |
| Request with invalid signature | Assert `401` |
| Request with valid JWT routes correctly | Assert downstream service receives request |
| Rate limit enforcement | Burst requests; assert `429` after threshold |
| `X-Request-ID` injected | Assert header present on forwarded request |

### 13.3 Controller Tests (`@WebMvcTest`)

Each service has `@WebMvcTest` tests for:
- All validation error cases (missing fields, invalid formats, out-of-range values) — assert `400` with correct `details` array
- Auth enforcement — assert `401` for missing token, `403` for wrong role
- Response shape — assert all required fields present in success responses

### 13.4 Repository Tests (`@DataJpaTest`)

- Custom JPQL queries (e.g., `findByCustomerIdAndStatus`, `findByIdempotencyKey`)
- Flyway migration correctness
- Optimistic locking: concurrent updates to the same entity throw `OptimisticLockingFailureException`

### 13.5 CI Behaviour

- Unit tests run on every push (fast, no infrastructure)
- Integration tests run on every pull request (Testcontainers spins up PostgreSQL)
- Build fails if Jacoco coverage drops below 80% on `service/` and `domain/` packages
- Checkstyle (Google style) runs on every push; build fails on violations


---

## 14. Implementation Tasks

Tasks are ordered by dependency. Each task is independently completable and testable.

### Task 0 — Shared Libraries (prerequisite for all services)

- [ ] **T0.1** Root `pom.xml`: define `<dependencyManagement>` for Spring Boot 3.x, Resilience4j, AWS SDK v2, Testcontainers, JUnit 5, Mockito, AssertJ, Awaitility, MapStruct, Flyway, SpringDoc OpenAPI. Set Java source/target to 21. Configure Checkstyle (Google style) and Jacoco (80% threshold on `service/` and `domain/`).
- [ ] **T0.2** `common-api`: `ApiResponse<T>`, `PagedResponse<T>`, `ErrorResponse`, `ErrorDetail` records. Unit tests for serialisation.
- [ ] **T0.3** `common-events`: `EventEnvelope` record with all fields from the event standards. Unit tests for serialisation and `@JsonIgnoreProperties(ignoreUnknown = true)` forward-compatibility.
- [ ] **T0.4** `common-security`: `JwtValidationFilter`, `SecurityContextHelper` (extracts userId, email, role from `SecurityContext`), `JwtProperties` `@ConfigurationProperties` record. Unit tests for filter logic.
- [ ] **T0.5** `common-observability`: MDC filter (injects `traceId`, `correlationId`, `serviceId`, `spanId`), Logback JSON config, Micrometer config. Unit tests for MDC population.

### Task 1 — auth-service

- [ ] **T1.1** Maven module scaffold: `pom.xml`, `AuthServiceApplication.java`, package structure per `structure.md`.
- [ ] **T1.2** Flyway migrations: `V1__create_auth_users.sql`, `V2__create_auth_refresh_tokens.sql`, `V3__create_auth_outbox.sql`. `@DataJpaTest` verifies schema.
- [ ] **T1.3** Domain entities: `User`, `RefreshToken`, `OutboxEvent` JPA entities with `@Version` fields.
- [ ] **T1.4** Repositories: `UserRepository` (findByEmail), `RefreshTokenRepository` (findByTokenHash), `OutboxRepository`.
- [ ] **T1.5** `JwtService`: generate RS256 access token; generate opaque refresh token; validate access token; extract claims. Unit tests covering all paths.
- [ ] **T1.6** `AuthServiceImpl`: register (bcrypt hash, outbox write, duplicate check); login (credential check, rate limit, token issuance); logout (revoke token); refresh (validate, issue new access token); getMe. Unit tests for all paths.
- [ ] **T1.7** `AuthController`: all five endpoints with `@Valid`, `@Operation`, `@ApiResponse` annotations. `@WebMvcTest` for validation and auth.
- [ ] **T1.8** `SecurityConfig`: whitelist `/api/v1/auth/register`, `/api/v1/auth/login`, `/api/v1/auth/refresh`, `/api/v1/auth/public-key`. All other endpoints require authentication.
- [ ] **T1.9** Rate limiter: Resilience4j `RateLimiter` bean for login endpoint (5 requests per 15 min per IP). Unit test for rate limit enforcement.
- [ ] **T1.10** `application.yml`, `application-local.yml`, `application-test.yml`. `AuthProperties` and `JwtProperties` `@ConfigurationProperties` records.
- [ ] **T1.11** Integration tests: all scenarios from Section 13.2 auth-service block.

### Task 2 — gateway-service

- [ ] **T2.1** Maven module scaffold: `pom.xml`, `GatewayApplication.java`, Spring Cloud Gateway dependency.
- [ ] **T2.2** `RouteLocatorConfig`: define routes for auth, product, and order services using `RouteLocator` beans (not YAML). Strip `/api/v1` prefix before forwarding if needed.
- [ ] **T2.3** JWT validation filter: `JwtAuthenticationFilter` (Spring Cloud Gateway `GlobalFilter`) — validates RS256 signature using public key from `JWT_PUBLIC_KEY` env var; rejects with `401` if invalid/expired; passes `X-User-Id`, `X-User-Role` headers downstream.
- [ ] **T2.4** `X-Request-ID` filter: generate UUID if header absent; propagate to downstream; include in response.
- [ ] **T2.5** Rate limiting: `RequestRateLimiter` filter — 100 req/s per authenticated user (keyed on `X-User-Id`), 10 req/s per IP for unauthenticated endpoints.
- [ ] **T2.6** `application.yml`, `application-local.yml`. `GatewayProperties` `@ConfigurationProperties` record.
- [ ] **T2.7** Integration tests: JWT validation, routing, rate limiting, `X-Request-ID` propagation.

### Task 3 — product-service

- [ ] **T3.1** Maven module scaffold: `pom.xml`, `ProductServiceApplication.java`, package structure.
- [ ] **T3.2** Flyway migrations: `V1__create_prd_products.sql`, `V2__create_prd_outbox.sql`. `@DataJpaTest` verifies schema.
- [ ] **T3.3** `Product` JPA entity with `@Version`. `ProductRepository` with `findByStatusAndCategoryAndNameStartingWithIgnoreCase` query.
- [ ] **T3.4** `ProductServiceImpl`: create (outbox write); update; deactivate; list (pagination, filter, search); getById. Unit tests for all paths.
- [ ] **T3.5** `InventoryClient`: Resilience4j circuit-breaker-wrapped `RestClient` call to inventory-service. Fallback returns `Optional.empty()`. In Phase 1, the base URL points to a non-existent service — the circuit breaker opens immediately and the fallback fires. Unit test for fallback behaviour.
- [ ] **T3.6** `ProductController`: all endpoints with `@PreAuthorize("hasRole('ADMIN')")` on write operations. `@WebMvcTest` for validation and role enforcement.
- [ ] **T3.7** `application.yml`, `application-local.yml`, `application-test.yml`. `ProductServiceProperties` record.
- [ ] **T3.8** Integration tests: all scenarios from Section 13.2 product-service block.

### Task 4 — order-service

- [ ] **T4.1** Maven module scaffold: `pom.xml`, `OrderServiceApplication.java`, package structure.
- [ ] **T4.2** Flyway migrations: `V1__create_ord_orders.sql`, `V2__create_ord_order_items.sql`, `V3__create_ord_order_timeline.sql`, `V4__create_ord_outbox.sql`. `@DataJpaTest` verifies schema.
- [ ] **T4.3** `Order`, `OrderItem`, `OrderTimeline`, `OutboxEvent` JPA entities. `OrderRepository` with `findByCustomerId`, `findByIdempotencyKey` queries.
- [ ] **T4.4** `ProductClient`: Resilience4j circuit-breaker-wrapped `RestClient` call to product-service (`GET /api/v1/products/{id}`). Fallback throws `ProductServiceUnavailableException`. Unit tests for success, 404 (throws `ProductNotFoundException`), 422 (inactive product), and fallback.
- [ ] **T4.5** `OrderServiceImpl`: create order (product resolution, total calculation, idempotency check, outbox write, timeline entry); list orders (customer filter vs admin); get order (ownership check); get timeline; cancel order (state check, idempotency). Unit tests for all paths.
- [ ] **T4.6** `OrderController`: all endpoints. `@WebMvcTest` for validation, auth, and role enforcement.
- [ ] **T4.7** `application.yml`, `application-local.yml`, `application-test.yml`. `OrderServiceProperties` record.
- [ ] **T4.8** Integration tests: all scenarios from Section 13.2 order-service block.

### Task 5 — docker-compose and local validation

- [ ] **T5.1** `docker-compose.yml`: auth-db, product-db, order-db (PostgreSQL 15), LocalStack, auth-service, gateway-service, product-service, order-service. Health checks on all services. Correct environment variable injection.
- [ ] **T5.2** `.env.example` with placeholder values for all required environment variables.
- [ ] **T5.3** `infra/localstack/init.sh`: create SNS topics and SQS queues for Phase 2 (no-op in Phase 1 but ready).
- [ ] **T5.4** End-to-end smoke test (manual): register → login → create product (admin) → list products → create order → get order → get timeline → cancel order. All steps through the gateway on port 8080.
- [ ] **T5.5** Update `README.md` with Phase 1 local setup instructions, port reference, and environment variable table.

