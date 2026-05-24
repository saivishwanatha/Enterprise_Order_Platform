# Technology Standards

## Language & Runtime

- **Java 21** — use virtual threads (`spring.threads.virtual.enabled=true`) where applicable.
- Use records for DTOs and value objects. Prefer sealed interfaces for domain state machines.
- No raw types. No unchecked casts. Enable `-Xlint:all` in compiler config.

## Framework

- **Spring Boot 3.x** — never downgrade to Boot 2.x patterns.
- Use constructor injection exclusively. Never use `@Autowired` on fields.
- Annotate all configuration classes with `@Configuration`. Never put `@Bean` methods in `@SpringBootApplication` classes.
- Use `application.yml` (not `application.properties`). Environment-specific overrides go in `application-{profile}.yml`.

## Database

- **PostgreSQL** (AWS RDS in production, Docker in local/CI).
- **Database-per-service**: each service owns exactly one schema/database. No cross-service joins.
- **Flyway** manages all schema migrations. Migration files live in `src/main/resources/db/migration/` and follow `V{n}__{description}.sql` naming.
- Never use `spring.jpa.hibernate.ddl-auto` set to anything other than `validate` in non-test profiles.
- Use Spring Data JPA repositories. Write JPQL or native SQL for complex queries; avoid Criteria API unless necessary.
- All entities must have an `@Version` field for optimistic locking.

## Security

- **Spring Security 6** with **JWT** (stateless sessions).
- JWTs are issued by `auth-service` and validated at the gateway and optionally at each service.
- Never store secrets in code or `application.yml`. Use AWS Secrets Manager or environment variables injected at runtime.
- All endpoints require authentication unless explicitly annotated `@PermitAll` / listed in the security whitelist.
- Use `@PreAuthorize` with SpEL expressions for method-level authorization.

## Resilience

- **Resilience4j** for circuit breakers, retry, rate limiting, and bulkhead.
- Every outbound HTTP call (RestClient / WebClient) must be wrapped in a circuit breaker.
- Retry policies: max 3 attempts, exponential backoff starting at 500 ms, jitter enabled.
- Circuit breaker thresholds: 50% failure rate over a 10-call sliding window; wait 30 s in open state.
- Fallback methods must be defined for every circuit breaker.

## API Documentation

- **SpringDoc OpenAPI 3** (`springdoc-openapi-starter-webmvc-ui`).
- Every controller must be annotated with `@Tag`. Every operation with `@Operation` and `@ApiResponse`.
- OpenAPI specs are exported to `docs/api-contracts/` as part of the build.

## Build

- **Maven** multi-module project at the root. Each service is a module.
- Java source/target compatibility: `21`.
- Enforce code style with Checkstyle (Google style). Build fails on violations.
- Use `spring-boot-maven-plugin` for executable JARs. No WAR packaging.

## Observability

- **Micrometer** with CloudWatch registry in production; Prometheus in local.
- **Spring Boot Actuator** exposed on a separate management port (`8081`). Expose `health`, `info`, `metrics`, `prometheus` endpoints only.
- Structured JSON logging via Logback. Every log line must include `traceId`, `spanId`, `serviceId`, and `correlationId` via MDC.
- Use **Spring Cloud Sleuth** / Micrometer Tracing for distributed tracing.

## Dependency Rules

- Pin all dependency versions in the root `pom.xml` via `<dependencyManagement>`.
- No `SNAPSHOT` dependencies in `main` branch builds.
- Shared code lives in `shared-libraries/` modules only. Services must not import each other's modules.
