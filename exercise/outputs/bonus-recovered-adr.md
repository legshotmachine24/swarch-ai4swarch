_Agent: Claude Code (Anthropic) · Model: Claude Opus 4.8 (`claude-opus-4-8`)_

_Reconstructed decision. Read: `docs/adr/template.md`, `system/online-boutique/protos/demo.proto`, `system/online-boutique/src/checkoutservice/main.go`, `system/online-boutique/src/frontend/main.go`._

# ADR-0003: Communicate between services synchronously over gRPC with one shared `demo.proto`

- **Status:** Reconstructed
- **Date:** 2026-10-27
- **Deciders:** Unknown (reconstructed from code; no original ADR exists)

## Context and problem

Online Boutique is a set of independently deployed microservices that must call one another to serve a page view and a checkout. A reconstruction must explain **why every service talks over synchronous gRPC defined by a single shared protobuf contract**, rather than REST/JSON or asynchronous messaging.

The evidence shows one `.proto` file, `system/online-boutique/protos/demo.proto`, declaring **all** service interfaces (`CartService`, `ProductCatalogService`, `CurrencyService`, `ShippingService`, `PaymentService`, `EmailService`, `CheckoutService`, `RecommendationService`, `AdService`) in a single `package hipstershop`. Both `frontend` and `checkoutservice` import generated stubs from a `genproto` package and open gRPC client connections to their dependencies.

## Decision drivers

- **Cross-language contracts:** services are written in different languages but must share one interface definition.
- **Low latency on the request path:** calls happen on page views and checkout.
- **Simplicity of a request/response model** for an interactive shop.

## Considered options

1. **Synchronous gRPC with one shared `demo.proto`** (the observed choice).
2. **REST/JSON over HTTP** between services.
3. **Asynchronous messaging** (a broker/event bus) between services.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| gRPC + shared proto | One typed contract across languages; compact HTTP/2 binary; streaming + health + interceptors built in | Shared `.proto` is a coupling point; binary is less browser/debug-friendly |
| REST/JSON | Ubiquitous, human-readable, easy to debug | No enforced cross-language schema; more boilerplate; larger payloads |
| Async messaging | Decoupling, resilience to downstream outages | Harder for the synchronous request/response of a page render; more infrastructure |

## Decision

Use **synchronous gRPC** for all inter-service calls, with **one shared `demo.proto`** as the single source of truth for every service interface, and generate language-specific stubs from it.

## Consequences

- **Positive:** a single typed contract keeps polyglot services in agreement (modifiability/interoperability).
- **Positive:** compact HTTP/2 transport suits the per-request call path (performance).
- **Negative:** synchronous chaining couples availability — if a downstream service is down, the caller's request fails unless it handles the error (e.g. `checkoutservice` logs and continues only for email, `main.go:273-277`).
- **Negative:** the shared `demo.proto` is a change-coupling point across all services.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | All service interfaces are defined in one shared `.proto` file under one package | **Evidenced** | `protos/demo.proto:17` (`package hipstershop`) declares `CartService`, `ProductCatalogService`, `CurrencyService`, `ShippingService`, `PaymentService`, `EmailService`, `CheckoutService`, `RecommendationService`, `AdService` |
| 2 | Services communicate over gRPC | **Evidenced** | `checkoutservice/main.go:210-219` (`grpc.NewClient`), `:142` (`RegisterCheckoutServiceServer`); `frontend/main.go:141-147` (`mustConnGRPC`) |
| 3 | `checkoutservice` calls payment, shipping, email, cart, catalog, currency **synchronously, in sequence** | **Evidenced** | `checkoutservice/main.go:252` (`chargeCard`), `:258` (`shipOrder`), `:273` (`sendOrderConfirmation`); clients `pb.NewXServiceClient` at `:314,325,341,360,370,380,387` |
| 4 | `frontend` opens gRPC connections to seven backend services | **Evidenced** | `frontend/main.go:141-147` (currency, productcatalog, cart, recommendation, shipping, checkout, ad) |
| 5 | Stubs are generated from the shared proto (`genproto`) | **Evidenced** | `checkoutservice/main.go:34` (`pb "…/checkoutservice/genproto"`) |
| 6 | gRPC runs over HTTP/2 with insecure (plaintext) credentials in-cluster | **Evidenced** | `checkoutservice/main.go:215` & `frontend/main.go:230` (`insecure.NewCredentials()`) |
| 7 | Calls are traced with OpenTelemetry gRPC interceptors | **Evidenced** | `checkoutservice/main.go:139,216` & `frontend/main.go:147,231` (`otelgrpc` handlers) |
| 8 | A failed email call does not fail the order; other failures do | **Evidenced** | `checkoutservice/main.go:273-277` (email: `log.Warnf` and continue) vs `:252-261` (payment/shipping return an error) |
| 9 | gRPC was chosen **for** type-safe cross-language contracts and low latency | **Inferred** | Consistent with a single proto + polyglot services, but no file states the rationale |
| 10 | Synchronous request/response was chosen **because** it is simpler for an interactive shop than async messaging | **Inferred** | Fits the code shape; the motivation is not recorded anywhere in the evidence |
| 11 | gRPC/protobuf is generally preferred over REST/JSON for internal polyglot microservice RPC | **Assumption** | General industry knowledge, not specific to this repo |
| 12 | Binary protobuf reduces payload size and CPU vs JSON | **Assumption** | General industry knowledge, not shown in these files |

> Honest note: claims **1–8 are directly evidenced** in the files; the **rationale** (claims 9–12) is **Inferred/Assumption** — the code shows *what* was built, not *why*. No original ADR exists.
