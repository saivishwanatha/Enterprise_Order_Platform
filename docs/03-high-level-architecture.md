# High-Level Architecture

## System Overview

The platform is a collection of independently deployable microservices behind a single API gateway. Services communicate synchronously via REST for queries and asynchronously via AWS SNS/SQS for all state-changing cross-service workflows. Each service owns its own PostgreSQL database. No service reads another service's database directly.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            CLIENTS                                       │
│                  (Browser / Mobile / Admin UI)                           │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY SERVICE                               │
│              (Spring Cloud Gateway — port 8080)                          │
│   JWT Validation │ Routing │ Rate Limiting │ Request/Response Logging    │
└──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬─────────────┘
   │      │      │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
 AUTH  PRODUCT  ORDER  INVEN  PAYMNT SHIPMNT INVCE  NOTIF  (future)
  SVC    SVC     SVC   TORY    SVC    SVC     SVC    SVC
         SVC
   │      │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
  RDS    RDS    RDS    RDS    RDS    RDS    RDS    RDS
 (auth) (prod) (ord)  (inv)  (pay)  (ship) (inv)  (notif)

                        │ SNS/SQS Event Bus │
                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     AWS SNS TOPICS (fan-out)                             │
│  order-events │ inventory-events │ payment-events │ shipment-events      │
│  product-events │ invoice-events                                         │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ SQS Subscriptions (per consumer)
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     SQS QUEUES (per consumer)                            │
│  inventory-order-queue  │  payment-inventory-queue                       │
│  shipment-payment-queue │  invoice-payment-queue                         │
│  notification-*-queues  │  order-*-queues (for saga callbacks)           │
│  + DLQ for every queue                                                   │
└─────────────────────────────────────────────────────────────────────────┘

                        S3: invoice PDFs
                        CloudWatch: logs, metrics, alarms
```

## Component Descriptions

### API Gateway Service
- Single public entry point. No service port is exposed externally.
- Validates JWT tokens on every request (except `/auth/register` and `/auth/login`).
- Routes requests to downstream services based on path prefix.
- Enforces rate limits: 100 req/s per authenticated user, 10 req/s per IP for public endpoints.
- Adds `X-Request-ID` (correlation ID) to every request before forwarding.

### Auth Service
- Issues JWT access tokens (RS256, 15 min) and refresh tokens (7 days).
- Manages user accounts and roles (`CUSTOMER`, `ADMIN`).
- Exposes the public key endpoint used by the gateway for token verification.

### Product Service
- Manages the product catalog: create, update, deactivate products.
- Publishes `product.created.v1` events so inventory-service can initialise stock records.
- Provides paginated, filterable product listings.

### Order Service
- Owns the order lifecycle and state machine.
- Creates orders in `PENDING` state and publishes `order.placed.v1` to start the saga.
- Listens for saga outcome events to advance or fail the order.
- Maintains the order timeline (event log per order).
- Exposes retry endpoint for admins.

### Inventory Service
- Manages stock levels and reservations.
- Listens for `order.placed.v1` → reserves stock → publishes `inventory.reserved.v1` or `inventory.reservation_failed.v1`.
- Listens for `order.cancelled.v1` and `payment.failed.v1` → releases reservations.

### Payment Service
- Simulates payment capture and failure.
- Listens for `inventory.reserved.v1` → attempts payment → publishes `payment.captured.v1` or `payment.failed.v1`.
- Listens for `order.cancelled.v1` → issues refund → publishes `payment.refunded.v1`.

### Shipment Service
- Listens for `payment.captured.v1` → creates shipment → publishes `shipment.created.v1`.
- Manages shipment status transitions and publishes events at each transition.

### Invoice Service
- Listens for `payment.captured.v1` → generates PDF → stores in S3 → publishes `invoice.generated.v1`.
- Exposes a pre-signed URL endpoint for invoice download.

### Notification Service
- Listens for all customer-facing events and sends email notifications.
- Stateless except for a notification log table for deduplication and audit.

## Data Flow: Happy Path Order

```
1. Customer → POST /api/v1/orders (gateway → order-service)
2. order-service creates order (PENDING), writes outbox row, commits
3. Outbox poller publishes order.placed.v1 → SNS order-events topic
4. inventory-service receives event via SQS, reserves stock
5. inventory-service publishes inventory.reserved.v1
6. payment-service receives event, captures payment
7. payment-service publishes payment.captured.v1
8. shipment-service receives event, creates shipment
9. invoice-service receives event, generates PDF, uploads to S3
10. notification-service receives events, sends emails
11. order-service receives shipment/invoice events, updates order status to CONFIRMED
```

## Infrastructure Layers

### Local Development
- `docker-compose.yml` starts all services + PostgreSQL instances + LocalStack.
- LocalStack emulates SNS, SQS, S3, and Secrets Manager.
- All services use the `local` Spring profile pointing AWS SDK to `localhost:4566`.

### AWS (First Deployment)
- Services run on EC2 instances (one per service, or ECS on EC2).
- Each service has its own RDS PostgreSQL instance in a private subnet.
- SNS topics and SQS queues provisioned via Terraform.
- S3 bucket for invoices with SSE-S3 encryption and blocked public access.
- CloudWatch for logs (structured JSON), metrics, and alarms.
- All resources tagged: `Project=enterprise-order-platform`, `Environment={env}`.

## Key Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Service decomposition | Microservices | Independent deployability, team autonomy (see ADR-001) |
| Data isolation | Database per service | No cross-service joins, independent schema evolution (see ADR-002) |
| Async messaging | AWS SNS + SQS | Managed, durable, fan-out capable (see ADR-003) |
| Event publishing safety | Outbox pattern | Prevents dual-write problem (see ADR-004) |
| Cross-service transactions | Choreography Saga | No distributed locks, eventual consistency (see ADR-005) |
