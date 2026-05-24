# Testing Standards

## Test Pyramid

```
         /\
        /  \   E2E (minimal — docker-compose smoke tests)
       /----\
      /      \  Integration (Spring slices + Testcontainers)
     /--------\
    /          \  Unit (pure Java, no Spring context)
   /____________\
```

- **Unit tests**: fast, no I/O, no Spring context. Cover all service and domain logic.
- **Integration tests**: test a slice of the application with real infrastructure (DB, queues) via Testcontainers.
- **Contract tests**: verify API and event contracts between services.
- Target: ≥ 80% line coverage on `service/` and `domain/` packages. Coverage is enforced by the Maven build.

## Unit Tests

- Framework: **JUnit 5** (`@ExtendWith(MockitoExtension.class)`).
- Mocking: **Mockito** only. No PowerMock.
- Test class naming: `{ClassUnderTest}Test` in the same package under `src/test/`.
- One test class per production class.
- Use `@DisplayName` on every test method with a human-readable description.
- Use `@Nested` classes to group tests by method or scenario.
- Arrange-Act-Assert structure. No logic in assertions — use AssertJ fluent assertions.
- Never use `Thread.sleep()` in tests. Use `Awaitility` for async assertions.
- Test data: use static factory methods or builder patterns. No `new` constructors with long argument lists in test bodies.

```java
// Good
@Test
@DisplayName("should throw OrderNotFoundException when order does not exist")
void shouldThrowWhenOrderNotFound() {
    // Arrange
    given(orderRepository.findById(ORDER_ID)).willReturn(Optional.empty());

    // Act & Assert
    assertThatThrownBy(() -> orderService.getOrder(ORDER_ID))
        .isInstanceOf(OrderNotFoundException.class)
        .hasMessageContaining(ORDER_ID.toString());
}
```

## Integration Tests

- Framework: **Testcontainers** for PostgreSQL, LocalStack (SNS/SQS/S3).
- Use `@SpringBootTest(webEnvironment = RANDOM_PORT)` for full-context tests.
- Use `@DataJpaTest` for repository-only tests (auto-configures Testcontainers PostgreSQL).
- Use `@WebMvcTest` for controller-only tests (mock the service layer).
- Annotate integration test classes with `@Tag("integration")`.
- Integration tests live in `src/test/java/.../integration/`.
- Use a shared `AbstractIntegrationTest` base class that starts Testcontainers once per test suite (`@TestcontainersConfig` with `@Container` as static fields).
- Use `application-test.yml` to override datasource and AWS endpoint URLs to point to Testcontainers instances.
- Database state: each test method must clean up after itself or use `@Transactional` (rolls back after test).

## Contract Tests

- Use **Spring Cloud Contract** or **Pact** for consumer-driven contract testing.
- Contracts live in `src/test/resources/contracts/`.
- Producers verify contracts as part of their build. Consumers generate stubs from contracts.
- Event contracts: verify that published event payloads match the schemas in `common-events`.

## Test Configuration

- `application-test.yml` must set:
  - `spring.jpa.hibernate.ddl-auto: create-drop` (Testcontainers DB is ephemeral)
  - `spring.flyway.enabled: true`
  - AWS endpoint overrides pointing to LocalStack container
- Never use `H2` in-memory database for tests. Always use Testcontainers PostgreSQL to match production.

## Naming & Organisation

- Unit test packages mirror production packages: `com.enterprise.orderservice.service` → `com.enterprise.orderservice.unit.service`.
- Integration test packages: `com.enterprise.orderservice.integration`.
- Test utility classes go in `src/test/java/.../support/` (builders, fixtures, matchers).

## What Must Be Tested

| Layer | Required Coverage |
|---|---|
| Domain entities / value objects | All state transitions and invariants |
| Service layer | All business rules, all exception paths |
| Repository layer | Custom queries (integration test with real DB) |
| Controllers | All endpoints: happy path + validation errors + auth (WebMvcTest) |
| Event listeners | Idempotency, happy path, failure/retry (integration test with LocalStack) |
| Outbox poller | Publishes pending rows, skips published rows |
| Circuit breakers | Open/closed/half-open transitions (unit test with mocked clock) |

## CI Rules

- Unit tests run on every push.
- Integration tests run on every pull request.
- Build fails if coverage drops below 80% on `service/` and `domain/` packages.
- Flaky tests must be fixed or removed within one sprint. No `@Disabled` without a linked issue comment.
