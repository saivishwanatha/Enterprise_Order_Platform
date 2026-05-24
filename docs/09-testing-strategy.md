# Testing Strategy

## Test Pyramid

```
              ▲
             /|\
            / | \        E2E / Smoke Tests
           /  |  \       (docker-compose, minimal, CI only)
          /   |   \
         /    |    \     Integration Tests
        /     |     \    (Testcontainers + LocalStack, per service)
       /      |      \
      /       |       \  Unit Tests
     /        |        \ (JUnit 5 + Mockito, no Spring context)
    /         |         \
   ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔
```

**Coverage target**: ≥ 80% line coverage on `service/` and `domain/` packages, enforced by the Maven build (Jacoco). Build fails if coverage drops below threshold.

---

## Unit Tests

**Scope**: pure Java logic — no Spring context, no I/O, no network.

**Framework**: JUnit 5 + Mockito + AssertJ

**What to unit test per service**:

| Layer | What to test |
|---|---|
| Domain entities | All state transitions, invariant enforcement, business rule violations |
| Service classes | All business logic paths, all exception paths, all branching conditions |
| Mappers | DTO ↔ entity mapping correctness |
| Outbox publisher | Correct event construction, correct outbox row creation |
| Saga event handlers | Correct state transitions triggered by each event type |
| Circuit breaker config | Open/closed/half-open transitions (mock clock) |

**Conventions**:
- Test class: `{ClassUnderTest}Test` in the same package under `src/test/java/.../unit/`.
- Use `@ExtendWith(MockitoExtension.class)`. Never use `@SpringBootTest` for unit tests.
- Use `@DisplayName` on every test method.
- Use `@Nested` to group tests by method or scenario.
- Arrange-Act-Assert structure. No logic in assertions.
- Use `BDDMockito.given(...)` style (not `Mockito.when(...)`).
- Use `assertThatThrownBy(...)` for exception assertions.
- Never use `Thread.sleep()`. Use `Awaitility` for async assertions.

**Example structure**:
```
unit/
  service/
    OrderServiceTest.java
    InventoryServiceTest.java
  domain/
    OrderTest.java
    ReservationTest.java
  event/
    OrderEventPublisherTest.java
```

---

## Integration Tests

**Scope**: a service slice with real infrastructure — real PostgreSQL, real LocalStack SNS/SQS/S3.

**Framework**: JUnit 5 + Testcontainers + Spring Boot Test slices + AssertJ + Awaitility

**Infrastructure**:
- PostgreSQL: `org.testcontainers:postgresql` — one container per test suite (static `@Container`).
- LocalStack: `org.testcontainers:localstack` — one container per test suite.
- Flyway runs automatically on test startup.
- `application-test.yml` overrides datasource URL and AWS endpoint to point to containers.

**Never use H2**. Always use Testcontainers PostgreSQL to match production behaviour (JSON operators, advisory locks, etc.).

**Test slices**:

| Slice | Annotation | Use for |
|---|---|---|
| Full context | `@SpringBootTest(webEnvironment = RANDOM_PORT)` | End-to-end within a service: HTTP → DB → event |
| Web layer only | `@WebMvcTest(OrderController.class)` | Controller validation, auth, response shape |
| Data layer only | `@DataJpaTest` | Repository custom queries, Flyway migrations |
| Messaging only | Custom slice | SQS listener → DB → outbox |

**Base class pattern**:
```java
// All integration tests extend this
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
@Testcontainers
abstract class AbstractIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Container
    static LocalStackContainer localstack = new LocalStackContainer(...)
        .withServices(SQS, SNS, S3);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("aws.endpoint-override", () -> localstack.getEndpointOverride(SQS).toString());
        // ...
    }
}
```

**What to integration test per service**:

| Concern | Test approach |
|---|---|
| REST endpoints (happy path) | `@SpringBootTest` + `TestRestTemplate` or `MockMvc` |
| REST endpoints (validation errors) | `@WebMvcTest` |
| REST endpoints (auth/403) | `@WebMvcTest` with mock JWT |
| Repository custom queries | `@DataJpaTest` with Testcontainers PostgreSQL |
| Flyway migrations | `@DataJpaTest` — verify schema is correct after migration |
| SQS listener (happy path) | Publish to LocalStack SQS, assert DB state after processing |
| SQS listener (idempotency) | Publish same event twice, assert processed once |
| SQS listener (failure → DLQ) | Publish malformed event, assert DLQ receives it |
| Outbox poller | Insert outbox row, run poller, assert SNS received message |
| S3 upload (invoice-service) | Assert PDF uploaded to LocalStack S3 |

**Test data cleanup**: each test method cleans up via `@Transactional` (rolls back) or explicit `@BeforeEach` / `@AfterEach` DELETE statements. Never rely on test ordering.

---

## Contract Tests

**Scope**: verify that event payloads published by producers match the schemas expected by consumers.

**Approach**: JSON Schema validation tests in `common-events`.

For each event type in `common-events`, a test verifies:
1. The event can be serialised to JSON without errors.
2. The serialised JSON matches the documented schema (required fields present, correct types).
3. The event can be deserialised from JSON with `@JsonIgnoreProperties(ignoreUnknown = true)` (forward compatibility).

These tests run as unit tests — no infrastructure required.

---

## Test Configuration (`application-test.yml`)

```yaml
spring:
  datasource:
    url: # set by @DynamicPropertySource
    username: test
    password: test
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true

aws:
  region: us-east-1
  endpoint-override: # set by @DynamicPropertySource

logging:
  level:
    com.enterprise: DEBUG
```

---

## Per-Service Test Checklist

### auth-service
- [ ] Register: success, duplicate email, validation errors
- [ ] Login: success, wrong password, rate limit
- [ ] Refresh token: success, expired token, revoked token
- [ ] JWT issued with correct claims and expiry
- [ ] `user.registered.v1` published to outbox on registration

### product-service
- [ ] Create product: success (admin), forbidden (customer), validation errors
- [ ] List products: pagination, category filter, search, inactive products excluded
- [ ] `product.created.v1` published to outbox on creation

### order-service
- [ ] Create order: success, product not found, idempotency key deduplication
- [ ] Order timeline: correct events in correct order
- [ ] Retry: success, max retries exceeded, order not in FAILED state
- [ ] Cancel: success, wrong owner, non-cancellable state
- [ ] Saga: `inventory.reservation_failed.v1` → order marked FAILED
- [ ] Saga: `payment.failed.v1` → order marked FAILED
- [ ] Saga: `invoice.generated.v1` + `shipment.created.v1` → order marked CONFIRMED

### inventory-service
- [ ] Reserve: success, insufficient stock, idempotency
- [ ] Release: success, idempotency
- [ ] Confirm: success, idempotency
- [ ] Initialise stock on `product.created.v1`
- [ ] Optimistic lock: concurrent reservations for same product

### payment-service
- [ ] Capture: success, failure (simulated), idempotency
- [ ] Refund: success, idempotency
- [ ] `payment.captured.v1` and `payment.failed.v1` published correctly

### shipment-service
- [ ] Create shipment: success, idempotency
- [ ] Status transitions: CREATED → DISPATCHED → IN_TRANSIT → DELIVERED
- [ ] Each transition publishes correct event

### invoice-service
- [ ] Generate invoice: success, PDF uploaded to S3, idempotency (second event returns existing)
- [ ] Pre-signed URL: correct expiry, correct S3 key

### notification-service
- [ ] Email sent for each event type
- [ ] Idempotency: same event delivered twice → one email sent
- [ ] Notification logged in `ntf_notifications`

---

## CI Pipeline

```
Push to any branch:
  └─► Unit tests (all services, parallel)
  └─► Checkstyle (build fails on violations)
  └─► Jacoco coverage check (≥ 80% on service/ and domain/)

Pull Request to main:
  └─► Unit tests
  └─► Integration tests (all services, Testcontainers)
  └─► Contract tests (common-events)
  └─► Coverage check

Merge to main:
  └─► All above
  └─► Docker image build (all services)
  └─► Smoke test against docker-compose stack
```

**Rules**:
- No `@Disabled` without a linked issue comment.
- Flaky tests must be fixed or removed within one sprint.
- Integration tests are tagged `@Tag("integration")` and can be excluded from local fast-feedback runs with `-DexcludedGroups=integration`.
