# Requirements

## Functional Requirements

### FR-01: User Registration
- A new user can register with an email address, password, first name, and last name.
- Email addresses must be unique across the system.
- Passwords are stored as bcrypt hashes. Plain-text passwords are never persisted or logged.
- On successful registration, the user receives a welcome notification.
- Duplicate registration attempts return a `409 Conflict` response.

### FR-02: User Login
- A registered user can log in with email and password.
- On success, the system returns a short-lived JWT access token (15 min) and a long-lived refresh token (7 days).
- The refresh token can be used to obtain a new access token without re-authenticating.
- Failed login attempts are rate-limited (5 attempts per 15 minutes per IP).
- Tokens are validated at the API gateway and optionally at each downstream service.

### FR-03: Product Listing
- Any authenticated user can list products with pagination and filtering by category and availability.
- Product data includes: ID, name, description, category, price, and available stock quantity.
- Stock quantity is a read-only projection — inventory-service is the source of truth for stock levels.
- Products can be searched by name (case-insensitive prefix match).

### FR-04: Product Creation (Admin)
- Users with the `ADMIN` role can create new products.
- A product requires: name, description, category, price, and initial stock quantity.
- Creating a product also initialises a stock record in inventory-service via an event.
- Admins can update product details and deactivate products.
- Deactivated products do not appear in customer-facing listings.

### FR-05: Order Creation
- An authenticated customer can create an order containing one or more line items (product ID + quantity).
- The order is created in `PENDING` status.
- Order creation triggers the fulfillment saga: inventory reservation → payment → shipment → invoice.
- A customer can view their own orders. Admins can view all orders.
- An order contains: order ID, customer ID, line items, total amount, status, and timestamps.

### FR-06: Inventory Reservation
- When an order is placed, inventory-service attempts to reserve the requested quantities.
- If all items are available, stock is decremented and a reservation record is created.
- If any item is unavailable, the entire reservation fails and the order transitions to `FAILED`.
- Reservations are released if payment fails or the order is cancelled.
- Inventory operations are idempotent — replaying the same reservation event has no effect.

### FR-07: Payment Processing
- After inventory is reserved, payment-service attempts to capture payment.
- Payment is simulated in v1 (configurable success/failure rate for testing).
- On success, the order transitions toward `PAID` and shipment is initiated.
- On failure, a compensation flow releases the inventory reservation and marks the order `FAILED`.
- Payment records are immutable once created. Refunds create a separate refund record.

### FR-08: Shipment Creation
- After payment is captured, shipment-service creates a shipment record.
- A shipment contains: shipment ID, order ID, tracking number (generated), carrier, status, and estimated delivery date.
- Shipment status transitions: `CREATED` → `DISPATCHED` → `IN_TRANSIT` → `DELIVERED`.
- Each status transition publishes a domain event consumed by order-service and notification-service.

### FR-09: Invoice Generation
- After payment is captured, invoice-service generates a PDF invoice.
- The PDF is stored in S3 at `invoices/{year}/{month}/{invoiceId}.pdf`.
- Invoice metadata (ID, order ID, S3 key, generated timestamp) is stored in the invoice-service database.
- Customers can retrieve a pre-signed S3 URL (15-minute expiry) to download their invoice.
- Invoice generation is idempotent — a second event for the same order returns the existing invoice.

### FR-10: Notifications
- notification-service sends notifications on the following events:
  - Order placed (confirmation)
  - Order failed (with reason)
  - Payment captured
  - Shipment created (with tracking number)
  - Shipment delivered
  - Invoice generated (with download link)
- v1 supports email notifications only. The notification channel is extensible.
- Notification delivery failures are retried up to 3 times before being dead-lettered.

### FR-11: Order Timeline
- Every order exposes a `/timeline` endpoint returning a chronological list of events.
- Each timeline entry contains: event type, timestamp, description, and relevant metadata.
- The timeline is built from domain events stored by order-service as it processes saga events.
- Admins can view the full internal timeline including system events. Customers see a simplified view.

### FR-12: Failed Order Retry
- Admins can trigger a manual retry on an order in `FAILED` status.
- Retry re-publishes the `order.placed` event, restarting the saga from the beginning.
- The system enforces a maximum of 3 retry attempts per order.
- Each retry attempt is recorded in the order timeline.

---

## Non-Functional Requirements

### NFR-01: Availability
- Each service targets 99.9% uptime independently.
- The failure of any single non-critical service (notification, invoice) must not prevent order creation or payment.

### NFR-02: Consistency
- Strong consistency within a single service boundary (ACID transactions).
- Eventual consistency across service boundaries via the Saga pattern.
- Maximum acceptable lag for cross-service state propagation: 5 seconds under normal load.

### NFR-03: Performance
- Order creation API: P99 latency < 500 ms.
- Product listing API: P99 latency < 200 ms.
- The system must handle 500 concurrent order placements without degradation.

### NFR-04: Security
- All endpoints require JWT authentication except `/api/v1/auth/register` and `/api/v1/auth/login`.
- JWTs are signed with RS256. Private key stored in AWS Secrets Manager.
- All data in transit uses TLS 1.2+. All data at rest is encrypted (RDS, S3, SQS, SNS).
- No secrets in source code, config files, or logs.

### NFR-05: Observability
- Structured JSON logs with `traceId`, `spanId`, `correlationId`, and `serviceId` on every log line.
- CloudWatch dashboards per service: error rate, P99 latency, throughput, DLQ depth.
- Mandatory CloudWatch alarms: 5xx error rate > 1%, P99 > 2 s, DLQ depth > 0.

### NFR-06: Auditability
- Every order state change produces a domain event with a correlation ID.
- The order timeline API provides a complete, human-readable audit trail for any order.

### NFR-07: Recoverability
- All SQS queues have a DLQ with `maxReceiveCount = 5`.
- Failed orders are retryable by admins up to 3 times.
- The outbox poller retries unpublished events indefinitely until published.

### NFR-08: Deployability
- Each service is independently deployable via Docker.
- Local development requires only `docker-compose up` to start the full stack.
- No service has a hard startup dependency on another service being healthy.
