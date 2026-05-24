# Failure Handling

## Failure Categories

| Category | Examples | Strategy |
|---|---|---|
| **Transient infrastructure** | DB connection blip, SQS timeout | Retry with backoff |
| **Business rule violation** | Insufficient stock, payment declined | Saga compensation |
| **Poison message** | Malformed event payload | DLQ + alert |
| **Downstream service unavailable** | payment-service down | Circuit breaker + DLQ retry |
| **Duplicate event delivery** | SQS at-least-once delivery | Idempotent consumers + processed_events table |
| **Partial saga failure** | Payment captured but shipment fails | Compensation transaction |

---

## Idempotency: Processed Events Table

Every service that consumes SQS events maintains a `{prefix}_processed_events` table.

**Algorithm for every event listener**:
```
1. Begin transaction
2. SELECT event_id FROM processed_events WHERE event_id = :eventId FOR UPDATE
3. If found → commit, return (already processed, skip silently)
4. Execute business logic
5. INSERT INTO processed_events (event_id, event_type, processed_at)
6. Write outbox row if this step publishes an event
7. Commit transaction
```

This guarantees exactly-once processing semantics even when SQS delivers the same message multiple times.

Processed event records are retained for 48 hours, then purged by a nightly scheduled job.

---

## Saga Failure Scenarios

### Scenario 1: Inventory Reservation Fails

**Trigger**: `inventory.reservation_failed.v1` received by order-service

**Compensation**:
1. order-service marks order `FAILED`
2. order-service publishes `order.failed.v1`
3. notification-service sends failure email to customer
4. No inventory to release (reservation never succeeded)

**No payment to refund** (payment never attempted).

---

### Scenario 2: Payment Fails

**Trigger**: `payment.failed.v1` received by order-service and inventory-service

**Compensation**:
1. inventory-service receives `payment.failed.v1` → releases reservation → publishes `inventory.released.v1`
2. order-service receives `payment.failed.v1` → marks order `FAILED` → publishes `order.failed.v1`
3. notification-service sends failure email

**Order of compensation**: inventory release and order failure happen concurrently (both listen to `payment.failed.v1`). Order-service does not wait for inventory release before marking the order failed.

---

### Scenario 3: Shipment Creation Fails

**Trigger**: shipment-service throws an exception processing `payment.captured.v1`

**Handling**:
1. SQS retries the message up to 5 times (visibility timeout: 30 s, exponential backoff).
2. After 5 failures, message moves to `{env}-shipment-payment-dlq`.
3. CloudWatch alarm fires on DLQ depth > 0.
4. Operations team investigates and replays from DLQ.
5. Payment is NOT automatically refunded — operations decides whether to replay or manually compensate.

**Rationale**: shipment failure after payment capture is an operational issue, not a business rule failure. Automatic refund would be premature.

---

### Scenario 4: Invoice Generation Fails

**Trigger**: invoice-service fails to generate PDF or upload to S3

**Handling**:
1. SQS retries up to 5 times.
2. After 5 failures, message moves to DLQ.
3. Invoice failure does NOT block order confirmation — order-service marks the order `CONFIRMED` when shipment is created, regardless of invoice status.
4. Operations replays the DLQ message to regenerate the invoice.

**Rationale**: invoice is a supporting concern. The customer's order is fulfilled even if the invoice is delayed.

---

### Scenario 5: Notification Fails

**Trigger**: notification-service fails to send email

**Handling**:
1. SQS retries up to 5 times.
2. After 5 failures, message moves to DLQ.
3. Notification failure never affects order state.

---

### Scenario 6: Order Service Crashes Mid-Saga

**Trigger**: order-service crashes after writing the outbox row but before the poller publishes the event.

**Recovery**:
1. On restart, the outbox poller finds `PENDING` rows and publishes them.
2. The saga continues from where it left off.
3. If the event was already published before the crash, the consumer's idempotency check prevents double-processing.

---

### Scenario 7: Duplicate Order Placement

**Trigger**: client sends the same `POST /api/v1/orders` twice with the same `Idempotency-Key`.

**Handling**:
1. order-service checks the `idempotency_key` column on `ord_orders`.
2. If found, returns the original `201` response with the existing order.
3. No duplicate saga is started.

---

## Retry and DLQ Configuration

| Queue | Max Receive Count | Visibility Timeout | DLQ |
|---|---|---|---|
| All queues | 5 | 30 s | `{queue-name}-dlq` |

**SQS retry backoff**: SQS does not natively support exponential backoff. The visibility timeout is fixed at 30 s. For finer-grained retry control, the consumer can explicitly change the message visibility timeout on each retry attempt (1st: 30 s, 2nd: 60 s, 3rd: 120 s, 4th: 240 s, 5th: 480 s).

---

## Circuit Breakers (Outbound HTTP)

All inter-service REST calls use Resilience4j circuit breakers.

| Setting | Value |
|---|---|
| Failure rate threshold | 50% |
| Sliding window size | 10 calls |
| Wait duration in open state | 30 s |
| Permitted calls in half-open | 3 |
| Connect timeout | 2 s |
| Read timeout | 5 s |
| Max retry attempts | 3 |
| Retry backoff | Exponential, starting 500 ms, jitter enabled |

Every circuit-broken call must have a fallback method defined. Fallbacks either return a cached response, a degraded response, or throw a domain exception that the caller handles gracefully.

---

## Failed Order Retry (Admin)

Admins can retry a `FAILED` order via `POST /api/v1/orders/{orderId}/retry`.

**Rules**:
- Maximum 3 retry attempts per order (`retry_count` column on `ord_orders`).
- Retry re-publishes `order.placed.v1` with `retryAttempt` incremented.
- Each retry is recorded in `ord_order_timeline`.
- If `retry_count >= 3`, the endpoint returns `409 Conflict`.
- The order status is reset to `PENDING` before the event is published.

---

## Outbox Poller Failure Handling

- The outbox poller runs every 5 seconds as a `@Scheduled` method.
- If SNS publish fails, the row remains `PENDING` and will be retried on the next poll cycle.
- Rows are never deleted — they are marked `PUBLISHED` after successful delivery.
- If a row has been `PENDING` for more than 10 minutes, a CloudWatch metric is emitted as a warning.
- Rows in `PUBLISHED` state older than 7 days are purged by a nightly job.

---

## Observability for Failures

| Signal | Mechanism |
|---|---|
| DLQ message arrived | CloudWatch alarm: `ApproximateNumberOfMessagesVisible > 0` on any DLQ |
| 5xx error rate spike | CloudWatch alarm: `5xxErrorRate > 1%` over 5 minutes per service |
| Outbox stuck | CloudWatch metric: `OutboxPendingOlderThan10Min > 0` |
| Circuit breaker opened | Micrometer metric: `resilience4j.circuitbreaker.state` = `open` |
| Saga compensation triggered | Structured log at `WARN` level with `correlationId` and `orderId` |

All failure events are logged at `ERROR` or `WARN` level with `correlationId`, `orderId` (where applicable), and the full exception message (no stack trace in `WARN`, full stack trace in `ERROR`).
