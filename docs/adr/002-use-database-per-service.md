# ADR-002: Use Database Per Service

**Status**: Accepted  
**Date**: 2024-01-15  
**Deciders**: Engineering Team

---

## Context

Having decided on microservices (ADR-001), we need to decide how services share data. The two main options are:

1. **Shared database** — all services connect to the same PostgreSQL instance (possibly different schemas).
2. **Database per service** — each service has its own database instance that no other service can access.

A third option, **shared schema** (all services in one schema), is rejected immediately as it provides no isolation at all.

---

## Decision

Each service owns its own **dedicated PostgreSQL database instance**. No service may connect to another service's database. Cross-service data access is only permitted via the service's REST API or domain events.

In production: one RDS instance per service.  
In local development: one PostgreSQL container per service in `docker-compose.yml`.  
For cost-constrained environments: one RDS instance with separate schemas and separate database users per service is acceptable, but the application-level rule (no cross-schema queries) must still be enforced.

---

## Rationale

**Why not a shared database?**

A shared database creates tight coupling at the data layer even when services are deployed independently:

- **Schema coupling**: a migration in one service's tables can break another service's queries. Teams cannot evolve their schema without coordinating with every other team.
- **No failure isolation**: a long-running query from one service can degrade the database for all services.
- **No independent scaling**: you cannot scale the payment service's database independently of the product catalog's database.
- **Undermines service boundaries**: if service A can read service B's tables directly, the boundary is a fiction. Teams will take shortcuts and query across services, creating hidden dependencies.

**Why database per service?**

- Each team owns their schema completely. They can add columns, rename tables, and run migrations without coordinating with other teams.
- Each database can be sized and scaled independently (payment-service needs more IOPS than notification-service).
- A database failure in one service does not affect others.
- Forces explicit contracts: if service A needs data from service B, it must go through B's API or consume B's events. This makes dependencies visible and versioned.

---

## Consequences

**Positive**:
- Complete schema autonomy per team.
- Independent database scaling and sizing.
- Database failures are isolated to one service.
- No cross-service query coupling.

**Negative / Mitigations**:
- **No cross-service joins**: if order-service needs the product name, it cannot join to the product table. Mitigation: snapshot the product name into the order line item at order creation time (denormalisation). This is an intentional trade-off.
- **Eventual consistency**: data that spans services (e.g., "show me all orders with their current stock levels") requires either an API composition at the gateway or an event-driven read model. Mitigation: the order timeline API composes data from events stored by order-service; it does not query other services at read time.
- **More infrastructure**: 8 RDS instances in production. Mitigation: Terraform manages all instances; cost is justified by the isolation benefits.
- **Data duplication**: product name and price are stored in both product-service and order-service (as a snapshot). Mitigation: the snapshot is intentional — it preserves the price at the time of order, which is correct business behaviour.

---

## Alternatives Considered

**Shared database with separate schemas**: rejected. Separate schemas with separate users provide some isolation, but teams can still accidentally (or intentionally) query across schemas. The application-level rule is hard to enforce without tooling. The schema coupling problem remains.

**Shared database, single schema**: rejected immediately. No isolation whatsoever.
