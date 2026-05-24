# API Standards

## General Rules

- All APIs are RESTful and versioned under `/api/v{n}/`.
- Base path per service is `/api/v1/{resource}` (e.g., `/api/v1/orders`).
- All request and response bodies are `application/json`.
- All timestamps are ISO 8601 UTC strings: `"2024-01-15T10:30:00Z"`.
- All IDs are UUIDs (type `string`, format `uuid`).
- Endpoints are lowercase, hyphen-separated: `/order-items`, not `/orderItems` or `/order_items`.

## HTTP Methods

| Operation | Method | Path |
|---|---|---|
| List (paginated) | GET | `/api/v1/orders` |
| Get by ID | GET | `/api/v1/orders/{id}` |
| Create | POST | `/api/v1/orders` |
| Full replace | PUT | `/api/v1/orders/{id}` |
| Partial update | PATCH | `/api/v1/orders/{id}` |
| Delete | DELETE | `/api/v1/orders/{id}` |
| Custom action | POST | `/api/v1/orders/{id}/cancel` |

- Use `POST` for actions that change state (e.g., `/cancel`, `/confirm`, `/refund`). Never use `GET` for state-changing operations.

## Request Validation

- All request DTOs must be Java `record` types annotated with Bean Validation constraints.
- Use `@Valid` on all `@RequestBody` parameters.
- Validation failures return `400 Bad Request` with the standard error envelope (see below).
- Required fields must use `@NotNull` or `@NotBlank`. Never rely on implicit null checks in service code.

## Standard Response Envelope

### Success (single resource)
```json
{
  "data": { ... },
  "meta": {
    "requestId": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Success (collection / paginated)
```json
{
  "data": [ ... ],
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
  },
  "meta": {
    "requestId": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Error
```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order with id 'abc-123' was not found.",
    "details": [ ],
    "requestId": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

- `code` is a SCREAMING_SNAKE_CASE string unique to the error type.
- `details` is an array of field-level validation errors: `{ "field": "quantity", "message": "must be > 0" }`.

## HTTP Status Codes

| Scenario | Code |
|---|---|
| Successful GET / PATCH / PUT | 200 |
| Successful POST (created) | 201 with `Location` header |
| Successful DELETE | 204 |
| Validation error | 400 |
| Unauthenticated | 401 |
| Forbidden | 403 |
| Not found | 404 |
| Conflict (duplicate, stale version) | 409 |
| Unprocessable business rule | 422 |
| Internal error | 500 |
| Downstream unavailable | 503 |

## Pagination

- All list endpoints support `page` (0-indexed) and `size` (default 20, max 100) query parameters.
- Sorting: `sort=fieldName,asc` or `sort=fieldName,desc`. Multiple sort params allowed.
- Use Spring Data's `Pageable` resolved from `@PageableDefault`.

## Error Handling

- A single `@RestControllerAdvice` class per service handles all exceptions.
- Map domain exceptions to HTTP status codes in the advice class, not in controllers.
- Never expose stack traces or internal class names in error responses.
- Log all 5xx errors at `ERROR` level with full stack trace and `requestId`.
- Log 4xx errors at `WARN` level without stack trace.

## OpenAPI Documentation

- Every controller class must have `@Tag(name = "...", description = "...")`.
- Every endpoint must have `@Operation(summary = "...", description = "...")`.
- Every endpoint must declare all possible `@ApiResponse` codes.
- Request/response schemas must use `@Schema` annotations on DTO fields with `description` and `example` values.
- The OpenAPI spec is available at `/api-docs` and Swagger UI at `/swagger-ui.html` in non-production profiles.

## Security

- All endpoints require a valid JWT Bearer token unless explicitly whitelisted.
- The `Authorization: Bearer {token}` header is the only accepted auth mechanism.
- Do not accept API keys or Basic auth on service endpoints (gateway handles pre-auth).
- Include `X-Request-ID` header propagation: read from incoming request, generate if absent, include in all responses and downstream calls.

## Idempotency

- `POST` endpoints that create resources must support an `Idempotency-Key` header.
- If the same key is received within 24 hours, return the original response with `200` instead of `201`.
- Store idempotency keys in the service's own database.
