---
name: session-handoff
version: 2.0.0
description: >
  Manages context continuity across Claude sessions.
  Invocation:
    /skill:session-handoff          → generate a handoff document for this session
    /skill:session-handoff [file]   → resume from a handoff file at the given path
  Also triggers proactively: when context compression begins, Claude emits an alert
  and prompts the user to run /skill:session-handoff before context is lost.
---

# Session Handoff Skill

## Invocation routing

- **No argument** → Mode 1: generate a handoff document for this session
- **File path argument** → Mode 2: read that file, then receive the handoff

## Context pressure watch (always active — not invocation-dependent)

Whenever the system indicates that prior messages are being compressed (the system-reminder containing "automatically compress prior messages" appears), emit this alert **before any other response**:

```
⚠️  Context pressure detected. Run /skill:session-handoff to preserve context before it's lost.
```

Do not wait to be asked. Surface this immediately. The user may not have noticed compression starting.

---

## Mode 0: During a Session

Do not wait until handoff generation to identify what matters. Capture as it happens:

- **Decision made** → note the decision and why, not just what
- **User correction** → note what Claude got wrong and the correct model immediately
- **Approach rejected** → note what was tried and why it failed
- **Constraint established** → note it as hard or soft at the point it's set

This makes generation fast and accurate. Handoffs reconstructed from memory at session end are lossy. Handoffs assembled from live captures are not.

---

## Mode 1: Generate a Handoff

Triggered by `/skill:session-handoff` with no argument.

### Output format

- Small handoff (fits comfortably in chat): output as a markdown code block the user can copy
- Large handoff (complex project, many decisions, substantial state): create a `.md` file

### Compression rules

Write the handoff **for Claude, not a human.** Maximise information density.

- Use fragments, symbols, key→value notation
- Drop all prose filler and transitional language
- Abbreviate freely where intent is unambiguous
- Bullet and nested structure over sentences
- Omit any section where the content would be "none" or "not applicable"

**The cut test** — before including any section, ask: would removing this cause a new session to reason differently or make a different decision? Yes → keep. No → cut. This is an attention rule, not just a size rule. Every section that survives competes for the same context window. Sections that don't earn their place crowd out sections that do.

**The only inclusion test:** would a new Claude session fully reconstruct context AND reasoning from this?

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
# [3-5 word title — specific enough to identify this conversation in a list.
#  Not "session handoff". Pick what makes it findable: the system, task, or
#  decision. e.g. "carapex spec audit", "auth DIP refactor", "pricing redesign".
#  This line influences the new session name.]

## WHAT THIS IS
[Purpose + end goal + exit condition. 1-3 lines max.
Exit condition: what state the world is in when this work is complete.]

## WHERE WE ARE
[Current state. What exists. What works. What doesn't. Specific.
For sequential work: current phase, next decision point, what's blocking.]

## CONSTRAINTS
[Must never / must always / must not change. Technical + style + process.
Mark each: HARD (never violated) or SOFT (default, overridable with explicit rationale).
Only rank or annotate constraints when two or more are known to be in tension.]

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

## OPEN QUESTIONS
[Unresolved. Who decides. Options if known.]

## CONTEXT NEW SESSION CANNOT INFER
[Background, preferences, mental model, framing, relationships between things.
Include working relationship state only if it would change how the new session engages —
established trust, known sensitivities, communication rhythm. Cut this line if the work
is purely technical and rapport context would not change any decision.]

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

## Mode 2: Receive a Handoff

Triggered by `/skill:session-handoff [file]` where a file path is provided.

### Workflow

1. **Read the file** at the given path using the Read tool before doing anything else
2. Read the entire handoff content carefully
3. Pay special attention to REASONING PATTERNS and USER CORRECTIONS — these are
   higher signal than the state sections
4. Confirm understanding explicitly — briefly restate: what you're working on,
   where things stand, what comes next, and the dominant reasoning pattern
5. **Stop there.** Do not begin any work yet.
6. Wait for the user to follow up with any additional files, documents, or context
7. Only begin working once the user signals they are ready

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
