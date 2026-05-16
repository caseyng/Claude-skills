# System Design Enrichment — Android

Android-specific concerns for component decomposition and cross-cutting concerns.
Apply after `enrichment-universal.md`.

---

## Process Model

Is this one Android process or multiple?

Android allows multiple processes within an app (`android:process` attribute). In most apps,
all components run in the same process. Running a service in a separate process provides
OS-level lifecycle isolation but eliminates in-memory state sharing — components in separate
processes must communicate via IPC (AIDL, Messenger, or a file-based channel).

**Default:** Single process. The foreground service runs in the same process as the UI.
State is shared via a process-scoped singleton (AppEventBus, singleton repositories).
Separate processes only if there is a specific reason: the service must continue when the
main process is killed, or requires a different permission domain.

**If single process:** components share state via in-memory singletons. Define which
component owns each singleton.

**If multiple processes:** all cross-process interactions must be explicit interaction contracts
using a defined IPC mechanism. There is no shared in-memory state across process boundaries.

---

## Database File Ownership

If the app has multiple data stores (app-owned DB + a DB written by a Go/native layer),
each database file has exactly one owner.

The owner:
- Controls the schema
- Runs migrations
- Writes data

**Go-written database (e.g., whatsmeow.db):** the Go layer owns it. The Android layer reads
it as a reader only — via a read-only cursor or read-only Room access. The Android layer does
not run migrations on this database. If the schema changes, the Go layer migrates it.

**App-owned database (e.g., Room database):** the Android layer owns it. The Go layer does
not write to it.

Mixing ownership (Android writes into a Go-owned table, or Go writes into an Android-owned table)
is a defect by design — it creates a schema governance conflict.

**Questions to resolve:**
- What databases does this app have?
- For each: which layer (Android, Go, other) owns it?
- For cross-reads (Android reading Go DB, or vice versa): what is the read interface?

---

## Service → UI Communication

A foreground service cannot update the UI directly. It runs on its own thread in the same
process (if single-process) but has no reference to the Activity or ViewModel.

**Default pattern:** AppEventBus — a `SharedFlow` singleton at the Application scope.
- Service writes events to the AppEventBus
- ViewModels collect from AppEventBus via `lifecycleScope`
- The Application class owns the AppEventBus singleton

One component (Application) owns the bus. All other components use it as a shared channel.

**Questions to resolve:**
- Which component owns the AppEventBus singleton (default: Application class)?
- What event types does the service publish?
- Which ViewModels subscribe to which event types?
- Is there any state (not just events) that the service must expose to the UI? If so, which component holds it?

---

## Foreground Service Ownership

There is one foreground service. It is the single owner of:
- The connection to the external service (WhatsApp, BLE device, etc.)
- The connection state
- The reconnection logic

No other component holds a copy of connection state. They read from the foreground service's
state or receive events through the AppEventBus.

**Questions to resolve:**
- What does the foreground service own? List explicitly.
- How does the UI read connection state — event from AppEventBus, or exposed as a flow on a shared object?
- If the service publishes connection state as an event, what events are defined?

---

## Android Component Exposure

Every Android component (Activity, Service, Receiver, Provider) is either exported or not.
Exported components can receive external intents from other apps. Unexported components cannot.

**Default:** `android:exported="false"` on everything except:
- `BOOT_COMPLETED` broadcast receiver — must be exported to receive the system broadcast
- Any Activity the user launches from a launcher — must be exported

**Questions to resolve:**
- Which receivers must be exported to receive system broadcasts?
- Are there any Activities or Services that other apps must be able to launch?
- For each exported component: what is the threat model for receiving an external intent?

---

## State Persistence and Android Lifecycle

Android kills processes. State that must survive must be persisted.

Three levels of state persistence:
1. **In-memory only** — lost on process death and rotation (acceptable for transient UI state)
2. **ViewModel** — survives rotation, lost on process death
3. **Persisted** (DataStore, Room) — survives process death and device reboot

Every piece of user-visible state must be assigned to one of these levels.

**Default:**
- User preferences and settings: DataStore (survives process death, async, type-safe)
- Structured app data (rules, schedules, log entries): Room
- Credentials and session keys: EncryptedSharedPreferences
- Transient UI state (dialog visibility, scroll position): ViewModel or in-memory only

**Questions to resolve:**
- For each piece of app state: which persistence level?
- Which component is responsible for reading and writing each persisted state?
- On process restore, who reads persisted state and restores the in-memory model?

---

## Foreground Service Type

The `foregroundServiceType` attribute is architectural — the OS enforces it.
The wrong type causes OS termination.

| Type | Use case | Constraints |
|---|---|---|
| `connectedDevice` | Persistent connection to an external device or service (Bluetooth, socket, websocket) | No time cap |
| `dataSync` | Syncing data to/from a server | 6-hour cap — wrong for long-running connections |
| `location` | Location tracking | Requires location permission |
| `mediaPlayback` | Music/video | Must be playing media |

**Default:** `connectedDevice` for any app that maintains a persistent socket or WebSocket connection.
`dataSync` is incorrect for persistent connections — the 6-hour cap will kill the service.

This is not configurable at runtime. The type is declared in the manifest and cannot be changed
without an app update. Get it right here.

---

## Boot Persistence

If the foreground service must restart after device reboot, a `BOOT_COMPLETED` receiver is required.

The receiver:
- Must be exported
- Starts the foreground service when the device boots
- Should respect user preferences (if the user disabled the service, do not restart it)

**Default:** if the app has a foreground service that must survive device reboot, a `BOOT_COMPLETED`
receiver is required. Assign it as a component or as part of the foreground service component.

**Questions to resolve:**
- Must the service restart after reboot?
- If yes: which component is the `BOOT_COMPLETED` receiver?
- Should the receiver check user preferences before starting the service?
