---
version: 1.0.0
skill: code-integrity-guardrail / go
---

# Go Guardrail Bindings

Go 1.21+ concrete additions to the universal code-integrity-guardrail.
This file only records what the universal skill cannot express without
Go-specific syntax, tooling, or behaviour.

---

## Phase 0: Go Security Tooling

Run before delivering any Go code. State results in the post-generation
verification statement.

| Tool | Command | What it catches |
|---|---|---|
| go vet | `go vet ./...` | Suspicious constructs (unreachable code, bad format strings, mutex copies) |
| staticcheck | `staticcheck ./...` | Deprecated APIs, unused code, correctness issues beyond vet |
| govulncheck | `govulncheck ./...` | Known CVEs in direct and transitive dependencies |
| go test | `go test ./...` | All tests must pass before delivery |

**Minimum bar:** `go vet ./...` with zero findings. `staticcheck ./...` with no errors
(warnings may be recorded but do not block delivery if they are pre-existing).

**When bash is unavailable:** State this explicitly. Cap all findings at FLAG or UNCERTAIN.
Note which vet checks apply to the generated patterns (e.g., if generating code with mutex,
flag the copy-checker concern even without running it).

---

## Security Syntax (Category 1)

Only patterns where Go-specific syntax is required to detect the violation.

### Insecure Randomness

```go
// Violation — predictable, not suitable for security use
import "math/rand"
token := fmt.Sprintf("%d", rand.Int63())
id := rand.Intn(1000000)

// Correct
import "crypto/rand"
b := make([]byte, 32)
if _, err := rand.Read(b); err != nil {
    return fmt.Errorf("generate token: %w", err)
}
```

### SQL Injection

```go
// Violations — any database/sql driver
db.QueryRowContext(ctx, "SELECT * FROM chats WHERE jid = '"+jid+"'")
db.ExecContext(ctx, fmt.Sprintf("DELETE FROM messages WHERE id = %s", id))

// Correct
db.QueryRowContext(ctx, "SELECT * FROM chats WHERE jid = ?", jid)
db.ExecContext(ctx, "DELETE FROM messages WHERE id = ?", id)
```

### TLS Verification Bypass

```go
// Violation
&tls.Config{InsecureSkipVerify: true}

// Correct — omit InsecureSkipVerify or set to false explicitly
&tls.Config{}
```

### Path Traversal

```go
// Violation — user input in path without validation
path := filepath.Join(baseDir, userInput)
// No check that path is still under baseDir

// Correct
path := filepath.Join(baseDir, filepath.Clean("/"+userInput))
if !strings.HasPrefix(path, baseDir+string(os.PathSeparator)) {
    return errors.New("invalid path")
}
```

---

## Common Hallucinations (Go Ecosystem)

Phantom packages that LLMs generate that do not exist or have wrong import paths:

| Hallucinated import | Reality |
|---|---|
| `github.com/go-sqlite3` | Does not exist; use `github.com/mattn/go-sqlite3` (CGO) or `modernc.org/sqlite` (pure Go) |
| `golang.org/x/crypto/bcrypt` | Correct path — but verify the function signatures; LLMs sometimes invent bcrypt APIs |
| `github.com/sirupsen/logrus/structured` | Does not exist; logrus API is on the root package |
| `github.com/uber/zap` | Wrong; correct import is `go.uber.org/zap` |
| `golang.org/x/net/http2` | Exists but is rarely needed directly; `net/http` handles HTTP/2 automatically |
| Any `gomobile/` import path | gomobile is a build tool, not an import; `golang.org/x/mobile/bind` is the actual package, rarely imported directly |

**Verification:** Before referencing any package, confirm the import path exists in
`pkg.go.dev`. An import that does not resolve will silently fail at `go mod tidy`.

---

## Pre-flight Checklist (Go-specific additions)

Extends the universal phases. Do not restate universal checks here.

### Phase 1 additions (before writing)

- [ ] Is CGO needed? If yes: is `modernc.org/sqlite` viable instead of `mattn/go-sqlite3`
      (required for gomobile compatibility — CGO breaks gomobile binding)
- [ ] Are all goroutines launched in this code guaranteed to exit? Identify the exit condition
      for each one before writing
- [ ] Will any type cross a gomobile boundary? If yes: verify it is on the supported types
      list in `go-engineering/references/gomobile.md`

### Phase 2 additions (after writing)

- [ ] Every `err` return is handled — no `_, _` assignment unless the error is genuinely
      irrelevant (rare; must be commented)
- [ ] No `interface{}` / `any` at exported boundaries — use typed interfaces
- [ ] No naked goroutine without a context or done channel for cancellation
- [ ] No `time.Sleep` in production code (use `time.After` in a select, or `time.NewTimer`)
- [ ] All `defer` calls execute in the expected scope (defers in loops execute at function
      return, not loop iteration — common bug)
- [ ] `sync.Mutex` is never copied after first use (vet catches this; confirm)

---

## Pressure Scenarios (Go-specific)

### "Just use `interface{}` for flexibility"

The flexibility is real; the cost is type assertion failures at runtime. Use a typed
interface. If the set of types is finite, use an enum-style constant or a sum type pattern.

```go
// Wrong
func Process(v interface{}) error { ... }

// Correct
type Processor interface {
    Process() error
}
```

### "Ignore the error here, it can't fail"

In Go, the correct response to "it can't fail" is a comment explaining why, not `_`.
If the function signature returns an error, the author of that function believed it could fail.

```go
// Wrong
db.Close() // ignored

// Correct
if err := db.Close(); err != nil {
    slog.Error("close db", "err", err)
}
```

### "Use `math/rand` for test data — it's just tests"

Tests that use predictable random data are fine with `math/rand`. But if the code under
test generates a random token and the test verifies it has a property, using `math/rand`
in production code will produce a security vulnerability that the test would not catch.
If the production code path is in scope, use `crypto/rand`.

### "Skip `go vet` — it's a small change"

`go vet` catches mutex copy bugs, unreachable code, and bad `Printf` format strings — all
of which compile without error. A "small change" can introduce any of these. Always run it.
