**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | the ADR's author | defends the ADR |
| **AI analyst** | | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | | role card P (business side) |
| **Site reliability engineer (SRE)** | | role card S (operations side) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-async-post-order-processing.md` (R-12 post-order design)
**Board members and roles:**
- Ratsaporn — Architect
- Weeraphat — AI analyst
- Poonyapat — Product owner (role card P)
- Boonchai — Site reliability engineer (role card S)
- Rachata — Security & privacy officer (role card X)

**Decision:** **accept with changes** (Architect and AI analyst voted accept; PO, SRE and Security voted accept with changes)

**Required changes:**

*Product owner (business):*
1. Update the design and acceptance criteria so the confirmation page shows the **actual, saved points total** — "points pending" or "within one minute" does not meet the campaign promise.
2. Define what happens when **loyalty is unavailable**, including the conflict with keeping checkout available.

*SRE (operations):*
3. Correct the experience assumption: the two developers used RabbitMQ **only on laptops**; production broker operations are new to this team, with only two people on call.
4. Make **operational readiness a release gate** — require monitoring dashboards, alerting, backups and a written runbook before introducing new stateful components (allow ≈ one week).
5. Assign **incident ownership**: name who receives alerts for broker, relay, consumer and storage failures, with escalation and recovery procedures.

*Security & privacy officer:*
6. Define **separate message schemas** for email, loyalty and analytics; document the personal data in each message; do not distribute customer details to every consumer.
7. Exclude **email addresses and full names from analytics**; assess whether order IDs or other identifiers could re-identify customers through linkage.
8. Specify **access permissions for each queue** (including retry and dead-letter queues); grant services and operators only the access they need.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Duplicate points cause financial loss; missing points cause complaints and break the marketing promise. Adding storage, outbox, RabbitMQ and a loyalty ledger creates delivery risk for a four-person team. The ADR has a testing plan but no results showing readiness. | Product owner | Yes (partly) | Yes — the financial loss from double points and the eventual-consistency gap are visible in the ADR; the AI flagged the points trade-off. But it could not weigh the *business promise* the way the PO can. |
| 2 | On-call load: broker, outbox and extra storage are substantial new responsibilities for a two-person rotation. | SRE | Yes | Partly — the AI flagged "three new stateful components for a 4-dev team", but it could not know the real on-call rotation or that the RabbitMQ experience was laptop-only; that lives in people's heads, not the files. |
| 3 | Unnecessary exposure: a shared event could distribute personal data to consumers that do not need it. | Security | Yes | Partly — the AI noted no data-minimisation/access control, but it could not apply the company's privacy policy or judge linkage risk without that context. |
| 4 | Payment can succeed before the order is durably recorded (charged-but-no-order window); the outbox does not protect this gap. | AI analyst | Yes | Yes — this is a pure code/ADR fact; the AI read `main.go` order (charge → ship → commit) and caught it. |
| 5 | Delivery guarantees need a clear scope (what exactly-once covers, and where the order is persisted). | Architect | Yes | Yes — the ADR's own Open questions raise it; the AI echoed it, but the architect sets the actual scope. |

**The AI's best finding** (one the humans had missed): Payment and shipping happen **before** the durable order commit, creating a window where a customer is charged with no recorded order. The outbox does **not** protect this gap.

**The AI's weakest finding** (generic, wrong or irrelevant): The **email capacity mismatch** (200 vs 50 emails/s). It is valid, but the ADR already explicitly calculates the 150/s backlog and names the follow-up, so the review added little new insight.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
