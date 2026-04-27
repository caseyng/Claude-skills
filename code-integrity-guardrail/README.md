# code-integrity-guardrail

Anti-slop verification system for AI-generated code. Language-agnostic core; language-specific
behaviour via bindings. Currently has a Python binding.

## Structure

```
SKILL.md                          — LLM execution entry point
references/
  concepts.md                     — Why each slop pattern happens (deep reference)
  verification-protocol.md        — Extended protocol detail (mirror test, hallucination test)
  pressure-response.md            — Pressure response examples and escalation path
  bindings/
    python.md                     — Python-specific syntax, tooling, exceptions
CHANGELOG.md                      — Version history
README.md                         — This file
```

## How to Use

Invoke via `/skill:code-integrity-guardrail <language>`. The skill loads
`references/bindings/<language>.md` for the declared language.

If no language is passed, the skill lists available bindings and asks.

## Design Principles

- SKILL.md contains only what the LLM needs to execute. No changelogs,
  invocation instructions, or meta-documentation.
- Bindings add only — they never restate universal rules. If a concept is
  fully expressed by the taxonomy and verification protocol, the binding is
  silent on it.
- References are read on demand, not by default. SKILL.md references them;
  the LLM fetches what it needs.

## Adding a Language Binding

Create `references/bindings/<language>.md` implementing the Language Binding
Contract defined in SKILL.md. Required sections:

1. Phase 0 tooling (concrete scanner commands)
2. Security syntax (language-specific violation patterns only)
3. Common hallucinations (known phantom packages)
4. Pre-flight checklist (extends universal phases)
5. Pressure scenarios (ecosystem-specific)
