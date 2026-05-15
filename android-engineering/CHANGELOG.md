# Changelog

## 1.0.0 — 2026-05-15

Initial release.

Covers: MVVM architecture with StateFlow/SharedFlow, single-Activity navigation,
Repository pattern, coroutine scoping rules, process death handling, Doze mode
and battery optimization, foreground service types (Android 14/15 hard requirements),
exact alarm permissions (Android 13+), timezone handling with java.time, permission
request patterns, notification channels, Material Design 3 Expressive (2025),
affordance rules, touch targets, accessibility (TalkBack, content descriptions),
navigation patterns, security (Keystore, EncryptedSharedPreferences, ProGuard,
Logcat exposure, network_security_config, Intent/PendingIntent security), standard
ecosystem library selection, Compose BOM, gomobile .aar integration, Room with
Go-written databases, DataStore, Timber logging, and testing strategy (ViewModel
unit tests, in-memory Room integration tests, Compose UI tests, fakes over mocks).

Composes with `code-integrity-guardrail kotlin` binding (also created in this release)
which covers: Room SQL injection, WebView JS security, hardcoded secrets, Logcat PII
exposure, Intent injection, PendingIntent mutability, null safety (`!!` force unwrap),
mutable state exposure, context leaks, GlobalScope, collectAsState vs
collectAsStateWithLifecycle, string resources, and common Kotlin/Android hallucinations.
