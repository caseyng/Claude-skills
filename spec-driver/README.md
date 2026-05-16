# spec-driver

**Version:** 1.0.0

Drives a single component's §1–§23 specification to Implementation Readiness = READY
using a three-agent cycle. Used at Stage 3 of the software-development-orchestrator pipeline.

---

## Why this exists

A single agent that writes a spec and then verifies it has two structural failure modes:

1. **Bias.** It cannot objectively critique what it just wrote. An agent that wrote §7
   (failure taxonomy) will tend to find its own taxonomy adequate during verification.

2. **Context pressure.** By the third iteration of a large spec, the draft alone may crowd
   out the verification tools in the context window, producing shallow verification.

Both are structural — they cannot be fixed by better prompting. Three agents with clean
context packages eliminate both.

---

## The three agents

| Agent | Receives | Produces | Constraint |
|---|---|---|---|
| **Drafter** | Component definition, engineering baseline, spec-contract-methodology, prior Judge instructions | §1–§23 draft | Does not verify. Places FLAGS for ambiguities. |
| **Critic** | The draft, verification tools (Tools A+B+C), prior decision log | Gap report | Does not write or fix. Reports only. Fresh context — no attachment to the draft. |
| **Judge** | The draft, the gap report, decision log history | READY verdict or fix instructions | Does not write or verify. Applies the single test to each gap. Classifies binding/non-binding. |

---

## Cycle protocol

```
Drafter → Critic → Judge → [READY or loop]
                            ↑_____________|
                            max 3 cycles, then ESCALATED
```

On READY: final spec written. On ESCALATED after 3 cycles: escalation entry written to decision log, orchestrator presents escalation gate to human.

---

## Invocation

Invoked by the software-development-orchestrator at Stage 3 — not directly by the human.

---

## Relationship to spec-contract-methodology

spec-driver **orchestrates** spec-contract-methodology — it does not replace it.

- The Drafter uses `spec-contract-methodology` to write the §1–§23 structure
- The Critic uses `spec-contract-methodology`'s verification tools (Tools A, B, C)
- The Judge uses `spec-contract-methodology`'s gap classification criteria

spec-contract-methodology is the content authority. spec-driver is the process authority.

---

## Structure

```
SKILL.md        — LLM execution entry point (orchestration protocol)
CHANGELOG.md    — Version history
README.md       — This file
```

No reference files — all role definitions and protocols are in SKILL.md. The actual
specification content (section definitions, verification tools) lives in spec-contract-methodology.
