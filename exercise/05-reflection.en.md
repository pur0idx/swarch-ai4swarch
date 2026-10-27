**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Chayutphong Soisri 6630054621
**My role on the review board:** Security & privacy officer

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I would trust AI to identify possible security weaknesses, flag sensitive data in architecture diagrams or event schemas, and suggest relevant controls. For example, the board identified that queued events and dead-letter records could contain customer email addresses, while the ADR did not yet define detailed retention and access rules. The AI review did not raise this, even though it covered the DLQ, because it had no privacy requirements as input; AI can suggest data minimization, access control and retention policies once asked. However, humans must verify the actual data flows and apply the organization's security and privacy requirements. AI cannot independently determine which data the business is permitted to retain or approve the resulting risk. As the Security Officer, I would use AI to support threat identification and review, but I would require explicit controls and evidence before considering the design secure enough for production.