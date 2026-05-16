# integration-testing

Tests the assembled system against the interaction contracts defined in the System Design Document.

## What this skill does

Stage 5 tests components in isolation — each component against its own spec. Stage 6 tests the
assembled system: real components communicating with each other, with stubs only at the external
boundary (dependencies the product does not own).

Every interaction contract from the System Design Document is a test target. The contracts
already define exactly what is passed, in what format, who owns it, and what happens on failure.
Integration tests verify that the actual assembled components honor these contracts.

## Two levels

**Contract-level tests** — one per interaction contract. Tests the pairwise interface: does
Component A's output arrive at Component B in the format Component B expects? Does the failure
path behave as the `on_failure` field in the contract specifies?

**End-to-end flow tests** — multi-contract chains that represent a user-meaningful action
crossing several components in sequence. These catch errors that pairwise tests miss:
missing event routing, state that is set in one component but not propagated to the next,
timing bugs between asynchronous components.

## Stub-at-boundary principle

Internal components (everything in the Stage 2 component list) are real in integration tests.
External dependencies (third-party APIs, hardware sensors, platform OS services requiring real
hardware) are stubbed at the point where the product's boundary component owns the connection.

This is what makes integration tests different from unit tests (which use mocks throughout)
and E2E tests (which require a fully deployed environment). The boundary is clear: real
inside, stubbed outside.

## Automated vs manual

Some contracts require conditions that cannot be reproduced in an automated test runner —
Doze mode behavior, hardware sensors, real network services with account state. These are
classified as MANUAL_REQUIRED and produce written test procedures (step-by-step instructions,
expected behavior, explicit pass/fail criteria) rather than runnable code.

## Output

An Integration Test Report mapping every contract to: PASS, FAIL (with violation detail),
or MANUAL_REQUIRED (with test procedure). Plus the test code for all automated contracts.

## Enrichment checklists

- `references/enrichment-android.md` — Android instrumented tests, ServiceTestRule, AppEventBus collection, StubWhatsAppClient usage, MANUAL_REQUIRED conditions and ADB commands
- `references/enrichment-backend.md` — HTTP integration tests, test database setup, WireMock stubs, message queue testing
