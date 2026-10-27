# ADR-0002: Process post-order actions asynchronously with RabbitMQ and a transactional outbox

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Poonyapat Jankrajang (architect), Online Boutique team

## Context and problem

Requirement **R-12**: after an order is placed, Online Boutique must (1) send the confirmation email through **MailFast** (replacing the mock `emailservice`), (2) award **loyalty points** via a new `loyaltyservice`, and (3) record an **"order placed" analytics event**. Marketing expects **more post-order actions next year** (SMS, fraud scoring, recommendations update), so the design must make adding actions cheap.

Today `checkoutservice.PlaceOrder` charges payment, ships, then calls email **synchronously over gRPC in sequence**; if email fails it only logs a warning and the order still succeeds (`system/online-boutique/src/checkoutservice/main.go:252-277`). There is **no order datastore**: `PlaceOrder` builds an `OrderResult` in memory and returns it (`main.go:265-279`).

Key facts (`docs/scenario.md`):

| Topic | Fact |
|---|---|
| Checkout latency | p95 of `PlaceOrder` must stay **≤ 500 ms** (today ≈ 300 ms) |
| Peak load | **200 orders/s** during the 11.11 sale |
| MailFast | p95 **1.5 s**, p99 **8 s**; ≈ **0.5 %** failures; rate limit **50 emails/s** |
| Loyalty points | awarded **within 1 minute**, **exactly once** (no double points) |
| Availability | an order must **never fail** because email, loyalty or analytics is down |
| Team | 4 devs; strong in Go/gRPC; **two have used RabbitMQ** |
| Operations | runs on Kubernetes; prefers **open-source, self-hosted** components |

## Decision drivers

1. **Availability:** an order must never fail because email/loyalty/analytics is down.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms while MailFast p95 is 1.5 s — checkout cannot block on MailFast.
3. **Exactly-once loyalty:** points awarded once, within 1 minute.
4. **Throughput mismatch:** 200 orders/s in vs MailFast's 50 emails/s out must be buffered.
5. **Modifiability:** future post-order actions should be new consumers, not checkout edits.
6. **Operability / cost:** open-source, self-hosted on Kubernetes; a tool part of the team already knows.

## Considered options

1. **Keep synchronous gRPC chaining** — checkout calls MailFast, loyalty and analytics inline, as it calls email today.
2. **Asynchronous via RabbitMQ + transactional outbox** — checkout commits the order and an `OrderPlaced` event to an outbox in one transaction; a relay publishes to RabbitMQ; independent consumers do email, loyalty and analytics.
3. **Managed cloud pub/sub** (e.g. Google Pub/Sub, SNS/SQS).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A. Synchronous gRPC chaining | No new infrastructure; strongly consistent | MailFast p95 1.5 s breaks the 500 ms budget; an email/loyalty outage can fail the order (breaks availability); the 50 emails/s cap throttles checkout; every new action re-touches checkout |
| B. RabbitMQ + outbox | Checkout returns without waiting on MailFast (performance); order survives consumer outages (availability); queue buffers the 200→50/s mismatch; new actions = new consumers (modifiability); open-source, self-hosted, two devs know it | New stateful components (broker, outbox, loyalty store) to operate; eventual consistency; needs idempotency + dead-letter handling |
| C. Managed cloud pub/sub | Less to operate; elastic scale | Conflicts with the open-source/self-hosted constraint; cloud lock-in and recurring cost |

## Decision

Adopt **Option B — RabbitMQ with a transactional outbox.** `checkoutservice` charges payment and ships (as today), then writes the **order row and an `OrderPlaced` event into an outbox in one local transaction**; a relay publishes committed outbox rows to **RabbitMQ**. Three consumers subscribe to the `OrderPlaced` event:

- **email relay** → calls MailFast, self-throttled to ≤ 50 emails/s, retries with back-off, dead-letters after repeated failure;
- **loyaltyservice** → writes to an **idempotent loyalty ledger keyed by order ID**, so redelivery cannot double-award (exactly-once effect);
- **analytics** → appends the event to the marketing stream.

`PlaceOrder` returns success as soon as the order + outbox row commit; it no longer waits on email/loyalty/analytics. RabbitMQ is chosen over Kafka because it is lighter to self-host on Kubernetes for this volume and two of four developers already know it.

## Consequences

- **Positive:** checkout no longer waits on MailFast's 1.5 s p95, protecting the ≤ 500 ms budget (**performance**).
- **Positive:** an email/loyalty/analytics outage backs up the queue but never fails the order (**availability**).
- **Positive:** the queue absorbs the **200 − 50 = 150 orders/s** email backlog at peak and drains it afterward (**scalability**); at-least-once delivery + idempotent ledger yields exactly-once points (**correctness**).
- **Positive:** future actions (SMS, fraud, recommendations) are new consumers, not checkout changes (**modifiability**).
- **Negative:** three new stateful components — RabbitMQ, the outbox/order store and the loyalty ledger — for a 4-person team (**operability**).
- **Negative:** points and email are **eventually consistent**; they may not exist when the confirmation page renders (**consistency** vs the campaign promise).
- **Risks and mitigations:** broker is a single point of failure → RabbitMQ cluster with mirrored queues; poison messages → dead-letter queues with alerting; backlog never draining if MailFast stays slow → alert on queue depth/age.

## Validation

- Load-test at **200 orders/s** and confirm `PlaceOrder` p95 stays **≤ 500 ms**.
- Chaos-test MailFast / loyalty / analytics down and confirm **0 orders fail** and the backlog drains after recovery.
- Replay the same `OrderPlaced` event and confirm points are awarded **exactly once** within **1 minute**.
- Review date: two weeks before 11.11.

## Open questions

- Payment and shipping still run **before** the order/outbox commit (following today's `main.go:252-273`). Should the order commit first, or payment be made idempotent/compensatable, to close the "charged but no order recorded" window?
- What should the confirmation page show for **points** if the loyalty consumer has not yet run (pending vs actual total)?
- Where does the **outbox/order store** live, and who owns its schema and backups?
