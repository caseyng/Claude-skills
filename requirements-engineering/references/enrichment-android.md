# Android Enrichment Checklist

Platform-specific concerns for Android apps. Apply after `enrichment-universal.md`.
For each item not addressed in the raw intent, propose a default and ask for confirmation.

---

## Background Operation

**Does the app need to do work with the screen off or app backgrounded?**
Android aggressively kills background processes. Work that must survive screen-off requires
a foreground service. Work that can be deferred can use WorkManager.
Default recommendation: if the app reacts to real-time events (messages, sensor changes,
connections), it needs a foreground service. Identify the correct foreground service type:
- `connectedDevice` — persistent socket or paired device connection (no time cap)
- `dataSync` — syncing data (6-hour cap, wrong for long-running connections)
- `location` — location tracking
Incorrect service type causes OS termination. This is architectural, not configurable later.

**Does the app need to survive device reboot?**
If yes, a `BOOT_COMPLETED` broadcast receiver is required.
Default: if background operation is needed, reboot persistence is almost always also needed.

---

## Battery and Power

**Battery optimisation exemption**
Apps with foreground services should request battery optimisation exemption to prevent
Doze mode from killing the service. Requires user consent.
Default: request exemption at first run with a clear explanation of why it is needed.

**Exact alarms for scheduled work**
`AlarmManager.setExactAndAllowWhileIdle` fires during Doze. `WorkManager` cannot hold a
socket and is not appropriate for time-critical delivery.
Default: if scheduled actions must fire at a specific time, use `AlarmManager` with
`SCHEDULE_EXACT_ALARM` permission. Graceful degradation if permission is denied.

---

## Permissions

**Which permissions does the app require?**
List every permission. For each: why it is needed, when it is requested, what happens if denied.
Default: request permissions at the point of use, not at app start. Explain why before requesting.
Graceful degradation when denied — app must not crash or silently break.

**Runtime permissions vs install-time permissions**
Dangerous permissions (location, contacts, microphone, activity recognition) must be
requested at runtime with user explanation.
Default: request at the moment the feature needs it, with an explanation the user can understand.

---

## Data Storage

**Sensitive data (credentials, session keys, tokens)**
Must use `EncryptedSharedPreferences` or Android Keystore.
Default: all credentials in `EncryptedSharedPreferences`. Never in plain SharedPreferences,
never in a database field without encryption, never in a file.

**User preferences and settings**
Default: `DataStore<Preferences>` — thread-safe, async, replaces SharedPreferences.

**Structured app data**
Default: Room with Kotlin coroutines. Full ownership, migrations, type safety.

**Data that must not leave the device**
Default: `android:allowBackup="false"` in the manifest prevents silent ADB or cloud backup
of app data, including databases containing session keys.

---

## Network Security

**Cleartext traffic**
Default: disallow all cleartext. Enforce via `network_security_config.xml` with
`cleartextTrafficPermitted="false"`. There are almost no valid reasons to allow cleartext
on Android apps built today.

---

## Architecture

**Screen rotation and process death**
ViewModels survive rotation. Persistent state must survive process death.
Default: MVVM with StateFlow/SharedFlow. Any state the user expects to survive must be
persisted — do not rely on in-memory state for anything important.

**Background-to-foreground communication**
Services cannot directly update UI. Use a SharedFlow singleton (AppEventBus pattern)
that the service writes to and ViewModels collect from.
Default: AppEventBus as a process-scoped singleton for all service → UI events.

---

## Security Baseline

**Component exposure**
Default: `android:exported="false"` on all components except those that must receive
external intents (e.g. `BOOT_COMPLETED` receiver).

**PendingIntents**
Default: `FLAG_IMMUTABLE` on all PendingIntents. Mutable PendingIntents are a security risk.

**App lock**
If the app handles sensitive data or performs actions on the user's behalf, biometric lock
is appropriate. Default: `BiometricPrompt` with PIN fallback, opt-in at first run.

---

## Testing

**Doze and background testing**
Doze behaviour cannot be fully tested in an emulator. Real device testing is required
for anything that depends on exact alarms or foreground service survival.
Default: flag Doze-dependent features as requiring manual Tier 3 testing on a real device.

**Minimum SDK**
State the minimum supported Android version. Features used must be available on the minimum SDK,
or have fallbacks. Newer APIs without fallbacks on the minimum SDK cause crashes.
Default: identify the minimum SDK early — it constrains every API choice in implementation.
