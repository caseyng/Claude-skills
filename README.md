# claude-skills

Reusable Claude skills built for one purpose: making AI output reliable enough to depend on.

Most Claude skills automate tasks. These enforce standards. The difference matters when the output has to hold up — in production, under edge cases, across sessions.

Each skill is a unit: a `.skill` file that installs into Claude, and a prompt file that documents the reasoning behind it. Install what you need. Read the prompt file if you want to understand the thinking.

---

## Skills

### [spec-contract-methodology](./spec-contract-methodology/)
LLMs are probabilistic. Without a precise specification, they fill gaps with assumptions — and those assumptions become bugs. This skill enforces a 23-section specification methodology rigorous enough that any engineer, in any language, could rebuild the system without reading the original code. Built because other spec-driven tools automate the process; this one enforces the standard.

### [session-handoff](./session-handoff/)
Claude has no memory between sessions. Most handoff tools preserve state — where you were, what was done. This one preserves reasoning: why decisions were made, what was tried and rejected, where Claude's thinking was wrong and you corrected it. A new session that only knows the decisions will repeat the same mistakes.

### [python-engineering](./python-engineering/)
AI-generated Python code has a characteristic failure mode: it works, and it's wrong. It passes the test, misses the design. This skill enforces senior engineering standards before a line is written — dependency inversion, proper ABCs, composition roots, registry patterns — so Claude codes like an engineer who has to maintain it, not one who just has to ship it.

---

## How to use

Install the `.skill` file into your Claude environment and reference it by name.

The `_prompt.md` file in each folder is optional reading. It documents the intent, decisions, and reasoning behind each skill — not instructions for the user, but a record of the thinking. Read it if you want to understand why the skill works the way it does.

---

## About

Built by [caseyng](https://github.com/caseyng) — someone who builds things properly, whether in leather, wood, or code.
