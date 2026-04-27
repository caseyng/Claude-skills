# Changelog

## 1.2.3 — 2026-04-27

### SKILL.md
- Removed specific statistics from Core Assumption (Veracode 2025 numbers, XSS/log injection
  percentages): unverifiable figures with specific citations undermine credibility if wrong.
  Core claim retained in defensible form: AI vulnerability rates are measurably higher;
  injection and XSS have disproportionately high failure rates; slopsquatting is named.

### README.md
- Removed "Universal" from tagline — one language binding doesn't earn the claim.
  Replaced with "Language-agnostic core; language-specific behaviour via bindings."

---

## 1.2.2 — 2026-04-25

### references/verification-protocol.md
- Mirror test: removed redundant red flags (covered by step 3); retained only "test duplicates implementation logic" which is not implied by step 3
- Hallucination test: removed separate "Tool limitation" block; moved bash/no-bash branch directly into step 1 so the caveat is inline with the action

### references/bindings/python.md
- Phase 0 minimum bar: changed "mypy with no errors on public interfaces" to "mypy with no errors on the files being delivered" — matches `mypy src/` command (no --strict)

### All reference files
- Version bumped to 1.2.2 for consistency with SKILL.md

---

## 1.2.1 — 2026-04-25

### SKILL.md
- Phase 1 confidence table: added explicit internal threshold percentages (100% → CONFIRMED, 80% → FLAG, 50% → UNCERTAIN); clarified that FLAG and UNCERTAIN do not block delivery

### references/verification-protocol.md
- Mirror test: added executable step — "If the test calls any function from the same module that is not part of the public interface, flag as mirror test"
- Hallucination test: added tool limitation notice — without bash or registry API access, verification is impossible; flag all unverifiable packages as UNCERTAIN

---

## 1.2.0 — 2026-04-25

### SKILL.md
- Scope calibration decision tree moved to front (Step 0), before taxonomy
- Added `# SCRIPT-NOT-A-LIBRARY` marker as formal mechanism for `print()` exception
- Fixed CORS rule: violation requires wildcard origin AND `allow_credentials=True` together; wildcard alone is lower-risk, not an automatic violation
- Added Legitimate Exceptions table: SQL constant f-string, `shell=True` with literal, wildcard CORS without credentials, print in scripts
- Added invalid exception justifications: "temporary", "internal tool", "sanitized", "we trust the input" do not qualify
- Verification Protocol: added Phase 0 (tool execution) with bash-available / bash-unavailable branches; renumbered Phases 1–5
- Confidence scoring: hybrid model — CONFIRMED / FLAG / UNCERTAIN labels in output; internal thresholds drive classification
- Sycophancy: replaced aspirational note with explicit trigger conditions (pushback, correction framing); 4-step check is not run on routine generation
- Removed alignment faking: unfalsifiable claim about model motivation. Unconditional protocol execution retained; justified by consistency as the only checkable property, not by speculation about model behavior
- Orthogonality declaration added: bindings add only, never restate
- Post-generation statement updated: requires tool output when bash available; explicit UNCERTAIN declaration when not
- Removed version history table (moved to CHANGELOG.md)
- Removed "How to use" invocation instructions (moved to README.md)
- Removed `date:` frontmatter

### references/bindings/python.md
- Phase 0 tooling section added: bandit, mypy, pip-audit, ruff with commands and minimum bar
- Fixed print() rule: violation only outside `# SCRIPT-NOT-A-LIBRARY` files
- Fixed CORS rule: violation requires wildcard + `allow_credentials=True`
- Added exception criteria per high-signal pattern (SQL f-string, shell=True)
- Stripped all restatements of universal rules — kept only Python-specific syntax, code examples, and tooling
- Pre-flight checklist: Phase 0 added; items trimmed to Python-specific only
- Architecture, logic, quality sections retained where Python syntax is required to detect the pattern

### references/verification-protocol.md
- Stripped phases (now in SKILL.md); retained Mirror Test and Hallucination Test extended procedures
- Added Post-Generation Statement examples: acceptable vs unacceptable

### references/pressure-response.md
- Stripped 4-step protocol (now in SKILL.md); retained examples, escalation path, non-negotiables list

### references/concepts.md
- Removed rule restatements; retained only mechanism explanations (why each pattern happens)
- Removed alignment faking entry; updated behavioral failure modes section

### New files
- `README.md` — human orientation, structure, design principles, binding guide
- `CHANGELOG.md` — this file; version history extracted from SKILL.md

---

## 1.1.0 — 2026-04-24

- Added slopsquatting to hallucination test
- Added behavioral failure modes section (instruction attenuation, task drift, alignment faking)
- Elevated XSS and log injection as high-priority patterns with empirical failure rates
- Added sycophancy defense to pressure response
- Renamed from `guardrails` to `code-integrity-guardrail`

## 1.0.0 — 2026-04-21

- Initial release: universal taxonomy, verification protocol, pressure response, language binding contract
