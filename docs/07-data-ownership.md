# Data Ownership

## Principles

- Each service owns exactly one database (or schema). No other service may connect to it.
- Cross-service data needs are satisfied by events (async) or REST API calls (sync).
- Services may store denormalised snapshots of foreign data (e.g., `product_name` on an order line item) to avoid runtime coupling, but must not treat those snapshots as authoritative.
- All tables are prefixed with a service abbreviation to prevent naming collisions if schemas are ever co-located for cost reasons.

---

## auth-service Database

**Schema prefix**: `auth_`

### auth_users
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `email` | VARCHAR(255) UNIQUE NOT NULL | |
| `password_hash` | VARCHAR(255) NOT NULL | bcrypt |
| `first_name` | VARCHAR(100) NOT NULL | |
| `last_name` | VARCHAR(100) NOT NULL | |
| `role` | VARCHAR(20) NOT NULL | `CUSTOMER`, `ADMIN` |
| `active` | BOOLEAN NOT NULL DEFAULT true | |
| `version` | BIGINT NOT NULL DEFAULT 0 | Optimistic lock |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `updated_at` | TIMESTAMPTZ NOT NULL | |

### auth_refresh_tokens
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL FK → auth_users | |
| `token_hash` | VARCHAR(255) UNIQUE NOT NULL | SHA-256 of the token |
| `expires_at` | TIMESTAMPTZ NOT NULL | |
| `revoked` | BOOLEAN NOT NULL DEFAULT false | |
| `created_at` | TIMESTAMPTZ NOT NULL | |

### auth_outbox
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `event_type` | VARCHAR(100) NOT NULL | |
| `aggregate_id` | UUID NOT NULL | |
| `payload` | JSONB NOT NULL | |
| `correlation_id` | UUID NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `PENDING`, `PUBLISHED` |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `published_at` | TIMESTAMPTZ | |

---

## product-service Database

**Schema prefix**: `prd_`

### prd_products
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `name` | VARCHAR(255) NOT NULL | |
| `description` | TEXT | |
| `category` | VARCHAR(100) NOT NULL | |
| `price` | NUMERIC(10,2) NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `ACTIVE`, `INACTIVE` |
| `version` | BIGINT NOT NULL DEFAULT 0 | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `updated_at` | TIMESTAMPTZ NOT NULL | |

### prd_outbox
Same structure as `auth_outbox`.

---

## order-service Database

**Schema prefix**: `ord_`

### ord_orders
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `customer_id` | UUID NOT NULL | FK to auth-service (logical, not DB FK) |
| `status` | VARCHAR(30) NOT NULL | `PENDING`, `CONFIRMED`, `FAILED`, `CANCELLED` |
| `total_amount` | NUMERIC(10,2) NOT NULL | |
| `retry_count` | INT NOT NULL DEFAULT 0 | Max 3 |
| `idempotency_key` | UUID UNIQUE | From `Idempotency-Key` header |
| `version` | BIGINT NOT NULL DEFAULT 0 | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `updated_at` | TIMESTAMPTZ NOT NULL | |

### ord_order_items
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID NOT NULL FK → ord_orders | |
| `product_id` | UUID NOT NULL | Logical FK to product-service |
| `product_name` | VARCHAR(255) NOT NULL | Snapshot at order time |
| `quantity` | INT NOT NULL | |
| `unit_price` | NUMERIC(10,2) NOT NULL | Snapshot at order time |

### ord_order_timeline
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID NOT NULL FK → ord_orders | |
| `event_type` | VARCHAR(100) NOT NULL | Human-readable event name |
| `description` | TEXT NOT NULL | |
| `metadata` | JSONB | Additional context |
| `occurred_at` | TIMESTAMPTZ NOT NULL | |

### ord_outbox
Same structure as `auth_outbox`.

### ord_processed_events
| Column | Type | Notes |
|---|---|---|
| `event_id` | UUID PK | The `eventId` from the envelope |
| `event_type` | VARCHAR(100) NOT NULL | |
| `processed_at` | TIMESTAMPTZ NOT NULL | |

---

## inventory-service Database

**Schema prefix**: `inv_`

### inv_stock_items
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `product_id` | UUID UNIQUE NOT NULL | Logical FK to product-service |
| `available_quantity` | INT NOT NULL DEFAULT 0 | |
| `reserved_quantity` | INT NOT NULL DEFAULT 0 | |
| `version` | BIGINT NOT NULL DEFAULT 0 | Optimistic lock for concurrent reservations |
| `updated_at` | TIMESTAMPTZ NOT NULL | |

### inv_reservations
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID UNIQUE NOT NULL | One reservation per order |
| `status` | VARCHAR(20) NOT NULL | `RESERVED`, `CONFIRMED`, `RELEASED` |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `updated_at` | TIMESTAMPTZ NOT NULL | |

### inv_reservation_items
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `reservation_id` | UUID NOT NULL FK → inv_reservations | |
| `product_id` | UUID NOT NULL | |
| `quantity` | INT NOT NULL | |

### inv_outbox
Same structure as `auth_outbox`.

### inv_processed_events
Same structure as `ord_processed_events`.

---

## payment-service Database

**Schema prefix**: `pay_`

### pay_payments
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID UNIQUE NOT NULL | One payment per order |
| `customer_id` | UUID NOT NULL | |
| `amount` | NUMERIC(10,2) NOT NULL | |
| `currency` | VARCHAR(3) NOT NULL DEFAULT 'USD' | |
| `status` | VARCHAR(20) NOT NULL | `PENDING`, `CAPTURED`, `FAILED`, `REFUNDED` |
| `method` | VARCHAR(50) NOT NULL | `SIMULATED` in v1 |
| `version` | BIGINT NOT NULL DEFAULT 0 | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `updated_at` | TIMESTAMPTZ NOT NULL | |

### pay_refunds
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `payment_id` | UUID NOT NULL FK → pay_payments | |
| `order_id` | UUID NOT NULL | |
| `amount` | NUMERIC(10,2) NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `PENDING`, `COMPLETED` |
| `created_at` | TIMESTAMPTZ NOT NULL | |

### pay_outbox
Same structure as `auth_outbox`.

### pay_processed_events
Same structure as `ord_processed_events`.

---

## shipment-service Database

**Schema prefix**: `shp_`

### shp_shipments
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID UNIQUE NOT NULL | |
| `tracking_number` | VARCHAR(50) UNIQUE NOT NULL | |
| `carrier` | VARCHAR(50) NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `CREATED`, `DISPATCHED`, `IN_TRANSIT`, `DELIVERED` |
| `estimated_delivery` | DATE | |
| `version` | BIGINT NOT NULL DEFAULT 0 | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `updated_at` | TIMESTAMPTZ NOT NULL | |

### shp_shipment_status_history
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `shipment_id` | UUID NOT NULL FK → shp_shipments | |
| `status` | VARCHAR(20) NOT NULL | |
| `occurred_at` | TIMESTAMPTZ NOT NULL | |

### shp_outbox
Same structure as `auth_outbox`.

### shp_processed_events
Same structure as `ord_processed_events`.

---

## invoice-service Database

**Schema prefix**: `inv2_`  *(prefix `ivs_` to avoid collision with inventory)*

### ivs_invoices
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID UNIQUE NOT NULL | |
| `customer_id` | UUID NOT NULL | |
| `amount` | NUMERIC(10,2) NOT NULL | |
| `s3_bucket` | VARCHAR(255) NOT NULL | |
| `s3_key` | VARCHAR(500) NOT NULL | `invoices/{year}/{month}/{id}.pdf` |
| `generated_at` | TIMESTAMPTZ NOT NULL | |

### ivs_processed_events
Same structure as `ord_processed_events`.

---

## notification-service Database

**Schema prefix**: `ntf_`

### ntf_notifications
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `recipient_email` | VARCHAR(255) NOT NULL | |
| `channel` | VARCHAR(20) NOT NULL | `EMAIL` |
| `event_type` | VARCHAR(100) NOT NULL | |
| `subject` | VARCHAR(255) NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `SENT`, `FAILED` |
| `sent_at` | TIMESTAMPTZ | |
| `created_at` | TIMESTAMPTZ NOT NULL | |

### ntf_processed_events
Same structure as `ord_processed_events`.

---

## Cross-Service Data Access Summary

| Data Needed By | Data Owned By | Access Method |
|---|---|---|
| Order line item product name | product-service | Snapshot copied at order creation time (REST call to product-service) |
| Order line item unit price | product-service | Snapshot copied at order creation time |
| Inventory stock in product listing | inventory-service | REST call from product-service (or gateway aggregation) |
| Customer email in events | auth-service | Included in `order.placed.v1` payload (order-service fetches at order creation) |
| Order total in payment | order-service | Included in `inventory.reserved.v1` → carried through saga via event payloads |
