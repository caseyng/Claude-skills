# Appendix C — LLM-Executable Verification Pipeline

This appendix formalises the verification methodology for execution by an LLM.
It covers the pipeline shape, the re-grounding mechanism, the gap report format,
and the integration of stress tests and the completeness checklist into a single pass.

**Read `spec-verification.md` first.** The verification methodology itself — the five
steps, the fixed cross-reference matrix, the invariant set structure — is there. This
appendix describes how to execute it reliably as a machine process.

---

## The Pipeline Shape

Spec work involves two distinct LLM phases with different failure modes.

**Generation phase.** The LLM fills all 23 sections. Where the input is ambiguous, the
LLM makes a provisional decision and flags it explicitly. The spec remains in `DRAFT`
state until all flags are resolved by human review.

**Verification phase.** A single integrated pass combining three complementary tools.
Each tool attacks a different surface. Running them separately across multiple sessions
is the primary cause of gaps surviving into implementation. They MUST be run together,
in the order below, in a single verification pass.

```
Input (idea / codebase / existing docs)
  → [Generation phase]
      LLM fills §1–§23
      Ambiguities → provisional answer + FLAG
      State: DRAFT until all FLAGs resolved by human

  → [Verification phase — single integrated pass]

      TOOL A: Completeness Sweep (25Q checklist from writing-process.md)
        Run all 25 questions against the spec.
        Each question maps to a section. A question with no satisfying answer = gap.
        Produces: checklist-findings.md
        Re-grounds on: nothing (first tool, reads spec directly)

      TOOL B: Stress Tests (8 vectors from stress-tests.md)
        Apply all 8 vectors. Each vector attacks a different reading strategy.
        A vector that raises a question the spec cannot answer = gap.
        Produces: stress-findings.md
        Re-grounds on: checklist-findings.md (to avoid re-finding the same gaps)

      TOOL C: Structural Verification (Steps 1–5)
        Step 1: Build Registry                → registry.md
        Step 2: Build Cross-Reference Matrix  → matrix.md        (re-grounds on: registry.md + fixed rules)
        Step 3: Build Invariant Sets          → invariants.md    (re-grounds on: registry.md + matrix.md)
        Step 4: Trace Execution Paths         → paths.md         (re-grounds on: invariants.md)
        Step 5: Verify Intent                 → intent-notes.md  (re-grounds on: full spec + paths.md)
        Re-grounds on: stress-findings.md before Step 1 (to avoid re-finding the same gaps)

      → Gap report (assembles findings from Tools A + B + C, deduplicated)
      → Each gap classified blocking or non-blocking
      Implementation Readiness: READY (0 blocking) | NOT READY (N blocking)

  → [Human review]
      Human receives: draft spec + gap report (together, not separately)
      Human resolves: FLAGs and blocking GAPs (non-blocking GAPs recorded, do not block)

  → [Resolution phase]
      LLM updates spec per human decisions on blocking gaps
      LLM re-runs verification (full integrated pass)

  → [Handoff gate]
      Implementation Readiness = READY (zero blocking gaps)
      Verification Currency = CURRENT
      → state: READY FOR IMPLEMENTATION HAND-OFF

  → [Implementation hand-off]
      Spec + implementation checklist → coding LLM
```

---

## Why the Three Tools Are Not Redundant

Each tool catches a class of gaps the others reliably miss.

**Tool A (25Q checklist)** sweeps specific question categories: boundary conditions,
sentinel values, assumptions, concurrency, callbacks, external dependencies. These are
enumerated questions with direct section mappings. A gap here means a specific question
has no satisfying answer in the spec. Steps 1–5 do not ask these questions — they check
whether entities that *appear in the spec* are covered correctly. They do not check
whether the right entities appear at all.

**Tool B (stress tests)** applies eight adversarial reading strategies — stateless
system, multi-tenant service, extension point, adversarial reader, etc. Each strategy
exposes a class of omission that is invisible from within the spec's own framing. A spec
that is internally consistent can still fail Vector 6 (adversarial reader) because it
leaves a MUST claim vacuously satisfiable. Steps 1–5 would not find this — the claim
exists and is cross-referenced correctly. Only the adversarial reading exposes it.

**Tool C (Steps 1–5)** checks internal consistency — whether entities that appear in
the spec are correctly cross-referenced across all required sections. This is the only
tool that catches contradictions (two sections making incompatible claims about the
same entity). Tools A and B read the spec section-by-section and do not systematically
cross-reference entities across the full section set.

Running all three in sequence, with re-grounding between them, is what produces a
zero-gap report in a single pass. Running them separately across sessions means each
session inherits an incomplete view of what has already been found.

---

## Why Re-grounding Is Structural, Not Optional

As a verification pass runs, context fills. The registry produced in Step 1 is further
back in context by Step 4. Cross-section consistency checks against content written
50,000 tokens earlier become unreliable. This is not an edge case — it is the default
behaviour of any LLM operating on a long spec.

The mitigation is externalisation. Each step produces a structured artefact — written
out, not held in memory. Each subsequent step receives that artefact as explicit input
before executing. The LLM is not asked to remember. It is given the information again.

Re-grounding granularity is per tool and per verification step.

**Re-grounding map:**

| Tool / Step | Re-ground on before starting |
|---|---|
| Tool A — 25Q checklist | Nothing. First tool, reads spec directly. |
| Tool B — Stress tests | `checklist-findings.md` |
| Tool C Step 1 — Build Registry | `stress-findings.md` |
| Tool C Step 2 — Build Matrix | `registry.md` + fixed required-presence rules |
| Tool C Step 3 — Build Invariant Sets | `registry.md` + `matrix.md` |
| Tool C Step 4 — Trace Execution Paths | `invariants.md` |
| Tool C Step 5 — Verify Intent | Full spec + `paths.md` |

"Re-ground" means: read the artefact in full at the start of the step. Do not proceed
until it is in context.

---

## The Fixed Cross-Reference Matrix

This table is a methodology-level contract. It does not change per spec. The LLM
applies it — it does not generate it.

| Entity type | Required in |
|---|---|
| Component | §2 (if named in prose), §5 (contract), §6 (lifecycle), §11 (pipeline position), §12 (if it has interaction contracts) |
| Public method | §3 (system boundary), §6 (lifecycle), §10 (atomicity) |
| Failure mode | §5 (component that produces it), §7 (taxonomy), §17 (propagation path), §18 (audit sequence if it changes the event set) |
| Audit event | §4 (schema), §18 (sequence diagrams) |
| Config field | §15 (definition), component contract that reads it |
| Extension abstraction | §16 (contract), §20 (stability level) |
| Sentinel value | §4 or §5 (where produced), §9 (encoding conventions) |
| State | §6 (lifecycle) |
| Named exception type | §2 (vocabulary), section where raised |
| Architectural constraint | §2b (class hierarchy, inheritance strategy, priority ordering, named patterns) |

Every MISSING cell in the per-spec matrix is a gap. A gap is not a judgment — it is
a binary result: required section is absent.

---

## The Self-Verification Problem and Why This Design Mitigates It

The LLM that wrote the spec runs the verification. The risk: the LLM cannot see its
own blind spots. It may resolve contradictions silently — confirming what it wrote
because it cannot perceive the error.

This design mitigates it through structural comparison, not judgment.

The gap report is produced by comparing two independently constructed artefacts:

1. **What the spec claims** — extracted into the registry and invariant sets
2. **What the cross-reference matrix requires** — derived from the fixed rules above

A gap is a diff between these two artefacts. The LLM is not making a judgment about
correctness. It is performing a structural comparison. A diff cannot be self-serving —
it is either present or absent.

The residual risk is systematic bias: if the LLM misunderstood the domain when writing
the spec, it will apply the same misunderstanding when building the matrix. Both
artefacts will agree, and both will be wrong in the same direction. Step 5 — intent
verification — is the only check that catches this. Step 5 cannot be automated. It
requires a human who understands the domain.

The integrated three-tool pass further reduces residual risk because the three tools
have different blind spots. A systematic bias that causes Tool C to miss a gap is less
likely to also cause Tool A's checklist questions to be answered satisfactorily and
Tool B's adversarial vectors to pass. The tools are structurally independent.

---

## The Gap Report Format

The gap report is the primary output of the verification phase and the primary input
to human review. Its structure must allow the human to make decisions without having
watched the verification happen.

**Format per gap entry:**

```
GAP-[N]
Tool:        [A (checklist) | B (stress test, vector N) | C-Step-N (structural)]
Entity:      [the registered entity name, exactly as in the registry]
Type:        MISSING | CONTRADICTION | WEAK_CLAIM | INTENT_FLAG
Blocking:    YES — [one sentence: why an implementor would build divergent behaviour without this]
             NO  — [one sentence: why the behaviour is inferrable from elsewhere in the spec]
Finding:     [one sentence: what is absent or inconsistent]
Evidence:    [section(s) and the specific claim(s) that conflict or are absent]
Required by: [the fixed rule or invariant that requires the missing content]
Resolution:  [OPEN — awaiting human decision]
```

The `Blocking` field is the key field for handoff decisions. It answers the test directly:
would a competent implementor diverge without this? The rationale must be stated — not
just YES or NO.

**Gap types:**

- `MISSING` — a required section does not mention an entity the matrix requires it to mention
- `CONTRADICTION` — two sections make claims about an entity that cannot both be true
- `WEAK_CLAIM` — a claim exists but is logically weaker than an equivalent claim in another section
- `INTENT_FLAG` — Step 5 only; a guarantee, assumption, or MUST with no traceable mechanism

**Gap report header:**

```
VERIFICATION PASS — [date]
Spec version:         [version]
Tools run:            [A + B + C (full) | A only | B + C | etc.]
Registry size:        [N entities]
Gap list:             [N blocking, M non-blocking]
Implementation Readiness: READY | NOT READY — see GAP-[N], GAP-[N]
Verification Currency:    CURRENT | N invalidating changes since last pass
```

---

## FLAG Format (Generation Phase)

During generation, when the LLM makes a provisional decision due to ambiguity:

```
FLAG-[N]
Section:     [§N]
Decision:    [what the LLM decided provisionally]
Reason:      [why the input was ambiguous]
Alternative: [the other plausible interpretation]
Impact:      [which other sections this decision affects]
Resolution:  [OPEN — awaiting human decision]
```

Flags must be self-explanatory. The human reads them without context from the
generation process.

---

## Spec Status Model

A spec is described by three independent values at any time.

---

### 1. Gap List — single source of truth

Every gap found by verification is classified:

- **Blocking** — a competent implementor reading the spec without this would build divergent behaviour. Prevents handoff.
  - Example: §2b absent for a class hierarchy — implementor would choose wrong inheritance strategy. = BLOCKING
- **Non-blocking** — missing cross-reference, absent §8 entry for behaviour already stated clearly elsewhere, §2 entry for a term whose meaning is inferrable. Does not prevent handoff.

The test: *"Would a competent implementor, reading the spec without this entry, build something that diverges from intended behaviour?"* Yes = blocking. No = non-blocking.

The gap list is expressed as: **N blocking, M non-blocking**.

---

### 2. Implementation Readiness — derived from gap list

| Value | Condition |
|---|---|
| `READY` | N = 0. Safe to hand off to a coding LLM. |
| `NOT READY` | N > 0. Lists the blocking gaps by ID. |

This is the handoff gate. Non-blocking gaps do not affect it.

---

### 3. Verification Currency — independent of gap list

Answers: is the gap list still trustworthy, or has the spec changed since the last pass in ways that may have introduced or resolved gaps?

| Value | Condition |
|---|---|
| `CURRENT` | Zero invalidating changes since last full pass. Gap list is trustworthy. |
| `N invalidating changes since last pass` | Re-run affected steps (per the invalidation table below) before trusting the gap list. |

A change is **invalidating** if and only if its type appears in the invalidation table. Rewording rationale, fixing typos, updating deployment guidance — not invalidating. Adding a component, renaming a method, adding a failure mode — invalidating. Currency is not time-based. A spec unchanged for two years is as current as one verified yesterday.

---

### Reading the status block

A spec header carries all three values:

```
Implementation Readiness:  READY
Gap List:                  0 blocking, 3 non-blocking
Verification Currency:     CURRENT
```

Or:

```
Implementation Readiness:  NOT READY — see GAP-3, GAP-7
Gap List:                  2 blocking, 5 non-blocking
Verification Currency:     2 invalidating changes since last pass (new failure mode, renamed method)
```

**A spec MUST NOT be handed to a coding LLM unless Implementation Readiness is `READY` and Verification Currency is `CURRENT`.**

---

## Design Rationale

**Why the goal is implementation readiness, not documentation completeness.**
A spec that chases zero gaps has no convergence criterion — every resolved gap introduces
new entities that the next pass examines. The correct exit condition is zero blocking gaps.
Non-blocking gaps do not cause implementation drift. Chasing them past the point of
readiness produces diminishing returns indefinitely. The verification pass still finds all
gaps — but the handoff gate checks only blocking gaps.

**Why blocking/non-blocking is a single test, not a category system.**
The test is: would a competent implementor, reading the spec without this entry, build
divergent behaviour? That question has a direct answer. A category system invites
negotiation. The single test does not.

**Why the three dimensions are orthogonal.**
Implementation Readiness is derived from the gap list. Verification Currency is independent
of the gap list — it measures whether the gap list is still valid. Conflating them (as a
single VERIFIED state does) means a spec with zero gaps but recent unverified changes
carries false confidence. Separating them makes each risk visible independently.

**Why currency is change-count not time-based.**
A spec unchanged for two years is as valid as one verified yesterday. Time is not the
variable — changes are. The invalidation table defines which changes matter. Currency
counts them.

**Why the three tools must run in a single pass.** Running them across separate sessions
means each session starts without the findings of prior tools in context. A gap found by
Tool A that should have been re-grounded into Tool C Step 1 instead gets re-discovered
in Step 3 or missed entirely. The integrated pass eliminates this by keeping all findings
in the same context window and using artefacts for explicit re-grounding.

**Why Tool A runs before Tool B.** The 25Q checklist produces findings in specific
question categories. Those categories (boundary conditions, sentinel values, concurrency)
are exactly what stress test Vectors 1–3 and 6–8 also probe. Running checklist first
means the stress tests re-ground on what's already been found, and focus on the
adversarial reading strategies that the checklist's enumerated questions don't cover.

**Why Tool B runs before Tool C.** Stress test findings inform what belongs in the
registry. A stress test that flags a missing lifecycle state (Vector 2) should cause
that state to appear in the registry so Tool C Step 2 checks for it in the cross-
reference matrix. Without this sequencing, the state gets missed by the matrix check
because it was never registered.

**Why the fixed matrix now includes named exception types.** The previous version of
the matrix omitted this row. Four named exception types survived two full verification
passes without §2 entries because no matrix rule required it. The row is added to
prevent recurrence.

**Why the spec has a status model.** Without explicit readiness state, "done" is
ambiguous. A coding LLM handed a NOT READY spec will implement against unresolved
ambiguities.

**Why flags must name the alternative interpretation.** A flag that says "ambiguous" is
useless for async review. The human needs to see what the LLM almost decided.

**Why gap entries reference the fixed rule.** Not just "§17 is missing" — but "missing
because: failure modes MUST have a propagation path per the fixed matrix." Without the
rule, the human cannot distinguish a genuine gap from a false positive.

**Why verification re-runs after resolving blocking gaps.** Resolving a blocking gap is
a spec change. A spec change can introduce new gaps. After resolving blocking gaps,
re-run affected steps before confirming READY.

**Why Step 5 cannot be automated.** Steps 1–4 verify internal consistency. Step 5
verifies intent — the spec describes the right thing. A spec can be internally
consistent and consistently wrong. No structural check can detect that. It requires a
human who understands the domain.

**Why human review receives spec and gap report together.** A gap report without a spec
is abstract. A spec without a gap report asks the human to re-do the verification.

---

## Relationship to Appendix A (Stress Tests) and Writing Process

The three tools are now integrated. Their relationship:

- **Tool A (25Q checklist)** is the "Before You Write" checklist from `writing-process.md`,
  repurposed as a verification tool. It was originally framed as pre-writing preparation.
  It is equally effective as a verification sweep — each question targets a specific gap
  category that neither stress tests nor structural verification systematically covers.

- **Tool B (stress tests)** is Appendix A (`stress-tests.md`). Eight vectors, all required.
  A spec that passes seven of eight has one uncovered dimension.

- **Tool C (Steps 1–5)** is the structural verification pipeline from this appendix.
  Vector 7 (Cross-Section Consistency) overlaps most closely with Steps 2–4 here.
  Both are required — Vector 7 asks specific questions; Steps 2–4 find failures Vector
  7's questions do not cover.

---

## Keeping Verification Current

Verification currency answers one question: is the gap list still trustworthy?

A change is **invalidating** if and only if its type appears in the table below. Any change
not in this table — rewording rationale, fixing typos, updating deployment guidance,
adding examples — is not invalidating. Currency is not time-based. A spec unchanged for
two years is as current as one verified yesterday.

When invalidating changes accumulate, re-run only the affected steps — not a full pass.
The invalidation table maps change type to the minimum re-run scope.

| Change type | Steps invalidated |
|---|---|
| New entity added | C Steps 1, 2, 3 |
| Entity renamed | C Steps 1, 2, 3 |
| New failure mode | C Steps 2, 3, 4 |
| New named exception type | A (Q5, Q22), C Steps 1, 2 |
| Pipeline ordering changed | C Step 4 |
| Audit sequence changed | C Step 4 |
| Security guarantee changed | C Step 5 |
| Design rationale changed | C Step 5 |
| New assumption added | C Step 5 |
| New boundary condition | A (Q7), C Step 3 |
| New public data format | A (Q22), C Steps 1, 2 |
| New or changed architectural constraint (§2b) | A (Q26), C Steps 1, 2 |

Track currency in the spec status block. Each invalidating change increments the count.
A full re-verification pass resets the count to zero and restores `CURRENT`.
