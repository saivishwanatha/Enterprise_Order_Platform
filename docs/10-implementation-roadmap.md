# Implementation Roadmap

## Guiding Principles

- Build the thinnest vertical slice that proves the architecture end-to-end before adding breadth.
- Each phase produces working, tested, deployable software — not just scaffolding.
- Shared libraries are built before the services that depend on them.
- The happy path is implemented before failure handling and compensation.

---

## Phase 0: Foundation (Week 1)

**Goal**: Repository structure, shared libraries, and local infrastructure are ready. No service code yet.

### Deliverables

- [ ] Root Maven POM with `<dependencyManagement>` for all dependencies (Spring Boot 3, Resilience4j, Testcontainers, AWS SDK v2, etc.)
- [ ] `shared-libraries/common-events` — `EventEnvelope` record, all event payload POJOs (one per event in the catalogue), Jackson config with `@JsonIgnoreProperties(ignoreUnknown = true)`
- [ ] `shared-libraries/common-api` — `ApiResponse<T>`, `PagedResponse<T>`, `ErrorResponse`, `ErrorDetail` records
- [ ] `shared-libraries/common-security` — JWT validation filter, `SecurityContextHelper`, `JwtProperties` config
- [ ] `shared-libraries/common-observability` — MDC filter (injects `traceId`, `correlationId`, `serviceId`), Logback JSON config, Micrometer config
- [ ] `infra/localstack/init.sh` — creates all SNS topics, SQS queues, DLQs, and S3 buckets on LocalStack startup
- [ ] `docker-compose.yml` — LocalStack, one PostgreSQL instance per service (8 instances), all services (stubbed with health endpoints)
- [ ] `infra/terraform/` — skeleton modules for VPC, RDS, SNS, SQS, S3, IAM
- [ ] Unit tests for all shared library classes
- [ ] Contract tests for all event payload schemas in `common-events`

### Definition of Done
- `docker-compose up` starts LocalStack and all PostgreSQL instances without errors.
- All shared library unit and contract tests pass.
- `mvn verify` passes from the root.

---

## Phase 1: Auth + Gateway (Week 2)

**Goal**: Users can register, log in, and all requests are authenticated at the gateway.

### Deliverables

- [ ] `auth-service` — full implementation: register, login, refresh token, `/me` endpoint
  - Flyway migrations for `auth_users`, `auth_refresh_tokens`, `auth_outbox`
  - JWT issued with RS256, private key loaded from environment variable
  - Outbox pattern: `user.registered.v1` published on registration
  - Rate limiting on login endpoint (in-memory, Resilience4j)
  - Unit tests: all service logic
  - Integration tests: all endpoints, outbox publishing, token validation
- [ ] `gateway-service` — routing, JWT validation, rate limiting, `X-Request-ID` injection
  - Routes configured for all 7 downstream services
  - JWT public key fetched from auth-service at startup
  - Integration tests: routing, auth rejection (401/403), rate limiting
- [ ] `notification-service` (partial) — `user.registered.v1` consumer → welcome email
  - Flyway migrations for `ntf_notifications`, `ntf_processed_events`
  - Idempotency check on `processed_events`
  - Integration test: event consumed, email logged, idempotency

### Definition of Done
- `POST /api/v1/auth/register` → `POST /api/v1/auth/login` → `GET /api/v1/auth/me` works end-to-end through the gateway.
- Unauthenticated requests to protected endpoints return `401`.
- Welcome notification is logged in `ntf_notifications`.

---

## Phase 2: Core Order Fulfillment Saga — Happy Path (Weeks 3–4)

**Goal**: A customer can place an order and it flows through inventory → payment → shipment → invoice → notification end-to-end.

### Deliverables

- [ ] `product-service` — full implementation
  - Flyway migrations for `prd_products`, `prd_outbox`
  - CRUD endpoints, pagination, filtering
  - `product.created.v1` published via outbox
  - Unit + integration tests

- [ ] `inventory-service` — full implementation
  - Flyway migrations for `inv_stock_items`, `inv_reservations`, `inv_reservation_items`, `inv_outbox`, `inv_processed_events`
  - Consumes `product.created.v1` → initialise stock
  - Consumes `order.placed.v1` → reserve stock → publish `inventory.reserved.v1`
  - Optimistic locking on `inv_stock_items` for concurrent reservations
  - Unit + integration tests (including idempotency and concurrent reservation tests)

- [ ] `order-service` — full implementation
  - Flyway migrations for `ord_orders`, `ord_order_items`, `ord_order_timeline`, `ord_outbox`, `ord_processed_events`
  - Create order endpoint, list/get endpoints, timeline endpoint
  - Publishes `order.placed.v1` via outbox
  - Consumes `inventory.reservation_failed.v1`, `payment.failed.v1`, `shipment.created.v1`, `invoice.generated.v1`
  - Unit + integration tests

- [ ] `payment-service` — full implementation
  - Flyway migrations for `pay_payments`, `pay_refunds`, `pay_outbox`, `pay_processed_events`
  - Consumes `inventory.reserved.v1` → capture payment → publish `payment.captured.v1` or `payment.failed.v1`
  - Configurable success/failure rate for testing
  - Unit + integration tests

- [ ] `shipment-service` — full implementation
  - Flyway migrations for `shp_shipments`, `shp_shipment_status_history`, `shp_outbox`, `shp_processed_events`
  - Consumes `payment.captured.v1` → create shipment → publish `shipment.created.v1`
  - Unit + integration tests

- [ ] `invoice-service` — full implementation
  - Flyway migrations for `ivs_invoices`, `ivs_processed_events`
  - Consumes `payment.captured.v1` → generate PDF → upload to S3 → publish `invoice.generated.v1`
  - Pre-signed URL endpoint
  - Unit + integration tests (LocalStack S3)

- [ ] `notification-service` — remaining consumers
  - `order.placed.v1`, `payment.captured.v1`, `shipment.created.v1`, `invoice.generated.v1`
  - Integration tests for each

### Definition of Done
- Full happy path works end-to-end via `docker-compose`: place order → inventory reserved → payment captured → shipment created → invoice generated → notifications sent.
- Order timeline shows all saga steps.
- All integration tests pass.

---

## Phase 3: Failure Handling and Compensation (Week 5)

**Goal**: The system handles all failure scenarios correctly and compensates automatically.

### Deliverables

- [ ] Saga compensation: `inventory.reservation_failed.v1` → order marked FAILED + notification
- [ ] Saga compensation: `payment.failed.v1` → inventory released + order marked FAILED + notification
- [ ] Order cancellation flow: `order.cancelled.v1` → inventory released + refund issued + notification
- [ ] Failed order retry endpoint (admin): max 3 attempts, timeline entry per retry
- [ ] DLQ monitoring: CloudWatch alarms for all DLQs
- [ ] Outbox stuck alert: CloudWatch metric for pending rows older than 10 minutes
- [ ] Circuit breakers on all outbound HTTP calls (gateway → services, product-service → inventory-service for stock display)
- [ ] Integration tests for all compensation flows
- [ ] Integration tests for idempotency (duplicate event delivery) for all consumers
- [ ] Integration tests for retry endpoint

### Definition of Done
- Payment failure automatically releases inventory and notifies the customer.
- Inventory failure marks the order FAILED and notifies the customer.
- Admin can retry a failed order up to 3 times.
- All compensation integration tests pass.

---

## Phase 4: Production Hardening (Week 6)

**Goal**: The system is ready for first AWS deployment.

### Deliverables

- [ ] Terraform: complete modules for all services (EC2, RDS, SQS, SNS, S3, IAM, Secrets Manager)
- [ ] Dockerfiles for all services (multi-stage build, non-root user, health check)
- [ ] CloudWatch log groups, metric filters, and dashboards for all services
- [ ] CloudWatch alarms: 5xx rate, P99 latency, DLQ depth, CPU utilisation
- [ ] AWS Secrets Manager integration: DB credentials and JWT private key loaded at startup
- [ ] `application-prod.yml` profiles for all services
- [ ] Smoke test suite: `docker-compose -f docker-compose.test.yml` runs a scripted happy-path test
- [ ] README.md: local setup instructions, environment variable reference, service port map
- [ ] All steering files reviewed and updated to reflect any implementation decisions made during build

### Definition of Done
- `terraform apply` provisions all AWS resources without errors.
- All services start successfully on EC2 with RDS and real SNS/SQS.
- Happy path order flow works on AWS.
- CloudWatch dashboards show metrics for all services.

---

## Service Port Reference

| Service | Local Port |
|---|---|
| gateway-service | 8080 |
| auth-service | 8081 |
| product-service | 8082 |
| order-service | 8083 |
| inventory-service | 8084 |
| payment-service | 8085 |
| shipment-service | 8086 |
| invoice-service | 8087 |
| notification-service | 8088 |
| LocalStack | 4566 |
| PostgreSQL (per service) | 5432–5439 |

---

## Dependency Build Order

```
Phase 0:
  common-events → common-api → common-security → common-observability

Phase 1:
  auth-service → gateway-service → notification-service (partial)

Phase 2:
  product-service
  inventory-service (depends on common-events)
  order-service (depends on common-events)
  payment-service (depends on common-events)
  shipment-service (depends on common-events)
  invoice-service (depends on common-events)
  notification-service (remaining consumers)

Phase 3:
  All services — compensation and failure handling additions

Phase 4:
  Infra, Docker, CloudWatch, Terraform
```
