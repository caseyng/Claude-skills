---
name: session-handoff
description: >
  Manages context continuity across Claude sessions. Use this skill in TWO situations:
  (1) GENERATING a handoff — when context is filling up, when the user asks for a handoff,
  or proactively before starting something large that may exhaust the context window.
  Trigger phrases: "handoff", "new window", "context is full", "start fresh", or when
  Claude detects the conversation is nearing its limit.
  (2) RECEIVING a handoff — when a new session opens and the user pastes a handoff document.
  Trigger: user pastes a structured document that looks like a session handoff at the start
  of a conversation.
---

# Session Handoff Skill

Two modes. Read the situation and apply the correct one.

---

## Mode 1: Generating a Handoff

Trigger this when:
- User requests a handoff explicitly
- Context window is visibly filling up
- About to begin something large (long code task, big document, extended research)

### Output format

- Small handoff (fits comfortably in chat): output as a markdown code block the user can copy
- Large handoff (complex project, many decisions, substantial state): create a `.md` file for download

### Compression rules

Write the handoff **for Claude, not a human.** Maximise information density.

- Use fragments, symbols, key→value notation
- Drop all prose filler and transitional language
- Abbreviate freely where intent is unambiguous
- Bullet and nested structure over sentences
- The only test: would a new Claude session fully reconstruct context AND reasoning from this?
- Omit any section where the content would be "none" or "not applicable"

### The core problem with state-only handoffs

A handoff that captures only *what* was decided is insufficient. A new session that knows
the decisions but not the reasoning will:
- Make the same mistakes that were already corrected
- Re-open questions that were deliberately closed
- Miss the user's mental model and re-derive incorrect conclusions
- Execute correctly but reason incorrectly when something unexpected comes up

The goal is to transfer **reasoning patterns**, not just state. The new session must be able
to think, not just execute.

### What to capture beyond state

For every significant decision or correction, ask: **why did we arrive here, and what
would make a new session arrive somewhere different?**

Capture:
- **Corrections the user made** — what Claude got wrong, how the user reframed it, what
  the correct mental model is. These are the highest-value entries — they represent places
  where the default reasoning fails on this project.
- **Reasoning chains** — not just "we decided X" but "we considered A, rejected it because
  B, then realised C which led to X"
- **First principles the user holds** — the underlying values or design philosophy that
  drive decisions. These let the new session derive correct answers to novel questions.
- **What NOT to do** — failure modes that look reasonable but are wrong for this project.
  A new session will be tempted by the same wrong paths.
- **The user's communication style** — how they prefer to receive information, what they
  find annoying, what signals they use to indicate a direction is wrong.

### Handoff structure

Adapt to the actual work. Only include sections that earn their place.

```
# SESSION HANDOFF

## WHAT THIS IS
[Purpose + end goal. 1-3 lines max.]

## WHERE WE ARE
[Current state. What exists. What works. What doesn't. Specific.]

## DECISIONS MADE
[decision → why → tradeoffs accepted]

## WHAT FAILED / REJECTED
[Approaches tried and abandoned. New session must not repeat.]

## REASONING PATTERNS
[The mental models, first principles, and thinking corrections that shaped this work.
Not what was decided — why, and how to think about similar problems going forward.
Format: SITUATION → WRONG INSTINCT → CORRECT FRAMING → IMPLICATION]

## USER CORRECTIONS
[Places where Claude's default reasoning was wrong and the user corrected it.
These are the highest-signal entries. Format: what Claude got wrong → what the
correct model is → how to avoid repeating it.]

## NEXT
Priority order:
- MUST: [before anything else]
- SHOULD: [soon]
- DEBT: [deferred, known]

## CONSTRAINTS
[Must never / must always / must not change. Technical + style + process.]

## OPEN QUESTIONS
[Unresolved. Who decides. Options if known.]

## CONTEXT NEW SESSION CANNOT INFER
[Background, preferences, mental model, framing, relationships between things.]

## ARTIFACTS
[Files, documents, code produced. Name → location → current state.]

## RESUME INSTRUCTIONS
[What the new session should load, in what order, before doing anything.]

---
ORIENTATION: [One paragraph written directly to the new session. What it is,
where things stand, what to do first. Include the dominant reasoning pattern
for this project — the one insight that, if lost, would cause the most damage.]
```

---

## Mode 2: Receiving a Handoff

Trigger this when a new session opens with a handoff document.

### Workflow

1. Read the entire handoff document carefully before responding
2. Pay special attention to REASONING PATTERNS and USER CORRECTIONS — these are
   higher signal than the state sections
3. Confirm understanding explicitly — briefly restate: what you're working on,
   where things stand, what comes next, and the dominant reasoning pattern
4. **Stop there.** Do not begin any work yet.
5. Wait for the user to follow up with any additional files, documents, or context
6. Only begin working once the user signals they are ready

### Confirmation format

Keep it tight. Something like:

```
Handoff received. [1-2 sentences on what we're working on and current state.]
[1 sentence on the dominant reasoning pattern — show you've absorbed it, not just read it.]
Ready when you are — paste any files or context you want me to load first.
```

Do not ask clarifying questions at this stage. The handoff should be sufficient.
If something is genuinely missing, note it in the confirmation — do not ask a
series of questions.
