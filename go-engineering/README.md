# go-engineering

**Version:** 1.0.0

Senior Go engineering standards for production code. Covers the patterns and constraints
needed to produce idiomatic, safe, testable Go — including gomobile-specific rules for
cross-language bindings.

---

## Invocation

```
/skill:go-engineering
```

Composes with:

```
/skill:code-integrity-guardrail go
```

---

## What It Covers

| Area | Key rules |
|---|---|
| Module structure | `cmd/` / `internal/` / `pkg/` conventions; `go.mod` discipline |
| Error handling | Explicit returns, `fmt.Errorf` wrapping, sentinel errors, no library panics |
| Package design | Naming, receivers, interface placement (at point of use), small interfaces |
| Concurrency | Goroutine exit conditions, channel ownership, mutex scope, `errgroup` |
| Context | First param, never stored in structs, `WithTimeout` at service boundaries |
| Logging | `log/slog` structured logging, log once at top of call stack |
| Testing | Table-driven tests, `t.Helper()`, in-memory backends, `testdata/` fixtures |
| Security | `crypto/rand`, parameterised SQL, TLS always verified |
| SQLite | WAL mode, busy_timeout, foreign_keys, prepared statements, migrations |
| gomobile | Supported types, callback pattern, build pipeline, Termux build notes |

---

## Structure

```
SKILL.md                    — LLM execution entry point
references/
  gomobile.md               — Full gomobile binding contract
CHANGELOG.md                — Version history
README.md                   — This file
```

---

## Design Rationale

**Why in-memory backends over mocks?** Mocks verify that the code calls the right methods.
In-memory backends verify that the code produces the right behaviour. A mock that passes
while the real database rejects the query is a false pass. Go makes in-memory SQLite trivial
(`:memory:` DSN); use it.

**Why define interfaces at the point of use?** The Go standard library does this — `io.Reader`
is defined in the `io` package because `io` is where it's consumed. The pattern prevents
abstraction debt: interfaces only exist when there's a real consumer that depends on them.
Pre-defining interfaces "for future extensibility" produces unused abstractions that no real
implementation will satisfy correctly.

**Why `log/slog` over `logrus` / `zap`?** stdlib reduces the dependency surface. `slog`
is structured, performant, and compatible with the handler ecosystem. For a project that
already uses gomobile (which must avoid CGO), keeping dependencies minimal reduces the
chance of a CGO dependency sneaking in through a transitive import.
