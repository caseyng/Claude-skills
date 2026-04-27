# Changelog

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
