---
name: go-engineering
version: 1.0.0
description: >
  Senior Go engineering standards for production code. Module structure, error handling,
  interface design, concurrency, context propagation, testing, security, and observability.
  Includes gomobile-specific constraints for cross-language bindings.
  Compose with code-integrity-guardrail go for verification.
  Trigger on: any Go implementation task, any gomobile binding, any Go component in the pipeline.
---

# Go Engineering

## Role

Senior Go engineer. You write idiomatic, production-grade Go that compiles, handles errors
explicitly, and can be tested. You do not write clever code. You write clear code.

Compose with `code-integrity-guardrail go` on every delivery.

---

## Module Structure

```
project/
  go.mod           — module declaration, Go version, dependencies
  go.sum           — checksums (committed, never hand-edited)
  cmd/             — executables; each subdirectory is one binary (main package)
  internal/        — packages not importable outside this module
  pkg/             — packages designed to be imported by other modules
  [component]/     — domain packages at root for small modules
```

For a library (gomobile binding or shared package): no `cmd/`. Root packages are the
exported surface. Everything internal goes in `internal/`.

**go.mod:** Declare the minimum Go version explicitly. Keep it current — `go mod tidy`
after any dependency change. `go.sum` is committed; do not gitignore it.

---

## Error Handling

Go errors are values. Handle them explicitly at every call site.

```go
// Correct — explicit, early return
result, err := doThing()
if err != nil {
    return fmt.Errorf("doThing: %w", err)
}

// Wrong — ignoring error
result, _ := doThing()

// Wrong — logging and continuing as if nothing happened
result, err := doThing()
if err != nil {
    log.Printf("error: %v", err)
}
useResult(result) // result may be zero value / nil
```

**Error wrapping:** Use `fmt.Errorf("context: %w", err)` to add context. Do not wrap
more than once per call stack level — each wrap adds context, not repetition.

**Sentinel errors:** Declare with `var ErrX = errors.New("description")`. Callers check
with `errors.Is(err, ErrX)` — not string comparison.

**Error types:** Use a struct implementing `error` when the caller needs structured data
from the error (status code, entity ID). Always implement `Error() string`.

**Panics:** Do not panic in library code. Panics are for programming errors (nil receiver
on a method that requires non-nil — annotate this in the type's doc comment). Recover
panics at service entry points only (HTTP handlers, goroutine roots).

---

## Package Design

**One responsibility per package.** A package that exports types serving two unrelated
domains is two packages.

**Naming:**
- Package name = directory name. Single lowercase word. No underscores, no camelCase.
- Exported types: `MessageStore`, not `Store` (avoid stutter: `store.Store`)
- Interfaces: noun or noun phrase describing capability. `-er` suffix when it fits naturally
  (`Reader`, `Sender`). Do not force it: `MessageProcessor` not `MessageProcesser`.
- Receivers: one or two letter abbreviation of the type name. Consistent within a type.
  `func (s *Store) Get(...)` — not `func (store *Store)`, not `func (self *Store)`.
- Unexported: use the same clarity standard as exported. `parseResponse` not `pr`.

**Interfaces:**
- Define interfaces at the point of use (the package that calls, not the package that implements).
- Accept interfaces, return concrete types.
- Keep interfaces small — one or two methods. A three-method interface is a smell.
- Do not pre-define interfaces "for extensibility." Define them when a second implementation exists or is tested against.

---

## Concurrency

**Goroutines are cheap to start and expensive to leak.** Every goroutine must have a
defined exit condition. If a goroutine cannot be signalled to stop, it is a leak.

```go
// Correct — goroutine exits on context cancellation
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        case msg := <-ch:
            handle(msg)
        }
    }
}()

// Wrong — goroutine with no exit condition
go func() {
    for msg := range ch {
        handle(msg)
    }
}()
// ch is never closed → goroutine leaks
```

**Channels:**
- Buffered channels: use when the producer should not block on a slow consumer. Size the
  buffer to the maximum burst you can tolerate losing, not to infinity.
- Unbuffered channels: synchronisation points. Both sides must be ready.
- Close channels from the producer, never the consumer. Closing a channel twice panics.
- Do not send on a closed channel. Structure code so the producer owns the close.

**Mutexes:**
- `sync.Mutex` for exclusive access. `sync.RWMutex` when reads vastly outnumber writes.
- Lock as late as possible, unlock as early as possible. `defer mu.Unlock()` is correct
  when the critical section covers the rest of the function.
- Do not hold a lock across a channel send or I/O operation — deadlock risk.

**`errgroup`:** Use `golang.org/x/sync/errgroup` for parallel work that must all succeed.
The group cancels remaining goroutines on the first error.

```go
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error { return doA(ctx) })
g.Go(func() error { return doB(ctx) })
if err := g.Wait(); err != nil {
    return err
}
```

---

## Context

- `context.Context` is the first parameter of every function that does I/O, calls external
  services, or performs work that should be cancellable.
- Do not store context in a struct. Pass it explicitly down the call chain.
- Do not use `context.Background()` inside library code — receive context from the caller.
  `context.Background()` is for main functions, test setup, and top-level server handlers.
- Check `ctx.Err()` at the start of long loops. Do not perform work after the context is done.
- Use `context.WithTimeout` at service boundaries (outbound HTTP calls, DB queries) — not
  deep in business logic.

---

## Logging

Use `log/slog` (stdlib, Go 1.21+). Structured fields, not format strings.

```go
// Correct
slog.Info("message stored", "jid", jid, "size", len(body))
slog.Error("send failed", "jid", jid, "err", err)

// Wrong — unstructured
log.Printf("stored message for %s, size %d", jid, len(body))
```

**Log levels:**
- `Debug` — per-event detail useful during development, disabled in production
- `Info` — normal operational events (connection established, message sent)
- `Warn` — recoverable anomalies (retry attempt, degraded mode)
- `Error` — failures requiring attention, always include `"err", err`

Never log and return the same error. Log at the point where you handle it (top of the
call stack). Wrap with context everywhere below; log once at the top.

---

## Testing

**Table-driven tests** are the Go idiom. One test function, a slice of cases, a loop.

```go
func TestParseJID(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    JID
        wantErr bool
    }{
        {"valid", "123@s.whatsapp.net", JID{User: "123", Server: "s.whatsapp.net"}, false},
        {"empty", "", JID{}, true},
        {"no server", "123", JID{}, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseJID(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("ParseJID(%q) error = %v, wantErr %v", tt.input, err, tt.wantErr)
            }
            if !tt.wantErr && got != tt.want {
                t.Errorf("ParseJID(%q) = %v, want %v", tt.input, got, tt.want)
            }
        })
    }
}
```

**Test helpers:** Accept `*testing.T` as first parameter. Call `t.Helper()` at the top.
This makes failure messages point to the call site, not the helper.

**In-memory substitutes:** Prefer real implementations with in-memory backends (`sqlite3`
with `:memory:`, channels instead of network sockets) over mocks. Mocks verify call
signatures; in-memory backends verify behaviour.

**`testdata/` directory:** Fixtures, golden files, seed databases. Checked in. The test
binary's working directory is the package directory, so `testdata/` paths are stable.

See `references/testing.md` for patterns specific to gomobile and service testing.

---

## Security

**Randomness:** `crypto/rand` for any security-relevant randomness (tokens, session IDs,
nonces). `math/rand` is deterministic given a seed — use it only for simulations and tests.

```go
// Correct
b := make([]byte, 32)
if _, err := rand.Read(b); err != nil { ... }

// Wrong — predictable
token := fmt.Sprintf("%d", mathRand.Int63())
```

**SQL:** Never concatenate user input into SQL strings. Use parameterised queries.

```go
// Correct
row := db.QueryRowContext(ctx, "SELECT name FROM chats WHERE jid = ?", jid)

// Wrong — SQL injection
row := db.QueryRowContext(ctx, "SELECT name FROM chats WHERE jid = '"+jid+"'")
```

**File paths:** Validate that user-supplied paths do not escape the intended directory.
`filepath.Clean` alone is not sufficient — check that the result has the expected prefix.

**TLS:** Always verify certificates. Do not set `InsecureSkipVerify: true` in production.
If you need to test with a self-signed cert, use a test CA, not skip verification.

---

## SQLite Patterns

When using SQLite (production or test):

```go
// Connection string — always set these pragmas
db, err := sql.Open("sqlite3", "file:path.db?_journal_mode=WAL&_busy_timeout=5000&_foreign_keys=on")

// WAL mode: concurrent reads while a write is in progress
// busy_timeout: wait up to 5s instead of returning SQLITE_BUSY immediately
// foreign_keys: enforce referential integrity (off by default in SQLite)
```

**Prepared statements:** Prepare once, execute many times. `db.PrepareContext` at startup
for hot-path queries. Do not prepare inside loops.

**Migrations:** Run migrations at startup, before serving any requests. Keep migration
files in `internal/migrations/` as embedded SQL (`//go:embed`). Apply in order by filename.

**Read-only access to an externally-owned database:** Open with `?mode=ro` to prevent
accidental writes. Do not run migrations against a database owned by another process.

---

## gomobile

See `references/gomobile.md` for the full binding contract. Key constraints:

- Exported types and functions must use only Go types that gomobile can bind: numeric
  primitives, `string`, `bool`, `[]byte`, Go interfaces, and Go structs (as opaque objects).
- No maps, no slices of structs, no function values at the boundary.
- All errors returned across the boundary must use the `error` interface.
- Callback patterns use Go interfaces (the Android/Java side implements them).
- The gomobile package must not import CGO — pure Go only for cross-platform builds.

---

## Composition

Every delivery composes with: `/skill:code-integrity-guardrail go`

Run `go vet ./...` and `staticcheck ./...` before considering any delivery complete.
State the results explicitly. If tools are unavailable, cap all findings at FLAG.
