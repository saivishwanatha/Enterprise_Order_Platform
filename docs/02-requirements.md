# Requirements

## Functional Requirements

### FR-01: User Registration
- A new user can register with an email address, password, first name, and last name.
- Email addresses must be unique across the system.
- Passwords are stored as bcrypt hashes. Plain-text passwords are never persisted or logged.
- On successful registration, the user receives a welcome notification.
- Duplicate registration attempts return a `409 Conflict` response.

**Acceptance Criteria**:
- Given a POST /api/v1/auth/register with a unique email and valid password, then the response is 201 and a `user.registered.v1` event is written to the outbox.
- Given a POST /api/v1/auth/register with an already-registered email, then the response is 409 and no event is written.
- Given a POST /api/v1/auth/register with a missing required field, then the response is 400 with field-level error details.

### FR-02: User Login
- A registered user can log in with email and password.
- On success, the system returns a short-lived JWT access token (15 min) and a long-lived refresh token (7 days).
- The refresh token can be used to obtain a new access token without re-authenticating.
- Failed login attempts are rate-limited (5 attempts per 15 minutes per IP).
- Tokens are validated at the API gateway and optionally at each downstream service.
- A user can log out by revoking their refresh token via `POST /api/v1/auth/logout`. The token is marked `revoked = true` in `auth_refresh_tokens`. Revoked tokens cannot be used to obtain new access tokens.
- Existing access tokens remain valid until their 15-minute expiry after logout (stateless JWT trade-off — explicitly accepted).

**Acceptance Criteria**:
- Given a POST /api/v1/auth/login with valid credentials, then the response is 200 with a JWT access token and a refresh token.
- Given a POST /api/v1/auth/login with an incorrect password 5 times within 15 minutes, then the 6th attempt returns 429.
- Given a POST /api/v1/auth/logout with a valid refresh token, then the token is revoked and a subsequent POST /api/v1/auth/refresh with that token returns 401.
- Given a POST /api/v1/auth/refresh with a revoked token, then the response is 401.

### FR-03: Product Listing
- Any authenticated user can list products with pagination and filtering by category and availability.
- Product data includes: ID, name, description, category, price, and available stock quantity.
- Stock quantity is a read-only projection fetched from inventory-service via a synchronous REST call. If inventory-service is unavailable, the product listing is returned with `availableStock: null` for all items (degraded mode). The response includes a `meta.stockDataAvailable: false` flag in degraded mode.
- Products can be searched by name (case-insensitive prefix match).

**Acceptance Criteria**:
- Given a GET /api/v1/products with a valid JWT, then the response is 200 with a paginated list of ACTIVE products including `availableStock` values.
- Given inventory-service is unavailable, then the response is still 200 with products listed and `availableStock: null` and `meta.stockDataAvailable: false`.
- Given a `category` filter, then only products in that category are returned.
- Given a `search` query, then only products whose name starts with the search term (case-insensitive) are returned.

### FR-04: Product Creation (Admin)
- Users with the `ADMIN` role can create new products.
- A product requires: name, description, category, price, and initial stock quantity.
- Creating a product also initialises a stock record in inventory-service via an event (`product.created.v1`).
- Admins can update product details and deactivate products.
- Deactivated products do not appear in customer-facing listings.
- When a product's price is updated, the change applies only to future orders. Existing order line items retain the price snapshot taken at order creation time.
- When a product is deactivated, inventory-service retains the stock record but marks it inactive. New orders referencing a deactivated product are rejected with 422.

**Acceptance Criteria**:
- Given an admin POSTs a valid product, then the response is 201, a `product.created.v1` event is in the outbox, and inventory-service will initialise a stock record when it consumes the event.
- Given a customer attempts to create a product, then the response is 403.
- Given an admin deactivates a product, then it no longer appears in GET /api/v1/products responses for customers.
- Given a customer attempts to order a deactivated product, then the response is 422.

### FR-05: Order Creation
- An authenticated customer can create an order containing one or more line items (product ID + quantity).
- The order request must include `productId` and `quantity` per line item. order-service resolves the current product name and unit price from product-service via a synchronous REST call at order creation time. If product-service is unavailable, order creation fails with `503 Service Unavailable`.
- The order is created in `PENDING` status.
- Order creation triggers the fulfillment saga: inventory reservation → payment → shipment → invoice.
- A customer can view their own orders. Admins can view all orders.
- An order contains: order ID, customer ID, line items (with product name and unit price snapshots), total amount, status, and timestamps.
- Order status visible to customers during the saga: `PENDING` → `INVENTORY_RESERVED` → `PAYMENT_CAPTURED` → `CONFIRMED`. Failed terminal state: `FAILED`. Customer-cancelled state: `CANCELLED`.

**Acceptance Criteria**:
- Given a customer with a valid JWT submits a POST /api/v1/orders with valid line items for in-stock products, then the response is 201, the order status is PENDING, and an `order.placed.v1` event is written to the outbox within the same transaction.
- Given the same request is submitted twice with the same `Idempotency-Key` header, then the second response is 200 with the original order — no duplicate saga is started.
- Given a line item references a non-existent or inactive product, then the response is 422 and no order is created.
- Given product-service is unavailable, then the response is 503 and no order is created.

### FR-06: Inventory Reservation
- When an order is placed, inventory-service attempts to reserve the requested quantities.
- If all items are available, stock is decremented and a reservation record is created.
- If any item is unavailable, the entire reservation fails and the order transitions to `FAILED`.
- Reservations are released if payment fails or the order is cancelled.
- Inventory operations are idempotent — replaying the same reservation event has no effect (checked via `processed_events` table).

**Acceptance Criteria**:
- Given an `order.placed.v1` event for in-stock items, then `inventory.reserved.v1` is published and `inv_stock_items.reserved_quantity` is incremented.
- Given an `order.placed.v1` event where any item has insufficient stock, then `inventory.reservation_failed.v1` is published and no stock is modified.
- Given the same `order.placed.v1` event is delivered twice, then the second delivery is a no-op (idempotency via `processed_events`).

### FR-07: Payment Processing
- After inventory is reserved, payment-service attempts to capture payment.
- Payment is simulated in v1 (configurable success/failure rate for testing).
- On success, the order transitions to `PAYMENT_CAPTURED` status and shipment is initiated.
- On failure, a compensation flow releases the inventory reservation and marks the order `FAILED`.
- Payment records are immutable once created. Refunds create a separate refund record.

**Acceptance Criteria**:
- Given an `inventory.reserved.v1` event and the simulated payment succeeds, then `payment.captured.v1` is published and the order status advances to `PAYMENT_CAPTURED`.
- Given an `inventory.reserved.v1` event and the simulated payment fails, then `payment.failed.v1` is published, inventory releases the reservation, and the order transitions to `FAILED`.
- Given the same `inventory.reserved.v1` event is delivered twice, then the second delivery is a no-op.

### FR-08: Shipment Creation
- After payment is captured, shipment-service creates a shipment record.
- A shipment contains: shipment ID, order ID, tracking number (generated), carrier, status, and estimated delivery date.
- Shipment status transitions: `CREATED` → `DISPATCHED` → `IN_TRANSIT` → `DELIVERED`.
- Status transitions are triggered by admin API calls (`POST /api/v1/shipments/{id}/dispatch`, `/in-transit`, `/deliver`). Each transition is idempotent — calling the same transition twice returns 200 with the current state.
- Each status transition publishes a domain event consumed by order-service and notification-service.

**Acceptance Criteria**:
- Given a `payment.captured.v1` event, then a shipment is created, `shipment.created.v1` is published, and the order status advances to `CONFIRMED`.
- Given an admin calls POST /api/v1/shipments/{id}/deliver, then `shipment.delivered.v1` is published and the order timeline is updated.
- Given the same `payment.captured.v1` event is delivered twice, then only one shipment is created (idempotency).

### FR-09: Invoice Generation
- After payment is captured, invoice-service generates a PDF invoice.
- The PDF is stored in S3 at `invoices/{year}/{month}/{invoiceId}.pdf`. The S3 bucket name is injected via the `AWS_INVOICE_BUCKET` environment variable.
- Invoice metadata (ID, order ID, S3 key, generated timestamp) is stored in the invoice-service database.
- Customers can retrieve a pre-signed S3 URL (15-minute expiry) to download their invoice via `GET /api/v1/invoices/{orderId}`. The pre-signed URL uses the regional S3 endpoint to avoid path-style URL deprecation issues.
- Invoice generation is idempotent — a second `payment.captured.v1` event for the same order returns the existing invoice without regenerating the PDF.
- Invoice failure does NOT block order confirmation. The order is marked `CONFIRMED` when `shipment.created.v1` is received, regardless of invoice status. If invoice generation fails permanently (DLQ), operations replays the message to regenerate.

**Acceptance Criteria**:
- Given a `payment.captured.v1` event, then a PDF is uploaded to S3, invoice metadata is persisted, and `invoice.generated.v1` is published.
- Given the same `payment.captured.v1` event is delivered twice, then only one invoice is generated and the second delivery is a no-op.
- Given a GET /api/v1/invoices/{orderId} for a generated invoice, then the response includes a pre-signed URL that expires in 15 minutes.

### FR-10: Notifications
- notification-service sends notifications on the following events:
  - Order placed (confirmation)
  - Order failed (with reason)
  - Payment captured
  - Shipment created (with tracking number)
  - Shipment delivered
  - Invoice generated (with download link)
- v1 supports email notifications only. The notification channel is extensible.
- Notification delivery failures are retried up to 5 times (matching the SQS `maxReceiveCount`) before being dead-lettered. This aligns with the platform-wide DLQ configuration.

**Acceptance Criteria**:
- Given an `order.placed.v1` event, then an order confirmation email is logged in `ntf_notifications` with status `SENT`.
- Given the same event is delivered twice, then only one notification is sent (idempotency via `processed_events`).
- Given the email provider is unavailable for all 5 retry attempts, then the message moves to the DLQ and a CloudWatch alarm fires.

### FR-11: Order Timeline
- Every order exposes a `/timeline` endpoint returning a chronological list of events.
- Each timeline entry contains: event type, timestamp, description, and relevant metadata.
- The timeline is built from domain events stored by order-service as it processes saga events.
- Admins can view the full internal timeline including system events. Customers see a simplified view (excludes internal saga events like `INVENTORY_CONFIRMED`).

**Acceptance Criteria**:
- Given a completed order, then GET /api/v1/orders/{id}/timeline returns events in chronological order covering all saga steps.
- Given a customer requests the timeline for an order they do not own, then the response is 403.
- Given an admin requests the timeline, then internal system events are included.

### FR-12: Failed Order Retry
- Admins can trigger a manual retry on an order in `FAILED` status.
- Retry re-publishes the `order.placed` event, restarting the saga from the beginning.
- The system enforces a maximum of 3 retry attempts per order.
- Each retry attempt is recorded in the order timeline.
- The retry endpoint is idempotent with respect to order state: if the order is already in `PENDING` (a previous retry is in progress), the endpoint returns `409` rather than starting a second concurrent saga.

**Acceptance Criteria**:
- Given an admin retries a FAILED order, then the order status resets to PENDING, `retry_count` increments, and `order.placed.v1` is published via the outbox.
- Given an admin retries an order that has already been retried 3 times, then the response is 409.
- Given an admin retries an order that is not in FAILED state, then the response is 409.
- Given the retry endpoint is called twice in rapid succession for the same order, then only one retry is initiated (idempotency guard on PENDING state check).

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
- The system must sustain 500 concurrent order placements with P99 < 500 ms and zero 5xx errors.
- Load testing is performed in Phase 4 using **Gatling** against a staging docker-compose stack. The test scenario: 500 virtual users, each placing one order with a 2-product line item, over a 60-second ramp-up. The test fails the build if P99 exceeds the threshold or any 5xx is returned.
- "Without degradation" is defined as: P99 remains within the stated threshold and the error rate stays below 0.1%.

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
- No service has a hard startup dependency on another service being healthy at startup time. The gateway caches the JWT public key locally (bundled as a config value or fetched with retries on startup) so it can start independently of auth-service.

### NFR-09: Data Retention and Privacy
- Customer PII (email address, name) is present in event payloads, notification logs, and order timeline metadata.
- Processed event records are retained for **7 days** minimum (matching SQS message retention of 4 days, with a 3-day buffer). Records older than 7 days in `PUBLISHED` or `processed` state are purged by a nightly job.
- Notification logs are retained for 90 days, then purged.
- Order data (including PII snapshots) is retained indefinitely in v1. A data deletion workflow is out of scope for v1 but must be designed before any production launch involving real customer data.
- No customer PII is written to CloudWatch logs. Email addresses and names must be masked or omitted from all structured log output.
