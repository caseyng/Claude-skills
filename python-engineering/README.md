# python-engineering

**Version:** 1.0.0

Senior Python engineering standards for local design integrity. Invoked as `/skill:python-engineering`.

---

## Purpose

This skill makes Claude behave as a thoughtful senior Python engineer when writing, reviewing, or refactoring Python code. It does not just enforce style — it surfaces design problems, flags tradeoffs, and refuses to silently produce inferior code.

The scope is **local design integrity**: how modules, classes, and functions are structured. It does not govern deployment architecture, async policy, observability, packaging, or org-wide conventions.

---

## What It Covers

| Area | Details |
|---|---|
| Design patterns | DIP, Strategy, Registry, Provider, Composition Root |
| Type hints | Full annotation of public interfaces; Python 3.10+ union syntax |
| Dataclasses | Config-as-dataclass, `from_dict()` at boundary, `field(default_factory=...)` |
| ABCs | Contract-first extensibility, extension protocol (`from_config`, `default_config`) |
| Resource management | Context managers for all externally-held resources; class-lifetime `close()` |
| Logging | `logging` over `print`; no secrets in logs; no logging in hot paths |
| Exception handling | Narrowest catch, specific exception types, documented broad catches |
| Failure semantics | Raise on invariant violations; return structured results for operational failures |
| Thread safety | Registry locking, compound-operation protection |
| Testing | Behaviour-over-implementation, stubs over mocks, unit/integration separation |
| Config design | Raw dicts at extensibility boundaries; discovery-driven default generation |
| Documentation | Contract-level docstrings; extension guides on ABCs; inline comments explain why |

---

## How to Invoke

```
/skill:python-engineering
```

Invoke whenever:
- Writing new Python modules, classes, or functions
- Reviewing or refactoring existing Python code
- Discussing Python patterns, design smells, or config architecture
- Designing ABCs, registries, providers, or composition roots

---

## Composition

This skill **requires** `code-integrity-guardrail python` (v1.2.x). The SKILL.md loads it automatically before proceeding. The guardrail handles universal code quality checks (mutable defaults, encoding, print statements, assert misuse, exception breadth) so this skill can focus on Python-specific design concerns.

If the guardrail has been updated to a major version beyond 1.x, verify compatibility before pairing.

---

## Reference Files

The SKILL.md loads reference files on demand rather than all at once. Each file is read only when the task reaches that area of concern.

| File | Covers |
|---|---|
| `references/architecture.md` | DIP, Strategy, Registry, Provider, Composition Root, lifecycle, extensibility |
| `references/python-patterns.md` | Type hints, dataclasses, ABCs, `__repr__`, resource management, autodiscovery, logging, exceptions, thread safety, encoding, failure semantics |
| `references/config-design.md` | Raw dicts at extensibility boundaries, discovery-driven config generation |
| `references/documentation.md` | Module docstrings with extension guides, contract-level class/method docs, inline comment standards |
| `references/testing.md` | What to test, test structure, stubs, unit/integration separation |

---

## Design Rationale

**Why composing with the guardrail instead of restating it?** The guardrail already covers universal slop patterns (mutable defaults, missing encoding, print over logging, bare Exception catches, assert for validation). Restating those in this skill would create drift — two sources of truth diverging. This skill reads clean on entry because the guardrail cleaned up mechanics first.

**Why "local design integrity" as the explicit scope?** Without a stated scope boundary, a senior engineer skill drifts into architecture, infra, and org standards where it has no grounding. The scope statement lets Claude say "that's outside scope" rather than hallucinating guidance.

**Why flag design smells before writing code?** Silently producing inferior code serves no one. If a request would produce a bad design, naming it upfront keeps the human in the loop on tradeoffs rather than discovering it in review.

**Why `from_config` as a stable method name across all extensible components?** Naming consistency across the extension protocol means a developer who has extended one component can extend any other without re-reading docs. The convention is stronger than any individual docstring.
