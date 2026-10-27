> Reviewer: Claude Code (Anthropic), model claude-opus-5-5. Evidence read: `docs/adr/0002-rabbitmq-post-order-events.md`, `docs/scenario.md`. The ADR's `main.go` line references were not re-checked (outside the allowed files).

# ATAM review: ADR-0002, RabbitMQ for post-order events

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | Shoppers | 200 `PlaceOrder`/s | 11.11 peak, all services up | Order is persisted with an outbox row and returned. No post-order call happens in the request | p95 ≤ 500 ms (today approximately 300 ms, so ≤ 200 ms headroom for the new order-store write) |
| S2 Availability / correctness | Ops failure | RabbitMQ (or a `loyaltyservice` pod) is killed for 2 min | Peak load | Orders keep succeeding; events are buffered and delivered after recovery | 100 % order success; every order gets loyalty points **exactly once**, **≤ 60 s** after the order (scenario.md) |
| S3 Reliability / timeliness | MailFast | 200 orders/s against a 50 emails/s limit, 0.5 % failures, p99 8 s | Peak lasting T minutes | Email consumer throttles, retries, dead-letters | 0 emails lost; ≤ 50 sends/s; backlog drained within **[not defined in ADR]** |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | The **email backlog grows by 150 msg/s** and drains only at 50/s minus ongoing orders. A 1-hour peak gives approximately 540,000 queued emails and at least 3 hours to drain, and that only if orders fall to zero. If post-peak traffic stays above 50 orders/s, the backlog never drains. | "Accepted" with no peak duration and no maximum delay (Open questions). The real bottleneck is MailFast's contract, and the ADR does not address it. |
| Risk | The outbox adds a **new order store to the `PlaceOrder` critical path**, written *after* `chargeCard`/`shipOrder` (remote calls). If that write fails, the card is charged but no order or event exists. | The ADR says the dual write is "solved", but it has moved to payment-then-DB. It also adds a new way for orders to fail, which contradicts driver 1 in spirit. |
| Risk | Broker HA is **delegated to "operators"**. The outbox prevents loss but not lateness. A RabbitMQ outage longer than about 60 s breaks the loyalty 1-minute target. | S2 cannot be met without a decided HA setup (e.g. quorum queues, 3 nodes). |
| Risk | The outbox relay runs "in `checkoutservice`", which has several replicas. Row claiming, leader election and polling interval are not specified. | Concurrent relays cause duplicate publishes and contention. The polling interval directly consumes the 60 s loyalty budget. |
| Sensitivity point | Relay polling interval + consumer prefetch + retry back-off for `loyaltyservice`. | The sum of these must stay ≤ 60 s at the 99th percentile, including redelivery. |
| Sensitivity point | Order-store write latency at 200 writes/s. | This is the only new cost in S1. The store is not yet chosen (Open questions), so S1 cannot yet be assessed. |
| Trade-off point | Transactional outbox: + reliability (no lost events) vs − performance (synchronous DB write), − availability (new dependency), − cost (new store, relay). | This is the central trade-off, but the ADR presents only its benefits in "Decision". |
| Trade-off point | Throttling email at 50/s: + reliability toward MailFast vs − timeliness for customers (hours of delay at peak). | Business impact (support tickets, "where is my order?") is not weighed. |
| Non-risk | One durable queue per action on a fanout/topic exchange. | Isolates the MailFast slowness from loyalty and analytics, and makes SMS, fraud scoring and recommendations purely additive (driver 4). |
| Non-risk | Idempotent `loyaltyservice` using a unique order-ID constraint. | This is the correct way to get exactly-once *effect* over at-least-once delivery, provided the award and the de-duplication record are in one transaction (not stated). |
| Non-risk | RabbitMQ chosen over Kafka. | At 200 msg/s with 2 experienced developers and a 4-person team, it fits driver 5. Kafka's replay is not required by R-12. |

## 3. Missing or weak

- **Options not considered:** CDC/log-tailing relay (e.g. Debezium) instead of a polling relay; raising the MailFast limit or adding a second email provider, which is the only fix for S3; publishing directly with publisher confirms plus a local retry buffer (no order store) as a lighter alternative to the outbox; a quorum-queue RabbitMQ setup as part of the decision rather than an open question.
- **Consequences not stated:** failure of the order-store write after payment (compensation/refund?); where `emptyUserCart` now sits in the flow; schema ownership and versioning of `OrderPlaced` across 3+ consumer teams; DLQ handling (who replays dead-lettered emails or loyalty awards?).
- **Unsupported or inconsistent claims:**
  - "Only one publish is added" (Consequences) contradicts the Decision: with the outbox, the request adds a **DB write**, not a publish. Validation also says "with the publish step added".
  - "Dual-write … **solved**": not true for the payment → DB boundary (see Risks).
  - "Approximately 1 failed email/s": the consumer is throttled to 50/s, so failures are approximately 0.25/s. Retries also consume rate-limit capacity, which lengthens the drain.
  - "Fanout/topic exchange": undecided. Choose one; topic allows future selective routing (e.g. fraud scoring only for high-value orders).

## 4. Verdict: **Accept with changes**

1. **Close the email-timeliness gap.** State the peak duration and a maximum email delay, compute the drain time, and add an option for the MailFast limit (higher tier or second provider), or explicitly get product sign-off for hours of delay.
2. **Make the outbox decision complete.** Choose the order store and measure its write latency against the 200 ms headroom. Define behaviour when the write fails after `chargeCard`. Specify how relay replicas claim rows and the polling interval within the 60 s budget.
3. **Decide broker HA in this ADR** (e.g. 3-node RabbitMQ with quorum queues, owner and on-call). Without it, S2 (loyalty ≤ 60 s during a broker failure) is not met. Then fix the "one publish" and "dual-write solved" wording.
