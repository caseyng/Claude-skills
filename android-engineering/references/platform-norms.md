---
skill: android-engineering
section: platform-norms
---

# Platform Norms

## Battery & Background: Doze Mode

When the device is unplugged, screen off, and stationary, Android enters Doze.
During Doze, the system defers: network access, wakelocks, standard alarms,
JobScheduler jobs, SyncAdapter syncs.

**Maintenance windows:** Doze periodically wakes to a maintenance window where
deferred work runs. Windows shrink over time. Apps cannot predict or control them.

**What survives Doze:**
- Foreground services (with visible notification) — fully exempt
- FCM high-priority messages — exempt
- `SCHEDULE_EXACT_ALARM` with user-granted permission — fires at exact time

**What does not survive Doze:**
- `WorkManager` tasks — deferred until maintenance window (acceptable for non-urgent work)
- Standard `AlarmManager` alarms without exact permission — deferred
- Background network calls — blocked entirely

**Rule:** For a persistent WhatsApp connection, a foreground service is mandatory.
WorkManager is for deferrable background tasks only.

---

## Foreground Service Types (Android 14+ — HARD requirement)

Android 14 requires every foreground service to declare an explicit type.
Missing type = service killed on Android 14+.

Declare in `AndroidManifest.xml`:
```xml
<service
    android:name=".WhatsAppService"
    android:foregroundServiceType="connectedDevice"
    android:exported="false" />
```

And request the corresponding permission:
```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />
```

**Type selection guide:**

| Use case | Type | Notes |
|---|---|---|
| Persistent socket / WhatsApp connection | `connectedDevice` | No runtime cap |
| File upload/download sync | `dataSync` | **6-hour cap in Android 15** |
| Media playback | `mediaPlayback` | No cap |
| Location tracking | `location` | Requires ACCESS_FINE_LOCATION |
| Camera | `camera` | Requires CAMERA |
| Custom / doesn't fit above | `specialUse` | Requires Play Store justification |

**Android 15 critical:** `dataSync` foreground services are capped at 6 hours per
24-hour period. After 6 hours, the system stops the service automatically.
For any persistent connection, use `connectedDevice`, not `dataSync`.

---

## Foreground Service: Background Start Restrictions (Android 12+)

Apps cannot start a foreground service while in the background, with narrow exceptions:

**Allowed background starts:**
- App transitions from a user-visible state (Activity, bound service)
- App receives a high-priority FCM message
- App is on the battery optimization exemption list
- BOOT_COMPLETED receiver (with restrictions — see below)

**BOOT_COMPLETED restrictions (Android 15+):**
Apps targeting Android 15 cannot start `dataSync`, `mediaPlayback`, or `camera`
foreground services from `BOOT_COMPLETED`. Use `connectedDevice` or bind to an
Activity-started service instead.

**Correct pattern for auto-start on boot:**
```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // Use connectedDevice type — not restricted from BOOT_COMPLETED
            val serviceIntent = Intent(context, WhatsAppService::class.java)
            context.startForegroundService(serviceIntent)
        }
    }
}
```

---

## Battery Optimization Exemption

For persistent socket connections, request exemption from battery optimization.
Without it, the OS may kill the app after extended inactivity even with a foreground service.

```kotlin
fun requestBatteryOptimizationExemption(context: Context) {
    val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
    if (!pm.isIgnoringBatteryOptimizations(context.packageName)) {
        val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
            data = Uri.parse("package:${context.packageName}")
        }
        context.startActivity(intent)
    }
}
```

**Never assume the exemption is granted.** Check `isIgnoringBatteryOptimizations`
before relying on it. Request at first run with clear explanation to the user.

---

## Exact Alarms (Scheduled Messages)

`SCHEDULE_EXACT_ALARM` is **not pre-granted** on Android 13+ for most apps.
Users must manually grant it in Settings → Apps → Special app access → Alarms & reminders.

```kotlin
// Check at runtime before scheduling
fun canScheduleExactAlarms(context: Context): Boolean {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val am = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
        am.canScheduleExactAlarms()
    } else {
        true
    }
}

// If not granted, send user to settings — never silently fall back
fun requestExactAlarmPermission(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val intent = Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM)
        context.startActivity(intent)
    }
}
```

**Listen for grant/revoke:**
```kotlin
// In manifest
<receiver android:name=".ExactAlarmPermissionReceiver"
          android:exported="false">
    <intent-filter>
        <action android:name="android.app.action.SCHEDULE_EXACT_ALARM_PERMISSION_STATE_CHANGED" />
    </intent-filter>
</receiver>
```

**Correct AlarmManager usage:**
```kotlin
fun scheduleMessage(context: Context, triggerAtMillis: Long, pendingIntent: PendingIntent) {
    val am = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        am.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerAtMillis, pendingIntent)
    } else {
        am.setExact(AlarmManager.RTC_WAKEUP, triggerAtMillis, pendingIntent)
    }
}
```

Use `RTC_WAKEUP` to fire even when device is in Doze maintenance window.
Use `setExactAndAllowWhileIdle` — `setExact` alone is deferred during Doze.

---

## Time & Timezone

**Always use `java.time` (API 26+):**

```kotlin
// Store as UTC epoch millis
val sendAtUtc = ZonedDateTime.of(2026, 5, 15, 14, 30, 0, 0, ZoneId.of("UTC"))
    .toInstant()
    .toEpochMilli()

// Display in user's local timezone
val local = Instant.ofEpochMilli(sendAtUtc)
    .atZone(ZoneId.systemDefault())
    .format(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT))
```

**Rules:**
- Store timestamps as UTC epoch millis (Long) — never as formatted strings
- Display using `ZoneId.systemDefault()` — never hardcode a timezone
- Never use `java.util.Date`, `java.util.Calendar`, or `SimpleDateFormat`
- Never do raw epoch arithmetic — use `Duration`, `Period`, `ChronoUnit`

**Timezone database:** Android updates via APEX modules (Android 10+). Governments
change timezone rules. Never assume a timezone offset is constant — always recompute.

---

## Permissions: Request Correctly

**Rules:**
1. Request contextually — only when the user takes an action that needs it
2. Never request on app launch (rejected by Play Store policy)
3. Show rationale before requesting if `shouldShowRequestPermissionRationale()` is true
4. Always handle denied case gracefully — degrade feature, don't crash

**Android 13+ notifications (POST_NOTIFICATIONS):**
```kotlin
// Must request at runtime — users can deny
ActivityCompat.requestPermissions(
    activity,
    arrayOf(Manifest.permission.POST_NOTIFICATIONS),
    REQUEST_CODE_NOTIFICATIONS
)
```

Notifications silently fail on Android 13+ if `POST_NOTIFICATIONS` not granted.
Check before creating any notification.

**Notifications: channels required (Android 8+):**
```kotlin
fun createNotificationChannels(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val channel = NotificationChannel(
            CHANNEL_ID_PERSISTENT,
            "WhatsApp Connection",
            NotificationManager.IMPORTANCE_LOW  // low = no sound, appears in drawer
        ).apply {
            description = "Maintains background WhatsApp connection"
        }
        val nm = context.getSystemService(NotificationManager::class.java)
        nm.createNotificationChannel(channel)
    }
}
```

Create channels in `Application.onCreate()` — idempotent, safe to call repeatedly.
Missing channel = notification silently dropped on Android 8+.

**Importance levels:**
- `IMPORTANCE_HIGH` — heads-up notification + sound (use for incoming messages user must see)
- `IMPORTANCE_DEFAULT` — sound, no heads-up
- `IMPORTANCE_LOW` — no sound (use for persistent service notification)
- `IMPORTANCE_MIN` — no sound, collapsed (use when notification is purely informational)
