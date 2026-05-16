# Changelog

## 1.8.0 — 2026-05-16

### Changed

- **Removed "Profile first" step from workflow.** Workflow previously opened with a step
  instructing the user to select their component's shapes from the profile table. This step
  is now internal to the Stage 3 agent: the orchestrator determines applicable shapes from
  the System Design Document. Workflow renumbered from 6 steps to 5.
- **Trimmed spec-profiles.md.** Removed Deployment Guidance, Future Directions, and
  References sections — these were planning and human-orientation content, not execution
  material. Kept: section requirements table (R/C/N per section per shape) and two notes.
  Added pipeline context note: Stage 3 agent determines shapes from System Design Document.
- **Moved Design Rationale from verification-pipeline.md to README.md.** ~90 lines of
  "Why..." entries belong in README (human-facing orientation), not in the appendix
  (LLM execution reference). Verification-pipeline.md is loaded into agent context;
  rationale that does not affect execution added unnecessary context load.
- **Updated section index row** for spec-profiles.md: "Profiles + guidance" → "Section requirements".
- **Updated Tool A question count** in SKILL.md workflow: "26-question" → "28-question"
  (reflecting Q27 and Q28 added in v1.7.0).

### Rationale

Profile selection was a human-facing step that predated the orchestrator pipeline. In the
pipeline, the Stage 3 agent receives the System Design Document and determines component
shapes from the component definition — the user never selects profiles. The verification
pipeline rationale content was LLM-context-expensive without being execution-guiding.

---

## 1.7.0 — 2026-04-27

### Added
Three authoring rules and verification checks derived from carapex spec audit post-mortem.
All four gaps found had a common thread: the verification pipeline is batch, not inline —
checks run once at review time but the errors were introduced during authoring.

- **Writing Rule: type relationship MUST** — "X is a Y subclass" is descriptive prose, not
  a contract. Binding type relationships MUST use RFC 2119 MUST. Addresses the failure mode
  where a glossary definition reads as documentation rather than a normative requirement,
  allowing it to slip past the RFC 2119 sweep (Q5 was not specific enough to flag type
  relationships written in prose form).
- **Writing Rule: audit field information disclosure** — fields that name which pattern
  matched, which check triggered, or which threshold was breached allow an attacker to
  enumerate the detection surface. Evaluated at field-authoring time, not at security review.
- **Q27 (Before You Write / Tool A)** — for every type relationship in §2 or §5: is it
  expressed with RFC 2119 MUST? Catches the GAP-3 class of error in verification sweep.
- **Q28 (Before You Write / Tool A)** — for every configuration field table in §15: does
  every default value agree with the prose description? Catches the GAP-4 class of error
  (table and prose as two representations of the same truth that can contradict each other).
- **When Spec Is Done checklist** — three new items corresponding to the three rules above.

### Rationale
The batch verification problem: by the time Tool A runs, the author has committed mentally
to the spec's structure. Contradictions between table and prose, prose-as-requirement without
MUST, and requirements with security side-effects are all easier to catch at the point of
authoring than in a single end-of-document pass. Adding them to Writing Rules and the
Before You Write checklist moves the catches earlier in the process.

---

## 1.6.0 — 2026-04-27

### Added
- **Post-Implementation Coverage Audit** (Workflow step 6): after the coding LLM delivers
  an implementation, every RFC 2119 MUST contract in the spec is confirmed present or
  recorded as a deviation. Addresses a structural gap: the previous methodology had no
  gate after implementation handoff, allowing implementation drift to go undetected.
- **Deviation classification**: `IMPLEMENTATION_DRIFT` (spec correct; fix implementation)
  vs `SPEC_ERROR_REVEALED` (implementation exposes spec error; amend spec, re-verify,
  re-handoff). Derived from carapex post-mortem — gaps bifurcate by which artefact is wrong.
- **Deviation report format** (`verification-pipeline.md`): `DEVIATION-[N]` entries with
  entity, type, finding, spec evidence, impl evidence, resolution. Parallel to gap report
  format.
- **Implementation Coverage** — 4th status dimension: `COMPLETE` (zero unresolved
  deviations) | `INCOMPLETE — N deviations (M unresolved)`. Independent of Implementation
  Readiness and Verification Currency.
- Non-negotiable: Implementation Coverage MUST reach COMPLETE before declaring project
  done; deviations MUST NOT be documented as code comments.
- Design rationale entries for why the audit is separate from pre-handoff verification,
  why deviations route to a fix surface (not comments), why `SPEC_ERROR_REVEALED`
  requires spec amendment, and why code quality is not a step in the coverage audit.

### Changed
- SKILL.md: v1.5.0 → v1.6.0
- Status block examples updated to show all four dimensions.

---

## 1.5.0 — 2026-04-27

### Added
- **§2b Architectural Constraints**: new section between §2 and §3 for structural decisions
  that cause wrong implementation if violated — class hierarchy, inheritance strategy,
  component priority ordering, named patterns. Addresses a gap discovered during MASP
  challenge prep: §5 assumes structure exists, §11 covers pipeline sequencing only,
  §16 covers extension contracts only. None captured structural roles.
- §2b entry in Fixed Cross-Reference Matrix (`verification-pipeline.md`)
- BLOCKING gap example for architectural constraint gaps ("implementor would choose wrong
  inheritance strategy without this") in gap classification section
- Invalidation table row: "New or changed architectural constraint → A (Q26), C Steps 1, 2"
- Q26 added to Before You Write checklist: structural decisions check
- Corresponding item added to When the Spec Is Done checklist
- §2b entry in How Sections Relate (SKILL.md)
- §2b handoff rule in Non-Negotiables (SKILL.md)
- `version` field added to SKILL.md frontmatter (was only in heading)
- Version removed from heading (consistent with other skills)
- `README.md` — human orientation, structure, design rationale
- `CHANGELOG.md` — this file

### Changed
- SKILL.md: v1.4 → v1.5.0

---

## 1.4.0 — prior

- Named exception types added to Fixed Cross-Reference Matrix: four named exception types
  survived two full verification passes without §2 entries because no matrix rule required it.
  Row added to prevent recurrence.

---

## 1.0.0 — initial

- 23-section specification methodology
- Three-tool integrated verification pass (completeness sweep, stress tests, structural verification)
- Blocking/non-blocking gap classification with single falsifiable test
- Spec status model: Implementation Readiness, Gap List, Verification Currency
- Re-grounding mechanism with externalised artefacts
- Spec profiles (Appendix B), stress tests (Appendix A), verification pipeline (Appendix C)
