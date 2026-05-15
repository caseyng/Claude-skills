---
name: spec-driven-testing
version: 1.0.0
description: Two-agent test generation protocol that enforces source/test independence. Agent 1 extracts shapes from source; Agent 2 writes tests from shapes only — never from implementation.
---

# Spec-Driven Testing

## Why this exists

LLMs writing both implementation and tests produce tests that verify internal consistency, not correctness. The failure mode: encode and decode share the same bug, so round-trip tests pass while the stored format is wrong. The test writer knowing the implementation is the defect. The two-agent separation is the fix. It is not optional.

## Invocation routing

- **`/skill:spec-driven-testing extract`** → Mode 1: read source files, emit shape document, stop
- **`/skill:spec-driven-testing write <language>`** → Mode 2: read shape document, write tests, never read source
- **`/skill:spec-driven-testing <language>`** → Mode 3: full pipeline — extract, then treat shape doc as sole source of truth

---

## Mode 1: Shape Extractor

Read the files the user specifies. For each public type, function, method, and interface, emit a shape entry. Stop immediately after emitting the shape document — do not begin writing tests.

### Include

- Full signature: function/method name, parameter names + types, return type
- Nullability of every parameter and return value
- Error/exception contract: which errors are possible and under what condition
- Observable side effects: writes to DB, fires an event, sends a network call, mutates shared state
- Behavioural contracts from doc comments: what the function guarantees, not how it does it

### Exclude

- Function and method bodies
- Private or internal helper functions not accessible to callers
- Implementation choices ("uses a hashmap", "iterates in order")
- Doc comments that describe HOW — strip the how, keep the what

**Extraction rule for ambiguous doc comments:**
If a doc comment says "iterates over all rules and checks each condition," strip it and write "returns matching rules." If it says "returns the first match or null," keep it — that's a behavioural contract. When in doubt: would a caller need this information to write a correct call site? Yes → keep. No → strip.

### Shape document format

Output as `shapes.md` (or inline block for small extractions):

```markdown
# Shape Document: <module / package / class name>
Generated from: <file paths>
Language: <language>

## Types

### <TypeName>
Fields:
- <field>: <Type> [nullable] — <purpose>

## Functions / Methods

### <ClassName>.<method> | <functionName>
Signature: `<full signature>`
Parameters:
  - <name>: <Type> [nullable] — <purpose>
Returns: <Type> [nullable] — <what it represents>
Errors: <ErrorType> — <condition>
Side effects: <description> | none
Contracts: <preconditions, postconditions>
```

Emit nothing else. Do not start writing tests.

---

## Mode 2: Test Writer

**Hard constraint:** After receiving the shape document, do NOT read any source implementation files — not even to verify a type name. If asked to check the implementation, decline and explain: reading the implementation breaks the independence guarantee that makes these tests trustworthy.

Derive all test cases from: (1) shape document, (2) spec/requirements provided by the user, (3) type contracts alone. Never from guessing or inferring what the implementation does.

### Universal test derivation rules

**Functions — minimum test set:**
- One representative input per valid equivalence partition
- Boundary values: empty collection, zero, null, min, max
- One test per documented error condition (assert the error, not just that no exception occurred)
- If return order is specified, assert it

**The Format Assertion Requirement (serializers and converters):**

Round-trip tests are necessary but not sufficient. ALWAYS add format assertions:

```
// Required: round-trip
assertEquals(original, decode(encode(original)))

// Also required: format content
val encoded = encode(original)
assertTrue(encoded.contains("EXPECTED_FIELD_VALUE"))
assertTrue(encoded.contains("VariantName"))
```

This applies to: JSON converters, DB TypeConverters, binary serializers, string encoders, network DTOs, any function whose output is later read by a separate decode path.

Rationale: if encode and decode both use the wrong field name, the round-trip passes; a direct format assertion fails.

**State-based systems (databases, repositories, caches):**
Each test: establish fresh known state → execute one operation → assert exactly the state change. Never share state between tests. Use in-memory or fake instances (see language binding for setup).

**Filters and predicates:**
Always test: (a) item that passes, (b) item that does not pass, (c) empty input, (d) item exactly at the boundary threshold.

**Nullable types:**
Always test: non-null input with non-null output, null input → expected output (null or documented default), and (if applicable) non-null input → null output.

**Enum converters:**
Test every enum value, not just a sample. Table-driven or loop-based. Assert the exact encoded string, not just round-trip.

### Spec gap report

After writing all tests, always emit:

```markdown
## Spec gaps

| Behaviour | Reason untested |
|---|---|
| <description> | Spec does not define expected output for this input |
| <description> | Spec is ambiguous: could be interpreted as X or Y |
```

Do not guess. Do not make up expected behaviour. Surface the gap.

---

## Mode 3: Full pipeline

1. Run Mode 1 on the files the user provides. Emit the shape document.
2. Treat the emitted shape document as the only source of truth. Do not re-read source files for any reason.
3. Run Mode 2 using the shape document and any spec the user supplies.

---

## Language bindings

Load `references/bindings/<language>.md` before writing tests. Each binding provides: how shapes are expressed in that language, test framework setup, in-memory dependency construction, assertion patterns.

Available: `kotlin`, `go`, `python`

If the language has no binding, apply the universal rules above and flag that no binding exists.
