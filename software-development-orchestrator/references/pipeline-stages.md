# Pipeline Stage Definitions

Each stage defines: inputs, skill status, parallelism, output schema, human gate, and dependencies.

**Iteration context:** For any iteration after the first, the context package MUST include
the prior iteration's output artifact and the stage's decision log. The agent reads the
decision log before doing any work — OPTIONS REJECTED entries are constraints, not starting
points. See `decision-log-format.md` for the agent's reading protocol.

**Decision log:** Agent writes iteration entries; orchestrator appends gate outcomes.
See `decision-log-format.md` for format.

**Iteration cap:** 3 orchestrator-level re-spawns per stage/component. On cap breach,
escalate per the Escalation Protocol in SKILL.md.

**Skill status:** EXISTS = skill is available and can be invoked. MISSING = no skill exists;
pipeline pauses and emits a gap notice when this stage is reached.

---

## Stage 1 — Requirements Engineering

**Skill:** `requirements-engineering` — EXISTS
**Parallelism:** Sequential

**Inputs:**
- Raw human intent (unstructured natural language string)
- Iteration > 1: prior artifact (`output/stage-1-requirements.md`) + decision log

**Output file path:** `output/stage-1-requirements.md`
**Decision log:** `output/decisions-stage1.md`

**What to record in decision log:** Feature prioritisation choices, scope inclusions/exclusions
with rationale, surfaced concerns that the human did not raise, open questions and why they
require human resolution rather than a provisional answer.

**Output schema — Requirements Document:**

| Field | Description |
|---|---|
| `project_name` | Short name for this project, used in all subsequent artifacts |
| `goal` | Single sentence: what is being built and why |
| `features` | List of features, each with: name, description, priority (core / should-have / deferred) |
| `platform_constraints` | Constraints imposed by the target platform (OS version requirements, runtime limits, deployment environment, regulatory) |
| `surfaced_concerns` | Concerns the human did not raise but domain knowledge requires addressing: security posture, observability, audit trail, error strategy, data retention, compliance, failure modes. These MUST be surfaced regardless of whether the human mentioned them. |
| `phase_plan` | Features assigned to phases. Each deferred feature lists the extension point designed into Phase 1 to keep later phases additive. |
| `scope_inclusions` | What is explicitly in scope for this build |
| `scope_exclusions` | What is explicitly out of scope, with brief reason |
| `open_questions` | Unresolved items requiring a human decision before system design can proceed. MAY be empty. |
| `assumptions` | Implicit constraints stated explicitly and confirmed by the human |

**Validation criteria (orchestrator checks before human gate):**
- All schema fields present and non-empty (except `open_questions`, which may be empty)
- `features` contains at least one core-priority item
- Every deferred feature in `phase_plan` names an extension point
- `surfaced_concerns` is non-empty (every real project has at least one concern to surface)
- `open_questions` contains only items that cannot be resolved without a human decision — not questions the skill should have answered

**Human gate:**
Present a summary and reference `output/stage-1-requirements.md`. Human reads the file and either:
- **APPROVE** → pipeline advances to Stage 2
- **REJECT** with feedback → rework specified items, re-present. Do not advance without approval.

**Dependencies:** None. This is the pipeline entry point.

---

## Stage 2 — System Design

**Skill:** `system-design` — EXISTS
**Parallelism:** Sequential

**Inputs:**
- Approved Requirements Document from Stage 1 (`output/stage-1-requirements.md`)
- Iteration > 1: prior artifact (`output/stage-2-system-design.md`) + decision log

**Output file path:** `output/stage-2-system-design.md`
**Decision log:** `output/decisions-stage2.md`

**What to record in decision log:** Component decomposition rationale and alternatives
rejected (decomposition is the highest-leverage decision — rejected decompositions MUST
be recorded to prevent re-litigation), shared type definitions and why they belong in
`shared_types` vs internal to a component, interaction contract `on_failure` choices,
technical decisions with alternatives considered.

**Output schema — System Design Document:**

| Field | Description |
|---|---|
| `components` | List of components. Each component: `name`, `purpose` (one sentence), `boundary` (what it owns and what it does not own), `inputs` (what it receives), `outputs` (what it produces), `platform` (implementation language/environment — see Platform-Skill Map below) |
| `shared_types` | Data structures that cross component boundaries — defined once here, referenced by interaction contracts and component specs. Each type: name, fields with types, required/optional. |
| `interaction_contracts` | For each pair of components that communicate: `from`, `to`, `what_is_passed`, `format` (complete type definition referencing shared_types), `owner` (which component is responsible for the data), `on_failure` |
| `cross_cutting_concerns` | Concerns that span multiple components: `name`, `description`, `affected_components`, `owner_component` (which component is responsible for enforcing it). Includes: error propagation strategy, logging, security boundary, data persistence ownership, observability. |
| `technical_decisions` | Key architecture decisions: `decision`, `rationale`, `alternatives_considered`, `tradeoffs_accepted` |
| `deferred_items` | Things explicitly not built in this version: `item`, `reason`, `reconsider_when` |

The component list in this document is the authoritative input to Stages 3, 4, and 5.
It is locked on approval — changes require returning to Stage 2.

**Validation criteria (orchestrator checks before human gate):**
- All schema fields present and non-empty
- Every component has a non-empty `boundary`, `inputs`, `outputs`, and `platform`
- Every component's `platform` value appears in the Platform-Skill Map below
- Every cross-component communication has a corresponding `interaction_contract`
- `cross_cutting_concerns` is non-empty
- Every component name is unique within the list

**Feasibility check (orchestrator runs at Stage 2 human gate, before presenting):**
For each component, look up its `platform` in the Platform-Skill Map. If the mapped
Stage 4 skill status is MISSING, include a non-blocking warning in the gate presentation:

```
⚠ Component '[name]' platform '[platform]' has no matching Stage 4 skill.
  Stage 4 will emit a gap notice when this component is reached.
  Proceed and build the skill before Stage 4, or redesign the component now.
```

This warning does not block Stage 2 approval — the human decides whether to proceed.

**Human gate:**
Present a summary and reference `output/stage-2-system-design.md`. Human reads the file and either:
- **APPROVE** → pipeline advances to Stage 3. Component list is locked.
- **REJECT** with feedback → rework specified items, re-present.

**Dependencies:** Stage 1 complete and approved.

---

## Stage 3 — Component Specification

**Skills:** `spec-contract-methodology` + `engineering-baseline` — EXISTS
- `spec-contract-methodology`: EXISTS
- `engineering-baseline`: EXISTS (owned by this orchestrator, at `references/engineering-baseline.md`)

**Parallelism:** Parallel per component

**Output file path (per component):** `output/stage-3-spec-[component-name].md`
**Decision log (per component):** `output/decisions-stage3-[component-name].md`
**State file (per component):** `output/state-stage3-[component-name].md`

**Context package per parallel agent:**
- Full System Design Document from Stage 2 (`output/stage-2-system-design.md`)
- This component's definition extracted from the System Design Document
- The `spec-contract-methodology` skill
- `references/engineering-baseline.md` — mandatory, included in every package
- Iteration > 1: prior artifact + decision log for this component

**What to record in decision log:** Which component shape(s) apply and why, §2b structural
decisions and alternatives rejected, failure mode naming choices, blocking gaps found by the
verification pass (Tools A+B+C) and how each was resolved, non-blocking gaps and why they
do not block handoff, any provisional decisions flagged in the spec.

Each agent drives its component to a completed §1-§23 specification using `assisted-epistemology`
to iterate until zero blocking gaps remain. The engineering baseline defines the floor — anything
in the baseline does not need to be re-established from scratch in the spec, but the spec must
be consistent with it.

**Output schema — Component Specification (per component):**

| Field | Description |
|---|---|
| `component_name` | Matches the name in the System Design Document |
| `spec_document` | Full §1-§23 specification |
| `implementation_readiness` | Must be READY (zero blocking gaps) before this component can proceed to Stage 4 |
| `verification_currency` | Must be CURRENT |
| `residual_gaps` | Non-blocking gaps recorded but not blocking handoff |

**Validation criteria (orchestrator checks before human gate):**
- `implementation_readiness` is READY for every component
- `verification_currency` is CURRENT for every component
- Every component name matches an entry in the Stage 2 component list
- No component's spec is missing a §1-§23 section

**Human gate:**
Present a readiness summary (component name + READY/READY WITH RESIDUALS per component) and
file references for each spec. Human reads the files and either:
- **APPROVE ALL** → pipeline advances to Stage 4
- **REJECT** specific components with feedback → rework those components only. Approved components do not re-run.

**Dependencies:** Stage 2 complete and approved.

---

## Stage 4 — Implementation

**Skills:** Platform engineering skill (per component) + `code-integrity-guardrail` — EXISTS
- `android-engineering`: EXISTS
- `python-engineering`: EXISTS
- `go-engineering`: EXISTS
- `code-integrity-guardrail`: EXISTS

Platform skill selection is **per component**, not per project. Each component's `platform`
field from Stage 2 determines which skill it receives. See Platform-Skill Map at the end of
this file. If a component's platform has no matching skill → emit a gap notice for that
component only. Other components proceed.

**Parallelism:** Parallel per component

**Output file path (per component):** `output/stage-4-impl-[component-name].md`
**Decision log (per component):** `output/decisions-stage4-[component-name].md`
**State file (per component):** `output/state-stage4-[component-name].md`

**Context package per parallel agent:**
- Full System Design Document from Stage 2 (`output/stage-2-system-design.md`)
- This component's specification from Stage 3 (`output/stage-3-spec-[component-name].md`)
- The platform engineering skill matching this component's `platform` field (from Platform-Skill Map)
- The `code-integrity-guardrail` skill with the language binding matching this component's platform
- Iteration > 1: prior artifact (`output/stage-4-impl-[component-name].md`) + decision log

**What to record in decision log:** `SPEC_ERROR_REVEALED` deviations found and how resolved
(these require spec amendment — record both the deviation and the spec change), significant
implementation decisions not specified in the spec, guardrail findings and how each was
resolved, any DI or structural choices made that the spec left to the implementor.

**Output schema — Implementation (per component):**

| Field | Description |
|---|---|
| `component_name` | Matches the name in Stage 3 |
| `source_files` | Implemented source file paths |
| `build_result` | PASS (code compiles/builds) or FAIL with error output. Compilation is verified before static analysis. |
| `guardrail_result` | PASS or list of findings |
| `deviations` | List of deviations, each with type: `IMPLEMENTATION_DRIFT`, `SPEC_ERROR_REVEALED`, or `UPSTREAM_REDESIGN_REQUIRED` |

**Deviation types:**

- `IMPLEMENTATION_DRIFT` — spec is correct; implementation does not satisfy the contract. Agent fixes before submitting output.
- `SPEC_ERROR_REVEALED` — implementation reveals a spec error (contradiction, missing clause, wrong requirement). Routes to Stage 3 for spec amendment.
- `UPSTREAM_REDESIGN_REQUIRED` — a shared type defined in Stage 2 cannot be implemented as specified on this platform. Routes to Stage 2. This is a structural failure: the design must be revisited before implementation can proceed.

**Validation criteria (orchestrator checks before human gate):**
- `build_result` is PASS for every component
- `guardrail_result` is PASS for every component (`IMPLEMENTATION_DRIFT` items resolved before submission)
- Every component name matches Stage 3
- All source file paths listed exist
- No unresolved `UPSTREAM_REDESIGN_REQUIRED` deviations

**Human gate:**
- **No `SPEC_ERROR_REVEALED` or `UPSTREAM_REDESIGN_REQUIRED` deviations, build PASS, guardrail PASS** → advance automatically to Stage 5. No human input needed.
- **`SPEC_ERROR_REVEALED` deviations** → present to human with file references. Human decides: amend spec (returns to Stage 3 for that component) or confirm implementation is correct and spec needs amendment.
- **`UPSTREAM_REDESIGN_REQUIRED` deviations** → present to human. Describe what was attempted and why the shared type cannot be implemented. Routes to Stage 2 — the shared type must be redesigned. This invalidates Stage 3 and Stage 4 work for all components that use the affected shared type. Human confirms scope of impact before returning to Stage 2.
- **`IMPLEMENTATION_DRIFT`** → agent fixes and re-runs guardrail before output is written. Does not surface to human unless re-run also fails.
- **`build_result` FAIL** → not surfaced to human. Treated as a validation failure — agent spawned with build error output as context. If build fails after 3 attempts, escalate per Escalation Protocol.

**Dependencies:** Stage 3 complete and approved.

---

## Stage 5 — Component Testing

**Skills:**
- `spec-driven-testing`: EXISTS
- `code-integrity-guardrail` Phase 0 (lint): EXISTS
- `spec-audit`: EXISTS

**Parallelism:** Parallel per component. Within each component, three sub-tasks run in parallel.

**Output file path (per component):** `output/stage-5-test-[component-name].md`
**Decision log (per component):** `output/decisions-stage5-[component-name].md`
**State file (per component):** `output/state-stage5-[component-name].md`

**Context package per parallel agent:**
- This component's source files from Stage 4 (paths listed in `output/stage-4-impl-[component-name].md`)
- This component's specification from Stage 3 (`output/stage-3-spec-[component-name].md`)
- The `spec-driven-testing` skill
- The `code-integrity-guardrail` skill
- The `spec-audit` skill (or gap notice if MISSING — the agent records SKIPPED for that sub-task)
- Iteration > 1: prior test results artifact + decision log

**What to record in decision log:** Spec compliance deviations found and how routed
(IMPLEMENTATION_DRIFT vs SPEC_ERROR_REVEALED), behavioral test failures and whether they
route to stage 4 or stage 3, lint findings and how resolved. Stage 5 rarely iterates at the
orchestrator level — failures typically route back to stage 4 rather than producing a new
stage 5 iteration. If stage 5 does iterate, record why the prior routing decision was
insufficient.

Each component agent runs three sub-tasks in parallel:

| Sub-task | Skill | What it checks |
|---|---|---|
| Lint | `code-integrity-guardrail` Phase 0 | Static analysis, type checking, known anti-patterns |
| Behavioral testing | `spec-driven-testing` | Tests written from spec shapes only, not implementation. Format assertion requirement applies. |
| Specification compliance | `spec-audit` | Every RFC 2119 MUST contract in the spec confirmed present in implementation |

**Output schema — Testing Results (per component):**

| Field | Description |
|---|---|
| `component_name` | Matches Stage 4 |
| `lint_result` | PASS or issues list |
| `behavioral_test_result` | Test count, pass count, fail count, spec gaps identified |
| `test_execution_artifact` | File path to raw test runner output. Required. Cannot be substituted with a claim — the file must exist and contain runner syntax. |
| `spec_compliance_result` | COMPLETE or deviation list (when spec-audit exists); SKIPPED if missing |

**Test execution artifact requirement:**

The agent MUST actually run the test suite and write the raw output to a file. Claims of test passage without this artifact are rejected. The artifact is the unfakeable evidence that tests ran.

| Platform | Command | Artifact path | Evidence pattern |
|---|---|---|---|
| Go | `go test -v ./... -count=1 2>&1` | `output/test-output-[component].txt` | Lines containing `--- PASS:` or `--- FAIL:` |
| Android (unit) | `./gradlew test` | `app/build/reports/tests/testDebugUnitTest/` | JUnit XML files at this path |
| Android (instrumentation) | `./gradlew connectedAndroidTest` | `app/build/outputs/androidTest-results/` | JUnit XML files at this path |
| Python | `pytest -v 2>&1` | `output/test-output-[component].txt` | Lines containing `PASSED` or `FAILED` |

**Validation criteria (orchestrator checks before human gate):**
- Every component has all four fields populated (`SKIPPED` counts for spec-audit only)
- Every component name matches Stage 4
- `lint_result` and `behavioral_test_result` present for every component
- `test_execution_artifact` file exists and is non-empty for every component
- `test_execution_artifact` contains platform-appropriate runner syntax (see table above)
- If `test_execution_artifact` is missing or empty: treated as a validation failure. Orchestrator does not surface this to the human — re-spawns the agent with explicit instruction to run the tests and capture output.

**Human gate:**
- **All sub-tasks pass (or SKIPPED)** → advance automatically to Stage 6. No human input needed.
- **Failures present** → present failing components with file references. Human routes each failure:
  - Behavioral test failure or spec compliance deviation → Stage 4 (fix implementation) or Stage 3 (amend spec)
  - Lint failure → Stage 4 (fix code)

**Dependencies:** Stage 4 complete for all components.

---

## Stage 6 — Integration Testing

**Skill:** `integration-testing` — EXISTS
**Parallelism:** Sequential — integration tests the assembled whole

**Output file path:** `output/stage-6-integration.md`
**Decision log:** `output/decisions-stage6.md`

**Context package:**
- All component source files from Stage 4
- System Design Document from Stage 2 (`output/stage-2-system-design.md`) — interaction contracts define what integration tests verify
- Iteration > 1: prior test results artifact + decision log

**What to record in decision log:** Contract violations found and the routing decision
(which component is the fix target and why), MANUAL_REQUIRED procedures written and why
they cannot be automated, any contracts that produced ambiguous results and how the
ambiguity was resolved.

**What integration testing verifies:**
Each interaction contract in the System Design Document is a test target.
Integration tests verify that the actual assembled components honor these contracts —
no mocks, no fakes, real implementations communicating.

**Output schema — Integration Test Results:**

| Field | Description |
|---|---|
| `contracts_tested` | List of interaction contracts tested, each with result: PASS or FAIL |
| `contract_violations` | Detailed description of any violated contracts |
| `failed_components` | Which components are involved in each violation |

**Validation criteria (orchestrator checks before human gate):**
- Every interaction contract from Stage 2 appears in `contracts_tested`
- Every failed contract has a corresponding entry in `contract_violations` with enough detail to route the failure

**Human gate:**
- **All contracts pass** → advance to Commissioning
- **Failures** → present violated contracts with file reference. Human routes failures to the relevant component (Stage 4 or Stage 5 for the failing component).

**Dependencies:** Stage 5 complete for all components.

---

## Commissioning (Consumer Profile)

Not a stage — a milestone. The product works. Human accepts delivery.

**Consumer profile:** Reached when Stage 6 passes and human approves the integration test results.

**Enterprise profile:** Commissioning is reached only after Stages 7, 8, and 9 all pass. Stage 6 approval advances to Stage 7, not directly to commissioning.

**Custom profile:** Human specifies which of Stages 7–9 apply at pipeline start. Commissioning is reached when all selected stages pass.

---

## Stage 7 — System Security Acceptance Testing (SSAT)

**Skill:** `ssat` — MISSING
**Parallelism:** Sequential — tests the assembled system
**Applies to:** Enterprise and custom profiles only

**Inputs:**
- All component source files from Stage 4
- System Design Document from Stage 2 (security boundary, auth components, data ownership)
- Component specs from Stage 3 (§19 Security Properties, §22 Assumptions on caller trust)

**What SSAT verifies:**
Each security control declared in the spec is tested to confirm it functions as designed:
- Authentication: unauthenticated requests are rejected at the declared boundary
- Authorisation: requests with insufficient privilege are denied
- Input validation: boundary conditions in §8 are enforced against hostile input
- Encryption: data at rest and in transit matches §19 contracts
- Session management: session expiry, token invalidation, re-authentication triggers
- Audit logging: §18 events are emitted correctly for all security-relevant actions

**Output schema:**

| Field | Description |
|---|---|
| `controls_tested` | List of security controls tested, each with: source (§N), result (PASS/FAIL/MANUAL_REQUIRED) |
| `control_failures` | Detailed description of any failed controls |
| `manual_procedures` | Written test procedures for controls requiring human execution |

**Human gate:**
- **All controls pass (or MANUAL_REQUIRED with procedures)** → advance to Stage 8
- **Failures** → route to component responsible for the failed control (Stage 4 or Stage 3 if contract was wrong)

**Dependencies:** Stage 6 complete and approved.

---

## Stage 8 — Vulnerability Assessment and Penetration Testing (VAPT)

**Skill:** `vapt` — MISSING
**Parallelism:** Sequential
**Applies to:** Enterprise and custom profiles only

**Inputs:**
- Running system (deployed from Stage 4 artifacts)
- System Design Document from Stage 2 (attack surface, trust boundaries, external dependencies)
- SSAT results from Stage 7

**What VAPT verifies:**
An external attacker cannot breach the system via known vulnerability classes:
- Automated vulnerability scanning (OWASP Top 10 surface coverage)
- Injection vectors: SQL, command, LDAP, XML
- Authentication bypass: brute force thresholds, credential stuffing surface
- Authorisation bypass: IDOR, privilege escalation paths
- Configuration weaknesses: exposed debug endpoints, default credentials, verbose errors
- Dependency vulnerabilities: known CVEs in production dependencies
- Network exposure: open ports beyond declared surface

**Note:** VAPT requires a running deployment and specialised tooling (scanners, proxies). The
skill orchestrates the process and integrates results. Skilled human review is required for
findings classification — the skill does not replace a security professional.

**Output schema:**

| Field | Description |
|---|---|
| `findings` | List of findings, each with: CVE/class, severity (CRITICAL/HIGH/MEDIUM/LOW/INFO), affected component, evidence |
| `remediation_required` | Findings classified as blocking: CRITICAL and HIGH |
| `accepted_risks` | LOW and INFO findings accepted by the human with rationale |

**Human gate:**
- **Zero CRITICAL or HIGH findings** → advance to Stage 9
- **CRITICAL or HIGH findings** → route to Stage 4 for remediation. Re-run Stage 8 after fix.

**Dependencies:** Stage 7 complete and approved. System must be deployed and accessible for scanning.

---

## Stage 9 — Stability Run

**Skill:** `stability-testing` — MISSING
**Parallelism:** Sequential
**Applies to:** Enterprise profile, custom profile, or any consumer project where §23 Performance Contracts must be verified

**Inputs:**
- Running system (same deployment as Stage 8, or fresh deployment)
- Component specs from Stage 3 (§23 Performance Contracts)
- System Design Document from Stage 2 (scaling model, deployment unit)

**What the stability run verifies:**
- **Load test:** system meets §23 performance contracts (response time, throughput) at expected peak load
- **Stress test:** system degrades gracefully beyond peak load — no data corruption, clean recovery
- **Soak test:** system is stable under sustained load over time — no memory leaks, connection exhaustion, log overflow
- **Recovery test:** system recovers correctly from failure injection (process kill, database disconnect, dependency timeout)

**Output schema:**

| Field | Description |
|---|---|
| `load_test_result` | PASS/FAIL per §23 contract tested, with measured values vs. targets |
| `stress_test_result` | Degradation behaviour at 2× and 5× peak load |
| `soak_test_result` | Resource consumption trends over soak duration |
| `recovery_test_result` | Recovery time and data integrity after each failure injection |
| `contracts_verified` | Which §23 Performance Contracts were verified and result |

**Human gate:**
- **All §23 contracts satisfied, no resource leaks, recovery within bounds** → advance to Commissioning
- **Failures** → route to Stage 4 for performance or stability fixes. Re-run affected test types only.

**Note on selective use:** Stage 9 can be run after Stage 6 for consumer projects where performance
contracts are critical, without running Stages 7 and 8. The pipeline profile determines this.

**Dependencies:** Stage 6 complete and approved. System must be deployed under conditions
representative of production (not localhost, not debug build).

---

## Platform-Skill Map

Maps each component `platform` value to the Stage 4 engineering skill and the matching
`code-integrity-guardrail` language binding. Used at Stage 2 feasibility check and at
Stage 4 agent spawn time.

| Platform value | Stage 4 skill | Guardrail binding | Status |
|---|---|---|---|
| `android` | `android-engineering` | `kotlin` | EXISTS |
| `go` | `go-engineering` | `go` | EXISTS |
| `python` | `python-engineering` | `python` | EXISTS |
| `typescript` | `typescript-engineering` | `typescript` | MISSING |
| `nodejs` | `nodejs-engineering` | `typescript` | MISSING |
| `swift-ios` | `ios-engineering` | `swift` | MISSING |
| `rust` | `rust-engineering` | `rust` | MISSING |

**Adding a new platform:** Create the platform engineering skill, add the guardrail binding
at `code-integrity-guardrail/references/bindings/<binding>.md`, then add a row here with
status EXISTS. The feasibility check at Stage 2 and the skill selection at Stage 4 are both
driven from this table.
