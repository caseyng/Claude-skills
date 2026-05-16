# gomobile Binding Contract

This file covers the constraints and patterns for building Go packages that are bound
for Android (and iOS) via `gomobile bind`. Violations produce either compile errors or
silent runtime failures — silent failures are worse.

---

## What gomobile Can Bind

gomobile exposes Go to Java/Kotlin through a code-generation layer. Only a subset of
Go types survive the boundary.

**Supported at the boundary:**

| Go type | Java/Kotlin equivalent | Notes |
|---|---|---|
| `int`, `int32`, `int64` | `int`, `long` | Use `int64` for timestamps |
| `float32`, `float64` | `float`, `double` | |
| `bool` | `boolean` | |
| `string` | `String` | Copied at boundary |
| `[]byte` | `byte[]` | Copied at boundary |
| Go interface (exported) | Java interface | Android side implements |
| Go struct (exported, pointer) | Opaque Java object | Methods accessible, fields not |
| `error` interface | `Exception` | Always as return value, not parameter |

**Not supported — will fail at bind time or runtime:**

| Go type | Why | Alternative |
|---|---|---|
| `map[K]V` | Not bridgeable | Define a struct or use repeated Get/Set methods |
| `[]SomeStruct` | Slice of structs not bindable | Return count + Get(i) pattern, or encode as JSON `[]byte` |
| `func(...)` as parameter | Function values not bindable | Define a Go interface with one method |
| Unexported types | Not visible to gomobile | Export what crosses the boundary |
| `interface{}` / `any` | Not bridgeable | Use typed interfaces or `[]byte` + serialisation |
| Multiple return values (beyond value + error) | Not supported | Wrap in a struct |

---

## Package Structure for gomobile

The bound package must be a single Go package with no CGO dependencies. If your
implementation uses CGO (e.g., `mattn/go-sqlite3`), replace it with a pure Go
alternative (e.g., `modernc.org/sqlite`).

```
go/
  client.go          — exported types and functions (the binding surface)
  client_internal.go — unexported implementation
  go.mod             — module declaration
  go.sum
```

The package name in `go.mod` becomes the Java package name after binding:
`bind -javapkg com.caseyng.whatsbot` overrides it.

---

## Exported Type Rules

**Structs:** Fields are not accessible from Java — only methods are. Do not put data in
exported struct fields expecting Java to read them. Expose via getter methods.

```go
// Wrong — field not accessible from Java
type Client struct {
    Connected bool
}

// Correct — method accessible from Java
type Client struct {
    connected bool
}
func (c *Client) IsConnected() bool { return c.connected }
```

**Methods:** Return at most one value plus error. gomobile does not support multiple
non-error return values.

```go
// Wrong — two return values
func (c *Client) GetAndCount() (string, int, error) { ... }

// Correct — wrap in a struct
type GetResult struct {
    Value string
    Count int
}
func (c *Client) GetAndCount() (*GetResult, error) { ... }
```

**Constructors:** Return `*Type, error`. Java sees this as a factory method that throws
on error.

```go
func NewClient(dbPath string) (*Client, error) { ... }
```

---

## Callback Pattern (Go interface → Java implementation)

When the Android side needs to react to Go events (incoming message, connection status
change), define a Go interface. Java/Kotlin implements it.

```go
// Go — define the callback interface
type MessageListener interface {
    OnMessage(jid, sender, text string)
}

// Go — accept it as a parameter
func (c *Client) SetMessageListener(l MessageListener) {
    c.listener = l
}

// Go — call it (thread-safe: gomobile handles cross-thread calls)
if c.listener != nil {
    c.listener.OnMessage(jid, sender, text)
}
```

```kotlin
// Kotlin — implement the Go interface
class MyListener : Whatsbot.MessageListener {
    override fun onMessage(jid: String, sender: String, text: String) {
        // handle on whatever thread this arrives on
    }
}
client.setMessageListener(MyListener())
```

**Threading:** gomobile callbacks arrive on a Go goroutine, not the Android main thread.
The Kotlin side is responsible for dispatching to the main thread if needed
(`Handler(Looper.getMainLooper()).post { ... }`).

---

## Error Handling Across the Boundary

All errors cross the boundary as `error` (Java `Exception`). Return errors; do not panic.

```go
// Correct — error propagates to Java as an Exception
func (c *Client) Connect(phoneNumber string) error {
    if phoneNumber == "" {
        return errors.New("phone number must not be empty")
    }
    return c.impl.Connect(phoneNumber)
}
```

Java callers must handle the exception. Kotlin callers use try/catch or runCatching.

---

## Build Pipeline

```bash
# Build script — run from the go/ directory
gomobile bind \
  -target android \
  -javapkg com.caseyng.whatsbot \
  -o ../android/app/libs/whatsbot-go.aar \
  .
```

**Prerequisites:**
- Go installed in the build environment
- `gomobile` installed: `go install golang.org/x/mobile/cmd/gomobile@latest`
- Android NDK in the path (gomobile finds it via `ANDROID_HOME` or `ANDROID_NDK_HOME`)
- Run `gomobile init` once after installation

**Build environment notes (Termux/proot):**
- NDK binaries are ARM-native — `gomobile bind` runs in Termux main shell, not Alpine proot
- The `.aar` output is platform-independent (contains ARM64, ARM32, x86, x86_64 slices)
- The `.aar` is gitignored — rebuild from source; do not commit it

**After binding:**
The `.aar` is placed in `android/app/libs/`. Android build picks it up automatically
if `libs/` is in the `implementation fileTree` dependency in `build.gradle.kts`.

---

## Testing gomobile-Bound Code

Test the Go layer independently before binding. The binding adds no testable logic —
test the Go implementation with Go tests.

```go
// Stub constructor for tests — no network, no real DB
func NewTestClient(dbPath string) (*Client, error) {
    // Opens DB with seed data, no WA connection
}

// Inject fake events for testing Android-side logic
func (c *Client) SimulateIncomingMessage(jid, senderName, text string) {
    if c.listener != nil {
        c.listener.OnMessage(jid, senderName, text)
    }
}
```

`testdata/stub.db` — pre-seeded SQLite with realistic fake data (fake JIDs, display names,
timestamps). Checked into the repo. Used by `NewTestClient`. Never use real user data
as test data.
