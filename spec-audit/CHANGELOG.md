# Changelog — spec-audit

## 1.0.0 — 2026-05-16

Initial release.

### What's in it

- Three-phase protocol: contract extraction (spec only), implementation analysis, report production
- Ordering constraint: contracts extracted completely before implementation is read — prevents
  rationalization (reading compliance into code by anchoring on what the code already does)
- Contract taxonomy: maps each spec-contract-methodology section (§5–§23) to the types of
  MUST contracts it produces and explains how to verify each type
- Severity classification: CRITICAL (breaks caller guarantee), HIGH (edge-case violation),
  MEDIUM (partial compliance)
- PASS/FAIL rule: zero CRITICAL, zero HIGH required for PASS
- Unverifiable flag: contracts that require runtime execution are flagged for spec-driven-testing

### Design decisions

- One agent, not two. spec-driven-testing uses two agents to enforce source/test independence.
  spec-audit needs no independence — it explicitly reads both spec and implementation. The
  protection mechanism is the ordering constraint, not agent separation.
- PASS/FAIL ignores MEDIUM and UNVERIFIABLE. MEDIUM violations are real but survivable at
  handoff — they indicate partial compliance that could produce subtle bugs, not missing
  contracts. UNVERIFIABLE items are delegated to spec-driven-testing.
- §22 (Assumptions) is not audited as implementation contracts. Assumptions are caller
  responsibilities. Exception: if the spec explicitly says "this component MUST validate
  assumption X," that is auditable and belongs in §5 or §8 where it is stated.
- Satisfied entries are listed by ID only — no detail. The report's value is in violations
  and unverifiables. Reproducing evidence for satisfied contracts wastes space and buries findings.

### What this unlocks in Stage 5

Stage 5 now has three parallel sub-tasks per component, each catching a different class of defect:
lint (code-integrity-guardrail), behavioral (spec-driven-testing), structural compliance (spec-audit).
A component that passes all three is clean at code quality, behavior, and contract level.

### Origin

Identified as a gap in the skill pipeline alongside spec-driven-testing. spec-driven-testing
was built to catch behavioral violations. spec-audit fills the remaining gap: structural
non-compliance that passing tests cannot surface. The canonical example is a wrong failure
name — the caller's error handler matches on a name the implementation never produces.
