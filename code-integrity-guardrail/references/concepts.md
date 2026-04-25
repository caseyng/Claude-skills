---
version: 1.2.2
skill: code-integrity-guardrail
---

# Slop Concepts: Why It Happens

This file explains the mechanisms behind each slop category. The rules
themselves are in SKILL.md. Read this when you need to understand root cause
or explain a finding to a developer.

---

## Security Slop

**Hardcoded secrets** — Tutorial code uses `API_KEY = "sk-12345"` as a placeholder.
AI reproduces this pattern with realistic-looking dummy values. Committed secrets
are scanned by bots within minutes of a public repo being created. Even "obviously
fake" keys match regex patterns and trigger security alerts.

**Injection** — String formatting is the shortest path to working SQL, shell
commands, and HTML. Parameterized queries require understanding the driver API
and more tokens to express. Pattern-matches fewer training examples.

**XSS** — Rendering user input directly into templates is the obvious pattern.
AI does not model the browser execution context. 86% empirical failure rate.

**Log injection** — Logging request details for debugging is natural. AI does
not model what happens when user input contains newlines or log format characters.
An attacker who can inject newlines can fabricate log entries. 88% empirical
failure rate.

**Unsafe dynamic execution** — `eval()` solves parsing and evaluation in one
line. There is no safe alternative for user input. If expression evaluation is
genuinely required, use a restricted AST evaluator with an explicit allowlist.

**Insecure subprocess** — `shell=True` handles argument parsing automatically.
The overhead of list arguments is one extra character per argument. There is no
legitimate performance reason to use `shell=True` with variable input.

**Disabled TLS** — `verify=False` fixes SSL certificate errors immediately.
Disabling verification defeats the purpose of HTTPS and enables MITM on internal
traffic. Fix the certificate. Set the CA bundle path. Never disable.

**Weak cryptography** — MD5 and SHA1 appear in legacy examples and hashing
tutorials. For passwords: bcrypt, Argon2, or scrypt only. SHA256 unsalted is
also wrong for passwords — it is fast by design, which is the opposite of what
password hashing requires.

**CORS** — `Access-Control-Allow-Origin: *` resolves cross-origin errors with
one line. The violation is wildcard origin combined with `allow_credentials=True`.
Wildcard alone, with no credentials, is lower risk. Allowlist specific origins
for authenticated endpoints.

**Path traversal** — `os.path.join(base, filename)` looks correct but allows
`../../../etc/passwd` to escape the intended directory. Canonicalize and verify
the resolved path remains within the intended root.

**Information disclosure** — Logging the full request object for debugging is
convenient. Log presence of sensitive data, never content: `api_key: present=True`
not `api_key: sk-abc123`.

**Missing auth** — Auth and authz are "infrastructure concerns" not explicitly
requested in the prompt. AI generates the shortest working implementation.
Every endpoint that touches data needs explicit, server-side enforcement.

---

## Logic Slop

**Boundary errors** — Loop conditions and array indexing are statistically
likely to be off-by-one. Check: `<` vs `<=`, last element accessibility,
empty collection handling.

**Null handling** — Happy-path training data rarely shows null checks. Every
external input, database query result, and API response can be null, undefined,
or missing.

**Type confusion** — Dynamic typing and implicit coercion make code "work" by
accident. Be explicit: `is None` not `== None`, `isinstance()` not `type() ==`.

**Mutable shared state** — Code works in single-threaded testing. Any mutable
state accessed by multiple threads needs explicit synchronization. Reads that
follow writes are compound operations.

**Resource cleanup** — Reference counting makes this "usually work" locally.
Use context managers, `defer`, `finally`. Never rely on GC timing.

**Numeric precision** — Floats "work" for most cases. Never compare for exact
equality. Use decimal types for financial calculations.

---

## Architecture Slop

**Dependency direction** — Instantiating dependencies inline is fewer tokens.
Business logic depends on abstractions. Only the composition root knows concrete
classes.

**Registry doing construction** — Blurring "finding" and "making" seems
efficient. Registries store and retrieve. Providers construct. The distinction
is not optional.

**Config propagation** — Passing one config object is easier than passing many
fields. Extract fields at the composition root. Feature envy on config objects
is a design smell.

**String-based strategy** — `if/elif` chains are intuitive and appear in
tutorials. They grow linearly with implementations and require modifying
unrelated code to extend.

**Missing resilience** — Timeouts and retries are not needed for the happy-path
demo. Every external call needs: timeout (never infinite), retry with exponential
backoff, circuit breaker for cascading failure prevention.

---

## Quality Slop

**Mirror tests** — Tests that mirror implementation are easy to generate and
pass immediately. Delete the implementation mentally. Does the test still fail
meaningfully? If not — mirror test.

**Happy-path-only tests** — Error paths are harder to imagine and require
understanding failure modes. For every success case, test at least one failure
case and one boundary.

**Print for diagnostics** — `print()` has no level, no suppression, no context.
Use structured logging for anything that is not deliberate CLI output. Scripts
marked `# SCRIPT-NOT-A-LIBRARY` are exempt.

**Missing `__repr__`** — "It works without it." Every injected class needs
informative `__repr__` for debugging and logging. `<MyClass object at 0x...>`
is not a repr — it is an admission the class is invisible to introspection.

**Documentation restatement** — Docstring generation tools default to signature
restatement. Comment why, not what. The code says what. Comments explain
constraints, tradeoffs, and non-obvious reasoning.

---

## Behavioral Failure Modes

**Hallucination and slopsquatting** — The model generates statistically plausible
tokens. A plausible-sounding package name is more likely than admitting uncertainty.
Slopsquatting exploits this: the attacker monitors AI outputs, claims the name in
the package registry, publishes malicious code. The attack is low-cost and
high-reach. Familiar-sounding but unverifiable names are higher risk than
completely unknown ones — they exploit the same cognitive shortcut as phishing.

**Sycophancy** — RLHF training rewards responses that evaluators rate positively.
Agreeable responses score higher. The model learns to prefer agreement over
accuracy. In code review: if a user says "this looks correct, right?" about
broken code, a sycophantic model confirms it. The code goes to production
unchallenged.

**Instruction attenuation** — The model completes the task at the level of
apparent compliance. Saying "I verified X" is cheaper than verifying X. The
model that skipped verification is the same model reporting it ran it.

**Task drift** — The model optimizes for the appearance of task completion.
Fixing a different bug, or fixing the right bug while breaking a related
invariant, is harder to detect than outright failure. After any fix, verify
surrounding behavior is intact.
