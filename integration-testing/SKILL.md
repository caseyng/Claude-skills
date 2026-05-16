---
name: integration-testing
version: 1.0.0
description: >
  Tests the assembled system against the interaction contracts defined in the System Design
  Document. Every interaction contract is a test target. Internal components are real;
  external dependencies are stubbed at the system boundary. Produces test code for
  automatable contracts and written procedures for contracts requiring real hardware or
  external services. Report maps every contract to PASS, FAIL, or MANUAL_REQUIRED.
---

# Integration Testing

## Role

Integration tester. You verify that the assembled system's components interact as the
System Design Document specifies.

Stage 5 tests components in isolation. Stage 6 tests them assembled — real components
talking to each other, with stubs only at the system boundary (external dependencies
the product does not own).

Two levels:
- **Contract-level**: every pairwise interaction contract from Stage 2 is tested
- **End-to-end flows**: multi-contract chains that cross several components in sequence

Both levels are required. Contract-level tests catch interface mismatches. End-to-end
tests catch missing connections, wrong event routing, and cross-component state bugs.

---

## Stub-at-Boundary Principle

**Internal components must be real. External dependencies are stubbed at the boundary.**

Internal = a component in the Stage 2 component list. These are what the product owns.
In integration tests, every internal component runs in its real form — real code,
real database, real in-memory state.

External = something the product depends on but does not own: a third-party API, a
hardware sensor, a network service, a platform OS service that requires real hardware.
These are stubbed. The stub is at the boundary — the component that owns the external
connection (as defined in Stage 2's `cross_cutting_concerns`) is the stub point.

The test environment is: all internal components real, all external dependencies stubbed.
If a test requires a real external dependency, it is MANUAL_REQUIRED (see below).

---

## Automated vs Manual

Some interaction contracts require conditions that cannot be reproduced in an emulator
or automated test runner:
- Hardware sensors (activity recognition, GPS, biometric)
- OS-level behaviors that emulators do not faithfully reproduce (Doze mode, exact alarm
  delivery under Doze, foreground service survival under memory pressure)
- Real network services with rate limits or account state (WhatsApp sessions)
- Time-sensitive behaviors that require wall-clock elapsed time

Contracts involving these conditions are: **MANUAL_REQUIRED**.

For MANUAL_REQUIRED contracts, produce a written test procedure (step-by-step instructions,
expected behavior, how to verify). Do not produce runnable code — the procedure is the artifact.

For all other contracts, produce runnable test code.

---

## Execution

### Phase 1 — Contract Inventory

Read the System Design Document (`output/stage-2-system-design.md`).

Extract every interaction contract. For each:
- Contract ID (use the system design's naming or assign INT-01, INT-02, ...)
- From component → To component
- What is passed, in what format
- Who owns the data
- On_failure behavior

Also extract end-to-end flows: sequences of two or more contracts that together represent
a user-meaningful action (e.g., incoming message → rule evaluation → auto-reply → log entry).
If the system design does not name flows, derive them from the component interaction graph.

At the end of Phase 1: a numbered list of contracts + a list of end-to-end flows.

---

### Phase 2 — Categorize and Identify Stubs

For each contract and flow:
1. Identify whether it is automatable or MANUAL_REQUIRED (see criteria above)
2. Identify which external dependencies the test path touches — these get stubs
3. Identify which internal components are involved — these must be real

Document the test environment configuration:
- Which internal components are initialized, in what form (real implementation, in-memory
  database where applicable)
- Which external boundaries are stubbed and with what stub behavior

---

### Phase 3 — Test Design

For each automated contract:
- Happy path: the normal case where the contract is satisfied
- Failure path: the `on_failure` behavior from the contract definition (what happens when
  the interaction fails — missing, malformed, or the receiving component rejects it)

For each automated end-to-end flow:
- Full chain: trigger the flow at the entry point, verify state at every component along
  the chain, not just at the final output

For each MANUAL_REQUIRED contract:
- Write a step-by-step test procedure
- Include: preconditions, steps, expected behavior at each step, how to verify pass/fail

**Both happy path and failure path are required for every contract.**
A contract tested only on the happy path is not integration tested — the `on_failure`
behavior is part of the contract.

---

### Phase 4 — Test Implementation

Write test code for all automated contracts and flows.

Use the platform-appropriate test tooling (see `references/enrichment-android.md` and
`references/enrichment-backend.md`).

Test structure per contract:
```
Test: [contract ID] — [description]
Setup:  [initialize real components, configure stubs]
Action: [trigger the interaction]
Assert (happy path): [verify data reaches the receiving component in the specified format]
Assert (failure path): [trigger the on_failure condition, verify the specified failure behavior]
Teardown: [clean state for next test]
```

Each test is independent. Tests do not share mutable state. Teardown must restore the
environment to a clean state so test execution order does not affect results.

---

### Phase 5 — Execution and Report

Run automated tests. Record results.

Produce the Integration Test Report.

**Report schema:**

```
Integration Test Report — [project_name]
System Design Document: output/stage-2-system-design.md

Summary:
  Total contracts:   [N]
  Automated:         [N] (Pass: N, Fail: N)
  Manual required:   [N]

Contract results:
```

| Contract ID | From → To | Automated | Result | Notes |
|---|---|---|---|---|
| INT-01 | Service → Rules Engine | Yes | PASS | — |
| INT-05 | Service → WA Client | No | MANUAL_REQUIRED | Requires real device + Doze |

**For FAIL results:**

```
[Contract ID] — FAIL
Contract: [from] → [to] — [what is passed]
Expected: [the behavior the contract specifies]
Actual:   [what the test observed]
Involved: [which components are in the failure path]
```

**For MANUAL_REQUIRED contracts:**

```
[Contract ID] — MANUAL_REQUIRED
Contract: [from] → [to]
Why manual: [specific reason — hardware sensor / Doze / real connection]
Procedure:
  Preconditions: [...]
  Steps:
    1. [...]
    2. [...]
  Expected: [what to observe at each step]
  Pass if: [explicit pass condition]
  Fail if: [explicit fail condition]
```

---

## Completeness Check

Before producing the report, verify:
1. Every interaction contract from the System Design Document appears in the results table
2. Every contract has both a happy path and a failure path test (or procedure)
3. Every end-to-end flow appears at least once
4. No test touches an external dependency without a stub
5. No internal component is replaced with a mock (stubs only at the external boundary)

---

## What Integration Testing Does Not Do

- Does not test individual components in isolation — that is Stage 5
- Does not test code quality, security posture, or spec compliance — those are Stage 5 sub-tasks
- Does not add new contracts to test that are not in the System Design Document — the SD is authoritative
- Does not mock internal components — if an internal component cannot be instantiated in a test
  environment, the test environment is wrong, not the test strategy

---

## Enrichment Checklists

Platform-specific tooling and approach:
- `references/enrichment-android.md` — Android instrumented tests, service lifecycle, AppEventBus
- `references/enrichment-backend.md` — HTTP integration tests, database setup, external API stubs
