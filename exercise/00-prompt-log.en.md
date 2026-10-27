**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID: Chayutphong Soisri 6630054621**
**Agent(s) used:** Claude Code (desktop app, run as separate sub-agents per run)
**Model(s):** claude-opus-5-5
**Plan / tier:** Claude Pro plan

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-…` run A, evidence: `demo.proto` | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/1-recover-architecture.md`, `demo.proto` | Yes | 3 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-…` run B, evidence: `demo.proto`, `kubernetes-manifests/` | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/1-recover-architecture.md`, `demo.proto`, all 13 files in `kubernetes-manifests/` | Yes (also tried to open the `kubernetes-manifests/` folder itself, which failed because it is a directory) | 4 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-…` run a, context: none, requirement: R-12 | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/2-write-adr.md`, `docs/adr/template.md` | Yes (did not open `docs/scenario.md`) | 3 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 4 | 2-b | Claude Code + Opus 5.5 | `prompts/2-…` run b, context: `scenario.md`, ADR 0001, `checkoutservice/main.go`, requirement: R-12 | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | Yes | 5 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 5 | 3 | Claude Code + Opus 5.5 | `prompts/3-…`, adr: `docs/adr/0002-rabbitmq-post-order-events.md` | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/3-review-adr.md`, `docs/adr/0002-rabbitmq-post-order-events.md`, `docs/scenario.md` | Yes (did not edit the ADR; did not check the `main.go` line numbers the ADR cites, as that file was out of scope) | 4 | Run in a new Claude Code session; verdict: accept with changes |
| 6 | Worksheet 01 | Claude Code + Opus 5.5 | Asked for commands to check 1B, then asked Claude to fill in the verify table, "Wrong or missing" and reflection | `exercise/outputs/1B-architecture.md`, `exercise/01-recover-architecture.en.md` | N/A (not an evidence task) | 4 | I ran the grep commands; Claude wrote the 5 table rows and the "Wrong or missing" line from my output, and rewrote my draft Reflect answers |
| 7 | Bonus | Claude Code + Opus 5.5 | `prompts/bonus-…` gRPC + shared proto decision | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/bonus-recover-decision.md`, `docs/adr/template.md`, `demo.proto`, `src/checkoutservice/main.go`, `src/frontend/main.go` | Yes | 4 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 8 | Worksheet 02 + ADR 0002 | Claude Code + Opus 5.5 | Asked Claude to fill the compare table, then gave my 3 decisions (outbox, accept email delay, move broker HA to open questions) | `exercise/outputs/2a-adr.md`, `exercise/outputs/2b-adr.md`, `docs/adr/0002-rabbitmq-post-order-events.md`, `exercise/02-write-adr.en.md` | N/A (not an evidence task) | 4 | Claude filled the compare table, applied my 3 decisions to ADR 0002 and drafted the change-table rows |

