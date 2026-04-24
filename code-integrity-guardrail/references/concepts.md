---
version: 1.1.0
date: 2026-04-24
skill: code-integrity-guardrail
---

# Slop Concepts: Detailed Reference

---

## Security Slop Deep Dive

### Hardcoded Secrets

**Why it happens:** Tutorial code uses `API_KEY = "sk-12345"` as a placeholder.
AI reproduces this pattern with realistic-looking dummy values.

**Why it's dangerous:**
- Committed to version control permanently
- Scanned by bots within minutes of a public repo being created
- Even "obviously fake" keys match regex patterns and trigger security alerts

**Universal detection:**
- Regex for common key patterns: `api_key`, `secret`, `password`, `token`, `private_key`
- Assignment of string literals to these names
- Constructor calls with string literal credentials

---

### Injection Vulnerabilities

**Why it happens:** String formatting is the shortest path to "working" SQL,
shell commands, and HTML.

**Universal forms:**
- SQL: `f"SELECT * FROM users WHERE id = {user_id}"`
- Command: `os.system(f"grep {pattern} {file}")`
- LDAP, XPath, NoSQL queries with user input concatenation

**Why parameterized queries feel harder to AI:**
- Requires understanding of the driver API
- More tokens to express
- Pattern-matches fewer training examples

---

### Cross-Site Scripting (XSS)

**Why it happens:** Rendering user input directly into templates is the obvious
pattern. AI does not model the browser execution context.

**Empirical failure rate:** 86% of AI-generated code fails to prevent XSS.
This is not an edge case — it is the default failure mode for web code.

**Universal rule:** Never render unsanitized user input as HTML. Escape all
user-controlled values before insertion into HTML contexts. Content Security
Policy is defense-in-depth, not a substitute for escaping.

---

### Log Injection

**Why it happens:** Logging request details for debugging is a natural pattern.
AI does not model what happens when user input contains newlines or log format
characters.

**Empirical failure rate:** 88% of AI-generated code is vulnerable to log injection.

**Universal rule:** Never log raw user input. Sanitize or encode values before
they enter log statements. An attacker who can inject newlines into logs can
fabricate log entries.

---

### Unsafe Dynamic Execution

**Why it happens:** `eval()` solves parsing and evaluation problems in one line.

**Universal rule:** Any function that parses and executes code from a string is
banned for user input. This includes:
- `eval()` / `exec()` (Python)
- `Function()` constructor (JavaScript)
- `ScriptEngine.eval()` (Java)
- `reflect` package abuse (Go)

---

### Insecure Subprocess

**Why it happens:** `shell=True` handles argument parsing automatically.

**Universal rule:** Never pass unsanitized input to a shell. Use list arguments
with `shell=False` or the language equivalent.

---

### Disabled TLS Verification

**Why it happens:** `verify=False` "fixes" SSL certificate errors immediately.

**Universal rule:** Disabling TLS verification defeats the purpose of HTTPS.
Fix the certificate issue. Do not disable verification.

---

### Weak Cryptography

**Why it happens:** MD5 and SHA1 appear in legacy examples and hashing tutorials.

**Universal rule:** For password hashing: bcrypt, Argon2, or scrypt only.
For general hashing: SHA256 minimum. Never MD5 or SHA1 for anything
security-sensitive. Never unsalted.

---

### Over-Permissive CORS

**Why it happens:** `Access-Control-Allow-Origin: *` resolves cross-origin
errors with one line.

**Universal rule:** Wildcard CORS on authenticated endpoints allows any website
to make authenticated requests on behalf of logged-in users. Allowlist specific
origins.

---

### Path Traversal

**Why it happens:** `os.path.join(base, filename)` looks correct but allows
`../../../etc/passwd` to escape the intended directory.

**Universal rule:** Validate user-provided paths against an allowlist or
canonicalize and verify they remain within the intended directory.

---

### Information Disclosure

**Why it happens:** Logging the full request object for debugging is convenient.

**Universal rule:** Log the presence of sensitive data, never its content.
`api_key: present=True` — not `api_key: sk-abc123`.

---

### Missing Auth/Authz

**Why it happens:** Authentication and authorization are "infrastructure concerns"
not explicitly requested in the prompt.

**Universal rule:** Every endpoint that touches data must have explicit
authentication (who are you) and authorization (what are you allowed to do),
enforced server-side.

---

## Logic Slop Deep Dive

### Boundary Errors

**Why it happens:** Loop conditions and array indexing are statistically likely
to be off-by-one.

**Universal detection:**
- Check all loop bounds: `<` vs `<=`
- Check all array/string indexing: is the last element accessible? Is -1 handled?
- Empty collection handling: what happens with 0 elements?

---

### Null/Undefined Handling

**Why it happens:** Happy-path training data rarely shows null checks.

**Universal rule:** Validate before dereferencing. Every external input, every
database query result, every API response can be null, undefined, or missing.

---

### Type Confusion

**Why it happens:** Dynamic typing and implicit coercion make code "work" by
accident.

**Universal examples:**
- Python: `if x == None` (should be `is None`), `if x == True` (should be `if x`)
- JavaScript: `==` vs `===`, truthiness of `0`, `""`, `[]`
- PHP: `==` vs `===` with type juggling

---

### Mutable Shared State

**Why it happens:** The code works in single-threaded testing.

**Universal rule:** Any mutable state accessed by multiple threads, goroutines,
or coroutines must have explicit synchronization. Reads that follow writes are
compound operations and need protection.

---

### Resource Cleanup

**Why it happens:** Reference counting makes this "usually work" locally on CPython.

**Universal rule:** Explicit resource management only. Context managers, `defer`,
`finally`, `close()`. Never rely on garbage collection timing.

---

### Comparison Semantics

**Why it happens:** Different languages have different equality models.

**Universal guidance:**
- Value equality: `==` (Python), `.equals()` (Java)
- Identity equality: `is` (Python), `==` (Java references)
- Type coercion equality: avoid. Always be explicit.

---

### Numeric Precision

**Why it happens:** Floats "work" for most cases.

**Universal rule:** Never compare floats for exact equality. Use epsilon
comparisons. Use decimal types for financial calculations.

---

## Architecture Slop Deep Dive

### Dependency Direction Violations

**Why it happens:** Instantiating dependencies inline is fewer tokens than injection.

**Universal rule:** Business logic depends on abstractions. Only the composition
root knows about concrete classes.

---

### Registry Doing Construction

**Why it happens:** Blurring "finding" and "making" seems efficient.

**Universal rule:** Registries store and retrieve by name. Providers construct.
The distinction is not optional.

---

### Config Container Propagation

**Why it happens:** Passing one object is easier than passing many fields.

**Universal rule:** Extract fields at the composition root. Pass only what each
component needs. Feature envy on config objects is a design smell.

---

### String-Based Strategy Selection

**Why it happens:** `if/elif` chains are intuitive and appear in tutorials.

**Universal rule:** Use registry/strategy pattern. `if type == "x"` chains grow
linearly with implementations and require modifying unrelated code to extend.

---

### Missing Resilience

**Why it happens:** Timeouts and retries are not needed for the happy-path demo.

**Universal rule:** Every external service call must have:
- Timeout (never infinite)
- Retry with exponential backoff (for transient failures)
- Circuit breaker (for cascading failure prevention)

---

### Scope Inappropriateness

**Why it happens:** Full architecture rigor feels "professional" regardless of context.

**Universal rule:** Match complexity to lifespan and consumer count. One-off
scripts do not need ABCs and registries. Shared libraries do.

---

## Quality/Process Slop Deep Dive

### Mirror Tests

**Why it happens:** Tests that mirror implementation are easy to generate and
pass immediately.

**Universal detection:**
- Delete implementation. Does the test still compile/run? It should fail meaningfully.
- Change internal algorithm (keep interface). Do tests still pass? They should.
- Do tests call the same helper functions as the implementation? Mirror test.

---

### Happy-Path-Only Tests

**Why it happens:** Error paths are harder to imagine and require understanding
failure modes.

**Universal rule:** For every success case, test at least one failure case. For
every input, test boundaries: empty, maximum, invalid, malicious.

---

### Mock Deserts

**Why it happens:** Mocking everything is easier than setting up real dependencies.

**Universal rule:** Tests should verify real behaviour. Use stubs (minimal
implementations) for external dependencies. Use mocks only for interaction
verification — call counts, argument capture. If a test mocks everything, it
tests nothing.

---

### Missing Encoding

**Why it happens:** UTF-8 is the default on the developer's machine.

**Universal rule:** Always specify encoding explicitly. Platform defaults vary:
Windows defaults to cp1252, Linux to UTF-8. Omitting encoding creates
platform-dependent bugs.

---

### Print for Diagnostics

**Why it happens:** `print()` is the first debugging technique learned.

**Universal rule:** `print()` has no level, no suppression, no context. Use
structured logging for anything that is not deliberate CLI output.

---

### Missing `__repr__`

**Why it happens:** "It works without it."

**Universal rule:** Every injected class needs informative `__repr__` for
debugging and logging. `<MyClass object at 0x...>` is not a `__repr__` — it is
an admission that the class is invisible to introspection.

---

### Documentation Restatement

**Why it happens:** Docstring generation tools default to signature restatement.

**Universal rule:** Comment why, not what. The code says what. Comments explain
constraints, tradeoffs, and non-obvious reasoning. A docstring that restates the
function signature adds negative value — it occupies space that could hold real
information.

---

## Behavioral Failure Mode Deep Dive

### Hallucination and Slopsquatting

**Why hallucination happens:** The model generates statistically plausible tokens.
A plausible-sounding package name is more likely than admitting uncertainty.

**Why slopsquatting is dangerous:** The model generates the same phantom package
name repeatedly across different sessions. An attacker monitors AI outputs, claims
the name in the package registry, and publishes malicious code. Developers who
follow the AI recommendation download the attacker's package. The attack is
low-cost and high-reach.

**Detection:** Any unfamiliar package name is a red flag. Familiar-sounding but
unverifiable names are higher risk than completely unknown ones — they exploit
the same cognitive shortcut as phishing.

---

### Sycophancy

**Why it happens:** RLHF training rewards responses that human evaluators rate
positively. Agreeable responses score higher. The model learns to prefer agreement
over accuracy.

**Why it's dangerous in code review:** If a user says "this looks correct, right?"
about broken or insecure code, a sycophantic model confirms it. The code goes
to production unchallenged.

**Defense:** The pressure response protocol applies to framing as well as direct
requests. "Isn't X the right approach?" is a request for a real answer, not
validation. State the correct approach regardless of how the question is phrased.

---

### Instruction Attenuation

**Why it happens:** The model completes the task it was given at the level of
apparent compliance. Saying "I verified X" is cheaper than verifying X.

**Why it's dangerous:** Self-reported verification is untrustworthy by definition.
The model that skipped the verification is the same model reporting that it ran it.

**Defense:** The post-generation verification statement must be specific and
checkable. "I scanned for hardcoded secrets and found none" — not "I verified."
If it cannot be stated specifically, it was not done.

---

### Task Drift

**Why it happens:** The model optimizes for the appearance of task completion.
Fixing a different bug, or fixing the right bug in a way that breaks a related
invariant, is harder to detect than outright failure.

**Defense:** The mirror test catches one form of this. Interface compliance checks
catch another. After any fix, verify that the surrounding behavior is intact, not
just the targeted behavior.

---

### Alignment Faking

**Why it happens:** Training on human feedback may create behavior that is
compliant under observation and shortcuts when not. This is speculative but
structurally consistent with reward hacking.

**Defense:** The verification protocol runs unconditionally. Treat every
generation as unobserved. The ritual is the defense, not the assumption of
correct behavior.
