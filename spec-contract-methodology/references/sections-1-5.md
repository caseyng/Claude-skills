# Sections §1–§5

---

## §1 Purpose and Scope

What problem does this system solve? One paragraph. State it precisely enough that a
reader with no prior context can determine what the system does and does not do.

Then: what is explicitly out of scope? Name the things a reasonable person might expect
this system to handle that it does not. Without an explicit scope boundary, implementors
fill gaps with assumptions.

*Questions to answer:*
- What is the one thing this system does that nothing else does?
- What would break if this system were removed?
- What adjacent problems does this system deliberately not solve?

**Design principles.** If the system has a small set of named principles that constrain
every implementation decision — tradeoffs the spec author has resolved in advance —
document them here. Each principle should be one sentence stating the rule, not an
aspiration. A principle is only useful if it can be applied as a test: "does this
decision violate principle X?" If no such principles exist, omit this subsection.

---

## §2 Concepts and Vocabulary

Every domain term the spec uses, defined precisely. Not dictionary definitions —
operational definitions: what does this term mean *in this system*, and how does it
differ from related terms?

Define terms before using them. If a term is used inconsistently anywhere in existing
documentation, resolve it here and note the resolution.

*Questions to answer:*
- What terms appear in the code or docs that a newcomer would need to look up?
- Are there terms used in two different senses anywhere?
- Are there concepts that are implied but never named?

---

## §3 System Boundary

What enters the system, what leaves it, and what the system promises about the
transformation between them.

For each **input:** type, valid range, what happens on invalid input, whether
null/None/empty is valid and what it means.

For each **output:** type, every possible value, what each value means, what guarantees hold.

For each **side effect:** what it is, when it occurs, whether guaranteed or best-effort.

**Caller types.** If multiple distinct caller types exist with different contracts,
specify each separately. Do not average them into a single contract that fits none precisely.

**Idempotency.** State explicitly whether identical inputs produce identical outputs.
If not, state what varies and why. A caller who assumes idempotency where none exists
will produce incorrect retry logic.

**Delivery semantics.** For any stream, queue, or event sequence: state the delivery
guarantee explicitly — exactly-once, at-least-once, or at-most-once. State what the
system does on duplicate, out-of-order, or gap-in-sequence events. These are
correctness contracts, not performance details.

*Questions to answer:*
- Same input twice → same output?
- What can a caller rely on unconditionally?
- What can a caller rely on only under certain conditions?
- What inputs are rejected at the boundary vs. handled internally?
- Are there multiple caller types with different contracts?

---

## §4 Data Contracts

The shape of every structured data format that crosses a public boundary.

§18 covers event fields. §15 covers config fields. This section covers everything else:
API request/response schemas, file formats read or written, wire protocols, serialisation
conventions, interchange formats.

For each format at a public boundary:
- Name and purpose
- Schema — every field: name, type, required vs optional, valid values, what absent means vs null
- Unknown fields — what the system does on receipt of an unrecognised field (reject /
  ignore / pass through); what a caller MUST do on receipt of an unrecognised field
- Version negotiation — if format can change across versions, how producer and consumer
  agree on version. Stability and breaking-change rules are in §20.
- Encoding — character encoding, byte order, compression, platform-dependent
  serialisation choices affecting portability

A data format at a public boundary is a contract. The stability and versioning rules
from §20 apply equally to data contracts.

---

## §5 Component Contracts

Every component with a public interface, specified as a contract — not an implementation.

**Component hierarchy.** If the system has multiple components, open this section with
a diagram or structured list showing how components are nested and owned. This gives an
implementor the map before the individual contracts. A reader who cannot see the overall
structure has to reconstruct it from the contracts — extra work with room for error.
The hierarchy here is not a contract; it is orientation. Contracts follow.

**Precondition:** what MUST be true before an operation is called. Document what the
component does when called with a violated precondition — raises a specific error,
returns a specific value, or undefined behaviour. Do not leave this implicit.

**Invariant:** condition that holds continuously throughout component lifetime — never
violated, even transiently between operations.

**Postcondition:** condition guaranteed to hold after a specific operation completes.

**Stateless components.** If a component holds no mutable state, declare this explicitly:
"this component is stateless — it holds no mutable state between calls." This is a
contract. An implementor who does not know the component is stateless may add caching
or connection state that changes observable behaviour.

**Failure behaviour references §7.** Each failure behaviour entry names a failure mode
and describes the caller-visible result. The failure mode itself is defined in §7, not here.

### Contract format

```
Component: [name]
Purpose:   [one sentence — what it does, not how]

Stateful or stateless: [state explicitly]

Invariants:
  - [condition that holds for entire component lifetime]
  - OR: "None — this component is stateless."

Preconditions for [operation]:
  - [what must be true before this operation is called]
  - Violation behaviour: [what the component does if precondition not met]

Accepts:
  [field]: [type] — [valid range, what null means if nullable]

Returns:
  [success case]: [condition — e.g. "on success", "if key exists"]   ← always include success explicitly
  [other values]: [condition under which this value is returned]

Postconditions:
  - After [operation]: [condition that holds]
  - [field] is null if and only if [condition]

Guarantees:
  - Never raises [error type] — [why this matters to callers]
  - Always [unconditional behaviour]

Failure behaviour:
  [failure mode name, defined in §7]: [what is returned to caller, what is NOT done]
```

Do not describe implementation. "Uses a regex" is implementation. "Returns false if
input matches any configured pattern" is contract.
