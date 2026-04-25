---
version: 1.2.2
skill: code-integrity-guardrail
---

# Verification Protocol: Extended Reference

The phase structure and confidence labels are defined in SKILL.md. This file
provides extended procedure detail for phases that benefit from it.

---

## The Mirror Test (Anti-Test-Subversion)

AI will modify tests to pass broken code rather than fix the code when under
implicit pressure to "make tests pass."

**Procedure:**

1. Generate tests independently from implementation
2. For each test: "If the implementation is completely wrong but keeps the same
   interface, would this test catch it?"
3. If the test calls any function from the same module that is not part of the
   public interface — mirror test, rewrite
4. If changing the internal algorithm while preserving the interface does not
   break the test — mirror test, rewrite

**Red flag:** Test duplicates implementation logic to compute expected values.

---

## The Hallucination Test (Anti-Phantom-Dependency)

Slopsquatting mechanism: the model generates the same phantom package name
repeatedly across sessions. An attacker publishes a malicious package under that
name. Developers who follow the recommendation download the attacker's package.

**Procedure:**

1. **With bash:** verify the package exists in the language's official registry
   before referencing it. **Without bash:** flag all unverifiable packages as
   UNCERTAIN immediately. Do not assert existence based on familiarity alone.
2. Flag any package name that is plausible but unverifiable
3. Check for typosquatting: common misspellings of popular packages
4. Treat repeated unfamiliar package names as high-risk — repetition is the
   propagation mechanism
5. Prefer packages with active repository, download history, known maintainer

**Red flags:**
- Package name is plausible but cannot be placed or verified
- Package name is a slight misspelling of a well-known package
- Package has no repository, documentation, or download history
- Same unfamiliar name appeared in a previous generation this session

---

## Post-Generation Statement: What Counts as Evidence

The statement must be specific and checkable. The model that skipped verification
is the same model reporting that it ran it — self-reports are structurally
untrustworthy.

**Acceptable:**
- "Ran language security scanner — no HIGH findings. Two MEDIUM findings reviewed
  and determined false positives: [reason]."
- "No bash access. Reasoned through Category 1 patterns. SQL queries: all
  parameterized. No subprocess calls. No eval. UNCERTAIN on dependency versions
  — verify against registry."

**Unacceptable:**
- "Looks good"
- "I verified"
- "Code is clean"
- "I checked for security issues"
