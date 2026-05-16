# Changelog — requirements-engineering

## 1.0.0 — 2026-05-16

Initial release.

### What's in it

- Four-phase execution: intent intake, discovery round, iteration until complete, document production
- Assumptions surfacing: implicit constraints stated explicitly for human confirmation,
  not asked as open questions
- Enrichment checklists: universal concerns + platform-specific (Android, web, backend)
- Phase mapping with extension point analysis: for every deferred feature, identifies what
  Phase 1 must include to keep later phases additive
- Completeness check: blocking/non-blocking test applied at requirements level — exit on
  information completeness, not round count
- Requirements Document output: structured format consumed directly by system-design as Stage 2 input

### Design decisions

- Assumptions are stated ("Assuming: X. Correct?"), not asked ("Is X true?") — human corrects
  rather than answers, which is faster and catches more errors
- Each discovery round presents all findings at once — no drip-feeding of individual questions
- Extension point test is explicit: "if Phase N is added later, what in Phase 1 changes?"
  Anything that changes is a missed extension point, designed in now at minimal cost
- Platform enrichment files are separate references loaded on demand, not inline — the skill
  does not load Android concerns for a web project

### Origin

Built as Stage 1 of the software-development-orchestrator pipeline. Emerged from a
discussion about what requirements engineering actually needs to do beyond gathering — specifically,
surfacing implicit assumptions (the "killers that cause drift and tech debt") and designing
phase boundaries that make later phases additive rather than rework-requiring.
