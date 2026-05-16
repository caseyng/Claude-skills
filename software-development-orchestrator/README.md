# software-development-orchestrator

Drives the full software development lifecycle from a human's raw intent through to a
commissioned, working product. Manages stages, human review gates, parallel agent execution,
decision logs, and gap detection for missing skills.

## What it does

The orchestrator is not a skill that does work — it is the process that sequences skills.
It holds the pipeline definition, packages context for parallel agents, presents human review
gates, advances automatically on approval, maintains decision logs per stage and component,
and detects when stateless agents are re-litigating closed decisions.

When a required skill does not exist, it stops and emits a gap notice describing exactly
what needs to be built. The gap notice is the specification for the missing skill.

---

## Pipeline Profiles

The orchestrator asks which profile applies at fresh start. The profile determines which
stages run and what constitutes commissioning.

| Profile | Stages | Use case |
|---|---|---|
| **Consumer** | 1–6 | Apps, web services, internal tools |
| **Enterprise** | 1–9 | Government, finance, healthcare, regulated deployments |
| **Custom** | 1–6 + selected optional | Performance-critical consumer, security-sensitive non-regulated |

A product that exits at Stage 6 is functionally tested and integration-verified. It is **not**
production-grade for regulated environments without Stages 7–9.

---

## Stages

### Core Stages (all profiles)

| Stage | Name | Skill | Status |
|---|---|---|---|
| 1 | Requirements Engineering | `requirements-engineering` | EXISTS |
| 2 | System Design | `system-design` | EXISTS |
| 3 | Component Specification | `spec-driver` + `spec-contract-methodology` | EXISTS |
| 4 | Implementation | `[platform]-engineering` + `code-integrity-guardrail` | EXISTS (android, go, python) |
| 5 | Component Testing | `spec-driven-testing` + `spec-audit` | EXISTS |
| 6 | Integration Testing | `integration-testing` | EXISTS |

### Enterprise Stages (enterprise and custom profiles)

| Stage | Name | Skill | Status |
|---|---|---|---|
| 7 | System Security Acceptance Testing (SSAT) | `ssat` | MISSING |
| 8 | Vulnerability Assessment & Penetration Testing (VAPT) | `vapt` | MISSING |
| 9 | Stability Run (performance, load, soak, recovery) | `stability-testing` | MISSING |

Stages 7–9 are fully defined in `references/pipeline-stages.md`. When the pipeline reaches
any of these stages without the corresponding skill, a gap notice is emitted. The gap notice
is the specification for what to build.

---

## What each stage produces

| Stage | Output | Gate |
|---|---|---|
| 1 | Requirements Document | Human approves |
| 2 | System Design Document (components, shared types, interaction contracts, technical decisions) | Human approves |
| 3 | §1–§23 Component Specification per component (Implementation Readiness = READY) | Human approves |
| 4 | Implemented source files per component (build PASS, guardrail PASS) | Auto-advance unless SPEC_ERROR_REVEALED or UPSTREAM_REDESIGN_REQUIRED |
| 5 | Test suite + unfakeable test execution artifact per component | Auto-advance unless failures |
| 6 | Integration test results (PASS/FAIL/MANUAL_REQUIRED per interaction contract) | Human approves |
| 7 | Security control test results (PASS/FAIL per §19 contract) | Human approves |
| 8 | VAPT findings (CRITICAL/HIGH → must fix; MEDIUM/LOW/INFO → accepted with rationale) | Human approves |
| 9 | Stability test results (load, stress, soak, recovery per §23 contracts) | Human approves |

---

## Key mechanisms

**Decision logs.** Every stage and component maintains a decision log recording what was
decided, why, and what was rejected. Agents that re-run receive the prior decision log as
context — OPTIONS REJECTED entries are constraints, preventing flip-flop across iterations.

**Iteration cap.** Each stage/component caps at 3 orchestrator-level re-spawns. At cap, the
orchestrator identifies the persistent contention point and escalates to the human with named
options. The human's decision becomes a hard constraint for one final iteration.

**Pipeline-state.md.** Updated after every agent completion. The human's query surface —
readable at any time to see what is in-progress, stuck, or escalated, without interrupting
the orchestrator.

**Platform routing.** Stage 4 selects the engineering skill per component based on the
`platform` field declared in Stage 2. Multi-language projects (e.g., Android + Go) route
each component to the appropriate skill independently.

**UPSTREAM_REDESIGN_REQUIRED.** A new deviation type at Stage 4: the shared type as designed
cannot be implemented on the target platform. Routes to Stage 2 (not Stage 3 or Stage 4 rework).
All Stage 3 and Stage 4 work for affected components is invalidated.

**Unfakeable test evidence.** Stage 5 requires the raw test runner output written to a file
for every component. Claims of test passage without this artifact are treated as validation
failure — the agent is re-spawned with explicit instruction to run tests and capture output.

---

## Invocation

Fresh start:
```
/skill:software-development-orchestrator
```

Resume:
```
/skill:software-development-orchestrator resume
```

---

## What is not yet built

**Enterprise stage skills (Stages 7–9):** The stages are fully defined in
`references/pipeline-stages.md`. The skills are not built. For most consumer applications
(Play Store, App Store, web services, internal tools), Stages 1–6 are sufficient. For
government, finance, healthcare, or any regulated production deployment, Stages 7–9 are
required before commissioning.

| Missing skill | Stage | When needed |
|---|---|---|
| `ssat` | 7 | Enterprise profile; security-sensitive custom |
| `vapt` | 8 | Enterprise profile |
| `stability-testing` | 9 | Enterprise profile; performance-critical consumer |

**Platform engineering skills:** `typescript-engineering`, `nodejs-engineering`,
`ios-engineering`, `rust-engineering`. Each MISSING skill emits a gap notice at Stage 2
feasibility check (warn) and Stage 4 execution (block). Build the skill to unblock.

**Stage 3 internal architecture:** Each Stage 3 component runs via `spec-driver`, which
orchestrates a Drafter→Critic→Judge cycle (up to 3 cycles) internally. The Critic is a
fresh agent that did not write the spec — preventing bias. The Judge applies the single
test independently to classify gaps. See `spec-driver/SKILL.md` for the full protocol.
