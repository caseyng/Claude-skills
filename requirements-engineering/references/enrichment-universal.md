# Universal Enrichment Checklist

These concerns apply to every software project regardless of platform.
For each item not addressed in the raw intent, present a recommended default and ask for confirmation.
Do not ask open questions — propose and ask for override.

---

## Security

**Authentication and authorisation**
Who can use this system and what can they do?
Default recommendation: if it handles personal or sensitive data, authentication is required.
If it is a single-user personal tool, OS-level or biometric lock is sufficient.

**Data sensitivity**
What data does the system handle? Is any of it sensitive (personal, financial, medical, credentials)?
Default: sensitive data must not be stored in plaintext. Credentials go in secure storage.

**Network communication**
Does the system make network calls? Is TLS enforced? Is there any case where plaintext is acceptable?
Default: TLS only. No cleartext traffic.

**Attack surface**
What components are exposed to untrusted input? User-supplied strings, network responses, file imports.
Default: validate at the boundary, trust nothing from outside the system.

---

## Observability

**Logging**
What events should be logged? What level of detail? Where do logs go?
Default: errors and significant state transitions logged. Debug logs in development only.

**Error reporting**
How are errors surfaced to the user? How are errors surfaced to the developer?
Default: user sees a clear actionable message. Developer sees stack trace and context.

**Audit trail**
Is a record of actions required? Who took what action, when?
Default: any system that acts autonomously on the user's behalf should log every automated action.

---

## Failure and Error Strategy

**What happens when the system fails?**
Does it fail silently, surface an error, retry, or fall back to a safe state?
Default: fail visibly. Silent failures are defects.

**Recovery**
After a failure, can the system recover automatically? Does it need user intervention?
Default: automatic recovery for transient failures (network, timeout). User intervention for
persistent failures (bad credentials, config errors).

**Dependency unavailability**
What happens if an external service the system depends on is unavailable?
Default: surface the failure clearly. Do not silently degrade.

---

## Data

**What is stored and for how long?**
What data does the system persist? Is there a retention policy?
Default: state what the system stores explicitly. If no policy is needed, say so.

**Data ownership and portability**
Who owns the stored data? Can the user export or delete it?
Default: for a personal-use system, the user owns all data. Deletion is always available.

**Backup and recovery**
Is the stored data recoverable if the device is lost or reset?
Default: for a personal tool, document whether data survives a device reset and whether
this is acceptable.

---

## Users and Usage

**Who uses this system?**
Single user or multiple users? Concurrent users?
Default: state explicitly. Single-user systems have different design constraints than multi-user.

**Usage frequency and scale**
How often is the system used? What load does it need to handle?
Default: personal tools are low-scale by design. State this explicitly — it is a constraint
that simplifies many design decisions.

**Accessibility**
Are there accessibility requirements? Screen readers, contrast, font size?
Default: follow platform accessibility guidelines as a minimum.

---

## External Dependencies

**Third-party services**
Does the system call external APIs? Which ones? What happens if they are unavailable or changed?
Default: document each external dependency. Rate limits, API key management, and failure
behaviour must be addressed.

**Updates and compatibility**
Does the system depend on specific versions of external components? What is the update strategy?
Default: document the minimum supported versions. Dependency pinning is the default for
production systems.

---

## Operational Concerns

**Deployment and distribution**
How does the system reach the user? App store, direct install, self-hosted?
Default: state the distribution mechanism explicitly. It affects many design decisions.

**Updates**
How does the system get updated after initial deployment?
Default: document the update mechanism, even if it is manual.

**Configuration**
What can the user configure? Where is configuration stored?
Default: user-configurable settings go in persistent, user-accessible storage. Hard-coded
values that should be configurable are a maintenance liability.
