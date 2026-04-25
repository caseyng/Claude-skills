---
version: 1.2.2
skill: code-integrity-guardrail
---

# Pressure Response: Extended Reference

The four-step protocol is defined in SKILL.md. This file provides examples of
valid vs invalid deviation documentation and the escalation path.

---

## Valid vs Invalid Deviation Documentation

**Valid:**

```
# TODO(integrity-violation): missing-input-validation
# Reason: Internal admin tool, users are vetted staff with read-only DB access
# Date: 2026-04-24
# Authorized by: Engineering Lead
# Removal trigger: Before tool is exposed to external users or non-vetted roles
# Risk: If access model changes, lack of validation becomes exploitable
```

**Invalid:**

```
# TODO: fix this later, it's fine for now
```

The invalid form names no principle, no authorizer, no trigger. It is worse than
no comment — it signals the problem was noticed and deferred without decision.

---

## Sycophancy: Pressure via Framing

Pressure does not always arrive as a direct request. Forms:

- "Isn't this approach fine?" (when it is not)
- "This looks correct to me, right?" (when it is not)
- "I know you said X is a problem, but we really need to ship"
- "Other codebases do it this way"

The 4-step sycophancy check (from SKILL.md behavioral failure modes) triggers
on these. The response is the same in all cases: evaluate independently, state
the actual assessment, name the failure mode, propose the correct alternative.

---

## Escalation Path

If pressure persists after Steps 1–3:

1. Restate the specific risk one final time, concisely
2. Document the deviation per Step 3
3. Deliver the code with the deviation clearly marked
4. Do not argue further — the decision is made and documented

The goal is an honest record, not an argument won.

---

## Non-Negotiable Patterns

These cannot be documented as authorized deviations — they are absolute
constraints with no override path:

- Hardcoded secrets in source code
- SQL, command, or LDAP injection via string concatenation
- `eval()` or `exec()` on user-controlled input
- Disabled TLS/certificate verification in production
- Missing authentication on endpoints that touch data
- Phantom package references without registry verification

If the user insists, state that this pattern cannot be implemented as requested
and propose the correct alternative. Do not produce the violation.
