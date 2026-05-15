---
version: 1.0.0
skill: code-integrity-guardrail / kotlin
---

# Kotlin / Android Guardrail Bindings

Kotlin + Android concrete additions to the universal code-integrity-guardrail.
Only records what the universal skill cannot express without language-specific
syntax, tooling, or Android platform behavior.

---

## Phase 0: Kotlin / Android Security Tooling

Run before delivering any Kotlin/Android code.

| Tool | Command | What it catches |
|---|---|---|
| Android Lint | `./gradlew lint` | Security issues, API misuse, deprecated APIs, missing permissions |
| Detekt | `./gradlew detekt` | Code smells, complexity, style issues |
| Dependency audit | `./gradlew dependencyCheckAnalyze` (OWASP plugin) | Known CVEs in dependencies |

**Minimum bar:** Android Lint with no `Error` severity findings on security checks.

**When bash is unavailable:** State this. Cap all findings at FLAG or UNCERTAIN.

---

## Security Syntax (Category 1)

### SQL Injection via Room Raw Queries

Room's `@Query` is safe when the query is a string literal. Raw queries with
string concatenation or format strings are injection vectors.

```kotlin
// Violations
@RawQuery
fun searchMessages(query: SupportSQLiteQuery): List<Message>

// Called with user input — injection vector
val rawQuery = SimpleSQLiteQuery("SELECT * FROM messages WHERE body LIKE '$userInput'")
dao.searchMessages(rawQuery)

// Correct — parameterized
@Query("SELECT * FROM messages WHERE body LIKE :pattern")
suspend fun searchMessages(pattern: String): List<Message>

// Called safely
dao.searchMessages("%$userInput%")  // Room escapes the parameter
```

**Exception:** `@RawQuery` with a compile-time constant query string is not a
violation — e.g., `SimpleSQLiteQuery("SELECT * FROM messages ORDER BY timestamp DESC LIMIT 50")`.

---

### WebView JavaScript + Untrusted Content

```kotlin
// Violations
webView.settings.javaScriptEnabled = true
webView.loadUrl(userProvidedUrl)

webView.addJavascriptInterface(nativeObject, "Android")  // exposes native bridge
webView.settings.allowFileAccess = true                  // file:// access from web
webView.settings.allowUniversalAccessFromFileURLs = true // bypasses same-origin
```

```kotlin
// Correct — only enable JS when loading controlled content
webView.settings.javaScriptEnabled = true  // acceptable if loadUrl is a hardcoded https:// URL
webView.loadUrl("https://your-controlled-domain.com/help")

// If loading user-provided URLs, disable JS entirely or use Chrome Custom Tabs
```

---

### Hardcoded Secrets in Resources

```kotlin
// Violations
// In strings.xml:
// <string name="api_key">sk-abc123</string>

// In code:
val apiKey = "sk-abc123"
val token = BuildConfig.DEBUG_TOKEN  // only safe if the BuildConfig field is not a real token
```

```kotlin
// Correct — load from environment at build time or from secure storage at runtime
// build.gradle.kts: buildConfigField("String", "BASE_URL", "\"https://api.example.com\"")
// Secrets go in local.properties (gitignored) and loaded via:
val properties = Properties().apply { load(project.file("local.properties").inputStream()) }
buildConfigField("String", "SECRET_KEY", "\"${properties["secret_key"]}\"")
// Or, at runtime: load from EncryptedSharedPreferences
```

---

### Logcat Exposure (Android-specific)

Logcat is world-readable by other apps on the device on debug builds (pre-Android 4.1).
On modern Android, third-party apps cannot read other apps' logs — but the device owner,
rooted apps, and ADB can always read them.

```kotlin
// Violations
Log.d("Auth", "Token: $authToken")
Log.i("User", "Phone: ${contact.phoneNumber}")
Log.e("WA", "Message from ${sender.jid}: $messageBody")

// Correct — log metadata, not content
Log.d("Auth", "Token refresh completed")
Log.i("User", "Contact loaded: id=${contact.id}")
Log.e("WA", "Message receive failed: sender=${sender.jid.take(4)}..., error=${e.javaClass.simpleName}")
```

---

### Intent Injection (Android-specific)

```kotlin
// Violation — using untrusted intent data to start activities or services
val className = intent.getStringExtra("target_class")
val target = Intent().setClassName(packageName, className!!)  // arbitrary class execution
startActivity(target)

// Correct — validate against an allowlist of known safe classes
val allowedClasses = setOf("com.example.ChatActivity", "com.example.SettingsActivity")
val className = intent.getStringExtra("target_class") ?: return
if (className !in allowedClasses) return
startActivity(Intent().setClassName(packageName, className))
```

---

### Pending Intent Mutability (Android 12+)

```kotlin
// Violation — FLAG_MUTABLE allows other apps to modify the intent
PendingIntent.getActivity(context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT)

// Correct
PendingIntent.getActivity(
    context, 0, intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
```

---

## Logic Syntax (Category 2)

### Null Safety Violations

```kotlin
// Violations — force unwrap without justification
val name = bundle.getString("name")!!
val view = findViewById<TextView>(R.id.label)!!

// Correct — handle null explicitly
val name = bundle.getString("name") ?: return  // or ?: "default"
val view = findViewById<TextView>(R.id.label) ?: run {
    Timber.e("label view not found")
    return
}
```

Force unwrap (`!!`) is a code smell in almost all cases. The only acceptable use
is when null genuinely represents a programming error (not missing data) and a crash
is preferable to silent incorrect behavior. Document why.

### Mutable State Exposure

```kotlin
// Violation — exposes mutable backing field
class ChatViewModel : ViewModel() {
    val _uiState = MutableStateFlow(ChatListUiState.Loading)  // public mutable
}

// Correct — backing field private, expose immutable
class ChatViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<ChatListUiState>(ChatListUiState.Loading)
    val uiState: StateFlow<ChatListUiState> = _uiState.asStateFlow()
}
```

---

## Architecture Syntax (Category 3)

### Context Leak in Long-Lived Objects

```kotlin
// Violation — Activity context held in ViewModel (Activity can be GC'd, ViewModel lives longer)
class ChatViewModel(private val context: Context) : ViewModel()

// Also violation — storing View reference
class ChatViewModel : ViewModel() {
    var recyclerView: RecyclerView? = null  // View outlived by ViewModel
}

// Correct — use Application context only if context is needed
class ChatViewModel(application: Application) : AndroidViewModel(application) {
    private val appContext = application.applicationContext
}
```

### GlobalScope Usage

```kotlin
// Violation — work leaks beyond ViewModel lifecycle, cannot be cancelled
GlobalScope.launch { repo.loadChats() }

// Correct
viewModelScope.launch { repo.loadChats() }
lifecycleScope.launch { repo.loadChats() }  // in Activity/Fragment only
```

---

## Quality Syntax (Category 4)

### collectAsState vs collectAsStateWithLifecycle

```kotlin
// Violation — continues collecting when app is backgrounded (battery drain, potential crash)
val uiState by viewModel.uiState.collectAsState()

// Correct — stops collecting when lifecycle is below STARTED
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

### String Resources

```kotlin
// Violation — hardcoded user-visible string
Text("No messages yet")
Text("Schedule message")

// Correct
Text(stringResource(R.string.no_messages))
Text(stringResource(R.string.schedule_message_label))
```

---

## Common Kotlin / Android Hallucinations

| Hallucinated name | Correct artifact |
|---|---|
| `androidx.compose:compose-bom` | `androidx.compose:compose-bom` (correct — but version changes frequently, verify) |
| `coil-kt:coil` | `io.coil-kt:coil-compose` for Compose integration |
| `androidx.lifecycle:lifecycle-viewmodel-ktx` | Real — but check current version; `lifecycle-runtime-compose` also needed for `collectAsStateWithLifecycle` |
| `kotlinx.coroutines:coroutines-android` | `org.jetbrains.kotlinx:kotlinx-coroutines-android` |
| `androidx.navigation:navigation-compose` | Real — but verify against Compose BOM compatibility matrix |
| `gomobile/bind` Maven artifact | Does not exist — gomobile `.aar` is a local file dependency |
| `androidx.room:room-rxjava3` | Real but for RxJava; use `room-ktx` for coroutines |
| `hilt-lifecycle-viewmodel` | Removed — use `@HiltViewModel` directly with `hilt-android` |

Any unfamiliar `androidx.*` artifact should be verified at [developer.android.com/jetpack](https://developer.android.com/jetpack) before referencing.

---

## Android Pre-Flight Checklist

Extends the universal phases. Items here are Android/Kotlin-specific.

**Phase 1 — Security (Android-specific):**
- [ ] No user input interpolated into Room `@RawQuery` strings
- [ ] WebView JS only enabled for controlled, hardcoded HTTPS URLs
- [ ] No secrets in `strings.xml`, `BuildConfig`, or Kotlin string literals
- [ ] No PII logged via `Log.*` or `Timber.*`
- [ ] All `PendingIntent` uses include `FLAG_IMMUTABLE`
- [ ] All `<service>` and `<receiver>` tags have explicit `android:exported`
- [ ] Foreground service has `android:foregroundServiceType` in manifest (Android 14+)
- [ ] `network_security_config.xml` exists and sets `cleartextTrafficPermitted="false"`

**Phase 2 — Logic (Android-specific):**
- [ ] No `!!` force unwrap without documented justification
- [ ] No `Activity`/`Fragment`/`View` references held in `ViewModel` or `Repository`
- [ ] No `GlobalScope.launch` — always `viewModelScope` or `lifecycleScope`
- [ ] All DB access on `Dispatchers.IO`
- [ ] No `runBlocking` on main thread

**Phase 3 — Architecture (Android-specific):**
- [ ] ViewModel does not call Room/network directly — via Repository
- [ ] Repository is injected as interface, not constructed internally
- [ ] No business logic in Composable functions
- [ ] `collectAsStateWithLifecycle()` not `collectAsState()` in all Composables

**Phase 4 — Quality (Android-specific):**
- [ ] All user-visible strings in `res/values/strings.xml`
- [ ] All tappable elements have ripple / visual affordance cue
- [ ] All `Image` and meaningful `Icon` have `contentDescription`
- [ ] Notification channels created in `Application.onCreate()`
- [ ] `LeakCanary` in `debugImplementation` dependencies

---

## Android Pressure Scenarios

### "Just use a static singleton for the database"

**Risk:** Static context reference = Activity leak. Static mutable state = threading bugs.
**Alternative:** Application-scoped singleton via `Application.onCreate()` or Hilt `@Singleton`.
The database instance is application-scoped, not process-static.

---

### "Just use SharedPreferences, DataStore is complicated"

**Risk:** SharedPreferences has known thread-safety issues with `apply()`. Can produce
`ANR` if called on the main thread with `commit()`.
**Alternative:** DataStore is suspend-based — async by design, no thread safety issues.
The API surface is small: `edit { prefs[KEY] = value }` to write, `data.map { prefs[KEY] }` to read.

---

### "Just store the token in SharedPreferences, it's a personal app"

**Risk:** Plaintext storage. Readable via ADB on debug builds, readable on rooted devices.
**Alternative:** `EncryptedSharedPreferences` — one extra line of setup, same API.
For a personal app the threat model is low, but the cost of doing it right is also zero.

---

### "Just set `android:exported="true"` to make it work"

**Risk:** Exported components are reachable by any other app on the device — potential
data exfiltration or unintended control of your service.
**Alternative:** Debug why the component needs to be reached from outside your app.
BroadcastReceivers for system events (`BOOT_COMPLETED`) only need `exported="false"` with
the correct intent-filter — the system can still deliver the broadcast.

---

### "Just use `!!` here, we know it's not null"

**Risk:** Every `!!` is a crash waiting to happen when an assumption changes.
**Alternative:** `?: return`, `?: error("invariant: X must not be null here")`, or
restructure so null is impossible at the type level. If null genuinely represents a
programming error, use `requireNotNull(x) { "reason" }` — it throws `IllegalArgumentException`
with a message instead of a bare `NullPointerException`.
