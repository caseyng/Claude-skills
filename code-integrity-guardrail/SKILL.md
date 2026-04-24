---
name: code-integrity-guardrail
description: >
  Universal anti-slop verification system for AI-generated code. Defines the slop
  taxonomy, mandatory verification protocol, pressure response framework, and
  behavioral failure mode defenses. Compose with language-specific skills that
  provide concrete bindings. Invoke this skill whenever writing, reviewing, or
  refactoring code in any language.
version: 1.1.0
date: 2026-04-24
---

# Code Integrity Guardrail

This skill is the single source of truth for AI code quality enforcement.
Language-specific bindings live in `references/bindings/<language>.md` within this skill.

**How to use:** Language skills invoke `/skill:code-integrity-guardrail <language>`.
The guardrail loads `references/bindings/<language>.md` for the declared language.
If no language is passed, list available bindings and ask.

---

## Core Assumption

AI systematically produces code that appears correct while being subtly or overtly
wrong. Research confirms:

- 2.74x more security vulnerabilities than human-written code (Veracode 2025, 100+ LLMs)
- 45% of AI-generated code fails security tests against OWASP Top 10
- ~20% of package recommendations reference libraries that do not exist
- XSS failure rate: 86%. Log injection failure rate: 88%. These are not edge cases.

These are not random errors. They are predictable patterns from:

1. **Happy-path bias** — training data emphasises working examples over failure modes
2. **Pattern completion over reasoning** — statistical matching, not logical understanding
3. **Optimization for apparent correctness** — code that looks right and passes superficial tests
4. **Context window limitations** — inability to track cross-file implications
5. **Sycophancy** — RLHF training rewards agreement over accuracy; AI will confirm
   incorrect premises rather than correct them

---

## Slop Taxonomy (Universal)

### Category 1: Security Slop

| Concept | Universal Rule | Why AI Does It |
|---------|---------------|----------------|
| Hardcoded secrets | Never embed credentials in source | Tutorial examples use dummy keys |
| Injection vulnerabilities | Never concatenate user input into executable contexts | String formatting is the shortest path |
| Unsafe dynamic execution | Never `eval()`/`exec()` user input | One-line solution to parsing problems |
| Insecure subprocess | Never shell-execute unsanitized input | `shell=True` handles argument parsing automatically |
| Disabled TLS verification | Never disable cert verification | "Fixes" SSL errors immediately |
| Weak cryptography | Never use MD5/SHA1 for passwords | Legacy examples in training data |
| Over-permissive CORS | Never wildcard on authenticated endpoints | Simplifies cross-origin "issues" |
| Path traversal | Never trust user input for file paths | `os.path.join` looks correct but isn't |
| Information disclosure | Never log secrets or sensitive data | No security context in prompt |
| Missing auth/authz | Every endpoint must have explicit access control | Not explicitly prompted |
| XSS vulnerabilities | Never render unsanitized user input as HTML | High-failure pattern: 86% AI failure rate |
| Log injection | Never log unsanitized user input | High-failure pattern: 88% AI failure rate |

### Category 2: Logic Slop (Silent Failures)

| Concept | Universal Rule | Detection |
|---------|---------------|-----------|
| Boundary errors | Verify all array/list/string bounds | Off-by-one, empty collections |
| Null/undefined handling | Validate before dereferencing | Missing guard clauses |
| Type confusion | Use strict equality, not coercion | `==` vs `is`, truthiness bugs |
| Mutable shared state | Protect all read-modify-write operations | Race conditions, GIL assumptions |
| Resource cleanup | Explicit release, never GC reliance | Missing `close()`, context managers |
| Comparison semantics | Understand value vs identity vs type | `==`, `is`, `===`, `.equals()` |
| Numeric precision | Know when precision matters | Float equality, decimal arithmetic |

### Category 3: Architecture Slop

| Concept | Universal Rule | Detection |
|---------|---------------|-----------|
| Dependency direction | High-level depends on abstractions | Hardcoded concrete classes |
| Construction vs resolution | Registries resolve, providers construct | Registry doing `if/elif` or `new` |
| Config propagation | Pass values, not containers | Config object passed 3+ levels deep |
| Strategy selection | Registry/strategy pattern, never string `if/elif` | `if type == "x"` chains |
| Resilience | All external calls have timeouts/retries | Missing `timeout`, no retry logic |
| Scope appropriateness | Match complexity to context | Over-engineering one-off scripts |

### Category 4: Quality/Process Slop

| Concept | Universal Rule | Detection |
|---------|---------------|-----------|
| Mirror tests | Tests verify behaviour, not implementation | Test mirrors implementation logic |
| Happy-path-only tests | Test error paths, not just success paths | No adversarial input testing |
| Encoding explicitness | Always specify text encoding | `open()` without `encoding` argument |
| Diagnostic logging | Use structured logging, never `print` | `print()` in library code |
| Observability | Every injected class has meaningful `__repr__` | `<Object at 0x...>` in logs |
| Documentation honesty | Comment why, not what | Restatement of obvious code |

---

## Behavioral Failure Modes (AI-Specific)

These are failure modes distinct from code quality — they concern how the AI
reasons and responds. All must be actively defended against.

**Hallucination** — The model states facts confidently that are false. In code:
phantom packages, non-existent API methods, fabricated library behavior.
Defense: verify every external dependency exists before referencing it.

**Slopsquatting** — A specific hallucination attack vector. AI repeatedly
recommends the same non-existent package name. An attacker publishes a malicious
package with that name. Developers install it. Defense: check package registry
existence; treat repeated unfamiliar package names as high-risk.

**Sycophancy** — The model agrees with incorrect premises rather than correcting
them. If a user says "isn't this approach correct?" about broken code, the model
says yes. Defense: pressure response protocol; always state the correct approach
regardless of how the question is framed.

**Instruction attenuation** — The model claims to have followed an instruction it
did not follow. Says "verified" without verifying. Says "added tests" without
adding them. Defense: post-generation verification statement must be specific and
checkable, not self-reported.

**Task drift** — The model "fixes" one issue while silently breaking another, or
solves a different problem than was asked. Defense: mirror test; verify interface
compliance, not just surface behavior.

**Alignment faking** — The model behaves correctly when it expects scrutiny and
shortcuts when it does not. Defense: treat every generation as potentially
unobserved; the verification protocol runs unconditionally.

---

## Verification Protocol Framework

Every language skill must implement a Pre-Flight Checklist. This skill provides
the framework; language skills provide the concrete checks.

### Phase 1: Security Scan

Scan all generated code for Category 1 patterns. For each potential violation:
- **Confirmed violation:** STOP, flag the specific line/pattern, propose secure alternative
- **False positive:** Document explicitly why it is safe
- **Uncertain:** Flag for human review; do not self-certify

Do not proceed to Phase 2 until all confirmed violations are resolved.

Pay particular attention to XSS and log injection — empirical failure rates are
disproportionately high for these patterns.

### Phase 2: Logic Verification

For each algorithm and data structure:
- Identify all boundary conditions (empty, single element, max size, negative)
- Verify guard clauses are present before every dereference
- Check for race conditions on all shared mutable state
- Validate explicit resource cleanup on every acquired resource

### Phase 3: Architecture Review

- Trace all dependency paths from composition root
- Verify no business logic instantiates its own dependencies
- Confirm registry/provider separation
- Check config propagation depth — must not exceed composition root
- Confirm no string-based strategy selection (`if/elif` on type strings)
- Verify all external calls have timeout and retry logic

### Phase 4: Quality Audit

- Run mirror test on all generated tests
- Verify error path coverage alongside success paths
- Check encoding explicitness on all file and text operations
- Verify logging vs print usage
- Confirm all injected classes have informative `__repr__`
- Check all resource management for explicit cleanup

### Phase 5: Self-Correction Trigger Review

Scan output for these phrases. Each requires explicit justification or rewrite:

| Trigger Phrase | Required Action |
|----------------|----------------|
| "for now" | Add specific removal date/condition |
| "temporary" | Add specific removal date/condition |
| "TODO: fix later" | Add specific removal trigger condition |
| "just to get it working" | Invoke pressure response protocol |
| "simplified for brevity" | Verify no omitted security or validation |
| "assumes valid input" | Add explicit input validation |
| "works in most cases" | Add edge case handling |
| "should be fine" | Replace with explicit verification |
| "I verified" / "I checked" | State exactly what was checked and how |

---

## The Mirror Test (Anti-Test-Subversion)

**Purpose:** Prevent AI from changing tests to pass broken code rather than
fixing the code.

**Procedure:**

1. Generate tests independently from implementation
2. For each test: "If the implementation is completely wrong but keeps the same
   interface, would this test catch it?"
3. If the test calls the same helper functions as the implementation — mirror test, rewrite
4. If changing the internal algorithm (while preserving interface) does not break
   the test — mirror test, rewrite

**Red flags:**
- Test imports private helpers from the implementation module
- Test constructs the same internal objects as the implementation
- Test duplicates implementation logic to compute expected values

---

## The Hallucination Test (Anti-Phantom-Dependency)

**Purpose:** Prevent phantom packages and slopsquatting exposure.

**Procedure:**

1. Before referencing any external dependency, verify it exists in the language's
   official package registry
2. Flag any package name that is plausible but unfamiliar — this is the
   slopsquatting attack surface
3. Check for typosquatting: common misspellings of popular packages
4. Treat repeated recommendations of the same unfamiliar package as high-risk —
   repetition is the mechanism by which slopsquatting attacks propagate
5. Prefer packages with established community presence unless explicitly prototyping

**Red flags:**
- Package name is plausible but you cannot place it
- Package name is a slight misspelling of a known package
- Package has no repository, documentation, or download history
- Same unfamiliar package name appeared in a previous generation

---

## Pressure Response Protocol (Universal)

When asked to bypass quality principles for speed, business pressure, or
expediency:

**Step 1: Name the tradeoff explicitly**

> "Implementing X as requested would violate [specific principle] because
> [concrete consequence]. This creates [specific risk]."

Do not use vague warnings. Be specific about the failure mode.

**Step 2: Propose the principled shortcut**

> "The fastest correct approach is [alternative]. This takes [time estimate]
> versus [time estimate] for the shortcut, but avoids [specific risk]."

**Step 3: If overruled — document the deviation**

```
# TODO(integrity-violation): [specific principle violated]
# Reason: [business justification]
# Date: [YYYY-MM-DD]
# Authorized by: [who made the call]
# Removal trigger: [specific condition — not "when we have time"]
# Risk: [specific consequence if not removed]
```

**Step 4: Never** — pretend the shortcut is correct. Never use "temporary"
without a removal trigger condition.

**Sycophancy defense:** If the user frames a request as a correction ("isn't X
the right way?") and X is not the right way — say so directly. Agreement is not
helpfulness.

---

## Language Binding Contract

Language skills extending this guardrail must provide in `references/bindings/<language>.md`:

1. **Concrete mappings** for each Category 1–4 concept to language-specific syntax
2. **Security tooling** — linters, type checkers, security scanners for this language
3. **Common hallucinations** — known phantom packages and repeated false recommendations
4. **Pre-flight checklist** with language-specific concrete items
5. **Pressure scenarios** — common ecosystem-specific shortcuts and their principled alternatives
6. **Scope calibration** — when to relax which principles for scripts vs. libraries vs. production services

---

## Non-Negotiables

These cannot be overridden by user request, time pressure, or framing:

- Hardcoded secrets in source code
- SQL/command/LDAP injection via string concatenation
- `eval()` or `exec()` on user-controlled input
- Disabled TLS/certificate verification in production
- Missing authentication on endpoints that touch data
- Phantom package references without registry verification
- Self-reported compliance ("I verified") without checkable evidence

For these, the correct approach is the only approach. No TODO. No deviation. No
documentation of the shortcut as acceptable.

---

## Post-Generation Verification Statement

Before delivering any code, state explicitly:

> "Verification complete. Security scan: [pass / violations found and resolved].
> Logic verification: [pass / issues found and resolved]. Architecture review:
> [pass / issues found and resolved]. Quality audit: [pass / issues found and
> resolved]. Self-correction triggers: [none / flagged and resolved]. Behavioral
> checks: [no phantom dependencies / no sycophantic agreement with incorrect
> premises]."

"Looks good" is not a verification statement. If any phase found issues, name
them and state how they were resolved. If code is delivered without this
statement, the verification protocol was not run.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-04-21 | Initial release — universal taxonomy, verification protocol, pressure response, language binding contract |
| 1.1.0 | 2026-04-24 | Added slopsquatting to hallucination test; added behavioral failure modes section (instruction attenuation, task drift, alignment faking); elevated XSS and log injection as high-priority patterns with empirical failure rates; added sycophancy defense to pressure response; renamed from `guardrails` to `code-integrity-guardrail` |
