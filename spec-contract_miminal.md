Specification Contract — Complete (1-Page)

Write §1-23 per template below.
Run 10-question verification against draft.
Classify gaps as BLOCKING or NON-BLOCKING.
Handoff when zero BLOCKING gaps remain.

---

§1 Purpose

One sentence: what it does, what it doesn't.

§2 Vocabulary

Define every domain term before first use. If a term has multiple possible meanings, state which one applies in this system.

§3 System Boundary

- Inputs: type, valid range, null/empty handling
- Outputs: all possible values, guarantees
- Side effects: what, when, guaranteed or best-effort
- Caller types: if multiple, specify each separately
- Idempotent? Delivery semantics (exactly-once / at-least-once / at-most-once)

§4 Data Contracts

Schema: field, type, required vs optional, unknown-field policy (reject / ignore / pass-through)

§5 Component Contracts

```
Component: [Name]
Stateful or stateless: [state explicitly]
Invariant: [holds always]
Pre: [must be true] → Violation: [what happens]
Accepts: [field]: [type] — [valid range]
Returns: [success]: [condition] / [failure]: [condition]
Guarantees: [Never raises X] / [Always does Y]
Post: [guaranteed after]
Failure behaviour: [failure mode name from §7]: [caller-visible result]
```

§6 Lifecycle

States, transitions, close() idempotent?

§7 Failure Taxonomy

Per failure: name, meaning, category (programming / operational / adversarial / internal), recoverability, representation

§8 Boundary Conditions

Empty, null, min-1, max+1. Exact behaviour, not "handles gracefully."

§9 Sentinel Values

Value → meaning per context. What it does NOT mean.

§10 Atomicity and State on Failure

Atomic? Partial state? Detectable? Retry-safe?

§11 Ordering and Sequencing

Why X before Y — correctness or security reason.

§12 Interaction Contracts

Who owns shared resources. Cross-reference §16.

§13 Concurrency and Re-entrancy

Thread-safe? Re-entrant? Violation consequence.

§14 External Dependencies

Required/optional, absence behaviour at startup and runtime.

§15 Configuration

Name, type, valid range, default, invalid-value behaviour, security-relevant?

§16 Extension Contracts

MUST implement, MAY override, MUST NEVER do. Cross-reference §12.

§17 Error Propagation

Origin → caller-visible form.

§18 Observability Contract

- Event name, when emitted, fields, guaranteed or best-effort
- What is NOT logged (explicit omissions)

§19 Security Properties

What guaranteed, what not, fail-closed or fail-open.

§20 Versioning and Evolution

Stable / evolving / internal per interface.

§21 What Is Not Specified

Explicit list of implementor decisions.

§22 Assumptions
Environmental, caller, operational. For each:
- What is assumed
- What happens if violated
- Violation detectable? If no, flag as HIGH-RISK — the system cannot know this failed.

§23 Performance-Correctness

Timeout, ordering, bounded memory — anything where violation means wrong result, not just slow.

---

Format Rule

MUST / SHOULD / MAY — capitalized, per RFC 2119. Lowercase "should" is a bug.

---

10-Question Verification

1. One sentence: what, out of scope?
2. Domain terms defined before use?
3. Inputs? Null/empty/invalid behaviour?
4. Outputs? Guarantees? Idempotent? Side effects (when, guaranteed)?
5. Public data structures? Invariants?
6. Components? Pre/invariant/post/guarantees per operation?
7. All failures? Categories? Recoverable?
8. Edge behaviour (empty, zero, min-1, max+1)?
9. Concurrency? Thread-safe or caller serializes?
10. What's NOT specified? What's NOT logged? Are any undetectable assumptions flagged HIGH-RISK?

Any question the spec cannot answer = GAP.

---

Gap Classification

- BLOCKING: Implementor would build WRONG behaviour without this.
- NON-BLOCKING: Implementor can INFER correct behaviour from elsewhere.

Only BLOCKING gaps block handoff.

---

Handoff Rule

Implementation Readiness = READY when zero blocking gaps remain.
Non-blocking gaps stay — record them, don't chase them.