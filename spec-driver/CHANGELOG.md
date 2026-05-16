# Changelog — spec-driver

## 1.0.0 — 2026-05-16

Initial release.

### What's in it

Three-agent cycle for Stage 3 component specification:

- **Drafter** — writes §1–§23 draft using `spec-contract-methodology`. Iteration 2+
  receives Judge's blocking gap instructions + decision log as constraints.
- **Critic** — runs Tools A+B+C (completeness sweep, stress tests, structural verification)
  against the draft. Fresh context — no attachment to what the Drafter wrote. Does not fix.
- **Judge** — classifies each gap as blocking/non-blocking using the single test. Produces
  READY verdict or concrete fix instructions for the next Drafter. Does not fix or rewrite.

Cycle cap: 3 full Drafter→Critic→Judge cycles. On cap with remaining blocking gaps:
escalation entry written to decision log, ESCALATED status returned to orchestrator.

Intermediate artefacts (draft, gaps, verdict) are overwritten each cycle. History is in
the decision log. Final spec written only on READY verdict.

### Design rationale

**Why separate agents instead of one agent looping internally?**
A single agent that writes and then verifies its own spec operates under two failure modes:
(1) bias — it cannot perceive its own blind spots; (2) context pressure — by cycle 3, the
spec draft alone may crowd out verification tools. Separate agents with clean context
packages avoid both. The separation is a correctness mechanism.

**Why a Judge as a third agent?**
Without the Judge, the Drafter-Critic loop tends toward one of two failure modes: the
Critic reports too many gaps as blocking (over-critical), or the Drafter iterates without
converging (the gap list shrinks but never reaches zero). The Judge applies the single test
independently — no attachment to either role's output — and makes the READY/RETURN decision
with explicit rationale. This is the convergence gate.

**What this replaces**
`assisted-epistemology` (v0.1-draft) was designed to do a similar job but predated the
pipeline architecture and the three-role framing. It is superseded by spec-driver for
Stage 3 work. The multi-role insight from assisted-epistemology is preserved here in a
cleaner form: fixed roles, clean context packages, explicit handoff contracts.

### Origin

Stage 3 of the software-development-orchestrator pipeline. The design emerged from
recognising that single-agent spec verification has two fundamental failure modes (bias
and context pressure) that are structural, not solvable by better prompting. The three-agent
architecture was chosen specifically to address these failure modes — the Critic must be a
different agent from the Drafter, and the Judge must be different from both.
