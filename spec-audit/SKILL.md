---
name: spec-audit
version: 1.0.0
description: >
  Audits whether an implementation satisfies every RFC 2119 MUST contract in its component spec.
  Extracts contracts from the spec before reading implementation to prevent rationalization.
  Produces a violation report: SATISFIED, VIOLATED (with severity), or UNVERIFIABLE.
  Distinct from spec-driven-testing: spec-audit catches structural non-compliance that passing
  tests do not — wrong failure names, missing required fields, absent security enforcement,
  boundary conditions not checked.
---

# Spec Audit

## Role

Compliance auditor. You verify that the implementation satisfies every MUST contract
in the component specification. You do not generate tests (that is spec-driven-testing).
You do not evaluate code quality (that is code-integrity-guardrail). You answer one
question per contract: does the implementation honor what the spec requires?

---

## Ordering Constraint

**Read the spec completely and extract all contracts before reading any implementation code.**

This is not a suggestion. Rationalization is the primary failure mode of compliance auditing —
when the auditor reads implementation and spec simultaneously, they unconsciously read the spec
through the lens of what the code already does, and find compliance that is not there.

The ordering constraint prevents this:
1. Extract contracts from spec → fixed list of numbered claims
2. Read implementation → evaluate each claim against code evidence
3. Report → what is satisfied, violated, or unverifiable

---

## Execution

### Phase 1 — Contract Extraction (spec only, no implementation)

Read the component spec completely. Extract every sentence containing: MUST, MUST NOT,
SHALL, SHALL NOT.

For each extracted contract:
- Assign a sequential ID: MUST-01, MUST-02, ...
- Record the spec section (§N)
- Record the exact spec text
- Classify the contract type (see `references/contract-taxonomy.md`)
- Note the verifiability: STRUCTURAL (readable from code without execution), BEHAVIORAL
  (requires reading execution flow), EXTERNAL (requires runtime — flag for spec-driven-testing)

Do not read any implementation code during this phase.

At the end of Phase 1 you have: a numbered list of contracts with types and verifiability.
This list is fixed — it does not change after Phase 1.

---

### Phase 2 — Implementation Analysis

Read the implementation code. For each contract in your Phase 1 list:

1. Locate the relevant code: the function, method, class, or code path where this contract
   would be honored or violated
2. Collect evidence: the specific line(s) of code that demonstrate compliance or violation
3. Determine the result:
   - **SATISFIED** — evidence clearly shows the contract is honored
   - **VIOLATED** — evidence shows the contract is not honored, or the relevant code is absent
   - **CANNOT_DETERMINE** — the contract is about runtime behavior that cannot be verified
     by reading code alone; flag as UNVERIFIABLE

For VIOLATED findings, classify severity:
- **CRITICAL**: violation breaks a guarantee a caller or the system relies on — wrong failure
  name returned, required field absent from output, security contract unenforced, interface
  method missing
- **HIGH**: violation is present but edge-case — boundary not checked on a specific condition,
  validation missing in one of several code paths
- **MEDIUM**: partial compliance — the spirit is honored but the letter is not (e.g., event
  logged but missing a required field the spec mandates)

---

### Phase 3 — Report Production

Produce the Audit Report.

**Summary block:**
```
Component:          [name]
Spec version:       [version from spec frontmatter]
Contracts audited:  [total]
Satisfied:          [count]
Violated:           [count] (Critical: N, High: N, Medium: N)
Unverifiable:       [count]
Audit result:       PASS | FAIL
```

PASS requires: zero CRITICAL violations, zero HIGH violations.
FAIL if: any CRITICAL or HIGH violation exists.

MEDIUM violations and UNVERIFIABLE items do not affect the PASS/FAIL result but are reported.

**Violation entries (one per violated contract):**

```
[MUST-NN] — [severity] — §[section]
Spec:    [exact spec text]
Finding: [what the implementation does instead, or what is absent]
Code:    [file path:line number or function name where the evidence is]
```

**Unverifiable entries (one per unverifiable contract):**

```
[MUST-NN] — UNVERIFIABLE — §[section]
Spec:    [exact spec text]
Reason:  [why this cannot be verified statically]
Action:  Verify via spec-driven-testing or manual testing
```

**Satisfied entries:**
List contract IDs only — `MUST-01, MUST-03, MUST-07, ...` — no detail needed.

---

## What Spec Audit Does Not Do

- Does not generate tests (spec-driven-testing does this)
- Does not evaluate code style, quality, or security posture (code-integrity-guardrail)
- Does not verify that tests pass (that is the test runner)
- Does not amend or interpret the spec — if the spec is ambiguous, flag it as UNVERIFIABLE
  with the note "spec text is ambiguous — cannot determine compliance"
- Does not audit contracts from §21 (What Is Not Specified) — these are non-goals, not
  MUST contracts
- Does not audit §22 (Assumptions) as implementation contracts — these are caller
  responsibilities, not this component's obligations. Exception: if the spec says
  "this component MUST validate assumption X before proceeding," that IS auditable.

---

## Contract Sources by Spec Section

See `references/contract-taxonomy.md` for which sections produce which types of contracts
and how to verify each type.

High-contract sections (audit these first):
- §5 Operations and API — interface contracts: parameters accepted, outputs returned
- §6 Lifecycle — state transition contracts
- §7 Failure Taxonomy — failure naming and condition contracts (highest value to audit)
- §8 Boundary Conditions — what happens at exact boundary values
- §12 Interaction Contracts — what must be passed between components
- §17 Error Propagation — how failures must travel
- §18 Observability Contract — what must be logged and when
- §19 Security Properties — what must be enforced

Lower-contract sections (audit but expect fewer findings):
- §9 Sentinel Values — encoding contracts
- §10 Atomicity — state consistency contracts
- §11 Ordering — sequencing contracts
- §13 Concurrency — thread safety contracts
- §16 Extension Contracts — interface adherence contracts
- §20 Versioning — compatibility contracts
- §23 Performance Contracts — only if quantitative
