> Drafted by: Claude Code (Anthropic), model claude-opus-5-5. Final decisions (outbox, email backlog, broker HA) by Chayutphong Soisri. Run b (with context: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`).

# ADR-0002: Publish an "order placed" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Chayutphong Soisri (architect), with the Online Boutique dev team (4 developers).

## Context and problem

Requirement R-12 (`docs/scenario.md`) adds three actions after an order is placed: (1) send the confirmation email through **MailFast**, an external provider that replaces the mock `emailservice` logic; (2) award loyalty points through a new `loyaltyservice`; (3) record an "order placed" analytics event. Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

How it works today (evidence):

- `checkoutService.PlaceOrder` runs everything in one synchronous gRPC request: `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`, lines 230-280).
- An email failure only logs a warning and the order still succeeds (`main.go` lines 273-277, `log.Warnf("failed to send order confirmation ...")`).
- Each downstream service is a fixed gRPC client wired by an env var (`mustMapEnv(&svc.emailSvcAddr, "EMAIL_SERVICE_ADDR")`, `main.go` line 115; struct `checkoutService`, lines 66-86). Adding an action today means changing `checkoutservice` code, config and deployment.
- `checkoutservice` has no data store of its own: the order ID is a fresh UUID (`uuid.NewUUID()`, line 233) and the order is only returned in the response (lines 265-279). No order persistence is evidenced in the provided files.

Facts that shape the decision (`docs/scenario.md`, "Quality goals and constraints"):

| Fact | Value | Why it matters |
|---|---|---|
| `PlaceOrder` p95 budget | ≤ 500 ms (today approximately 300 ms) | Only approximately 200 ms of headroom |
| Peak load | 200 orders/s (11.11 sale) | Sets the event rate |
| MailFast latency | p95 1.5 s, p99 8 s | A synchronous call alone breaks the 500 ms budget |
| MailFast reliability | approximately 0.5 % of calls fail | Approximately 1 failed email/s at peak; needs retries |
| MailFast rate limit | 50 emails/s | At 200 orders/s, approximately 150 emails/s pile up; needs buffering |
| Loyalty points | within 1 minute, **exactly once** | Needs durable delivery plus de-duplication |
| Availability | an order must never fail because email, loyalty or analytics is down | Post-order actions must be decoupled from the order |
| Team | 4 devs, strong Go/gRPC, 2 have used RabbitMQ | Favours tech the team knows |
| Operations | Kubernetes; prefers open-source, self-hosted | Rules out cloud-only managed brokers |

The problem: **how should `checkoutservice` trigger post-order actions so that checkout stays fast and available, MailFast's slowness and rate limit are absorbed, loyalty points are awarded exactly once, and new actions can be added without changing checkout?**

## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 must stay ≤ 500 ms at 200 orders/s.
3. **Reliability / correctness:** loyalty points within 1 minute, exactly once; emails must survive MailFast failures (0.5 %) and its 50 emails/s rate limit.
4. **Modifiability (extensibility):** SMS, fraud scoring and recommendations next year should be added without editing `checkoutservice`.
5. **Team fit and operability:** Go/gRPC skills, RabbitMQ experience, open-source and self-hosted on Kubernetes.

## Considered options

1. **Synchronous gRPC calls in `PlaceOrder`** (extend today's pattern: call `emailservice`→MailFast, `loyaltyservice` and an analytics service one after another).
2. **In-process asynchronous calls** (`checkoutservice` starts goroutines that call each post-order service after replying, with in-memory retries).
3. **Publish an `OrderPlaced` event to a self-hosted RabbitMQ broker**; each post-order action is an independent consumer with its own queue.
4. **Publish an `OrderPlaced` event to a self-hosted Apache Kafka cluster**; each action is a consumer group.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC | Simplest; matches current code; team knows gRPC; immediate errors | MailFast p95 1.5 s / p99 8 s breaks the 500 ms budget; 50 emails/s limit cannot serve 200 orders/s; each new action adds latency and a new failure point; `checkoutservice` must change for every new action |
| 2. In-process async (goroutines) | No new infrastructure; checkout latency unaffected; quick to build in Go | Work is lost if a `checkoutservice` pod restarts (no durability), so "exactly once within 1 minute" cannot be guaranteed; retries and rate limiting live in memory per replica; still couples checkout to every action (poor modifiability) |
| 3. RabbitMQ events | Checkout only publishes one message (low added latency); durable queues buffer the MailFast backlog and retries; each consumer fails independently; new actions = new queue bound to the exchange, no checkout change; open-source, runs on Kubernetes; two devs know it | New stateful component to run and monitor; at-least-once delivery means consumers must de-duplicate; publish and order are not atomic (event can be lost if publish fails after payment) |
| 4. Kafka events | Durable, replayable log (useful for analytics and future consumers); very high throughput | Heavier to operate (brokers, partitions; more ops work for a 4-person team); nobody on the team has used it (not evidenced); throughput far beyond 200 msg/s; still at-least-once, so the same de-duplication is needed |

## Decision

Choose **option 3: publish an `OrderPlaced` event to a self-hosted RabbitMQ broker** on the existing Kubernetes cluster.

- After `shipOrder` succeeds, `PlaceOrder` writes the order and an `OrderPlaced` outbox row (order ID, email, items, totals) to a new **order store in one local transaction**, then returns. It no longer calls `emailservice` directly (replaces `sendOrderConfirmation`, `main.go` lines 273-277).
- An **outbox relay** in `checkoutservice` reads unsent outbox rows and publishes them as persistent messages to a **fanout/topic exchange** with publisher confirms, marking each row sent only after the broker confirms. If RabbitMQ is down, rows wait in the outbox and are published when it recovers, so no event is lost and the order still succeeds.
- Each post-order action has its own durable queue and consumer:
  - `emailservice` → MailFast, with a consumer rate limit ≤ 50 emails/s (prefetch/throttle), retry with back-off, and a dead-letter queue.
  - `loyaltyservice` → awards points **idempotently**, keyed by order ID (unique constraint on order ID in its store), giving exactly-once *effect* on top of at-least-once delivery.
  - an analytics consumer → records the "order placed" event.
- Next year's actions (SMS, fraud scoring, recommendations) are added as new queues bound to the same exchange.

Why: it is the only option that meets drivers 1-4 together. It keeps the slow MailFast call out of the checkout path (performance), lets each action fail on its own (availability), buffers the 150 emails/s surplus at peak (reliability), and makes new actions additive (modifiability). RabbitMQ beats Kafka on driver 5: the team has experience with it and it is lighter to run at 200 msg/s.

## Consequences

- **Positive:** `PlaceOrder` latency no longer depends on MailFast or the other actions; only one publish is added (quality attribute: performance).
- **Positive:** email, loyalty or analytics outages do not fail orders; messages wait in their queues until the consumer recovers (quality attribute: availability).
- **Positive:** new post-order actions are new consumers; `checkoutservice` no longer needs a gRPC client per action (quality attribute: modifiability).
- **Positive:** queues absorb MailFast's 50 emails/s limit and 0.5 % failures through throttling, retries and a DLQ (quality attribute: reliability).
- **Negative (accepted):** emails are eventually sent, not sent during the request; at a sustained 200 orders/s the email backlog grows by approximately 150 messages/s and drains only after the peak. This delay is accepted: R-12 requires that an order never fails, not that the email is instant, and the 50 emails/s limit is respected (quality attribute: timeliness / user experience).
- **Negative:** one more stateful component to run, upgrade and monitor, plus two new data stores: the `checkoutservice` order store for the outbox and the `loyaltyservice` store (quality attribute: operational cost / complexity).
- **Negative:** the outbox adds one local database write to `PlaceOrder`, which uses part of the approximately 200 ms latency headroom (quality attribute: performance).
- **Negative:** end-to-end flow is harder to trace and debug across async hops (quality attribute: observability). [Inferred] The existing OpenTelemetry setup (`main.go` lines 134-140, 154-178) could be extended by passing trace context in message headers.
- **Risks and mitigations:**
  - *Lost event (dual-write):* payment succeeds but the publish fails. Losing the event would mean a customer silently misses loyalty points, which breaks the exactly-once requirement, so this is **solved with the transactional outbox** above rather than accepted.
  - *Outbox relay lag or duplicates:* if the relay crashes after publishing but before marking a row sent, the event is published again. This is covered by the idempotent consumers below; relay lag must stay well under the 1-minute loyalty target.
  - *Duplicate delivery:* RabbitMQ is at-least-once. Mitigate with idempotent consumers keyed by order ID (mandatory for `loyaltyservice`, recommended for email so a customer is not mailed twice).
  - *Broker is a single point of failure for post-order actions:* orders still succeed if it is down, because events wait in the outbox. How to make the broker itself highly available (cluster size, queue type) is left to the broker's operators; see Open questions.
  - *Email backlog at peak:* accepted (see Consequences). Monitor queue depth and email delay, and alert if the backlog has not drained after the peak.

## Validation

- Load-test `PlaceOrder` at 200 orders/s (e.g. with the Locust `loadgenerator`, as in ADR-0001) and check p95 ≤ 500 ms with the publish step added.
- In the same test, measure loyalty lag: 100 % of orders have points awarded within 1 minute, and **zero** duplicate awards (count of awards per order ID = 1), including when messages are redelivered (kill a `loyaltyservice` pod mid-test).
- Outbox test: stop RabbitMQ during the load test; order success rate must stay 100 %, and every outbox row must be published and processed after RabbitMQ recovers (no missing loyalty awards).
- Chaos test: stop MailFast (or stub it to fail), `loyaltyservice` and the analytics consumer one at a time; order success rate must stay 100 %, and queued work must complete after recovery.
- Check MailFast send rate never exceeds 50 emails/s and failed sends end in retries or the DLQ, not lost.
- Review this ADR after the 11.11 sale, using queue-depth and email-delay metrics.

## Open questions

- **Email delay limit:** a delayed email during the peak is accepted, but what maximum delay should trigger an alert? Not given.
- **Peak duration:** how long does 200 orders/s last during 11.11? Needed to size queue storage and estimate the email drain time.
- **Order store:** which database will `checkoutservice` use for orders and the outbox table? `checkoutservice` has none today (Not evidenced in the provided files).
- **MailFast idempotency:** does MailFast support an idempotency key to prevent duplicate emails on retry? Not given.
- **Analytics destination:** where does the marketing team's analytics event go (a database, a third-party tool)? Not given.
- **Loyalty data store:** which store will `loyaltyservice` use for points and the order-ID de-duplication record? Not given.
- **Event schema and versioning:** format of `OrderPlaced` (e.g. Protobuf to match the team's gRPC skills [Inferred]) and who owns it.
- **RabbitMQ operations and high availability:** who runs and is on call for the broker, and what HA setup (for example a replicated cluster with quorum queues) and budget are needed? Not given.
