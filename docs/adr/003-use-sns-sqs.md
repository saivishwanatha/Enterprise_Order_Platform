# ADR-003: Use AWS SNS + SQS for Asynchronous Messaging

**Status**: Accepted  
**Date**: 2024-01-15  
**Deciders**: Engineering Team

---

## Context

The platform uses a Choreography Saga (ADR-005) for cross-service workflows. This requires a reliable, durable message broker that supports:

- **Fan-out**: one event (e.g., `payment.captured.v1`) must reach multiple consumers (shipment-service, invoice-service, notification-service) independently.
- **Durability**: messages must not be lost if a consumer is temporarily unavailable.
- **At-least-once delivery**: messages are retried until acknowledged.
- **Dead-letter queues**: unprocessable messages must be captured for investigation.
- **Decoupling**: producers must not know about consumers.

The platform is targeting AWS as its cloud provider. The primary candidates are:

1. **AWS SNS + SQS** — managed pub/sub (SNS) with durable queues (SQS).
2. **Apache Kafka** — distributed log, self-managed or via MSK.
3. **RabbitMQ** — AMQP broker, self-managed or via Amazon MQ.

---

## Decision

We will use **AWS SNS for fan-out** and **AWS SQS for per-consumer durable queues**.

- One SNS topic per aggregate domain (e.g., `prod-order-events`).
- One SQS queue per consumer per aggregate (e.g., `prod-inventory-order-queue`).
- SQS queues subscribe to SNS topics with message attribute filter policies on `eventType`.
- Every SQS queue has a paired DLQ.
- LocalStack emulates SNS and SQS for local development and integration tests.

---

## Rationale

### SNS + SQS vs. Kafka

| Factor | SNS + SQS | Kafka (MSK) |
|---|---|---|
| Managed service | ✅ Fully managed, no ops | ⚠️ MSK is managed but more complex |
| Fan-out | ✅ SNS fan-out is native | ✅ Multiple consumer groups |
| Message retention | ✅ Up to 14 days (SQS) | ✅ Configurable, long-term replay |
| Event replay | ❌ No replay after consumption | ✅ Replay from any offset |
| Ordering | ⚠️ FIFO queues for ordering (added cost) | ✅ Per-partition ordering |
| Throughput | ✅ Sufficient for this use case | ✅ Higher throughput ceiling |
| Operational complexity | ✅ Zero ops | ❌ Cluster management, partition tuning |
| Local dev | ✅ LocalStack emulates SNS/SQS | ⚠️ Requires Kafka container |
| Cost | ✅ Pay per message | ❌ MSK cluster cost even at low volume |
| AWS integration | ✅ Native IAM, CloudWatch, Lambda triggers | ⚠️ Requires more configuration |

**Kafka is the better choice if**: we need event replay for building read models, we need strict ordering across all events for an aggregate, or we expect very high throughput (millions of messages/day). For this platform at v1 scale, SNS+SQS is simpler, cheaper, and sufficient.

### SNS + SQS vs. RabbitMQ

RabbitMQ (Amazon MQ) requires managing broker instances, HA configuration, and has a steeper operational curve. SNS+SQS is fully managed and integrates natively with IAM and CloudWatch. RabbitMQ offers no meaningful advantage for this use case.

### Why SNS + SQS together (not SQS alone)?

SQS alone supports point-to-point messaging. When `payment.captured.v1` needs to reach three consumers (shipment, invoice, notification), SQS alone would require the producer to know about all three queues and publish to each. SNS provides the fan-out: the producer publishes once to the SNS topic, and SNS delivers to all subscribed SQS queues. Adding a new consumer requires only a new SQS subscription — the producer is unchanged.

---

## Consequences

**Positive**:
- Zero operational overhead — no broker to manage.
- Native AWS IAM integration for access control.
- CloudWatch metrics and alarms out of the box.
- LocalStack provides a faithful local emulation.
- Adding a new consumer requires only a new SQS subscription and Terraform change — no producer code changes.
- DLQs are a first-class SQS feature.

**Negative / Mitigations**:
- **No event replay**: once a message is consumed and deleted from SQS, it cannot be replayed. Mitigation: the Outbox pattern (ADR-004) retains the event in the database. For replay, we can re-publish from the outbox. For analytics/audit, the order timeline serves as the event log.
- **No strict ordering** (standard queues): SQS standard queues deliver messages at-least-once and in approximate order. Mitigation: consumers are idempotent (processed_events table) and the saga is designed to tolerate out-of-order delivery within a single saga instance.
- **Message size limit**: 256 KB per message. Mitigation: for large payloads (e.g., invoice PDFs), store in S3 and include the S3 key in the event.
- **At-least-once delivery**: messages may be delivered more than once. Mitigation: idempotent consumers with the processed_events table (see ADR-004 and failure-handling.md).

---

## Alternatives Considered

**Kafka (MSK)**: rejected for v1 due to higher operational complexity and cost. The platform does not require event replay or strict ordering at this stage. Kafka remains the right choice if the platform grows to require those features.

**RabbitMQ (Amazon MQ)**: rejected. No meaningful advantage over SNS+SQS for this use case, and requires broker management.

**Direct HTTP callbacks**: rejected. Synchronous callbacks between services create tight coupling and do not provide durability — if the consumer is down when the producer calls, the event is lost.
