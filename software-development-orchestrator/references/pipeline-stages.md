# Pipeline Stage Definitions

Each stage defines: inputs, skill status, parallelism, output schema, human gate, and dependencies.

**Skill status:** EXISTS = skill is available and can be invoked. MISSING = no skill exists;
pipeline pauses and emits a gap notice when this stage is reached.

---

## Stage 1 — Requirements Engineering

**Skill:** `requirements-engineering` — MISSING
**Parallelism:** Sequential

**Inputs:**
- Raw human intent (unstructured natural language string)

**Output schema — Requirements Document:**

| Field | Description |
|---|---|
| `project_name` | Short name for this project, used in all subsequent artifacts |
| `goal` | Single sentence: what is being built and why |
| `features` | List of features, each with: name, description, priority (core / should-have / deferred) |
| `platform_constraints` | Constraints imposed by the target platform (OS version requirements, runtime limits, deployment environment, regulatory) |
| `surfaced_concerns` | Concerns the human did not raise but domain knowledge requires addressing: security posture, observability, audit trail, error strategy, data retention, compliance, failure modes. These MUST be surfaced regardless of whether the human mentioned them. |
| `scope_inclusions` | What is explicitly in scope for this build |
| `scope_exclusions` | What is explicitly out of scope, with brief reason |
| `open_questions` | Unresolved items requiring a human decision before system design can proceed. MAY be empty. |
| `assumptions` | Things taken as given without proof — environmental, caller, operational |

**Human gate:**
Present the full Requirements Document. Human reviews and either:
- **APPROVE** → pipeline advances to Stage 2
- **REJECT** with feedback → rework specified items, re-present. Do not advance without approval.

**Dependencies:** None. This is the pipeline entry point.

---

## Stage 2 — System Design

**Skill:** `system-design` — MISSING
**Parallelism:** Sequential

**Inputs:**
- Approved Requirements Document from Stage 1

**Output schema — System Design Document:**

| Field | Description |
|---|---|
| `components` | List of components. Each component: `name`, `purpose` (one sentence), `boundary` (what it owns and what it does not own), `inputs` (what it receives), `outputs` (what it produces) |
| `interaction_contracts` | For each pair of components that communicate: `from`, `to`, `what_is_passed`, `format`, `owner` (which component is responsible for the data) |
| `cross_cutting_concerns` | Concerns that span multiple components: `name`, `description`, `affected_components`, `owner_component` (which component is responsible for enforcing it). Includes: error propagation strategy, logging, security boundary, data persistence ownership, observability. |
| `technical_decisions` | Key architecture decisions: `decision`, `rationale`, `alternatives_considered`, `tradeoffs_accepted` |
| `deferred_items` | Things explicitly not built in this version: `item`, `reason`, `reconsider_when` |

The component list in this document is the authoritative input to Stages 3, 4, and 5.
It is locked on approval — changes require returning to Stage 2.

**Human gate:**
Present the full System Design Document. Human reviews and either:
- **APPROVE** → pipeline advances to Stage 3. Component list is locked.
- **REJECT** with feedback → rework specified items, re-present.

**Dependencies:** Stage 1 complete and approved.

---

## Stage 3 — Component Specification

**Skills:** `assisted-epistemology` + `spec-contract-methodology` + `engineering-baseline` — PARTIALLY EXISTS
- `spec-contract-methodology`: EXISTS
- `assisted-epistemology`: EXISTS (spec at v0.1-draft, not yet verified)
- `engineering-baseline`: MISSING

**Parallelism:** Parallel per component

**Context package per parallel agent:**
- Full System Design Document from Stage 2
- This component's definition (name, purpose, boundary, inputs, outputs)
- The `spec-contract-methodology` skill
- The `engineering-baseline` (when built)

Each agent drives its component to a completed §1-§23 specification using `assisted-epistemology`
to iterate until zero blocking gaps remain.

**Output schema — Component Specification (per component):**

| Field | Description |
|---|---|
| `component_name` | Matches the name in the System Design Document |
| `spec_document` | Full §1-§23 specification |
| `implementation_readiness` | Must be READY (zero blocking gaps) before this component can proceed to Stage 4 |
| `verification_currency` | Must be CURRENT |
| `residual_gaps` | Non-blocking gaps recorded but not blocking handoff |

**Human gate:**
Present all component specifications as a set, with implementation readiness status for each.
Human reviews and either:
- **APPROVE ALL** → pipeline advances to Stage 4
- **REJECT** specific components with feedback → rework those components only. Approved components do not re-run.

**Dependencies:** Stage 2 complete and approved.

---

## Stage 4 — Implementation

**Skills:** Platform engineering skill + `code-integrity-guardrail` — EXISTS
- `android-engineering`: EXISTS
- `python-engineering`: EXISTS
- `code-integrity-guardrail`: EXISTS

Select the platform engineering skill based on the target platform identified in the
Requirements Document.

**Parallelism:** Parallel per component

**Context package per parallel agent:**
- Full System Design Document from Stage 2
- This component's specification from Stage 3
- The appropriate platform engineering skill
- The `code-integrity-guardrail` skill

**Output schema — Implementation (per component):**

| Field | Description |
|---|---|
| `component_name` | Matches the name in Stage 3 |
| `source_files` | Implemented code |
| `guardrail_result` | PASS or list of findings |
| `deviations` | Any `IMPLEMENTATION_DRIFT` or `SPEC_ERROR_REVEALED` deviations from the component spec |

**Human gate:**
- **No deviations, guardrail PASS** → advance automatically to Stage 5. No human input needed.
- **`SPEC_ERROR_REVEALED` deviations** → present to human. Human decides: amend spec (returns to Stage 3 for that component) or confirm implementation is correct and spec is wrong.
- **`IMPLEMENTATION_DRIFT`** → agent fixes and re-runs guardrail. Does not surface to human unless re-run also fails.

**Dependencies:** Stage 3 complete and approved.

---

## Stage 5 — Component Testing

**Skills:**
- `spec-driven-testing`: EXISTS
- `code-integrity-guardrail` Phase 0 (lint): EXISTS
- `spec-audit`: MISSING

**Parallelism:** Parallel per component. Within each component, three sub-tasks run in parallel.

**Context package per parallel agent:**
- This component's source code from Stage 4
- This component's specification from Stage 3
- All three testing skills (or gap notice for `spec-audit`)

Each component agent runs three sub-tasks in parallel:

| Sub-task | Skill | What it checks |
|---|---|---|
| Lint | `code-integrity-guardrail` Phase 0 | Static analysis, type checking, known anti-patterns |
| Behavioral testing | `spec-driven-testing` | Tests written from spec shapes only, not implementation. Format assertion requirement applies. |
| Specification compliance | `spec-audit` — MISSING | Every RFC 2119 MUST contract in the spec confirmed present in implementation |

**Output schema — Testing Results (per component):**

| Field | Description |
|---|---|
| `component_name` | Matches Stage 4 |
| `lint_result` | PASS or issues list |
| `behavioral_test_result` | Test count, pass count, fail count, spec gaps identified by Agent 2 |
| `spec_compliance_result` | COMPLETE or deviation list (when spec-audit exists); SKIPPED if missing |

**Human gate:**
- **All sub-tasks pass** → advance automatically to Stage 6. No human input needed.
- **Failures present** → present failing components with details. Human routes each failure:
  - Behavioral test failure or spec compliance deviation → Stage 4 (fix implementation) or Stage 3 (amend spec)
  - Lint failure → Stage 4 (fix code)

**Dependencies:** Stage 4 complete for all components.

---

## Stage 6 — Integration Testing

**Skill:** `integration-testing` — MISSING (approach not yet decided)
**Parallelism:** Sequential — integration tests the assembled whole

**Inputs:**
- All implemented components from Stage 4
- System Design Document from Stage 2 (interaction contracts define what integration tests verify)

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

**Human gate:**
- **All contracts pass** → advance to Commissioning
- **Failures** → present violated contracts with detail. Human routes failures to the relevant component (Stage 4 or Stage 5 for the failing component).

**Dependencies:** Stage 5 complete for all components.

---

## Commissioning

Not a stage — a milestone. The product works. Human accepts delivery.
Reached when Stage 6 passes and human approves the integration test results.
