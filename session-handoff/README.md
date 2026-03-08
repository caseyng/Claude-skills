# session-handoff

## The problem

Claude has no memory between sessions. When the context window fills and you start fresh, the obvious loss is state — where you were, what was done. The less obvious loss is reasoning: why decisions were made, what was tried and rejected, where Claude's default thinking was wrong and you corrected it.

A new session given only state will repeat the same mistakes. It knows what was decided but not why, so when something unexpected comes up, it derives the wrong answer from scratch.

## What this skill does

Two modes, one skill.

**Generating a handoff.** When context is filling up, this skill produces a structured document that captures not just state but reasoning patterns, corrections, and the thinking that shaped the work. Written for Claude, not for humans — maximum information density, minimum prose.

**Receiving a handoff.** When a new session opens with a handoff document, this skill reads it correctly — prioritising corrections and reasoning patterns over state, confirming understanding before touching anything, waiting for you to signal readiness before proceeding.

The goal is continuity of thought, not just continuity of facts.

There are many session handoff tools — plugins, hooks, YAML-based state files, auto-memory. Almost all of them capture what happened. This skill captures why decisions were made and what Claude got wrong — because a new session that inherits only the decisions will reason incorrectly when something unexpected comes up.

## When to use it

- Any long project spanning multiple sessions
- Before starting something large that will exhaust the context window
- When you've corrected Claude multiple times and can't afford to explain it all again
- When handing a project to a new session and needing it to think, not just execute

## Files

- `session-handoff.skill` — install into Claude
- `session-handoff_prompt.md` — activation prompt and usage guide
