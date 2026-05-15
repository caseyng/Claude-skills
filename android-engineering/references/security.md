---
skill: android-engineering
section: security
---

# Security

## Logcat Exposure

**On debug builds, Logcat is world-readable by any app on the device.**
On release builds, `Log.d` and `Log.v` calls are typically stripped by ProGuard/R8,
but only if the build is configured correctly.

**Rules:**
- Never log tokens, session IDs, WA credentials, pairing codes, or message content
- Never log user JIDs or phone numbers
- Use `Log.d` only for developer diagnostics during development — never leave
  sensitive data in any log level
- Verify ProGuard strips log calls on release: `LogRemoving` rules must be in `proguard-rules.pro`

```proguard
# Strip debug/verbose logging on release
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int d(...);
}
```

---

## Secure Storage

**Never use `SharedPreferences` for secrets** — stored as plaintext XML on disk,
readable if device is rooted or via ADB on debug builds.

**For secrets (tokens, keys):**
```kotlin
// EncryptedSharedPreferences — keys and values both encrypted
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val prefs = EncryptedSharedPreferences.create(
    context,
    "secret_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

**For non-secret app config/preferences:** Use `DataStore<Preferences>` (not SharedPreferences).
DataStore is async, transactional, and handles concurrent access correctly.
SharedPreferences has known thread-safety issues and is deprecated for new code.

---

## Android Keystore

Use for cryptographic keys that must never leave the device.

```kotlin
// Generate a key in hardware-backed Keystore
val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
keyGenerator.init(
    KeyGenParameterSpec.Builder("whatsbot_key", KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
        .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
        .setKeySize(256)
        .setUserAuthenticationRequired(false)  // true = requires biometric on each use
        .build()
)
keyGenerator.generateKey()
```

**Rules:**
- Keys generated in Keystore cannot be exported — they live in hardware security module
- `setUserAuthenticationRequired(true)` binds key use to biometric verification
- Never store the raw key material anywhere — the key alias is the only reference needed

---

## Biometric Authentication

```kotlin
val biometricPrompt = BiometricPrompt(
    activity,
    ContextCompat.getMainExecutor(activity),
    object : BiometricPrompt.AuthenticationCallback() {
        override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
            // Use result.cryptoObject to decrypt data
        }
        override fun onAuthenticationFailed() { /* Wrong biometric — retry */ }
        override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
            // Fallback to PIN or abort
        }
    }
)

val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("Confirm identity")
    .setSubtitle("Access requires verification")
    .setNegativeButtonText("Cancel")
    .build()

biometricPrompt.authenticate(promptInfo)
```

**Rule:** Biometric should unlock a Keystore key that then decrypts the data.
Biometric alone (without backing encryption) is not sufficient — it can be bypassed
on rooted devices.

---

## Network Security

**TLS only — enforce via `network_security_config.xml`:**

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config">
```

`cleartextTrafficPermitted="false"` blocks all HTTP. Any HTTP request throws
`IOException`. Do not add exceptions without explicit justification.

---

## ProGuard / R8 (Release Builds)

Enable R8 on release builds in `build.gradle.kts`:

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
    }
}
```

**Keep rules for gomobile `.aar`:**
```proguard
# Keep gomobile-exported classes — obfuscating these breaks JNI calls
-keep class go.** { *; }
-keep class whatsbot.** { *; }
```

**Always test the release build** before shipping — obfuscation can break reflection-based
code that works fine in debug. Run with `./gradlew assembleRelease` and install on device.

---

## Intent Security

**Exported components:** Set `android:exported="false"` on all components that should
not be reachable from other apps. Android 12+ requires explicit exported declaration.

```xml
<!-- Correct — explicit -->
<activity android:name=".MainActivity" android:exported="true" />
<service android:name=".WhatsAppService" android:exported="false" />
<receiver android:name=".BootReceiver" android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

**Pending Intents:** Always use `FLAG_IMMUTABLE` (Android 12+):
```kotlin
val pendingIntent = PendingIntent.getBroadcast(
    context, requestCode, intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
```

`FLAG_MUTABLE` pending intents can be hijacked by other apps to inject data.
Only use `FLAG_MUTABLE` when explicitly required (e.g., inline reply notifications).
