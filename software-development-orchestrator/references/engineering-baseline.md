# Engineering Baseline

Owned by the software-development-orchestrator. Packaged into every Stage 3 agent's context.

This document defines the floor — the norms any competent engineer applies automatically.
Nothing in this baseline needs to be derived from the spec or argued for. What the baseline
defines, the spec assumes. What conflicts with the baseline is a spec defect.

The purpose of encoding this is not to tell engineers what to do. It is to give an LLM
the same background model a senior engineer brings to every problem — so that the LLM
does not spec or implement against the baseline's implicit constraints.

---

## Names Must Describe Their Subject

Every name — module, function, class, variable, endpoint, file, configuration key — must
describe what it is without requiring the reader to open it.

A name that requires reading the implementation to understand has failed. A name that
requires knowing a metaphor, abbreviation, or internal naming convention has failed.

The test: can someone unfamiliar with this codebase read the name and form a correct
model of what it does? If not, rename it.

This applies to error names, event names, state names, and notification channels — not
just source identifiers.

---

## Failures Are Named

Every failure mode a component can produce must have a name. The name appears in the
component's contract. Callers handle failures by name — they do not pattern-match on
exception messages or error codes whose meaning is not in the contract.

Anonymous failures (unnamed exceptions, generic error codes, silent returns) are defects.
A caller that cannot name the failure it is handling cannot handle it correctly.

The failure taxonomy is defined during specification, not during implementation.
Discovering a new failure mode during implementation is a spec gap, not a feature.

---

## Nothing Is Silent

A component that fails silently has corrupted the system's ability to understand itself.

Every automated action must be observable: what happened, when, why, and what the outcome
was. An observer reading the output — without reading source code — must be able to
reconstruct execution.

Silent success is acceptable. Silent failure is a defect. Silent partial completion
(some work done, some not, no indication of which) is a defect.

This applies to: background services, scheduled actions, rule evaluations, retries,
fallbacks, and anything that operates without direct human interaction.

---

## Responsibility Boundaries

A component guards against failures it can reach. It does not guard against states
the caller is responsible for preventing.

If a function requires a non-null argument and all callers are internal code under the
same team's control, the function does not add a null check. The null check belongs in
the caller or the type system, not here. Adding it here is noise that obscures the
real contract and creates a false sense of safety.

Caller responsibilities are documented as assumptions in the component's contract,
not handled as boundary conditions in the implementation.

The test: can this input state actually arrive at this component given the system as
designed? If no, it is the caller's responsibility — document it as an assumption,
not a guard.

---

## Validation at System Boundaries Only

Validate input at the system boundary: where data enters from the user, from an
external API, from a network call, from a file the user controls.

Do not validate between internal components under the same team's ownership.
Internal calls trust the type system and the caller's contract. Adding redundant
validation at every internal boundary adds noise, creates false safety, and
obscures where real validation happens.

The test: is this data from a source the system controls? If yes, validate at entry.
If the data comes from our own code, trust it.

---

## Extension Without Modification

A deferred feature must be addable later by writing new code, not by editing
existing code. If adding the feature later requires changing an existing file,
the extension point is missing.

Extension points are minimal: an interface, a stub, an empty configuration field,
a type parameter. They cost almost nothing at Phase 1. Their absence costs rework.

The test: if this deferred feature were added in Phase N, which existing files change?
If any file changes beyond wiring up the new code, the extension point is missing.
Design it in now.

---

## State Lives in One Place

Every piece of state has one owner. The owner is responsible for its consistency.
Other components read from the owner — they do not hold copies.

Duplicated state (the same fact stored in two places that can disagree) is a defect
by design, not a defect by bug. If the architecture allows two components to have
different views of the same fact, the architecture is wrong.

This applies to: connection status, session credentials, rule configurations, user
preferences, and any other state that multiple components may need to read.

---

## Contracts Before Code

Interfaces and data schemas are defined before the code that uses them.

A function's contract — what it requires, what it guarantees, what it produces on
failure — is derivable from its signature and name alone, without reading the body.

If the name and signature require reading the body to understand, the contract
has not been defined — it has been inlined into the implementation.

---

## Security Defaults

These are not features to consider. They are the starting point.

- Secrets never in source control, build artifacts, or logs
- Network communication: TLS only, cleartext prohibited
- Sensitive data at rest: encrypted
- Permissions: request only what is needed, at the point of need, with an explanation
- Component exposure: unexported by default; exported only when external use is required
- Pending references (intents, callbacks): immutable by default

Any deviation from these defaults requires an explicit justification in the spec.
Absence of justification means the default applies.

---

## Observability Is Designed In

Logging, tracing, and error reporting are not added after the fact. They are designed
into the component at specification time.

Every component that performs automated actions must specify: what events it logs,
at what level, with what context fields. An action log entry must contain enough
information to answer — without reading source code — what happened, to what, when,
why it was triggered, and what the outcome was.

A component that cannot be understood from its output alone is not finished.
