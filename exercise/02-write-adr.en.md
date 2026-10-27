**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | **No** — none were available; all put under Open questions (latency budget, peak, MailFast limits, exactly-once window) | **Yes** — p95 ≤ 500 ms, 200 orders/s, MailFast 50 emails/s / 1.5 s p95 / 0.5 % fail, points within 1 min |
| Options considered (list them) | 1) Synchronous calls from checkout 2) Async via a message broker 3) Post-order orchestrator service | 1) Keep synchronous gRPC chaining 2) RabbitMQ + transactional outbox 3) Managed cloud pub/sub |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | **No specifics** — named idempotency in the abstract but flagged exactly-once and MailFast limits as Open questions (no numbers) | **Yes** — idempotent loyalty ledger keyed by order ID (exactly-once under at-least-once delivery); email relay throttled to ≤ 50/s, queue buffers the 200 − 50 = **150 orders/s** backlog, drains after peak |
| Anything invented (a number, tool or fact not in the scenario)? | **No** — kept all unknowns in Open questions instead of guessing | **No** — used only scenario facts; the one inference (order/outbox store location) is flagged as an Open question, and RabbitMQ is from the scenario's team-skills fact |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

> Final ADR: [`docs/adr/0002-async-post-order-processing.md`](../docs/adr/0002-async-post-order-processing.md)

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Committed the broker choice to **RabbitMQ over Kafka** (the 2b draft left "which broker" softer) and said *why* — lighter to self-host on K8s at this volume and two of four devs know it | The architect must make the call, not leave it open; it ties the choice to the team-skills and self-hosted facts |
| 2 | Made the **outbox transactional with the order row** and the loyalty ledger **idempotent keyed by order ID**, stated explicitly as the exactly-once mechanism | Turns "exactly once" from a goal into a concrete design; at-least-once + idempotency is the honest way to get it |
| 3 | Added the explicit decision that **`PlaceOrder` returns as soon as order+outbox commit** (does not wait on email/loyalty/analytics), plus a validation/review date | Makes the ≤ 500 ms performance claim testable and ties it to the availability requirement |

> Note (take-ownership): I deliberately **kept** the open question that payment/shipping still run before the order commit. It is a real weakness I want the review board to debate rather than hide — and the AI review flagged it.
