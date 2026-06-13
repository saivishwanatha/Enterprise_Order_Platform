# ADR-001: Use Microservices Architecture

**Status**: Accepted  
**Date**: 2024-01-15  
**Deciders**: Engineering Team(own project)

---

## Context

We are building an enterprise order fulfillment platform that spans multiple distinct business domains: identity, product catalog, order management, inventory, payments, shipping, invoicing, and notifications. We need to decide on the top-level architectural style.

The two primary candidates are:

1. **Modular Monolith** — a single deployable unit with well-defined internal module boundaries.
2. **Microservices** — independently deployable services, each owning its domain.

The platform has the following characteristics that inform this decision:

- Multiple teams will own different domains (auth, payments, shipping are natural team boundaries).
- Different domains have very different scaling requirements (inventory and payment under flash-sale load vs. invoice generation).
- The business expects to evolve each domain independently (e.g., swap the payment provider without touching the order flow).
- The system must tolerate partial failures — a notification service outage must not prevent order placement.

---

## Decision

We will use a **microservices architecture** with one service per bounded context.

Services:
- `auth-service`, `gateway-service`, `product-service`, `order-service`, `inventory-service`, `payment-service`, `shipment-service`, `invoice-service`, `notification-service`

---

## Rationale

| Factor | Microservices | Modular Monolith |
|---|---|---|
| Independent deployability | ✅ Each service deploys on its own schedule | ❌ Full redeploy for any change |
| Independent scalability | ✅ Scale inventory/payment independently | ❌ Scale the whole application |
| Failure isolation | ✅ Notification failure doesn't affect orders | ❌ One bug can crash everything |
| Team autonomy | ✅ Teams own their service end-to-end | ⚠️ Module boundaries erode over time |
| Technology flexibility | ✅ Each service can evolve its stack | ❌ Locked to one stack |
| Operational complexity | ❌ More moving parts, distributed tracing needed | ✅ Simpler to operate |
| Local development | ❌ Requires docker-compose to run all services | ✅ Single process |
| Data consistency | ❌ Eventual consistency across services | ✅ ACID transactions across modules |

The operational complexity trade-off is accepted because:
- We are investing in shared observability libraries (`common-observability`) and CloudWatch dashboards from day one.
- Docker Compose and LocalStack make local development tractable.
- The team autonomy and failure isolation benefits outweigh the operational cost at this scale.

---

## Consequences

**Positive**:
- Teams can deploy independently without coordination.
- A failure in notification-service or invoice-service does not block order fulfillment.
- Each service can be scaled to match its load profile.
- Clear ownership boundaries reduce cross-team merge conflicts.

**Negative / Mitigations**:
- Distributed tracing is required to debug cross-service flows → mitigated by `common-observability` with `correlationId` propagation.
- No cross-service ACID transactions → mitigated by the Saga pattern (ADR-005) and Outbox pattern (ADR-004).
- More infrastructure to manage → mitigated by Terraform and Docker Compose.
- Local development requires running multiple services → mitigated by `docker-compose up` starting the full stack.

---

## Alternatives Considered

**Modular Monolith**: rejected because it does not provide failure isolation (a payment bug can crash the notification module) and does not support independent scaling. It would be the right choice for a smaller team or earlier-stage product, but the team structure and scaling requirements favour microservices here.
