Agent: Claude Code (Anthropic), model claude-opus-5-5

# ADR-0002: Publish an "order placed" event and handle post-order actions in independent subscribers

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not given (see Open questions)

## Context and problem

After an order is placed, Online Boutique must do three new things (from the requirement):

1. Send the order confirmation email through **MailFast**, a new external email provider. MailFast replaces the mock `emailservice` logic.
2. Award **loyalty points** through a new `loyaltyservice` (to be built).
3. Record an **"order placed" analytics event** for the marketing team.

Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

The problem: how should the order-placing part of the system trigger these post-order actions? The choice decides how tightly order placement is coupled to each action, and how much work it takes to add the next actions.

Facts not available for this ADR (context: none):

- Which service places orders today and how it currently calls `emailservice` (protocol, sync or async). Not evidenced in the provided files.
- Order volume, latency targets for checkout, and MailFast's availability, rate limits and API style. Not given.
- Whether a message broker already exists in the system. Not given.

## Decision drivers

Most important first. The ranking is a proposal, since no quality goals were given.

- **Modifiability / extensibility:** adding SMS, fraud scoring and a recommendations update next year should not require changing the order-placing code each time.
- **Availability of checkout:** a failure or slowdown in MailFast (external), `loyaltyservice` (new) or analytics should not make placing an order fail.
- **Performance (checkout latency):** the customer should not wait for the email, points and analytics to finish.
- **Reliability of each action:** each action should eventually happen once (no lost emails or points, no double points), even after temporary failures.
- **Simplicity / operational cost:** the team must build, run and debug the solution.

## Considered options

1. **Option A - Synchronous direct calls:** the order-placing service calls MailFast (or an adapter replacing `emailservice`), `loyaltyservice` and the analytics sink one after another before it returns to the customer.
2. **Option B - Event-driven publish/subscribe:** the order-placing service publishes one `OrderPlaced` event to a message broker. Each post-order action is an independent subscriber: an email adapter (calls MailFast), `loyaltyservice`, and an analytics consumer.
3. **Option C - Post-order orchestrator service:** the order-placing service makes one call (or sends one message) to a new `postorderservice`, which calls each action in turn and manages retries.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A - Synchronous direct calls | Simplest to build and debug; flow is visible in one place; customer gets an immediate success/failure for each action; no new infrastructure | Checkout latency is the sum of all calls; an outage in MailFast or `loyaltyservice` can fail or delay checkout; every new action (SMS, fraud, recommendations) changes the order-placing code; retries and partial failures (email sent, points failed) must be handled inline |
| B - Event-driven pub/sub | Order placement is decoupled from the actions; new actions are added as new subscribers without touching order placement; checkout latency includes only the publish; a slow or down subscriber does not affect checkout or other subscribers; broker buffers events during outages; analytics fits naturally as an event consumer | New infrastructure (broker) to run and monitor; eventual consistency (email and points arrive later); harder end-to-end tracing; needs idempotent consumers and a dead-letter / retry policy; risk of losing the event if the order is saved but the publish fails (needs outbox or similar) |
| C - Post-order orchestrator | Order placement makes one call only; all post-order logic in one place, easy to see the flow and handle ordering (e.g. fraud score before email); central retry handling | Orchestrator becomes a new single point of failure and a change hotspot: every new action still edits it; if called synchronously, latency and availability problems of A move into it; if called asynchronously, it still needs a queue, so it adds a service on top of the cost of B |

## Decision

Choose **Option B - event-driven publish/subscribe**.

- The order-placing service publishes one `OrderPlaced` event (order ID, customer, items, totals, email address) after the order is placed.
- Three subscribers consume it independently:
  - an **email adapter** (replacing the mock `emailservice` logic) that calls MailFast,
  - the new `loyaltyservice`, which awards points,
  - an **analytics consumer** that records the "order placed" event for marketing.
- Next year's actions (SMS, fraud scoring, recommendations update) are added as further subscribers.

Why: B best satisfies the top drivers. It is the only option where adding the expected new actions does not change existing code (modifiability), and where an external provider outage does not block checkout (availability, performance). Its costs (broker, eventual consistency, idempotency) are accepted as the price of these goals.

Note: if fraud scoring must **block** an order before it is confirmed, it is not a post-order action and does not fit this event; it would need a synchronous call in the checkout path. **[Inferred]** — to be confirmed (see Open questions).

## Consequences

- **Positive:** New post-order actions are added as new subscribers without changing the order-placing service or other subscribers. (quality attribute: modifiability)
- **Positive:** Failures or slowness in MailFast, `loyaltyservice` or analytics do not fail checkout; the broker keeps events until consumers recover. (quality attribute: availability)
- **Positive:** Checkout response time includes only publishing the event, not the three actions. (quality attribute: performance)
- **Positive:** Each subscriber can be scaled, deployed and owned separately (e.g. by the marketing team for analytics). (quality attribute: scalability, deployability)
- **Negative:** A message broker must be introduced, operated and monitored. (quality attribute: cost, operability)
- **Negative:** Email and loyalty points are eventually consistent; the customer may see the confirmation page before the email or points arrive. (quality attribute: consistency / usability)
- **Negative:** End-to-end flow is spread across services and harder to trace and debug. (quality attribute: observability, testability)
- **Risks and mitigations:**
  - *Event lost between saving the order and publishing* → use a transactional outbox or publish-with-confirmation and retry. (reliability)
  - *Duplicate delivery causes double points or double emails* → make consumers idempotent using the order ID as a deduplication key. (correctness)
  - *MailFast outage or rate limiting* → retries with backoff in the email adapter, a dead-letter queue, and alerting. (availability)
  - *Event schema changes break consumers* → version the `OrderPlaced` schema and evolve it in a backward-compatible way. (modifiability)
  - *Personal data (email address) in events read by many consumers* → limit fields in the event, restrict subscriber access. (security / privacy)

## Validation

- **Fault-injection test:** stop MailFast (stub) and `loyaltyservice`; checkout must still succeed, and emails and points must be delivered after they recover.
- **Latency comparison:** measure checkout latency before and after; the added time should be only the event publish. (Target not given — see Open questions.)
- **Duplicate-delivery test:** redeliver the same `OrderPlaced` event; points and emails must not be duplicated.
- **Extensibility check:** add a stub SMS subscriber; it must require no change to the order-placing service.
- **Metrics in production:** consumer lag, dead-letter queue size, MailFast error rate, time from order to email sent.
- **Review date:** revisit when the first of next year's actions (SMS, fraud scoring, recommendations) is designed.

## Open questions

- Who are the deciders for this ADR?
- Which service places orders today, and how does it call `emailservice` now (protocol, sync/async)? Not evidenced in the provided files.
- What are the expected order volume and peak load?
- Is there a checkout latency target and an availability target?
- Is a message broker already in use or allowed? Which one (team skills, hosting, cost limits)?
- What does MailFast offer: API type (REST, SMTP), SLA, rate limits, retries, webhooks for delivery status, cost per email?
- How fast must the confirmation email and the loyalty points arrive after the order (acceptable delay)?
- Where does marketing want analytics events to go (existing analytics platform, data warehouse)? What fields are needed?
- Must fraud scoring be able to block or cancel an order (synchronous, before confirmation) or only flag it afterwards?
- Are there data-privacy rules on sending customer email addresses and order data to MailFast and to analytics?
- Who owns `loyaltyservice` and the analytics consumer, and in which language/technology will they be built?
