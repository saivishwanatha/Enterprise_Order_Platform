# ADR-004: Use the Transactional Outbox Pattern

**Status**: Accepted  
**Date**: 2024-01-15  
**Deciders**: Engineering Team

---

## Context

Every service in this platform needs to publish domain events to SNS after performing a local database transaction. For example, when order-service creates an order, it must:

1. Insert a row into `ord_orders`.
2. Publish `order.placed.v1` to SNS.

The naive implementation does both in sequence:

```java
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(new Order(...));
    snsClient.publish(orderPlacedEvent);  // ← PROBLEM
    return order;
}
```

This creates the **dual-write problem**: two separate systems (the database and SNS) are updated in a single operation, but there is no distributed transaction spanning both. This means:

- If the database commit succeeds but SNS publish fails → the order exists but no saga starts. The order is stuck in `PENDING` forever.
- If SNS publish succeeds but the database rolls back → the saga starts for an order that doesn't exist. Downstream services process a ghost order.
- If the service crashes between the DB commit and the SNS publish → same as the first case.

Neither failure mode is acceptable.

---

## Decision

Every service that publishes domain events will use the **Transactional Outbox Pattern**.

**Implementation**:

1. Each service has an `{prefix}_outbox` table in its own database.
2. Within the same `@Transactional` method that performs the business operation, a row is inserted into the outbox table. The outbox row contains the serialised event payload.
3. The database transaction commits atomically: either both the business row and the outbox row are committed, or neither is.
4. A separate **outbox poller** (a `@Scheduled` Spring bean) runs every 5 seconds, reads `PENDING` outbox rows, publishes them to SNS, and marks them `PUBLISHED`.
5. The SNS publish is outside any database transaction. If it fails, the row remains `PENDING` and will be retried on the next poll cycle.

**Outbox table schema** (per service):
```sql
CREATE TABLE {prefix}_outbox (
    id            UUID PRIMARY KEY,
    event_type    VARCHAR(100) NOT NULL,
    aggregate_id  UUID NOT NULL,
    payload       JSONB NOT NULL,
    correlation_id UUID NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at  TIMESTAMPTZ
);
CREATE INDEX ON {prefix}_outbox (status, created_at) WHERE status = 'PENDING';
```

---

## Rationale

The Transactional Outbox pattern solves the dual-write problem by reducing it to a single-system write: the database. The outbox row and the business row are committed in the same ACID transaction. The SNS publish becomes a best-effort, retryable operation that reads from the database — a system we already trust.

**Why not use a distributed transaction (2PC)?**

Two-phase commit across a relational database and SNS is not supported by AWS SNS. Even if it were, 2PC introduces blocking locks and is fragile in the face of coordinator failures. It is not a viable option for a cloud-native system.

**Why not publish directly to SNS inside the transaction?**

SNS is an external system. Publishing inside a `@Transactional` method means the SNS call happens before the transaction commits. If the transaction rolls back after the SNS publish, the event has already been sent — there is no way to un-publish it. This is the ghost-order scenario described above.

**Why a polling approach rather than CDC (Change Data Capture)?**

CDC (e.g., Debezium reading the PostgreSQL WAL) is a more sophisticated approach that avoids polling latency and database load. However, it requires additional infrastructure (Kafka Connect or a Debezium connector), operational expertise, and is harder to run locally. For this platform's scale and team size, a simple polling approach is sufficient. The 5-second polling interval is acceptable given the eventual consistency model. CDC can be adopted later if polling becomes a bottleneck.

---

## Consequences

**Positive**:
- Eliminates the dual-write problem. Business state and event publication are atomic.
- Simple to implement and understand — just a table and a scheduled job.
- Works with any message broker (SNS, Kafka, RabbitMQ) without changing the business logic.
- Provides a natural audit log of all published events.
- Retries are automatic — failed SNS publishes are retried on the next poll cycle without any additional infrastructure.

**Negative / Mitigations**:
- **Polling latency**: events are published within 5 seconds of the transaction commit, not immediately. Mitigation: 5 seconds is acceptable for this platform's eventual consistency model. The polling interval is configurable.
- **Database load**: the poller queries the outbox table every 5 seconds. Mitigation: the `PENDING` partial index makes this query efficient even with a large outbox table.
- **At-least-once publishing**: if the poller publishes to SNS and then crashes before marking the row `PUBLISHED`, the event will be published again on the next poll cycle. Mitigation: consumers are idempotent via the `processed_events` table (see failure-handling.md). Duplicate events are silently ignored.
- **Outbox table growth**: published rows accumulate. Mitigation: a nightly job purges rows in `PUBLISHED` state older than 7 days.
- **Poller is a single point of failure per service**: if the poller thread dies, events stop being published. Mitigation: Spring's `@Scheduled` runs on a managed thread pool; failures are logged and the next execution is attempted. CloudWatch alarm fires if pending rows are older than 10 minutes.

---

## Alternatives Considered

**Publish directly to SNS inside `@Transactional`**: rejected. Creates the ghost-event problem (event published for a rolled-back transaction).

**Publish after `@Transactional` returns**: rejected. Creates the stuck-order problem (transaction committed but SNS publish fails or service crashes).

**Change Data Capture (Debezium)**: viable but adds operational complexity. Deferred to a future phase if polling becomes a bottleneck.

**Saga Orchestrator with its own state**: the orchestrator approach (ADR-005 alternative) would centralise event publishing, but we chose choreography. The outbox pattern applies equally to choreography.
