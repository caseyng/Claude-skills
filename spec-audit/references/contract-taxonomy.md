# Contract Taxonomy

Maps each spec-contract-methodology section to the types of MUST contracts it produces
and explains how to verify each type. Used during Phase 2 of spec-audit.

---

## §5 — Operations and API

**Contract types produced:**
- Parameter acceptance contracts: "MUST accept [type/format]"
- Output contracts: "result MUST contain [field]", "MUST return [type]"
- Pre/post condition contracts: "MUST [validate/check] before [operation]"

**How to verify:**
- Parameter types: check function/method signature. Type annotation or parameter declaration
  must match the spec's stated type.
- Required output fields: find the return value or output construction site. Every field marked
  MUST be present must appear in every code path that produces an output, not just the happy path.
- Pre-condition checks: find the validation call. It must occur before the guarded operation,
  not after. A check that occurs after the operation it guards is not a pre-condition check.

**Common violations:**
- Output type differs from spec (spec says string, implementation returns int)
- Required field absent from error path (field present on success, absent on failure, spec requires both)
- Pre-condition check is present but does not reject — logs and continues instead of returning an error

---

## §6 — Lifecycle

**Contract types produced:**
- State transition contracts: "MUST transition to [state] when [event]"
- Invalid state contracts: "MUST [reject/error] when called in [state]"
- Initialization contracts: "MUST [initialize X] before accepting calls"

**How to verify:**
- State transitions: find the state variable and the event handler. Verify the transition
  occurs on the correct event, not a different one.
- Invalid state handling: find the state check at the operation entry point. Verify it
  produces the correct named error (from §7), not a generic one.
- Initialization: find the initialization call and verify it occurs before any operation
  that depends on initialized state.

**Common violations:**
- Wrong state name in transition (different from the spec's named state)
- Generic error returned for invalid state instead of the named failure mode from §7
- Initialization happens lazily inside an operation rather than at startup (may work but violates the contract)

---

## §7 — Failure Taxonomy

**Contract types produced:**
- Naming contracts: the named failure mode MUST be the exact type produced on [condition]
- Condition contracts: failure mode MUST be produced when [condition] is met
- Information contracts: failure MUST carry [fields]
- Closed-set contracts: failure set MUST be closed (callers enumerate all possibilities)

**How to verify — naming:**
Find the code path for the stated condition. Verify the error/exception type, error code,
or failure enum value matches the spec's exact name. A different name for the same condition
is a violation — callers handle failures by name, and a wrong name means the caller's
error handler does not match.

**How to verify — condition:**
Find the condition check in the code. Verify it fires on exactly the condition the spec states,
not a broader or narrower condition. "Greater than MAX" is not the same as "greater than or
equal to MAX."

**How to verify — information carried:**
Find where the failure is constructed. Verify every field marked as required by the spec is
populated. Fields marked as "conditionally present" are acceptable if absent — only required
fields are audited as MUST.

**How to verify — closed set:**
Count the failure modes defined in §7. Count the failure types the implementation can produce
(error types, error codes, enum values). If the implementation can produce a failure not in §7,
the set is not closed — that is a violation (a caller cannot handle a failure they don't know exists).

**Common violations — highest value to audit:**
- Implementation raises a generic exception/error instead of the named failure mode
- Condition is checked at the wrong boundary (checks parameter, not the derived value)
- Required field in failure is always nil/empty in the implementation
- Implementation can produce failures not listed in §7 (open failure set when closed set is specified)

---

## §8 — Boundary Conditions

**Contract types produced:**
- Boundary behavior contracts: "MUST [produce failure / return value / clamp] when input [op] [boundary value]"
- Empty/null input contracts: "MUST [reject / return empty / handle] when input is [empty/null/zero]"
- Maximum/minimum contracts: "MUST NOT accept input [op] [value]"

**How to verify:**
Find the boundary check in the code. Verify:
1. The check uses the correct comparison operator (> vs >=, < vs <=)
2. The check uses the correct boundary value (off-by-one is a violation)
3. The action on boundary violation matches the spec (reject with named failure vs clamp vs return empty)

**Common violations:**
- Off-by-one: spec says "greater than MAX" but code checks "greater than or equal to MAX" (or vice versa)
- Wrong response: spec says return named failure, code silently truncates or clamps
- Missing check: boundary is mentioned in the spec but no corresponding check exists in the code

---

## §9 — Sentinel Values and Encoding Conventions

**Contract types produced:**
- Encoding contracts: "[field] MUST be encoded as [format]"
- Sentinel contracts: "[sentinel value] MUST represent [meaning]"
- Round-trip contracts: "serialized form MUST include [key]"

**How to verify:**
Find the serialization/deserialization code. Verify:
- Field names in the wire format match the spec's stated names exactly (case-sensitive)
- Sentinel values are the exact values in the spec, not approximations
- Round-trip: encode then decode produces the original value

**Common violations:**
- Field name in wire format differs from spec (camelCase vs snake_case when spec says one)
- Sentinel value is a magic number not matching the spec's stated value
- Decoder does not handle the sentinel value the spec defines

---

## §10 — Atomicity and State on Failure

**Contract types produced:**
- Rollback contracts: "state MUST be preserved [rolled back / unchanged] on failure"
- Partial state contracts: "MUST NOT leave [resource] in partially modified state"
- Idempotency contracts: "MUST be safe to retry" / "MUST NOT apply twice"

**How to verify:**
Find the code path where the relevant failure occurs. Verify:
- For rollback: is there a compensating action (transaction rollback, undo write) in the failure path?
- For atomicity: are all writes wrapped in a transaction or equivalent?
- For idempotency: is there a check-before-apply guard (check if already applied, then apply)?

**Common violations:**
- Failure path writes partial state then returns error (partial write)
- Transaction is started but never rolled back on failure
- No idempotency guard where spec requires one

---

## §11 — Ordering and Sequencing

**Contract types produced:**
- Delivery order contracts: "events MUST be delivered in [submission / arrival] order"
- Dependency order contracts: "[operation A] MUST complete before [operation B]"
- Queue ordering contracts: "MUST NOT reorder [items]"

**How to verify:**
For ordering contracts, find the queue or buffer data structure. If the structure is FIFO (array,
linked list, queue), ordering is structurally guaranteed. If the structure is unordered (map, set,
concurrent queue without ordering), verify explicit sorting or sequencing is applied.

**Common violations:**
- Using a HashMap/concurrent map where ordered delivery is required
- Launching parallel goroutines/tasks where sequential execution is required
- Sorting is applied on one path but not all paths

---

## §12 — Interaction Contracts

**Contract types produced:**
- Data passing contracts: "MUST pass [data] to [component]"
- Format contracts: "MUST pass [data] as [format/type]"
- Ownership contracts: "MUST NOT retain [resource] after passing to [component]"

**How to verify:**
Find the call site where the interaction occurs. Verify:
- The correct data is included in the call
- The format matches the spec
- If ownership is transferred, the sending component does not access the data after the call

**Common violations:**
- Data is partially passed (some required fields omitted)
- Format is wrong (passes raw struct instead of serialized form, or vice versa)
- Sending component caches a reference after transferring ownership

---

## §17 — Error Propagation

**Contract types produced:**
- Propagation contracts: "[failure from dependency] MUST be surfaced to caller as [named failure]"
- Wrapping contracts: "MUST NOT swallow [failure]" / "MUST include [cause] in propagated failure"
- Translation contracts: "external failure MUST be translated to [internal named failure]"

**How to verify:**
Find the call to the dependency and the error handling block. Verify:
- The error is not silently swallowed (no empty catch/if-err-ignore patterns)
- If translation is required, the correct internal failure name is produced (not a pass-through of the raw external error)
- If wrapping is required, the original cause is included

**Common violations — high value to audit:**
- Error is logged and nil/null returned (swallowed — caller sees success when failure occurred)
- External error passed through directly instead of translated to the internal failure taxonomy
- Failure wrapping omits the cause (caller cannot diagnose root cause)

---

## §18 — Observability Contract

**Contract types produced:**
- Event logging contracts: "MUST log [event] when [action] occurs"
- Field contracts: "log entry MUST include [fields]"
- Level contracts: "MUST log at [level] for [event type]"
- Timing contracts: "MUST log [before / after] [operation]"

**How to verify:**
Find the code path where the logged event occurs. Verify:
- A logging call exists in that path
- The required fields are populated in the log call
- The log level matches the spec
- Log timing (before vs after operation) matches the spec

**Common violations:**
- Logging call is missing from one of several code paths (logs on success, not on failure)
- Required field is not populated (nil/empty in the log call)
- Log is at the wrong level (INFO when spec says ERROR)
- Log occurs after the operation when spec requires before (or vice versa — matters for crash-consistency)

---

## §19 — Security Properties

**Contract types produced:**
- Enforcement contracts: "MUST [validate / authenticate / authorize] before [operation]"
- Storage contracts: "MUST store [credential] in [encrypted store]"
- Exposure contracts: "MUST NOT [expose / transmit] [data] via [channel]"
- Transport contracts: "MUST use [TLS / encrypted channel] for [communication]"

**How to verify:**
- Enforcement: find the validation call and confirm it is on the code path before the guarded
  operation. A validation call that the caller can bypass is not enforcement.
- Storage: find where credentials are persisted. Verify the storage mechanism matches the spec
  (EncryptedSharedPreferences, Keystore, etc.).
- Exposure: search for any code path that writes the sensitive data to a log, response body,
  or unencrypted store.
- Transport: find the network call setup and verify TLS/encryption is configured.

**Common violations:**
- Validation call exists but can be bypassed by a different call path
- Credentials written to plain SharedPreferences or a log statement
- Sensitive data included in error responses or log output
- HTTP connection used where HTTPS is required

---

## §22 — Assumptions

Assumptions are caller responsibilities — they define what the caller must ensure before
calling this component. They are NOT this component's obligations.

**Exception:** if the spec says "this component MUST validate [assumption] before proceeding
and MUST return [named failure] if violated" — that IS an auditable contract (it lives in §8
or §5, but may be referenced from §22). Audit it where it is stated.

**What not to audit:** "caller MUST provide a non-null value" — this is a caller obligation,
not an implementation contract. Do not create a MUST contract from it unless the spec also
says the component validates it.
