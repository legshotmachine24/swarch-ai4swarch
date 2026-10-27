**Language:** [ภาษาไทย](bonus-recover-decision.md) | English

# Bonus (optional, take-home): Recover an unwritten decision (approximately 15 min)

> **Lecture link:** T5 (Dhar et al. generate decisions from code and diagrams to reduce *knowledge vaporization*), AICH4.
> **Question:** when AI explains *why* a system was built this way, how much of the explanation is evidence and how much is a confident guess?

## Run

Start a new chat or session in your agent and send:

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
```

Want the ADR in Thai? Add a line `language: Thai`.

The agent writes the reconstructed ADR, including its **Claim check** table, to [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md).

## Audit the claim check

Choose **five** claims from the agent's table: at least two it marked *Evidenced* and two it marked *Inferred*. Check each one yourself.

| # | Claim (short) | AI says | You say | Why (file path or reasoning) |
|---|---|---|---|---|
| 1 | All service interfaces live in one shared proto | Evidenced | ✅ Agree | `protos/demo.proto:17` `package hipstershop` — all 9 services are in that one file |
| 2 | `checkoutservice` calls payment→shipping→email synchronously in sequence | Evidenced | ✅ Agree | `src/checkoutservice/main.go:252,258,273` — plain sequential calls, each blocking |
| 3 | A failed email does not fail the order; other failures do | Evidenced | ✅ Agree | `main.go:273-277` email path only `log.Warnf`; `:252-261` payment/shipping return errors |
| 4 | gRPC was chosen **for** type-safe cross-language contracts + low latency | Inferred | ✅ Agree it's Inferred | Reasonable, but no file states *why*; it's a guess from the single-proto + polyglot shape |
| 5 | Synchronous (not async) was chosen because it's simpler for an interactive shop | Inferred | ✅ Agree it's Inferred | The code shows sync calls, but the motivation ("because simpler") is nowhere in the evidence |

**Count:** of all the agent's claims, how many are *Evidenced* according to the agent? **8 of 12** (claims 1–8). According to you? **8** — the eight evidenced claims all check out against the cited files; the other four are correctly marked Inferred/Assumption.

## Reflect

1. Did the agent ever label a guess as *Evidenced*? Give an example, or explain how you checked that it didn't.
   - No. I checked every "Evidenced" row by opening the cited file and line. Each one points at something real in the code (e.g. the sequential calls at `main.go:252,258,273`). Every claim about *why* the decision was made — e.g. "gRPC chosen for type safety and low latency" — was marked **Inferred**, not Evidenced. So the agent kept the what/why line honest.
2. A new developer reads only this reconstructed ADR. What wrong belief could they form?
   - That the synchronous-gRPC choice was a **deliberate, documented decision** with the rationale listed — when in fact the "why" is reconstructed guesswork. They might also over-generalise "a failed email doesn't fail the order" into a system-wide resilience pattern, when the code shows it is a one-off for email only (`main.go:273-277`).
3. What can AI recover from code, and what can only the original architects know? Use one example from your ADR.
   - AI recovers **structure and behaviour** — the shared `demo.proto`, which service calls which, the exact call order, the insecure in-cluster transport. It **cannot** recover the **motivation**: whether gRPC was chosen over REST for performance, for cross-language contracts, or simply because the template did. That trade-off (claim 4) lives only with the original architects; the code shows the choice, never the reason.
