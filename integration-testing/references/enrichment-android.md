# Integration Testing Enrichment — Android

Android-specific tooling and approach for integration tests. Apply in Phases 3 and 4.

---

## Test Type: Instrumented Tests

Android integration tests run as instrumented tests — they execute on a device or emulator,
have access to the Android SDK, and can interact with real Android components (Services,
Activities, databases, notification system).

**Build configuration:**
```kotlin
// app/build.gradle.kts
androidTest {
    // Instrumented tests in src/androidTest/
}
testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
```

**Dependencies:**
```kotlin
androidTestImplementation("androidx.test.ext:junit:1.x.x")
androidTestImplementation("androidx.test:rules:1.x.x")
androidTestImplementation("androidx.test.espresso:espresso-core:3.x.x")
androidTestImplementation("androidx.compose.ui:ui-test-junit4:x.x.x")
androidTestImplementation("kotlinx-coroutines-test:x.x.x")
```

---

## Service Integration Tests

To test a foreground service (e.g., WhatsAppForegroundService) in integration tests:

**ServiceTestRule** starts and stops the service around each test:
```kotlin
@get:Rule
val serviceRule = ServiceTestRule()

@Test
fun incomingMessageTriggersMatchingRule() {
    val intent = Intent(context, WhatsAppForegroundService::class.java)
    serviceRule.startService(intent)
    // service is now running; interact via AppEventBus or Binder
}
```

Alternatively, bind to the service via `ServiceTestRule.bindService()` to get a Binder
reference and call service methods directly (if the service exposes a Binder).

**Stub client injection:** The service must be configured to use `StubWhatsAppClient`
in test builds. Use `BuildConfig.USE_STUB` + a flag, or inject the client via a
test-only Application class in `src/androidTest/`.

---

## Real Room Database (In-Memory)

Use in-memory Room for integration tests — same schema, same DAO, same migration logic,
but wiped between tests:

```kotlin
val db = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
    .allowMainThreadQueries()  // only in tests
    .build()
```

This is a real Room database. SQL queries, DAOs, and migrations run against it exactly
as in production. It is not a mock.

For testing interactions involving the Go-written database (whatsmeow.db), copy the
`testdata/stub.db` fixture to a temp path and open it read-only.

---

## AppEventBus / SharedFlow Collection

The foreground service publishes events to AppEventBus (a `SharedFlow` singleton).
To collect events in a test:

```kotlin
@Test
fun servicePublishesActionEventOnRuleFire() = runTest {
    val events = mutableListOf<AppEvent>()
    val job = launch { AppEventBus.events.collect { events.add(it) } }

    // trigger the interaction
    service.simulateIncomingMessage(fakeJid, "Alice", "I need help")

    advanceUntilIdle()
    job.cancel()

    assertThat(events).contains(AppEvent.AutoReplySent(fakeJid, "I'm driving"))
}
```

Use `kotlinx-coroutines-test`'s `runTest` and `advanceUntilIdle()` to drain the coroutine
scheduler before asserting.

---

## Compose UI Integration Tests

To test an end-to-end UI flow (e.g., onboarding → pairing stub → home screen):

```kotlin
@get:Rule
val composeTestRule = createAndroidComposeRule<MainActivity>()

@Test
fun onboardingFlowNavigatesToHome() {
    composeTestRule.onNodeWithText("Get Started").performClick()
    // ...
    composeTestRule.onNodeWithText("Home").assertIsDisplayed()
}
```

Use a stub client (via `BuildConfig.USE_STUB`) so the UI test does not require a real
WhatsApp connection.

---

## StubWhatsAppClient

The stub client provides the boundary between internal components and the external
WhatsApp service. In integration tests:

```kotlin
// Application.kt (androidTest variant or via BuildConfig flag)
val client = WhatsBot.newTestClient(stubDbPath)

// Inject an incoming message event into the system
client.simulateIncomingMessage(jid = "6598765432@s.whatsapp.net", senderName = "Alice", text = "Help!")

// Inject an activity change
client.simulateActivity("IN_VEHICLE")
```

`simulateIncomingMessage` fires the same event path as a real incoming message —
the service's message handler, the rules engine evaluation, and all downstream effects
(log write, reply dispatch, notification) are real.

`simulateActivity` fires the activity recognition event path.

The stub database (`testdata/stub.db`) must be seeded with realistic contacts and groups
(fake JIDs, realistic display names). It is checked into the repo and used by `newTestClient`.

---

## MANUAL_REQUIRED Conditions on Android

These contract behaviors cannot be automated — they require a real device + specific hardware
or OS conditions:

| Condition | Why it cannot be automated |
|---|---|
| Doze mode survival | Emulators do not faithfully reproduce Doze battery optimization behavior |
| Exact alarm under Doze | `AlarmManager.setExactAndAllowWhileIdle` firing during Doze requires real idle state |
| ActivityRecognition events | Hardware sensor; emulator does not produce real transitions |
| Real WA pair + message delivery | Requires a live WA account; use test SIM only |
| Foreground service under memory pressure | OS memory pressure cannot be reliably reproduced in emulator |
| Biometric prompt | Requires physical fingerprint sensor or face unlock |

For MANUAL_REQUIRED contracts, write step-by-step procedures using ADB commands where possible
to accelerate manual execution:

```bash
# Force Doze mode
adb shell dumpsys deviceidle force-idle

# Exit Doze mode
adb shell dumpsys deviceidle unforce

# Check foreground service state
adb shell dumpsys activity services | grep WhatsApp

# Trigger an exact alarm test by setting a schedule 1 minute out, then force Doze
```

---

## Test Isolation

Each integration test must start from a clean state:

```kotlin
@Before
fun setup() {
    db.clearAllTables()
    AppEventBus.reset()   // clear any buffered events
    ruleRepository.deleteAll()
}

@After
fun teardown() {
    db.close()
    serviceRule.unbindService()
}
```

Tests that leave state (rules, log entries, schedules) will cause false positives or false
negatives in subsequent tests. Use `@Before` and `@After` to enforce isolation.
