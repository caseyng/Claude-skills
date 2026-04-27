SPEC CONTRACT METHODOLOGY — BACKLOG

Status: Deferred. Complete after MASP challenge.

---

CHANGE: Add §2b Architectural Constraints

Origin: MASP challenge prep — discovered that structural decisions (class hierarchy,
inheritance strategy, component priority, named patterns) don't fit any existing section.
§5 covers behavioural contracts but not class relationships.
§11 covers pipeline sequencing but not registry priority ordering.
§16 covers extension contracts but assumes structure already exists.

What §2b captures:
- Class hierarchy and inheritance strategy
- What extends what, and what must NOT
- Component priority and ordering rules not captured by §11
- Named patterns the implementation must follow (registry, mixin, catch-all strategy)
- Decisions that, if violated, produce wrong implementation — not implementor freedom

What §2b does NOT capture:
- Implementation detail (data structures, algorithm internals)
- Rationale or diagrams
- Anything already covered by §5, §11, §16

Section name: "§2b Architectural Constraints"
Rationale: "Architectural" = accurate term for humans. "Constraints" = operative signal for LLMs.
Combined: unambiguous for both audiences. Keeps the section disciplined — only decisions
that cause wrong implementation if violated.

Placement: After §2 Vocabulary, before §3 System Boundary. Structural decisions must be
declared before component contracts are specified.

---

CHANGES REQUIRED IN METHODOLOGY

1. Add §2b definition to the section template (§1–§23)

2. Update 10-Question Verification — add question:
   "Are structural decisions declared? Class hierarchy, inheritance strategy, priority
   ordering, named patterns — anything that, if left unspecified, produces wrong implementation?"

3. Update Gap Classification examples — add example of an Architectural Constraint gap:
   "Implementor would choose wrong inheritance strategy without this" = BLOCKING

4. Update Handoff Rule note — spec is not handoff-ready if §2b has blocking gaps

---

REFERENCE: MASP §2b (example of what the section looks like in practice)

- LLMCaller: mixin. Decomposer and BaseAgent both inherit it. Not registered. Not spawned.
- Decomposer: plain class. Does NOT extend BaseAgent. Interfaces are incompatible.
- Registry priority: specific types declared before generic_audit. generic_audit is always last.
- Orchestrator deduplicates SubTasks by task_type before spawning agents.
- LLMBoundary: wraps every LLM call. Called before and after LLM in Decomposer and every Agent.
