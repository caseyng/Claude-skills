# Changelog — software-development-orchestrator

## 1.1.0 — 2026-05-16

### Changes

- **Artifact model** — agents are now explicitly ephemeral: each receives a complete context
  package, writes output to a defined file path, and terminates. The orchestrator reads
  validation fields only — never full artifact content.
- **Validation step** — added between Execute and human gate. The orchestrator validates
  mechanically (schema, completeness, stage-specific criteria) before presenting to the human.
  Failed output is not surfaced to the human — a new agent is spawned with the prior file +
  correction instructions.
- **Human gate format updated** — artifacts section now references output files rather than
  reproducing content. Human reads files directly.
- **Engineering baseline** — created as `references/engineering-baseline.md`, owned by the
  orchestrator, packaged into every Stage 3 agent's context. Nine principles: naming,
  failure taxonomy, observability, responsibility boundaries, validation placement, extension
  points, state ownership, contracts-before-code, security defaults.
- **Stage 1 status updated** — `requirements-engineering` was built in this cycle; status
  updated from MISSING to EXISTS.
- **Output file paths** — defined for every stage in `pipeline-stages.md`.
- **Validation criteria** — defined per stage in `pipeline-stages.md`.
- **Rework loops** — explicitly defined: new agent receives prior output file + rejection
  reasons; does not replay full prior agent context.

### Skills status after this release

Exists: `requirements-engineering`, `spec-contract-methodology`, `assisted-epistemology`
(v0.1-draft), `android-engineering`, `python-engineering`, `code-integrity-guardrail`,
`spec-driven-testing`, `session-handoff`, `engineering-baseline` (this skill, v1.1.0)

Missing (will emit gap notice): `system-design`, `spec-audit`, `integration-testing`

---

## 1.0.0 — 2026-05-16

Initial release.

### What's in it

- Six-stage pipeline: Requirements Engineering → System Design → Component Specification →
  Implementation → Component Testing → Integration Testing → Commissioning
- Human review gate at every stage boundary with consistent format
- Gap detection: when a required skill is missing, emits a structured gap notice and stops.
  The gap notice is the specification for what to build next.
- Parallelism declarations: Stages 3, 4, 5 run one agent per component in parallel.
  Stage 5 runs lint, behavioral testing, and spec compliance in parallel within each component.
- Pipeline state document: structured markdown, written after every stage and gate decision,
  used to resume across sessions
- Commissioning milestone: explicit exit condition distinct from the last stage

### Skills referenced

Exists: `spec-contract-methodology`, `assisted-epistemology` (v0.1-draft), `android-engineering`,
`python-engineering`, `code-integrity-guardrail`, `spec-driven-testing`, `session-handoff`

Missing (will emit gap notice): `requirements-engineering`, `system-design`, `engineering-baseline`, `spec-audit`, `integration-testing`

### Origin

Emerged from a design session on whatsbot (WhatsApp automation Android app) where the skill
pipeline was analysed end-to-end. The existing skills (spec-contract, code-integrity-guardrail,
spec-driven-testing, android-engineering) covered Stages 3–5 but had no orchestrating layer
and no Stage 1, 2, or 6. The orchestrator was identified as the highest-leverage first build:
it defines the contracts for every missing skill and surfaces gaps as structured, actionable notices.
