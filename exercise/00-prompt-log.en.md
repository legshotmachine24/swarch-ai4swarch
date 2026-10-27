**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Poonyapat Jankrajang
**Agent(s) used:** Claude Code (CLI, agent mode)
**Model(s):** Claude Opus 4.8 (`claude-opus-4-8`)
**Plan / tier:** Claude Code

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code / Opus 4.8 | `prompts/1-recover-architecture.md` run A, evidence `protos/demo.proto` | `prompts/1-recover-architecture.md`, `protos/demo.proto` | Yes | 4 | From the proto alone it got 9 services but **no dependencies, no frontend, no data store** — correctly left the diagram edges empty and listed the rest under Gaps |
| 2 | 1-B | Claude Code / Opus 4.8 | `prompts/1-recover-architecture.md` run B, evidence `protos/demo.proto` + `kubernetes-manifests/` | `protos/demo.proto`, all 11 `kubernetes-manifests/*.yaml`, `kustomization.yaml`, `README.md` | Yes | 5 | Recovered 17 dependencies from `*_SERVICE_ADDR` env vars + `redis-cart`. Caught that `shoppingassistantservice` is referenced but not deployed |
| 3 | 2-a | Claude Code / Opus 4.8 | `prompts/2-write-adr.md` run a, context `none`, requirement = R-12 | `docs/adr/template.md`, R-12 text only | Yes | 2 | Generic: no scenario numbers, long Open-questions list. Correctly did **not** invent numbers. Confirmed it did not open `docs/scenario.md` |
| 4 | 2-b | Claude Code / Opus 4.8 | `prompts/2-write-adr.md` run b, context scenario + ADR-0001 + checkoutservice | `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go` | Yes | 5 | Used every scenario number; spotted there is no order datastore in `main.go`; matched ADR-0001's style |
| 5 | 3 | Claude Code / Opus 4.8 | `prompts/3-review-adr.md`, adr `docs/adr/0002-async-post-order-processing.md` | `docs/adr/0002-async-post-order-processing.md`, `docs/scenario.md` | Yes | 4 | ATAM review; did not edit the ADR. Best finding: payment/shipping before the durable order commit |
| 6 | bonus | Claude Code / Opus 4.8 | `prompts/bonus-recover-decision.md`, decision = synchronous gRPC | `docs/adr/template.md`, `protos/demo.proto`, `src/checkoutservice/main.go`, `src/frontend/main.go` | Yes | 4 | Reconstructed ADR with a Claim check; rationale honestly marked Inferred/Assumption |
| 7 | | | | | | | |
