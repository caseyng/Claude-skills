# Changelog — software-development-orchestrator

## 1.2.0 — 2026-05-16

### Added

- **Decision logs** — each stage and component now produces a decision log alongside its
  output artifact. Format defined in `references/decision-log-format.md`. Two-writer protocol:
  agent writes iteration entries, orchestrator appends gate outcomes. Parallel agents own their
  own files — never concurrent writes to the same decision log.

- **Iteration context feeding** — from iteration 2 onward, the agent context package includes
  the prior iteration artifact and the decision log. OPTIONS REJECTED entries in the log are
  treated as constraints by the next agent, not starting points. This is the primary mechanism
  preventing flip-flop across iterations.

- **Escalation Protocol** — when a stage/component reaches the iteration cap (3) without
  converging, the orchestrator reads the decision log, identifies the persistent contention
  point, and presents an Escalation Gate to the human. The human chooses between named options.
  Their decision becomes a hard constraint for one final iteration.

- **Per-component state files** — `output/state-stage[N]-[component].md`. Written by the
  orchestrator (not agents) after each agent completion. Tracks: status, iteration count,
  artifact path, decision log path, blocking issues, last gate feedback.

- **Updated `pipeline-state.md` schema** — now includes iteration counts per stage/component,
  decision log references, and an Escalated Items section for stuck components. This file is
  the human's query surface — readable at any time without interrupting the orchestrator.

- **Iteration count in human gate presentation** — every gate now shows "Iteration: N of 3"
  so the human knows how many attempts have been made and how close to escalation the stage is.

- **`system-design`, `spec-audit`, `integration-testing` status updated** — built in prior
  cycle; status updated from MISSING to EXISTS.

### Changed

- Execution Protocol: steps 4–10 updated to reflect iteration context, decision log protocol,
  state file update cadence, and escalation path.
- State Management section updated: two-level state, update cadence, resume protocol.
- `references/pipeline-stages.md`: every stage now defines decision log file path, state file
  path (parallel stages), what to record in the decision log, and iteration context package.
- `references/pipeline-state.md`: fully rewritten with two-level schema (per-component + pipeline).
- `assisted-epistemology` removed from Stage 3 skills list — superseded by the spec-contract
  methodology's internal Drafter→Critic→Judge loop and the orchestrator-level iteration protocol.

### Skills status after this release

Exists: `requirements-engineering`, `system-design`, `spec-contract-methodology`,
`android-engineering`, `python-engineering`, `code-integrity-guardrail`, `spec-driven-testing`,
`spec-audit`, `integration-testing`, `session-handoff`, `engineering-baseline` (this skill)

Missing (will emit gap notice): none — all six stages have skills

---

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
