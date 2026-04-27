# claude-skills

Reusable Claude skills built for one purpose: making AI output reliable enough to depend on.

Most Claude skills automate tasks. These enforce standards. The difference matters when the output has to hold up — in production, under edge cases, across sessions.

Each skill is a `SKILL.md` file loaded by Claude when invoked as `/skill:<name>`. A `README.md` explains each skill's purpose and design. Install what you need.

---

## Skills

### [code-integrity-guardrail](./code-integrity-guardrail/)

AI-generated code has a characteristic failure mode: it looks right, passes the test, and contains slop — mutable defaults, bare exception catches, missing encodings, phantom packages. This skill is a universal verification layer that runs before any code is accepted. It defines a slop taxonomy, a mandatory verification protocol, and a pressure response framework for when shortcuts are tempting. Language-specific bindings extend it without restating it.

Invoke as `/skill:code-integrity-guardrail <language>`. Currently has a Python binding.

---

### [python-engineering](./python-engineering/)

AI-generated Python code has a characteristic failure mode: it works, and it's wrong. It passes the test, misses the design. This skill enforces senior engineering standards — dependency inversion, proper ABCs, composition roots, registry patterns — so Claude codes like an engineer who has to maintain it, not one who just has to ship it.

**Requires `code-integrity-guardrail python`.** The skill loads the guardrail automatically before proceeding. The guardrail handles universal code quality checks; this skill handles Python-specific design. They are not interchangeable — both are needed.

Invoke as `/skill:python-engineering`.

---

### [spec-contract-methodology](./spec-contract-methodology/)

LLMs are probabilistic. Without a precise specification, they fill gaps with assumptions — and those assumptions become bugs. This skill enforces a 23-section specification methodology rigorous enough that any engineer, in any language, could rebuild the system without reading the original code. Structural decisions, failure modes, boundary conditions, and extension contracts all have named homes. A spec that passes verification has zero blocking gaps — every ambiguity an implementor would reach for the code to resolve is eliminated first.

Invoke as `/skill:spec-contract-methodology`.

---

### [session-handoff](./session-handoff/)

Claude has no memory between sessions. Most handoff tools preserve state — where you were, what was done. This one preserves reasoning: why decisions were made, what was tried and rejected, where Claude's thinking was wrong and you corrected it. A new session that only knows the decisions will repeat the same mistakes.

Invoke as `/skill:session-handoff` to generate a handoff, or `/skill:session-handoff <path>` to resume from one.

---

## Structure

Each skill follows the same layout:

```
<skill-name>/
  SKILL.md        — LLM execution entry point. Self-contained enough to act without reading anything else.
  README.md       — Purpose, invocation, design rationale. For humans.
  CHANGELOG.md    — Version history. Kept out of SKILL.md to reduce context load.
  references/     — Supporting material loaded on demand by SKILL.md.
```

---

## About

Built by [caseyng](https://github.com/caseyng) — someone who builds things properly, whether in leather, wood, or code.
