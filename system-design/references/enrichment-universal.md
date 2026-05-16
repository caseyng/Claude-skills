# System Design Enrichment — Universal

Concerns that apply to every project regardless of platform. Apply in Phase 4.
For each concern not already resolved by the component decomposition, propose a default
owner and enforcement mechanism, then confirm with the human.

---

## Startup and Initialization Sequence

In what order do components initialize? What happens if a component fails to start?

The startup sequence matters when:
- Component B requires Component A to be ready before it can operate
- A failed startup leaves the system in a partially initialized state

Default: identify the initialization dependency chain. Document it as a cross-cutting concern
owned by the top-level entry point (main process, Application class, etc.). Any component that
can fail to initialize must have a defined behavior: abort startup, start in degraded mode, or retry.

**Questions to resolve:**
- Does any component require another to be ready before it starts?
- What does the system do if a required component fails to initialize?
- Is partial startup acceptable (some components up, some down)?

---

## Shutdown and Cleanup

How does the system shut down cleanly?

Default: identify what each component needs to do before termination — flush pending writes,
close network connections, commit in-progress transactions. The component that receives the
shutdown signal is responsible for sequencing cleanup in reverse initialization order.

**Questions to resolve:**
- What state can be lost without consequence during an abrupt shutdown?
- What state must be persisted before the process exits?
- Which component coordinates the shutdown sequence?

---

## Failure Isolation

If one component fails, can others continue?

Default: each component fails independently unless there is an explicit reason for cascade
failure. A component that depends on a failed component degrades gracefully or enters a
defined waiting state — it does not crash.

**Questions to resolve:**
- Which components are on the critical path (their failure stops all meaningful operation)?
- Which components can operate in degraded mode when a dependency is unavailable?
- What does "degraded mode" look like for each non-critical component?

---

## State Consistency on Crash and Restart

When the process is killed unexpectedly and restarts, is state consistent?

Default: assume crash can happen at any point. State is consistent if the system can restart
and resume correctly without manual intervention. Inconsistent state (partial write, uncommitted
transaction, in-flight message) must be detectable and recoverable.

**Questions to resolve:**
- Which component is responsible for detecting inconsistent state on startup?
- What is the recovery action for each type of inconsistency (retry, discard, alert)?
- Are there operations that must be atomic across components? If so, who coordinates the transaction?

---

## Deployment Unit

Is this one process or multiple? Same machine or distributed?

This decision has architectural consequences:
- **One process**: components can share in-memory state and communicate via function calls or shared objects
- **Multiple processes**: components cannot share in-memory state; they must communicate via IPC, file, socket, or message bus

Default: one process unless there is a specific reason to separate (OS-enforced lifecycle separation,
different permission domains, independent scaling requirements). Document the reason if separating.

**Questions to resolve:**
- Does any component require a different OS-level lifecycle than the main process?
- Does any component run at a different privilege level?
- Are there independent scaling requirements that force process separation?

---

## External Dependency Ownership

What external systems does this product depend on? Which component owns each?

Every external dependency (third-party API, hardware device, OS service, cloud backend)
must be owned by exactly one component. That component:
- Holds the connection or session
- Handles reconnection and retry
- Translates external failures into named internal failure modes
- Is the only component that knows the external system exists

Default: one component per external dependency. Other components interact with the owning
component, not with the external system directly. This isolates the blast radius when the
external system changes or fails.

**Questions to resolve:**
- What external systems does this product depend on?
- For each: which component owns the connection and hides the external dependency?
- What happens to the owning component (and its dependents) when the external system is unavailable?

---

## Configuration

Where does configuration live? Which component reads it? Can it change at runtime?

**Static configuration** (set at startup, not changed while running): owned by the entry-point
component and injected into dependents at initialization. Not re-read after startup.

**Dynamic configuration** (can change while running): owned by a dedicated settings component
that notifies dependents on change. Dependents do not poll configuration — they receive updates.

Default: static configuration for most settings. Dynamic configuration only for settings the
user can change without restarting the application.

**Questions to resolve:**
- Which settings are static (require restart to change)?
- Which settings are dynamic (can change at runtime)?
- Which component is responsible for reading, validating, and distributing each class of config?

---

## Data Ownership

Every data store has exactly one owner. Other components are readers only.

The owner:
- Controls the schema
- Runs migrations
- Writes data
- Defines what other components may read and in what form

Readers:
- Never write to a store they don't own
- Access data through the owner's defined interface, not by directly querying the store
- Exception: read-only access to the same physical store is acceptable if the owner controls the schema

Default: assign data ownership as part of the component decomposition. If two components
both need to write to the same data, the boundary is wrong — merge them or introduce an owner.

**Questions to resolve:**
- For each data store: which component owns it (controls schema, runs migrations, writes)?
- For each component that reads data it doesn't own: does it go through the owner's interface or directly to the store?
- Is there any data that two components need to write? If yes, the decomposition needs adjustment.

---

## Versioning and Migration

When a newer version of the system starts against older persistent data, what happens?

Default: the component that owns each data store is responsible for detecting and migrating
its schema. Migrations run on startup, before the component begins normal operation.
Migrations must be backward-compatible with the previous version for the duration of a
deployment window (so rollback is safe).

**Questions to resolve:**
- Which component runs migrations for each data store?
- Are migrations backward-compatible (is rollback safe)?
- What happens if migration fails — abort startup or start in degraded mode?

---

## Observability

Where do logs go? Which component owns the logging channel?

Logs must be sufficient to reconstruct what the system did, to whom, when, why, and with
what outcome — without reading source code.

**Action log**: every automated action the system takes on the user's behalf must be logged
with enough context to answer: what action, on what target, triggered by what, at what time,
with what result.

**Error log**: every failure with enough context to route it to the responsible component
and understand what state the system was in.

Default: one component owns the logging channel. Other components write structured log events
to it. The owning component is responsible for format, routing, and retention.

**Questions to resolve:**
- Which component owns the logging channel?
- What structured fields are required on every log entry?
- Where do logs go (local file, OS log, remote sink)?
- Which component surfaces errors to the user?

---

## Security Boundary

Where is authentication and authorization enforced?

Default: enforce at the system boundary — the point where external input enters the system.
Internal components trust that input has already been validated and authorized.
Do not re-validate authorization at every internal component.

Sensitive data (credentials, session keys, tokens) must be owned by exactly one component
and never passed beyond it in plain form.

**Questions to resolve:**
- Where is the system boundary (the outermost point where untrusted input enters)?
- Which component owns authentication and authorization?
- Which component holds sensitive credentials? Is it the only one that needs them?

---

## Error Propagation

How do errors travel between components?

An error in Component A reaches Component B's caller as: a named failure mode (from the
engineering baseline), a status code, an event, or a thrown exception — defined in the
interaction contract, not improvised at implementation time.

Default: errors are named (from the failure taxonomy established in the component spec),
propagated as return values or typed events, and never swallowed silently.

**Questions to resolve:**
- What is the mechanism for error propagation between each component pair (return value, exception, event)?
- Who is responsible for translating external failures into the internal failure taxonomy?
- Which errors are user-visible? Which component surfaces them?
