**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No: it had no scenario, so it lists volume, latency and MailFast limits as "Not given" (2a line 24, Open questions) | Yes: 500 ms p95 budget, 200 orders/s, MailFast p95 1.5 s / p99 8 s, 50 emails/s, points within 1 minute (2b lines 24–31) |
| Options considered (list them) | A) synchronous direct calls, B) event-driven pub/sub with a generic broker, C) post-order orchestrator service | 1) synchronous gRPC (today), 2) in-process goroutines, 3) RabbitMQ events, 4) Kafka events |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Only generically: idempotent consumers keyed by order ID, retries with backoff and a DLQ (line 77–78); no rate figure, broker not named | Yes: loyalty is idempotent on order ID with a unique constraint (exactly-once *effect* over at-least-once delivery); email consumer throttled to ≤ 50 emails/s with retry and DLQ (lines 66–67); notes a backlog of approximately 150 emails/s at peak |
| Anything invented (a number, tool or fact not in the scenario)? | No numbers invented; it chose pub/sub without knowing the team or constraints | Mostly grounded, but adds details not in the scenario: a 3-node cluster with quorum queues (line 85, 105), and a unique-constraint store for `loyaltyservice` |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Replaced "publish after payment, log on failure" with a transactional outbox: `PlaceOrder` writes the order and an `OrderPlaced` row to a new order store in one transaction, and a relay publishes it to RabbitMQ. Added an outbox test to Validation. | A lost event means a customer silently misses loyalty points, which breaks the "exactly once" requirement. The AI left this as an open question; I decided it is not acceptable. Cost: a new order store and one extra DB write in `PlaceOrder`. |
| 2 | Marked the email backlog (approximately 150 emails/s during the 11.11 peak) as an accepted consequence, and changed the open question to "what delay should trigger an alert". | R-12 says an order must never fail, not that the email must be instant. Queuing keeps MailFast under its 50 emails/s limit, so a delay during the peak is a fair trade-off. |
| 3 | Removed "3-node cluster with quorum queues" from the risks and moved broker HA into Open questions. Filled in Deciders. | That setup is not in the scenario and I could not justify the size. With the outbox, orders and events survive a broker outage anyway, so HA is an operations decision, not part of this ADR. |
