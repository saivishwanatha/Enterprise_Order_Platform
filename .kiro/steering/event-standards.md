# Event Standards

## Event Envelope

Every domain event published to SNS must be wrapped in the `EventEnvelope` from `common-events`:

```json
{
  "eventId": "uuid-v4",
  "eventType": "order.placed.v1",
  "aggregateId": "uuid-of-the-aggregate",
  "aggregateType": "Order",
  "occurredAt": "2024-01-15T10:30:00Z",
  "correlationId": "uuid-propagated-from-request",
  "causationId": "uuid-of-the-event-that-caused-this",
  "serviceSource": "order-service",
  "schemaVersion": 1,
  "payload": { ... }
}
```

- `eventId`: unique per event instance. Used for deduplication.
- `eventType`: dot-separated, lowercase, versioned — `{aggregate}.{verb}.v{n}` (e.g., `order.placed.v1`).
- `correlationId`: propagated from the originating HTTP request's `X-Request-ID`. Never null.
- `causationId`: the `eventId` of the event that triggered this one (null for events originating from HTTP requests).
- `schemaVersion`: integer, incremented on breaking changes.

## Naming Conventions

- Event types: `{aggregate}.{past-tense-verb}.v{n}`
- Examples: `order.placed.v1`, `inventory.reserved.v1`, `payment.captured.v1`, `shipment.created.v1`
- SNS topic names: `{env}-{aggregate}-events` (e.g., `prod-order-events`)
- SQS queue names: `{env}-{service}-{aggregate}-queue` (e.g., `prod-inventory-order-queue`)
- DLQ names: `{env}-{service}-{aggregate}-dlq` (e.g., `prod-inventory-order-dlq`)

## Domain Events Catalogue

| Event Type | Producer | Consumers |
|---|---|---|
| `order.placed.v1` | order-service | inventory-service, notification-service |
| `order.cancelled.v1` | order-service | inventory-service, payment-service, notification-service |
| `order.failed.v1` | order-service | notification-service |
| `inventory.reserved.v1` | inventory-service | payment-service |
| `inventory.reservation_failed.v1` | inventory-service | order-service |
| `inventory.released.v1` | inventory-service | order-service |
| `payment.captured.v1` | payment-service | shipment-service, invoice-service |
| `payment.failed.v1` | payment-service | order-service, inventory-service |
| `payment.refunded.v1` | payment-service | order-service, notification-service |
| `shipment.created.v1` | shipment-service | notification-service |
| `shipment.delivered.v1` | shipment-service | order-service, notification-service |
| `invoice.generated.v1` | invoice-service | notification-service |

## Publishing Rules

- Events are published via the **Outbox Pattern** only. Never publish directly to SNS inside a `@Transactional` method.
- The outbox poller publishes to SNS using the `EventEnvelope` as the SNS message body.
- SNS message attributes must include `eventType` (String) to enable SQS subscription filter policies.
- Use SNS message filtering so each SQS queue only receives the event types it needs.

## Consuming Rules

- Every SQS listener must be idempotent. Check `eventId` against a processed-events table before handling.
- Processed event IDs are stored for 48 hours minimum.
- If processing fails, do not delete the message — let SQS retry up to the configured `maxReceiveCount` (default: 5).
- After `maxReceiveCount` retries, the message moves to the DLQ automatically.
- DLQ messages must trigger a CloudWatch alarm.
- Listeners must not throw unchecked exceptions that would prevent the message from being acknowledged after successful processing.

## Schema Evolution

- **Backward-compatible changes** (adding optional fields, adding new event types): increment nothing, no consumer changes required.
- **Breaking changes** (removing fields, renaming fields, changing field types): increment `schemaVersion` and create a new `eventType` version (e.g., `order.placed.v2`). Run both versions in parallel during migration.
- Never remove a field from a published event schema without a deprecation period of at least one release cycle.
- All event payload classes in `common-events` must be annotated with `@JsonIgnoreProperties(ignoreUnknown = true)` to tolerate forward-compatible additions.

## SQS Configuration Standards

- Visibility timeout: 30 s (must be greater than the maximum expected processing time).
- Message retention: 4 days.
- DLQ `maxReceiveCount`: 5.
- Long polling: `receiveMessageWaitTimeSeconds = 20`.
- All queues and topics are provisioned via Terraform. No manual console creation.

## Local Development

- LocalStack emulates SNS and SQS locally.
- Init scripts in `infra/localstack/` create all topics and queues on LocalStack startup.
- Use the `local` Spring profile which points AWS SDK endpoints to `http://localhost:4566`.
