---
skill: android-engineering
section: testing
---

# Testing

## Test Pyramid

```
         [ UI Tests ]           ← fewest, slowest, most fragile
      [ Integration Tests ]     ← Room + Repository with in-memory DB
   [ ViewModel / Unit Tests ]   ← most, fastest, no Android dependency
```

Write most tests at the ViewModel and Repository level.
UI tests verify wiring, not logic.

---

## ViewModel Unit Tests

ViewModels have no Android dependency when Repository is injected as an interface.
Tests are plain JUnit — no emulator needed.

```kotlin
class ChatListViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()  // replaces Dispatchers.Main with test dispatcher

    private val fakeRepo = FakeChatRepository()
    private lateinit var viewModel: ChatListViewModel

    @Before
    fun setup() {
        viewModel = ChatListViewModel(fakeRepo)
    }

    @Test
    fun `initial state is loading`() {
        // ViewModel not yet loaded
        assertThat(viewModel.uiState.value).isInstanceOf(ChatListUiState.Loading::class.java)
    }

    @Test
    fun `loads chats on init`() = runTest {
        fakeRepo.chats = listOf(Chat(jid = "123@s.whatsapp.net", name = "Alice"))

        viewModel = ChatListViewModel(fakeRepo)
        advanceUntilIdle()

        val state = viewModel.uiState.value as ChatListUiState.Success
        assertThat(state.chats).hasSize(1)
        assertThat(state.chats[0].name).isEqualTo("Alice")
    }

    @Test
    fun `emits error state when repo throws`() = runTest {
        fakeRepo.shouldThrow = true

        viewModel = ChatListViewModel(fakeRepo)
        advanceUntilIdle()

        assertThat(viewModel.uiState.value).isInstanceOf(ChatListUiState.Error::class.java)
    }
}
```

**Rules:**
- Test ViewModel via its public `StateFlow` and `SharedFlow` — not internal methods
- Use `runTest` for coroutine tests — wraps TestCoroutineScope, auto-advances time
- Use `advanceUntilIdle()` to let coroutines complete in tests
- Use `MainDispatcherRule` to replace `Dispatchers.Main` (required for ViewModel tests)

---

## Fake Repository (not Mock)

Use fakes over mocks. Fakes are in-memory implementations of the interface.
Mocks couple tests to implementation details — change the ViewModel internals
and mocks break even if behavior is unchanged.

```kotlin
class FakeChatRepository : ChatRepository {
    var chats: List<Chat> = emptyList()
    var shouldThrow = false

    override suspend fun getChats(): List<Chat> {
        if (shouldThrow) throw IOException("Network error")
        return chats
    }

    override fun observeChats(): Flow<List<Chat>> = flowOf(chats)
}
```

**Rule:** Fakes live in `src/test/` only — never ship to production.

---

## MainDispatcherRule

Required for any test that involves `viewModelScope` or `Dispatchers.Main`:

```kotlin
class MainDispatcherRule(
    private val dispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
        dispatcher.cleanupTestCoroutines()
    }
}
```

---

## Repository / Room Integration Tests

Test Repository with an **in-memory Room database** — not a mock DAO.
In-memory Room is fast and catches real SQL issues that mocked DAOs miss.

```kotlin
@RunWith(AndroidJUnit4::class)  // requires instrumented test (androidTest/)
class ChatRepositoryTest {

    private lateinit var db: AppDatabase
    private lateinit var repo: ChatRepositoryImpl

    @Before
    fun setup() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()  // allowMainThread OK in tests only
        repo = ChatRepositoryImpl(db)
    }

    @After
    fun teardown() {
        db.close()
    }

    @Test
    fun insertAndRetrieveScheduledMessage() = runTest {
        val msg = ScheduledMessage(toJid = "123@s.whatsapp.net", body = "hello", sendAt = 1000L)
        repo.scheduleMessage(msg)

        val scheduled = repo.getPendingMessages()
        assertThat(scheduled).hasSize(1)
        assertThat(scheduled[0].body).isEqualTo("hello")
    }
}
```

**Note:** In-memory Room tests are instrumented tests (`src/androidTest/`).
They run on an emulator or device. Keep them focused on data layer correctness.

---

## Compose UI Tests

Test that Composables render and respond correctly. Test wiring, not business logic.
Business logic is already tested in ViewModel tests.

```kotlin
@RunWith(AndroidJUnit4::class)
class ChatListScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun showsLoadingIndicator_whenStateIsLoading() {
        composeTestRule.setContent {
            ChatListScreen(
                uiState = ChatListUiState.Loading,
                onNavigateToChat = {}
            )
        }

        composeTestRule.onNodeWithTag("LoadingIndicator").assertIsDisplayed()
    }

    @Test
    fun showsChatItems_whenStateIsSuccess() {
        val chats = listOf(Chat(jid = "123@s.whatsapp.net", name = "Alice"))

        composeTestRule.setContent {
            ChatListScreen(
                uiState = ChatListUiState.Success(chats),
                onNavigateToChat = {}
            )
        }

        composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
    }

    @Test
    fun tappingChat_invokesCallback() {
        var tappedJid: String? = null
        val chats = listOf(Chat(jid = "123@s.whatsapp.net", name = "Alice"))

        composeTestRule.setContent {
            ChatListScreen(
                uiState = ChatListUiState.Success(chats),
                onNavigateToChat = { tappedJid = it }
            )
        }

        composeTestRule.onNodeWithText("Alice").performClick()
        assertThat(tappedJid).isEqualTo("123@s.whatsapp.net")
    }
}
```

**Rules:**
- Pass UiState directly to Composable in tests — do not wire a real ViewModel
- Use `testTag` on ambiguous elements, semantic labels on interactive elements
- Test: loading state renders, empty state renders, success state renders, tap callbacks fire
- Do not test that the ViewModel is called — that is an implementation detail

---

## The Mirror Test (Android)

Before delivering any test, ask: if the ViewModel returned garbage data, would this
test catch it?

```kotlin
// Mirror test — derives expected from the same data passed in
@Test
fun chatCount() {
    val chats = listOf(Chat("a"), Chat("b"))
    viewModel.load(chats)
    assertThat(viewModel.uiState.value.chats.size).isEqualTo(chats.size)  // always passes
}

// Correct — uses known expected value
@Test
fun twoChatsAreDisplayed() {
    fakeRepo.chats = listOf(Chat("a"), Chat("b"))
    viewModel = ChatListViewModel(fakeRepo)
    advanceUntilIdle()
    assertThat(viewModel.uiState.value.chats).hasSize(2)
}
```

---

## What to Test

| Component | Test type | What to verify |
|---|---|---|
| ViewModel | Unit (JUnit + runTest) | State transitions for all UiState values, event emissions, error paths |
| Repository | Integration (in-memory Room) | Insert/read/delete correctness, query filters, edge cases (empty result) |
| Domain / UseCase | Unit | Business rule correctness, boundary conditions |
| Composable | Compose UI test | Renders for each UiState, tap callbacks fire, accessibility labels present |
| Service | Manual / integration | Cannot easily unit-test — verify behavior on device |

**Do not test:**
- ViewModel coroutine internals (test the StateFlow output, not the launch body)
- Room DAO SQL directly (test via Repository)
- Android framework classes (`Context`, `NotificationManager`) — mock at the boundary
