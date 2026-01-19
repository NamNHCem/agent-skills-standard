---
name: Kotlin Coroutines
description: Advanced coroutines patterns for asynchronous programming.
metadata:
  labels: [kotlin, coroutines, async, flow, channels]
  triggers:
    files: ['**/*.kt']
    keywords: [suspend, launch, async, Flow, Channel, withContext, coroutineScope]
---

# Kotlin Coroutines

## **Priority: P0 (CRITICAL)**

Structured concurrency and asynchronous programming with Kotlin Coroutines.

## Coroutine Scopes

### Structured Concurrency
```kotlin
// ✅ Good - Use proper scope
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            val data = repository.fetchData()
            _uiState.value = UiState.Success(data)
        }
    }
}

// ❌ Bad - GlobalScope leaks
GlobalScope.launch { ... } // Never use in production
```

### Common Scopes
- **`viewModelScope`**: Auto-cancels when ViewModel cleared
- **`lifecycleScope`**: Tied to Activity/Fragment lifecycle
- **`rememberCoroutineScope()`**: For Compose UI events
- **`CoroutineScope(Job() + Dispatchers.Main)`**: Custom scopes

## Dispatchers

### Choosing the Right Dispatcher
- **`Dispatchers.Main`**: UI updates, light operations
- **`Dispatchers.IO`**: Network, database, file I/O (64 threads)
- **`Dispatchers.Default`**: CPU-intensive work (CPU cores)
- **`Dispatchers.Unconfined`**: Advanced use only (avoid)

```kotlin
suspend fun fetchUser(): User = withContext(Dispatchers.IO) {
    database.getUserDao().getUser()
}
```

## Launch vs Async

### Launch (Fire-and-Forget)
```kotlin
// ✅ Use for side effects
viewModelScope.launch {
    analytics.logEvent("screen_view")
}
```

### Async (Concurrent Results)
```kotlin
// ✅ Use for parallel operations
suspend fun loadDashboard() = coroutineScope {
    val userDeferred = async { fetchUser() }
    val postsDeferred = async { fetchPosts() }
    
    Dashboard(
        user = userDeferred.await(),
        posts = postsDeferred.await()
    )
}
```

## Flow

### Cold Streams
```kotlin
// ✅ Good - Repository returns Flow
class UserRepository {
    fun observeUser(id: String): Flow<User> = flow {
        val user = api.getUser(id)
        emit(user)
    }.flowOn(Dispatchers.IO)
}

// ViewModel collects
viewModelScope.launch {
    repository.observeUser(userId)
        .catch { e -> handleError(e) }
        .collect { user -> _uiState.value = user }
}
```

### Flow Operators
- **`map`**: Transform emissions
- **`filter`**: Conditional emissions
- **`flatMapConcat`**: Sequential flattening
- **`flatMapMerge`**: Concurrent flattening
- **`combine`**: Combine multiple flows
- **`zip`**: Pair emissions
- **`debounce`**: Delay emissions
- **`distinctUntilChanged`**: Skip duplicates

### StateFlow & SharedFlow
```kotlin
// ✅ StateFlow - Single state holder
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
}

// ✅ SharedFlow - Events (no initial value)
private val _events = MutableSharedFlow<Event>()
val events: SharedFlow<Event> = _events.asSharedFlow()
```

## Exception Handling

### Try-Catch in Suspend Functions
```kotlin
suspend fun fetchData(): Result<Data> = try {
    val data = api.getData()
    Result.Success(data)
} catch (e: IOException) {
    Result.Error("Network error: ${e.message}")
}
```

### CoroutineExceptionHandler
```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e(TAG, "Coroutine exception", exception)
}

viewModelScope.launch(handler) {
    riskyOperation()
}
```

### SupervisorJob
```kotlin
// ✅ Good - Child failure doesn't cancel siblings
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

scope.launch {
    // If this fails, other children continue
    riskyOperation1()
}
scope.launch {
    riskyOperation2()
}
```

## Cancellation

### Cooperative Cancellation
```kotlin
suspend fun processItems(items: List<Item>) {
    for (item in items) {
        ensureActive() // Check cancellation
        processItem(item)
    }
}

// Or use yield()
suspend fun longRunningTask() {
    repeat(1000) {
        yield() // Cooperative cancellation point
        doWork()
    }
}
```

### Cleanup with Finally
```kotlin
try {
    suspendCancellableCoroutine<Unit> { continuation ->
        // Setup
    }
} finally {
    // Cleanup (always executes)
    releaseResources()
}
```

## Testing

### runTest for Coroutines
```kotlin
@Test
fun testDataLoading() = runTest {
    val viewModel = MyViewModel(repository)
    viewModel.loadData()
    
    advanceUntilIdle() // Fast-forward virtual time
    assertEquals(UiState.Success, viewModel.uiState.value)
}
```

### Turbine for Flow Testing
```kotlin
@Test
fun testUserFlow() = runTest {
    repository.observeUser("123").test {
        assertEquals(User("John"), awaitItem())
        awaitComplete()
    }
}
```

## Anti-Patterns

- **No `runBlocking` in Production**: Blocks threads (tests only)
- **No GlobalScope**: Always use structured scopes
- **No Ignoring Cancellation**: Respect `isActive` checks
- **No Blocking Calls on Main**: Use `withContext(Dispatchers.IO)`
- **No Uncaught Exceptions**: Always handle with try-catch or handler

## Reference & Examples

For detailed coroutine patterns and Flow examples:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/language | kotlin/flow-advanced | jetpack-compose/state-management | jetpack-compose/architecture

> **💡 Next Step**: For production-ready Flow patterns (lazy observation, stateIn/shareIn, lifecycle-aware collection), see [kotlin/flow-advanced](../flow-advanced/SKILL.md).
