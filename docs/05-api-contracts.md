# API Contracts

All endpoints are prefixed with `/api/v1/`. All requests and responses use `application/json`. All timestamps are ISO 8601 UTC. All IDs are UUIDs. Authentication is via `Authorization: Bearer {jwt}` unless marked **Public**.

---

## Auth Service (`/api/v1/auth`)

### POST /api/v1/auth/register — Public
Register a new user.

**Request**
```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane.doe@example.com",
  "password": "S3cur3P@ssword"
}
```

**Response 201**
```json
{
  "data": {
    "userId": "uuid",
    "email": "jane.doe@example.com",
    "firstName": "Jane",
    "lastName": "Doe",
    "role": "CUSTOMER",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

**Errors**: `400` (validation), `409` (email already registered)

---

### POST /api/v1/auth/login — Public
Authenticate and receive tokens.

**Request**
```json
{
  "email": "jane.doe@example.com",
  "password": "S3cur3P@ssword"
}
```

**Response 200**
```json
{
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "expiresIn": 900,
    "tokenType": "Bearer"
  }
}
```

**Errors**: `400` (validation), `401` (invalid credentials), `429` (rate limited)

---

### POST /api/v1/auth/refresh — Public
Exchange a refresh token for a new access token.

**Request**
```json
{ "refreshToken": "eyJ..." }
```

**Response 200**
```json
{
  "data": {
    "accessToken": "eyJ...",
    "expiresIn": 900,
    "tokenType": "Bearer"
  }
}
```

**Errors**: `401` (invalid or expired refresh token)

---

### GET /api/v1/auth/me — Authenticated
Get the current user's profile.

**Response 200**
```json
{
  "data": {
    "userId": "uuid",
    "email": "jane.doe@example.com",
    "firstName": "Jane",
    "lastName": "Doe",
    "role": "CUSTOMER",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

---

## Product Service (`/api/v1/products`)

### GET /api/v1/products — Authenticated
List products with pagination and filtering.

**Query Parameters**: `page` (default 0), `size` (default 20, max 100), `category`, `search` (name prefix), `sort` (e.g. `price,asc`)

**Response 200**
```json
{
  "data": [
    {
      "productId": "uuid",
      "name": "Wireless Keyboard",
      "description": "Compact wireless keyboard",
      "category": "ELECTRONICS",
      "price": 49.99,
      "availableStock": 150,
      "status": "ACTIVE",
      "createdAt": "2024-01-10T08:00:00Z"
    }
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "totalPages": 1
  }
}
```

---

### GET /api/v1/products/{productId} — Authenticated
Get a single product by ID.

**Response 200**: single product object (same shape as list item)

**Errors**: `404`

---

### POST /api/v1/products — Admin only
Create a new product.

**Request**
```json
{
  "name": "Wireless Keyboard",
  "description": "Compact wireless keyboard",
  "category": "ELECTRONICS",
  "price": 49.99,
  "initialStock": 200
}
```

**Response 201** with `Location: /api/v1/products/{productId}`
```json
{
  "data": {
    "productId": "uuid",
    "name": "Wireless Keyboard",
    "category": "ELECTRONICS",
    "price": 49.99,
    "status": "ACTIVE",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

**Errors**: `400` (validation), `403` (not admin)

---

### PATCH /api/v1/products/{productId} — Admin only
Update product details.

**Request** (all fields optional)
```json
{
  "name": "Wireless Keyboard Pro",
  "price": 59.99,
  "description": "Updated description"
}
```

**Response 200**: updated product object

**Errors**: `400`, `403`, `404`

---

### POST /api/v1/products/{productId}/deactivate — Admin only
Deactivate a product (removes from customer listings).

**Response 200**: updated product object with `"status": "INACTIVE"`

---

## Order Service (`/api/v1/orders`)

### POST /api/v1/orders — Authenticated
Create a new order.

**Request**
```json
{
  "lineItems": [
    { "productId": "uuid", "quantity": 2 },
    { "productId": "uuid", "quantity": 1 }
  ]
}
```

**Headers**: `Idempotency-Key: {uuid}` (optional, prevents duplicate orders)

**Response 201** with `Location: /api/v1/orders/{orderId}`
```json
{
  "data": {
    "orderId": "uuid",
    "customerId": "uuid",
    "status": "PENDING",
    "lineItems": [
      {
        "productId": "uuid",
        "productName": "Wireless Keyboard",
        "quantity": 2,
        "unitPrice": 49.99,
        "subtotal": 99.98
      }
    ],
    "totalAmount": 99.98,
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

**Errors**: `400` (validation), `422` (product not found or inactive)

---

### GET /api/v1/orders — Authenticated
List orders. Customers see only their own orders. Admins see all.

**Query Parameters**: `page`, `size`, `status`, `sort`

**Response 200**: paginated list of order objects

---

### GET /api/v1/orders/{orderId} — Authenticated
Get a single order. Customers can only access their own orders.

**Response 200**: single order object

**Errors**: `403` (not owner and not admin), `404`

---

### GET /api/v1/orders/{orderId}/timeline — Authenticated
Get the event timeline for an order.

**Response 200**
```json
{
  "data": {
    "orderId": "uuid",
    "events": [
      {
        "eventType": "ORDER_PLACED",
        "description": "Order placed by customer",
        "occurredAt": "2024-01-15T10:30:00Z",
        "metadata": {}
      },
      {
        "eventType": "INVENTORY_RESERVED",
        "description": "Stock reserved for all items",
        "occurredAt": "2024-01-15T10:30:02Z",
        "metadata": { "reservationId": "uuid" }
      },
      {
        "eventType": "PAYMENT_CAPTURED",
        "description": "Payment of $99.98 captured",
        "occurredAt": "2024-01-15T10:30:04Z",
        "metadata": { "paymentId": "uuid", "amount": 99.98 }
      }
    ]
  }
}
```

---

### POST /api/v1/orders/{orderId}/cancel — Authenticated
Cancel an order. Customers can cancel their own `PENDING` orders. Admins can cancel any order.

**Response 200**: updated order object with `"status": "CANCELLED"`

**Errors**: `403`, `404`, `409` (order not in cancellable state)

---

### POST /api/v1/orders/{orderId}/retry — Admin only
Retry a failed order (max 3 attempts).

**Response 202**
```json
{
  "data": {
    "orderId": "uuid",
    "status": "PENDING",
    "retryAttempt": 2,
    "message": "Order retry initiated"
  }
}
```

**Errors**: `403`, `404`, `409` (order not in FAILED state or max retries exceeded)

---

## Inventory Service (`/api/v1/inventory`)

### GET /api/v1/inventory/{productId} — Authenticated
Get stock level for a product.

**Response 200**
```json
{
  "data": {
    "productId": "uuid",
    "availableQuantity": 150,
    "reservedQuantity": 10,
    "totalQuantity": 160,
    "updatedAt": "2024-01-15T10:00:00Z"
  }
}
```

**Errors**: `404`

---

### PATCH /api/v1/inventory/{productId}/stock — Admin only
Manually adjust stock level (e.g., after physical stock count).

**Request**
```json
{
  "adjustment": 50,
  "reason": "Stock count correction"
}
```

**Response 200**: updated stock object

---

## Payment Service (`/api/v1/payments`)

### GET /api/v1/payments/{orderId} — Authenticated
Get payment record for an order. Customers can only access their own orders' payments.

**Response 200**
```json
{
  "data": {
    "paymentId": "uuid",
    "orderId": "uuid",
    "amount": 99.98,
    "status": "CAPTURED",
    "method": "SIMULATED",
    "capturedAt": "2024-01-15T10:30:04Z"
  }
}
```

**Errors**: `403`, `404`

---

## Shipment Service (`/api/v1/shipments`)

### GET /api/v1/shipments/{orderId} — Authenticated
Get shipment for an order.

**Response 200**
```json
{
  "data": {
    "shipmentId": "uuid",
    "orderId": "uuid",
    "trackingNumber": "TRK-20240115-ABCD",
    "carrier": "INTERNAL",
    "status": "DISPATCHED",
    "estimatedDelivery": "2024-01-18",
    "createdAt": "2024-01-15T10:30:06Z",
    "statusHistory": [
      { "status": "CREATED", "occurredAt": "2024-01-15T10:30:06Z" },
      { "status": "DISPATCHED", "occurredAt": "2024-01-15T14:00:00Z" }
    ]
  }
}
```

**Errors**: `403`, `404`

---

## Invoice Service (`/api/v1/invoices`)

### GET /api/v1/invoices/{orderId} — Authenticated
Get invoice metadata for an order.

**Response 200**
```json
{
  "data": {
    "invoiceId": "uuid",
    "orderId": "uuid",
    "customerId": "uuid",
    "amount": 99.98,
    "generatedAt": "2024-01-15T10:30:08Z",
    "downloadUrl": "https://s3.amazonaws.com/...?X-Amz-Expires=900&..."
  }
}
```

**Errors**: `403`, `404`

---

## Standard Error Response

All error responses follow this shape:

```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order with id 'abc-123' was not found.",
    "details": [],
    "requestId": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

For validation errors, `details` contains field-level errors:
```json
"details": [
  { "field": "lineItems", "message": "must not be empty" },
  { "field": "lineItems[0].quantity", "message": "must be greater than 0" }
]
```
