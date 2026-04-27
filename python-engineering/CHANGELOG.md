# Changelog

## 1.0.1 — 2026-04-27

### README.md
- Removed guardrail version pin from Composition section. A pin implies version-coupling
  that doesn't exist — the skills are compositionally complementary, not version-locked.
  Guardrail improvements benefit python-engineering automatically. The only breaking change
  would be a guardrail invocation interface change, not a content upgrade. Pin added and
  immediately reverted after critique identified it as incorrect framing.

---

## 1.0.0 — 2026-04-27

Initial release.

Covers: DIP, Strategy, Registry, Provider, Composition Root, dependency lifecycle, extensibility protocol, type hints, dataclasses, ABCs, `__repr__`, resource management, autodiscovery, mutable defaults, logging, exception handling, thread safety, encoding, failure semantics, typed result objects, specific exception types, config design (raw dicts at boundary, discovery-driven generation), documentation standards, and testing standards.

Composes with `code-integrity-guardrail python` to avoid restating universal mechanics checks.
