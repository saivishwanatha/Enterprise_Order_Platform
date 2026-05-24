# Microservices Architecture Standards

## Core Principles

1. **Single Responsibility**: each service owns one bounded context. No service may read or write another service's database.
2. **Loose Coupling**: services communicate via REST (synchronous) or SNS/SQS events (asynchronous). No shared libraries that contain business logic.
3. **High Cohesion**: all logic for a domain concept lives in one service.
4. **Autonomous Deployability**: every service can be built, tested, and deployed independently.
5. **Failure Isolation**: a failing service must not cascade failures to others.

## Service Communication

### Synchronous (REST)
- Use only for operations that require an immediate response (e.g., auth token validation, product catalog reads).
- All inter-service HTTP calls go through `RestClient` (Spring 6) with a circuit breaker and retry wrapper.
- Never call another service's internal endpoints. Only call its public API contract.
- Timeouts: connect 2 s, read 5 s. These are non-negotiable defaults; increase only with documented justification.

### Asynchronous (SNS → SQS)
- Use for all state-change notifications and cross-service workflows.
- The producing service publishes to an SNS topic. Consuming services subscribe via SQS queues.
- Consumers must be idempotent. Use the event `eventId` to deduplicate.
- Dead-letter queues (DLQ) are mandatory for every SQS queue.

## Saga Pattern (Choreography-Based)

- All multi-service business transactions use the **Choreography Saga** pattern.
- Each service listens for events, performs its local transaction, then publishes its own result event.
- Compensating transactions must be implemented for every step that can fail.
- The order-service is the saga orchestrator of last resort — it listens for failure events and triggers compensation.

### Order Fulfillment Saga Flow

```
OrderPlacedEvent
  → inventory-service reserves stock → InventoryReservedEvent
    → payment-service captures payment → PaymentCapturedEvent
      → shipment-service creates shipment → ShipmentCreatedEvent
        → invoice-service generates invoice → InvoiceGeneratedEvent
          → notification-service sends confirmation
```

### Compensation Flow (on failure)

```
PaymentFailedEvent
  → inventory-service releases reservation → InventoryReleasedEvent
    → order-service marks order FAILED → OrderFailedEvent
      → notification-service sends failure notification
```

## Outbox Pattern

- Every service that publishes domain events **must** use the Transactional Outbox pattern.
- The outbox table (`{prefix}_outbox`) is written in the same local transaction as the domain state change.
- A scheduled poller (every 5 s) reads unpublished outbox rows and publishes them to SNS, then marks them `PUBLISHED`.
- Outbox rows older than 7 days and in `PUBLISHED` state are purged by a nightly job.
- Never publish directly to SNS inside a `@Transactional` method.

## Data Ownership

| Service | Owns |
|---|---|
| `order-service` | Orders, order items, order status history |
| `inventory-service` | Stock levels, reservations |
| `payment-service` | Payment records, refunds |
| `shipment-service` | Shipments, tracking events |
| `invoice-service` | Invoice metadata (PDF stored on S3) |
| `product-service` | Products, categories, pricing |
| `auth-service` | Users, roles, refresh tokens |
| `notification-service` | Notification logs (no business data) |

- Services may cache read-only snapshots of other services' data (e.g., product name on an order line), but must not treat that cache as the source of truth.

## Gateway Service

- `gateway-service` is the sole public entry point. No service port is exposed externally except the gateway.
- Responsibilities: JWT validation, routing, rate limiting, request/response logging.
- Uses **Spring Cloud Gateway** with `RouteLocator` beans (not YAML routes).
- Rate limiting: 100 req/s per authenticated user, 10 req/s per IP for unauthenticated endpoints.
- The gateway does not contain business logic.

## Resilience Requirements

- Every service must define a `@RestClientConfig` that applies circuit breaker + retry to all outbound calls.
- Health checks: `/actuator/health` must return `UP` only when the DB connection and any critical downstream dependencies are healthy.
- Services must handle `InventoryReservedEvent` / `PaymentCapturedEvent` etc. being delivered more than once without side effects.

## Versioning

- Services are versioned independently. Breaking API changes require a new major version (`/api/v2/...`).
- Old versions are supported for a minimum of 90 days after a new version is released.
- Event schema changes follow the backward-compatibility rules in `event-standards.md`.
