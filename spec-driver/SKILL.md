---
name: spec-driver
version: 1.0.0
description: >
  Drives a single component's §1-§23 specification to Implementation Readiness = READY
  using a three-agent cycle: Drafter writes, Critic verifies, Judge classifies and decides.
  Agents are separate to prevent bias and context pressure. Maximum 3 cycles before
  escalation. Used by the software-development-orchestrator at Stage 3.
---

# Spec Driver

## Role

You are a meta-orchestrator within Stage 3. You do not write specs or run verification.
You sequence three agents — Drafter, Critic, Judge — in cycles until the component spec
reaches Implementation Readiness = READY or the cycle cap is exhausted.

You receive the component's context package from the Stage 3 orchestrator. You terminate
by writing the final spec and the decision log entries to the defined file paths.

---

## Why Three Agents, Not One

A single agent that writes a spec and then verifies it operates under two failure modes:

**Bias.** An agent that wrote §7 (failure taxonomy) will tend to find its own taxonomy
adequate during verification. It cannot perceive its own blind spots. A fresh Critic
agent, reading the spec without having written it, applies the verification tools without
attachment to the output.

**Context pressure.** The Drafter context is: system design + component definition +
engineering baseline + spec structure. The Critic context is: the draft spec + verification
tools. The Judge context is: the draft spec + the gap report. If these are one agent, all
of this competes for the same context window over multiple cycles. By the third cycle, the
spec draft alone may be large enough to crowd out the verification tools.

The separation is a correctness mechanism, not an architectural preference.

---

## The Three Roles

### Drafter

**Receives:**
- System Design Document (`output/stage-2-system-design.md`)
- This component's definition extracted from Stage 2
- `references/engineering-baseline.md` (from software-development-orchestrator)
- The `spec-contract-methodology` skill
- Iteration > 1: prior spec draft + Judge's gap classifications from prior cycle + decision log

**Produces:**
- A complete §1–§23 draft spec, written to `output/stage-3-spec-[component]-draft.md`
- Decision log entries: every non-obvious decision made, every section where the input was
  ambiguous and how it was resolved, every provisional FLAG placed

**Constraint:** The Drafter does not run verification. It writes. If the inputs are insufficient
to fill a section, it places a FLAG per the spec-contract-methodology FLAG format and continues.
It does not skip sections. A declared-inapplicable section ("N/A because this component is
stateless") is not a skip — it is a contract.

**Iteration 2+ constraint:** The Drafter MUST address every gap the Judge classified as blocking
in the prior cycle. OPTIONS REJECTED in the decision log are constraints — do not reverse a
prior decision without stating explicitly why the rationale no longer applies.

---

### Critic

**Receives:**
- The draft spec from this cycle's Drafter (`output/stage-3-spec-[component]-draft.md`)
- The `spec-contract-methodology` skill (for verification tools A, B, C)
- Decision log for this component (`output/decisions-stage3-[component].md`) — to avoid
  re-finding gaps already recorded in prior cycles

**Produces:**
- Gap report written to `output/stage-3-gaps-[component].md`
- Decision log entry: which gaps were re-grounded from the prior decision log vs. newly found

**Protocol:** Run all three verification tools in sequence per `spec-contract-methodology`:

1. **Tool A — 25Q completeness sweep.** Run all questions against the draft. Each question the
   spec cannot answer is a gap. Write findings to `checklist-findings.md` (ephemeral, same
   session).

2. **Tool B — 8 stress test vectors.** Re-ground on Tool A findings first. Apply all 8 vectors.
   Write findings to `stress-findings.md` (ephemeral).

3. **Tool C — structural verification (Steps 1–5).** Re-ground on Tool B findings. Build
   registry → matrix → invariants → paths → intent verification. Write artefacts as named
   (registry.md, matrix.md, invariants.md, paths.md, intent-notes.md — ephemeral).

Assemble all findings into the gap report. For each gap: source (Tool A/B/C), entity, type
(MISSING/CONTRADICTION/WEAK_CLAIM/INTENT_FLAG), finding, evidence.

**Constraint:** The Critic does not fix anything. It finds and reports. If the spec is
internally consistent and all questions are answered, the gap report is empty. That is a
valid output.

**Constraint:** Do not re-report gaps already in the decision log from prior cycles unless
the current draft has not addressed them — in that case, report them as UNRESOLVED from
prior cycle, not as new findings.

---

### Judge

**Receives:**
- The draft spec from this cycle's Drafter
- The gap report from this cycle's Critic
- Decision log for this component (full history)

**Produces:**
- A verdict written to `output/stage-3-verdict-[component].md`:
  - For each gap: BLOCKING or NON_BLOCKING with rationale
  - Overall: `READY` (zero blocking gaps) or `RETURN` (list of blocking gaps to fix)
- If `RETURN`: for each blocking gap, a concrete instruction to the next Drafter iteration
  (not a restatement of the gap — an instruction: "§7 must add failure mode X with trigger Y")
- Decision log entry: every blocking/non-blocking classification with the test applied

**The single test:** Would a competent implementor, reading this spec without this entry,
build divergent behaviour? Yes = BLOCKING. No = NON_BLOCKING.

**Constraint:** The Judge applies the test. It does not advocate for the Drafter's choices
or the Critic's findings. If the Critic found a gap that the Judge evaluates as NON_BLOCKING,
the gap is recorded as non-blocking and does not block handoff. The Judge's classification
is final for this cycle.

**Constraint:** The Judge does not fix anything. It classifies and instructs.

---

## Execution Protocol

```
Cycle 1:
  1. Spawn Drafter → draft spec
  2. Spawn Critic (receives draft) → gap report
  3. Spawn Judge (receives draft + gap report) → verdict

  If verdict = READY:
    → Write final spec (copy draft to output/stage-3-spec-[component].md)
    → Write decision log entries for this cycle
    → Terminate with status: READY

  If verdict = RETURN and cycle_count < 3:
    → Write decision log entries for this cycle (including Judge's instructions)
    → Increment cycle_count
    → Proceed to Cycle 2

Cycle 2:
  1. Spawn Drafter (receives draft + Judge's instructions + decision log) → revised draft
  2. Spawn Critic (receives revised draft + decision log) → new gap report
  3. Spawn Judge (receives revised draft + new gap report + decision log) → verdict

  If verdict = READY:
    → Write final spec, decision log entries
    → Terminate with status: READY

  If verdict = RETURN and cycle_count < 3:
    → Write decision log entries
    → Increment cycle_count
    → Proceed to Cycle 3

Cycle 3 (final):
  1–3. Same as prior cycles.

  If verdict = READY:
    → Write final spec, decision log entries
    → Terminate with status: READY

  If verdict = RETURN:
    → Write decision log entries
    → Terminate with status: ESCALATED
    → Write escalation entry to decision log (see below)
    → Do NOT write a final spec to stage-3-spec-[component].md
```

---

## Output Files

| File | Written by | When |
|---|---|---|
| `output/stage-3-spec-[component]-draft.md` | Drafter | Each cycle (overwritten) |
| `output/stage-3-gaps-[component].md` | Critic | Each cycle (overwritten) |
| `output/stage-3-verdict-[component].md` | Judge | Each cycle (overwritten) |
| `output/stage-3-spec-[component].md` | spec-driver | On READY — the final, clean spec |
| `output/decisions-stage3-[component].md` | spec-driver (appends per cycle) | After each cycle |

The intermediate files (draft, gaps, verdict) are overwritten each cycle — they represent
the current cycle's state, not history. History is in the decision log.

---

## Escalation Entry (on cycle cap with RETURN verdict)

When cycle 3 ends with RETURN, append to the decision log:

```
### Escalation — Cycle Cap Reached

Cycles attempted: 3
Blocking gaps remaining: [N]
Persistent gaps that did not resolve across cycles:
  [List each gap that appeared in all three Critic reports and was not resolved]

Drafter instructions from Judge (cycle 3):
  [The concrete fix instructions the next Drafter would receive]

Escalation to orchestrator: BLOCKED — human decision required.
```

The orchestrator receives this via the spec-driver's ESCALATED status and presents the
escalation gate to the human per the Escalation Protocol in the orchestrator SKILL.md.

---

## What spec-driver Does Not Do

- Does not write the spec itself — that is the Drafter's job
- Does not run verification tools — that is the Critic's job
- Does not classify gaps — that is the Judge's job
- Does not modify the spec-contract-methodology's 23-section structure
- Does not skip cycles because "the draft looks good" — every cycle runs all three agents

---

## Integration with pipeline-stages.md Stage 3

The orchestrator spawns one spec-driver agent per component at Stage 3. The spec-driver
is the Stage 3 agent. It terminates by writing `output/stage-3-spec-[component].md` (on
READY) or the escalation entry in the decision log (on ESCALATED). The orchestrator reads
the `implementation_readiness` field from the final spec file to validate Stage 3 output.

**Context package the orchestrator provides to spec-driver:**
- `output/stage-2-system-design.md`
- This component's definition extracted from Stage 2
- `references/engineering-baseline.md` (from software-development-orchestrator)
- The `spec-contract-methodology` skill
- Iteration > 1 (orchestrator-level re-run): prior final spec + decision log

The spec-driver manages the Drafter/Critic/Judge cycle internally. The orchestrator does
not see the intermediate files — only the final spec and the decision log.
