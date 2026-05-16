---
name: software-development-orchestrator
version: 1.2.0
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
   - First iteration: agent receives the standard context package for this stage
   - Subsequent iterations: agent receives the standard context package + prior iteration artifact + decision log for this stage/component
   - Each agent writes: its output artifact + its iteration entry in the decision log
   - Agents do not communicate with each other — parallel agents each own their own decision log and output files
   - For parallel stages: collect all output file paths before proceeding to validation
5. Validate (before the human sees anything):
   - Read the validation fields from each output file — not full content
   - Apply the stage's validation criteria from `references/pipeline-stages.md`
   - If validation fails and iteration count < cap (3): spawn a new agent with prior output + decision log + specific failure + correction instructions. Increment iteration count. Re-validate.
   - If validation fails and iteration count = cap: escalate (see Escalation Protocol below). Do not spawn another agent.
   - Do not surface failed output to the human.
6. Update per-component state files immediately after validation (before gate presentation). Merge into `pipeline-state.md`.
7. Present the human gate review package (see Human Gate Format below):
   - Include a summary of what was produced, iteration count, and that validation passed
   - Reference the output file paths and decision log path — the human reads the files directly
   - Do not reproduce full artifact content in the gate presentation
8. Append the gate outcome to the decision log. Update the state document.
9. On APPROVE: advance to next stage
10. On REJECT with feedback:
    - Record human feedback in the decision log's Gate Decision block
    - If iteration count < cap (3): spawn a new agent per rejected item with prior output + decision log + rejection feedback. Re-validate, then re-present. Previously approved items do not re-run.
    - If iteration count = cap: escalate (see Escalation Protocol below).

---

## Artifact Model

Agents write to files. The orchestrator stays lean.

**Agent output:** Each agent writes its stage output to the file path defined in `references/pipeline-stages.md`. The agent does not return content inline — it writes the file and reports the path.

**Orchestrator reads:** Validation fields only — never full artifact content. Full content is consumed by the human at the gate, and by downstream agents as part of their context package.

**Context packages:** Each agent receives every file it needs, listed explicitly in the stage definition. For iterations after the first, the context package also includes the prior iteration artifact and the decision log. Nothing is implicit. If a file is not in the package, the agent does not have it.

**Rework loops:** When a stage is rejected — by validation or by human — a new agent is spawned. That agent receives: the prior output file, the decision log, the specific rejection reasons, and correction instructions. The decision log is the mechanism that prevents the new agent from re-litigating already-closed decisions.

**Decision logs:** Each iterating stage produces a decision log alongside its output artifact. Format: `references/decision-log-format.md`. Two writers — the agent writes iteration entries; the orchestrator appends gate outcomes. Always sequential, never concurrent. Parallel agents each own their own decision log file.

**State files:** Two levels of state are maintained. Per-component state files track iteration count, current artifact, and blocking issues for each unit of parallel work. The pipeline-level `pipeline-state.md` is the merged human-readable view, updated after every agent completion. Format: `references/pipeline-state.md`. The pipeline-state.md is the human's query surface — reading it answers "where are things" without interrogating the orchestrator.

**State document:** `pipeline-state.md` records file paths, statuses, iteration counts, and decision log references. It is the orchestrator's complete view of the pipeline. On resume, load it and continue from the recorded state.

---

## Escalation Protocol

When a stage/component reaches the iteration cap (3) without passing validation or receiving
human approval, do not spawn another agent. Escalate.

**How to escalate:**

1. Read the complete decision log for this stage/component
2. Identify the persistent contention: the specific question or decision that has not converged
   across all iterations — the one that each agent re-opens or that the human keeps rejecting
3. Identify the competing positions: what position does each iteration land on, and what is the
   reasoning in each case
4. Update the state file for this component: `Status: ESCALATED`
5. Update `pipeline-state.md`
6. Present the Escalation Gate to the human:

```
════════════════════════════════════════════════════
ESCALATION — Stage [N] — [Component Name]
════════════════════════════════════════════════════

3 iterations attempted without convergence.

Persistent contention:
[One paragraph: the specific question or decision that has not resolved.
 State both positions without editorializing.]

Decision log: [path]
Latest artifact: [path]

── Options ────────────────────────────────────────
A: [option A — what it means, what it resolves, what it requires]
B: [option B — what it means, what it resolves, what it requires]
[C: if applicable]

── Decision ───────────────────────────────────────
Choose an option above, or provide your own direction.
I will resume this component with your decision as a hard constraint.
════════════════════════════════════════════════════
```

7. On human decision: record the decision in the decision log (Escalation Entry → Human decision field). Spawn one final iteration with the human decision as an explicit constraint in the context package. If this iteration also fails, surface the output directly to the human — do not escalate again.

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
Iteration: [N of 3] (first attempt | rework after [feedback summary])

── Artifacts ──────────────────────────────────────
[Summary of what was produced — one or two sentences per artifact.
File references for the human to read directly.
Do not reproduce full artifact content here — reference the files.]

Decision log: [path] — records all decisions made, options rejected, and prior gate feedback

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

Two levels of state. Both written to the project's working directory.

**Per-component state files** (parallel stages only): One file per component per stage.
Path pattern: `output/state-stage[N]-[component-name].md`. Written by the orchestrator after
each agent completion. Never written by agents — the orchestrator owns state.

**Pipeline state** (`pipeline-state.md`): The merged human-readable view. Written after every
agent completion and after every human gate decision. This is the human's query surface —
the human reads it to understand pipeline status without interrupting the orchestrator.

**Update cadence:**
- After every agent completes (before gate presentation): update the component's state file + merge into `pipeline-state.md`
- After every gate decision: append gate outcome to decision log + update `pipeline-state.md`
- On resume: read `pipeline-state.md` first, then the decision logs for any in-progress stages

**On resume:** Load `pipeline-state.md`. For any stage/component that is `in_progress` or
`awaiting_gate`, read its decision log to reconstruct what was being worked on. Do not
reconstruct from artifact content — the decision log is authoritative for iteration state.

State schemas: `references/pipeline-state.md`

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
