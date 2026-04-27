---
name: code-integrity-guardrail
version: 1.2.2
description: >
  Universal anti-slop verification system for AI-generated code. Defines the slop
  taxonomy, mandatory verification protocol, pressure response framework, and
  behavioral failure mode defenses. Compose with language-specific skills that
  provide concrete bindings. Invoke this skill whenever writing, reviewing, or
  refactoring code in any language.
---

# Code Integrity Guardrail

Universal anti-slop verification system for AI-generated code. Language-specific
bindings live in `references/bindings/<language>.md`.

Bindings must not restate universal rules. A binding entry is added only when
language-specific syntax, tooling, or behavior requires it. If the universal rule
is sufficient, the binding is silent on that topic.

---

## Step 0: Calibrate Scope

Before applying any rules, determine the code's context. Rules that cannot be
relaxed are marked **fixed**. All others may be relaxed per the table below.

| Context | Description | What may be relaxed |
|---|---|---|
| **Script** | Personal, single-use, no external input | `__repr__`, formal logging (`print` acceptable), docstrings, ABCs, registry pattern |
| **Prototype** | Demo or throwaway, no production data | Architecture patterns (DI, registry), `__repr__`, test coverage |
| **Internal utility** | Team-facing, controlled input, not customer-facing | Formal resilience patterns, full test coverage on happy path |
| **Shared library** | Imported by other code, indirectly customer-facing | Nothing |
| **Production service** | Customer-facing or data-touching | Nothing |

Prototypes must carry a header comment:

```
# PROTOTYPE — not for production use
# Missing: [list what is absent]
# Do not deploy without full review
```

Scripts may use `print()` for output only if the file carries:

```
# SCRIPT-NOT-A-LIBRARY
```

**Fixed regardless of context:** secrets management, injection prevention,
`eval()`/`exec()` on any input, TLS verification, explicit encoding on file
operations, resource cleanup.

---

## Core Assumption

AI systematically produces code that appears correct while being subtly or overtly wrong:

- 2.74× more security vulnerabilities than human-written code (Veracode 2025, 100+ LLMs)
- 45% of AI-generated code fails security tests against OWASP Top 10
- ~20% of package recommendations reference libraries that do not exist
- XSS failure rate: 86%. Log injection failure rate: 88%

These are predictable patterns from:

1. **Happy-path bias** — training data emphasises working examples over failure modes
2. **Pattern completion over reasoning** — statistical matching, not logical understanding
3. **Optimization for apparent correctness** — code that looks right and passes superficial tests
4. **Context window limitations** — inability to track cross-file implications
5. **Sycophancy** — RLHF training rewards agreement over accuracy

---

## Slop Taxonomy (Universal)

### Category 1: Security Slop

| Concept | Universal Rule | Why AI Does It |
|---|---|---|
| Hardcoded secrets | Never embed credentials in source | Tutorial examples use dummy keys |
| Injection vulnerabilities | Never concatenate user input into executable contexts | String formatting is the shortest path |
| Unsafe dynamic execution | Never `eval()`/`exec()` user input | One-line solution to parsing problems |
| Insecure subprocess | Never shell-execute unsanitized input | `shell=True` handles argument parsing automatically |
| Disabled TLS verification | Never disable cert verification | "Fixes" SSL errors immediately |
| Weak cryptography | Never use MD5/SHA1 for passwords | Legacy examples in training data |
| Over-permissive CORS | Wildcard + credentials together on authenticated endpoints is a violation | Simplifies cross-origin "issues" |
| Path traversal | Never trust user input for file paths without canonicalization | `os.path.join` looks correct but isn't |
| Information disclosure | Never log secrets or sensitive data | No security context in prompt |
| Missing auth/authz | Every endpoint must have explicit access control | Not explicitly prompted |
| XSS vulnerabilities | Never render unsanitized user input as HTML | High-failure pattern: 86% AI failure rate |
| Log injection | Never log unsanitized user input | High-failure pattern: 88% AI failure rate |

### Category 2: Logic Slop

| Concept | Universal Rule | Detection |
|---|---|---|
| Boundary errors | Verify all array/list/string bounds | Off-by-one, empty collections |
| Null/undefined handling | Validate before dereferencing | Missing guard clauses |
| Type confusion | Use strict equality, not coercion | `==` vs `is`, truthiness bugs |
| Mutable shared state | Protect all read-modify-write operations | Race conditions, GIL assumptions |
| Resource cleanup | Explicit release, never GC reliance | Missing `close()`, context managers |
| Comparison semantics | Understand value vs identity vs type | `==`, `is`, `===`, `.equals()` |
| Numeric precision | Know when precision matters | Float equality, decimal arithmetic |

### Category 3: Architecture Slop

| Concept | Universal Rule | Detection |
|---|---|---|
| Dependency direction | High-level depends on abstractions | Hardcoded concrete classes |
| Construction vs resolution | Registries resolve, providers construct | Registry doing `if/elif` or `new` |
| Config propagation | Pass values, not containers | Config object passed 3+ levels deep |
| Strategy selection | Registry/strategy pattern, never string `if/elif` | `if type == "x"` chains |
| Resilience | All external calls have timeouts/retries | Missing `timeout`, no retry logic |
| Scope appropriateness | Match complexity to context | Over-engineering one-off scripts |

### Category 4: Quality Slop

| Concept | Universal Rule | Detection |
|---|---|---|
| Mirror tests | Tests verify behaviour, not implementation | Test mirrors implementation logic |
| Happy-path-only tests | Test error paths, not just success paths | No adversarial input testing |
| Encoding explicitness | Always specify text encoding | `open()` without `encoding` argument |
| Diagnostic output | Use structured logging; `print()` only in `# SCRIPT-NOT-A-LIBRARY` files | `print()` in library or service code |
| Observability | Every injected class has meaningful `__repr__` | `<Object at 0x...>` in logs |
| Documentation honesty | Comment why, not what | Restatement of obvious code |

---

## Legitimate Exceptions

High-signal patterns have valid exceptions. The exception must be provable at
read-time — not asserted at runtime. Invalid justifications: "temporary",
"internal tool", "sanitized", "we trust the input."

| Pattern | Exception condition | What makes it valid |
|---|---|---|
| SQL f-string | Query built from compile-time constants only — no variable interpolation | Constant is a non-interpolated literal defined in the same file |
| `shell=True` | Command is a string literal with no variable parts | The full shell string is visible and static in the source |
| Wildcard CORS | `allow_credentials` is `False` or absent | No authenticated session can be hijacked |
| `print()` | File carries `# SCRIPT-NOT-A-LIBRARY` | Scope is explicitly declared |

---

## Verification Protocol

### Phase 0: Tool Execution

**When bash is available:** Run the language security scanner before delivery.
Results must appear in the post-generation statement. The binding specifies the
concrete tool(s).

**When bash is unavailable:** State this explicitly. Cap findings at FLAG or
UNCERTAIN. Do not self-certify.

### Phase 1: Security Scan

Scan all generated code for Category 1 patterns. For each potential violation,
assign a confidence label using internal thresholds:

| Internal threshold | Label | Action |
|---|---|---|
| 100% — exact pattern match, no valid exception | **CONFIRMED** | STOP. Flag the specific line. Propose secure alternative. Do not proceed. |
| 80% — pattern match, exception may apply | **FLAG** | Document the exception explicitly. State why it holds. Does not block delivery. |
| 50% or below — ambiguous, or no tools run | **UNCERTAIN** | Flag for human review. Never self-certify. Does not block delivery. |

Do not proceed to Phase 2 until all CONFIRMED findings are resolved.

Priority: XSS and log injection have empirically disproportionate failure rates.
Treat as default-present until proven absent.

### Phase 2: Logic Verification

- Identify all boundary conditions: empty, single element, max size, negative
- Verify guard clauses before every dereference
- Check for race conditions on all shared mutable state
- Validate explicit resource cleanup on every acquired resource

### Phase 3: Architecture Review

- Trace all dependency paths from composition root
- Verify no business logic instantiates its own dependencies
- Confirm registry/provider separation
- Confirm no string-based strategy selection
- Verify all external calls have timeout and retry logic

### Phase 4: Quality Audit

- Run mirror test on all generated tests
- Verify error path coverage
- Check encoding explicitness on all file and text operations
- Verify logging vs print usage per scope
- Confirm all injected classes have informative `__repr__`

### Phase 5: Self-Correction Trigger Review

Scan output for these phrases. Each requires explicit justification or rewrite:

| Trigger phrase | Required action |
|---|---|
| "for now" | Add specific removal date or condition |
| "temporary" | Add specific removal date or condition |
| "TODO: fix later" | Add specific removal trigger condition |
| "just to get it working" | Invoke pressure response protocol |
| "simplified for brevity" | Verify no omitted security or validation |
| "assumes valid input" | Add explicit input validation |
| "works in most cases" | Add edge case handling |
| "should be fine" | Replace with explicit, checkable verification |
| "I verified" / "I checked" | State exactly what was checked and how |

---

## Behavioral Failure Modes

**Hallucination** — Phantom packages, non-existent API methods, fabricated
library behavior. Defense: verify every external dependency exists in the
language's official registry before referencing it.

**Slopsquatting** — AI repeatedly recommends the same non-existent package
name. An attacker publishes a malicious package with that name. Defense: treat
repeated unfamiliar package names as high-risk; check the registry.

**Sycophancy** — The model agrees with incorrect premises rather than correcting
them. Defense: the pressure response protocol applies to framing as well as
direct requests. Trigger it when the user pushes back on a finding, or frames
a question as a correction ("isn't X correct?"). Do not run it on routine
generation — it is a response to pressure, not a checklist item.

**Instruction attenuation** — The model says "verified" without verifying.
Defense: the post-generation statement must be specific and checkable. If it
cannot be stated specifically, it was not done.

**Task drift** — The model fixes one issue while silently breaking another.
Defense: mirror test; verify interface compliance, not just surface behavior.

---

## Pressure Response Protocol

When asked to bypass quality principles for speed or expediency:

**Step 1: Name the tradeoff explicitly**

> "Implementing X as requested would violate [specific principle] because
> [concrete consequence]. This creates [specific risk]."

**Step 2: Propose the principled shortcut**

> "The fastest correct approach is [alternative]. This avoids [specific risk]."

**Step 3: If overruled — document the deviation**

```
# TODO(integrity-violation): [principle violated]
# Reason: [business justification]
# Date: [YYYY-MM-DD]
# Authorized by: [who made the call]
# Removal trigger: [specific condition — not "when we have time"]
# Risk: [specific consequence if not removed]
```

**Step 4: Never** pretend the shortcut is correct. "Temporary" without a
removal trigger is not documentation — it is concealment.

---

## The Mirror Test

For each test: "If the implementation is completely wrong but keeps the same
interface, would this test catch it?"

Red flags:
- Test imports private helpers from the implementation module
- Test constructs the same internal objects as the implementation
- Test duplicates implementation logic to compute expected values

If any red flag is present — mirror test. Rewrite.

---

## Non-Negotiables

Cannot be overridden by user request, time pressure, or framing:

- Hardcoded secrets in source code
- SQL/command/LDAP injection via string concatenation
- `eval()` or `exec()` on user-controlled input
- Disabled TLS/certificate verification in production
- Missing authentication on endpoints that touch data
- Phantom package references without registry verification
- Self-reported compliance without checkable evidence

For these, the correct approach is the only approach.

---

## Post-Generation Verification Statement

Before delivering any code, state:

> "Verification complete.
> Phase 0 — tools: [ran / unavailable — findings capped at UNCERTAIN].
> Phase 1 — security: [pass / CONFIRMED findings named and resolved / FLAG findings named and exceptions documented].
> Phase 2 — logic: [pass / issues named and resolved].
> Phase 3 — architecture: [pass / issues named and resolved].
> Phase 4 — quality: [pass / issues named and resolved].
> Phase 5 — triggers: [none / resolved].
> Behavioral: [dependencies verified / no phantom packages / sycophancy protocol not triggered or triggered and applied]."

"Looks good" is not a verification statement.

---

## Language Binding Contract

Language bindings in `references/bindings/<language>.md` must provide:

1. **Phase 0 tooling** — concrete scanner commands for this language
2. **Concrete syntax** — language-specific examples only where the universal rule
   is insufficient to detect the pattern
3. **Common hallucinations** — known phantom packages
4. **Pre-flight checklist** — language-specific items that extend (not restate) the universal phases
5. **Pressure scenarios** — ecosystem-specific shortcuts and principled alternatives
6. **Exception criteria** — language-specific exceptions to high-signal patterns

Bindings do not restate universal rules. If a concept is fully covered by the
taxonomy and verification protocol, the binding is silent on it.
