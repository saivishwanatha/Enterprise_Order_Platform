# Problem Statement

## Background

Modern e-commerce and B2B procurement operations require a reliable, scalable order fulfillment pipeline. A single failed step — a payment that goes through but inventory never gets reserved, or a shipment created without a corresponding invoice — creates real financial and operational damage: customer disputes, manual reconciliation work, and loss of trust.

Monolithic order systems solve this with database transactions, but they break down under load, make independent team ownership impossible, and couple unrelated concerns (authentication, payments, shipping) into a single deployable unit that becomes a bottleneck for every team.

## The Problem

We need an order fulfillment platform that:

1. **Handles the full order lifecycle** — from a customer browsing products and placing an order, through payment capture, inventory reservation, shipment dispatch, invoice generation, and delivery confirmation — without any step being silently skipped or duplicated.

2. **Tolerates partial failures gracefully** — if payment fails after inventory is reserved, the reservation must be released automatically. If the notification service is down, the order must still complete. Failures in one area must not corrupt state in another.

3. **Scales independently** — during a flash sale, the inventory and payment services will be under far more load than the invoice or notification services. Each concern must be scalable on its own terms.

4. **Supports multiple teams working in parallel** — auth, payments, shipping, and catalog are owned by different teams. They need independent deployment pipelines, independent databases, and clear contracts between them.

5. **Provides full auditability** — every state change must be traceable. Operations teams need to answer "what happened to order X and when?" without joining across multiple systems manually.

6. **Recovers from failures without manual intervention** — failed orders must be retryable. Dead-lettered events must be visible and actionable. The system must self-heal where possible.

## Who Uses This System

| Actor | Needs |
|---|---|
| **Customer** | Register, log in, browse products, place orders, track order status, receive notifications |
| **Admin** | Manage product catalog, view all orders, trigger manual retries on failed orders |
| **Operations Team** | Monitor system health, inspect DLQs, view order timelines, investigate failures |
| **Internal Services** | Consume domain events to perform their slice of the fulfillment pipeline |

## What Success Looks Like

- A customer can place an order and receive a confirmation notification within seconds.
- A failed payment automatically releases inventory and notifies the customer — no manual intervention.
- Each service can be deployed independently without a coordinated release.
- An operations engineer can trace the full lifecycle of any order through structured logs and a timeline API.
- The system handles 500 concurrent orders without degradation.
- A new team can onboard to a single service without needing to understand the entire codebase.

## Out of Scope (v1)

- Multi-currency and international tax calculation
- Marketplace / multi-vendor support
- Real payment gateway integration (payment service will simulate capture/failure)
- Mobile push notifications (email and in-app only)
- Customer-facing returns and refunds UI (refund events will be handled internally)
