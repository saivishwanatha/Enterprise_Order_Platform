# Phase 1 Implementation Tickets

## Overview

Phase 1 delivers a working, authenticated API surface for four foundational services: `auth-service`, `gateway-service`, `product-service`, and `order-service`. By the end of Phase 1 a user can register, log in, browse products, and place an order persisted in `PENDING` status — all requests flowing through the gateway with JWT validation enforced.

Tickets are ordered by dependency within each group. Every ticket is independently completable and testable.

---

## Ticket Groups

| Prefix | Group |
|---|---|
| [INFRA](#infra) | Shared libraries, Maven build, Docker, local infrastructure |
| [AUTH](#auth) | `auth-service` implementation |
| [GATEWAY](#gateway) | `gateway-service` implementation |
| [PRODUCT](#product) | `product-service` implementation |
| [ORDER](#order) | `order-service` implementation |

### Recommended Build Order

```
INFRA-001 → INFRA-002 → INFRA-003 → INFRA-004 → INFRA-005
     ↓
AUTH-001 → AUTH-002 → AUTH-003 → AUTH-004 → AUTH-005
     ↓
GATEWAY-001 → GATEWAY-002 → GATEWAY-003 → GATEWAY-004
     ↓
PRODUCT-001 → PRODUCT-002 → PRODUCT-003 → PRODUCT-004
     ↓
ORDER-001 → ORDER-002 → ORDER-003 → ORDER-004
     ↓
INFRA-006 (docker-compose + smoke test)
```

---

## INFRA

---

### INFRA-001 — Root Maven POM and Build Configuration

**Goal**
Establish the root Maven multi-module POM that all services inherit from, with dependency management, code quality enforcement, and coverage gates wired in before any service code is written.

**Scope**
- Root `pom.xml` with `<dependencyManagement>` pinning exact versions for: Spring Boot 3.x, Spring Cloud Gateway, Resilience4j, AWS SDK v2, Testcontainers, JUnit 5, Mockito, AssertJ, Awaitility, MapStruct, Flyway, SpringDoc OpenAPI 3, WireMock, Logback
- Java source/target compatibility: `21`
- `spring.threads.virtual.enabled=true` default property
- Checkstyle plugin with Google style; build fails on any violation
- Jacoco plugin with 80% line coverage threshold scoped to `service/` and `domain/` packages
- `spring-boot-maven-plugin` for executable JARs; no WAR packaging
- Compiler argument `-Xlint:all`
- Module declarations for: `shared-libraries/common-api`, `shared-libraries/common-events`, `shared-libraries/common-security`, `shared-libraries/common-observability`, `auth-service`, `gateway-service`, `product-service`, `order-service`
- No `SNAPSHOT` dependencies permitted

**Acceptance Criteria**
1. `mvn verify` runs from the root without errors against an empty project (no production source)
2. Introducing a deliberate Checkstyle violation in any module causes `mvn verify` to fail with a Checkstyle error
3. Dropping test coverage below 80% on a `service/` class causes `mvn verify` to fail with a Jacoco threshold error
4. All dependency versions are pinned — no open ranges (`+`, `LATEST`, `RELEASE`)
5. Each declared module resolves without error
6. Running `mvn dependency:tree` shows no duplicate dependency versions across modules

**Testing Requirement**
No application tests at this stage. Validate tooling by: (a) adding a trivial class with a deliberate Checkstyle violation, confirming build failure, then removing it; (b) adding an untested service class and confirming Jacoco fails below threshold, then removing it.

**Definition of Done**
- `mvn verify` passes from the root with all modules declared
- Checkstyle and Jacoco enforcement verified to fail correctly on synthetic violations
- No `SNAPSHOT` or open-range dependencies present

---

### INFRA-002 — common-api Shared Library

**Goal**
Build the shared response envelope and error model records used by every service so that all APIs return a consistent JSON shape.

**Scope**
- `ApiResponse<T>` record: `data`, `meta` (`requestId` UUID, `timestamp` ISO-8601 UTC)
- `PagedResponse<T>` record: `data`, `pagination` (`page`, `size`, `totalElements`, `totalPages`), `meta`
- `ErrorResponse` record: top-level `error` object containing `code`, `message`, `details`, `requestId`, `timestamp`
- `ErrorDetail` record: `field`, `message` — used for Bean Validation field errors
- Jackson config: ISO 8601 UTC timestamps, `null` fields excluded from output
- No Spring beans — pure Java library

**Acceptance Criteria**
1. `ApiResponse`, `PagedResponse`, and `ErrorResponse` serialise to the exact JSON shapes defined in `docs/11-phase-1-technical-design.md` Section 9
2. All three response types deserialise correctly from their own serialised JSON (round-trip)
3. `ErrorResponse.details` accepts a list of `ErrorDetail` and serialises them as an array
4. `null` fields are absent from JSON output — not rendered as `"field": null`
5. Module declares no Spring dependency
6. Unit test coverage ≥ 80% on all classes

**Testing Requirement**
JUnit 5 + AssertJ unit tests: serialisation round-trip for each record type; verify `null` field exclusion; verify `details` array shape in `ErrorResponse`; verify `pagination` block in `PagedResponse`.

**Definition of Done**
- All unit tests pass; `mvn verify` passes with coverage threshold met
- Module is importable by other services via an explicit `pom.xml` dependency declaration

---

### INFRA-003 — common-events Shared Library

**Goal**
Build the `EventEnvelope` wrapper and Phase 1 domain event payload POJOs so every outbox write produces a consistently structured event ready for Phase 2 SNS publishing.

**Scope**
- `EventEnvelope` record: `eventId` (UUID), `eventType` (String), `aggregateId` (UUID), `aggregateType` (String), `occurredAt` (Instant), `correlationId` (UUID), `causationId` (UUID, nullable), `serviceSource` (String), `schemaVersion` (int), `payload` (Object)
- Phase 1 payload POJOs: `UserRegisteredPayload`, `ProductCreatedPayload`, `OrderPlacedPayload`
- All payload classes annotated with `@JsonIgnoreProperties(ignoreUnknown = true)`
- `eventType` values follow `{aggregate}.{past-tense-verb}.v{n}` convention: `user.registered.v1`, `product.created.v1`, `order.placed.v1`
- No Spring beans — pure Java library

**Acceptance Criteria**
1. `EventEnvelope` serialises to the JSON shape defined in `docs/event-standards.md`
2. `@JsonIgnoreProperties(ignoreUnknown = true)` is present on all payload classes — deserialising JSON with extra unknown fields does not throw
3. All payload POJOs are immutable (Java records)
4. `eventType` strings for all three payloads match the naming convention exactly
5. Unit test coverage ≥ 80%

**Testing Requirement**
JUnit 5 + AssertJ: serialisation round-trip for `EventEnvelope` wrapping each payload type; forward-compatibility test — deserialise JSON with an extra unknown field and assert no exception is thrown.

**Definition of Done**
- All unit tests pass; `mvn verify` passes
- Module importable by services via explicit `pom.xml` dependency

---

### INFRA-004 — common-security Shared Library

**Goal**
Build the reusable JWT validation filter and `SecurityContextHelper` so all downstream services share identical token-parsing logic without duplicating it.

**Scope**
- `JwtValidationFilter` — Spring Security `OncePerRequestFilter`; validates RS256 JWT from `Authorization: Bearer` header; populates `SecurityContext` with `userId` (UUID), `email` (String), `role` (String) extracted from claims; returns `401` with `UNAUTHORIZED` error code on invalid, expired, or malformed token
- `SecurityContextHelper` — helper to extract `userId`, `email`, `role` from the current `SecurityContext`; throws `IllegalStateException` if called outside a security context
- `JwtProperties` — `@ConfigurationProperties(prefix = "jwt")` record: `publicKey` (PEM string)
- May contain Spring beans (`@Configuration`)

**Acceptance Criteria**
1. A valid RS256 JWT produced with a matching private key populates `SecurityContext` with correct `userId`, `email`, and `role`
2. An expired JWT causes the filter to write `401` and halt the filter chain — the request does not reach the controller
3. A JWT with an invalid signature causes `401`
4. A request with no `Authorization` header causes `401` on a protected endpoint
5. `SecurityContextHelper` correctly extracts each of the three claims after a valid token is processed
6. `SecurityContextHelper` throws `IllegalStateException` when no authentication is present in the context
7. Unit test coverage ≥ 80% on filter and helper logic

**Testing Requirement**
JUnit 5 + `MockitoExtension`: generate a test RSA 2048-bit key pair in `@BeforeAll`; test all JWT validation paths using that pair. Use `MockHttpServletRequest`/`MockHttpServletResponse` and `MockFilterChain`. Test `SecurityContextHelper` extraction for each claim and the exception path.

**Definition of Done**
- All unit tests pass; `mvn verify` passes
- Module importable by auth-service and all downstream services

---

### INFRA-005 — common-observability Shared Library

**Goal**
Build the MDC filter, structured JSON logging config, and Micrometer setup so every service emits consistent, correlated log lines and metrics from day one.

**Scope**
- `MdcFilter` — `OncePerRequestFilter`; reads `X-Request-ID` header and writes `correlationId` to MDC; generates a new UUID if the header is absent; writes `serviceId` (read from `spring.application.name`); clears all MDC keys after the response is committed
- `logback-spring.xml` — JSON encoder producing lines with fields: `timestamp`, `level`, `serviceId`, `traceId`, `spanId`, `correlationId`, `message`, `logger`
- Micrometer config bean: Prometheus registry for `local` and `test` profiles; CloudWatch registry wired (but disabled) for `prod` profile
- Spring Boot Actuator: `health`, `info`, `metrics`, `prometheus` exposed on management port `8081` only — not on `8080`
- May contain Spring beans

**Acceptance Criteria**
1. Every log line produced by a service running the filter includes `correlationId`, `serviceId`, `level`, `message`, and `timestamp` in JSON format
2. MDC keys are absent after the request completes — no leakage between sequential requests in the same thread
3. `X-Request-ID` from the incoming request is used as `correlationId`; a new UUID is generated when the header is absent
4. Actuator endpoints respond on port `8081`; a request to `http://localhost:8080/actuator/health` returns `404`
5. Unit test coverage ≥ 80% on `MdcFilter`

**Testing Requirement**
JUnit 5 unit tests for `MdcFilter`: use `MockHttpServletRequest`/`MockHttpServletResponse`; assert MDC populated from header; assert UUID generated when header absent; assert MDC cleared after `doFilterInternal` completes (including when downstream filter throws).

**Definition of Done**
- All unit tests pass; `mvn verify` passes
- Module importable by all services

---

### INFRA-006 — Docker Compose and Local Infrastructure

**Goal**
Provide a single `docker-compose up --build` command that starts all Phase 1 databases, LocalStack, and services with correct networking, health checks, and environment variable injection.

**Scope**
- `docker-compose.yml`: `auth-db` (PostgreSQL 15, port 5432), `product-db` (port 5433), `order-db` (port 5434), LocalStack pinned version (port 4566), `auth-service` (8081), `gateway-service` (8080), `product-service` (8082), `order-service` (8083)
- `depends_on` with `condition: service_healthy` so services wait for their DB before starting
- PostgreSQL health check: `pg_isready`; service health check: `GET /actuator/health` returns `200`
- Environment variables for all services sourced from `.env` file (JWT keys, datasource URLs, inter-service URLs)
- `.env.example` committed with placeholder values for every required variable; `.env` gitignored
- `infra/localstack/init.sh`: creates all Phase 2 SNS topics (`{env}-order-events`, `{env}-payment-events`, etc.) and SQS queues with DLQs so Phase 2 requires no compose changes
- LocalStack version pinned — no `latest` tag
- `README.md` updated with: local setup steps, RSA key generation command, port reference table, environment variable table

**Acceptance Criteria**
1. `docker-compose up --build` starts all containers and all service health checks return `UP` within 60 seconds
2. Each service connects exclusively to its own dedicated PostgreSQL instance — verified by checking `spring.datasource.url` in each service's `application-local.yml`
3. LocalStack starts and `init.sh` creates all required SNS/SQS resources without errors (check LocalStack init logs)
4. Removing the `.env` file and running `docker-compose up` fails with a clear error indicating missing required variables
5. `docker-compose down -v` cleanly removes all containers and volumes with no errors
6. `.env.example` contains every variable name referenced in `docker-compose.yml`

**Testing Requirement**
Manual smoke test documented in `README.md`: `docker-compose up --build` → poll all `/actuator/health` endpoints until `UP` → execute the Phase 1 end-to-end flow (register → login → create product → list products → create order → get order → cancel order) via `curl` or Swagger UI → `docker-compose down -v`.

**Definition of Done**
- All containers start and all health checks pass
- LocalStack `init.sh` runs without errors
- `README.md` contains complete local setup instructions
- `.env.example` is committed with all required variable placeholders
- `.env` is present in `.gitignore`

---

## AUTH

---

### AUTH-001 — auth-service Module Scaffold and Flyway Migrations

**Goal**
Bootstrap the `auth-service` Maven module with the correct package structure and create all database tables via Flyway so subsequent tickets have a stable schema to build on.

**Scope**
- `auth-service/pom.xml` inheriting root POM; explicit dependency declarations on `common-api`, `common-events`, `common-security`, `common-observability`
- `AuthServiceApplication.java` in `com.enterprise.authservice`
- Package structure: `config/`, `controller/`, `service/`, `domain/`, `repository/`, `dto/`, `mapper/`, `event/`, `outbox/`, `exception/`
- Flyway migrations:
  - `V1__create_auth_users.sql` — `auth_users` table per Section 5.1 of the technical design
  - `V2__create_auth_refresh_tokens.sql` — `auth_refresh_tokens` table
  - `V3__create_auth_outbox.sql` — `auth_outbox` table with `status` values `PENDING` / `PUBLISHED`
- `application.yml`, `application-local.yml`, `application-test.yml`
- `AuthProperties` and `JwtProperties` `@ConfigurationProperties` records
- Default server port `8081`, management port `8081` (separate)

**Acceptance Criteria**
1. `mvn package -pl auth-service` produces an executable JAR without errors
2. All three Flyway migrations run in order without errors against a Testcontainers PostgreSQL 15 instance
3. `auth_users` has columns: `id` (UUID PK), `email` (UNIQUE NOT NULL), `password_hash`, `first_name`, `last_name`, `role`, `active` (DEFAULT true), `version` (BIGINT DEFAULT 0), `created_at`, `updated_at` — all with correct types
4. `auth_refresh_tokens` has: `id`, `user_id`, `token_hash` (UNIQUE NOT NULL), `expires_at`, `revoked` (DEFAULT false), `created_at`
5. `auth_outbox` has: `id`, `event_type`, `aggregate_id`, `payload` (JSONB), `correlation_id`, `status`, `created_at`, `published_at` (nullable)
6. Package structure matches `structure.md` exactly — no extra packages, no missing packages

**Testing Requirement**
`@DataJpaTest` with Testcontainers PostgreSQL: assert all three tables exist after Flyway runs; assert `email` column has a unique constraint (insert duplicate, expect `DataIntegrityViolationException`); assert `version` column exists on `auth_users`; assert `payload` column type is JSONB.

**Definition of Done**
- `@DataJpaTest` migration tests pass
- Module builds and packages cleanly
- Package structure matches the standard

---

### AUTH-002 — JPA Entities, Repositories, and JwtService

**Goal**
Build the domain layer — JPA entities, repositories, and the JWT utility class — so the service layer in AUTH-003 has everything it needs without mixing concerns.

**Scope**
- `User` entity: maps `auth_users`; `@Version` on `version` field; `role` as enum `UserRole { CUSTOMER, ADMIN }`
- `RefreshToken` entity: maps `auth_refresh_tokens`; no `@Version` (append-only)
- `OutboxEvent` entity: maps `auth_outbox`; `status` as enum `OutboxStatus { PENDING, PUBLISHED }`
- `UserRepository`: `findByEmail(String email) → Optional<User>`
- `RefreshTokenRepository`: `findByTokenHash(String hash) → Optional<RefreshToken>`
- `OutboxRepository`: standard `JpaRepository` save
- `JwtService`:
  - `generateAccessToken(User user) → String` — RS256, 900 s expiry, claims: `sub`, `email`, `role`, `iat`, `exp`
  - `generateRefreshToken() → String` — 256-bit secure random opaque token
  - `hashToken(String token) → String` — SHA-256 hex
  - `validateAccessToken(String token) → Claims` — throws on invalid/expired
  - Private key loaded from `JwtProperties.privateKey` (PEM PKCS#8)

**Acceptance Criteria**
1. `User` entity persists and retrieves correctly; `version` increments on update
2. `UserRepository.findByEmail` returns the correct user or `Optional.empty()`
3. `RefreshTokenRepository.findByTokenHash` returns the correct token or `Optional.empty()`
4. `JwtService.generateAccessToken` produces a verifiable RS256 JWT containing `sub`, `email`, and `role` claims with `exp` set 900 s in the future
5. `JwtService.validateAccessToken` rejects a token signed with a different private key
6. `JwtService.validateAccessToken` rejects a token with `exp` in the past
7. `JwtService.hashToken` is deterministic — same input always produces the same hash
8. Unit test coverage ≥ 80% on `JwtService`

**Testing Requirement**
Unit tests (`MockitoExtension`) for `JwtService`: generate a test RSA key pair in `@BeforeAll`; test all paths listed in acceptance criteria 4–7. Repository tests: `@DataJpaTest` with Testcontainers asserting `findByEmail` and `findByTokenHash` return correct results and `Optional.empty()` on miss.

**Definition of Done**
- All unit and `@DataJpaTest` tests pass
- `mvn verify` passes with coverage threshold met

---

### AUTH-003 — AuthServiceImpl — Core Business Logic

**Goal**
Implement all auth business rules — registration, login, logout, token refresh, and profile retrieval — with full unit test coverage before the HTTP layer is wired.

**Scope**
- `AuthService` interface: `register`, `login`, `logout`, `refresh`, `getMe`
- `AuthServiceImpl` implementing the interface; constructor-injected dependencies only
- `register`: bcrypt hash (cost 12), duplicate email check, persist `User`, write `user.registered.v1` outbox row in the same `@Transactional` block, return `UserResponse`
- `login`: load user by email, verify bcrypt password, check `active` flag, issue access + refresh tokens, persist `RefreshToken` (store SHA-256 hash), return `LoginResponse`
- `logout`: find refresh token by hash, mark `revoked = true`, return success
- `refresh`: find token by hash, verify not revoked and not expired, issue new access token, return `AccessTokenResponse`
- `getMe`: load user by id from `SecurityContext`, return `UserResponse`
- All domain exceptions in `exception/` package: `EmailAlreadyRegisteredException`, `InvalidCredentialsException`, `TokenRevokedException`, `TokenExpiredException`, `UserNotFoundException`

**Acceptance Criteria**
1. `register` with a new email persists a `User` with a bcrypt-hashed password and writes one `PENDING` outbox row with `event_type = 'user.registered.v1'` in the same transaction
2. `register` with a duplicate email throws `EmailAlreadyRegisteredException` and writes no outbox row
3. `login` with correct credentials returns an access token and a refresh token
4. `login` with wrong password throws `InvalidCredentialsException`
5. `logout` marks the refresh token `revoked = true`; a subsequent `refresh` call with the same token throws `TokenRevokedException`
6. `refresh` with an expired token throws `TokenExpiredException`
7. `getMe` returns the correct user for the authenticated principal
8. Unit test coverage ≥ 80% on `AuthServiceImpl`

**Testing Requirement**
JUnit 5 + Mockito: mock `UserRepository`, `RefreshTokenRepository`, `OutboxRepository`, `JwtService`, `PasswordEncoder`. Test every acceptance criterion path as a `@Nested` class per method. Use `@DisplayName` on every test method. Use AssertJ `assertThatThrownBy` for exception assertions. No `Thread.sleep` — no async in this layer.

**Definition of Done**
- All unit tests pass
- `mvn verify` passes with coverage threshold met on `service/` package

---

### AUTH-004 — AuthController, SecurityConfig, and Rate Limiter

**Goal**
Expose all auth endpoints over HTTP, enforce Spring Security access rules, and apply Resilience4j rate limiting on the login endpoint.

**Scope**
- `AuthController`: `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/logout`, `POST /api/v1/auth/refresh`, `GET /api/v1/auth/me`, `GET /api/v1/auth/public-key`
- All endpoints annotated with `@Tag`, `@Operation`, `@ApiResponse` (SpringDoc)
- All `@RequestBody` parameters annotated with `@Valid`
- `GET /api/v1/auth/public-key` returns the RS256 public key as PEM string — Public endpoint
- `SecurityConfig`: whitelist `register`, `login`, `refresh`, `public-key`; all other endpoints require authentication; uses `JwtValidationFilter` from `common-security`
- `LoginRateLimiterConfig`: Resilience4j `RateLimiter` bean — 5 requests per 15-minute window per IP; applied as an `@Around` aspect or inline check in `AuthServiceImpl.login`
- `GlobalExceptionHandler` (`@RestControllerAdvice`): maps all domain exceptions to correct HTTP codes and `ErrorResponse` shape; never exposes stack traces; logs 5xx at `ERROR`, 4xx at `WARN`

**Acceptance Criteria**
1. `POST /api/v1/auth/register` with valid body returns `201` with correct `UserResponse` payload
2. `POST /api/v1/auth/register` with invalid body (missing `email`, weak password) returns `400` with `details` array containing field-level errors
3. `POST /api/v1/auth/register` with duplicate email returns `409` with `EMAIL_ALREADY_REGISTERED` code
4. `POST /api/v1/auth/login` with correct credentials returns `200` with `accessToken`, `refreshToken`, `expiresIn: 900`, `tokenType: "Bearer"`
5. `POST /api/v1/auth/login` with wrong credentials returns `401` with `INVALID_CREDENTIALS` code
6. 6th `POST /api/v1/auth/login` within 15 minutes from the same IP returns `429` with `RATE_LIMIT_EXCEEDED` code
7. `GET /api/v1/auth/me` without a JWT returns `401`
8. `GET /api/v1/auth/me` with a valid JWT returns `200` with the authenticated user's profile
9. `GET /api/v1/auth/public-key` returns `200` with PEM-encoded public key — no JWT required

**Testing Requirement**
`@WebMvcTest(AuthController.class)` with mocked `AuthService`: test all validation error cases (assert `400` + field details); test auth enforcement (assert `401` on missing token); test response shape for success cases. Separate `@SpringBootTest` + Testcontainers integration tests: rate limit enforcement (6 rapid login requests → assert 6th is `429`); outbox row written on successful registration.

**Definition of Done**
- All `@WebMvcTest` and integration tests pass
- `mvn verify` passes
- OpenAPI spec rendered at `http://localhost:8081/swagger-ui.html` in local profile

---

### AUTH-005 — auth-service Integration Test Suite

**Goal**
Cover all end-to-end auth scenarios against a real PostgreSQL database and verify the outbox write guarantee under the same transaction.

**Scope**
- `AbstractIntegrationTest` base class: static Testcontainers PostgreSQL 15 container; `@DynamicPropertySource` overrides datasource URL; all integration test classes extend this
- Integration test classes tagged `@Tag("integration")` in `src/test/java/.../integration/`
- Tests covering all scenarios from Section 13.2 of the technical design (auth-service block)
- `application-test.yml`: `spring.jpa.hibernate.ddl-auto: create-drop`, `spring.flyway.enabled: true`

**Acceptance Criteria**
1. `POST /register` happy path: assert `201`, correct response body, exactly one `auth_outbox` row with `status = 'PENDING'` and `event_type = 'user.registered.v1'`
2. `POST /register` duplicate email: assert `409`, zero outbox rows written in the duplicate attempt
3. `POST /register` validation errors: assert `400` with correct `details` array
4. `POST /login` success: assert `200`, JWT claims (`sub`, `email`, `role`) are correct, refresh token row exists in DB
5. `POST /login` wrong password: assert `401` with `INVALID_CREDENTIALS`
6. `POST /login` rate limit: send 6 requests in rapid succession from the same IP; assert 6th returns `429`
7. `POST /logout` followed by `POST /refresh` with the same token: assert `refresh` returns `401` with `TOKEN_REVOKED`
8. `POST /refresh` with expired token: assert `401` with `TOKEN_EXPIRED`
9. Flyway `@DataJpaTest`: all three tables present with correct columns after migration

**Testing Requirement**
JUnit 5 + `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate` + Testcontainers PostgreSQL. Each test method cleans up its own data (`@Transactional` or explicit delete in `@AfterEach`). No `Thread.sleep` — use `Awaitility` for any async assertions.

**Definition of Done**
- All integration tests pass in isolation and when run as a suite
- `mvn verify -pl auth-service -P integration` passes
- No `@Disabled` tests without a linked issue comment

---

## GATEWAY

---

### GATEWAY-001 — gateway-service Module Scaffold and Route Configuration

**Goal**
Bootstrap the `gateway-service` module and define all Phase 1 routing rules using `RouteLocator` beans so the gateway correctly forwards requests to the three downstream services.

**Scope**
- `gateway-service/pom.xml` inheriting root POM; Spring Cloud Gateway dependency; explicit dependency on `common-observability`
- `GatewayApplication.java` in `com.enterprise.gatewayservice`
- `RouteLocatorConfig` (`@Configuration`): defines routes via `RouteLocator` beans (not YAML) for:
  - `/api/v1/auth/**` → `auth-service:8081`
  - `/api/v1/products/**` → `product-service:8082`
  - `/api/v1/orders/**` → `order-service:8083`
- `GatewayProperties` `@ConfigurationProperties` record: `authServiceUrl`, `productServiceUrl`, `orderServiceUrl`
- `application.yml`, `application-local.yml`
- Default server port `8080`, management port `8081`

**Acceptance Criteria**
1. `mvn package -pl gateway-service` produces an executable JAR without errors
2. A request to `GET /api/v1/auth/me` is forwarded to `auth-service` (verified via WireMock stub returning `200`)
3. A request to `GET /api/v1/products` is forwarded to `product-service` (WireMock)
4. A request to `POST /api/v1/orders` is forwarded to `order-service` (WireMock)
5. A request to an unmapped path (e.g. `/api/v1/unknown`) returns `404`
6. Service URLs are read from `GatewayProperties` — no hardcoded hostnames in `RouteLocatorConfig`

**Testing Requirement**
Integration test with `@SpringBootTest(webEnvironment = RANDOM_PORT)` + WireMock stubs for each downstream service: assert that requests to each path prefix are forwarded to the correct stub; assert `404` for unmapped paths.

**Definition of Done**
- Route integration tests pass
- Module builds and packages cleanly
- No hardcoded service URLs in source code

---

### GATEWAY-002 — JWT Validation Filter

**Goal**
Implement the gateway-level JWT validation filter that rejects unauthenticated requests before they reach downstream services, keeping role-enforcement in the services themselves.

**Scope**
- `JwtAuthenticationFilter` — Spring Cloud Gateway `GlobalFilter` + `Ordered`
- Reads `Authorization: Bearer {token}` header; validates RS256 signature using public key from `JWT_PUBLIC_KEY` env var (PEM) loaded at startup via `GatewayProperties`
- On valid token: injects `X-User-Id` and `X-User-Role` headers into the forwarded request; passes `Authorization` header downstream unchanged
- On invalid or expired token: short-circuits with `401` and `UNAUTHORIZED` error response (JSON matching `ErrorResponse` shape from `common-api`)
- Whitelist (no JWT required): `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `GET /api/v1/auth/public-key`
- RS256 public key loaded once at startup — no runtime calls to auth-service

**Acceptance Criteria**
1. Request with a valid JWT is forwarded; downstream stub receives `X-User-Id` and `X-User-Role` headers
2. Request with an expired JWT returns `401` with `{"error": {"code": "UNAUTHORIZED", ...}}`
3. Request with an invalid signature returns `401`
4. Request with no `Authorization` header to a protected path returns `401`
5. `POST /api/v1/auth/login` with no JWT is forwarded without `401` (whitelisted)
6. `GET /api/v1/auth/public-key` with no JWT is forwarded (whitelisted)
7. Gateway does not inspect or enforce roles — `X-User-Role` is injected and passed downstream only

**Testing Requirement**
Integration tests with `@SpringBootTest` + WireMock: test all acceptance criteria paths. Use a test RSA key pair generated in `@BeforeAll`. Unit tests for the filter's token parsing and whitelist path logic using Mockito-mocked `ServerWebExchange`.

**Definition of Done**
- All unit and integration tests pass
- `mvn verify` passes

---

### GATEWAY-003 — X-Request-ID Filter and Rate Limiting

**Goal**
Ensure every forwarded request carries a correlation ID and that per-user and per-IP rate limits are enforced at the gateway edge.

**Scope**
- `RequestIdFilter` — `GlobalFilter`; reads `X-Request-ID` header; generates a UUID if absent; injects the value into the forwarded request headers and the response headers
- `RateLimitingFilter` (or Spring Cloud Gateway `RequestRateLimiter` filter with in-memory token bucket):
  - Authenticated requests (identified by `X-User-Id` header set by `JwtAuthenticationFilter`): 100 req/s per user
  - Unauthenticated requests: 10 req/s per source IP
  - Exceeding the limit returns `429` with `RATE_LIMIT_EXCEEDED` error code

**Acceptance Criteria**
1. A request arriving without `X-Request-ID` is forwarded with a newly generated UUID in the `X-Request-ID` header; the same UUID appears in the response `X-Request-ID` header
2. A request arriving with an existing `X-Request-ID` value has that exact value propagated — no regeneration
3. Sending 101 requests per second from the same authenticated user causes the 101st to return `429`
4. Sending 11 requests per second from the same unauthenticated IP causes the 11th to return `429`
5. Rate limit counters are independent per user/IP — one user hitting the limit does not affect another user

**Testing Requirement**
Integration tests with `@SpringBootTest` + WireMock: assert `X-Request-ID` propagation for both present and absent header cases; assert `429` after burst for authenticated and unauthenticated paths. Unit tests for the `RequestIdFilter` logic.

**Definition of Done**
- All tests pass
- `mvn verify` passes

---

### GATEWAY-004 — gateway-service Integration Test Suite

**Goal**
Validate the complete gateway behaviour end-to-end: JWT validation, routing, header injection, and rate limiting all working together.

**Scope**
- `AbstractGatewayIntegrationTest` base class with WireMock server started once per suite
- Integration tests tagged `@Tag("integration")`
- Test scenarios covering all items from Section 13.2 gateway-service block of the technical design

**Acceptance Criteria**
1. Request with no JWT to a protected path → `401`
2. Request with an expired JWT → `401`
3. Request with an invalid signature JWT → `401`
4. Request with a valid JWT to `/api/v1/products` → WireMock stub receives the request with `X-User-Id` and `X-User-Role` headers
5. Request to a whitelisted path (`/api/v1/auth/login`) with no JWT → forwarded, no `401`
6. `X-Request-ID` is injected when absent and propagated when present
7. Burst of 101 authenticated requests/s → 101st returns `429`

**Testing Requirement**
JUnit 5 + `@SpringBootTest(webEnvironment = RANDOM_PORT)` + WireMock. Test RSA key pair generated in `@BeforeAll`. Each test verifies the response code and, where relevant, the forwarded request headers captured by WireMock.

**Definition of Done**
- All integration tests pass in isolation and as a suite
- `mvn verify -pl gateway-service -P integration` passes

---

## PRODUCT

---

### PRODUCT-001 — product-service Module Scaffold and Flyway Migrations

**Goal**
Bootstrap the `product-service` module and create all database tables via Flyway.

**Scope**
- `product-service/pom.xml` inheriting root POM; explicit dependencies on `common-api`, `common-events`, `common-security`, `common-observability`
- `ProductServiceApplication.java` in `com.enterprise.productservice`
- Package structure: `config/`, `controller/`, `service/`, `domain/`, `repository/`, `dto/`, `mapper/`, `event/`, `outbox/`, `exception/`
- Flyway migrations:
  - `V1__create_prd_products.sql` — `prd_products` table per Section 5.2 of the technical design
  - `V2__create_prd_outbox.sql` — `prd_outbox` table (same structure as `auth_outbox`)
- `application.yml`, `application-local.yml`, `application-test.yml`
- `ProductServiceProperties` `@ConfigurationProperties` record (inventory service base URL for Phase 2)
- Default server port `8082`, management port `8081`

**Acceptance Criteria**
1. `mvn package -pl product-service` produces an executable JAR
2. Both Flyway migrations run without errors against Testcontainers PostgreSQL 15
3. `prd_products` has: `id` (UUID PK), `name` (VARCHAR 255 NOT NULL), `description` (TEXT nullable), `category` (VARCHAR 100 NOT NULL), `price` (NUMERIC(10,2) NOT NULL), `status` (VARCHAR 20 NOT NULL), `version` (BIGINT DEFAULT 0), `created_at`, `updated_at`
4. `prd_outbox` matches the `auth_outbox` structure exactly
5. Package structure matches `structure.md`

**Testing Requirement**
`@DataJpaTest` with Testcontainers: assert both tables exist with correct column definitions; assert `price` column type is `NUMERIC(10,2)`; assert `version` column exists.

**Definition of Done**
- `@DataJpaTest` migration tests pass
- Module builds and packages cleanly

---

### PRODUCT-002 — Product Domain, Repositories, and InventoryClient Stub

**Goal**
Build the product domain entity, repository with custom queries, and the inventory service client (circuit-breaker-wrapped, fallback fires immediately in Phase 1) so the service layer has stable collaborators.

**Scope**
- `Product` entity: maps `prd_products`; `status` as enum `ProductStatus { ACTIVE, INACTIVE }`; `@Version` on `version`
- `OutboxEvent` entity: maps `prd_outbox`
- `ProductRepository`:
  - `findByIdAndStatus(UUID id, ProductStatus status) → Optional<Product>`
  - Custom JPQL query: `findAllByFilters(ProductStatus status, String category, String namePrefix, Pageable pageable) → Page<Product>` — `category` and `namePrefix` are optional (nullable = no filter applied)
- `InventoryClient`: Resilience4j circuit-breaker-wrapped `RestClient` call to `GET /api/v1/inventory/stock/{productId}`; fallback returns `Optional.empty()` immediately; in Phase 1 the base URL points to a non-existent host — the circuit breaker opens and the fallback fires on every call; base URL from `ProductServiceProperties`

**Acceptance Criteria**
1. `Product` entity persists and retrieves; `version` increments on update; optimistic locking throws `ObjectOptimisticLockingFailureException` on concurrent update
2. `ProductRepository.findAllByFilters` with `status = ACTIVE` returns only active products
3. `findAllByFilters` with `category = "ELECTRONICS"` returns only products in that category
4. `findAllByFilters` with `namePrefix = "wire"` returns products whose name starts with `wire` (case-insensitive)
5. `InventoryClient` always returns `Optional.empty()` in Phase 1 — circuit breaker fallback fires without throwing an unhandled exception
6. Unit test coverage ≥ 80% on `InventoryClient` fallback logic

**Testing Requirement**
`@DataJpaTest` with Testcontainers: test all `ProductRepository` query scenarios including pagination (`page`, `size` respected) and combined filters. Unit tests for `InventoryClient` fallback using Mockito-mocked `RestClient` that throws `ConnectException` — assert fallback returns `Optional.empty()`.

**Definition of Done**
- All `@DataJpaTest` and unit tests pass
- `mvn verify` passes

---

### PRODUCT-003 — ProductServiceImpl and ProductController

**Goal**
Implement all product business logic and expose it over HTTP with correct role enforcement and OpenAPI documentation.

**Scope**
- `ProductService` interface: `createProduct`, `updateProduct`, `deactivateProduct`, `listProducts`, `getProductById`
- `ProductServiceImpl`: constructor-injected; all methods `@Transactional` where they write; outbox write on `createProduct` in same transaction; `availableStock` always `null` (InventoryClient fallback); deactivate is idempotent
- `ProductController`: all five endpoints from Section 6.3; `@PreAuthorize("hasRole('ADMIN')")` on `POST`, `PATCH`, `POST /{id}/deactivate`; `@PageableDefault(size = 20)` on list endpoint
- `ProductMapper` (MapStruct or manual): `Product` → `ProductResponse`; `CreateProductRequest` → `Product`
- `GlobalExceptionHandler`: maps `ProductNotFoundException` → `404`, `AccessDeniedException` → `403`; follows `ErrorResponse` shape
- Domain exceptions: `ProductNotFoundException`
- Request DTOs: `CreateProductRequest`, `UpdateProductRequest` (all fields optional, at least one required — custom validator)

**Acceptance Criteria**
1. `POST /api/v1/products` by an ADMIN returns `201` with `Location` header and correct response body; one `prd_outbox` row with `event_type = 'product.created.v1'` written in the same transaction
2. `POST /api/v1/products` by a CUSTOMER returns `403`
3. `POST /api/v1/products` with `price = 0` returns `400` with field-level error for `price`
4. `GET /api/v1/products` returns paginated results; `availableStock` is `null` and `meta.stockDataAvailable` is `false` in every response
5. `GET /api/v1/products?category=ELECTRONICS` returns only ELECTRONICS products
6. `GET /api/v1/products?search=wire` returns only products whose name starts with `wire` (case-insensitive)
7. `GET /api/v1/products` excludes `INACTIVE` products from results
8. `PATCH /api/v1/products/{id}` by ADMIN updates only the supplied fields; returns `200`
9. `POST /api/v1/products/{id}/deactivate` by ADMIN sets `status = INACTIVE`; calling it again on an already-inactive product returns `200` (idempotent)
10. `GET /api/v1/products/{id}` for a non-existent product returns `404` with `PRODUCT_NOT_FOUND`
11. Unit test coverage ≥ 80% on `ProductServiceImpl`

**Testing Requirement**
Unit tests (Mockito): all `ProductServiceImpl` paths including outbox write, idempotent deactivate, and inactive product retrieval. `@WebMvcTest(ProductController.class)`: all validation error cases, role enforcement (`401`/`403`), and response shape for each endpoint.

**Definition of Done**
- All unit and `@WebMvcTest` tests pass
- `mvn verify` passes with coverage threshold met

---

### PRODUCT-004 — product-service Integration Test Suite

**Goal**
Cover all product-service end-to-end scenarios against a real database including the outbox write guarantee.

**Scope**
- `AbstractIntegrationTest` base class with static Testcontainers PostgreSQL 15 container
- Integration tests tagged `@Tag("integration")` in `src/test/java/.../integration/`
- All scenarios from Section 13.2 product-service block of the technical design
- `application-test.yml`: Flyway enabled, `ddl-auto: create-drop`

**Acceptance Criteria**
1. ADMIN creates product → `201`, product row in DB, exactly one `prd_outbox` row with `event_type = 'product.created.v1'` and `status = 'PENDING'`
2. CUSTOMER creates product → `403`, no DB rows written
3. `GET /products` pagination: request page 0 size 2 from 5 products → `totalElements: 5`, `totalPages: 3`, 2 items returned
4. `GET /products?category=ELECTRONICS` → only ELECTRONICS products in `data`
5. `GET /products?search=wire` → case-insensitive prefix match
6. `GET /products` → `INACTIVE` products absent from results
7. `POST /products/{id}/deactivate` → product `status = 'INACTIVE'` in DB; calling again → `200`, no duplicate outbox row
8. `PATCH /products/{id}` price update → new price persisted; `version` incremented
9. Flyway `@DataJpaTest` → both tables present with correct schema

**Testing Requirement**
JUnit 5 + `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate` + Testcontainers. Each test cleans up its data in `@AfterEach` or via `@Transactional` rollback. JWT tokens for ADMIN and CUSTOMER roles are generated using a test RSA key pair in `@BeforeAll`.

**Definition of Done**
- All integration tests pass
- `mvn verify -pl product-service -P integration` passes

---

## ORDER

---

### ORDER-001 — order-service Module Scaffold and Flyway Migrations

**Goal**
Bootstrap the `order-service` module and create all four order database tables via Flyway.

**Scope**
- `order-service/pom.xml` inheriting root POM; explicit dependencies on `common-api`, `common-events`, `common-security`, `common-observability`
- `OrderServiceApplication.java` in `com.enterprise.orderservice`
- Package structure: `config/`, `controller/`, `service/`, `domain/`, `repository/`, `dto/`, `mapper/`, `event/`, `outbox/`, `exception/`
- Flyway migrations:
  - `V1__create_ord_orders.sql` — `ord_orders` table per Section 5.3 of the technical design
  - `V2__create_ord_order_items.sql` — `ord_order_items` table
  - `V3__create_ord_order_timeline.sql` — `ord_order_timeline` table
  - `V4__create_ord_outbox.sql` — `ord_outbox` table
- `application.yml`, `application-local.yml`, `application-test.yml`
- `OrderServiceProperties` `@ConfigurationProperties` record: `productServiceUrl`
- Default server port `8083`, management port `8081`

**Acceptance Criteria**
1. `mvn package -pl order-service` produces an executable JAR
2. All four Flyway migrations run in order without errors against Testcontainers PostgreSQL 15
3. `ord_orders` has: `id`, `customer_id`, `status` (VARCHAR 30), `total_amount` (NUMERIC(10,2)), `retry_count` (INT DEFAULT 0), `idempotency_key` (UUID UNIQUE nullable), `version` (BIGINT DEFAULT 0), `created_at`, `updated_at`
4. `ord_order_items` has: `id`, `order_id` (FK → `ord_orders`), `product_id`, `product_name`, `quantity` (INT), `unit_price` (NUMERIC(10,2)) — no `version` column on this table
5. `ord_order_timeline` has: `id`, `order_id` (FK → `ord_orders`), `event_type`, `description`, `metadata` (JSONB nullable), `occurred_at`
6. `ord_outbox` matches the `auth_outbox` structure exactly

**Testing Requirement**
`@DataJpaTest` with Testcontainers: assert all four tables exist; assert `idempotency_key` has a unique constraint (insert duplicate, expect `DataIntegrityViolationException`); assert `order_id` FK on `ord_order_items` exists.

**Definition of Done**
- `@DataJpaTest` migration tests pass
- Module builds and packages cleanly

---

### ORDER-002 — Order Domain, Repositories, and ProductClient

**Goal**
Build the order domain entities, repositories with custom queries, and the circuit-breaker-wrapped product-service client so the service layer has all collaborators available before business logic is written.

**Scope**
- `Order` entity: maps `ord_orders`; `status` as sealed interface or enum `OrderStatus { PENDING, CANCELLED }`; `@Version` on `version`
- `OrderItem` entity: maps `ord_order_items`; `@ManyToOne` to `Order`; no `@Version`
- `OrderTimeline` entity: maps `ord_order_timeline`; `@ManyToOne` to `Order`
- `OutboxEvent` entity: maps `ord_outbox`
- `OrderRepository`:
  - `findByIdAndCustomerId(UUID id, UUID customerId) → Optional<Order>`
  - `findByCustomerId(UUID customerId, Pageable pageable) → Page<Order>`
  - `findByIdempotencyKey(UUID key) → Optional<Order>`
- `ProductClient`:
  - `getProduct(UUID productId) → ProductSnapshot` — calls `GET /api/v1/products/{productId}` on product-service
  - Resilience4j circuit breaker (`productServiceCircuitBreaker`): 50% failure rate over 10-call sliding window, 30 s open state wait
  - Retry: 3 attempts, exponential backoff 500 ms, jitter enabled
  - Timeouts: connect 2 s, read 5 s
  - Fallback: throws `ProductServiceUnavailableException`
  - If product returns `404` or `status = INACTIVE`: throws `ProductNotFoundException` or `ProductInactiveException` respectively
- `ProductSnapshot` record: `productId`, `name`, `price`, `status`

**Acceptance Criteria**
1. `Order` entity persists with `PENDING` status; optimistic locking throws on concurrent update
2. `OrderRepository.findByCustomerId` returns only orders belonging to that customer, paginated
3. `OrderRepository.findByIdempotencyKey` returns the correct order or `Optional.empty()`
4. `ProductClient` with a successful WireMock stub returns a populated `ProductSnapshot`
5. `ProductClient` with a WireMock stub returning `404` throws `ProductNotFoundException`
6. `ProductClient` with WireMock returning `503` on all retries throws `ProductServiceUnavailableException` after 3 attempts
7. Unit test coverage ≥ 80% on `ProductClient`

**Testing Requirement**
`@DataJpaTest` with Testcontainers: test all `OrderRepository` queries. Unit tests for `ProductClient`: use WireMock (or Mockito-mocked `RestClient`) to simulate success, `404`, inactive product, and repeated `503` (verify retry count = 3 before fallback fires).

**Definition of Done**
- All `@DataJpaTest` and unit tests pass
- `mvn verify` passes

---

### ORDER-003 — OrderServiceImpl and OrderController

**Goal**
Implement all order business rules — create, list, get, timeline, cancel — and expose them over HTTP with idempotency, ownership checks, and full OpenAPI documentation.

**Scope**
- `OrderService` interface: `createOrder`, `listOrders`, `getOrder`, `getOrderTimeline`, `cancelOrder`
- `OrderServiceImpl`:
  - `createOrder`: check idempotency key (return existing if found); resolve each `productId` via `ProductClient`; reject duplicate `productId` entries in the same request (`422`); calculate `totalAmount`; persist `Order` + `OrderItems`; write `order.placed.v1` outbox row; write `ORDER_PLACED` timeline entry — all in one `@Transactional` block
  - `listOrders`: customers see only their own orders; admins see all — driven by role from `SecurityContext`
  - `getOrder`: ownership check — customers can only access their own orders
  - `getOrderTimeline`: ownership check; return timeline events in ascending `occurred_at` order
  - `cancelOrder`: only `PENDING` orders; idempotent if already `CANCELLED`; write `ORDER_CANCELLED` timeline entry; write `ord_outbox` row for `order.cancelled.v1` (Phase 2 will consume it)
- `OrderController`: all five endpoints from Section 6.4; `Idempotency-Key` header support on `POST /orders`
- `TotalAmountCalculator`: calculates `sum(quantity * unitPrice)` rounded to 2 decimal places
- `OrderMapper` (MapStruct or manual): `Order` + `OrderItems` → `OrderResponse`; calculates `subtotal` per line item
- Domain exceptions: `OrderNotFoundException`, `OrderNotCancellableException`, `ProductInactiveException`, `ProductServiceUnavailableException`, `ForbiddenException`
- `GlobalExceptionHandler`: maps each exception to correct HTTP code + `ErrorResponse`

**Acceptance Criteria**
1. `POST /api/v1/orders` with valid line items returns `201` with `Location` header; order has `status = PENDING`; one `ord_outbox` row (`order.placed.v1`); one `ord_order_timeline` row (`ORDER_PLACED`)
2. `POST /api/v1/orders` with the same `Idempotency-Key` within 24 hours returns `200` with the original order — no new DB rows
3. `POST /api/v1/orders` with a duplicate `productId` in `lineItems` returns `422` with `DUPLICATE_LINE_ITEMS`
4. `POST /api/v1/orders` when product-service returns `503` returns `503` with `PRODUCT_SERVICE_UNAVAILABLE`
5. `POST /api/v1/orders` when product is `INACTIVE` returns `422` with `PRODUCT_INACTIVE`
6. `GET /api/v1/orders` as a CUSTOMER returns only that customer's orders
7. `GET /api/v1/orders` as an ADMIN returns orders from all customers
8. `GET /api/v1/orders/{id}` by a customer for another customer's order returns `403`
9. `POST /api/v1/orders/{id}/cancel` on a `PENDING` order returns `200` with `status = CANCELLED`; `ORDER_CANCELLED` timeline entry written
10. `POST /api/v1/orders/{id}/cancel` on an already-`CANCELLED` order returns `200` (idempotent)
11. `POST /api/v1/orders/{id}/cancel` on a non-`PENDING`, non-`CANCELLED` order returns `409` with `ORDER_NOT_CANCELLABLE`
12. `TotalAmountCalculator` correctly rounds to 2 decimal places
13. Unit test coverage ≥ 80% on `OrderServiceImpl` and `TotalAmountCalculator`

**Testing Requirement**
Unit tests (Mockito): all `OrderServiceImpl` paths — mock `ProductClient`, `OrderRepository`, `OutboxRepository`. Separate `@Nested` classes per method. `@WebMvcTest(OrderController.class)`: all validation error cases, auth enforcement, idempotency header handling, response shape. Unit tests for `TotalAmountCalculator`: single item, multiple items, rounding edge cases.

**Definition of Done**
- All unit and `@WebMvcTest` tests pass
- `mvn verify` passes with coverage threshold met on `service/` and `domain/`

---

### ORDER-004 — order-service Integration Test Suite

**Goal**
Cover all order-service end-to-end scenarios against a real database with WireMock standing in for product-service, verifying the outbox and timeline write guarantees under transactional boundaries.

**Scope**
- `AbstractIntegrationTest` base class with static Testcontainers PostgreSQL 15 and WireMock server
- Integration tests tagged `@Tag("integration")` in `src/test/java/.../integration/`
- All scenarios from Section 13.2 order-service block of the technical design
- `application-test.yml`: Flyway enabled, `ddl-auto: create-drop`; `product-service.base-url` overridden to WireMock server URL

**Acceptance Criteria**
1. `POST /orders` success: `201`, order row in DB with `status = 'PENDING'`, one `ord_outbox` row (`order.placed.v1`, `status = 'PENDING'`), one `ord_order_timeline` row (`ORDER_PLACED`)
2. `POST /orders` idempotency key reuse: `200` with original order; no new rows in `ord_orders`, `ord_outbox`, or `ord_order_timeline`
3. `POST /orders` with product-service WireMock returning `503` on all attempts: `503` response; no rows written to any order table
4. `POST /orders` with WireMock returning inactive product: `422` response; no rows written
5. `GET /orders` as customer A: returns only customer A's orders — customer B's orders not present
6. `GET /orders/{id}` by customer B for customer A's order: `403`
7. `GET /orders/{id}/timeline`: events returned in ascending `occurred_at` order
8. `POST /orders/{id}/cancel` on PENDING order: `200`, `status = 'CANCELLED'`, `ORDER_CANCELLED` timeline entry written
9. `POST /orders/{id}/cancel` on already-CANCELLED order: `200`, no duplicate timeline entry
10. `POST /orders/{id}/cancel` on a non-PENDING/non-CANCELLED order: `409` with `ORDER_NOT_CANCELLABLE`
11. Flyway `@DataJpaTest`: all four tables present with correct schema

**Testing Requirement**
JUnit 5 + `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate` + Testcontainers PostgreSQL + WireMock. JWT tokens for CUSTOMER and ADMIN roles generated with a test RSA key pair in `@BeforeAll`. Each test cleans up its data in `@AfterEach`.

**Definition of Done**
- All integration tests pass in isolation and as a suite
- `mvn verify -pl order-service -P integration` passes
- No `@Disabled` tests without a linked issue comment

---

## Ticket Summary

| Ticket | Group | Title | Depends On |
|---|---|---|---|
| INFRA-001 | INFRA | Root Maven POM and Build Configuration | — |
| INFRA-002 | INFRA | common-api Shared Library | INFRA-001 |
| INFRA-003 | INFRA | common-events Shared Library | INFRA-001 |
| INFRA-004 | INFRA | common-security Shared Library | INFRA-001 |
| INFRA-005 | INFRA | common-observability Shared Library | INFRA-001 |
| AUTH-001 | AUTH | auth-service Scaffold and Flyway Migrations | INFRA-001–005 |
| AUTH-002 | AUTH | JPA Entities, Repositories, and JwtService | AUTH-001 |
| AUTH-003 | AUTH | AuthServiceImpl Core Business Logic | AUTH-002 |
| AUTH-004 | AUTH | AuthController, SecurityConfig, and Rate Limiter | AUTH-003 |
| AUTH-005 | AUTH | auth-service Integration Test Suite | AUTH-004 |
| GATEWAY-001 | GATEWAY | gateway-service Scaffold and Route Configuration | INFRA-001–005 |
| GATEWAY-002 | GATEWAY | JWT Validation Filter | GATEWAY-001, AUTH-002 |
| GATEWAY-003 | GATEWAY | X-Request-ID Filter and Rate Limiting | GATEWAY-002 |
| GATEWAY-004 | GATEWAY | gateway-service Integration Test Suite | GATEWAY-003 |
| PRODUCT-001 | PRODUCT | product-service Scaffold and Flyway Migrations | INFRA-001–005 |
| PRODUCT-002 | PRODUCT | Product Domain, Repositories, and InventoryClient Stub | PRODUCT-001 |
| PRODUCT-003 | PRODUCT | ProductServiceImpl and ProductController | PRODUCT-002 |
| PRODUCT-004 | PRODUCT | product-service Integration Test Suite | PRODUCT-003 |
| ORDER-001 | ORDER | order-service Scaffold and Flyway Migrations | INFRA-001–005 |
| ORDER-002 | ORDER | Order Domain, Repositories, and ProductClient | ORDER-001 |
| ORDER-003 | ORDER | OrderServiceImpl and OrderController | ORDER-002, PRODUCT-003 |
| ORDER-004 | ORDER | order-service Integration Test Suite | ORDER-003 |
| INFRA-006 | INFRA | Docker Compose and Local Infrastructure | AUTH-005, GATEWAY-004, PRODUCT-004, ORDER-004 |
