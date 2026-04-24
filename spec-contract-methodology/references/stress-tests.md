# Appendix A — Stress-Test Protocol

Before declaring a spec complete, apply each test vector below. Each vector applies
the spec to a different system shape or reading strategy. If the spec cannot answer
the questions a vector raises, it is incomplete in that dimension.

Run every vector. A spec that passes seven of eight has one uncovered dimension.
That is where the implementation will diverge from the intended behaviour.

---

## Vector 1 — Stateless System

*Apply the spec to a system with no persistent state: a text transformer, a validator, a calculator.*

- Does every section either have content or an explicit inapplicability statement?
- Is statelessness declared as a positive contract in §5 and §10, or just absent?
- Does §6 say "stateless — no lifecycle" rather than being empty?
- Could an implementor accidentally add caching or connection state and still claim to satisfy the spec?

---

## Vector 2 — Stateful Single-Caller Service

*Apply the spec to a service with lifecycle: constructed, used, closed.*

- Are all valid states named?
- Is construction failure handled — can a half-constructed object be used?
- Is `close()` idempotent? Is that stated?
- What happens on every operation in the closed state?
- Can an implementor infer the minimum call sequence to reach a usable state?

---

## Vector 3 — Stateful Shared Service (Multi-Tenant)

*Apply the spec to a service called by multiple independent callers simultaneously.*

- Are per-caller contracts distinguished from global contracts?
- Does §19 address cross-caller interference — can one caller affect another's results?
- Does §13 specify what happens under concurrent access?
- Does §19 state whether one caller can exhaust resources for others?
- Is the absence of isolation stated explicitly if none is provided?

---

## Vector 4 — Streaming or Event-Driven System

*Apply the spec to a system that processes a continuous stream of events.*

- Does §3 state delivery semantics — exactly-once, at-least-once, at-most-once?
- Does the spec define what happens on duplicate events, out-of-order events, gaps?
- Does §18 distinguish observability events from processing events if they overlap?
- Does §11 document why step ordering is correctness-critical vs incidental?
- Does §10 address what happens to partially-processed batches on failure?

---

## Vector 5 — Extension Point

*Apply the spec to something others will implement: a plugin interface, a backend abstraction, a checker protocol.*

- Does §16 specify what "correct implementation" means behaviourally, not just structurally?
- Does it state what the extension MUST NEVER do, with consequences?
- Is resource ownership unambiguous — what does the extension own vs reference?
- Can an extension author implement correctly without reading any existing implementation?
- Does §7 include failure modes an extension can produce, or only ones the core produces?

---

## Vector 6 — Adversarial Reader

*Read the spec as someone trying to follow it literally while producing a subtly wrong implementation.*

- Can any writing rule be satisfied by weakening a claim until trivially true?
- Can "no implementation details" be satisfied while still encoding algorithm choices in behavioural language?
- Are there guarantees that hold for the happy path but leave failure paths unspecified?
- Are there any "always" or "never" claims whose edge-case exceptions are undocumented?
- Can a caller follow the sentinel value conventions literally and still misinterpret a result?
- Is there any place where an omission could be read as either "not applicable" or "forgot to specify"?

---

## Vector 7 — Cross-Section Consistency

*Read sections against each other rather than in isolation.*

- Does every failure mode in §5 appear in §7?
- Does every failure mode in §7 have a propagation path in §17?
- Is resource ownership in §12 consistent with §16?
- Are sentinel values in §9 consistent with the types declared in §5?
- Does §23 contain anything that §21 also claims to defer to implementors?
- Are any assumptions in §22 contradicted by guarantees elsewhere?
- Does §4 cover every data format mentioned in §18, §15, or §3?
- Are §22 caller concurrency assumptions consistent with §13 concurrency guarantees?
- Does §20 explicitly cover data formats, not just operational APIs?

---

## Vector 8 — Embedded Library or Callback-Driven System

*Apply the spec to a library with no process boundary, or a system that invokes caller-supplied callbacks.*

- If the system holds durable state: does §10 specify what condition it is left in after ungraceful shutdown (process kill, crash)?
- Does the spec address whether the system performs automatic recovery on next startup, or requires operator intervention?
- If the system accepts callbacks: does §12 specify when each callback is called, on which thread, re-entrancy rules, and what happens if the callback raises?
- Can a caller implement a callback that is safe under concurrent calls, error conditions, and re-entrant invocation using only the spec?
- If the library is compiled separately from its host: does §20 distinguish API stability from ABI stability?
