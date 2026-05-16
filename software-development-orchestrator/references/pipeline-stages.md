# Pipeline Stage Definitions

Each stage defines: inputs, skill status, parallelism, output schema, human gate, and dependencies.

**Skill status:** EXISTS = skill is available and can be invoked. MISSING = no skill exists;
pipeline pauses and emits a gap notice when this stage is reached.

---

## Stage 1 — Requirements Engineering

**Skill:** `requirements-engineering` — EXISTS
**Parallelism:** Sequential

**Inputs:**
- Raw human intent (unstructured natural language string)

**Output file path:** `output/stage-1-requirements.md`

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

**Output file path:** `output/stage-2-system-design.md`

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

**Validation criteria (orchestrator checks before human gate):**
- All schema fields present and non-empty
- Every component has a non-empty `boundary`, `inputs`, and `outputs`
- Every cross-component communication has a corresponding `interaction_contract`
- `cross_cutting_concerns` is non-empty
- Every component name is unique within the list

**Human gate:**
Present a summary and reference `output/stage-2-system-design.md`. Human reads the file and either:
- **APPROVE** → pipeline advances to Stage 3. Component list is locked.
- **REJECT** with feedback → rework specified items, re-present.

**Dependencies:** Stage 1 complete and approved.

---

## Stage 3 — Component Specification

**Skills:** `assisted-epistemology` + `spec-contract-methodology` + `engineering-baseline` — PARTIALLY EXISTS
- `spec-contract-methodology`: EXISTS
- `assisted-epistemology`: EXISTS (spec at v0.1-draft, not yet verified)
- `engineering-baseline`: EXISTS (owned by this orchestrator, at `references/engineering-baseline.md`)

**Parallelism:** Parallel per component

**Output file path (per component):** `output/stage-3-spec-[component-name].md`

**Context package per parallel agent:**
- Full System Design Document from Stage 2 (`output/stage-2-system-design.md`)
- This component's definition extracted from the System Design Document
- The `spec-contract-methodology` skill
- The `assisted-epistemology` skill
- `references/engineering-baseline.md` — mandatory, included in every package

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

**Skills:** Platform engineering skill + `code-integrity-guardrail` — EXISTS
- `android-engineering`: EXISTS
- `python-engineering`: EXISTS
- `code-integrity-guardrail`: EXISTS

Select the platform engineering skill based on the target platform identified in the
Requirements Document.

**Parallelism:** Parallel per component

**Output file path (per component):** `output/stage-4-impl-[component-name].md`

**Context package per parallel agent:**
- Full System Design Document from Stage 2 (`output/stage-2-system-design.md`)
- This component's specification from Stage 3 (`output/stage-3-spec-[component-name].md`)
- The appropriate platform engineering skill for this component's platform
- The `code-integrity-guardrail` skill

**Output schema — Implementation (per component):**

| Field | Description |
|---|---|
| `component_name` | Matches the name in Stage 3 |
| `source_files` | Implemented source file paths |
| `guardrail_result` | PASS or list of findings |
| `deviations` | Any `IMPLEMENTATION_DRIFT` or `SPEC_ERROR_REVEALED` deviations from the component spec |

**Validation criteria (orchestrator checks before human gate):**
- `guardrail_result` is PASS for every component (IMPLEMENTATION_DRIFT items must have been resolved and re-run before presenting)
- Every component name matches Stage 3
- All source file paths listed exist

**Human gate:**
- **No `SPEC_ERROR_REVEALED` deviations, guardrail PASS** → advance automatically to Stage 5. No human input needed.
- **`SPEC_ERROR_REVEALED` deviations** → present to human with file references. Human decides: amend spec (returns to Stage 3 for that component) or confirm implementation is correct and spec needs correction.
- **`IMPLEMENTATION_DRIFT`** → agent fixes and re-runs guardrail before output is written. Does not surface to human unless re-run also fails.

**Dependencies:** Stage 3 complete and approved.

---

## Stage 5 — Component Testing

**Skills:**
- `spec-driven-testing`: EXISTS
- `code-integrity-guardrail` Phase 0 (lint): EXISTS
- `spec-audit`: EXISTS

**Parallelism:** Parallel per component. Within each component, three sub-tasks run in parallel.

**Output file path (per component):** `output/stage-5-test-[component-name].md`

**Context package per parallel agent:**
- This component's source files from Stage 4 (paths listed in `output/stage-4-impl-[component-name].md`)
- This component's specification from Stage 3 (`output/stage-3-spec-[component-name].md`)
- The `spec-driven-testing` skill
- The `code-integrity-guardrail` skill
- The `spec-audit` skill (or gap notice if MISSING — the agent records SKIPPED for that sub-task)

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
| `behavioral_test_result` | Test count, pass count, fail count, spec gaps identified by Agent 2 |
| `spec_compliance_result` | COMPLETE or deviation list (when spec-audit exists); SKIPPED if missing |

**Validation criteria (orchestrator checks before human gate):**
- Every component has all three sub-task fields populated (SKIPPED counts as populated for spec-audit while missing)
- Every component name matches Stage 4
- `lint_result` and `behavioral_test_result` present for every component

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

**Context package:**
- All component source files from Stage 4
- System Design Document from Stage 2 (`output/stage-2-system-design.md`) — interaction contracts define what integration tests verify

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

## Commissioning

Not a stage — a milestone. The product works. Human accepts delivery.
Reached when Stage 6 passes and human approves the integration test results.
