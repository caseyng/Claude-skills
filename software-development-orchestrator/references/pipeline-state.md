# Pipeline State Document Schema

The pipeline state document is written to `pipeline-state.md` in the project's working directory.
It is updated after every stage completion and after every human gate decision.
On session resume, it is the first thing read.

---

## Format

The state document is a structured markdown file. Every field is required unless marked optional.

---

```markdown
# Pipeline State — [project_name]

## Overview
Project:       [project_name]
Active stage:  [1–6 | COMMISSIONED]
Last updated:  [date]

## Stage Status

| Stage | Name                      | Status      | Gate         |
|-------|---------------------------|-------------|--------------|
| 1     | Requirements Engineering  | [status]    | [gate]       |
| 2     | System Design             | [status]    | [gate]       |
| 3     | Component Specification   | [status]    | [gate]       |
| 4     | Implementation            | [status]    | [gate]       |
| 5     | Component Testing         | [status]    | [gate]       |
| 6     | Integration Testing       | [status]    | [gate]       |

Status values:  pending | in_progress | awaiting_gate | complete | blocked_by_gap
Gate values:    — | pending_human | approved | rejected

## Component List
(Populated from Stage 2. Locked on Stage 2 approval. Changes require returning to Stage 2.)

- [component name]
- [component name]
...

## Per-Component Progress
(Populated when Stage 2 is approved. One row per component per parallel stage.)

### Stage 3 — Component Specification
| Component | Status      | Gate           |
|-----------|-------------|----------------|
| [name]    | [status]    | [gate]         |

### Stage 4 — Implementation
| Component | Status      | Gate           |
|-----------|-------------|----------------|
| [name]    | [status]    | [gate]         |

### Stage 5 — Component Testing
| Component | Lint   | Behavioral | Compliance | Gate    |
|-----------|--------|------------|------------|---------|
| [name]    | [pass/fail] | [pass/fail] | [pass/fail/skipped] | [gate] |

## Artifacts
(Location or brief summary of each stage's primary output. Update as stages complete.)

Stage 1 — Requirements Document: [location or inline summary]
Stage 2 — System Design Document: [location or inline summary]
Stage 3 — Component Specs: [location per component]
Stage 4 — Implementation: [source location per component]
Stage 5 — Test Results: [summary per component]
Stage 6 — Integration Results: [summary]

## Gaps Encountered
(All missing skills flagged during this pipeline run.)

| Stage | Skill              | Status    |
|-------|--------------------|-----------|
| [N]   | [skill-name]       | [open/resolved] |

## Open Items
(Pending decisions, deviations awaiting routing, rejection feedback being acted on.)

- [item description]
```

---

## Field Definitions

**`active_stage`:** The stage currently executing or awaiting a human gate.
Set to `COMMISSIONED` when Stage 6 is approved and commissioning is declared.

**`status` (stage-level):**
- `pending` — not yet started
- `in_progress` — executing (skill invoked, waiting for output)
- `awaiting_gate` — output produced, human review package presented, waiting for decision
- `complete` — human approved; this stage will not re-run unless explicitly returned to
- `blocked_by_gap` — a required skill is missing; pipeline paused at this stage

**`gate` (stage-level):**
- `—` — gate not yet reached
- `pending_human` — review package presented, awaiting human decision
- `approved` — human approved; stage is complete
- `rejected` — human rejected; rework in progress

**Component list:** Populated from Stage 2's `components` field on approval.
This list drives Stages 3, 4, and 5. It is immutable after Stage 2 approval.
Changing the component list requires returning the entire pipeline to Stage 2.

**Gaps encountered:** Every time the orchestrator encounters a MISSING skill, it records
it here. This is the living gap register for the pipeline and informs which skills need to
be built to un-block the pipeline.

**Open items:** Transient entries — a routing decision pending human input,
a deviation being acted on, rejection feedback being incorporated.
Remove entries when resolved.
