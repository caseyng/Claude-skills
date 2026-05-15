---
skill: android-engineering
section: ecosystem
---

# Ecosystem

## Standard Library Selection

| Need | Use | Do not use | Notes |
|---|---|---|---|
| UI | Jetpack Compose + Material3 | XML layouts, Views | All new code in Compose |
| State management | `StateFlow` + `SharedFlow` | `LiveData` | LiveData is legacy; StateFlow integrates with coroutines |
| Async / threading | Kotlin Coroutines + `Flow` | `RxJava`, `AsyncTask`, `Thread` | AsyncTask removed in API 33 |
| Local database | Room | Raw SQLite cursors (for new tables) | Raw cursors acceptable for read-only access to Go-written tables |
| App preferences/config | DataStore (Preferences) | SharedPreferences | SharedPreferences has thread-safety issues, deprecated for new code |
| Dependency injection | Manual DI or Hilt | Dagger, Koin | Manual is fine for small app; Hilt when it grows |
| Navigation | Compose Navigation (`androidx.navigation.compose`) | Intent-based nav, manual back stack | |
| HTTP (if needed) | Retrofit + OkHttp | Volley, `HttpUrlConnection` | |
| Image loading (if needed) | Coil (`io.coil-kt:coil-compose`) | Glide, Picasso | Coil is Compose-native |
| JSON serialization | Kotlin Serialization (`kotlinx.serialization`) | Gson, Moshi | Kotlin Serialization is the modern default |
| Encrypted storage | `androidx.security:security-crypto` (EncryptedSharedPreferences) | Manual encryption, raw SharedPreferences | |
| Background work (deferrable) | WorkManager | `JobScheduler` directly, `AlarmManager` for non-exact work | WorkManager wraps JobScheduler/AlarmManager |
| Scheduled exact alarms | `AlarmManager.setExactAndAllowWhileIdle` | `Handler.postDelayed`, `Timer` | Requires `SCHEDULE_EXACT_ALARM` permission on Android 13+ |
| Persistent socket / connection | Foreground Service | WorkManager, `JobService` | WorkManager cannot maintain a persistent connection |
| Memory leak detection | LeakCanary (debug builds only) | — | Add as `debugImplementation` |
| Logging (debug) | Timber | `android.util.Log` directly | Timber respects debug/release, easier to control |

---

## Versioning Conventions

**`build.gradle.kts` (Kotlin DSL — use this, not Groovy):**

```kotlin
android {
    compileSdk = 35
    defaultConfig {
        minSdk = 26        // API 26 = Android 8.0, covers 99%+ of active devices
        targetSdk = 35
    }
}
```

**Use version catalogs (`libs.versions.toml`)** for dependency versions — single source
of truth, no version duplication across modules.

```toml
# gradle/libs.versions.toml
[versions]
compose-bom = "2025.05.00"
kotlin = "2.0.21"
hilt = "2.52"

[libraries]
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
```

**Compose BOM:** Always use the Compose Bill of Materials to align Compose library
versions. Individual version pinning causes version conflicts.

```kotlin
dependencies {
    val composeBom = platform(libs.compose.bom)
    implementation(composeBom)
    implementation(libs.compose.ui)
    implementation(libs.compose.material3)
    // No version needed — BOM manages it
}
```

---

## gomobile `.aar` Integration

The Go library (`whatsbot-go`) is compiled to an `.aar` via `gomobile bind` and
included as a local file dependency.

**Directory structure:**
```
app/
  libs/
    whatsbot-go.aar   ← gomobile output, do not commit binary to git
  build.gradle.kts
```

**`build.gradle.kts`:**
```kotlin
dependencies {
    implementation(files("libs/whatsbot-go.aar"))
    implementation("golang.org/x/mobile:bind:0.0.0")  // gomobile runtime (check exact artifact)
}
```

**Build script (`scripts/build-go.sh`):**
```bash
#!/bin/bash
set -euo pipefail
cd /path/to/whatsbot-go
gomobile bind -target android -o /path/to/android-app/app/libs/whatsbot-go.aar .
```

**Rules:**
- Never commit the `.aar` binary to git — add to `.gitignore`
- The `.aar` is a build artifact, regenerated from source
- gomobile boundary: no `interface{}`, channels, maps, or function types across the boundary
- The Go `Listener` interface + `Message` struct + `Client` are the only exposed types

---

## Room with Go-Written Database

The Go library owns and writes to the SQLite database. Android reads it.

**Read-only access pattern:**
```kotlin
@Database(
    entities = [WaChat::class, WaMessage::class],
    version = 1,
    exportSchema = false
)
abstract class WhatsAppDatabase : RoomDatabase() {
    abstract fun chatDao(): WaChatDao
    abstract fun messageDao(): WaMessageDao
}

// Open the existing DB — do not create a new one
val db = Room.databaseBuilder(context, WhatsAppDatabase::class.java, "whatsmeow.db")
    .createFromFile(File(context.filesDir, "whatsmeow.db"))  // existing Go DB
    .fallbackToDestructiveMigration()  // if schema changes, rebuild (read-only so OK)
    .build()
```

**Critical:** Do not run Room migrations against the Go-written tables. Room's
migration system is for Room-owned schema. Go-written tables are managed by Go.

**Separate Room DB for app-owned tables** (scheduled messages, rules, config):
```kotlin
@Database(entities = [ScheduledMessage::class, AutoReplyRule::class], version = 1)
abstract class AppDatabase : RoomDatabase()
// This DB is fully owned by Kotlin — migrations apply here
```

---

## DataStore Usage

```kotlin
// Define keys
val ONBOARDING_COMPLETE = booleanPreferencesKey("onboarding_complete")
val PAIRED_PHONE = stringPreferencesKey("paired_phone")

// Read
val isOnboarded: Flow<Boolean> = context.dataStore.data
    .map { prefs -> prefs[ONBOARDING_COMPLETE] ?: false }

// Write
suspend fun setOnboardingComplete() {
    context.dataStore.edit { prefs ->
        prefs[ONBOARDING_COMPLETE] = true
    }
}
```

DataStore operations are `suspend` — always call from a coroutine.
Never call `.first()` with `runBlocking` to make them synchronous.

---

## Timber Logging

```kotlin
// Application.onCreate()
if (BuildConfig.DEBUG) {
    Timber.plant(Timber.DebugTree())
}
// Release: no tree planted = no logging = no Logcat exposure

// Usage
Timber.d("Chat loaded: jid=%s count=%d", jid, messages.size)
Timber.w("Exact alarm permission denied — scheduling degraded")
Timber.e(exception, "Failed to send scheduled message")
```

Never log PII (JIDs, phone numbers, message content) even in debug.
