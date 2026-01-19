---
name: Kotlin Flow Advanced Patterns
description: Advanced Flow patterns for production-ready Android apps with lazy observation, state sharing, and lifecycle-aware collection.
metadata:
  labels: [kotlin, flow, stateflow, sharedflow, statein, sharein, lifecycle]
  triggers:
    files: ['**/*ViewModel.kt', '**/*Repository.kt', '**/*UseCase.kt']
    keywords: [Flow, StateFlow, SharedFlow, stateIn, shareIn, SharingStarted, WhileSubscribed, collectAsStateWithLifecycle, repeatOnLifecycle]
---

# Kotlin Flow Advanced Patterns

## **Priority: P0 (CRITICAL)**

Production-ready Flow patterns focusing on lazy observation, state sharing, and lifecycle-aware collection to prevent bugs, save resources, and improve code predictability.

> **📚 Prerequisite**: This skill assumes familiarity with coroutines fundamentals (scopes, dispatchers, launch/async). See [kotlin/coroutines](../coroutines/SKILL.md) for basics.

## Core Principles

### 4 Truths About Coroutines

1. **Coroutine ≠ Thread**: Lightweight tasks that suspend/resume on a thread pool
2. **Async Like Sequential Code**: Escape callback hell with suspend functions
3. **Old Idea, Modern Tools**: Cooperative multitasking with suspend keyword
4. **Structured Concurrency**: Scopes have lifecycle, parent cancel → children cancel

### Golden Rules

- **UI Layer**: Use `repeatOnLifecycle` / `collectAsStateWithLifecycle`
- **ViewModel Layer**: Use `stateIn`/`shareIn` + appropriate `SharingStarted`
- **Data Layer**: Explicit cache/invalidate strategy

## Cold vs Hot Flows

### Cold Flow (Default)
```kotlin
// ❌ Problem: Re-executes for each collector
fun getUser(): Flow<User> = flow {
    val user = api.getUser() // Called EVERY collect
    emit(user)
}

// If 2 collectors → 2 API calls!
viewModel.getUser().collect { } // API call 1
viewModel.getUser().collect { } // API call 2
```

### Hot Flow (Shared)
```kotlin
// ✅ Solution: Share upstream with stateIn
val user: StateFlow<User> = flow {
    val user = api.getUser() // Called ONCE
    emit(user)
}.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),
    initialValue = User.Empty
)

// Multiple collectors → 1 API call
user.collect { } // Shares upstream
user.collect { } // Shares upstream
```

## StateIn vs ShareIn

### StateIn (For State)
```kotlin
// ✅ Use for UI state (always has current value)
val uiState: StateFlow<UiState> = repository
    .observeData()
    .map { data -> UiState.Success(data) }
    .catch { emit(UiState.Error(it.message)) }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = UiState.Loading
    )
```

**Key Features:**
- Always has a value (replay = 1)
- New collectors get current state immediately
- Conflates updates (only latest value matters)

### ShareIn (For Events)
```kotlin
// ✅ Use for one-time events (no initial value)
val events: SharedFlow<UiEvent> = eventChannel
    .receiveAsFlow()
    .shareIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(),
        replay = 0 // No replay for events
    )
```

**Key Features:**
- No initial value
- Can configure replay buffer
- All active collectors receive emissions

## SharingStarted Strategies

### WhileSubscribed (Recommended)
```kotlin
// ✅ Best for most cases: stops when no subscribers
SharingStarted.WhileSubscribed(
    stopTimeoutMillis = 5000, // Keep alive 5s after last unsubscribe
    replayExpirationMillis = 0 // Don't replay stale data
)
```

**Use Cases:**
- Screen-specific data
- Resource-intensive operations
- Data that should refresh on return

### Eagerly
```kotlin
// ⚠️ Use sparingly: starts immediately, never stops
SharingStarted.Eagerly
```

**Use Cases:**
- App-level state (user session, settings)
- Data needed before UI subscribes
- Expensive one-time initialization

### Lazily
```kotlin
// Starts on first subscriber, never stops
SharingStarted.Lazily
```

**Use Cases:**
- Cached data that should persist
- Singleton-like behavior

## Lazy Observation Pattern

### ❌ Anti-Pattern: Init Side-Effect
```kotlin
// ❌ BAD: Runs immediately when VM created
class MyViewModel @Inject constructor(
    private val useCase: GetVideosUseCase
) : ViewModel() {
    
    private val _videos = MutableStateFlow<List<Video>>(emptyList())
    val videos: StateFlow<List<Video>> = _videos.asStateFlow()
    
    init {
        loadVideos() // ❌ Runs even if UI never subscribes!
    }
    
    private fun loadVideos() {
        viewModelScope.launch {
            useCase().collect { list ->
                _videos.value = list // ❌ Manual plumbing
            }
        }
    }
}
```

**Problems:**
- Runs even if UI never subscribes (waste)
- Hard to test
- Manual state management
- Can be called multiple times → race conditions

### ✅ Correct: Lazy with stateIn
```kotlin
// ✅ GOOD: Only starts when UI subscribes
@HiltViewModel
class MyViewModel @Inject constructor(
    private val useCase: GetVideosUseCase
) : ViewModel() {
    
    val videos: StateFlow<List<Video>> = useCase()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}
```

**Benefits:**
- Lazy: only starts when UI subscribes
- Auto-cancels when UI stops
- Shares upstream (no duplicate work)
- Replays last value to new subscribers

## Lifecycle-Aware Collection

### Fragment: repeatOnLifecycle
```kotlin
// ✅ CORRECT: Cancels when STOPPED
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            // Separate launch for each flow
            launch {
                viewModel.uiState.collect { state ->
                    updateUI(state)
                }
            }
            
            launch {
                viewModel.events.collect { event ->
                    handleEvent(event)
                }
            }
        }
    }
}

// ❌ WRONG: Keeps collecting when STOPPED
viewLifecycleOwner.lifecycleScope.launch {
    viewModel.uiState.collect { state -> // ❌ Wastes resources
        updateUI(state)
    }
}
```

### Compose: collectAsStateWithLifecycle
```kotlin
// ✅ CORRECT: Lifecycle-aware
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    MyContent(uiState)
}

// ❌ WRONG: Not lifecycle-aware
@Composable
fun MyScreenBad(viewModel: MyViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState() // ❌ Keeps collecting
    
    MyContent(uiState)
}
```

## State vs Events Pattern

### State (StateFlow)
```kotlin
// ✅ For persistent UI state
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

fun loadData() {
    viewModelScope.launch {
        _uiState.value = UiState.Loading
        try {
            val data = repository.getData()
            _uiState.value = UiState.Success(data)
        } catch (e: Exception) {
            _uiState.value = UiState.Error(e.message)
        }
    }
}
```

### Events (SharedFlow)
```kotlin
// ✅ For one-time events (navigation, toasts, snackbars)
private val _events = MutableSharedFlow<UiEvent>(
    extraBufferCapacity = 1 // Don't suspend on emit
)
val events: SharedFlow<UiEvent> = _events.asSharedFlow()

fun deleteItem() {
    viewModelScope.launch {
        _events.tryEmit(UiEvent.ShowLoading)
        when (val result = repository.delete()) {
            is Success -> _events.tryEmit(UiEvent.ShowSuccess("Deleted"))
            is Error -> _events.tryEmit(UiEvent.ShowError(result.message))
        }
    }
}

sealed interface UiEvent {
    data object ShowLoading : UiEvent
    data class ShowSuccess(val message: String) : UiEvent
    data class ShowError(val message: String) : UiEvent
}
```

## Flow Operators Cookbook

### Search with Debounce
```kotlin
private val rawQuery = MutableStateFlow("")

val searchResults: StateFlow<SearchState> = rawQuery
    .debounce(300) // Wait 300ms after typing stops
    .distinctUntilChanged() // Skip duplicate queries
    .flatMapLatest { query -> // Cancel previous search
        flow {
            emit(SearchState.Loading)
            val results = repository.search(query)
            emit(SearchState.Success(results))
        }.catch { emit(SearchState.Error(it.message)) }
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = SearchState.Idle
    )

fun onQueryChange(query: String) {
    rawQuery.value = query
}
```

### Combine Multiple Sources
```kotlin
val uiState: StateFlow<HomeUiState> = combine(
    repository.observeContent(),
    repository.observeWeather(),
    userPreferences.observeSettings()
) { content, weather, settings ->
    HomeUiState.Success(
        content = content,
        weather = weather,
        settings = settings
    )
}.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),
    initialValue = HomeUiState.Loading
)
```

### Refresh/Retry Pattern
```kotlin
private val refreshTrigger = MutableSharedFlow<Unit>(replay = 1)

val data: StateFlow<DataState> = refreshTrigger
    .onStart { emit(Unit) } // Emit initial value
    .flatMapLatest {
        flow {
            emit(DataState.Loading)
            val result = repository.fetchData()
            emit(DataState.Success(result))
        }.catch { emit(DataState.Error(it.message)) }
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = DataState.Loading
    )

fun refresh() {
    refreshTrigger.tryEmit(Unit)
}
```

## CancellationException Handling

### ❌ Wrong: Catching CancellationException
```kotlin
// ❌ BAD: Swallows cancellation signal
viewModelScope.launch {
    try {
        val data = repository.getData()
        _state.value = Success(data)
    } catch (e: Exception) { // ❌ Catches CancellationException!
        _state.value = Error(e.message)
    }
}
```

### ✅ Correct: Rethrow CancellationException
```kotlin
// ✅ GOOD: Preserves cancellation
viewModelScope.launch {
    try {
        val data = repository.getData()
        _state.value = Success(data)
    } catch (e: CancellationException) {
        throw e // ✅ Rethrow cancellation
    } catch (e: Exception) {
        _state.value = Error(e.message)
    }
}

// Or use runCatching with proper handling
val result = runCatching {
    repository.getData()
}.onFailure { e ->
    if (e is CancellationException) throw e // ✅ Rethrow
    _state.value = Error(e.message)
}
```

## Navigation + WhileSubscribed Gotcha

### Problem
```kotlin
// Navigate Main → Detail: Main goes STOP
// If back after > 5s: WhileSubscribed stops upstream
// If upstream = cold flow → re-fetches on return!
```

### Solutions

**Option 1: Cache at Data Layer**
```kotlin
// ✅ Repository caches data
class UserRepository {
    private val cache = MutableStateFlow<User?>(null)
    
    fun observeUser(): Flow<User> = flow {
        cache.value?.let { emit(it) }
        val fresh = api.getUser()
        cache.value = fresh
        emit(fresh)
    }
}
```

**Option 2: Increase Timeout**
```kotlin
// ✅ Keep alive longer for quick navigation
SharingStarted.WhileSubscribed(
    stopTimeoutMillis = 30_000 // 30 seconds
)
```

**Option 3: Custom SharingStarted**
```kotlin
// ✅ One-time fetch, replay forever
fun <T> Flow<T>.stateInWhileSubscribedOnce(
    scope: CoroutineScope,
    initialValue: T
): StateFlow<T> {
    var started = false
    return stateIn(
        scope = scope,
        started = SharingStarted.WhileSubscribed {
            if (!started) {
                started = true
                SharingCommand.START
            } else {
                SharingCommand.STOP_AND_RESET_REPLAY_CACHE
            }
        },
        initialValue = initialValue
    )
}
```

## Anti-Patterns Summary

- **No init side-effects**: Use lazy observation with `stateIn`
- **No manual collect in init**: Let UI subscription drive data loading
- **No collectAsState in Compose**: Use `collectAsStateWithLifecycle`
- **No collect without repeatOnLifecycle in Fragment**: Wastes resources
- **No catching CancellationException**: Breaks structured concurrency
- **No Flow for one-shot operations**: Use suspend functions
- **No multiple collectors on cold Flow**: Share with `stateIn`/`shareIn`

## Quick Decision Cheat Sheet

1. **State UI** → `StateFlow` with `stateIn`
2. **Events** → `SharedFlow` or `Channel`
3. **Fragment collection** → `repeatOnLifecycle(STARTED)`
4. **Compose collection** → `collectAsStateWithLifecycle()`
5. **Initial data** → Lazy observation (avoid init/LaunchedEffect)
6. **Search** → `debounce` + `distinctUntilChanged` + `flatMapLatest`
7. **Combine sources** → `combine`
8. **Refresh** → Trigger flow with `MutableSharedFlow` + `flatMapLatest`
9. **Threading** → `flowOn(dispatcher)` for upstream work
10. **Sharing** → `WhileSubscribed(5000)` for most cases

## Reference & Examples

For complete examples and migration guides:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/coroutines | jetpack-compose/state-management | jetpack-compose/architecture
