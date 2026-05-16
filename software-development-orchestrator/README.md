# software-development-orchestrator

Drives the full software development lifecycle from a human's raw intent through to a
commissioned, working product. Manages six stages, human review gates, parallel agent
execution, and gap detection for missing skills.

## What it does

The orchestrator is not a skill that does work — it is the process that sequences skills.
It holds the pipeline definition, invokes each stage's skill, packages context for parallel
agents, presents human review gates, and advances automatically on approval.

When a required skill does not exist, it stops and emits a gap notice describing exactly
what needs to be built. The gap list is the living roadmap of skills that still need to
be created.

## The six stages

| Stage | Name | Skill |
|---|---|---|
| 1 | Requirements Engineering | `requirements-engineering` — MISSING |
| 2 | System Design | `system-design` — MISSING |
| 3 | Component Specification | `assisted-epistemology` + `spec-contract-methodology` — partially exists |
| 4 | Implementation | `android-engineering` / `python-engineering` + `code-integrity-guardrail` — EXISTS |
| 5 | Component Testing | `spec-driven-testing` + `spec-audit` — partially exists |
| 6 | Integration Testing | `integration-testing` — MISSING |

## Invocation

Fresh start — the human provides raw intent:
```
/skill:software-development-orchestrator
```

Resume a pipeline already in progress:
```
/skill:software-development-orchestrator resume
```
The orchestrator will ask for the path to the pipeline state document.

## Human checkpoints

The human reviews and approves at the end of each stage. Between approvals, the orchestrator
drives all work autonomously. The human does not decide when to move to the next stage — that
happens automatically on approval.

Parallel stages (3, 4, 5) run one agent per component simultaneously. Each agent receives a
complete, self-contained context package. After all agents complete, the orchestrator presents
a single gate for the human to review.

## State persistence

Pipeline state is written to `pipeline-state.md` in the project directory after every stage
and gate decision. This is the resume artifact. If a session ends mid-pipeline, the state
document is the only thing needed to continue.

## Commissioning

Commissioning is not a stage — it is the exit condition. When Stage 6 passes and the human
accepts the integration test results, the pipeline is complete and the product is delivered.

## What is not yet built

The orchestrator surfaces gaps automatically. At the time of writing, the following skills
are missing and will halt the pipeline when their stage is reached:

- `requirements-engineering` (Stage 1)
- `system-design` (Stage 2)
- `engineering-baseline` (needed by Stage 3)
- `spec-audit` (needed by Stage 5)
- `integration-testing` (Stage 6)

Each gap notice includes a complete description of what the missing skill must do, which
makes the gap notices the specification for what to build next.
