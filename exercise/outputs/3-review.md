_Agent: Claude Code (Anthropic) · Model: Claude Opus 4.8 (`claude-opus-4-8`)_

_ATAM-style review of `docs/adr/0002-async-post-order-processing.md`. Read: that ADR and `docs/scenario.md`. The ADR was not edited._

# Review: ADR-0002 (async post-order processing)

## 1. Quality-attribute scenarios

| # | Source · Stimulus · Environment · Response · Measure |
|---|---|
| QA1 (Availability) | **Source:** MailFast · **Stimulus:** MailFast is down · **Environment:** 11.11 peak, 200 orders/s · **Response:** orders still commit; `OrderPlaced` events queue in RabbitMQ; emails send on recovery · **Measure:** **0 orders fail** due to email being down; backlog drains within minutes of recovery |
| QA2 (Performance) | **Source:** shopper · **Stimulus:** places an order · **Environment:** 200 orders/s peak · **Response:** `PlaceOrder` charges, ships, commits order+outbox, returns — without waiting for consumers · **Measure:** **p95 ≤ 500 ms** (today ≈ 300 ms) |
| QA3 (Correctness) | **Source:** RabbitMQ · **Stimulus:** `OrderPlaced` redelivered to loyalty · **Environment:** at-least-once delivery · **Response:** idempotent ledger keyed by order ID ignores the duplicate · **Measure:** points awarded **exactly once**, within **1 minute** |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | Payment and shipping run **before** the order/outbox commit (`main.go:252-273` flow is kept). If the process dies after `chargeCard`/`shipOrder` but before the outbox commit, the customer is **charged with no recorded order and no event** — the outbox protects the event→order link, not the charge→order link. | A charged customer with no order and no email during the 11.11 sale is the worst failure here; it is a correctness/availability gap the outbox does **not** close. |
| **Trade-off point** | Moving email/loyalty async improves availability and the 500 ms budget, but makes points **eventually consistent** — they may not exist when the confirmation page renders. | Availability/performance ↑ vs the marketing promise that the shopper sees points immediately. The ADR flags this only as an open question. |
| **Sensitivity point** | The p95 budget is sensitive to **whether checkout waits for the broker ack and to outbox-write latency**. If the relay publishes synchronously inside `PlaceOrder`, RabbitMQ latency re-enters the 500 ms budget. | A 500 ms target with ≈ 300 ms baseline leaves little headroom; one synchronous broker/db call can break it. |
| **Risk** | **Three new stateful components** (RabbitMQ cluster, outbox/order store, loyalty ledger) land on a **4-developer team** with no stated production broker experience and no monitoring/runbook in the ADR. | Operability risk at peak; "two have used RabbitMQ" is a thin basis for running a clustered broker under 200 orders/s. |
| **Non-risk** | Choosing **RabbitMQ, self-hosted on Kubernetes** with a queue to buffer the 200→50/s mismatch. | Fits the open-source/self-hosted constraint and the exactly-once requirement via idempotent consumers; sound. |

## 3. Missing or weak

- **Email backlog (150/s) is already calculated** in the ADR (200 − 50/s) and handled by the queue — correct, but it adds little new; it is the least interesting risk here.
- **Order persistence is under-specified:** today there is no order DB; the ADR introduces an outbox/order store but does not say where it lives, its durability, or backups.
- **No behaviour defined for loyalty/analytics being unavailable** beyond "queue backs up" — in particular the confirmation-page points display.
- **No access control / data-minimisation** on the event or queues (what personal data rides on `OrderPlaced`, who may read each queue).
- **No operational readiness gate:** dashboards, alerts, dead-letter handling and a runbook are mentioned as mitigations but not required before launch.

## 4. Verdict

**Accept with changes.** Three most important:

1. **Close the charge→order window:** commit the order (and outbox row) **before** charging, or make `chargeCard` idempotent/compensatable, so a crash cannot leave a charged customer without an order.
2. **Define the points/loyalty-unavailable behaviour**, including what the confirmation page shows (pending vs actual), so the design matches the campaign promise.
3. **Require operational readiness before 11.11:** RabbitMQ cluster + queue-depth/age alerts, dead-letter queues, backups and a written runbook, with named on-call ownership.
