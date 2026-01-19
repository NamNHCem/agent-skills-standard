---
name: Jetpack Compose State Management
description: Advanced state management patterns for Jetpack Compose.
metadata:
  labels: [compose, state, viewmodel, stateflow, events]
  triggers:
    files: ['**/*ViewModel.kt', '**/*State.kt']
    keywords: [StateFlow, MutableStateFlow, collectAsStateWithLifecycle, rememberSaveable]
---

# Jetpack Compose State Management

## **Priority: P0 (CRITICAL)**

Effective state management patterns for predictable and performant Compose UIs.

## State Hoisting

### Stateless vs Stateful Composables
```kotlin
// ✅ Stateless (Reusable)
@Composable
fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    TextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier
    )
}

// ✅ Stateful (Convenience)
@Composable
fun SearchBarStateful(modifier: Modifier = Modifier) {
    var query by rememberSaveable { mutableStateOf("") }
    SearchBar(
        query = query,
        onQueryChange = { query = it },
        modifier = modifier
    )
}
```

### Hoist State to Appropriate Level
```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    var selectedTab by remember { mutableStateOf(0) } // Local UI state
    
    HomeContent(
        uiState = uiState,
        selectedTab = selectedTab,
        onTabSelected = { selectedTab = it },
        onAction = viewModel::handleAction
    )
}
```

## ViewModel State

### StateFlow Pattern
```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    // ✅ Good - Immutable public API
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    // ✅ Good - Combine multiple flows
    val combinedState: StateFlow<CombinedState> = combine(
        repository.observeUsers(),
        repository.observeSettings()
    ) { users, settings ->
        CombinedState(users, settings)
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = CombinedState.Empty
    )
}
```

### Collecting State in Compose
```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = hiltViewModel()) {
    // ✅ Good - Lifecycle-aware collection
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // ❌ Bad - Doesn't respect lifecycle
    val uiState by viewModel.uiState.collectAsState()
}
```

## Event Handling

### One-Time Events with SharedFlow
```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor() : ViewModel() {
    
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()
    
    fun onSaveClick() {
        viewModelScope.launch {
            try {
                repository.save()
                _events.emit(UiEvent.ShowSnackbar("Saved successfully"))
            } catch (e: Exception) {
                _events.emit(UiEvent.ShowError(e.message))
            }
        }
    }
}

sealed interface UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent
    data class ShowError(val message: String?) : UiEvent
    data class NavigateTo(val route: String) : UiEvent
}
```

### Consuming Events
```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }
    
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is UiEvent.ShowError -> {
                    // Handle error
                }
                is UiEvent.NavigateTo -> {
                    // Navigate
                }
            }
        }
    }
    
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) {
        // Content
    }
}
```

## State Preservation

### RememberSaveable
```kotlin
@Composable
fun SearchScreen() {
    // ✅ Survives process death
    var query by rememberSaveable { mutableStateOf("") }
    var selectedFilter by rememberSaveable { mutableStateOf(Filter.ALL) }
    
    SearchContent(query, selectedFilter)
}
```

### Custom Saver
```kotlin
data class SearchState(
    val query: String,
    val filters: List<Filter>
)

val SearchStateSaver = Saver<SearchState, Bundle>(
    save = { state ->
        Bundle().apply {
            putString("query", state.query)
            putParcelableArrayList("filters", ArrayList(state.filters))
        }
    },
    restore = { bundle ->
        SearchState(
            query = bundle.getString("query") ?: "",
            filters = bundle.getParcelableArrayList("filters") ?: emptyList()
        )
    }
)

@Composable
fun SearchScreen() {
    var searchState by rememberSaveable(stateSaver = SearchStateSaver) {
        mutableStateOf(SearchState("", emptyList()))
    }
}
```

## Derived State

### DerivedStateOf
```kotlin
@Composable
fun FilteredList(items: List<Item>, query: String) {
    // ✅ Good - Only recomputes when dependencies change
    val filteredItems by remember(items, query) {
        derivedStateOf {
            items.filter { it.name.contains(query, ignoreCase = true) }
        }
    }
    
    LazyColumn {
        items(filteredItems) { item ->
            ItemRow(item)
        }
    }
}

// ❌ Bad - Recomputes on every recomposition
@Composable
fun FilteredListBad(items: List<Item>, query: String) {
    val filteredItems = items.filter { it.name.contains(query, ignoreCase = true) }
    // ...
}
```

## State Holders

### Custom State Holder
```kotlin
@Stable
class SearchState(
    initialQuery: String = "",
    initialFilters: List<Filter> = emptyList()
) {
    var query by mutableStateOf(initialQuery)
        private set
    
    var filters by mutableStateOf(initialFilters)
        private set
    
    val hasActiveFilters: Boolean
        get() = filters.isNotEmpty()
    
    fun updateQuery(newQuery: String) {
        query = newQuery
    }
    
    fun toggleFilter(filter: Filter) {
        filters = if (filter in filters) {
            filters - filter
        } else {
            filters + filter
        }
    }
    
    fun clear() {
        query = ""
        filters = emptyList()
    }
}

@Composable
fun rememberSearchState(
    initialQuery: String = "",
    initialFilters: List<Filter> = emptyList()
): SearchState {
    return rememberSaveable(saver = SearchState.Saver) {
        SearchState(initialQuery, initialFilters)
    }
}
```

## Complex State Updates

### Update Patterns
```kotlin
@HiltViewModel
class TodoViewModel @Inject constructor() : ViewModel() {
    
    private val _uiState = MutableStateFlow(TodoUiState())
    val uiState: StateFlow<TodoUiState> = _uiState.asStateFlow()
    
    // ✅ Good - Immutable updates
    fun addTodo(title: String) {
        _uiState.update { currentState ->
            currentState.copy(
                todos = currentState.todos + Todo(title)
            )
        }
    }
    
    fun toggleTodo(id: String) {
        _uiState.update { currentState ->
            currentState.copy(
                todos = currentState.todos.map { todo ->
                    if (todo.id == id) todo.copy(completed = !todo.completed)
                    else todo
                }
            )
        }
    }
}

data class TodoUiState(
    val todos: List<Todo> = emptyList(),
    val filter: Filter = Filter.ALL,
    val isLoading: Boolean = false
)
```

## Testing State

### ViewModel Testing
```kotlin
@Test
fun `when user loads data, state updates to success`() = runTest {
    val viewModel = HomeViewModel(fakeRepository)
    
    // Collect state
    val states = mutableListOf<UiState>()
    val job = launch {
        viewModel.uiState.collect { states.add(it) }
    }
    
    // Trigger action
    viewModel.loadData()
    advanceUntilIdle()
    
    // Assert
    assertEquals(UiState.Loading, states[0])
    assertEquals(UiState.Success(data), states[1])
    
    job.cancel()
}
```

### Turbine for Flow Testing
```kotlin
@Test
fun `events are emitted correctly`() = runTest {
    val viewModel = HomeViewModel()
    
    viewModel.events.test {
        viewModel.onSaveClick()
        
        val event = awaitItem()
        assertTrue(event is UiEvent.ShowSnackbar)
        assertEquals("Saved successfully", (event as UiEvent.ShowSnackbar).message)
    }
}
```

## Anti-Patterns

- **No Mutable State in Composables**: Hoist to ViewModel
- **No collectAsState() for Lifecycle**: Use collectAsStateWithLifecycle()
- **No State in remember { }**: Use mutableStateOf
- **No Ignoring Configuration Changes**: Use rememberSaveable
- **No Direct State Mutation**: Use update { } or emit new values

## Reference & Examples

For advanced state patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

jetpack-compose/architecture | jetpack-compose/ui-components | kotlin/coroutines
