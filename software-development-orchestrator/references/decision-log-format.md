# Decision Log Format

Each stage and component in the pipeline maintains a decision log. The log is the
durable record of what was decided, why, what was rejected, and what human gates produced.
It is the primary mechanism that prevents stateless agents from re-litigating closed decisions
across iterations.

**Two writers, one file.** The agent writes iteration entries during its run. The orchestrator
appends the gate outcome after the human gate. These two writes are always sequential — agent
first, orchestrator after. Never concurrent. Parallel agents each write to their own file.

---

## File Paths

| Stage | Parallelism | Decision log path |
|---|---|---|
| Stage 1 | Sequential | `output/decisions-stage1.md` |
| Stage 2 | Sequential | `output/decisions-stage2.md` |
| Stage 3 | Per component | `output/decisions-stage3-[component-name].md` |
| Stage 4 | Per component | `output/decisions-stage4-[component-name].md` |
| Stage 5 | Per component | `output/decisions-stage5-[component-name].md` |
| Stage 6 | Sequential | `output/decisions-stage6.md` |

---

## Log Header

Written once when the log is first created. Never updated.

```markdown
# Decision Log — Stage [N] — [Stage Name][: Component Name]

Project: [project_name]
Stage: [N]
Component: [name] | —
Opened: [date]
Iteration cap: 3
```

---

## Per-Iteration Entry

One entry per agent run. The agent writes all fields up to and including Iteration Outcome.
The orchestrator appends the Gate Decision block after the human gate resolves.

```markdown
---

## Iteration [N] — [date]

Prior iteration artifact: [file path] | —
Gate feedback applied: [summary of what the previous gate said, or "—"]

### Decisions Made

<!-- One row per non-obvious decision. Obvious or forced decisions need not be listed. -->

| Decision | Why | Tradeoffs Accepted |
|---|---|---|
| [decision] | [rationale] | [what was given up] |

### Options Rejected

<!-- Every option that was seriously considered and discarded. This is the flip-flop prevention record.
     A new iteration MUST NOT reverse a rejection without stating why the rationale no longer applies. -->

| Option | Why Rejected |
|---|---|
| [option] | [reason it was set aside] |

### Ambiguities Resolved

<!-- Ambiguous inputs or open questions from prior iterations resolved in this run. -->

| Ambiguity | Resolution | Rationale |
|---|---|---|
| [ambiguity] | [how it was resolved] | [why this resolution over alternatives] |

### Open Questions Carried Forward

<!-- Questions this iteration could not close. Each must name why it remains open. -->

| Question | Why Not Resolved | Options |
|---|---|---|
| [question] | [reason still open] | [A or B] |

### Iteration Outcome

Status: CONVERGED | NOT_CONVERGED | BLOCKED
Output artifact: [file path]
[Stage 3 only] Implementation Readiness: READY | NOT READY — [N blocking gaps]
[Stage 3 only] Residual gaps: [N non-blocking]
Notes: [anything the next iteration or the human needs to understand]

<!-- Orchestrator writes below this line after the human gate resolves. Agent does not write here. -->

### Gate Decision

Outcome: APPROVED | REJECTED | ESCALATED | AUTO_ADVANCED
Date: [date]
Feedback: [human feedback verbatim, or "None — approved" or "None — auto-advanced"]
Next action: [advanced to Stage N+1 | rework spawned: iteration N+1 | escalated: [reason]]
```

---

## Escalation Entry

Written by the orchestrator — not an agent — when the iteration cap is reached without convergence.
The escalation entry replaces what would have been a Gate Decision block on the final iteration.

```markdown
### Gate Decision

Outcome: ESCALATED — ITERATION CAP REACHED
Date: [date]
Iterations attempted: 3
Persistent contention: [one paragraph: the specific decision or question that has not converged
  across all iterations, with the two or more positions that keep recurring]
Options presented to human:
  A: [option A — what choosing this resolves and what it requires]
  B: [option B — what choosing this resolves and what it requires]
  C: [option C, if applicable]
Human decision: [OPEN — awaiting human input]
```

The orchestrator presents this to the human in the Escalation Gate format (see SKILL.md).
Once the human decides, the orchestrator appends:

```markdown
Human decision: [A | B | C | custom decision]
Decision rationale: [human's stated reason, or "none given"]
Next action: [rework spawned with human decision as constraint | stage abandoned | stage returned to prior stage]
```

---

## How a New Agent Reads the Decision Log

When an agent receives a decision log as part of its context package, it MUST:

1. Read all OPTIONS REJECTED entries. These are closed. To reverse a rejection, the agent
   must explicitly state why the rejection rationale no longer applies — not just re-choose
   the rejected option.

2. Read all DECISIONS MADE entries. These are constraints on this iteration. Override only
   with explicit justification tied to new information.

3. Read GATE FEEDBACK APPLIED. This is what the previous human gate said. The agent's
   work must address this feedback specifically — not generically.

4. Read OPEN QUESTIONS CARRIED FORWARD. These are unresolved items the agent should attempt
   to close. If an open question cannot be closed, the agent must state why and carry it forward.

The decision log is not background reading. It is a constraint document. An agent that ignores
it and reconsiders closed options is producing a lower-quality output than one that worked
within the established constraints.

---

## What Counts as a Decision Worth Recording

Record: design decisions with non-obvious rationale, options that were genuinely considered
before being discarded, resolution of ambiguous inputs, human gate feedback and how it was addressed.

Do not record: forced choices with no alternatives, mechanical completeness work, decisions that
follow directly from the spec or requirements without any judgment.

When in doubt, record. A decision log with too much context is better than one that misses the
entry that would have prevented a flip-flop.
