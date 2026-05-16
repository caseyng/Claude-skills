# system-design

Translates an approved Requirements Document into a System Design Document.

## What this skill does

System design sits between requirements and component specification. Requirements capture intent
(what to build and for whom). Component specs capture implementation contracts (exactly how each
piece works). System design occupies the middle: it decomposes the product into components,
defines how they interact, assigns cross-cutting concerns, and records the architectural decisions
that constrain everything downstream.

Two functions distinguish this skill from a generic decomposition exercise:

**Pushback.** Requirements are written from intent — they can promise anything. Architecture is
constrained by technical reality. This skill identifies requirements that are infeasible as stated,
contradictory with other requirements, or ambiguous enough that different readings produce different
architectures. These are surfaced and resolved before committing to a design.

**Confirmed decomposition.** The component list is the highest-leverage decision in the pipeline.
Every Stage 3 spec, every Stage 4 implementation, and Stage 6 integration testing are all structured
around it. Getting it wrong means every downstream stage inherits the flaw. The skill confirms the
component list with the human before proceeding to interaction contracts and cross-cutting concerns.

## What it produces

A System Design Document with five parts:

1. **Components** — what each component is, what it owns, what it does not own
2. **Interaction contracts** — what is passed between components, in what format, who owns it, what happens on failure
3. **Cross-cutting concerns** — security boundary, error propagation, observability, data ownership, startup/shutdown — each with one assigned owner
4. **Technical decisions** — every significant choice with rationale, alternatives considered, and tradeoffs accepted
5. **Deferred items** — what is not built in this version and why

The component list locks on human approval and becomes the authoritative input to Stages 3–5.

## Why the decomposition confirmation matters

Stage 3 specs one component at a time. Stage 4 implements one component at a time.
Stage 6 tests the interactions between components. If the decomposition is wrong:
- Two components that should be one are specced and implemented separately, then awkwardly
  coupled at integration time
- One component that should be two has overlapping ownership — state can disagree, changes
  conflict, the boundary is ambiguous in every spec that follows

A wrong decomposition is not caught by testing. It is caught by pain during implementation
and integration. The skill is designed to surface this pain before it becomes structural.

## Enrichment checklists

- `references/enrichment-universal.md` — system-level concerns for every project
- `references/enrichment-android.md` — Android process model, database ownership, service → UI communication, lifecycle, component exposure

## Output

Stage 2 artifact: `output/stage-2-system-design.md` (path set by the orchestrator).
Consumed directly by the `spec-contract-methodology` skill as input to Stage 3.
