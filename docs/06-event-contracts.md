# Event Contracts

## Event Envelope

Every event published to SNS is wrapped in this envelope. The `payload` field contains the event-specific data.

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "order.placed.v1",
  "aggregateId": "order-uuid",
  "aggregateType": "Order",
  "occurredAt": "2024-01-15T10:30:00Z",
  "correlationId": "request-uuid-from-X-Request-ID",
  "causationId": "uuid-of-triggering-event-or-null",
  "serviceSource": "order-service",
  "schemaVersion": 1,
  "payload": { }
}
```

| Field | Type | Description |
|---|---|---|
| `eventId` | UUID | Unique per event instance. Used for deduplication. |
| `eventType` | String | `{aggregate}.{verb}.v{n}` — dot-separated, lowercase, versioned. |
| `aggregateId` | UUID | ID of the root aggregate this event is about. |
| `aggregateType` | String | PascalCase name of the aggregate (e.g., `Order`, `Payment`). |
| `occurredAt` | ISO 8601 UTC | When the event occurred (not when it was published). |
| `correlationId` | UUID | Propagated from the originating HTTP `X-Request-ID`. Never null. |
| `causationId` | UUID or null | `eventId` of the event that caused this one. Null for HTTP-originated events. |
| `serviceSource` | String | Name of the publishing service. |
| `schemaVersion` | Integer | Incremented on breaking payload changes. |

---

## SNS Topics

| Topic Name | Events Published |
|---|---|
| `{env}-order-events` | `order.placed.v1`, `order.cancelled.v1`, `order.failed.v1`, `order.confirmed.v1` |
| `{env}-inventory-events` | `inventory.reserved.v1`, `inventory.reservation_failed.v1`, `inventory.released.v1` |
| `{env}-payment-events` | `payment.captured.v1`, `payment.failed.v1`, `payment.refunded.v1` |
| `{env}-shipment-events` | `shipment.created.v1`, `shipment.dispatched.v1`, `shipment.in_transit.v1`, `shipment.delivered.v1` |
| `{env}-invoice-events` | `invoice.generated.v1` |
| `{env}-product-events` | `product.created.v1`, `product.deactivated.v1` |
| `{env}-user-events` | `user.registered.v1` |

Note: `product.updated.v1` is not published. Price changes apply to future orders only via the snapshot mechanism; no downstream service needs to react to a price update in v1.

---

## SQS Queues and Subscriptions

| Queue Name | Subscribed Topic | Filter: `eventType` | Consumer Service |
|---|---|---|---|
| `{env}-inventory-order-queue` | order-events | `order.placed.v1`, `order.cancelled.v1` | inventory-service |
| `{env}-inventory-payment-queue` | payment-events | `payment.failed.v1`, `payment.captured.v1` | inventory-service |
| `{env}-inventory-product-queue` | product-events | `product.created.v1` | inventory-service |
| `{env}-payment-inventory-queue` | inventory-events | `inventory.reserved.v1` | payment-service |
| `{env}-payment-order-queue` | order-events | `order.cancelled.v1` | payment-service |
| `{env}-shipment-payment-queue` | payment-events | `payment.captured.v1` | shipment-service |
| `{env}-invoice-payment-queue` | payment-events | `payment.captured.v1` | invoice-service |
| `{env}-order-inventory-queue` | inventory-events | `inventory.reservation_failed.v1` | order-service |
| `{env}-order-payment-queue` | payment-events | `payment.failed.v1` | order-service |
| `{env}-order-shipment-queue` | shipment-events | `shipment.created.v1`, `shipment.delivered.v1` | order-service |
| `{env}-order-invoice-queue` | invoice-events | `invoice.generated.v1` | order-service |
| `{env}-notification-order-queue` | order-events | `order.placed.v1`, `order.failed.v1` | notification-service |
| `{env}-notification-payment-queue` | payment-events | `payment.captured.v1` | notification-service |
| `{env}-notification-shipment-queue` | shipment-events | `shipment.created.v1`, `shipment.delivered.v1` | notification-service |
| `{env}-notification-invoice-queue` | invoice-events | `invoice.generated.v1` | notification-service |
| `{env}-notification-user-queue` | user-events | `user.registered.v1` | notification-service |

Every queue has a paired DLQ: `{queue-name}-dlq`.

---

## Event Payloads

### order.placed.v1
**Producer**: order-service  
**Consumers**: inventory-service, notification-service

```json
{
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "lineItems": [
    {
      "productId": "uuid",
      "productName": "Wireless Keyboard",
      "quantity": 2,
      "unitPrice": 49.99
    }
  ],
  "totalAmount": 99.98,
  "currency": "USD",
  "retryAttempt": 0
}
```

Note: `totalAmount` and `currency` are carried forward through the saga. inventory-service includes them in `inventory.reserved.v1` so payment-service can capture the correct amount without calling order-service.

---

### order.cancelled.v1
**Producer**: order-service  
**Consumers**: inventory-service, payment-service, notification-service

```json
{
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "reason": "CUSTOMER_REQUESTED",
  "cancelledAt": "2024-01-15T11:00:00Z"
}
```

---

### order.failed.v1
**Producer**: order-service  
**Consumers**: notification-service

```json
{
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "reason": "PAYMENT_FAILED",
  "failedAt": "2024-01-15T10:30:10Z"
}
```

---

### order.confirmed.v1
**Producer**: order-service  
**Consumers**: (future — analytics, loyalty)

```json
{
  "orderId": "uuid",
  "customerId": "uuid",
  "totalAmount": 99.98,
  "confirmedAt": "2024-01-15T10:30:15Z"
}
```

---

### inventory.reserved.v1
**Producer**: inventory-service  
**Consumers**: payment-service

```json
{
  "reservationId": "uuid",
  "orderId": "uuid",
  "customerId": "uuid",
  "totalAmount": 99.98,
  "currency": "USD",
  "reservedItems": [
    { "productId": "uuid", "quantity": 2 }
  ],
  "reservedAt": "2024-01-15T10:30:02Z"
}
```

Note: `totalAmount` and `currency` are carried from the `order.placed.v1` payload so payment-service does not need to call order-service to know the charge amount.

---

### inventory.reservation_failed.v1
**Producer**: inventory-service  
**Consumers**: order-service

```json
{
  "orderId": "uuid",
  "reason": "INSUFFICIENT_STOCK",
  "failedItems": [
    { "productId": "uuid", "requested": 2, "available": 1 }
  ],
  "failedAt": "2024-01-15T10:30:02Z"
}
```

---

### inventory.released.v1
**Producer**: inventory-service  
**Consumers**: order-service (for timeline)

```json
{
  "reservationId": "uuid",
  "orderId": "uuid",
  "releasedItems": [
    { "productId": "uuid", "quantity": 2 }
  ],
  "releasedAt": "2024-01-15T10:30:12Z"
}
```

---

### payment.captured.v1
**Producer**: payment-service  
**Consumers**: shipment-service, invoice-service, notification-service

```json
{
  "paymentId": "uuid",
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "amount": 99.98,
  "currency": "USD",
  "capturedAt": "2024-01-15T10:30:04Z"
}
```

---

### payment.failed.v1
**Producer**: payment-service  
**Consumers**: order-service, inventory-service

```json
{
  "paymentId": "uuid",
  "orderId": "uuid",
  "reason": "INSUFFICIENT_FUNDS",
  "failedAt": "2024-01-15T10:30:04Z"
}
```

---

### payment.refunded.v1
**Producer**: payment-service  
**Consumers**: order-service, notification-service

```json
{
  "refundId": "uuid",
  "paymentId": "uuid",
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "amount": 99.98,
  "refundedAt": "2024-01-15T11:00:05Z"
}
```

---

### shipment.created.v1
**Producer**: shipment-service  
**Consumers**: order-service, notification-service

```json
{
  "shipmentId": "uuid",
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "trackingNumber": "TRK-20240115-ABCD",
  "carrier": "INTERNAL",
  "estimatedDelivery": "2024-01-18",
  "createdAt": "2024-01-15T10:30:06Z"
}
```

---

### shipment.dispatched.v1 / shipment.in_transit.v1
**Producer**: shipment-service  
**Consumers**: order-service, notification-service

```json
{
  "shipmentId": "uuid",
  "orderId": "uuid",
  "trackingNumber": "TRK-20240115-ABCD",
  "status": "DISPATCHED",
  "occurredAt": "2024-01-15T14:00:00Z"
}
```

---

### shipment.delivered.v1
**Producer**: shipment-service  
**Consumers**: order-service, notification-service

```json
{
  "shipmentId": "uuid",
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "trackingNumber": "TRK-20240115-ABCD",
  "deliveredAt": "2024-01-18T14:30:00Z"
}
```

---

### invoice.generated.v1
**Producer**: invoice-service  
**Consumers**: order-service, notification-service

```json
{
  "invoiceId": "uuid",
  "orderId": "uuid",
  "customerId": "uuid",
  "customerEmail": "jane.doe@example.com",
  "amount": 99.98,
  "s3Key": "invoices/2024/01/invoice-uuid.pdf",
  "generatedAt": "2024-01-15T10:30:08Z"
}
```

---

### product.created.v1
**Producer**: product-service  
**Consumers**: inventory-service

```json
{
  "productId": "uuid",
  "name": "Wireless Keyboard",
  "initialStock": 200,
  "createdAt": "2024-01-10T08:00:00Z"
}
```

---

### user.registered.v1
**Producer**: auth-service  
**Consumers**: notification-service

```json
{
  "userId": "uuid",
  "email": "jane.doe@example.com",
  "firstName": "Jane",
  "registeredAt": "2024-01-15T09:00:00Z"
}
```

---

## Schema Evolution Rules

1. **Backward-compatible additions** (new optional fields): no version bump required. Consumers use `@JsonIgnoreProperties(ignoreUnknown = true)`.
2. **Breaking changes** (removing or renaming fields, changing types): bump the version suffix (`v1` → `v2`). Run both versions in parallel during migration. Deprecate the old version with a minimum 90-day notice.
3. Never remove a field without a deprecation period.
4. All event payload classes in `common-events` must be annotated with `@JsonIgnoreProperties(ignoreUnknown = true)`.
