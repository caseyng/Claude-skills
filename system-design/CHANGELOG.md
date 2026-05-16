# Changelog — system-design

## 1.1.0 — 2026-05-16

### Added

- **`shared_types` field** in output schema. Data structures that cross component boundaries
  are collected here once and referenced by interaction contracts and component specs.
  Component specs receive Stage 2 as context — they MUST NOT re-define shared types.
  A type independently re-defined in two component specs will diverge.
- **Type definition requirement for interaction contracts.** The `format` field in every
  interaction contract MUST be a complete type definition (field names, types, required vs
  optional) — not a type name alone. "format: MessageEvent" is not a format.
- **Seam checks in completeness checklist** (items 5 and 6): every `format` field must be
  a complete type definition; every shared data structure must appear in `shared_types`.
- **Stage 2 litmus test** added as final completeness check: "Would a component spec writer,
  reading this design, make a wrong structural decision for any component?" Blocking if yes.
- **Six-phase execution model** clarified: Phase 3 (Interaction Contracts) is now separate
  from Phase 4 (Cross-Cutting Concerns + Technical Decisions) to emphasize the confirmation
  gate after component decomposition.

### Changed

- SKILL.md: v1.0.0 → v1.1.0
- Completeness check expanded from 8 to 10 items + litmus test.

### Rationale

The seam problem: when multiple component specs independently define the same shared data
structure, the definitions will diverge during implementation. Stage 2 is the only point
where all component boundaries are visible at once. Defining shared types here, referenced
by all subsequent stages, closes this failure mode. The Stage 2 litmus test matches the
universal pipeline pattern: every stage exits when no remaining question would cause the
next stage's agent to make a materially different output.

---

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
