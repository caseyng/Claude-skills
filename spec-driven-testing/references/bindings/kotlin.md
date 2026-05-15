# Kotlin binding — spec-driven-testing

## Shape extraction (Agent 1 additions)

Extract from Kotlin source:
- `fun` and `suspend fun` — note `suspend` in the shape; callers must know
- Nullability: `?` suffix on types is a contract, always preserve it
- `sealed class` / `sealed interface` variants are part of the type shape — list all variants and their fields
- `@Throws` annotation → include in Errors field
- `Flow<T>` return type means "observable stream that emits T over time" — state this in the shape
- `companion object` factory functions are part of the public API surface

Do not extract: `private`, `internal`, `protected` members unless they are the only path to a public behaviour.

## Test framework (Agent 2)

**Local JVM tests** (`src/test/`) — no Android runtime:
```kotlin
import org.junit.Test
import org.junit.Before
import junit.framework.TestCase.assertEquals
import junit.framework.TestCase.assertTrue
import junit.framework.TestCase.assertNull
import junit.framework.TestCase.assertNotNull

class MyTest {
    @Before fun setUp() { ... }
    @Test fun myTest() { ... }
}
```

**Instrumented tests** (`src/androidTest/`) — needs emulator or device:
```kotlin
import androidx.test.ext.junit.runners.AndroidJUnit4
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class MyInstrumentedTest { ... }
```

## In-memory dependencies

**Room database:**
```kotlin
import androidx.room.Room
import androidx.test.core.app.ApplicationProvider

val db = Room.inMemoryDatabaseBuilder(
    ApplicationProvider.getApplicationContext(),
    AppDatabase::class.java
).allowMainThreadQueries().build()
```
Always `db.close()` in `@After`.

**TypeConverters — no dependency needed:**
Instantiate directly: `val converters = Converters()`. Pure JVM, no Room or Android needed.

## Coroutines

```kotlin
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.flow.first

@Test
fun myTest() = runTest {
    val result = dao.observeAll().first()
    // ...
}
```

Use `runTest` for all `suspend` functions and Flow collection. Do not use `runBlocking` in tests.

## Assertions

Prefer JUnit4 assertion methods with message arguments for clarity:
```kotlin
assertEquals("message describing what should be equal", expected, actual)
assertTrue("message describing what should be true", condition)
```

For sealed class variants, use `is` checks before casting:
```kotlin
assertTrue("trigger should be ActivityTrigger", decoded is RuleTrigger.ActivityTrigger)
assertEquals(ActivityType.IN_VEHICLE, (decoded as RuleTrigger.ActivityTrigger).activityType)
```

## Format Assertion Requirement — Kotlin specifics

For kotlinx.serialization sealed classes, the encoded JSON contains a `type` discriminator field. Assert it:
```kotlin
val encoded = converters.myTypeToString(value)
assertTrue(encoded.contains("VariantName"))   // discriminator
assertTrue(encoded.contains("fieldValue"))    // data field
```

Do not assert the full JSON string literally — discriminator format can include the full package path. Assert substring presence for variant names and field values.

## Test structure pattern

```kotlin
@Test
fun operationName_scenario_expectedOutcome() = runTest {
    // Arrange — establish known state
    // Act — one operation
    // Assert — exactly the expected state change
}
```

One operation per test. Do not chain multiple operations unless testing their interaction is the explicit test purpose.
