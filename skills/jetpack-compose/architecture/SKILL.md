---
name: Jetpack Compose Architecture
description: Clean Architecture patterns for Jetpack Compose applications.
metadata:
  labels: [compose, architecture, mvvm, mvi, clean-architecture]
  triggers:
    files: ['**/*ViewModel.kt', '**/*Screen.kt', '**/*UseCase.kt']
    keywords: [ViewModel, Screen, UseCase, Repository, UiState]
---

# Jetpack Compose Architecture

## **Priority: P0 (CRITICAL)**

Clean Architecture patterns for scalable Jetpack Compose applications.

## Architecture Layers

### Presentation Layer (UI)
```text
presentation/
├── screens/
│   ├── home/
│   │   ├── HomeScreen.kt
│   │   ├── HomeViewModel.kt
│   │   └── HomeUiState.kt
│   └── details/
│       ├── DetailsScreen.kt
│       └── DetailsViewModel.kt
└── components/
    ├── AppButton.kt
    └── LoadingIndicator.kt
```

### Domain Layer (Business Logic)
```text
domain/
├── model/
│   └── User.kt
├── repository/
│   └── UserRepository.kt (interface)
└── usecase/
    ├── GetUserUseCase.kt
    └── UpdateUserUseCase.kt
```

### Data Layer
```text
data/
├── repository/
│   └── UserRepositoryImpl.kt
├── remote/
│   └── UserApi.kt
└── local/
    └── UserDao.kt
```

## MVVM Pattern

### ViewModel
```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<HomeUiState>(HomeUiState.Loading)
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    init {
        loadUser()
    }
    
    private fun loadUser() {
        viewModelScope.launch {
            getUserUseCase()
                .catch { e -> _uiState.value = HomeUiState.Error(e.message) }
                .collect { user -> _uiState.value = HomeUiState.Success(user) }
        }
    }
    
    fun onRefresh() {
        _uiState.value = HomeUiState.Loading
        loadUser()
    }
}
```

### UiState (Sealed Class)
```kotlin
sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Success(val user: User) : HomeUiState
    data class Error(val message: String?) : HomeUiState
}
```

### Screen (Composable)
```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel(),
    onNavigateToDetails: (String) -> Unit
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    HomeContent(
        uiState = uiState,
        onRefresh = viewModel::onRefresh,
        onUserClick = onNavigateToDetails
    )
}

@Composable
private fun HomeContent(
    uiState: HomeUiState,
    onRefresh: () -> Unit,
    onUserClick: (String) -> Unit
) {
    when (uiState) {
        is HomeUiState.Loading -> LoadingIndicator()
        is HomeUiState.Success -> UserList(uiState.user, onUserClick)
        is HomeUiState.Error -> ErrorView(uiState.message, onRefresh)
    }
}
```

## MVI Pattern (Alternative)

### Intent-based ViewModel
```kotlin
sealed interface HomeIntent {
    data object LoadUser : HomeIntent
    data object Refresh : HomeIntent
    data class SelectUser(val id: String) : HomeIntent
}

@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<HomeUiState>(HomeUiState.Loading)
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    fun handleIntent(intent: HomeIntent) {
        when (intent) {
            is HomeIntent.LoadUser -> loadUser()
            is HomeIntent.Refresh -> refresh()
            is HomeIntent.SelectUser -> selectUser(intent.id)
        }
    }
}
```

## Use Cases

### Single Responsibility
```kotlin
class GetUserUseCase @Inject constructor(
    private val repository: UserRepository
) {
    operator fun invoke(userId: String): Flow<User> {
        return repository.getUser(userId)
    }
}

class UpdateUserUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(user: User): Result<Unit> {
        return try {
            repository.updateUser(user)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

## Repository Pattern

### Interface (Domain Layer)
```kotlin
interface UserRepository {
    fun getUser(id: String): Flow<User>
    suspend fun updateUser(user: User)
    fun observeUsers(): Flow<List<User>>
}
```

### Implementation (Data Layer)
```kotlin
class UserRepositoryImpl @Inject constructor(
    private val remoteDataSource: UserApi,
    private val localDataSource: UserDao,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) : UserRepository {
    
    override fun getUser(id: String): Flow<User> = flow {
        // Try local first
        val localUser = localDataSource.getUser(id)
        if (localUser != null) {
            emit(localUser.toDomain())
        }
        
        // Fetch from remote
        val remoteUser = remoteDataSource.getUser(id)
        localDataSource.insertUser(remoteUser.toEntity())
        emit(remoteUser.toDomain())
    }.flowOn(dispatcher)
}
```

## Dependency Injection

### Hilt (Recommended for Android)
```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
    private val savedStateHandle: SavedStateHandle
) : ViewModel()
```

> **📚 For complete Hilt setup and patterns**: See [kotlin/hilt-di](../../kotlin/hilt-di/SKILL.md) for comprehensive DI guide.

### Koin (Alternative for KMP)
```kotlin
class HomeViewModel(
    private val getUserUseCase: GetUserUseCase
) : ViewModel()

// Module
val viewModelModule = module {
    viewModel { HomeViewModel(get()) }
}
```

> **📚 For complete Koin setup**: See [kotlin/koin-di](../../kotlin/koin-di/SKILL.md).

## State Management

### StateFlow for State
```kotlin
// ✅ Good - Single source of truth
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()
```

### SharedFlow for Events
```kotlin
// ✅ Good - One-time events
private val _events = MutableSharedFlow<UiEvent>()
val events: SharedFlow<UiEvent> = _events.asSharedFlow()

sealed interface UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent
    data class NavigateTo(val route: String) : UiEvent
}
```

## Anti-Patterns

- **No Business Logic in Composables**: Keep UI pure
- **No Direct Repository Access**: Use ViewModels/UseCases
- **No Mutable State Exposure**: Expose read-only StateFlow
- **No Context in ViewModel**: Pass dependencies explicitly
- **No Large ViewModels**: Split into multiple if >300 lines

## Reference & Examples

For complete architecture examples:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

jetpack-compose/state-management | jetpack-compose/navigation | kotlin/coroutines
