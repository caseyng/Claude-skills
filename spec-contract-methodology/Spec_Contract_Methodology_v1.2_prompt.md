# Specification Contract Methodology
Version: 1.2

## Table of Contents

- [How the Sections Relate](#how-the-sections-relate)
- [The Sections](#the-sections)
  - [§1 Purpose and Scope](#1-purpose-and-scope)
  - [§2 Concepts and Vocabulary](#2-concepts-and-vocabulary)
  - [§3 System Boundary](#3-system-boundary)
  - [§4 Data Contracts](#4-data-contracts)
  - [§5 Component Contracts](#5-component-contracts)
  - [§6 Lifecycle](#6-lifecycle)
  - [§7 Failure Taxonomy](#7-failure-taxonomy)
  - [§8 Boundary Conditions](#8-boundary-conditions)
  - [§9 Sentinel Values and Encoding Conventions](#9-sentinel-values-and-encoding-conventions)
  - [§10 Atomicity and State on Failure](#10-atomicity-and-state-on-failure)
  - [§11 Ordering and Sequencing](#11-ordering-and-sequencing)
  - [§12 Interaction Contracts](#12-interaction-contracts)
  - [§13 Concurrency and Re-entrancy](#13-concurrency-and-re-entrancy)
  - [§14 External Dependencies](#14-external-dependencies)
  - [§15 Configuration](#15-configuration)
  - [§16 Extension Contracts](#16-extension-contracts)
  - [§17 Error Propagation](#17-error-propagation)
  - [§18 Observability Contract](#18-observability-contract)
  - [§19 Security Properties](#19-security-properties)
  - [§20 Versioning and Evolution](#20-versioning-and-evolution)
  - [§21 What Is Not Specified](#21-what-is-not-specified)
  - [§22 Assumptions](#22-assumptions)
  - [§23 Performance Contracts](#23-performance-contracts)
- [Writing Rules](#writing-rules)
- [Before You Write](#before-you-write)
- [When the Spec Is Done](#when-the-spec-is-done)
- [Appendix A — Stress-Test Protocol](#appendix-a--stress-test-protocol)
- [Appendix B — Spec Profile Table](#appendix-b--spec-profile-table)
- [Appendix C — LLM-Executable Verification Pipeline](#appendix-c--llm-executable-verification-pipeline)
- [Deployment Guidance](#deployment-guidance)
- [Future Directions](#future-directions)
- [References](#references)

---

You are writing a specification. Not documenting existing code. Not describing what the current implementation happens to do. Write the definition of what this system must do — in terms precise enough that an implementor in any language could produce a behaviourally equivalent system.

The test: hand this document to a competent engineer who has never seen the codebase. They should be able to produce a behaviourally equivalent system. If they would need to read the original code to resolve any ambiguity, the spec is incomplete.

Read all existing materials — code, existing docs, tests, comments — before writing anything. Where the code and stated intent conflict, resolve the conflict before writing — a spec that describes a known bug is a transcript, not a spec.

---

## How the Sections Relate

Before writing, understand how the sections connect. Several cover related ground from different angles — knowing the division prevents duplication and gaps.


**§4 Data Contracts** documents the shape of structured data that crosses public boundaries. §18 Observability Contract covers event fields specifically. §15 Configuration covers config fields. §4 covers everything else — API schemas, file formats, wire protocols, serialisation conventions.

**§5 Component Contracts** is the master description of each component's interface: what it accepts, what it returns, its invariants, preconditions, and postconditions. It references failure modes by name but does not define them.

**§7 Failure Taxonomy** is the master list of every failure mode. §5 references this list. §17 traces how failures from this list travel through the system. A failure mode MUST appear in §7 before it can be referenced in §5 or §17.

**§8 Boundary Conditions** specifies the exact behaviour at the edges of each component's contract. §5 states the rule; §8 covers the edge cases.

**§10 Atomicity and State on Failure** documents internal system state after a failure. §7 documents what the caller sees (the error). §17 documents how that error travels. §10 documents what the system's internal data state is — which the caller may need to query before retrying.

**§12 Interaction Contracts** and **§16 Extension Contracts** both cover resource ownership at boundaries. §12 covers established components interacting. §16 covers new extensions the spec author cannot fully anticipate. Both MUST document who owns what — cross-reference them explicitly.

**§22 Assumptions** documents what the spec takes as given. §14 External Dependencies documents what the system reaches out to. §15 Configuration documents what operators supply. These are three different categories — do not conflate them.

**§23 Performance Contracts** documents the boundary between performance and correctness. §21 What Is Not Specified defers performance characteristics to implementers. §23 carves out the properties that look like performance constraints but are actually correctness contracts — where violation produces a wrong or undelivered result visible to callers, not merely a slower correct one.


---

## The Sections

Every spec MUST contain all of the following. Do not pad sections. Do not omit them. A section with nothing to say means the system is underspecified in that area — that is a finding, not a reason to skip the section.

For sections that are genuinely not applicable, write one sentence stating why it does not apply and what property of the system makes it inapplicable. That sentence is a contract — it tells a implementor what they may assume.

---

### 1. Purpose and Scope

What problem does this system solve? One paragraph. State it precisely enough that a reader with no prior context can determine what the system does and does not do.

Then: what is explicitly out of scope? Name the things a reasonable person might expect this system to handle that it does not. Without an explicit scope boundary, implementers fill gaps with assumptions.

*Questions to answer:*
- What is the one thing this system does that nothing else does?
- What would break if this system were removed?
- What adjacent problems does this system deliberately not solve?

**Design principles.** If the system has a small set of named principles that constrain every implementation decision — tradeoffs the spec author has resolved in advance — document them here. Each principle should be one sentence stating the rule, not an aspiration. A principle is only useful if it can be applied as a test: "does this decision violate principle X?" If no such principles exist, omit this subsection.

---

### 2. Concepts and Vocabulary

Every domain term the spec uses, defined precisely. Not dictionary definitions — operational definitions. What does this term mean *in this system*, and how does it differ from related terms?

Define terms before using them. If a term is used inconsistently anywhere in the existing documentation, resolve it here and note the resolution.

*Questions to answer:*
- What terms appear in the code or docs that a newcomer would need to look up?
- Are there terms that are used in two different senses anywhere?
- Are there concepts that are implied but never named?

---

### 3. System Boundary

What enters the system, what leaves it, and what the system promises about the transformation between them.

For each input: type, valid range, what happens on invalid input, whether `null`/`None`/empty is valid and what it means.

For each output: type, every possible value, what each value means, what guarantees hold.

For each side effect: what it is, when it occurs, whether it is guaranteed or best-effort.

**Caller types.** If the system has multiple distinct caller types — different clients, roles, or integration points — with different contracts, specify each separately. Do not average them into a single contract that fits none precisely.

**Idempotency.** State explicitly whether calling the system with identical inputs produces identical outputs. If not, state what varies and why. A caller who assumes idempotency where none exists will produce incorrect retry logic.

**Delivery semantics.** If the system processes a stream, queue, or sequence of events, state the delivery guarantee explicitly: exactly-once, at-least-once, or at-most-once. State what the system does when it receives a duplicate, an out-of-order event, or a gap in sequence. These are correctness contracts, not performance details.

*Questions to answer:*
- If I call this system with the same input twice, do I get the same output?
- What can a caller rely on unconditionally?
- What can a caller rely on only under certain conditions?
- What inputs are rejected at the boundary vs. passed through and handled internally?
- Are there multiple caller types? Do they have different contracts?

---

### 4. Data Contracts

The shape of every structured data format that crosses a public boundary.

§18 covers event fields. §15 covers configuration fields. This section covers everything else: API request and response schemas, file formats the system reads or writes, wire protocols, serialisation conventions, interchange formats.

For each data format at a public boundary:
- Name and purpose
- Schema — every field: name, type, required vs optional, valid values, what absent means vs null
- Unknown fields — what the system does when it receives a field not in this schema (reject, ignore, pass through). What a caller MUST do when it receives a field not in this schema.
- Version negotiation — if the format can change across versions, document here how producer and consumer agree on which version to use. The stability and breaking-change rules that govern this are in §20.
- Encoding — character encoding, byte order, compression, any platform-dependent serialisation choices that affect portability

**A data format at a public boundary is a contract.** If the schema changes in a breaking way, existing producers or consumers break. The stability and versioning rules from §20 apply to data contracts as much as to operational contracts.

---

### 5. Component Contracts

Every component that has a public interface, specified as a contract — not an implementation.

**Component hierarchy.** If the system has multiple components, open this section with a diagram or structured list showing how components are nested and owned. This gives a implementor the map before the individual contracts. A reader who cannot see the overall structure has to reconstruct it from the contracts — that is extra work with room for error. The hierarchy here is not a contract; it is orientation. Contracts follow.

**Distinguish preconditions, postconditions, and invariants.**

- *Precondition:* what MUST be true before an operation is called. Document what the component does when called with a violated precondition — raises a specific error, returns a specific value, or has undefined behaviour. Do not leave this implicit.
- *Invariant:* a condition that holds continuously throughout the lifetime of a component — never violated, even transiently between operations.
- *Postcondition:* a condition guaranteed to hold after a specific operation completes.

**Stateless components.** If a component holds no mutable state, declare this explicitly: "this component is stateless — it holds no mutable state between calls." This is a contract. A implementor who does not know the component is stateless may add caching or connection state that changes observable behaviour.

**Failure behaviour references §7.** The contract template includes a failure behaviour field. Each entry names a failure mode and describes the caller-visible result. The failure mode itself — its definition, category, recoverability — is defined in §7, not here.

The format for a contract:

```
Component: [name]
Purpose:   [one sentence — what it does, not how]

Stateful or stateless: [state explicitly]

Invariants:
  - [condition that holds for the entire lifetime of this component]
  - OR: "None — this component is stateless."

Preconditions for [operation]:
  - [what must be true before this operation is called]
  - Violation behaviour: [what the component does if precondition is not met]

Accepts:
  [field]: [type] — [valid range, what null means if nullable]

Returns:
  [success case]: [condition — e.g. "on success", "if key exists"]  ← MUST always include the success return explicitly
  [other values]: [condition under which this value is returned]

Postconditions:
  - After [operation]: [condition that holds]
  - [field] is null if and only if [condition]

Guarantees:
  - Never raises [error type] — [why this matters to callers]
  - Always [unconditional behaviour]

Failure behaviour:
  [failure mode name, defined in §7]: [what is returned to the caller, what is NOT done]
```

Do not describe implementation. "Uses a regex" is implementation. "Returns false if input matches any configured pattern" is contract.

---

### 6. Lifecycle

Every stateful component's valid states and valid transitions between them.

If a component is stateless (declared in §5), write: "Stateless — no lifecycle to specify." Do not pad.

For each stateful component:
- What state it is in immediately after successful construction
- What state it is in if construction fails partway through — valid partial state, or invalid and not to be used?
- What operations are valid in each state
- What calling an operation in an invalid state does — raises, returns error, silently ignores. Each is a different contract.
- What state it is in after `close()` or equivalent
- Whether it can be reused after `close()`
- Whether `close()` is idempotent — what happens on the second call

Use a state diagram if the transitions are non-trivial. Either way: every state and every valid transition MUST be named.

*Questions to answer:*
- What happens if you call `process()` before `init()`?
- What happens if you call `process()` after `close()`?
- What happens if you call `close()` twice?
- What happens if construction fails partway through?
- What is the minimum sequence of calls to bring the component into a usable state?

---

### 7. Failure Taxonomy

This is the master list of every failure mode the system can produce. §5 references this list. §17 traces how these failures travel. A failure MUST be defined here before it can be referenced elsewhere.

**First: distinguish failure from defined operational outcomes.** Not every negative result is a failure. "Key not found", "input rejected", "empty result" are defined outcomes. A failure is a condition that prevented the system from producing any defined outcome at all. Document defined outcomes in §5. Document failures here. If the distinction is unclear, resolve it explicitly and state the resolution.

For each failure mode:
- Name — exact string or constant if it appears in output or error messages
- Meaning — what condition caused it, precisely
- Category — one of four:
  - *Programming error:* the caller did something wrong — wrong argument type, called after close, violated a precondition
  - *Operational failure:* an external dependency was unavailable, timed out, or returned an unexpected response
  - *Adversarial signal:* input was designed to bypass, exploit, or destabilise the system
  - *Internal invariant violation:* a condition that should be impossible given a correct implementation, but which the system detects and fails on defensively. This category means "a bug in the system was detected at runtime." Callers cannot recover from this by changing their behaviour — only a fix to the system resolves it. These are the highest-severity failures and MUST NOT be conflated with programming errors.
- Recoverability — retry immediately, retry after backoff, or permanent for this input
- Information carried — what data accompanies this failure; which fields are guaranteed present vs. conditionally present
- Representation — returned as a structured value, raised as an exception, or both. If both are possible for the same condition, state when each applies.
- What is NOT done — what the system explicitly does not do when this failure occurs

**Declare whether the set is closed.** If callers MAY safely enumerate all values and branch exhaustively, say so. If future values MAY be added, say so.

**Open sets require a handling contract.** If the set is open, specify what the caller MUST do when receiving an unknown value: treat as fatal, treat as the most conservative known failure, ignore, or something else. An open set without a handling contract is not a complete specification.

---

### 8. Boundary Conditions

For every operation: what happens at the exact edges of valid input.

§5's `Accepts` field covers what is valid. This section covers what happens at the exact edges of valid — minimum, maximum, empty, null, and degenerate inputs. Do not duplicate the validity rule here; document the edge behaviour.

For each operation, explicitly specify behaviour for:
- Empty input (empty string, empty list, zero-length collection)
- Null/None input where the type is nullable
- The minimum valid value and one below it
- The maximum valid value and one above it
- Input that is structurally valid but semantically degenerate (e.g. a list with one element where multiple are expected, a range where min equals max)
- Input that was valid in a previous call but the system's state has since changed

"Handles gracefully" is not a specification. "Returns an empty result" is. "Raises a configuration error identifying the field by name" is. State the exact behaviour.

---

### 9. Sentinel Values and Encoding Conventions

Every place the system uses `null`/`None`/zero/empty string/special values as semantic signals — documented explicitly.

§5 documents the contract for each field. §8 documents edge behaviour. This section documents values that carry meaning as signals — particularly where the same value means different things in different contexts, and what the caller MUST do when they encounter one.

For each:
- Where it appears — field name, return value, parameter
- What it means in that specific context
- What it does NOT mean — if the same value means different things in different contexts, make each distinction explicit
- What a caller MUST do when they encounter it

The same value with multiple meanings in different contexts, none of them documented, will produce one bug per meaning in a implementation.

---

### 10. Atomicity and State on Failure

For any operation that modifies state: what is the state of the system if the operation fails midway?

If the component is stateless, write: "Stateless — no mutable state, section not applicable." That is a contract.

For each state-modifying operation:
- Whether the operation is atomic — either completes fully or has no effect
- If not atomic: what partial state is possible, and whether the system can recover from it
- Whether the previous state is preserved on failure (rollback), undefined (corrupt), or deterministically one of a known set of states
- Whether the caller can detect that a partial write occurred, and how
- Whether a retry of a failed operation can cause duplicate effects — if so, what the duplicate looks like and how to detect it

This section is distinct from §7 (what error the caller sees), §17 (how errors travel), and §6 (what lifecycle state the component is in). This section documents internal data state after failure — which the caller may need to query before retrying safely.

**Ungraceful shutdown.** If the system holds durable state and the process is killed without calling close() — by signal, crash, or power loss — state explicitly what condition the durable state is left in. Is it always recoverable? Is it detectable as corrupt? Does the system perform recovery automatically on next startup, or does it require operator intervention? A spec that does not answer these questions describes a system that cannot be safely deployed where crashes are possible.

---

### 11. Ordering and Sequencing

If the system has a pipeline, chain, or sequence of operations: document the ordering constraints explicitly.

For each step:
- What it reads from the context (inputs from prior steps)
- What it produces (outputs for subsequent steps)
- What invariants must hold when it runs
- Why it MUST precede or follow adjacent steps — state the correctness or security reason, not just the order

"The current implementation does X before Y" is not a sequencing constraint. "X MUST precede Y because Y depends on the output of X" is. "X MUST precede Y because allowing Y to run before X creates security gap Z" is.

If the order can be changed without correctness or security impact, say so explicitly. A implementor who sees an ordering without a reason will assume the reason exists.

---

### 12. Interaction Contracts

Where two components interact, new contracts emerge that are not visible from either component's individual specification. These MUST be documented here.

For each significant interaction:
- Which components are involved
- Who initiates (caller) and who responds (callee)
- What the caller guarantees before calling (preconditions the callee MAY assume)
- What the callee guarantees after returning (postconditions the caller MAY rely on)
- Who owns shared resources — who constructs them, who closes them, who MUST NOT close them
- What happens if the interaction fails midway — is either component left in a defined state?

Resource ownership documented here MUST be consistent with §16 Extension Contracts. Cross-reference explicitly.

*Common interaction contracts that are frequently undocumented:*
- Resource ownership at component boundaries
- Callback or hook contracts — see below
- Shared mutable state access patterns (who reads, who writes, in what order)
- Event sequencing guarantees between a producer and consumer

**Callback contracts.** If the system accepts callbacks or event handlers supplied by the caller — whether at construction or per-call — specify for each callback point:
- When the callback will be called — the exact condition, not "when relevant"
- Which thread or execution context it will be called on
- Whether the callback may be called re-entrantly (from within a call the caller is already in)
- What happens if the callback raises an error — propagated, suppressed, logged, undefined behaviour
- Whether the callback will be called again after it raises
- Whether the callback may call back into the system, and if so which operations are safe

Re-entrancy constraints for callbacks belong in §13. Document the full concurrency contract there; reference it here.

A callback implemented without these constraints will work under light load and fail unpredictably under concurrency or error conditions.

---

### 13. Concurrency and Re-entrancy

State explicitly what guarantees the system makes under concurrent access. "Thread-safe" is not a specification. State the precise guarantees.

For each component or operation:
- Whether it is safe to call from multiple threads simultaneously
- Whether it is safe to call re-entrantly (from within a callback invoked by the same component)
- What the consequences of violating these constraints are — data corruption, deadlock, undefined behaviour, or a specific error
- Whether any internal synchronisation is provided, and if so what it covers and what it does not

If the system does not provide concurrency guarantees and delegates responsibility to the caller, say so explicitly. If the spec says nothing about concurrent access, a implementor will invent a behaviour — state what concurrent callers may expect, even if the answer is undefined behaviour.

**Cross-reference §22.** Caller assumptions about call patterns (single-threaded, no concurrent calls) belong in §22 Assumptions. §13 documents what the system itself guarantees or enforces. They MUST be consistent: if §22 assumes single-threaded callers, §13 MUST state that concurrent calls produce undefined behaviour (or specify the consequence). If §13 makes the system safe for concurrent calls, §22 MUST NOT assume single-threaded access.

---

### 14. External Dependencies

Every resource the system requires that exists outside its own boundary.

For each dependency:
- Name and what it provides
- Whether it is required (system cannot start without it) or optional (system degrades without it)
- What "optional" means precisely — which capabilities are unavailable, which errors are produced, whether degradation is signalled to the caller
- What happens at startup if absent — fails with a specific error, starts in a degraded mode, silently continues
- What happens at runtime if it becomes unavailable — fails the current operation, queues, retries, or fails open
- Whether it is verified at startup (eager) or on first use (lazy), and why

*Examples:* libraries that must be installed, network endpoints, files on disk, operating system features, environment variables.

Distinct from §15 Configuration (operator-supplied values), §3 System Boundary (caller-supplied inputs), and §22 Assumptions (properties taken as given). External dependencies are things the system actively reaches out to.

---

### 15. Configuration

Every configuration field the system accepts.

For each field:
- Name
- Type
- Valid range or valid values — exhaustive if the set is small, bounds if continuous
- What `null`/`None`/absent means — and whether null and absent mean the same thing
- Default value and why that default was chosen
- What happens if an invalid value is supplied — at construction time or at runtime
- Whether the field affects security posture — mark explicitly, a misconfigured security field may silently weaken protection
- Whether the field can be changed after construction, and if so what the effect is on in-flight operations

Do not describe defaults as "sensible" or "safe" without defining what those words mean in this context. A default safe for one deployment may be unsafe for another.

---

### 16. Extension Contracts

If the system is designed to be extended — new implementations, new plugins, new handlers — specify what an extension MUST provide and what it MUST preserve.

For each extension point:

**MUST implement:** operations with no universal default. State what "correct implementation" means — not just the signature but the full behavioural contract the system will rely on.

**MAY override:** operations with a sensible base behaviour. State what the base behaviour is so an extension author knows what they are deviating from.

**MUST NEVER do:** invariants the extension must preserve. State the consequence of violating each one — data corruption, double-close, security bypass, deadlock.

**Owns vs. references:** which resources the extension constructs and owns (MUST close) vs. which are injected (MUST NOT close). Cross-reference §12 — resource ownership MUST be consistent between both sections.

**Registration:** how an extension registers itself. Whether existing files must be edited. Whether registration changes security posture.

---

### 17. Error Propagation

For each failure mode in §7: document how it travels through the system from origin to the caller's boundary.

For each failure mode:
- Where it originates — which component, which operation
- Whether it is absorbed at origin — logged, converted to a structured result, suppressed with stated rationale
- Whether it is transformed as it propagates — wrapped, annotated, reclassified
- Where it surfaces to the caller — which boundary, in what form
- Whether the system remains in a defined state after propagation

A caller who does not know that a failure in component C surfaces as error E at the top-level boundary cannot write correct error handling. This is not derivable from §5 alone — §5 documents each component's individual contract, not how failures travel across them.

*Questions to answer:*
- If a dependency fails mid-operation, what does the top-level caller see?
- If an extension raises an unexpected error, does it propagate or get converted?
- Are any errors silently swallowed? If so, what signal if any does the caller receive, and what is the rationale?

**Caller-side exception types.** If the spec defines exception types that callers MAY raise from response values — as a control-flow convenience rather than exceptions raised by the system itself — document them explicitly and separately from internal propagation. For each type: the condition it represents, which failure modes or outcomes it maps to, and what it is distinct from. State clearly that these types are never raised by the system internally. The distinction matters: a caller who conflates a system-raised exception with a caller-convenience type cannot write correct error handling.

---

### 18. Observability Contract

If the system emits events, logs, metrics, or audit records: specify them as a contract.

For each event type:
- Name
- When it is emitted — the condition, not a line number
- Fields carried — names, types, meanings, which are always present vs. conditionally present
- Whether emission is guaranteed or best-effort
- Whether the event is emitted before or after the operation it describes — matters for crash recovery and audit completeness
- Whether fields can be null and under what condition
- Whether the event can be emitted more than once for a single logical operation

If the system is event-driven and observability events overlap with processing events, document which are internal processing records and which are external observability signals, and whether they carry different guarantees.

If events form a sequence that callers can reconstruct state from, document the sequencing guarantees.

**What is not logged.** For security-sensitive or privacy-sensitive systems: explicitly state what the system deliberately omits from its observability output. Omissions that are deliberate design decisions — not oversights — MUST be named. An operator who does not know the system omits certain data cannot reason about what the audit record does and does not prove. A implementor who sees no guidance will make their own omission decisions.

**Schema versioning.** If the event schema can change across system versions, specify how consumers determine which schema applies to a given record. A consumer handling records from multiple versions of the system cannot rely on field presence inference — they need a declared mechanism. State which record or field carries the version signal, and what a consumer MUST do when handling records whose version they do not recognise.

---

### 19. Security Properties

What the system guarantees under adversarial input. Be precise — "secure" is not a property.

**Single-caller adversarial input.** For each security-relevant decision:
- What the system guarantees — precisely enough to be falsified by a test
- What the system explicitly does not guarantee
- Whether the decision is fail-closed or fail-open, and why
- What an adversary who knows the full specification can do, and what they cannot do

**Multi-caller and resource isolation.** If the system serves multiple callers, document what isolation guarantees exist between them: whether one caller's input can affect another caller's output, whether one caller can exhaust resources that degrade service for others, and what the system does when resource limits are reached. Absence of isolation is a security property and MUST be stated, not assumed.

Document every tradeoff between availability and security. They MUST be explicit so operators can make informed choices.

---

### 20. Versioning and Evolution

Which parts of this spec are stable contracts callers MAY depend on across versions, and which may change. This includes operational interfaces (operations, parameters, return values), data formats (schemas, field names, encoding), event contracts (§18), and configuration fields (§15) — all are versioned contracts subject to the rules below.

For each part of the public interface:
- Stability level: stable (no change without major version increment), evolving (MAY change with notice), internal (not a public contract)
- What compatibility guarantees exist — does adding a field to a response break existing callers?
- What constitutes a breaking change
- How breaking changes are communicated

**Spec maintenance.** The spec is a versioned document. Each substantive change MUST be recorded: what changed, why, and what the previous contract was. A reader picking up the spec must be able to tell whether it is current. Mark sections that are provisional so implementers know where the contract is still being established.

---

### 21. What Is Not Specified

An explicit list of decisions left to implementers.

Things the spec intentionally does not prescribe: internal data structures, algorithm choices, logging verbosity and format, concurrency model, memory layout, retry strategies. An implementer MAY choose any approach that satisfies the contracts above.

Without this section, a implementor cannot distinguish "doesn't matter" from "the spec forgot."

---

### 22. Assumptions

Every condition the spec takes as given without proof.

A implementation that satisfies every written contract may still behave differently if these assumptions are violated in a different environment. An assumption not stated in the spec is a failure mode the implementor cannot anticipate.

Classify each assumption by type:

**Environmental assumptions** — properties of the runtime or platform the system depends on:
- Arithmetic (e.g. integer overflow wraps, floats conform to IEEE 754)
- Clock (e.g. the system clock is monotonic, never goes backwards)
- Filesystem (e.g. writes to a single file are atomic up to page size, the filesystem does not silently corrupt)
- Memory (e.g. the runtime manages memory, there is no manual deallocation)
- Character encoding (e.g. strings are UTF-8, the platform default is not assumed)

**Caller assumptions** — what the system assumes about the entity calling it:
- Trust level (e.g. the caller is authenticated, the caller is cooperative, the caller is adversarial)
- Call pattern (e.g. the caller calls from a single thread, the caller does not call after close)
- Input validity (e.g. the caller validates inputs before calling, or the system validates them itself — make this explicit)

**Operational assumptions** — what the system assumes about its deployment context:
- Network (e.g. the network is reachable at startup, latency is bounded)
- Configuration (e.g. the config file exists and is readable, secrets are present)
- Time (e.g. the wall clock is approximately correct, NTP is running)

For each assumption: state it explicitly, state what happens if it is violated, and state whether the violation is detectable. An assumption whose violation is undetectable is a failure mode the implementor cannot anticipate and MUST be flagged.

---

### 23. Performance Contracts

The boundary between performance properties and correctness contracts.

**The test:** if the property is violated, does the caller receive an incorrect result, a missing guarantee, or an undeliverable contract? If yes, it is a correctness contract and belongs here. If the caller receives a correct result more slowly, it is a performance characteristic and belongs in §21.

*Examples of correctness contracts that appear to be performance:*
- Timeout: "the operation MUST return within N seconds or produce a timeout failure" — this is correctness. The caller's retry logic depends on it.
- Ordering: "events MUST be delivered in the order they were submitted" — this is correctness. A caller building state from events depends on it. Note: delivery semantics (exactly-once, at-least-once, at-most-once) belong in §3 System Boundary, not here. Ordering constraints that derive from those semantics belong here.
- Bounded memory: "the system MUST NOT buffer more than N items" — if exceeding the bound causes data loss, this is correctness.

*Examples of genuine performance characteristics (belong in §21, not here):*
- Throughput targets
- Latency percentiles (except when they define SLAs the system guarantees)
- CPU and memory usage

For each property that looks like a performance constraint but affects correctness:
- State the property precisely
- State what the caller observes if it is violated
- State whether the system enforces the bound internally or delegates enforcement to the caller

---

## Writing Rules

**Specify what, never how.** The spec defines observable behaviour — what the system accepts, what it produces, what it guarantees. Any sentence describing an algorithm, data structure, library, or internal process is implementation detail and does not belong in the spec. "Uses NFC normalisation" is how. "Treats canonically equivalent inputs as identical" is what.

**Every guarantee MUST be testable at the level of specificity that distinguishes correct from incorrect implementations.** A guarantee weakened until trivially testable is not a guarantee. "The operation completes" specifies nothing. "The operation returns the stored value, or null if the key was never written" specifies the contract. If you cannot write a test that a subtly wrong implementation would fail, the guarantee is not specific enough.

**Use MUST, SHOULD, MAY consistently and capitalised, per RFC 2119.** MUST: required for conformance, violation is a defect. SHOULD: strongly recommended, deviation requires justification. MAY: optional, either choice is conformant. Do not use lowercase "should" when you mean MUST.

**Invariants, preconditions, and postconditions are three different things.** An invariant holds continuously — never violated, even between operations. A precondition MUST be true before an operation is called. A postcondition is guaranteed true after an operation completes. Confusing them produces specs that appear complete but leave critical behaviour undefined.

**Distinguish failure from defined operational outcome.** "Not found", "rejected", "empty result" are defined outcomes. A failure is when the system cannot produce any defined outcome. Mixing them produces a failure taxonomy callers cannot reason about cleanly.

**Open sets require a handling contract, not just a declaration.** Declaring a failure set open tells callers new values may appear. It does not tell them what to do when one does. Specify the handling contract.

**Declare stateless components explicitly — do not leave it implied.** If a component holds no mutable state, declare it explicitly in §5 and §10. A implementor who sees nothing may add state.

**Assumptions MUST be stated, not inherited.** Every environmental, caller, and operational assumption the spec depends on MUST appear in §22. An assumption not stated in the spec is a failure mode the implementor cannot anticipate.

**Performance and correctness MUST be separated.** For every property that looks like a performance constraint but affects correctness, apply the test in §23: does violation produce a wrong or undeliverable result for callers? If yes, it is a correctness contract. If no, it is a performance characteristic and belongs in §21.

**Name the reason, not just the rule.** "X MUST precede Y because Y depends on the output of X" is more useful than "X MUST precede Y." The reason lets a implementor distinguish a correctness-critical ordering from an incidental one.

**Resolve conflicts before writing.** If existing code and docs disagree, decide which is correct and write from that decision. Note the resolution. Do not describe both versions.

**Closed sets MUST be declared closed.** If a caller can enumerate all values and branch exhaustively, say "this is an exhaustive list."

**Specify do-nothing behaviour explicitly.** If the system does nothing in response to a condition, say so — a implementor who sees no specification will not assume no behaviour, they will invent one.

**Atomicity MUST be stated, not assumed.** Whether a state-modifying operation is atomic is a contract.

---

## Before You Write

Read everything: the existing spec, the code, the tests, the comments. Answer every question below before writing a single section.

1. Where does existing documentation describe *what* vs *how*? Only *what* belongs in the spec.
2. Where do code and docs conflict? Resolve each conflict. Note the resolution.
3. What does the code do that the docs never mention? Each is a potential gap.
4. What do the docs promise that the code doesn't enforce? Each is either a code bug or a doc overclaim.
5. What would a implementor in another language get wrong from the existing docs alone?
6. What sentinel values does the code use? Are all documented? Is the same value used with multiple meanings?
7. What happens at boundary conditions of every operation — empty, null, zero, min, max, one-past-max?
8. What interactions between components carry contracts not visible from either component alone?
9. What does the system do under concurrent access? Is that documented?
10. Which parts of the public interface are stable across versions, and which may change?
11. What external dependencies does the system have? Required vs optional? What happens at startup and runtime if each is absent?
12. For every state-modifying operation: what is the system's state if it fails midway?
13. For every negative result: failure mode or defined operational outcome? Are they distinguished?
14. Are there multiple caller types? Do any have different contracts from the default?
15. Does the system serve multiple callers simultaneously? What isolation guarantees exist between them?
16. Does the system process streams or queues? What are the delivery semantics?
17. Which components are stateless? Is that declared explicitly?
18. For every failure mode in §7: is its full propagation path documented in §17?
19. For every open set: is the handling contract for unknown values specified?
20. What does the spec take as given — environmental, caller, and operational assumptions? Are any of them undetectable if violated?
21. Which properties look like performance constraints but affect correctness? Are they in §23 rather than §21?
22. What structured data formats cross public boundaries beyond events and config? Are they in §4?
23. Does the system hold durable state? What is its condition after an ungraceful shutdown (process kill, crash, power loss)?
24. Does the system accept callbacks or per-call handlers from callers? Is the full callback contract — timing, thread, re-entrancy, exception handling — specified?
25. After completing the spec, re-read every section: are all domain terms used anywhere present in §2?

---

## When the Spec Is Done

The spec is complete when:

- A implementor can produce a behaviourally equivalent system without reading the original code
- Every public contract is specified in falsifiable, implementation-distinguishing terms
- Every precondition, invariant, and postcondition is stated separately and unambiguously
- Every failure mode is in §7, referenced from §5, and traced through §17
- Every failure mode is distinguished from defined operational outcomes
- Every boundary condition is explicitly handled
- Every stateless component is declared stateless in §5 and §10
- Every state-modifying operation specifies atomicity and system state on failure
- Ungraceful shutdown behaviour is specified for any system with durable state
- Every sentinel value is documented; multiple meanings of the same value are distinguished
- Every interaction contract between components is documented and consistent with §16
- Every external dependency is documented with required/optional status and absence behaviour
- Every assumption is stated in §22; assumptions whose violation is undetectable are flagged
- Every property that looks like a performance constraint but affects correctness is classified as a correctness contract or implementation detail
- Every structured data format at a public boundary is in §4
- Concurrency guarantees, or their explicit absence, are stated
- Delivery semantics are declared for any streaming or queue-based interface
- Multi-caller isolation guarantees are stated for any shared service
- Every extension point has a contract an extension author can implement without reading existing code
- Every callback point has a contract specifying when called, which thread, re-entrancy, and exception handling
- Every failure mode has a category; internal invariant violations are flagged separately from programming errors
- Every open set has a handling contract for unknown values
- Every security-relevant decision is explicit, justified, and states what is and is not guaranteed
- Versioning and stability are declared for every public interface
- The spec document itself is versioned and substantive changes are recorded
- The set of things not specified is explicitly stated

**Final verification pass.** For every line of code that implements observable behaviour: find where the spec covers it. If you cannot find it, the spec is incomplete. Do not stop until every observable behaviour is covered by the spec.

---

## Appendix A — Stress-Test Protocol

Before declaring a spec complete, apply each test vector below. Each vector applies the spec to a different system shape or reading strategy. If the spec cannot answer the questions a vector raises, it is incomplete in that dimension.

Run every vector. A spec that passes seven of eight has one uncovered dimension. That is where the implementation will diverge from the intended behaviour.

---

### Vector 1 — Stateless System

*Apply the spec to a system with no persistent state: a text transformer, a validator, a calculator.*

- Does every section either have content or an explicit inapplicability statement?
- Is statelesness declared as a positive contract in §5 and §10, or just absent?
- Does §6 say "stateless — no lifecycle" rather than being empty?
- Could a implementor accidentally add caching or connection state and still claim to satisfy the spec?

---

### Vector 2 — Stateful Single-Caller Service

*Apply the spec to a service with lifecycle: constructed, used, closed.*

- Are all valid states named?
- Is construction failure handled — can a half-constructed object be used?
- Is close() idempotent? Is that stated?
- What happens on every operation in the closed state?
- Can a implementor infer the minimum call sequence to reach a usable state?

---

### Vector 3 — Stateful Shared Service (Multi-Tenant)

*Apply the spec to a service called by multiple independent callers simultaneously.*

- Are per-caller contracts distinguished from global contracts?
- Does §19 address cross-caller interference — can one caller affect another's results?
- Does §13 specify what happens under concurrent access?
- Does §19 state whether one caller can exhaust resources for others?
- Is the absence of isolation stated explicitly if none is provided?

---

### Vector 4 — Streaming or Event-Driven System

*Apply the spec to a system that processes a continuous stream of events.*

- Does §3 state delivery semantics — exactly-once, at-least-once, at-most-once?
- Does the spec define what happens on duplicate events, out-of-order events, gaps?
- Does §18 distinguish observability events from processing events if they overlap?
- Does §11 document why step ordering is correctness-critical vs incidental?
- Does §10 address what happens to partially-processed batches on failure?

---

### Vector 5 — Extension Point

*Apply the spec to something others will implement: a plugin interface, a backend abstraction, a checker protocol.*

- Does §16 specify what "correct implementation" means behaviourally, not just structurally?
- Does it state what the extension MUST NEVER do, with consequences?
- Is resource ownership unambiguous — what does the extension own vs reference?
- Can an extension author implement correctly without reading any existing implementation?
- Does §7 include failure modes an extension can produce, or only ones the core produces?

---

### Vector 6 — Adversarial Reader

*Read the spec as someone trying to follow it literally while producing a subtly wrong implementation.*

- Can any writing rule be satisfied by weakening a claim until it is trivially true?
- Can "no implementation details" be satisfied while still encoding algorithm choices in behavioural language?
- Are there guarantees that hold for the happy path but leave failure paths unspecified?
- Are there any "always" or "never" claims whose edge-case exceptions are undocumented?
- Can a caller follow the sentinel value conventions literally and still misinterpret a result?
- Is there any place where an omission in the spec could be read as either "not applicable" or "forgot to specify"?

---

### Vector 7 — Cross-Section Consistency

*Read sections against each other rather than in isolation.*

- Does every failure mode in §5 appear in §7?
- Does every failure mode in §7 have a propagation path in §17?
- Is resource ownership in §12 consistent with §16?
- Are sentinel values in §9 consistent with the types declared in §5?
- Does §23 contain anything that §21 also claims to defer to implementers?
- Are any assumptions in §22 contradicted by guarantees elsewhere?
- Does §4 cover every data format mentioned in §18, §15, or §3?
- Are §22 caller concurrency assumptions consistent with §13 concurrency guarantees?
- Does §20 explicitly cover data formats, not just operational APIs?

---

### Vector 8 — Embedded Library or Callback-Driven System

*Apply the spec to a library with no process boundary, or a system that invokes caller-supplied callbacks.*

- If the system holds durable state: does §10 specify what condition it is left in after ungraceful shutdown (process kill, crash)?
- Does the spec address whether the system performs automatic recovery on next startup, or requires operator intervention?
- If the system accepts callbacks: does §12 specify when each callback is called, on which thread, re-entrancy rules, and what happens if the callback raises?
- Can a caller implement a callback that is safe under concurrent calls, error conditions, and re-entrant invocation using only the spec?
- If the library is compiled separately from its host: does §20 distinguish API stability from ABI stability?

---

## Appendix B — Spec Profile Table

Before writing, identify which system shapes apply to the project. Most systems span multiple shapes. For each applicable shape, the table identifies which sections are required, conditional, or inapplicable.

**How to use:** identify the applicable shapes in the left column. For each shape, check which sections are marked R (required), C (conditional — required if the system has the stated property), or N (declare inapplicable with one sentence). The union of required sections across all applicable shapes is the minimum complete spec for this project.

| Section | Stateless Transform | Stateful Single-Caller | Stateful Shared Service | Streaming / Event-Driven | Extension Point | Security-Sensitive |
|---|---|---|---|---|---|---|
| §1 Purpose and Scope | R | R | R | R | R | R |
| §2 Concepts and Vocabulary | R | R | R | R | R | R |
| §3 System Boundary | R | R | R | R | R | R |
| §4 Data Contracts | C (if structured I/O) | C (if structured I/O) | C (if structured I/O) | R | C (if structured I/O) | R |
| §5 Component Contracts | R | R | R | R | R | R |
| §6 Lifecycle | N (declare stateless) | R | R | R | R | R |
| §7 Failure Taxonomy | R | R | R | R | R | R |
| §8 Boundary Conditions | R | R | R | R | R | R |
| §9 Sentinel Values | R | R | R | R | R | R |
| §10 Atomicity and State on Failure | N (declare stateless) | R | R | R | C (if stateful) | R |
| §11 Ordering and Sequencing | N | C (if pipeline) | C (if pipeline) | R | C (if pipeline) | R |
| §12 Interaction Contracts | C (if >1 component) | C (if >1 component) | R | R | R | R |
| §13 Concurrency and Re-entrancy | R | R | R | R | R | R |
| §14 External Dependencies | C (if any) | C (if any) | C (if any) | C (if any) | C (if any) | R |
| §15 Configuration | C (if configurable) | C (if configurable) | C (if configurable) | C (if configurable) | C (if configurable) | R |
| §16 Extension Contracts | N | N | N | N | R | C (if extensible) |
| §17 Error Propagation | R | R | R | R | R | R |
| §18 Observability Contract | C (if emits) | C (if emits) | C (if emits) | R | C (if emits) | R |
| §19 Security Properties | C (if handles untrusted input) | C (if security-relevant) | R | C (if security-relevant) | R | R |
| §20 Versioning and Evolution | R | R | R | R | R | R |
| §21 What Is Not Specified | R | R | R | R | R | R |
| §22 Assumptions | R | R | R | R | R | R |
| §23 Performance Contracts | N | C (if timeouts or ordering) | C (if timeouts or ordering) | R | C (if bounded) | R |

**Key:** R = Required. C = Conditional (required if the stated property applies). N = Not applicable (write one sentence declaring it inapplicable and why).

**Note on the Extension Point shape.** When writing a spec for an extension point — an interface others will implement — the spec author is specifying a contract for unknown future implementers. §16 is required and is typically the most detailed section. The stress-test Vector 5 is the most important vector to run for this shape.

**Note on the Security-Sensitive shape.** Any system that handles untrusted input, makes access control decisions, or sits on a security boundary should be treated as security-sensitive regardless of its primary shape. §19, §22 (caller trust assumptions), and §14 (dependency availability on auth failure) are all required.

---

---

## Appendix C — LLM-Executable Verification Pipeline

This appendix formalises the five-step structural verification methodology (see `spec-verification.md`) for execution by an LLM. It covers three things the human-readable version does not: the pipeline shape, the re-grounding mechanism, and the gap report format.

The verification methodology itself — the five steps, the fixed cross-reference matrix, the invariant set structure — is in `spec-verification.md`. Read that first. This appendix describes how to execute it reliably as a machine process.

---

### The Pipeline Shape

Spec work involves two distinct LLM phases. They have different failure modes. They require different design.

**Generation phase.** The LLM fills all 23 sections. Where the input is ambiguous, the LLM makes a provisional decision and flags it explicitly. The spec remains in `DRAFT` state until all flags are resolved.

**Verification phase.** The LLM runs the five-step structural verification against the completed draft. Each step produces an externalised artefact. Each artefact is fed back in as input to the next step. The verification produces a gap report. The spec remains in `UNVERIFIED` state until the gap report is empty.

Human review happens once — after both phases are complete. The human sees the draft spec and the gap report together. That pairing is the design. A gap report without a spec is abstract. A spec without a gap report asks the human to do verification manually.

```
Input (idea / codebase / existing docs)
  → [Generation phase]
      LLM fills §1–§23
      Ambiguities → provisional answer + FLAG
      State: DRAFT until all FLAGs resolved by human
  → [Verification phase]
      Step 1: Build Registry                → registry.md
      Step 2: Build Cross-Reference Matrix  → matrix.md        (re-grounds on: registry.md + fixed rules)
      Step 3: Build Invariant Sets          → invariants.md    (re-grounds on: registry.md + matrix.md)
      Step 4: Trace Execution Paths         → paths.md         (re-grounds on: invariants.md)
      Step 5: Verify Intent                 → intent-notes.md  (re-grounds on: full spec + paths.md)
      → Gap report (assembles findings from all steps)
      State: UNVERIFIED until gap report is empty
  → [Human review]
      Human receives: draft spec + gap report (together, not separately)
      Human resolves: FLAGs and GAPs
  → [Resolution phase]
      LLM updates spec per human decisions
      LLM re-runs verification
  → [Sign-off gate]
      No open FLAGs, no open GAPs → state: VERIFIED
  → [Implementation hand-off]
      Verified spec + implementation checklist → coding LLM
```

---

### Why Re-grounding Is Structural, Not Optional

As a verification pass runs, context fills. The registry produced in Step 1 is further back in context by Step 4. Cross-section consistency checks against content written 50,000 tokens earlier become unreliable. This is not an edge case — it is the default behaviour of any LLM operating on a long spec.

The mitigation is externalisation. Each step produces a structured artefact — written out, not held in memory. Each subsequent step receives that artefact as explicit input before executing. The LLM is not asked to remember. It is given the information again.

Re-grounding granularity is per verification step, not per entity or per section. The within-step drift risk is low — each step operates on a single artefact it just produced. The cross-step drift risk is the problem this solves.

**Re-grounding map:**

| Step | Re-ground on before starting |
|---|---|
| Step 1 — Build Registry | Nothing. This is the first artefact. |
| Step 2 — Build Cross-Reference Matrix | `registry.md` + fixed required-presence rules (below) |
| Step 3 — Build Invariant Sets | `registry.md` + `matrix.md` |
| Step 4 — Trace Execution Paths | `invariants.md` (the most recent externalised artefact) |
| Step 5 — Verify Intent | Full spec + `paths.md` |

"Re-ground" means: read the artefact in full at the start of the step. Do not proceed until it is in context.

---

### The Fixed Cross-Reference Matrix

This table is a methodology-level contract. It does not change per spec. The LLM applies it — it does not generate it.

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

Every MISSING cell in the per-spec matrix is a gap. A gap is not a judgment — it is a binary result: required section is absent.

---

### The Self-Verification Problem and Why This Design Mitigates It

The LLM that wrote the spec runs the verification. The risk: the LLM cannot see its own blind spots. It may resolve contradictions silently — confirming what it wrote because it cannot perceive the error.

This design mitigates it through structural comparison, not judgment.

The gap report is not produced by asking "does this look right?" It is produced by comparing two independently constructed artefacts:

1. **What the spec claims** — extracted into the registry and invariant sets
2. **What the cross-reference matrix requires** — derived from the fixed rules above

A gap is a diff between these two artefacts. The LLM is not making a judgment about correctness. It is performing a structural comparison. A diff cannot be self-serving — it is either present or absent.

The residual risk is systematic bias: if the LLM misunderstood the domain when writing the spec, it will apply the same misunderstanding when building the matrix. Both artefacts will agree, and both will be wrong in the same direction. Step 5 — intent verification — is the only check that catches this. Step 5 cannot be automated. It requires a human who understands the domain.

The correlated-error probability for Steps 1–4 is low because the registry and the matrix are independently derived. Getting both wrong in a way that makes them agree requires the same error to manifest in two different structural operations. This is possible but unlikely for systematic spec errors. It is not a guarantee — it is a structural property that makes silent self-confirmation much harder than in a purely judgement-based verification.

---

### The Gap Report Format

The gap report is the primary output of the verification phase. It is also the primary input to human review. Its structure must allow the human to make decisions without having watched the verification happen.

**Format per gap entry:**

```
GAP-[N]
Step:        [which verification step found this]
Entity:      [the registered entity name, exactly as in the registry]
Type:        MISSING | CONTRADICTION | WEAK_CLAIM | INTENT_FLAG
Finding:     [one sentence: what is absent or inconsistent]
Evidence:    [section(s) and the specific claim(s) that conflict or are absent]
Required by: [the fixed rule or invariant that requires the missing content]
Resolution:  [OPEN — awaiting human decision]
```

**Gap types:**

- `MISSING` — a required section does not mention an entity the matrix requires it to mention
- `CONTRADICTION` — two sections make claims about an entity that cannot both be true
- `WEAK_CLAIM` — a claim exists but is logically weaker than an equivalent claim in another section (e.g. "present when safe=true" vs "null iff safe=false" — one is a strict subset)
- `INTENT_FLAG` — Step 5 only; a guarantee, assumption, or MUST with no traceable mechanism, or a rationale that may be incorrect

**Gap report header:**

```
VERIFICATION PASS — [date]
Spec version:    [version]
Steps run:       [1–5 / 1–3 / 4 only / 5 only]
Registry size:   [N entities]
Gaps found:      [N]
  MISSING:       [N]
  CONTRADICTION: [N]
  WEAK_CLAIM:    [N]
  INTENT_FLAG:   [N]
State:           UNVERIFIED (gaps remain) | VERIFIED (no gaps)
```

The gap report is empty when every gap entry has `Resolution: RESOLVED` and the spec has been updated accordingly. The header state changes to `VERIFIED`. A new verification pass is logged.

---

### FLAG Format (Generation Phase)

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

Flags must be self-explanatory. The human reads them without context from the generation process.

---

### Verification State Machine

A spec has exactly one of three states at any time:

| State | Meaning |
|---|---|
| `DRAFT` | Generation complete. Open FLAGs remain. Human has not reviewed. |
| `UNVERIFIED` | All FLAGs resolved. Verification run complete. Open GAPs remain. |
| `VERIFIED` | All FLAGs resolved. All GAPs resolved. Ready for implementation hand-off. |

State transitions:

- `DRAFT` → `UNVERIFIED`: human resolves all FLAGs, LLM updates spec, verification pass runs
- `UNVERIFIED` → `VERIFIED`: human resolves all GAPs, LLM updates spec, re-verification pass finds no gaps
- `VERIFIED` → `UNVERIFIED`: any substantive spec change invalidates verification; re-run required

A spec MUST NOT be handed to a coding LLM in `DRAFT` or `UNVERIFIED` state.

---

### Design Rationale

These are the decisions in this pipeline that are not obvious until something goes wrong. Each one was made for a specific reason. The reason matters — it determines what breaks if you change the decision.

**Why the spec has a state machine.**
Without explicit states, "done" is ambiguous. A coding LLM handed a DRAFT spec will implement against unresolved ambiguities. The state machine is a correctness gate, not process overhead. The downstream LLM needs a signal it can trust, not a human saying "I think it's ready."

**Why flags must name the alternative interpretation.**
A flag that says "ambiguous" is useless for async review. The human needs to see what the LLM almost decided, not just that a decision was made. The alternative interpretation is what makes the provisional decision reviewable without re-reading the entire input.

**Why gap entries reference the fixed rule that requires the missing content.**
Not just "§17 is missing for this failure mode" — but "missing because: failure modes MUST have a propagation path per the fixed matrix." Without the rule, the human cannot distinguish a genuine gap from a false positive. The gap must be self-contained. The human reviewing it was not present when it was found.

**Why verification re-runs after resolution.**
Resolving a gap is a spec change. A spec change can introduce new gaps. A human who adds a failure mode to §7 to close GAP-3 may not realise that failure mode now needs §17 coverage. The re-run is not bureaucratic — gap resolution is itself a spec mutation, and spec mutations invalidate verification.

**Why the implementation checklist is a separate output.**
The spec is written for human comprehension and LLM verification. A coding LLM needs something different — ordered, actionable, with explicit dependencies between tasks. The same information rendered differently for a different consumer. Collapsing them produces a document that serves neither well.

**Why human review receives spec and gap report together.**
The gap report is not a summary of problems to fix in isolation. It is the verification evidence that makes the spec reviewable. A spec without a gap report asks the human to re-do the verification. A gap report without the spec is abstract. The pairing is what makes async one-sitting review possible.

**Why Step 5 cannot be automated.**
Steps 1–4 verify internal consistency — the spec agrees with itself. Step 5 verifies intent — the spec describes the right thing. A spec can be internally consistent and consistently wrong. Detecting that requires understanding what the system is supposed to do well enough to recognise when the spec doesn't capture it. No structural check can substitute for that judgment.

---

### Relationship to Appendix A (Stress Tests)

Stress tests (Appendix A) and structural verification (this appendix) are complementary. They are not redundant.

Stress tests ask: can this spec describe a wide variety of system shapes? They test completeness and generality. Run once, before declaring the spec done.

Structural verification asks: are the claims in this spec consistent with each other? It tests internal coherence and cross-section agreement. Run continuously, on every substantive change.

Vector 7 (Cross-Section Consistency) in the stress-test protocol overlaps most closely with Steps 2–4 here. Vector 7 asks specific questions about known consistency risks. The verification pipeline finds consistency failures that Vector 7's specific questions do not cover. Both are required.

---

### Keeping Verification Current

Every substantive spec change invalidates some verification steps. When a change is made, identify which steps are invalidated and re-run only those.

| Change type | Steps invalidated |
|---|---|
| New entity added | 1, 2, 3 |
| Entity renamed | 1, 2, 3 |
| New failure mode | 2, 3, 4 |
| Pipeline ordering changed | 4 |
| Audit sequence changed | 4 |
| Security guarantee changed | 5 |
| Design rationale changed | 5 |
| New assumption added | 5 |

Track verification state in the spec itself. Add a verification log entry for each pass: date, steps run, gaps found, gaps resolved. A spec whose last full verification pass was months ago and has had significant changes since is a known risk. Make the risk visible.

---

---

## Deployment Guidance

The spec declares what the system must do. This section — added after §23 in any spec that warrants it — captures what operators must do to keep the system working correctly in production over time.

Deployment guidance is not a correctness contract. Violating it does not make the spec wrong. It exists because a spec that is pure theory leaves operators without guidance on the gap between "deployed correctly at launch" and "remains effective over time."

---

### When to add a Deployment Guidance section

Add it when any of the following apply:
- The system has static defences (filters, thresholds, classifiers) that degrade against an active adversary over time.
- The system produces a forensic or audit record whose retention, access control, or availability has security implications.
- The system depends on external components (models, services, files) whose currency or availability affects security posture, not just availability.
- Configuration changes are security-relevant — a misconfigured deployment silently weakens protection without producing an error.

If none of these apply, the section is optional. A system with no time-varying security properties and no operator-tunable safety controls may not need it.

---

### What belongs here vs. in the spec

**Belongs in the spec (correctness contract):** behaviour that, if violated, produces a wrong result or broken guarantee visible to callers or auditors. "The audit backend MUST NOT raise in `log()`" is a correctness contract — it belongs in §16.

**Belongs in Deployment Guidance (operational guidance):** behaviour that, if neglected, degrades effectiveness or increases risk over time without breaking a stated contract. "Audit logs SHOULD be retained for a period sufficient for incident investigation" is operational guidance — it belongs here.

When the boundary is unclear, ask: does violating this produce a wrong result now, or a worse outcome later? Now → spec. Later → Deployment Guidance.

---

### Structure

Write deployment guidance as named subsections, one concern per subsection. Each subsection states:
- What the concern is and why it matters.
- What operators MUST or SHOULD do about it.
- What the consequence of neglect is, stated concretely.

Reference the Deployment Guidance section from the relevant spec sections (e.g. from §19 Security Properties when noting that static defences degrade). Do not embed operational guidance inline in spec sections — cross-reference instead.

---

## Future Directions

Add a Future Directions section (§24 or equivalent, after §23) when the system has known, intentional extension paths that current architectural decisions are designed to accommodate.

---

### Purpose

Future Directions is not a contract. Nothing in it is a current commitment. It exists to:
- Prevent future designers from treating current architectural limits as permanent.
- Document the invariants that future extensions MUST preserve (fail-closed contracts, interface stability rules).
- Ensure current design decisions are understood in the context of what they are designed to accommodate.

---

### What belongs here

Document each known extension direction as a named subsection. For each:
- What the extension achieves — one paragraph, no implementation detail.
- What constraints it operates under — invariants from the current spec that MUST be preserved.
- What design questions remain open — the unknowns that make this a future direction rather than a current contract.

Future Directions MUST NOT describe implementation approaches. The how is for the design phase when the extension is actually built. The section describes intent and constraints only.

---

### Keeping it current

When an anticipated extension is built:
1. Update the relevant spec sections (§16 for new extension contracts, §7 for new failure modes, §17 for propagation paths, §20 for stability levels).
2. Mark the Future Directions entry as implemented, or remove it.
3. Re-run the stress-test pass (Appendix A) against the updated spec.

A Future Directions section that is never updated becomes fiction. Treat it as a backlog, not an archive.

---

## References

These are the foundational sources this methodology draws from. Consult them when the system requires more rigour than this document provides.

**Design by Contract (Meyer, 1992)**
The origin of preconditions, postconditions, and invariants as a formal specification method. The component contract format in §5 is a prose adaptation of DbC. For systems where correctness is critical, consider applying DbC formally.
*Bertrand Meyer, "Applying 'Design by Contract'", IEEE Computer, 1992.*

**RFC 2119 — Key words for use in RFCs to indicate requirement levels**
Defines MUST, SHOULD, MAY, MUST NOT, SHOULD NOT precisely. The writing rules and sections in this skill use these words in conformance with RFC 2119.
*https://www.rfc-editor.org/rfc/rfc2119*

**ISO/IEC/IEEE 29148 — Systems and software engineering: Requirements engineering**
The international standard for software requirements specifications. The right reference when the system will be built by multiple independent teams or must satisfy regulatory requirements.
*ISO/IEC/IEEE 29148:2018*

**The C4 Model (Brown)**
A lightweight diagramming approach for system context, containers, components, and code. Useful for §3 and §5 when the system is complex enough that prose alone is insufficient.
*https://c4model.com*

**TLA+ / Alloy**
Formal specification languages for systems where ordering, sequencing, and concurrency constraints cannot be captured unambiguously in prose. If §11 or §13 cannot be made falsifiable in prose, consider a formal model.
*TLA+: https://lamport.azurewebsites.net/tla/tla.html*
*Alloy: https://alloytools.org*

**Software Engineering at Google (Winters, Manshreck, Wright — O'Reilly, 2020)**
Practical guidance on how specifications fit into an engineering workflow — when to write them, how detailed to make them, and how to prevent them from going stale.
