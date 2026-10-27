Agent: Claude Code (Anthropic), model claude-opus-5-5

# ADR-R001: Use synchronous gRPC with one shared `demo.proto` contract for inter-service communication

- **Status:** Reconstructed
- **Date:** 2026-10-27 (date of reconstruction; original decision date Not evidenced in the provided files)
- **Deciders:** Not evidenced in the provided files (copyright headers name Google LLC: `demo.proto` line 1, `checkoutservice/main.go` line 1)

## Context and problem

Online Boutique is split into many services that must call each other to serve a page or place an order.

- One proto file, `system/online-boutique/protos/demo.proto`, defines 9 gRPC services in one package `hipstershop`: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`.
- Every RPC there is unary (request → response); none uses `stream`.
- Shared message types are reused across services: `Money` (Currency, Payment, Shipping, Checkout), `CartItem` (Cart, Shipping, Checkout), `Address` (Shipping, Checkout).
- `checkoutservice` orchestrates an order by calling 6 services one after another and blocking on each reply (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `prepOrderItems` in `src/checkoutservice/main.go`).
- `frontend` serves HTTP to users (`mux.NewRouter`, `http.ListenAndServe`) and opens gRPC clients to 7 backend services (`mustConnGRPC` calls, `src/frontend/main.go` lines 141–147).
- Services are written in more than one language? **Not evidenced in the provided files** (only two Go services were read).

Problem: which communication style and contract format should the services use?

## Decision drivers

1. One strongly typed, shared contract so that services in the system agree on message shapes (Inferred).
2. Simple request/response flow that matches a user waiting for a page or an order result (Inferred).
3. Observability: trace one request across many services (Evidenced: `otelgrpc` handlers in both files).
4. Ease of running the system as a demo / sample (Inferred).
5. Performance (compact binary encoding, HTTP/2) (Assumption).

## Considered options

1. **Synchronous gRPC (unary RPC) with one shared `demo.proto`** (chosen).
2. Synchronous REST/JSON over HTTP, with per-service OpenAPI or no formal contract.
3. Asynchronous messaging via a message broker (e.g. RabbitMQ or Kafka) for some or all interactions, e.g. order confirmation email.
4. Synchronous gRPC, but with one `.proto` file per service (separately versioned contracts).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC + one shared proto | One source of truth for all contracts; generated typed clients (`pb.New...Client`); shared types (`Money`, `Address`) used consistently; built-in health check protocol (`healthpb`); ready OTel instrumentation (`otelgrpc`) | Every caller blocks; failure of one callee fails the whole call chain; any change to the shared file touches all services' generated code; tight coupling of all teams to one file |
| 2. REST/JSON | Human-readable; easy to debug with curl/browser; no codegen needed | No compile-time typed contract; larger payloads; types like `Money` must be kept in sync by hand (Assumption) |
| 3. Async messaging | Decouples availability (e.g. checkout succeeds even if email is down); absorbs load spikes | Needs a broker to run and operate (cost, complexity); order result is not returned immediately; eventual consistency; harder tracing (Assumption) |
| 4. gRPC + per-service protos | Services can version contracts independently; clearer ownership | Shared types (`Money`, `Address`, `CartItem`) must be duplicated or put in a common file; more files to manage (Inferred) |

## Decision

Use synchronous unary gRPC for all service-to-service calls, with all contracts in one shared file `demo.proto` (package `hipstershop`). The frontend translates user HTTP requests into gRPC calls.

Why (reconstructed, mostly **[Inferred]**): for a demo system the priorities seem to be one easy-to-read contract, typed generated clients and end-to-end tracing. A synchronous chain also matches the checkout flow, where the user waits for an `OrderResult` that includes a payment transaction and a shipping tracking ID. The original rationale is Not evidenced in the provided files.

## Consequences

- **Positive:** One typed contract; mismatched message fields are caught at code generation / compile time (quality attribute: modifiability, interoperability).
- **Positive:** Distributed traces across all calls through `otelgrpc` client and server handlers and W3C `TraceContext` propagation (quality attribute: observability / testability).
- **Positive:** Simple, easy-to-follow control flow in `PlaceOrder` (quality attribute: understandability / modifiability).
- **Negative:** Checkout latency is the sum of sequential calls: GetCart, then GetProduct + Convert per cart item (N+1 pattern in `prepOrderItems`), GetQuote, Convert, Charge, ShipOrder, EmptyCart, SendOrderConfirmation (quality attribute: performance).
- **Negative:** Temporal coupling: if cart, catalog, currency, shipping or payment is down, `PlaceOrder` fails (quality attribute: availability).
- **Negative:** No partial-failure handling after payment: if `ShipOrder` fails after `Charge` succeeded, an error is returned with no compensation (quality attribute: reliability / consistency).
- **Negative:** Any change in the shared proto affects all services that generate code from it (quality attribute: modifiability, team independence).
- **Negative:** Connections use `insecure.NewCredentials()` (no TLS) (quality attribute: security).
- **Risks and mitigations:**
  - Cascading failures / hanging calls: the evidence shows no per-call deadlines or retries; `convertCurrency` even uses `context.TODO()`, dropping the caller's context. Mitigation: set deadlines, retries and circuit breakers (or a service mesh) **[Inferred]**.
  - Non-critical work done synchronously: the code already tolerates failure of `EmptyCart` (result ignored) and `SendOrderConfirmation` (only `log.Warnf`). Mitigation: move these to async messaging (option 3) **[Inferred]**.
  - Plaintext traffic: enable mTLS **[Inferred]**.

## Validation

- Load test `PlaceOrder` with carts of different sizes; measure p95/p99 latency growth with item count (N+1 calls).
- Fault-injection test: stop `paymentservice`, `shippingservice`, `emailservice` one at a time and check the checkout result.
- Check traces in the collector show one complete trace per checkout.
- Review whenever a new service is added to `demo.proto`.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | One proto file defines 9 services in package `hipstershop` | Evidenced | `system/online-boutique/protos/demo.proto` (`package hipstershop`; `service CartService` … `service AdService`) |
| 2 | All RPCs are unary; no streaming | Evidenced | `demo.proto`: no `stream` keyword in any `rpc` |
| 3 | `Money`, `CartItem`, `Address` are shared across services | Evidenced | `demo.proto` (`GetQuoteRequest`, `ChargeRequest`, `PlaceOrderRequest`, `OrderItem` use them) |
| 4 | checkoutservice calls 6 services synchronously in sequence | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `mustConnGRPC` x6, lines 118–123) |
| 5 | frontend serves HTTP and uses gRPC clients to 7 backends | Evidenced | `src/frontend/main.go` (`mux.NewRouter`, `http.ListenAndServe`, `mustConnGRPC` lines 141–147) |
| 6 | Both services use code generated from the shared proto | Evidenced | `src/checkoutservice/main.go` (`pb "…/checkoutservice/genproto"`, `pb.NewCartServiceClient`); `demo.proto` (`option go_package`) |
| 7 | How frontend talks to the shopping assistant (it has an address but no gRPC connection) | Inferred | `src/frontend/main.go` (`shoppingAssistantSvcAddr` mapped but no `mustConnGRPC` for it); protocol Not evidenced |
| 8 | "All" services (including those not read, and in other languages) use gRPC | Inferred | Only two Go services read; proto lists 9 services but implementations Not evidenced |
| 9 | Deciders / date of original decision | Inferred | Only Google LLC copyright headers; no ADR in evidence |
| 10 | Driver: a shared typed contract was a goal | Inferred | Suggested by single proto, not stated |
| 11 | Driver: request/response fits a user waiting for a result | Inferred | `PlaceOrderResponse` returns `OrderResult`; intent not stated |
| 12 | Driver: observability via OTel tracing | Evidenced | both files (`otelgrpc.NewServerHandler`, `otelgrpc.NewClientHandler`, `propagation.TraceContext`, `initTracing`) |
| 13 | Driver: demo simplicity | Inferred | Repo is called "microservices-demo" (`demo.proto` `go_package`); rationale not stated |
| 14 | gRPC/protobuf gives compact binary encoding and HTTP/2 | Assumption | General knowledge |
| 15 | REST/JSON pros and cons | Assumption | General knowledge |
| 16 | Async messaging pros and cons (broker cost, eventual consistency) | Assumption | General knowledge; no broker in evidence |
| 17 | Per-service protos would require duplicated or common shared types | Inferred | Follows from claim 3 |
| 18 | Options 2–4 were actually considered by the original team | Inferred | Not evidenced in the provided files |
| 19 | gRPC health checking protocol is used | Evidenced | `src/checkoutservice/main.go` (`healthpb.RegisterHealthServer`, `Check`) |
| 20 | Checkout latency = sum of sequential calls, with N+1 per item | Evidenced | `src/checkoutservice/main.go` (`prepOrderItems` loop: `GetProduct` + `convertCurrency` per item) |
| 21 | A failed downstream call fails `PlaceOrder` | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder` returns `status.Errorf` on cart/catalog/currency/shipping/payment errors) |
| 22 | No compensation if shipping fails after payment | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`: `chargeCard` then `shipOrder` returns `codes.Unavailable`, no refund call) |
| 23 | Shared proto change affects all services | Inferred | Follows from one file / one package; build process Not evidenced |
| 24 | Connections are not encrypted | Evidenced | both files (`mustConnGRPC`: `insecure.NewCredentials()`) |
| 25 | No per-call deadlines or retries; `context.TODO()` in currency call | Evidenced | `src/checkoutservice/main.go` (`convertCurrency`; other calls pass `ctx` with no timeout set) |
| 26 | Email and cart-emptying failures are tolerated | Evidenced | `src/checkoutservice/main.go` (`_ = cs.emptyUserCart`; `log.Warnf` on `sendOrderConfirmation`) |
| 27 | Mitigations (deadlines, circuit breakers, async email, mTLS) would help | Assumption | General industry practice |
