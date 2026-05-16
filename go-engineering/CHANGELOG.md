# Changelog — go-engineering

## 1.0.0 — 2026-05-16

Initial release.

### What's in it

- Module structure: `cmd/`, `internal/`, `pkg/` conventions; `go.mod` / `go.sum` rules
- Error handling: explicit returns, `fmt.Errorf` wrapping, sentinel errors, no panics in
  library code, recover only at service entry points
- Package design: naming conventions, receiver naming, interface placement (point of use),
  accept interfaces / return concrete types
- Concurrency: goroutine exit conditions, channel ownership rules, mutex patterns, `errgroup`
- Context propagation: first parameter convention, no storage in structs, `WithTimeout` at
  service boundaries
- Logging: `log/slog` (structured), log levels, log-once-at-top rule
- Testing: table-driven tests, `t.Helper()`, in-memory backends over mocks, `testdata/`
- Security: `crypto/rand` vs `math/rand`, parameterised SQL, `filepath.Clean` path validation,
  TLS always verify
- SQLite patterns: WAL + busy_timeout + foreign_keys pragmas, prepared statements, migrations
- gomobile: `references/gomobile.md` — full binding contract (supported types, callback pattern,
  build pipeline, Termux/proot environment notes, test stub constructor)

### Composition

Composes with `code-integrity-guardrail go`. Go binding for guardrail added at
`code-integrity-guardrail/references/bindings/go.md`.

### Origin

Stage 4 skill for Go components in the software-development-orchestrator pipeline.
Emerged from the whatsbot project (Go + gomobile + Android), where the Go layer
(whatsmeow client, SQLite access, message handling) needed the same systematic
treatment as the Android layer. gomobile-specific constraints were documented from
direct experience with the whatsbot build pipeline in Termux.
