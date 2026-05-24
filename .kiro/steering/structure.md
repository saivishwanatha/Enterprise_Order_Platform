# Project Structure Standards

## Repository Layout

```
enterprise-order-platform/
├── pom.xml                          # Root Maven POM (parent, dependency management)
├── docker-compose.yml               # Local dev: all services + infra
├── docker-compose.test.yml          # CI: services + LocalStack + test DBs
│
├── shared-libraries/
│   ├── common-api/                  # Shared DTOs, pagination wrappers, error models
│   ├── common-events/               # Event envelope, all domain event POJOs
│   ├── common-security/             # JWT filter, SecurityContext helpers
│   └── common-observability/        # MDC config, tracing beans, log format
│
├── auth-service/
├── gateway-service/
├── order-service/
├── inventory-service/
├── payment-service/
├── shipment-service/
├── invoice-service/
├── notification-service/
├── product-service/
│
├── infra/
│   ├── terraform/                   # AWS infrastructure as code
│   ├── localstack/                  # LocalStack init scripts (SNS/SQS/S3 setup)
│   ├── docker/                      # Base Dockerfiles
│   └── aws-scripts/                 # One-off AWS CLI scripts
│
└── docs/
    ├── adr/                         # Architecture Decision Records
    ├── api-contracts/               # Generated OpenAPI specs
    └── event-contracts/             # Event schema definitions
```

## Per-Service Layout

Every service follows this exact package and directory structure:

```
{service}-service/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/enterprise/{service}/
    │   │   ├── {Service}Application.java
    │   │   ├── config/              # Spring @Configuration classes
    │   │   ├── controller/          # REST controllers (@RestController)
    │   │   ├── service/             # Business logic (interfaces + impls)
    │   │   ├── domain/              # Entities, value objects, domain events
    │   │   ├── repository/          # Spring Data JPA repositories
    │   │   ├── dto/                 # Request/response records
    │   │   ├── mapper/              # MapStruct mappers
    │   │   ├── event/               # Event publishers and listeners
    │   │   ├── outbox/              # Outbox table entity + scheduler
    │   │   └── exception/           # Custom exceptions + @ControllerAdvice
    │   └── resources/
    │       ├── application.yml
    │       ├── application-local.yml
    │       ├── application-prod.yml
    │       └── db/migration/        # Flyway SQL scripts
    └── test/
        ├── java/com/enterprise/{service}/
        │   ├── unit/                # Pure unit tests (no Spring context)
        │   ├── integration/         # @SpringBootTest slice tests
        │   └── contract/            # Consumer-driven contract tests
        └── resources/
            └── application-test.yml
```

## Naming Conventions

- **Packages**: all lowercase, no underscores — `com.enterprise.orderservice.domain`
- **Classes**: PascalCase. Suffix rules:
  - Controllers: `*Controller`
  - Services (interface): `*Service`, implementation: `*ServiceImpl`
  - Repositories: `*Repository`
  - Entities: no suffix (e.g., `Order`, `OrderItem`)
  - DTOs: `*Request`, `*Response`
  - Events: `*Event` (e.g., `OrderPlacedEvent`)
  - Mappers: `*Mapper`
  - Exceptions: `*Exception`
- **Database tables**: `snake_case`, prefixed with service abbreviation (e.g., `ord_orders`, `inv_stock_items`).
- **Flyway scripts**: `V1__create_orders_table.sql`, `V2__add_status_index.sql`

## Shared Libraries Rules

- `common-api`: only DTOs, pagination, and error response models. No Spring beans.
- `common-events`: only event POJOs and the `EventEnvelope` wrapper. No Spring beans.
- `common-security`: JWT validation filter and `SecurityContextHelper`. May have Spring beans.
- `common-observability`: MDC filter, tracing configuration. May have Spring beans.
- Services must declare shared library dependencies explicitly in their `pom.xml`. No transitive reliance.

## Configuration Rules

- All environment-specific values (DB URLs, AWS endpoints, secrets) are injected via environment variables.
- Use `@ConfigurationProperties` with a dedicated `*Properties` record class for each logical config group.
- Never hardcode ports, hostnames, or credentials anywhere in source code.
- Default server port: `8080`. Management port: `8081`. Each service gets a unique default port documented in its `README.md`.
