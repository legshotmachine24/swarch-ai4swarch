_Agent: Claude Code (Anthropic) · Model: Claude Opus 4.8 (`claude-opus-4-8`)_

_Run a — zero-shot. Read only `docs/adr/template.md` and the requirement text below. No scenario, code or example ADR was read._

# ADR-0002: Process post-order actions asynchronously via a message broker

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Online Boutique architecture team

## Context and problem

Requirement **R-12**: after an order is placed, the system must (1) send the order-confirmation email through a new external provider, **MailFast**, which replaces the mock `emailservice` logic; (2) award **loyalty points** through a new `loyaltyservice` that has yet to be built; and (3) record an **"order placed" analytics event** for marketing. Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

Today these actions would be triggered from the checkout flow. The open problem is **how to attach a growing list of post-order side-effects to an order** without the design having to change every time a new action is added, and without one slow or failing side-effect harming the others or the order itself.

> The requirement text alone does not give latency budgets, peak load, MailFast's limits, the exactly-once rule for points, or the team's skills and infrastructure. Those are listed under **Open questions** rather than guessed at.

## Decision drivers

- **Modifiability / extensibility:** marketing has explicitly said more post-order actions are coming; adding one should not require changing the others.
- **Fault isolation:** a failure in email, loyalty or analytics should not cascade.
- **Decoupling:** post-order consumers should not need to know about each other.

## Considered options

1. **Synchronous calls from checkout** — checkout calls email, loyalty and analytics in sequence (as the current email call is made).
2. **Asynchronous messaging via a message broker** — checkout publishes one "order placed" event; each action is an independent consumer.
3. **A post-order orchestrator service** — a new service that receives the order and calls each action, still synchronously.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A. Synchronous from checkout | Simplest to build; no new infrastructure | Each new action re-couples to checkout; a slow/failed action delays or breaks checkout; poor fit for "more actions next year" |
| B. Async via message broker | New actions are just new consumers (strong modifiability); failures isolated per consumer; natural buffering | New infrastructure to run and operate; eventual consistency; needs idempotency to avoid duplicates |
| C. Post-order orchestrator | Keeps checkout thin; one place to see the flow | Still synchronous and coupled inside the orchestrator; orchestrator becomes a new single point of failure |

## Decision

Adopt **Option B: publish a single "order placed" event to a message broker, with one independent consumer per post-order action** (email→MailFast, loyalty, analytics). New post-order actions (SMS, fraud scoring, recommendations) are added as new consumers without changing checkout or the existing consumers. This directly satisfies the stated driver that more actions are expected.

## Consequences

- **Positive:** adding a future action is a new consumer, not a change to checkout (modifiability).
- **Positive:** a failing or slow consumer does not block the others or the order (availability / fault isolation).
- **Negative:** a message broker is new infrastructure to run, monitor and secure (operability).
- **Negative:** processing becomes eventually consistent; points/emails arrive after the order, not during it (consistency) — acceptable only if the business allows it (see Open questions).
- **Risks and mitigations:** duplicate delivery can award points twice or send two emails — mitigate with idempotent consumers; the broker is a single point of failure — mitigate with clustering. These cannot be fully specified without the numbers below.

## Validation

Once the missing numbers are known: load-test at the real peak order rate and confirm the checkout latency budget still holds; verify loyalty points are awarded exactly once under retries; confirm an order still succeeds when a consumer (email/loyalty/analytics) is down.

## Open questions

- What is the **checkout latency budget**, and may the order complete before these actions finish?
- What is the **peak order rate** the design must sustain?
- What are **MailFast's** throughput limit, latency and failure rate?
- Must loyalty points be **exactly once**, and within what time window?
- Is it a hard rule that an order **never fails** because email/loyalty/analytics is down?
- What are the **team's skills** and the **infrastructure constraints** (self-hosted vs managed, which broker)?
- Does an order need to be **durably stored** before these events are emitted?
