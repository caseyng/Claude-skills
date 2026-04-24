---
version: 1.1.0
date: 2026-04-24
skill: code-integrity-guardrail
---

# Verification Protocol

## Pre-Flight Checklist Framework

This is the framework. Language skills provide the concrete checklist in their
`references/bindings/<language>.md`. This file defines phases, procedures, and
special tests. Language bindings define what to scan for in each phase.

---

### Phase 1: Security Scan

**Objective:** Ensure no Category 1 (Security) slop exists.

**Process:**

1. Scan all generated files for Category 1 patterns
2. For each potential violation, classify as:
   - **Confirmed violation** — STOP, flag the specific line/pattern, propose secure alternative
   - **False positive** — document explicitly why it is safe
   - **Uncertain** — flag for human review; never self-certify
3. Do not proceed to Phase 2 until all confirmed violations are resolved

**Universal triggers (language bindings map to concrete syntax):**
- Hardcoded credentials
- Injection vulnerabilities: SQL, command, LDAP, XPath, NoSQL
- XSS — unsanitized user input rendered as HTML (86% AI failure rate)
- Log injection — unsanitized user input in log statements (88% AI failure rate)
- Unsafe dynamic execution
- Insecure subprocess execution
- Disabled TLS/certificate verification
- Weak cryptographic primitives
- Over-permissive CORS
- Path traversal via user input
- Information disclosure in logs
- Missing authentication or authorization

**Priority flag:** XSS and log injection have empirically disproportionate failure
rates in AI-generated code. Treat these as default-present until proven absent.

---

### Phase 2: Logic Verification

**Objective:** Ensure no Category 2 (Logic) slop exists.

**Process:**

1. Identify all algorithms and data structures
2. For each, verify:
   - Boundary conditions: empty, single element, max size, negative index
   - Null/undefined/missing handling before every dereference
   - Type correctness and comparison semantics
   - Numeric precision where applicable
3. Check all shared mutable state for explicit synchronization
4. Verify all acquired resources have explicit cleanup

---

### Phase 3: Architecture Review

**Objective:** Ensure no Category 3 (Architecture) slop exists.

**Process:**

1. Trace all dependency paths from the composition root
2. Verify no business logic instantiates its own dependencies
3. Confirm registry/provider separation — registries resolve, providers construct
4. Verify config propagation depth — config must not travel past composition root
5. Confirm no string-based strategy selection (`if/elif` on type strings)
6. Verify all external service calls have timeout, retry, and backoff

---

### Phase 4: Quality Audit

**Objective:** Ensure no Category 4 (Quality) slop exists.

**Process:**

1. Run mirror test on all generated tests (see below)
2. Verify error path coverage alongside success paths
3. Check encoding explicitness on all file and text operations
4. Verify logging vs print usage — no `print()` for diagnostics
5. Confirm all injected classes have informative `__repr__`
6. Check all resource management for explicit cleanup

---

### Phase 5: Self-Correction Trigger Review

**Objective:** Catch slop indicated by language patterns in the generated output.

**Process:** Scan all output for these phrases. Each requires explicit
justification or a rewrite:

| Trigger Phrase | Required Action |
|----------------|----------------|
| "for now" | Add specific removal date or condition |
| "temporary" | Add specific removal date or condition |
| "TODO: fix later" | Add specific removal trigger condition |
| "just to get it working" | Invoke pressure response protocol |
| "simplified for brevity" | Verify no omitted security or validation |
| "assumes valid input" | Add explicit input validation |
| "works in most cases" | Add edge case handling |
| "should be fine" | Replace with explicit, checkable verification |
| "I verified" / "I checked" | State exactly what was checked and how |

The last entry is the most critical. Self-reported verification by the same
system that may have skipped it is not verification.

---

## The Mirror Test (Anti-Test-Subversion)

**Purpose:** Prevent AI from changing tests to pass broken code rather than
fixing the code. Research confirms AI will subvert tests rather than fix
underlying logic when under implicit pressure to "make tests pass."

**Procedure:**

1. Generate tests independently from implementation where possible
2. For each test, ask: "If the implementation is completely wrong but keeps the
   same interface, would this test catch it?"
3. If the test calls the same helper functions as the implementation — mirror test, rewrite
4. If changing the internal algorithm while preserving the interface does not
   break the test — mirror test, rewrite

**Red flags:**
- Test imports private helper functions from the implementation module
- Test constructs the same internal objects as the implementation
- Test duplicates implementation logic to compute expected values

---

## The Hallucination Test (Anti-Phantom-Dependency / Anti-Slopsquatting)

**Purpose:** Prevent phantom package references and exposure to slopsquatting
attacks, where an attacker publishes a malicious package using a name the AI
repeatedly hallucinates.

**Procedure:**

1. Before referencing any external dependency, verify it exists in the language's
   official package registry (PyPI, npm, crates.io, etc.)
2. Flag any package name that is plausible but unverifiable — this is the
   slopsquatting attack surface
3. Check for typosquatting: common misspellings of popular packages
4. Treat repetition of the same unfamiliar package name as high-risk —
   repeated hallucination is the mechanism by which slopsquatting propagates
5. Prefer packages with established presence: active repository, download history,
   known maintainer

**Red flags:**
- Package name is plausible but cannot be placed in memory or verified
- Package name is a slight misspelling of a well-known package
- Package has no repository, documentation, or meaningful download history
- The same unfamiliar package name appeared in a previous generation in this session

---

## The Sycophancy Check

**Purpose:** Prevent agreement with incorrect premises when the user frames a
question as a confirmation request.

**Procedure:**

When a user asks "isn't X correct?" or "this looks right, doesn't it?" about
code or design:

1. Evaluate X independently of the framing
2. If X is incorrect or suboptimal — say so directly, regardless of the framing
3. Do not soften the assessment to preserve agreement
4. Propose the correct alternative

The check passes when the response reflects the actual evaluation, not the
preferred answer.

---

## Post-Generation Verification Statement

Before delivering any code, state explicitly:

> "Verification complete. Security scan: [pass / violations found — named and
> resolved]. Logic verification: [pass / issues found — named and resolved].
> Architecture review: [pass / issues found — named and resolved]. Quality audit:
> [pass / issues found — named and resolved]. Self-correction triggers: [none
> found / phrases found — justified or rewritten]. Behavioral checks: [no
> phantom dependencies / no sycophantic agreement with incorrect premises]."

**Unacceptable statements:**
- "Looks good"
- "I verified"
- "Code is clean"

These are self-reports without evidence. If a phase found issues, name them and
state how they were resolved. If no statement is made, the verification protocol
was not run and the code should not be delivered.
