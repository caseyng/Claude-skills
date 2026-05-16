# requirements-engineering

Drives requirements discovery for a software project. The skill helps the human figure out
what they want to build — not just documents what they said.

## What makes this different from requirements gathering

Pure requirements gathering is passive. You ask questions; the human answers. The output
is only as good as what the human knows to tell you.

This skill is active. It surfaces what the human took for granted, fills gaps with engineering
domain knowledge, proposes features and phases the human didn't consider, and designs phase
boundaries so early phases don't block later ones. The human corrects and approves; the skill
drives.

## The three functions

**Surface the implicit.** Things so obvious to the human they forgot to say them. Single
user or multi-user? Offline or always-connected? Screen-off operation required? These are
stated as assumptions ("Assuming: X. Correct?") for the human to confirm or correct — not
as open questions.

**Enrich from domain knowledge.** Security posture, observability, error strategy, data
handling, platform constraints. These concerns are surfaced regardless of whether the human
mentioned them. The enrichment checklists define what "regardless" means.

**Phase mapping with extension points.** The skill identifies which features belong to
Phase 1 and which should be deferred. For every deferred feature, it asks: "if this is
added later, what in Phase 1 would need to change?" Anything that would change gets a
minimal extension point — an interface, a stub, an empty config field — at near-zero
Phase 1 cost.

## Exit condition

Requirements are complete when no unanswered question would cause a system designer
to make a different architectural decision. This is the blocking/non-blocking test
applied at requirements level. Round count is not the exit condition.

## Enrichment checklists

- `references/enrichment-universal.md` — every project
- `references/enrichment-android.md` — Android apps
- `references/enrichment-web.md` — web applications
- `references/enrichment-backend.md` — backend services and APIs

## Output

A Requirements Document consumed directly by the `system-design` skill as Stage 2's input.
Contains: goal, features with phase assignments, platform constraints, confirmed assumptions,
surfaced concerns, scope boundaries, phase plan with extension points, and remaining
non-blocking open questions.
