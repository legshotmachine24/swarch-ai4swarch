_Agent: Claude Code (Anthropic) · Model: Claude Opus 4.8 (`claude-opus-4-8`)_

_Run b — with context. Read: `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`._

# ADR-0002: Process post-order actions asynchronously with RabbitMQ and a transactional outbox

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Online Boutique architecture team

## Context and problem

Requirement **R-12**: after an order is placed, Online Boutique must (1) send the confirmation email through **MailFast** (replacing the mock `emailservice`), (2) award **loyalty points** via a new `loyaltyservice`, and (3) record an **"order placed" analytics event**. Marketing expects **more post-order actions next year** (SMS, fraud scoring, recommendations update).

Today `checkoutservice.PlaceOrder` charges payment, ships, then calls email **synchronously over gRPC in sequence**; if email fails it only logs a warning and the order still succeeds (`system/online-boutique/src/checkoutservice/main.go:252-277`). There is **no order datastore** — `PlaceOrder` builds an `OrderResult` in memory and returns it (`main.go:265-279`).

Facts and numbers (`docs/scenario.md`):

| Topic | Fact |
|---|---|
| Checkout latency | p95 of `PlaceOrder` must stay **≤ 500 ms** (today ≈ 300 ms) |
| Peak load | **200 orders/s** during the 11.11 sale |
| MailFast | p95 **1.5 s**, p99 **8 s**; ≈ **0.5 %** failures; rate limit **50 emails/s** |
| Loyalty points | awarded **within 1 minute**, **exactly once** |
| Availability | an order must **never fail** because email, loyalty or analytics is down |
| Team | 4 devs; strong in Go/gRPC; **two have used RabbitMQ** |
| Operations | runs on Kubernetes; prefers **open-source, self-hosted** over cloud-managed |

## Decision drivers

1. **Availability:** an order must never fail because email/loyalty/analytics is down.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms even though MailFast p95 is 1.5 s — so checkout cannot wait for MailFast.
3. **Exactly-once loyalty:** points awarded once, within 1 minute.
4. **Throughput mismatch:** 200 orders/s in vs MailFast's 50 emails/s out must be absorbed, not dropped.
5. **Modifiability:** future post-order actions should be new consumers, not checkout changes.
6. **Operability / cost:** open-source, self-hosted, Kubernetes; a tool two of four devs already know.

## Considered options

1. **Keep synchronous gRPC chaining** — extend the current pattern; checkout calls MailFast, loyalty and analytics inline.
2. **Asynchronous via RabbitMQ + transactional outbox** — checkout writes an `OrderPlaced` event to an outbox in the same transaction as the order; a relay publishes to RabbitMQ; independent consumers handle email, loyalty and analytics.
3. **Managed cloud pub/sub** (e.g. Google Pub/Sub, SNS/SQS) — same async shape, managed.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A. Synchronous gRPC chaining | No new infrastructure; strongly consistent | MailFast p95 1.5 s blows the 500 ms budget; email/loyalty outage can fail the order (violates availability); 50 emails/s rate limit throttles checkout; every new action re-touches checkout |
| B. RabbitMQ + outbox | Checkout returns without waiting for MailFast (performance); order survives consumer outages (availability); queue buffers the 200→50/s mismatch; new actions = new consumers (modifiability); open-source, self-hosted, team knows it | New stateful components (broker, outbox, loyalty store) to operate; eventual consistency; needs idempotency + dead-letter handling |
| C. Managed cloud pub/sub | Less to operate; scales easily | Conflicts with the "open-source, self-hosted" constraint; cloud lock-in and cost |

## Decision

Adopt **Option B**. `checkoutservice` keeps charging payment and shipping synchronously, then writes the order **and** an `OrderPlaced` event into a **transactional outbox** in one local transaction; a relay publishes outbox rows to **RabbitMQ**. Three consumers subscribe:

- **email relay** → calls MailFast, throttled to ≤ 50 emails/s, retries with back-off, dead-letters after N attempts;
- **loyaltyservice** → writes to an idempotent loyalty ledger keyed by order ID (exactly-once effect even on redelivery);
- **analytics** → appends the event to the marketing stream.

`PlaceOrder` returns success as soon as the order + outbox row are committed; it no longer waits for email/loyalty/analytics. RabbitMQ is open-source, self-hostable on Kubernetes, and two developers already know it.

## Consequences

- **Positive:** checkout no longer waits on MailFast's 1.5 s p95, protecting the ≤ 500 ms budget (performance).
- **Positive:** email/loyalty/analytics outages leave the queue backed up but never fail the order (availability).
- **Positive:** the queue absorbs the **200 − 50 = 150 orders/s** email backlog during peak and drains it after (scalability); at-least-once delivery + idempotent ledger gives exactly-once points (correctness).
- **Positive:** future actions (SMS, fraud, recommendations) are added as new consumers (modifiability).
- **Negative:** three new stateful things to run and monitor — RabbitMQ, the outbox store and the loyalty ledger — for a 4-person team (operability).
- **Negative:** points/email are eventually consistent; they are not guaranteed to exist at the moment the confirmation page renders (consistency vs the campaign promise).
- **Risks and mitigations:** broker is a single point of failure → run a RabbitMQ cluster with mirrored queues; poison messages → dead-letter queues + alerting; backlog never drains if MailFast stays slow → monitor queue depth and age.

## Validation

- Load-test at **200 orders/s** and confirm `PlaceOrder` p95 stays **≤ 500 ms**.
- Kill MailFast / loyalty / analytics in a chaos test and confirm **0 orders fail** and the backlog drains afterward.
- Replay the same `OrderPlaced` event and confirm points are awarded **exactly once** and within **1 minute**.

## Open questions

- Payment and shipping still happen **before** the order/outbox commit (following today's `main.go:252-273` order). Should the order be committed first, or payment made idempotent/compensatable, to close the "charged but no order" window?
- What should the confirmation page show for **points** if the loyalty consumer has not run yet (pending vs actual)?
- Where does the **outbox/order table** live (new DB? reuse an existing one?), and who owns its schema?
