# Go binding — spec-driven-testing

## Shape extraction (Agent 1 additions)

Extract from Go source:
- Exported identifiers only (uppercase first letter) — unexported are implementation detail
- Interface definitions are the primary API surface — list all method signatures
- `error` as a return value is a contract: "this function can fail" — always note it
- Named return values: if named, the name is part of the contract (it documents what the value represents)
- Receiver type: value vs pointer receiver is part of the signature for methods
- `context.Context` as first parameter — note it, callers must know
- Struct fields that are exported are part of the shape

Do not extract: unexported functions, unexported struct fields, init() bodies, internal package code.

## Test framework (Agent 2)

Go uses the stdlib `testing` package. No third-party framework required.

```go
package mypackage_test   // use _test package for black-box tests

import (
    "testing"
)

func TestFunctionName_Scenario_ExpectedOutcome(t *testing.T) {
    // ...
}
```

Use the `_test` package suffix for black-box tests (cannot access unexported identifiers). Use the same package name (no suffix) only when testing unexported helpers — avoid in spec-driven tests.

## Table-driven tests — idiomatic Go pattern

For equivalence partitions and boundary values:

```go
func TestConverter(t *testing.T) {
    tests := []struct {
        name  string
        input InputType
        want  OutputType
    }{
        {"valid input", validInput, expectedOutput},
        {"empty input", emptyInput, expectedEmpty},
        {"nil input", nil, expectedNil},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Convert(tt.input)
            if got != tt.want {
                t.Errorf("Convert(%v) = %v, want %v", tt.input, got, tt.want)
            }
        })
    }
}
```

## In-memory dependencies

**SQLite (mattn/go-sqlite3):**
```go
import (
    "database/sql"
    _ "github.com/mattn/go-sqlite3"
)

db, err := sql.Open("sqlite3", ":memory:")
if err != nil {
    t.Fatalf("open db: %v", err)
}
defer db.Close()
```

**Interface fakes — prefer over mocks:**
```go
type fakeListener struct {
    received []*Message
}

func (f *fakeListener) OnMessage(msg *Message) {
    f.received = append(f.received, msg)
}
```

## Assertions

Go stdlib has no assertion helpers. Use `t.Errorf` (test continues) or `t.Fatalf` (test stops):

```go
if got != want {
    t.Errorf("FunctionName() = %v, want %v", got, want)
}
if err != nil {
    t.Fatalf("unexpected error: %v", err)
}
```

For struct equality, use `reflect.DeepEqual` or compare field by field:
```go
if !reflect.DeepEqual(got, want) {
    t.Errorf("got %+v, want %+v", got, want)
}
```

## Format Assertion Requirement — Go specifics

For JSON marshalling or string encoding:
```go
encoded, err := json.Marshal(value)
if err != nil {
    t.Fatalf("marshal: %v", err)
}
// Round-trip
var decoded MyType
if err := json.Unmarshal(encoded, &decoded); err != nil {
    t.Fatalf("unmarshal: %v", err)
}
if !reflect.DeepEqual(value, decoded) {
    t.Errorf("round-trip failed: got %+v, want %+v", decoded, value)
}
// Format assertion — checks the encoded bytes contain expected values
if !bytes.Contains(encoded, []byte(`"expected_field":"expected_value"`)) {
    t.Errorf("encoded JSON missing expected field: %s", encoded)
}
```

## Error testing

```go
// Assert an error is returned
_, err := FunctionThatShouldFail(invalidInput)
if err == nil {
    t.Error("expected error, got nil")
}

// Assert a specific error type or message
if !errors.Is(err, ErrExpected) {
    t.Errorf("got error %v, want %v", err, ErrExpected)
}
```

## Test structure pattern

Test function names: `Test<FunctionName>_<scenario>_<expectedOutcome>`
```go
func TestNewTestClient_MemoryPath_SeedsCorrectChatCount(t *testing.T) { ... }
func TestSimulateIncomingMessage_NonStubClient_IsNoop(t *testing.T) { ... }
```
