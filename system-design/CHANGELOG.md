# Changelog — system-design

## 1.0.0 — 2026-05-16

Initial release.

### What's in it

- Four-phase execution: intake + feasibility check, component decomposition (with human
  confirmation before proceeding), interaction contracts + cross-cutting concerns + technical
  decisions, document production
- Pushback function: blocking architectural questions surfaced in Phase 1 before design begins.
  Infeasible or ambiguous requirements redirected rather than silently designed around.
- Decomposition gate: component list confirmed by the human before interaction contracts
  and cross-cutting concerns are defined. The decomposition decision is the highest-leverage
  decision in the pipeline — confirm it before building on it.
- Decomposition principles: single responsibility, single ownership, clear boundary (one-sentence
  test), right size (one Stage 3 session per component), additive coupling
- Completeness check: 8-point verification before document production
- Enrichment checklists: universal system-level concerns + Android-specific concerns
- Output schema matches the orchestrator's Stage 2 definition

### Enrichment coverage

- Universal: startup sequence, shutdown, failure isolation, state consistency on restart,
  deployment unit, external dependency ownership, configuration, data ownership, versioning
  and migration, observability, security boundary, error propagation
- Android: process model, database file ownership, service → UI communication, foreground service
  ownership and type, Android component exposure, state persistence and lifecycle, boot persistence

### Design decisions

- Component decomposition is confirmed before contracts are defined — not surfaced alongside them.
  An unconfirmed decomposition makes all subsequent work provisional.
- "What a component is not" is explicit in the skill: layers are not components, technologies
  are not components, cross-cutting concerns are not components. These confusions are common
  and produce unstable decompositions.
- Every interaction contract includes `on_failure` — how the interaction behaves when it fails.
  This becomes the test target for Stage 6 (integration testing).
- Technical decisions are recorded in the System Design Document, not in implementation comments
  or commit messages. Decisions not recorded here will be re-made during implementation,
  usually inconsistently.

### Origin

Stage 2 of the software-development-orchestrator pipeline. Emerged from analyzing the whatsbot
Android app scope — the requirements phase produced a detailed feature set, but decomposing it
into components (foreground service, rules engine, scheduler, action log, settings, pairing UI)
with clear ownership boundaries required a distinct design step. The whatsbot plan document was
informally filling this role; this skill formalises it.
