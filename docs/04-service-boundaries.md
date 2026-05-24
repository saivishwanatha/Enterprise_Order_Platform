# Service Boundaries

## Boundary Rules

1. A service owns its data exclusively. No other service may read or write its database.
2. A service exposes its data to others only via its REST API or domain events.
3. A service must not import another service's domain classes or repositories.
4. Shared code (DTOs, event POJOs, utilities) lives in `shared-libraries/` only.
5. A service may cache a read-only snapshot of another service's data (e.g., product name on an order line item), but must not treat that cache as authoritative.

---

## auth-service

**Bounded Context**: Identity and Access Management

**Owns**:
- User accounts (credentials, profile, roles)
- Refresh tokens
- Role assignments

**Responsibilities**:
- Register new users
- Authenticate users and issue JWT access + refresh tokens
- Expose the RS256 public key for token verification
- Validate refresh tokens and issue new access tokens
- Manage user roles (admin operations)

**Does NOT own**:
- Order history, product data, payment records — any business domain data

**Publishes**:
- `user.registered.v1`

**Consumes**: nothing

**Synchronous dependencies**: none

**Port**: 8081

---

## gateway-service

**Bounded Context**: API Ingress and Cross-Cutting Concerns

**Owns**: nothing (stateless)

**Responsibilities**:
- Validate JWT tokens on every inbound request
- Route requests to downstream services by path prefix
- Enforce rate limits
- Inject `X-Request-ID` correlation header
- Log all inbound requests and responses (excluding sensitive fields)

**Does NOT own**: any business data

**Publishes**: nothing

**Consumes**: nothing (reads auth-service public key at startup for JWT verification)

**Routing Table**:

| Path Prefix | Downstream Service |
|---|---|
| `/api/v1/auth/**` | auth-service |
| `/api/v1/products/**` | product-service |
| `/api/v1/orders/**` | order-service |
| `/api/v1/inventory/**` | inventory-service |
| `/api/v1/payments/**` | payment-service |
| `/api/v1/shipments/**` | shipment-service |
| `/api/v1/invoices/**` | invoice-service |

**Port**: 8080 (public)

---

## product-service

**Bounded Context**: Product Catalog

**Owns**:
- Products (name, description, category, price, status)
- Product categories

**Responsibilities**:
- CRUD operations on products (admin)
- Paginated, filterable product listings (all authenticated users)
- Publish `product.created.v1` when a new product is created so inventory-service can initialise stock

**Does NOT own**:
- Stock levels (owned by inventory-service)
- Order line items (owned by order-service)

**Publishes**:
- `product.created.v1`
- `product.updated.v1`
- `product.deactivated.v1`

**Consumes**: nothing

**Synchronous dependencies**: none

**Port**: 8082

---

## order-service

**Bounded Context**: Order Lifecycle Management

**Owns**:
- Orders (status, customer ID, total amount, timestamps)
- Order line items (product ID, product name snapshot, quantity, unit price)
- Order status history (timeline events)
- Outbox table for order events

**Responsibilities**:
- Create orders and initiate the fulfillment saga
- Maintain the order state machine: `PENDING` → `CONFIRMED` → `FAILED` / `CANCELLED`
- Listen for saga outcome events and advance order state
- Expose the order timeline API
- Allow admins to trigger retries on failed orders

**Order State Machine**:
```
PENDING
  ├─► CONFIRMED (shipment created + invoice generated)
  ├─► FAILED (inventory reservation failed OR payment failed)
  └─► CANCELLED (customer or admin cancellation)
```

**Does NOT own**:
- Inventory levels, payment records, shipment details, invoice files

**Publishes**:
- `order.placed.v1`
- `order.cancelled.v1`
- `order.failed.v1`
- `order.confirmed.v1`

**Consumes**:
- `inventory.reservation_failed.v1` → mark order FAILED, trigger compensation
- `payment.failed.v1` → mark order FAILED
- `shipment.created.v1` → record in timeline
- `shipment.delivered.v1` → update order status
- `invoice.generated.v1` → record in timeline, mark order CONFIRMED

**Synchronous dependencies**: none (all cross-service communication is event-driven)

**Port**: 8083

---

## inventory-service

**Bounded Context**: Stock Management

**Owns**:
- Stock items (product ID, available quantity, reserved quantity)
- Reservations (order ID, product ID, quantity, status)
- Processed events table (for idempotency)

**Responsibilities**:
- Reserve stock when an order is placed
- Release reservations on order failure or cancellation
- Confirm reservations when payment is captured (decrement available stock permanently)
- Initialise stock records when a new product is created

**Does NOT own**:
- Product details (name, price) — reads a snapshot from the event payload
- Order details

**Publishes**:
- `inventory.reserved.v1`
- `inventory.reservation_failed.v1`
- `inventory.released.v1`

**Consumes**:
- `order.placed.v1` → attempt reservation
- `order.cancelled.v1` → release reservation
- `payment.failed.v1` → release reservation
- `payment.captured.v1` → confirm reservation (finalise stock decrement)
- `product.created.v1` → initialise stock record

**Synchronous dependencies**: none

**Port**: 8084

---

## payment-service

**Bounded Context**: Payment Processing

**Owns**:
- Payment records (order ID, amount, status, method, timestamps)
- Refund records
- Processed events table (for idempotency)

**Responsibilities**:
- Capture payment after inventory is reserved
- Issue refunds on order cancellation
- Simulate payment success/failure (v1)

**Does NOT own**:
- Order state, inventory levels

**Publishes**:
- `payment.captured.v1`
- `payment.failed.v1`
- `payment.refunded.v1`

**Consumes**:
- `inventory.reserved.v1` → attempt payment capture
- `order.cancelled.v1` → issue refund if payment was captured

**Synchronous dependencies**: none

**Port**: 8085

---

## shipment-service

**Bounded Context**: Shipment and Delivery

**Owns**:
- Shipments (order ID, tracking number, carrier, status, estimated delivery)
- Shipment status history
- Processed events table (for idempotency)

**Responsibilities**:
- Create shipments after payment is captured
- Manage shipment status transitions
- Generate tracking numbers

**Does NOT own**:
- Order state, payment records

**Publishes**:
- `shipment.created.v1`
- `shipment.dispatched.v1`
- `shipment.in_transit.v1`
- `shipment.delivered.v1`

**Consumes**:
- `payment.captured.v1` → create shipment

**Synchronous dependencies**: none

**Port**: 8086

---

## invoice-service

**Bounded Context**: Invoice Generation and Storage

**Owns**:
- Invoice metadata (invoice ID, order ID, customer ID, amount, S3 key, generated timestamp)
- Processed events table (for idempotency)

**Responsibilities**:
- Generate PDF invoices after payment is captured
- Store PDFs in S3
- Provide pre-signed download URLs
- Idempotent: return existing invoice if already generated for an order

**Does NOT own**:
- Invoice PDF content after upload (S3 owns the file)
- Order or payment data

**Publishes**:
- `invoice.generated.v1`

**Consumes**:
- `payment.captured.v1` → generate invoice

**Synchronous dependencies**: AWS S3

**Port**: 8087

---

## notification-service

**Bounded Context**: Customer Notifications

**Owns**:
- Notification log (notification ID, recipient, channel, event type, status, sent timestamp)
- Processed events table (for idempotency)

**Responsibilities**:
- Send email notifications on domain events
- Log all notification attempts
- Deduplicate notifications using processed events table

**Does NOT own**:
- Any business domain data

**Publishes**: nothing

**Consumes**:
- `user.registered.v1` → welcome email
- `order.placed.v1` → order confirmation email
- `order.failed.v1` → failure notification email
- `payment.captured.v1` → payment confirmation email
- `shipment.created.v1` → shipping confirmation with tracking number
- `shipment.delivered.v1` → delivery confirmation email
- `invoice.generated.v1` → invoice available email with download link

**Synchronous dependencies**: email provider (SMTP / SES in production)

**Port**: 8088

---

## Dependency Graph

```
                    ┌─────────────┐
                    │   gateway   │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
        auth-svc       product-svc      order-svc
                           │               │
                    product.created    order.placed
                           │               │
                           ▼               ▼
                      inventory-svc ◄──────┘
                           │
                    inventory.reserved
                           │
                           ▼
                       payment-svc
                      /           \
             payment.captured   payment.failed
                /       \              \
         shipment-svc  invoice-svc   order-svc
              │              │       inventory-svc
         shipment.*    invoice.generated
              │              │
              └──────┬───────┘
                     ▼
              notification-svc
```
