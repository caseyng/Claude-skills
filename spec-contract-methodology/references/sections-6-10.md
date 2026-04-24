# Sections §6–§10

---

## §6 Lifecycle

Every stateful component's valid states and valid transitions between them.

If stateless (declared in §5): write "Stateless — no lifecycle to specify." Do not pad.

For each stateful component:
- State immediately after successful construction
- State if construction fails partway — valid partial state, or invalid and not to be used?
- Operations valid in each state
- What calling an operation in an invalid state does — raises / returns error / silently ignores (each is a different contract)
- State after `close()` or equivalent
- Whether reusable after `close()`
- Whether `close()` is idempotent — what happens on second call

Use a state diagram if transitions are non-trivial. Either way: every state and every
valid transition MUST be named.

*Questions to answer:*
- What happens if you call `process()` before `init()`?
- What happens if you call `process()` after `close()`?
- What happens if you call `close()` twice?
- What happens if construction fails partway through?
- What is the minimum call sequence to reach a usable state?

---

## §7 Failure Taxonomy

Master list of every failure mode the system can produce. §5 references this list.
§17 traces how failures travel. A failure MUST be defined here before it can be
referenced elsewhere.

**First: distinguish failure from defined operational outcome.** "Key not found",
"input rejected", "empty result" are defined outcomes. A failure is a condition that
prevented the system from producing any defined outcome. Document defined outcomes in
§5. Document failures here. If the distinction is unclear, resolve it explicitly and
state the resolution.

For each failure mode:
- **Name** — exact string or constant if it appears in output or error messages
- **Meaning** — what condition caused it, precisely
- **Category** — one of four:
  - *Programming error:* caller did something wrong — wrong arg type, called after close, violated precondition
  - *Operational failure:* external dependency unavailable, timed out, or returned unexpected response
  - *Adversarial signal:* input designed to bypass, exploit, or destabilise the system
  - *Internal invariant violation:* condition that should be impossible given correct implementation, detected defensively. Means "a bug in the system was detected at runtime." Callers cannot recover from this by changing their behaviour — only a fix to the system resolves it. These are the highest-severity failures and MUST NOT be conflated with programming errors.
- **Recoverability** — retry immediately / retry after backoff / permanent for this input
- **Information carried** — what data accompanies this failure; which fields guaranteed present vs conditionally present
- **Representation** — returned as structured value / raised as exception / both. If both possible for the same condition, state when each applies.
- **What is NOT done** — what the system explicitly does not do when this failure occurs

**Declare whether the set is closed.** If callers MAY safely enumerate all values and
branch exhaustively, say so. If future values MAY be added, say so.

**Open sets require a handling contract.** Specify what the caller MUST do when
receiving an unknown value: treat as fatal / treat as most conservative known failure /
ignore / other. An open set without a handling contract is an incomplete specification.

---

## §8 Boundary Conditions

For every operation: exact behaviour at the edges of valid input.

§5's `Accepts` field covers what is valid. This section covers what happens *at the exact
edges* — do not duplicate the validity rule; document the edge behaviour.

For each operation, explicitly specify behaviour for:
- Empty input (empty string, empty list, zero-length collection)
- Null/None input where the type is nullable
- Minimum valid value and one below it
- Maximum valid value and one above it
- Input structurally valid but semantically degenerate (e.g. list with one element
  where multiple expected; range where min equals max)
- Input that was valid in a previous call but system state has since changed

"Handles gracefully" is not a specification. "Returns an empty result" is.
"Raises a configuration error identifying the field by name" is. State the exact behaviour.

---

## §9 Sentinel Values and Encoding Conventions

Every place the system uses null/None/zero/empty string/special values as semantic signals.

§5 documents the contract per field. §8 documents edge behaviour. This section documents
values that carry meaning as signals — particularly where the same value means different
things in different contexts.

For each:
- Where it appears — field name, return value, parameter
- What it means in that specific context
- What it does NOT mean — if the same value means different things in different contexts, make each distinction explicit
- What a caller MUST do when they encounter it

The same value with multiple meanings in different contexts, none of them documented,
will produce one bug per meaning in an implementation.

---

## §10 Atomicity and State on Failure

For any operation that modifies state: what is the state of the system if the operation
fails midway?

If stateless: write "Stateless — no mutable state, section not applicable." That is a contract.

For each state-modifying operation:
- Whether the operation is atomic — either completes fully or has no effect
- If not atomic: what partial state is possible, and whether the system can recover from it
- Whether previous state is preserved on failure (rollback), undefined (corrupt), or deterministically one of a known set of states
- Whether the caller can detect that a partial write occurred, and how
- Whether a retry of a failed operation can cause duplicate effects — if so, what the duplicate looks like and how to detect it

This section is distinct from §7 (what error the caller sees), §17 (how errors travel),
and §6 (what lifecycle state the component is in). This section documents internal
data state after failure — which the caller may need to query before retrying safely.

**Ungraceful shutdown.** If the system holds durable state and the process is killed
without calling `close()` — by signal, crash, or power loss — state explicitly:
- What condition the durable state is left in
- Whether it is always recoverable
- Whether it is detectable as corrupt
- Whether the system performs recovery automatically on next startup, or requires operator intervention

A spec that does not answer these questions describes a system that cannot be safely
deployed where crashes are possible.
