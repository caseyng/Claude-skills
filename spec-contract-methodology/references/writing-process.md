# Writing Process

---

## Writing Rules

**Specify what, never how.** Observable behaviour only. Any sentence describing an
algorithm, data structure, library, or internal process is implementation detail.
"Uses NFC normalisation" = how. "Treats canonically equivalent inputs as identical" = what.

**Every guarantee MUST be testable** at the level of specificity that distinguishes
correct from incorrect implementations. A guarantee weakened until trivially testable
is not a guarantee. "The operation completes" specifies nothing. "Returns the stored
value, or null if the key was never written" specifies the contract. If you cannot
write a test that a subtly wrong implementation would fail, the guarantee is not
specific enough.

**MUST / SHOULD / MAY capitalised, per RFC 2119.**
- MUST: required for conformance, violation is a defect
- SHOULD: strongly recommended, deviation requires justification
- MAY: optional, either choice is conformant
- Do not use lowercase "should" when you mean MUST

**Invariants, preconditions, postconditions are three different things.**
- Invariant: holds continuously — never violated, even between operations
- Precondition: MUST be true before an operation is called
- Postcondition: guaranteed true after an operation completes
Confusing them produces specs that appear complete but leave critical behaviour undefined.

**Distinguish failure from defined operational outcome.** "Not found" / "rejected" /
"empty result" are defined outcomes. A failure is when the system cannot produce any
defined outcome. Mixing them produces a failure taxonomy callers cannot reason about cleanly.

**Open sets require a handling contract, not just a declaration.** Declaring a failure
set open tells callers new values may appear. Specify what to do when one does.

**Declare stateless components explicitly.** If a component holds no mutable state,
declare it in §5 and §10. An implementor who sees nothing may add state.

**Assumptions MUST be stated, not inherited.** Every environmental, caller, and
operational assumption MUST appear in §22. An unstated assumption is a failure mode
the implementor cannot anticipate.

**Performance and correctness MUST be separated.** Apply the §23 test: does violation
produce a wrong or undeliverable result? Yes → correctness contract. No → §21.

**Name the reason, not just the rule.** "X MUST precede Y because Y depends on the
output of X" is more useful than "X MUST precede Y." The reason lets an implementor
distinguish correctness-critical ordering from incidental ordering.

**Resolve conflicts before writing.** If existing code and docs disagree, decide which
is correct and write from that decision. Note the resolution. Do not describe both versions.

**Closed sets MUST be declared closed.** If a caller can enumerate all values and
branch exhaustively, say "this is an exhaustive list."

**Specify do-nothing behaviour explicitly.** If the system does nothing in response to
a condition, say so — an implementor who sees no specification will not assume no
behaviour; they will invent one.

**Atomicity MUST be stated, not assumed.** Whether a state-modifying operation is
atomic is a contract.

---

## Before You Write — 25 Questions

Read everything: the existing spec, the code, the tests, the comments. Answer every
question before writing a single section.

Map each answer to its section number. An answer with no section means the spec has a
gap. A section with no answer means the system is underspecified in that area.

1. Where does existing documentation describe *what* vs *how*? Only *what* belongs in the spec.
2. Where do code and docs conflict? Resolve each. Note the resolution.
3. What does the code do that the docs never mention? Each is a potential gap.
4. What do the docs promise that the code doesn't enforce? Each is either a code bug or a doc overclaim.
5. What would an implementor in another language get wrong from existing docs alone?
6. What sentinel values does the code use? All documented? Same value with multiple meanings?
7. What happens at boundary conditions of every operation — empty, null, zero, min, max, one-past-max?
8. What interactions between components carry contracts not visible from either component alone?
9. What does the system do under concurrent access? Is that documented?
10. Which parts of the public interface are stable across versions, and which may change?
11. What external dependencies does the system have? Required vs optional? What happens at startup and runtime if each is absent?
12. For every state-modifying operation: what is the system's state if it fails midway?
13. For every negative result: failure mode or defined operational outcome? Are they distinguished?
14. Are there multiple caller types? Do any have different contracts from the default?
15. Does the system serve multiple callers simultaneously? What isolation guarantees exist?
16. Does the system process streams or queues? What are the delivery semantics?
17. Which components are stateless? Is that declared explicitly?
18. For every failure mode in §7: is its full propagation path documented in §17?
19. For every open set: is the handling contract for unknown values specified?
20. What does the spec take as given — environmental, caller, and operational assumptions? Are any undetectable if violated?
21. Which properties look like performance constraints but affect correctness? Are they in §23 rather than §21?
22. What structured data formats cross public boundaries beyond events and config? Are they in §4?
23. Does the system hold durable state? What is its condition after ungraceful shutdown (process kill, crash, power loss)?
24. Does the system accept callbacks or per-call handlers? Is the full callback contract — timing, thread, re-entrancy, exception handling — specified?
25. After completing the spec, re-read every section: are all domain terms used anywhere present in §2?
26. Are structural decisions declared in §2b? Class hierarchy, inheritance strategy, priority ordering, named patterns — anything that, if left unspecified, would cause an implementor to choose the wrong structure.

---

## When the Spec Is Done

The spec is complete when all of the following hold:

- [ ] Structural decisions declared in §2b — class hierarchy, inheritance strategy, priority ordering, named patterns that constrain implementation
- [ ] An implementor can produce a behaviourally equivalent system without reading the original code
- [ ] Every public contract specified in falsifiable, implementation-distinguishing terms
- [ ] Every precondition, invariant, and postcondition stated separately and unambiguously
- [ ] Every failure mode in §7, referenced from §5, and traced through §17
- [ ] Every failure mode distinguished from defined operational outcomes
- [ ] Every boundary condition explicitly handled
- [ ] Every stateless component declared stateless in §5 and §10
- [ ] Every state-modifying operation specifies atomicity and system state on failure
- [ ] Ungraceful shutdown behaviour specified for any system with durable state
- [ ] Every sentinel value documented; multiple meanings of the same value distinguished
- [ ] Every interaction contract between components documented and consistent with §16
- [ ] Every external dependency documented with required/optional status and absence behaviour
- [ ] Every assumption stated in §22; assumptions whose violation is undetectable are flagged
- [ ] Every property that looks like a performance constraint but affects correctness is classified as a correctness contract or implementation detail
- [ ] Every structured data format at a public boundary in §4
- [ ] Concurrency guarantees, or their explicit absence, stated
- [ ] Delivery semantics declared for any streaming or queue-based interface
- [ ] Multi-caller isolation guarantees stated for any shared service
- [ ] Every extension point has a contract an extension author can implement without reading existing code
- [ ] Every callback point has a contract specifying when called, which thread, re-entrancy, and exception handling
- [ ] Every failure mode has a category; internal invariant violations flagged separately from programming errors
- [ ] Every open set has a handling contract for unknown values
- [ ] Every security-relevant decision explicit, justified, states what is and is not guaranteed
- [ ] Versioning and stability declared for every public interface
- [ ] Spec document itself is versioned; substantive changes recorded
- [ ] Set of things not specified is explicitly stated

**Handoff gate.** Implementation Readiness is READY (zero blocking gaps) and Verification
Currency is CURRENT. Non-blocking gaps MAY remain — record them, do not chase them.
The question for every remaining gap: would a competent implementor build divergent
behaviour without it? If no, it does not block handoff.
