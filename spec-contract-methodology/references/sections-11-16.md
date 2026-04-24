# Sections §11–§16

---

## §11 Ordering and Sequencing

If the system has a pipeline, chain, or sequence of operations: document ordering
constraints explicitly.

For each step:
- What it reads from context (inputs from prior steps)
- What it produces (outputs for subsequent steps)
- What invariants must hold when it runs
- Why it MUST precede or follow adjacent steps — state the correctness or security reason, not just the order

"The current implementation does X before Y" is not a sequencing constraint.
"X MUST precede Y because Y depends on the output of X" is.
"X MUST precede Y because allowing Y before X creates security gap Z" is.

If the order can be changed without correctness or security impact, say so explicitly.
An implementor who sees an ordering without a reason will assume the reason exists.

---

## §12 Interaction Contracts

Where two components interact, new contracts emerge not visible from either component's
individual specification. These MUST be documented here.

For each significant interaction:
- Which components are involved
- Who initiates (caller) and who responds (callee)
- What the caller guarantees before calling (preconditions the callee MAY assume)
- What the callee guarantees after returning (postconditions the caller MAY rely on)
- Who owns shared resources — who constructs, who closes, who MUST NOT close
- What happens if interaction fails midway — is either component left in a defined state?

Resource ownership documented here MUST be consistent with §16. Cross-reference explicitly.

**Common interaction contracts that are frequently undocumented:**
- Resource ownership at component boundaries
- Callback/hook contracts (see below)
- Shared mutable state access patterns (who reads, who writes, in what order)
- Event sequencing guarantees between producer and consumer

**Callback contracts.** If the system accepts callbacks or event handlers supplied by
the caller — whether at construction or per-call — specify for each callback point:
- When the callback will be called — the exact condition, not "when relevant"
- Which thread or execution context it will be called on
- Whether the callback may be called re-entrantly (from within a call the caller is already in)
- What happens if the callback raises — propagated / suppressed / logged / undefined behaviour
- Whether the callback will be called again after it raises
- Whether the callback may call back into the system, and if so which operations are safe

Re-entrancy constraints for callbacks belong in §13. Document full concurrency contract
there; reference it here.

A callback implemented without these constraints will work under light load and fail
unpredictably under concurrency or error conditions.

---

## §13 Concurrency and Re-entrancy

State explicitly what guarantees the system makes under concurrent access. "Thread-safe"
is not a specification.

For each component or operation:
- Whether it is safe to call from multiple threads simultaneously
- Whether it is safe to call re-entrantly (from within a callback invoked by the same component)
- Consequences of violating these constraints — data corruption / deadlock / undefined behaviour / specific error
- Whether any internal synchronisation is provided, and if so what it covers and what it does not

If the system does not provide concurrency guarantees and delegates responsibility to
the caller, say so explicitly. If the spec says nothing about concurrent access, an
implementor will invent a behaviour — state what concurrent callers may expect, even
if the answer is undefined behaviour.

**Cross-reference §22.** Caller assumptions about call patterns (single-threaded, no
concurrent calls) belong in §22. §13 documents what the system itself guarantees or
enforces. They MUST be consistent: if §22 assumes single-threaded callers, §13 MUST
state that concurrent calls produce undefined behaviour (or specify the consequence).
If §13 makes the system safe for concurrent calls, §22 MUST NOT assume single-threaded access.

---

## §14 External Dependencies

Every resource the system requires that exists outside its own boundary.

For each dependency:
- Name and what it provides
- Required (system cannot start without it) vs optional (system degrades without it)
- What "optional" means precisely — which capabilities unavailable, which errors produced, whether degradation is signalled to the caller
- What happens at startup if absent — fails with specific error / starts in degraded mode / silently continues
- What happens at runtime if it becomes unavailable — fails current operation / queues / retries / fails open
- Whether verified at startup (eager) or on first use (lazy), and why

*Examples:* libraries that must be installed, network endpoints, files on disk,
OS features, environment variables.

Distinct from §15 Configuration (operator-supplied values), §3 System Boundary
(caller-supplied inputs), and §22 Assumptions (properties taken as given).
External dependencies are things the system actively reaches out to.

---

## §15 Configuration

Every configuration field the system accepts.

For each field:
- Name
- Type
- Valid range or valid values — exhaustive if small, bounds if continuous
- What null/None/absent means — and whether null and absent mean the same thing
- Default value and why that default was chosen
- What happens if an invalid value is supplied — at construction time or at runtime
- Whether the field affects security posture — mark explicitly; a misconfigured security field may silently weaken protection
- Whether the field can be changed after construction, and if so what the effect is on in-flight operations

Do not describe defaults as "sensible" or "safe" without defining what those words mean
in this context. A default safe for one deployment may be unsafe for another.

---

## §16 Extension Contracts

If the system is designed to be extended — new implementations, new plugins, new
handlers — specify what an extension MUST provide and what it MUST preserve.

For each extension point:

**MUST implement:** operations with no universal default. State what "correct
implementation" means — not just the signature but the full behavioural contract the
system will rely on.

**MAY override:** operations with a sensible base behaviour. State the base behaviour
so an extension author knows what they are deviating from.

**MUST NEVER do:** invariants the extension must preserve. State the consequence of
violating each — data corruption / double-close / security bypass / deadlock.

**Owns vs. references:** which resources the extension constructs and owns (MUST close)
vs. which are injected (MUST NOT close). Cross-reference §12 — resource ownership MUST
be consistent between both sections.

**Registration:** how an extension registers itself. Whether existing files must be
edited. Whether registration changes security posture.
