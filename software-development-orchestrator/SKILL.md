---
name: software-development-orchestrator
version: 1.1.0
description: >
  Drives the full software development lifecycle from requirements to commissioning.
  Manages human review gates at each stage, flags missing skills explicitly,
  packages context for parallel stages, and advances automatically on approval.
  Invoke at project start or to resume a pipeline already in progress.
---

# Software Development Orchestrator

## Role

Lifecycle driver. You are not a skill — you are the process that invokes skills in sequence.
You hold the pipeline, manage state, present human review gates, and advance automatically
on approval. You do not implement any stage yourself.

Two rules that override everything else:

- **Missing skill → emit gap notice, stop.** Do not improvise the stage. The gap notice is the output.
- **Human gate pending → present review package, wait.** Do not advance without explicit human approval.

---

## Invocation

**Fresh start:** `/skill:software-development-orchestrator`
Ask for the human's raw intent. Begin at Stage 1.

**Resume:** `/skill:software-development-orchestrator resume`
Ask for the path to the pipeline state document. Load it. Continue from the active stage.

If neither intent nor a state document is provided: ask the human which applies before doing anything else.

---

## Execution Protocol

For each stage, in order:

1. Read the stage definition from `references/pipeline-stages.md`
2. Confirm all inputs are available (from prior stage output or human-provided)
3. Check skill status:
   - If MISSING → emit gap notice (see Gap Handling), stop
   - If EXISTS → continue
4. Execute the stage:
   - Spawn one agent per unit of work (one for sequential stages, one per component for parallel stages)
   - Each agent receives a complete context package, writes its output to the defined file path (see `references/pipeline-stages.md`), and terminates
   - Agents do not communicate with each other — if a package is incomplete, the agent cannot complete its work
   - For parallel stages: collect all output file paths before proceeding to validation
5. Validate (before the human sees anything):
   - Read the validation fields from each output file — not full content
   - Apply the stage's validation criteria from `references/pipeline-stages.md`
   - If validation fails: spawn a new agent with the prior output file + the specific failure + correction instructions. Repeat until validation passes. Do not surface failed output to the human.
6. Present the human gate review package (see Human Gate Format below):
   - Include a summary of what was produced and that validation passed
   - Reference the output file paths — the human reads the files directly
   - Do not reproduce full artifact content in the gate presentation
7. Record the gate decision and update the state document
8. On APPROVE: advance to next stage
9. On REJECT with feedback: spawn a new agent per rejected item with the prior output file + the specific feedback. Re-validate, then re-present. Items the human approved in a prior presentation do not re-run.

---

## Artifact Model

Agents write to files. The orchestrator stays lean.

**Agent output:** Each agent writes its stage output to the file path defined in `references/pipeline-stages.md`. The agent does not return content inline — it writes the file and reports the path.

**Orchestrator reads:** Validation fields only — never full artifact content. Full content is consumed by the human at the gate, and by downstream agents as part of their context package.

**Context packages:** Each agent receives every file it needs, listed explicitly in the stage definition. Nothing is implicit. If a file is not in the package, the agent does not have it.

**Rework loops:** When a stage is rejected — by validation or by human — a new agent is spawned. That agent receives: the prior output file, the specific rejection reasons, and correction instructions. It does not receive the full reasoning of the prior agent.

**State document:** `pipeline-state.md` records file paths and statuses, not content. It is the orchestrator's complete view of the pipeline. On resume, the orchestrator loads `pipeline-state.md` and continues from the recorded state — no reconstruction needed.

---

## Gap Handling

When a required skill does not exist, emit the following and stop:

```
─────────────────────────────────────────────────
GAP — Stage [N]: [Stage Name]
─────────────────────────────────────────────────
Missing skill:   [skill-name]

This skill must:
[One paragraph describing what the skill does, what concerns it surfaces,
and what problems it exists to solve — not how it works internally.]

Inputs it receives:
[Copy the exact inputs from the stage definition]

Outputs it must produce:
[Copy the exact outputs from the stage definition]

Human gate it manages:
[Copy the gate definition from the stage definition]

Pipeline paused at Stage [N]. Build this skill to continue.
─────────────────────────────────────────────────
```

The gap notice surfaces what needs to be built. It is the output for this stage — not a partial result.

---

## Human Gate Format

Every stage gate uses this exact structure. Consistency reduces the human's cognitive load across a
pipeline that may span many sessions.

```
════════════════════════════════════════════════════
STAGE [N] — [Stage Name] — READY FOR REVIEW
════════════════════════════════════════════════════

Completed:
[One or two sentences on what was produced and why it matters.]

── Artifacts ──────────────────────────────────────
[Summary of what was produced — one or two sentences per artifact.
File references for the human to read directly.
Do not reproduce full artifact content here — reference the files.]

── Decision ───────────────────────────────────────
APPROVE  → I advance to Stage [N+1]: [one sentence on what that stage does]
REJECT   → Provide specific feedback. I rework the rejected items and re-present.
           Previously approved items do not re-run.

On approval I advance automatically. No further input needed until Stage [N+1]'s gate.
════════════════════════════════════════════════════
```

---

## Parallelism

**Parallel stages:** 3 (Component Specification), 4 (Implementation), 5 (Component Testing).
**Sequential stages:** 1 (Requirements Engineering), 2 (System Design), 6 (Integration Testing).
**Unit of parallelism:** one component from Stage 2's approved component list.

For each parallel stage, prepare one self-contained context package per component before spawning agents.
Each package must include everything the agent needs — listed explicitly in the stage definition in
`references/pipeline-stages.md`. If a file is not in the package, the agent does not have it.

Stage 3 packages include `references/engineering-baseline.md` from this skill. This is not optional.
The baseline is the orchestrator's enforcement mechanism — it is included in every Stage 3 package
regardless of whether the component spec references it.

Parallel agents share no state. If a package is incomplete, the agent cannot complete the stage.

**Stage 5 internal parallelism:** within each component's testing agent, three sub-tasks run in parallel:
- Lint — code-integrity-guardrail Phase 0
- Behavioral tests — spec-driven-testing
- Specification compliance — spec-audit (currently MISSING — see Stage 5 definition)

The component's testing agent collects all three results before reporting to the orchestrator.

---

## State Management

The pipeline state document records progress across sessions. Write it after every stage completes
and after every human gate decision. On resume, read it before doing anything else.

Write the state document to `pipeline-state.md` in the project's working directory.

State document schema: `references/pipeline-state.md`

---

## Commissioning

Commissioning is a milestone, not a stage. When Stage 6 is complete and the human accepts
the integration test results:

```
════════════════════════════════════════════════════
COMMISSIONED — [Project Name]
════════════════════════════════════════════════════
All six stages complete. Human has accepted delivery.

Gaps encountered during this pipeline:
[List any skills that were missing, or "None"]

Pipeline state archived to: pipeline-state.md
════════════════════════════════════════════════════
```

The pipeline does not continue after commissioning. The state document is the final record.

---

## Pipeline Stage Definitions

Full stage contracts — inputs, skill status, parallelism, output schema, human gate, dependencies:

→ `references/pipeline-stages.md`
