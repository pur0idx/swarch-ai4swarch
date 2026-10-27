**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | Pitchanan Thaiphayak (the ADR's author) | defends the ADR |
| **AI analyst** | Kotchapan | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Pon | role card P (business side) |
| **Site reliability engineer (SRE)** | Piyapanchanok | role card S (operations side) |
| **Security & privacy officer** | Chayutphong Soisri | role card X (security & privacy) |

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

**ADR reviewed:** `docs/adr/0002-…` (ADR-0002: Publish an `OrderPlaced` event to RabbitMQ for post-order actions) by Pitchanan Thaiphayak
**Board members and roles:** Pitchanan Thaiphayak (architect), Kotchapan (AI analyst), Pon (product owner), Piyapanchanok (SRE), Chayutphong Soisri (security & privacy officer)

**Decision:** accept with changes (vote 5–0)

**Required changes:**
1. **Clarify payment and outbox consistency.** Document the transaction boundaries between payment, order persistence and event publication, and define recovery and reconciliation if a failure occurs between these steps.
2. **Specify delivery and recovery guarantees.** Document relay operation, retries, duplicate handling, dead-letter processing, monitoring and recovery, and validate that the system meets its workload and latency targets.
3. **Define business-level service expectations.** Agree on acceptable email delays, backlog recovery, and the one-minute loyalty requirement during broker outages; support broker comparisons with evidence.
4. **Add security and privacy requirements.** Minimize personal data in events; define broker authentication, authorization, connection protection, retention, and dead-letter queue access.
5. **Validate the design before production.** Test peak throughput, broker outages, consumer failures, duplicate delivery and recovery behaviour.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Payment may succeed without a durable event (payment-to-outbox boundary) | Architect and SRE | Yes | Yes: it can infer the consistency risk from the documented transaction boundary, but the correct recovery policy needs human agreement. |
| 2 | Loyalty may miss its one-minute target during a broker outage | PO and SRE | Yes | Partly: it can identify the conflict, but business priorities and acceptable degradation need stakeholder decisions. |
| 3 | Email backlog grows at peak (200 orders/s vs 50 emails/s, approximately 9,000 emails/min) | PO | Yes | Partly: it can calculate the backlog rate, but acceptable delay and peak duration need business confirmation. |
| 4 | Broker comparison (RabbitMQ vs Kafka) claims may be unsupported | Architect | Yes | Yes: it can flag unsupported statements and ask for evidence. |
| 5 | Personal data (customer emails) may be exposed or retained in queued events and dead-letter records | Security | No (not a main finding; the ADR only lists privacy as an open question) | Partly: it could flag the open question and suggest controls, but the organization's security and privacy requirements were not in its input. |

**The AI's best finding** (one the humans had missed): the payment-to-outbox consistency gap. It exposed a failure scenario the architecture must handle explicitly, and showed the board that having an outbox is not the same as guaranteeing that payment and event stay consistent.
**The AI's weakest finding** (generic, wrong or irrelevant): the 30-minute peak example. The backlog calculation is useful, but the peak duration is an assumption, not a scenario requirement, and could mislead if read as one.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
