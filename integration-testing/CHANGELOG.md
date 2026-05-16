# Changelog — integration-testing

## 1.0.0 — 2026-05-16

Initial release.

### What's in it

- Five-phase execution: contract inventory, categorization and stub identification, test design,
  test implementation, execution and report
- Stub-at-boundary principle: internal components are real, external dependencies stubbed at
  the boundary component that owns the external connection
- Two test levels per contract: happy path AND failure path (`on_failure` from the contract
  is a required test target, not optional)
- Two integration levels: pairwise contract tests + end-to-end flow tests
- MANUAL_REQUIRED classification: contracts involving hardware sensors, OS-level behaviors
  (Doze mode), or real external services produce written test procedures instead of runnable code
- Report schema: every contract mapped to PASS / FAIL / MANUAL_REQUIRED, with violation
  detail and manual procedures as applicable
- Test isolation requirement: each test starts from clean state; `@Before`/`@After` teardown is mandatory

### Enrichment coverage

- Android: instrumented tests, ServiceTestRule, in-memory Room database, AppEventBus /
  SharedFlow collection in runTest, StubWhatsAppClient (`simulateIncomingMessage`,
  `simulateActivity`), Compose UI integration tests, MANUAL_REQUIRED table with ADB commands
- Backend: in-process server + real test database, test containers, WireMock/MockServer
  for external APIs, HTTP integration test pattern, message queue integration, MANUAL_REQUIRED table

### Design decisions

- **No mocking of internal components.** If an internal component cannot be instantiated
  in the test environment, the test environment is wrong. Mocking an internal component for
  integration tests defeats the purpose — you are testing the mock's behavior, not the
  component's.
- **Both happy and failure paths required.** The `on_failure` field in every interaction
  contract is a first-class test target. An integration test that only covers the happy
  path has not verified the contract.
- **MANUAL_REQUIRED is a first-class result, not a failure.** Some contracts genuinely
  require conditions that automated testing cannot reproduce. Acknowledging this explicitly
  (with written procedures) is more honest and more useful than either skipping those
  contracts or attempting to automate them unreliably.
- **End-to-end flows are required, not optional.** Pairwise contract tests catch interface
  mismatches. They do not catch missing event routing, state propagation bugs, or timing
  issues across the full component chain. Both levels are necessary.

### Origin

Stage 6 of the software-development-orchestrator pipeline. The approach was initially marked
as "not yet decided." The resolution: the interaction contracts defined in Stage 2 (System
Design) are the test targets — they explicitly define what is passed, in what format, what
happens on failure. The integration testing strategy follows directly from the contract
definitions. The stub-at-boundary principle distinguishes this stage from unit testing
(which mocks everything) and E2E testing (which requires a deployed environment).
