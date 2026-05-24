# ADR-005: Use the Choreography Saga Pattern

**Status**: Accepted  
**Date**: 2024-01-15  
**Deciders**: Engineering Team

---

## Context

The order fulfillment workflow spans multiple services: order-service, inventory-service, payment-service, shipment-service, and invoice-service. Each service has its own database (ADR-002), so there is no single ACID transaction that can span all steps.

We need a pattern for managing this multi-step, multi-service business transaction that:

- Guarantees that all steps either complete successfully or are compensated (rolled back) if any step fails.
- Does not require distributed locks or a two-phase commit.
- Tolerates individual service failures without corrupting the overall system state.

The three main options are:

1. **Two-Phase Commit (2PC)** — a distributed transaction coordinator locks all participants until all agree to commit.
2. **Orchestration Saga** — a central orchestrator service tells each participant what to do and tracks the overall state.
3. **Choreography Saga** — each service reacts to events and publishes its own result events. No central coordinator.

---

## Decision

We will use the **Choreography Saga** pattern.

Each service listens for domain events, performs its local transaction, and publishes its own result event. Compensation is triggered by failure events that each service listens for.

**Happy path**:
```
order.placed.v1
  → inventory-service reserves stock → inventory.reserved.v1
    → payment-service captures payment → payment.captured.v1
      → shipment-service creates shipment → shipment.created.v1
      → invoice-service generates invoice → invoice.generated.v1
        → order-service marks order CONFIRMED
```

**Compensation (payment failure)**:
```
payment.failed.v1
  → inventory-service releases reservation → inventory.released.v1
  → order-service marks order FAILED → order.failed.v1
    → notification-service sends failure email
```

---

## Rationale

### Why not 2PC?

Two-phase commit requires a distributed transaction coordinator that holds locks across all participants until the transaction commits or aborts. In a microservices architecture:

- AWS SNS/SQS does not participate in distributed transactions.
- Holding locks across network calls creates deadlock risk and severely limits throughput.
- The coordinator is a single point of failure.
- 2PC is fundamentally incompatible with the eventual consistency model we have accepted.

2PC is rejected.

### Why Choreography over Orchestration?

| Factor | Choreography | Orchestration |
|---|---|---|
| Coupling | ✅ Services only know about events, not each other | ❌ Orchestrator knows all participants |
| Single point of failure | ✅ No central coordinator | ❌ Orchestrator failure stops all sagas |
| Simplicity (small sagas) | ✅ Simple for linear flows | ⚠️ More setup for simple flows |
| Visibility | ❌ Saga state is distributed across services | ✅ Orchestrator has full saga state |
| Complex branching | ❌ Hard to reason about complex conditional flows | ✅ Orchestrator handles branching explicitly |
| Team autonomy | ✅ Each team owns their service's reaction | ❌ Orchestrator team becomes a bottleneck |
| Debugging | ❌ Requires distributed tracing to follow a saga | ✅ Orchestrator state is queryable |

**For this platform**, the order fulfillment saga is a **linear flow** with well-defined compensation steps. There is no complex branching. The choreography approach keeps each service autonomous — inventory-service does not need to know that payment-service exists; it only knows that when it receives `order.placed.v1`, it should reserve stock and publish a result.

The visibility trade-off is mitigated by:
- The `ord_order_timeline` table in order-service, which records every saga event as it passes through.
- The `correlationId` on every event, which allows distributed tracing to reconstruct the full saga flow in CloudWatch.
- The order timeline API (`GET /api/v1/orders/{id}/timeline`) which gives operations a human-readable view of every saga step.

### Why order-service is the "saga anchor"

order-service is the natural anchor because:
- It owns the order aggregate, which is the root of the entire workflow.
- It listens for all saga outcome events (`inventory.reservation_failed.v1`, `payment.failed.v1`, `shipment.created.v1`, `invoice.generated.v1`) and records them in the timeline.
- It is the only service that knows the full saga has completed (when both shipment and invoice events are received).
- It publishes `order.failed.v1` and `order.confirmed.v1` as the final saga outcomes.

order-service does not tell other services what to do — it reacts to their events and maintains the order's state. This is choreography, not orchestration.

---

## Compensation Design

Every saga step that can fail must have a defined compensation:

| Step | Success Event | Failure Event | Compensation |
|---|---|---|---|
| Inventory reservation | `inventory.reserved.v1` | `inventory.reservation_failed.v1` | None needed (reservation never happened) |
| Payment capture | `payment.captured.v1` | `payment.failed.v1` | Release inventory reservation |
| Shipment creation | `shipment.created.v1` | (operational failure → DLQ) | Manual intervention |
| Invoice generation | `invoice.generated.v1` | (operational failure → DLQ) | Replay from DLQ |

Compensation transactions are themselves idempotent — releasing a reservation that was already released is a no-op.

---

## Consequences

**Positive**:
- No central coordinator — no single point of failure for the saga.
- Each service is autonomous and only knows about its own events.
- Adding a new consumer (e.g., a loyalty-points service that listens to `order.confirmed.v1`) requires no changes to existing services.
- Linear flows are easy to reason about.

**Negative / Mitigations**:
- **Distributed saga state**: no single place holds the complete saga state. Mitigation: order-service's timeline table and the `correlationId` on all events provide full traceability.
- **Cyclic event risk**: if compensation events trigger further events that trigger further compensation, the system can loop. Mitigation: compensation events (`inventory.released.v1`, `payment.refunded.v1`) are terminal — no service publishes a new saga-advancing event in response to them.
- **Harder to debug than orchestration**: following a saga requires tracing events across multiple services. Mitigation: structured logging with `correlationId` and the order timeline API.
- **Eventual consistency**: the order is not immediately `CONFIRMED` — it transitions through states as events arrive. Mitigation: the order status API reflects the current state, and the timeline shows the progression. Clients poll or use the timeline to track progress.

---

## Alternatives Considered

**Orchestration Saga (e.g., AWS Step Functions)**: viable and provides better visibility. Rejected for v1 because it introduces a central coordinator that all teams must coordinate changes through, and Step Functions adds cost and complexity. Can be revisited if the saga logic becomes too complex for choreography.

**Two-Phase Commit**: rejected. Incompatible with the eventual consistency model and AWS SNS/SQS.
