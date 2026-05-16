# spec-contract-methodology

**Version:** 1.5.0

Specification methodology that eliminates implementor ambiguity before code is written. A spec that passes verification has zero blocking gaps — every decision a competent engineer would reach for the code to resolve has been made explicit.

---

## Purpose

LLMs fill specification gaps with assumptions. Those assumptions become bugs. This skill enforces a 23-section methodology rigorous enough that any engineer, in any language, could rebuild the system without reading the original code.

The goal is **implementation readiness**, not documentation completeness. Every decision in the methodology serves a single question: would a competent implementor diverge without this? If yes, it's blocking. If no, it's not.

---

## Invocation

```
/skill:spec-contract-methodology
```

Triggers on: "write a spec", "spec out this system", "document this design", "spec review",
"formalise this design", "specification contract". Also triggers proactively when a design
is being discussed and a spec would prevent future rework.

---

## What It Covers

| Section | Content |
|---|---|
| §1 | Purpose and Design Principles |
| §2 | Concepts and Vocabulary |
| §2b | Architectural Constraints — class hierarchy, inheritance strategy, priority ordering, named patterns |
| §3 | System Boundary |
| §4 | Data Contracts |
| §5 | Component Contracts |
| §6 | Lifecycle |
| §7 | Failure Taxonomy |
| §8 | Boundary Conditions |
| §9 | Sentinel Values |
| §10 | Atomicity and State on Failure |
| §11 | Ordering and Sequencing |
| §12 | Interaction Contracts |
| §13 | Concurrency |
| §14 | External Dependencies |
| §15 | Configuration |
| §16 | Extension Contracts |
| §17 | Error Propagation |
| §18 | Observability |
| §19 | Security |
| §20 | Versioning and Stability |
| §21 | What Is Not Specified |
| §22 | Assumptions |
| §23 | Performance Contracts |

Every section is either populated or explicitly declared inapplicable with a one-sentence contract stating why.

---

## Verification

Verification is a single integrated pass combining three tools in sequence:

- **Tool A** — 26-question completeness sweep
- **Tool B** — 8 adversarial stress test vectors
- **Tool C** — 5-step structural verification pipeline with externalised artefacts

Each tool re-grounds on the previous tool's findings. Splitting across sessions breaks re-grounding and is the primary cause of gaps surviving into implementation.

**Handoff gate:** Implementation Readiness = READY (zero blocking gaps) and Verification Currency = CURRENT.

---

## Structure

```
SKILL.md                          — LLM execution entry point
references/
  sections-1-5.md                 — §1–§5 section definitions
  sections-6-10.md                — §6–§10 section definitions
  sections-11-16.md               — §11–§16 section definitions
  sections-17-23.md               — §17–§23 section definitions
  writing-process.md              — Writing rules, 26-question checklist, done checklist
  stress-tests.md                 — Appendix A: 8 stress test vectors
  spec-profiles.md                — Appendix B: system shape profiles, minimum required sections
  verification-pipeline.md        — Appendix C: integrated verification pass, gap report format, status model
CHANGELOG.md                      — Version history
README.md                         — This file
```

---

## Design Rationale

**Why the goal is implementation readiness, not documentation completeness.** A spec that chases zero gaps has no convergence criterion — every resolved gap introduces new entities the next pass examines. The correct exit condition is zero blocking gaps. Non-blocking gaps do not cause implementation drift. Chasing them past readiness produces diminishing returns indefinitely.

**Why blocking/non-blocking is a single test, not a category system.** The test is: would a competent implementor, reading the spec without this entry, build divergent behaviour? That question has a direct answer. A category system invites negotiation. The single test does not.

**Why the three status dimensions are orthogonal.** Implementation Readiness is derived from the gap list. Verification Currency is independent — it measures whether the gap list is still valid. Implementation Coverage measures post-implementation conformance. Conflating any two (e.g. a single VERIFIED state) produces false confidence when only one condition is met. Separating them makes each risk independently visible.

**Why currency is change-count, not time-based.** A spec unchanged for two years is as valid as one verified yesterday. Time is not the variable — changes are. The invalidation table defines which changes matter. Currency counts them.

**Why a fixed cross-reference matrix?** Per-spec matrix generation reintroduces the self-verification problem: the LLM checks its own work using rules it derives from the same work. The fixed matrix is a methodology-level contract, applied by the LLM but not generated by it.

**Why Tool A runs before Tool B.** The 25Q checklist produces findings in specific categories (boundary conditions, sentinel values, concurrency) that stress test Vectors 1–3 and 6–8 also probe. Running checklist first means stress tests re-ground on what's already found and focus on adversarial reading strategies the enumerated questions don't cover.

**Why Tool B runs before Tool C.** Stress test findings inform what belongs in the registry. A stress test that flags a missing lifecycle state should cause that state to appear in the registry so Tool C Step 2 checks it in the cross-reference matrix. Without this sequencing, the state gets missed by the matrix check because it was never registered.

**Why must the three verification tools run in a single pass?** Running them across separate sessions means each session starts without prior tools' findings in context. A gap found by Tool A that should re-ground into Tool C Step 1 instead gets re-discovered in Step 3 or missed entirely. The integrated pass eliminates this by keeping all findings in the same context window.

**Why the spec has a status model.** Without explicit readiness state, "done" is ambiguous. A coding LLM handed a NOT READY spec will implement against unresolved ambiguities.

**Why flags must name the alternative interpretation.** A flag that says "ambiguous" is useless for async review. The human needs to see what the LLM almost decided.

**Why gap entries reference the fixed rule.** Not just "§17 is missing" — but "missing because: failure modes MUST have a propagation path per the fixed matrix." Without the rule, the human cannot distinguish a genuine gap from a false positive.

**Why verification re-runs after resolving blocking gaps.** Resolving a blocking gap is a spec change. A spec change can introduce new gaps. After resolving blocking gaps, re-run affected steps before confirming READY.

**Why Step 5 cannot be automated.** Steps 1–4 verify internal consistency. Step 5 verifies intent — that the spec describes the right thing. A spec can be internally consistent and consistently wrong. No structural check detects that. It requires a human who understands the domain.

**Why human review receives spec and gap report together.** A gap report without a spec is abstract. A spec without a gap report asks the human to re-do the verification.

**Why §2b as a distinct section from §5?** §5 specifies component contracts assuming structure already exists. §2b declares the structural decisions themselves — inheritance strategy, priority ordering, named patterns — that, if left unspecified, cause an implementor to build the wrong structure before any contracts are evaluated.

**Why the Post-Implementation Coverage Audit is separate from pre-handoff verification.** Pre-handoff verification (Tools A + B + C) checks that the spec is internally consistent and implementation-ready. It cannot check the implementation — the implementation does not exist yet. The audit checks that the implementation satisfies the spec. These are orthogonal concerns at different lifecycle points. Conflating them produces a workflow where implementation gaps are never systematically caught.

**Why deviations route to a fix surface, not to code comments.** A code comment documenting a spec deviation is not a resolution — it is a deferral. The comment is invisible to the spec, invisible to the next verification pass, and invisible to the next implementor who reads the spec. Every deviation must either change the implementation or change the spec.

**Why `SPEC_ERROR_REVEALED` deviations require spec amendment and re-verification.** An implementation that diverges from the spec because the spec is wrong is not an implementation bug. Fixing the code to match a wrong spec produces a conforming-but-incorrect system. The spec is the source of truth — its authority holds only when it is correct.

**Why code quality verification is not a step in the coverage audit.** Code quality (security, logic correctness, hallucinated packages) is orthogonal to spec fidelity. The audit asks: does the implementation satisfy the MUST contracts? Code quality asks: is the implementation correct code? Running them together muddles classification — a code quality finding is not a DEVIATION, and a coverage gap is not a code quality issue.
