---
skill: android-engineering
section: architecture
---

# Architecture

## Layer Model

```
UI (Composables)
  ↓ observes StateFlow / collects events
ViewModel
  ↓ calls
UseCase (optional — add only if ViewModel grows complex)
  ↓ calls
Repository (interface)
  ↓ implemented by
DataSource (Room, Go library, network, DataStore)
```

**Hard rules:**
- Nothing skips a layer. ViewModel never touches Room directly.
- Dependency direction is always downward. Repository knows nothing about ViewModel.
- Repository is an interface. ViewModel depends on the abstraction, not the implementation.
- Composition root (Application class or Hilt module) wires concrete implementations to interfaces.

---

## ViewModel

**What belongs here:** business logic, state transformation, event handling,
calls to Repository, coroutine launching.

**What does not belong here:** any `Context` reference (except `applicationContext`
via `AndroidViewModel` when strictly necessary), any reference to Activity, Fragment,
View, or NavController.

**State:**

```kotlin
// Single sealed state class per screen
sealed class ChatListUiState {
    object Loading : ChatListUiState()
    data class Success(val chats: List<Chat>) : ChatListUiState()
    data class Error(val message: String) : ChatListUiState()
}

class ChatListViewModel(private val chatRepo: ChatRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<ChatListUiState>(ChatListUiState.Loading)
    val uiState: StateFlow<ChatListUiState> = _uiState.asStateFlow()

    // One-time events that must not replay on rotation
    private val _events = MutableSharedFlow<ChatListEvent>()
    val events: SharedFlow<ChatListEvent> = _events.asSharedFlow()

    init { loadChats() }

    private fun loadChats() {
        viewModelScope.launch {
            _uiState.value = try {
                ChatListUiState.Success(chatRepo.getChats())
            } catch (e: Exception) {
                ChatListUiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}
```

**Rules:**
- Expose `StateFlow`, never `MutableStateFlow`
- Use `SharedFlow` for navigation, toasts, snackbars — events that must not replay
- All coroutines launched via `viewModelScope.launch` — auto-cancelled on ViewModel clear
- Never `runBlocking` in a ViewModel

---

## Composable / UI Layer

```kotlin
@Composable
fun ChatListScreen(
    viewModel: ChatListViewModel = hiltViewModel(),
    onNavigateToChat: (String) -> Unit
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // One-time event collection
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is ChatListEvent.NavigateToChat -> onNavigateToChat(event.jid)
            }
        }
    }

    when (uiState) {
        is ChatListUiState.Loading -> LoadingIndicator()
        is ChatListUiState.Success -> ChatList((uiState as ChatListUiState.Success).chats)
        is ChatListUiState.Error -> ErrorMessage((uiState as ChatListUiState.Error).message)
    }
}
```

**Rules:**
- `collectAsStateWithLifecycle()` — not `collectAsState()`. The lifecycle-aware version
  stops collecting when the app is backgrounded, preventing wasted work and subtle bugs.
- Composables are pure functions of state. No logic, no decisions, no mutations.
- Pass lambdas for callbacks — never pass ViewModel into nested Composables.
- Navigation decisions are made in ViewModel (emit event) or nav graph, never in Composable body.

---

## Repository

```kotlin
interface ChatRepository {
    suspend fun getChats(): List<Chat>
    suspend fun getChatHistory(jid: String, limit: Int): List<Message>
    fun observeChats(): Flow<List<Chat>>
}

class ChatRepositoryImpl(
    private val db: AppDatabase,
    private val goClient: WhatsAppClient  // gomobile Go library
) : ChatRepository {
    override suspend fun getChats(): List<Chat> = withContext(Dispatchers.IO) {
        db.chatDao().getAllChats().map { it.toDomain() }
    }
}
```

**Rules:**
- All I/O on `Dispatchers.IO` — never default dispatcher, never main thread
- Returns domain objects, not DB entities or raw types
- Repository is the only class that knows about data sources
- Interface + implementation separation enables testing with fakes

---

## Coroutine Scoping

| Scope | When to use |
|---|---|
| `viewModelScope` | ViewModel-initiated work — cancels when ViewModel clears |
| `lifecycleScope` | Activity/Fragment-initiated work — cancels on destroy |
| `rememberCoroutineScope()` | Composable-initiated work — cancels when Composable leaves composition |
| `ServiceScope` (custom) | Service work — cancel in `onDestroy`. Never `GlobalScope`. |

**Never use `GlobalScope`** — work leaks beyond any lifecycle, cannot be cancelled,
cannot be tested.

---

## Process Death

ViewModel survives configuration change (rotation) but **not** process death.

| What survives process death | Mechanism |
|---|---|
| Simple primitives (selected chat ID, scroll position) | `SavedStateHandle` in ViewModel |
| Screen content | Reconstructed from Repository on next launch |
| Nothing is kept across death by default | That is correct — reconstruct, don't persist UI state |

```kotlin
class ChatViewModel(
    private val savedState: SavedStateHandle,
    private val repo: ChatRepository
) : ViewModel() {
    private val chatJid: String = savedState["chat_jid"] ?: ""
}
```

---

## Dependency Injection

Manual DI is acceptable for personal apps. For larger apps, Hilt is the standard.

**Manual (simple):**
```kotlin
class MyApplication : Application() {
    val database by lazy { AppDatabase.create(this) }
    val chatRepository by lazy { ChatRepositoryImpl(database, WhatsAppClient()) }
}

// In Activity/Fragment
val repo = (application as MyApplication).chatRepository
val vm: ChatListViewModel by viewModels { ChatListViewModelFactory(repo) }
```

**Hilt (when manual becomes unwieldy):**
```kotlin
@HiltViewModel
class ChatListViewModel @Inject constructor(
    private val chatRepo: ChatRepository
) : ViewModel()
```

---

## Single Activity Architecture

One `MainActivity`, all screens are Composables navigated via Compose Navigation.

```kotlin
@Composable
fun AppNavGraph(navController: NavHostController) {
    NavHost(navController, startDestination = "chat_list") {
        composable("chat_list") {
            ChatListScreen(onNavigateToChat = { jid ->
                navController.navigate("chat/$jid")
            })
        }
        composable("chat/{jid}") { backStackEntry ->
            val jid = backStackEntry.arguments?.getString("jid") ?: return@composable
            ChatScreen(jid = jid)
        }
    }
}
```
