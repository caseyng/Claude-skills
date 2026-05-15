# android-engineering

**Version:** 1.0.0

Senior Android engineering standards for Kotlin + Jetpack Compose apps. Invoked as `/skill:android-engineering`.

---

## Purpose

This skill makes Claude behave as a thoughtful senior Android engineer when writing, reviewing, or designing Android code. It surfaces design problems, enforces platform norms, and applies correct Material Design 3 patterns — not just style enforcement.

Scope: **local design integrity** — ViewModels, Composables, Repositories, Services, and their interactions. Does not govern CI/CD, Play Store policy, or server-side concerns.

---

## What It Covers

| Area | Details |
|---|---|
| Architecture | MVVM layers, StateFlow, ViewModelScope, Repository pattern, process death, single-Activity |
| Platform norms | Doze, foreground service types (Android 14/15 hard requirements), exact alarms, timezone, permissions, notification channels |
| Design & UX | Material Design 3 Expressive (2025+), affordance rule, touch targets, navigation, accessibility, error/empty states |
| Security | Keystore, EncryptedSharedPreferences, ProGuard, Logcat exposure, network_security_config, Intent security |
| Ecosystem | Standard library selection, version catalogs, Compose BOM, gomobile .aar integration, Room with Go-written DB |
| Testing | ViewModel unit tests (JUnit + runTest), in-memory Room integration tests, Compose UI tests, fakes over mocks |

---

## How to Invoke

```
/skill:android-engineering
```

Invoke whenever:
- Writing new ViewModels, Composables, Repositories, or Services
- Reviewing or refactoring existing Android/Kotlin code
- Making architecture decisions (layer boundaries, DI, state management)
- Designing UI screens or navigation flows
- Handling Android platform concerns (background work, scheduling, permissions)

---

## Composition

This skill **requires** `code-integrity-guardrail kotlin`. The SKILL.md loads it automatically. The guardrail handles universal code quality and security patterns plus Kotlin/Android-specific violations (SQL injection in Room, WebView JS, null safety, Logcat exposure, Intent injection).

---

## Reference Files

| File | Covers |
|---|---|
| `references/architecture.md` | MVVM layers, StateFlow patterns, ViewModel rules, Repository interface, coroutine scoping, process death, single-Activity navigation |
| `references/platform-norms.md` | Doze mode, foreground service types and Android 14/15 restrictions, exact alarms (SCHEDULE_EXACT_ALARM), timezone handling, permissions, notification channels |
| `references/design-ux.md` | Affordance rule, Material Design 3 Expressive, button hierarchy, touch targets (48dp min), navigation patterns, accessibility, empty/error states |
| `references/security.md` | Logcat exposure, EncryptedSharedPreferences, Android Keystore, biometric auth, network security config, ProGuard, Intent security |
| `references/ecosystem.md` | Library selection table (what to use and what not to), version catalogs, Compose BOM, gomobile .aar integration, Room with Go-owned DB, DataStore, Timber |
| `references/testing.md` | Test pyramid, ViewModel unit tests, fake repositories, MainDispatcherRule, in-memory Room integration tests, Compose UI tests, mirror test |

---

## Design Rationale

**Why foreground service types get a hard-requirement flag?** Android 14+ kills services
with missing or wrong types. This is a deploy-time failure with no error message. It must
be caught before any code ships, not discovered when the service stops working on a
newer Android version.

**Why affordance is the first design rule?** It is the most common UX failure in real
Android apps (including first-party Google apps). A screen that functions correctly but
communicates nothing about what is interactive is a broken user experience. Naming it
first ensures it is never treated as polish.

**Why fakes over mocks in tests?** Mocks couple tests to implementation details.
Changing how a ViewModel calls a Repository breaks mocked tests even when the observable
behavior is unchanged. Fakes test behavior contracts, not call sequences.
