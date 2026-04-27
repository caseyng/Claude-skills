# session-handoff

Manages context continuity across Claude sessions. Generates compact handoff documents that transfer not just state but reasoning patterns to new sessions.

## Invocation

```
/skill:session-handoff              # generate a handoff for the current session
/skill:session-handoff path/to/handoff.md   # resume from a saved handoff file
```

## Modes

### Generate (`/skill:session-handoff`)

Produces a compressed handoff document capturing decisions, reasoning, corrections, and next steps. Output format depends on complexity:
- Small handoff → markdown code block in chat (copy-paste)
- Large handoff → `.md` file

### Resume (`/skill:session-handoff [file]`)

Reads the handoff file at the given path, absorbs its context, and confirms readiness. Claude will restate what it's working on, current state, next steps, and the dominant reasoning pattern — then wait for the user to signal readiness before doing anything.

### Auto-alert (always active)

When the system begins compressing prior messages (context pressure), Claude emits an alert before any other response:

```
⚠️  Context pressure detected. Run /skill:session-handoff to preserve context before it's lost.
```

This is not hook-based — it is a skill-level instruction that fires when Claude observes the compression system message. No configuration required.

## Design notes

**Why no hook for auto-alert?** Hooks cannot observe Claude's internal context window state. The practical trigger is the system-injected compression notice, which Claude can observe and react to directly.

**Why file path for resume instead of paste?** A file path lets Claude read the handoff independently of the chat window, avoiding truncation of large handoffs and keeping the conversation cleaner.

**Handoffs are written for Claude, not humans.** High information density using fragments, symbols, and key→value notation. The goal is to transfer reasoning patterns, not just state — a new session that knows decisions but not reasoning will repeat mistakes.
