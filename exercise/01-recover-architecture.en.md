**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| frontend → checkoutservice | `frontend.yaml` `CHECKOUT_SERVICE_ADDR=checkoutservice:5050` | ✅ | Found at `frontend.yaml:79-80`. Source confirms it: `src/frontend/main.go:136` maps the env var and `:146` dials it over gRPC. |
| checkoutservice → paymentservice | `checkoutservice.yaml` `PAYMENT_SERVICE_ADDR=paymentservice:50051` | ✅ | Found at `checkoutservice.yaml:61-62`. Source confirms a real call: `src/checkoutservice/main.go:369-377` `chargeCard` → `PaymentServiceClient.Charge`. |
| checkoutservice → emailservice | `checkoutservice.yaml` `EMAIL_SERVICE_ADDR=emailservice:5000` | ✅ | Found at `checkoutservice.yaml:63-64`. Port 5000 matches `emailservice.yaml:82` (Service port 5000 → container 8080). Source: `main.go:379-384` `sendOrderConfirmation`. |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` `PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550` | ✅ | Found at `recommendationservice.yaml:60-61`. (Recommendationservice source not read, but the env var + the catalog's grpc port 3550 are consistent.) |
| cartservice → redis-cart | `cartservice.yaml` `REDIS_ADDR=redis-cart:6379` | ✅ | Found at `cartservice.yaml:50-51`; `redis-cart` Deployment + Service are in the same file (`:92,146`). Protocol is Redis-over-TCP, not gRPC — correctly labelled. |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question):
  - `frontend.yaml:83-84` sets `SHOPPING_ASSISTANT_SERVICE_ADDR=shoppingassistantservice:80`, but **no manifest deploys `shoppingassistantservice`** and it is not in `kustomization.yaml`. Run B correctly left it out of the diagram and flagged it under Gaps — a naive recovery would have drawn a dangling dependency.
  - **Protocols are inferred, not proven by the manifests.** The `*_SERVICE_ADDR` vars prove the dependency exists; "gRPC" comes from the callee's grpc probe/port + `demo.proto`. Verifying against the caller source (`main.go`) is what actually confirms the call is made.
  - `loadgenerator` is commented out in `kustomization.yaml:24`, so in the kustomize build the only driver of `frontend` is the external `User` — worth questioning whether `loadgenerator` belongs in the running system diagram.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
   - From the `.proto` alone the agent could recover the **9 service interfaces** but **none of the wiring**: no caller→callee edges, no frontend, no `redis-cart` data store, and no implementation languages. A `.proto` is an interface contract — it says what each service *offers*, never who *calls* it or how it is deployed. The runtime call graph only appears in Run B, encoded in the `*_SERVICE_ADDR` / `REDIS_ADDR` environment variables in the Kubernetes manifests.
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
   - Inventing a **call graph from shared message types**. In Run A it is tempting to draw `checkoutservice → payment/shipping/email` because they share `Money`, `Address`, `OrderResult` — but the proto never shows those calls. A newcomer who trusted such a diagram would believe dependencies that aren't evidenced (and could still be wrong in Run B, e.g. assuming the dangling `shoppingassistantservice` is a live dependency). Reusing a type is not the same as making a call.
