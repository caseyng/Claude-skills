# spec-driven-testing

A two-agent test generation protocol that enforces separation between implementation knowledge and test authorship.

## The problem it solves

When the same agent writes both implementation and tests, the tests verify internal consistency rather than correctness. The clearest example: a TypeConverter that encodes data with wrong field names passes all round-trip tests because both encode and decode share the same bug. Neither test catches anything. The oracle — the thing that knows what "correct" looks like — is absent.

The solution is enforced agent separation. Agent 1 (Shape Extractor) reads source code and emits only the public API surface: signatures, types, contracts, side effects. No implementation bodies. Agent 2 (Test Writer) receives only that shape document and derives test cases from the stated contracts and requirements. It cannot reproduce implementation bugs it has never seen.

## Usage

```
/skill:spec-driven-testing extract           → produce a shape document from source files
/skill:spec-driven-testing write kotlin      → write tests from a shape document (no source access)
/skill:spec-driven-testing kotlin            → full pipeline: extract then write
```

## What "shape" means

A shape document captures the API surface that callers depend on:
- Function and method signatures with all parameter and return types
- Nullability of every type
- Error and exception contracts
- Observable side effects (writes to DB, fires an event)
- Behavioural contracts from documentation

It explicitly excludes implementation bodies, private helpers, and implementation-revealing comments.

## The Format Assertion Requirement

The most important rule in this skill. For any function that serializes data — JSON converters, DB TypeConverters, string encoders, network DTOs — round-trip tests alone are insufficient. The test must also assert that the serialized output contains expected field values and variant names. Round-trip tests pass when encode and decode share the same bug. Format assertions do not.

## Language bindings

- `kotlin.md` — JUnit4, Room in-memory, runTest, Flow.first(), sealed class assertions
- `go.md` — stdlib testing, table-driven tests, in-memory SQLite, interface fakes
- `python.md` — pytest, fixtures, parametrize, fake patterns, JSON format assertions

## Composing with other skills

- **spec-contract-methodology**: The spec produced by that skill is the requirement input for Agent 2. Shape document + spec = complete black-box test derivation.
- **code-integrity-guardrail**: Run on the generated test code to catch hallucinated imports or phantom APIs before the tests are committed.

## Skill architecture rules followed

- SKILL.md is the execution entry point, self-contained
- This README contains rationale — not repeated in SKILL.md
- Bindings add only language-specific content, no restatement of universal rules
