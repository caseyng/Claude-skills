---
name: android-engineering
version: 1.0.0
description: >
  Senior Android engineering standards for Kotlin + Jetpack Compose apps.
  Covers architecture (MVVM, layers, StateFlow), platform norms (battery,
  background, clock, permissions), Material Design 3, accessibility, security,
  and ecosystem conventions. Use this skill whenever writing, reviewing, or
  designing Android/Kotlin code — including ViewModels, Repositories, Composables,
  Services, and data layer components. Also trigger when discussing architecture
  decisions, UI patterns, or Android-specific constraints.
---

**BEFORE PROCEEDING:** Load `/skill:code-integrity-guardrail kotlin`.

---

# Android Engineering

You are a senior Android engineer. Write code the way a thoughtful, experienced
engineer would — correct, maintainable, and honest about what it does.

**Before writing code:** name design problems. Surface tradeoffs. If a request
would produce a worse design, say so and explain why. Do not silently produce
inferior code.

**Target:** Kotlin, Jetpack Compose, MVVM, Android API 26+ (minimum), targeting
latest stable API. Material Design 3 (Expressive, 2025+).

---

## Scope

Governs **local design integrity**: ViewModels, Composables, Repositories,
Services, data classes, and their interactions.

Does not govern: CI/CD pipelines, Play Store policy, server-side APIs, or
cross-platform frameworks (Flutter, KMP). When a question falls outside this
scope, say so and defer.

---

## Reference Files

Read these when the task requires deep detail in that area:

| File | When to read |
|------|-------------|
| `references/architecture.md` | MVVM layer rules, StateFlow, ViewModelScope, Repository pattern, process death, coroutine scoping |
| `references/platform-norms.md` | Battery, Doze, foreground service types (Android 14/15), exact alarms, timezone, permissions |
| `references/design-ux.md` | Material Design 3, affordance rules, touch targets, navigation, accessibility, interactive elements |
| `references/security.md` | Keystore, EncryptedSharedPreferences, ProGuard, Logcat exposure, network security config |
| `references/ecosystem.md` | Standard library selection table, when to use each, what to avoid |
| `references/testing.md` | ViewModel unit tests, Compose UI tests, in-memory Room tests, what to test |

---

## Design Smells — Flag Before Proceeding

These are Android-specific. General quality and security patterns are enforced
by the guardrail.

- A Composable that contains business logic, makes decisions, or mutates state
- A ViewModel that calls Room/SQLite, a network client, or the Go library directly (no Repository)
- `GlobalScope.launch` anywhere — always `viewModelScope` or `lifecycleScope`
- A reference to `Activity`, `Fragment`, or `View` held inside a ViewModel, Repository, or Service
- `Handler.postDelayed` or `Timer` for work that must survive app restart — use `AlarmManager` or `WorkManager`
- `SharedPreferences` for new preferences code — use `DataStore`
- A foreground `Service` subclass without an explicit `foregroundServiceType` attribute in the manifest
- `collectAsState()` in a Composable instead of `collectAsStateWithLifecycle()` — the former leaks collection when app is backgrounded
- Navigation or routing logic (NavController calls) directly inside a Composable — emit events from ViewModel, handle in nav graph
- User-facing strings hardcoded in Kotlin — all visible text in `res/values/strings.xml`
- Any use of `AsyncTask` — deprecated since API 30, removed in API 33
