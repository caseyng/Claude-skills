# Pipeline State Document Schema

Two levels of state are maintained in the project's working directory.

**Per-component state files** (`output/state-stage[N]-[component].md`): Written by the
orchestrator after each agent completion. One file per component per parallel stage.
Parallel agents do not write state — the orchestrator does, after collecting results.

**Pipeline-level state** (`pipeline-state.md`): The merged human-readable view. Written
after every agent completion and after every gate decision. This is the human's query
surface — reading it answers "where are things" without requiring orchestrator interaction.

On session resume: read `pipeline-state.md` first, then the decision logs for any
`in_progress` or `awaiting_gate` stages.

---

## Per-Component State File

Path: `output/state-stage[N]-[component-name].md`

```markdown
# State — Stage [N] — [Component Name]

Stage: [N]
Component: [name]
Last updated: [date]

Status: not_started | in_progress | awaiting_gate | approved | rejected | escalated | blocked_by_gap
Current iteration: [N]
Iteration cap: 3
Gate status: — | pending_human | approved | rejected | escalated

Artifacts:
  Output:       [file path, or "—"]
  Decision log: [file path, or "—"]

Blocking issues:
  [description of what is blocking, or "None"]

Last gate feedback:
  [feedback from most recent human gate rejection, or "None"]
```

Status values:
- `not_started` — agent not yet spawned for this component
- `in_progress` — agent running; output not yet collected
- `awaiting_gate` — output collected and validated; human gate presented
- `approved` — human approved; component will not re-run unless explicitly returned to
- `rejected` — human rejected; rework spawned (iteration incremented)
- `escalated` — iteration cap reached; human decision pending
- `blocked_by_gap` — a required skill is missing

---

## Pipeline-Level State Document

Path: `pipeline-state.md` in the project working directory.

```markdown
# Pipeline State — [project_name]

## Overview
Project:       [project_name]
Active stage:  [1–6 | COMMISSIONED]
Last updated:  [date]

## Stage Status

| Stage | Name                      | Status           | Iteration | Gate              |
|-------|---------------------------|------------------|-----------|-------------------|
| 1     | Requirements Engineering  | [status]         | [N/3]     | [gate]            |
| 2     | System Design             | [status]         | [N/3]     | [gate]            |
| 3     | Component Specification   | [status]         | —         | [gate]            |
| 4     | Implementation            | [status]         | —         | [gate]            |
| 5     | Component Testing         | [status]         | —         | [gate]            |
| 6     | Integration Testing       | [status]         | [N/3]     | [gate]            |

Status values:  not_started | in_progress | awaiting_gate | complete | blocked_by_gap
Gate values:    — | pending_human | approved | rejected | escalated

## Component List
(Populated from Stage 2. Locked on Stage 2 approval. Changes require returning to Stage 2.)

- [component name]
- [component name]

## Per-Component Progress
(Populated when Stage 2 is approved.)

### Stage 3 — Component Specification
| Component | Status      | Iteration | Gate              | Decision Log                              |
|-----------|-------------|-----------|-------------------|-------------------------------------------|
| [name]    | [status]    | [N/3]     | [gate]            | output/decisions-stage3-[name].md         |

### Stage 4 — Implementation
| Component | Status      | Iteration | Gate              | Decision Log                              |
|-----------|-------------|-----------|-------------------|-------------------------------------------|
| [name]    | [status]    | [N/3]     | [gate]            | output/decisions-stage4-[name].md         |

### Stage 5 — Component Testing
| Component | Lint        | Behavioral    | Compliance         | Gate    | Decision Log                        |
|-----------|-------------|---------------|--------------------|---------|-------------------------------------|
| [name]    | [pass/fail] | [pass/fail]   | [pass/fail/skip]   | [gate]  | output/decisions-stage5-[name].md   |

## Escalated Items
(Components that reached the iteration cap without convergence. Human decision pending.)

| Stage | Component | Persistent Contention                     | Decision Log                        |
|-------|-----------|-------------------------------------------|-------------------------------------|
| [N]   | [name]    | [one sentence describing the stalemate]   | output/decisions-stage[N]-[name].md |

## Artifacts
Stage 1 — Requirements Document:    output/stage-1-requirements.md
Stage 2 — System Design Document:   output/stage-2-system-design.md
Stage 3 — Component Specs:          output/stage-3-spec-[name].md (one per component)
Stage 4 — Implementation:           output/stage-4-impl-[name].md (one per component)
Stage 5 — Test Results:             output/stage-5-test-[name].md (one per component)
Stage 6 — Integration Results:      output/stage-6-integration.md

## Gaps Encountered
(Missing skills flagged during this pipeline run.)

| Stage | Skill              | Status          |
|-------|--------------------|-----------------|
| [N]   | [skill-name]       | [open/resolved] |

## Open Items
(Pending routing decisions, deviations awaiting action, feedback being incorporated.)

- [item description]
```

---

## Field Definitions

**`active_stage`:** The stage currently executing or awaiting a human gate. Set to
`COMMISSIONED` when Stage 6 is approved.

**`iteration` (component level):** N/3 — current iteration number over the cap. Reset to
1 when a stage is returned to (e.g. Stage 3 re-runs after a Stage 5 failure).

**`escalated items`:** Components where the iteration cap was reached. Each entry remains
until the human provides a decision and the component either converges or is abandoned.

**`gaps encountered`:** Every missing skill flagged during this pipeline run. Informs which
skills need to be built. Remove when resolved.

**`open items`:** Transient. Remove entries when resolved.
