# Sections §17–§23

---

## §17 Error Propagation

For each failure mode in §7: how it travels through the system from origin to the
caller's boundary.

For each failure mode:
- Where it originates — which component, which operation
- Whether absorbed at origin — logged, converted to structured result, suppressed with stated rationale
- Whether transformed as it propagates — wrapped, annotated, reclassified
- Where it surfaces to the caller — which boundary, in what form
- Whether the system remains in a defined state after propagation

A caller who does not know that a failure in component C surfaces as error E at the
top-level boundary cannot write correct error handling. This is not derivable from §5
alone — §5 documents each component's individual contract, not how failures travel
across them.

*Questions to answer:*
- If a dependency fails mid-operation, what does the top-level caller see?
- If an extension raises an unexpected error, does it propagate or get converted?
- Are any errors silently swallowed? If so, what signal if any does the caller receive, and what is the rationale?

**Caller-side exception types.** If the spec defines exception types that callers MAY
raise from response values — as a control-flow convenience rather than exceptions raised
by the system itself — document them explicitly and separately from internal propagation.
For each type: the condition it represents, which failure modes or outcomes it maps to,
and what it is distinct from. State clearly that these types are never raised by the
system internally. The distinction matters: a caller who conflates a system-raised
exception with a caller-convenience type cannot write correct error handling.

---

## §18 Observability Contract

If the system emits events, logs, metrics, or audit records: specify them as a contract.

For each event type:
- Name
- When emitted — the condition, not a line number
- Fields carried — names, types, meanings, which always present vs conditionally present
- Whether emission is guaranteed or best-effort
- Whether emitted before or after the operation it describes — matters for crash recovery and audit completeness
- Whether fields can be null and under what condition
- Whether the event can be emitted more than once for a single logical operation

If the system is event-driven and observability events overlap with processing events:
document which are internal processing records and which are external observability
signals, and whether they carry different guarantees.

If events form a sequence callers can reconstruct state from: document sequencing guarantees.

**What is not logged.** For security-sensitive or privacy-sensitive systems: explicitly
state what the system deliberately omits from its observability output. Omissions that
are deliberate design decisions — not oversights — MUST be named. An operator who does
not know the system omits certain data cannot reason about what the audit record does
and does not prove. An implementor who sees no guidance will make their own omission
decisions.

**Schema versioning.** If the event schema can change across system versions, specify
how consumers determine which schema applies to a given record. A consumer handling
records from multiple versions of the system cannot rely on field presence inference —
they need a declared mechanism. State which record or field carries the version signal,
and what a consumer MUST do when handling records whose version they do not recognise.

---

## §19 Security Properties

What the system guarantees under adversarial input. "Secure" is not a property. Be precise.

**Single-caller adversarial input.** For each security-relevant decision:
- What the system guarantees — precisely enough to be falsified by a test
- What the system explicitly does not guarantee
- Whether the decision is fail-closed or fail-open, and why
- What an adversary who knows the full specification can do, and what they cannot do

**Multi-caller and resource isolation.** If the system serves multiple callers, document:
- Whether one caller's input can affect another caller's output
- Whether one caller can exhaust resources that degrade service for others
- What the system does when resource limits are reached
- Absence of isolation is a security property and MUST be stated, not assumed

Document every tradeoff between availability and security explicitly so operators can
make informed choices.

---

## §20 Versioning and Evolution

Which parts of the spec are stable contracts callers MAY depend on across versions,
and which may change. Covers: operational interfaces, data formats (schemas, field
names, encoding), event contracts (§18), configuration fields (§15).

For each part of the public interface:
- Stability level: *stable* (no change without major version increment) / *evolving* (MAY change with notice) / *internal* (not a public contract)
- What compatibility guarantees exist — does adding a field to a response break existing callers?
- What constitutes a breaking change
- How breaking changes are communicated

**Spec maintenance.** The spec is a versioned document. Each substantive change MUST
be recorded: what changed, why, and what the previous contract was. A reader must be
able to tell whether the spec is current. Mark provisional sections so implementors
know where the contract is still being established.

---

## §21 What Is Not Specified

An explicit list of decisions left to implementors.

Things the spec intentionally does not prescribe: internal data structures, algorithm
choices, logging verbosity and format, concurrency model, memory layout, retry strategies.
An implementor MAY choose any approach that satisfies the contracts above.

Without this section, an implementor cannot distinguish "doesn't matter" from
"the spec forgot."

---

## §22 Assumptions

Every condition the spec takes as given without proof. An implementation that satisfies
every written contract may still behave differently if these assumptions are violated
in a different environment. An assumption not stated in the spec is a failure mode the
implementor cannot anticipate.

Classify each assumption by type:

**Environmental** — properties of the runtime or platform:
- Arithmetic (e.g. integer overflow wraps, floats conform to IEEE 754)
- Clock (e.g. system clock is monotonic, never goes backwards)
- Filesystem (e.g. writes to a single file are atomic up to page size, the filesystem does not silently corrupt)
- Memory (e.g. runtime manages memory, no manual deallocation)
- Character encoding (e.g. strings are UTF-8, platform default not assumed)

**Caller** — what the system assumes about the entity calling it:
- Trust level (e.g. caller is authenticated / cooperative / adversarial)
- Call pattern (e.g. caller calls from single thread, caller does not call after close)
- Input validity (e.g. caller validates inputs before calling, or system validates itself — make explicit)

**Operational** — what the system assumes about its deployment context:
- Network (e.g. network reachable at startup, latency bounded)
- Configuration (e.g. config file exists and is readable, secrets present)
- Time (e.g. wall clock approximately correct, NTP running)

For each assumption: state it, state what happens if violated, state whether violation
is detectable. An assumption whose violation is undetectable is a failure mode the
implementor cannot anticipate and MUST be flagged.

---

## §23 Performance Contracts

The boundary between performance properties and correctness contracts.

**The test:** if the property is violated, does the caller receive an incorrect result,
a missing guarantee, or an undeliverable contract? If yes → correctness contract,
belongs here. If caller receives correct result more slowly → performance characteristic,
belongs in §21.

*Correctness contracts that appear to be performance:*
- Timeout: "operation MUST return within N seconds or produce a timeout failure" — correctness. Caller's retry logic depends on it.
- Ordering: "events MUST be delivered in submission order" — correctness. Callers building state from events depend on it. Note: delivery semantics (exactly-once etc.) belong in §3; ordering constraints deriving from those semantics belong here.
- Bounded memory: "system MUST NOT buffer more than N items" — if exceeding causes data loss, this is correctness.

*Genuine performance characteristics — belong in §21, not here:*
- Throughput targets
- Latency percentiles (except when defining SLAs the system guarantees)
- CPU and memory usage

For each property that looks like performance but affects correctness:
- State the property precisely
- State what the caller observes if violated
- State whether the system enforces the bound internally or delegates to the caller
