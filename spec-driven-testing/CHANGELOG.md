# Changelog — spec-driven-testing

## 1.0.0 — 2026-05-15

Initial release.

### What's in it

- Two-agent protocol: Shape Extractor (Mode 1) and Test Writer (Mode 2), plus full pipeline (Mode 3)
- Shape document format: structured markdown capturing signatures, contracts, nullability, side effects — nothing else
- Universal test derivation rules: equivalence partitioning, boundary values, error coverage, filter four-case rule, nullable two-case rule
- The Format Assertion Requirement: named and prominent; derived from the observation that round-trip tests pass when both encode and decode share the same bug
- Spec gap report: mandatory output from Agent 2 whenever a behaviour cannot be tested because the spec doesn't define expected output
- Language bindings: Kotlin (JUnit4, Room in-memory, runTest), Go (table-driven, in-memory SQLite, fakes), Python (pytest, fixtures, parametrize)

### Origin

Emerged from a whatsbot Android project session where TypeConverter tests were needed. Observed that the same Claude instance writing both Converters.kt and the tests would naturally write round-trip tests that pass even when the serialization format is wrong. Resolved by spawning a separate agent with only the interface signatures and spec — it independently derived test cases and surfaced spec gaps the implementation author hadn't considered. Generalised that protocol into this skill.
