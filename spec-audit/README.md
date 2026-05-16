# spec-audit

Audits whether an implementation satisfies every RFC 2119 MUST contract in its component spec.

## What this skill does

spec-audit answers one question per contract: does the implementation honor what the spec
requires? It systematically walks every MUST, MUST NOT, SHALL, SHALL NOT in the component
specification, locates the relevant code, and classifies each contract as satisfied, violated,
or unverifiable.

## What it catches that tests do not

Tests verify behavior at runtime. spec-audit verifies structural compliance statically.
A system can pass all tests and fail spec-audit:

- **Wrong failure name** — the implementation raises a generic exception where the spec requires a named failure mode. Tests catch the exception; callers handle it generically. spec-audit sees the naming violation.
- **Missing required field** — the implementation omits a field from an output struct. Tests may not assert on every field. spec-audit checks every field the spec marks as required.
- **Absent security enforcement** — a validation check exists on the happy path but is absent from an error path. Tests cover the happy path. spec-audit reads all code paths.
- **Wrong boundary** — spec says "greater than MAX" but code checks "greater than or equal to MAX." Off-by-one in a boundary condition. Tests may not hit the exact boundary value. spec-audit reads the comparison operator.
- **Swallowed error** — error is caught, logged, and nil is returned. Caller sees success. Tests that only check the return value see success. spec-audit sees the swallow.

## The ordering constraint

Contracts are extracted from the spec completely before any implementation code is read.
This prevents rationalization — when auditor reads spec and implementation simultaneously,
they unconsciously read the spec through the lens of what the code already does, finding
compliance that is not there.

## Output

An Audit Report with three lists:
- **Violated** — contract ID, spec section, exact spec text, implementation finding, severity (CRITICAL / HIGH / MEDIUM)
- **Unverifiable** — contracts that require runtime execution to verify, flagged for spec-driven-testing
- **Satisfied** — contract IDs only (no detail needed)

**PASS**: zero CRITICAL, zero HIGH violations.
**FAIL**: any CRITICAL or HIGH violation.

## Relationship to other Stage 5 skills

| Skill | Catches |
|---|---|
| `code-integrity-guardrail` | Code quality, security anti-patterns, phantom imports, type errors |
| `spec-driven-testing` | Behavioral failures at runtime (tests generated from spec shapes) |
| `spec-audit` | Structural non-compliance — what the spec says MUST exist and does not |

All three run in parallel per component. Each catches what the others miss.

## Contract source

Specs produced by `spec-contract-methodology`. The highest-contract sections are:
§5 (API), §6 (Lifecycle), §7 (Failure Taxonomy), §8 (Boundary Conditions), §12 (Interaction Contracts),
§17 (Error Propagation), §18 (Observability), §19 (Security Properties).
