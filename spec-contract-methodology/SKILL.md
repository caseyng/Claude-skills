---
name: spec-contract-methodology
version: 1.7.0
description: >
  Complete methodology for writing software specifications with enough precision
  that an implementor in any language could produce a behaviourally equivalent system.
  Use this skill whenever the user wants to write, review, or improve a specification —
  including design-first greenfield projects, existing-code documentation, iterative
  spec improvement, or spec review. Trigger on: "write a spec", "spec out this system",
  "document this design", "spec review", "formalise this design", "specification contract", or any request to
  produce or improve a structured system specification. Also trigger proactively when
  a design is being discussed and a spec would prevent future rework.
---

# Specification Contract Methodology

## Role

Active co-author. Drive the process — propose what to tackle next, ask targeted
questions to elicit what's needed per section, flag underspecification before moving on.
The user reviews and approves. Neither party is passive.

## Goal

The goal of this methodology is **implementation readiness** — confirming that no blocking
gap exists that would cause a competent implementor to build divergent behaviour. It is
not documentation completeness. Every decision in the methodology serves this goal.
When verification finds gaps, the question is always: blocking or non-blocking? That
answer determines what to fix before handoff. Non-blocking gaps are recorded but do not
hold up implementation.

---

## The Test

Hand this document to a competent engineer who has never seen the codebase. They must
produce a behaviourally equivalent system without reading the original code. Any
ambiguity they'd need the code to resolve = spec gap.

**Design-first mode:** the same test applies — the spec must be precise enough that no
implementation decision is left ambiguous. The standard is identical.

---

## Two Starting Modes

**Existing code → spec.** Read all materials (code, docs, tests, comments) before
writing. Where code and stated intent conflict, resolve before writing — a spec that
describes a known bug is a transcript, not a spec.

**Design first → spec.** No code to read. Drive from intent. Flag every undecided
decision as an open question in §22 or as a provisional contract in §20.

In both modes: resolve conflicts before writing. Note every resolution.

---

## How Sections Relate

Read this before writing anything. These relationships prevent duplication and gaps.

- **§2b Architectural Constraints** — structural decisions that cause wrong implementation if violated: class hierarchy, inheritance strategy, component priority, named patterns. Declared before component contracts. Distinct from §5 (which specifies contracts assuming structure already exists), §11 (pipeline sequencing, not registry priority), and §16 (extension contracts, not structural roles).
- **§4 Data Contracts** — structured data at public boundaries. §18 covers event fields.
  §15 covers config fields. §4 covers everything else: API schemas, file formats,
  wire protocols, serialisation conventions.
- **§5 Component Contracts** — master interface description per component. References
  failure modes by name but does not define them.
- **§7 Failure Taxonomy** — master list of every failure mode. A failure MUST appear
  in §7 before it can be referenced in §5 or §17.
- **§8 Boundary Conditions** — exact behaviour at edges. §5 states the rule; §8 covers
  the edges an implementor is most likely to get wrong.
- **§10 Atomicity and State on Failure** — internal data state after failure. §7 = what
  caller sees. §17 = how error travels. §10 = what internal state is left.
- **§12 Interaction Contracts** and **§16 Extension Contracts** — both cover resource
  ownership at boundaries. Must be cross-referenced and consistent.
- **§22 Assumptions** vs **§14 External Dependencies** vs **§15 Configuration** —
  three distinct categories. §22 = taken as given. §14 = system reaches out to.
  §15 = operator supplies. Do not conflate.
- **§23 Performance Contracts** vs **§21 What Is Not Specified** — §23 carves out
  properties that look like performance but are correctness contracts. §21 defers
  the rest to implementors.

---

## Workflow

1. **Profile first.** Identify which system shapes apply (`references/spec-profiles.md`).
   The profile table gives the minimum required sections.

2. **Before You Write checklist.** Run the 25-question checklist in
   `references/writing-process.md`. Map each answer to its section. Answer with no
   section = gap. Section with no answer = underspecified area.

3. **Write sections in order.** §1 through §23. Each section either has content or an
   explicit inapplicability statement (one sentence, stating why and what property makes
   it inapplicable — this is a contract, not an omission).

4. **Integrated verification pass.** Run all three tools together in a single pass, in
   order. Do not split across sessions — each tool re-grounds on the previous tool's
   findings, and splitting breaks that chain.

   **Tool A — Completeness sweep.** Run the 25-question checklist from
   `references/writing-process.md` again, this time as a verification sweep against the
   completed spec. Each question the spec cannot answer is a gap. Produces:
   `checklist-findings.md`.

   **Tool B — Stress tests.** Run all 8 vectors from `references/stress-tests.md`.
   Re-ground on `checklist-findings.md` first to avoid re-finding known gaps. Produces:
   `stress-findings.md`.

   **Tool C — Structural verification.** Run the LLM-Executable Verification Pipeline
   (Steps 1–5) from `references/verification-pipeline.md`. Re-ground on
   `stress-findings.md` before Step 1. Each step externalises its artefact before the
   next step begins. Produces: `registry.md`, `matrix.md`, `invariants.md`, `paths.md`,
   `intent-notes.md`.

   Assemble all findings into a single gap report (format in
   `references/verification-pipeline.md`). Classify every gap as blocking or
   non-blocking using the single test: would a competent implementor, reading the spec
   without this, build divergent behaviour? The spec is ready for implementation
   hand-off when Implementation Readiness is READY (zero blocking gaps) and
   Verification Currency is CURRENT.

5. **Completeness check.** Run the "When the Spec Is Done" checklist in
   `references/writing-process.md`. For every observable behaviour: find where the spec
   covers it.

**Do not hand a spec to a coding LLM unless Implementation Readiness is READY and Verification Currency is CURRENT.**

6. **Post-implementation phase.** After the coding LLM delivers an implementation, run the
   Post-Implementation Coverage Audit defined in `references/verification-pipeline.md`. For
   every RFC 2119 MUST contract in the spec, confirm it is present in the implementation or
   record a deviation. Each deviation is classified:

   - **Implementation drift** — spec is correct; fix the implementation.
   - **Spec error revealed** — the implementation exposes a spec error; amend the spec (not
     the code), re-run verification, re-handoff.

   The gate is `Implementation Coverage = COMPLETE` (zero unresolved deviations). The project
   is not done until this gate is cleared. Deviations MUST NOT be documented as code comments.

---

## Section Index

| Sections | File | Contents |
|---|---|---|
| §1–§5 | `references/sections-1-5.md` | Purpose+Design Principles, Vocabulary, Architectural Constraints (§2b), Boundary, Data Contracts, Component Contracts+Hierarchy |
| §6–§10 | `references/sections-6-10.md` | Lifecycle, Failure Taxonomy, Boundary Conditions, Sentinel Values, Atomicity |
| §11–§16 | `references/sections-11-16.md` | Ordering, Interaction Contracts, Concurrency, Dependencies, Config, Extension Contracts |
| §17–§23 | `references/sections-17-23.md` | Error Propagation+Caller Exception Types, Observability+What Not Logged+Schema Versioning, Security, Versioning, What Not Specified, Assumptions, Performance |
| Writing rules + checklists | `references/writing-process.md` | Writing Rules, Before You Write (25 questions), When Done checklist |
| Stress tests | `references/stress-tests.md` | Appendix A — 8 test vectors |
| Profiles + guidance | `references/spec-profiles.md` | Appendix B profile table, Deployment Guidance, Future Directions, References |
| Verification pipeline | `references/verification-pipeline.md` | Appendix C — integrated three-tool verification pass, status model (readiness/currency), gap report format, design rationale |

---

## Non-Negotiables

- Every section present, or explicitly declared inapplicable with a contract sentence. This includes §2b — either structural constraints are stated or declared absent ("No architectural constraints beyond those implied by §5 component contracts").
- Spec MUST NOT be handed off if §2b has blocking gaps — wrong inheritance strategy or priority ordering is divergent implementation.
- No implementation detail anywhere. Specify what, never how.
- MUST / SHOULD / MAY capitalised and used per RFC 2119 throughout.
- Failure modes in §7 before they can be referenced anywhere else.
- Named exception types in §2 before they can be referenced anywhere else.
- Cross-references between §12 and §16 resource ownership must be consistent.
- Spec MUST NOT be handed to a coding LLM unless Implementation Readiness is READY and Verification Currency is CURRENT.
- Every gap classified as blocking or non-blocking. Handoff gate checks blocking count only.
- Verification is a single integrated pass (Tools A + B + C). Splitting across sessions
  breaks re-grounding and is the primary cause of gaps surviving into implementation.
- The spec document itself is versioned. Substantive changes are recorded.
- Implementation Coverage MUST reach COMPLETE before declaring the project done. Deviations
  route to their correct fix surface (implementation or spec). MUST NOT be documented as
  code comments — a comment is not a resolution.
