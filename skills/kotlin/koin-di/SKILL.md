---
name: Koin Dependency Injection
description: Lightweight dependency injection framework for Kotlin with DSL-based configuration.
metadata:
  labels: [kotlin, koin, dependency-injection, di, android]
  triggers:
    files: ['**/di/*.kt', '**/koin/*.kt', '**/*Module.kt']
    keywords: [koinApplication, module, single, factory, viewModel, inject, get, koinViewModel]
---

# Koin Dependency Injection

## **Priority: P1 (OPERATIONAL)**

Lightweight, pragmatic dependency injection for Kotlin applications with simple DSL-based configuration.

## Why Koin?

### Koin vs Hilt/Dagger
- **Koin**: Pure Kotlin DSL, no code generation, runtime DI, easier learning curve
- **Hilt**: Annotation-based, compile-time DI, better performance, more boilerplate

**Use Koin when**:
- Kotlin Multiplatform projects
- Simpler DI needs
- Faster build times preferred
- Team prefers DSL over annotations

**Use Hilt when**:
- Android-only projects
- Need compile-time safety
- Large-scale enterprise apps
- Performance is critical

## Setup

### Gradle Dependencies
```kotlin
// build.gradle.kts
dependencies {
    // Koin Core
    implementation("io.insert-koin:koin-core:3.5.3")
    
    // Koin Android
    implementation("io.insert-koin:koin-android:3.5.3")
    
    // Koin Compose
    implementation("io.insert-koin:koin-androidx-compose:3.5.3")
    
    // Koin Test
    testImplementation("io.insert-koin:koin-test:3.5.3")
    testImplementation("io.insert-koin:koin-test-junit4:3.5.3")
}
```

### Application Setup
```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        startKoin {
            androidLogger() // Android logger
            androidContext(this@MyApplication) // Android context
            modules(appModule, dataModule, domainModule) // Modules
        }
    }
}
```

## Module Definition

### Basic Module
```kotlin
val appModule = module {
    // Singleton - Created once, shared
    single<UserRepository> { UserRepositoryImpl(get()) }
    
    // Factory - New instance every time
    factory<GetUserUseCase> { GetUserUseCase(get()) }
    
    // ViewModel
    viewModel { HomeViewModel(get(), get()) }
}
```

### Named Dependencies
```kotlin
val networkModule = module {
    // Named instances
    single(named("auth")) { 
        provideOkHttpClient(authInterceptor = get()) 
    }
    
    single(named("public")) { 
        provideOkHttpClient(authInterceptor = null) 
    }
    
    single<AuthApi> { 
        provideRetrofit(get(named("auth"))) 
    }
    
    single<PublicApi> { 
        provideRetrofit(get(named("public"))) 
    }
}
```

### Scoped Dependencies
```kotlin
val dataModule = module {
    // Scope to activity/fragment lifecycle
    scope<MainActivity> {
        scoped { MainActivityPresenter(get()) }
    }
    
    // Scope to custom scope
    scope(named("session")) {
        scoped<SessionManager> { SessionManagerImpl(get()) }
    }
}
```

## Injection Patterns

### Constructor Injection (Recommended)
```kotlin
// ✅ Good - Constructor injection
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao
) {
    suspend fun getUser(id: String): User {
        return api.getUser(id)
    }
}

// Module
val dataModule = module {
    single<UserRepository> { UserRepository(get(), get()) }
}
```

### Property Injection
```kotlin
class MyViewModel : ViewModel() {
    // Lazy injection
    private val repository: UserRepository by inject()
    
    // Direct injection
    private val useCase: GetUserUseCase = get()
}
```

### Android Components

#### Activity
```kotlin
class MainActivity : AppCompatActivity() {
    // Lazy injection
    private val viewModel: MainViewModel by viewModel()
    
    // With parameters
    private val detailViewModel: DetailViewModel by viewModel {
        parametersOf(userId)
    }
}
```

#### Fragment
```kotlin
class HomeFragment : Fragment() {
    private val viewModel: HomeViewModel by viewModel()
    
    // Shared ViewModel with Activity
    private val sharedViewModel: SharedViewModel by activityViewModel()
}
```

#### Compose
```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = koinViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    HomeContent(uiState)
}

// With parameters
@Composable
fun DetailScreen(userId: String) {
    val viewModel: DetailViewModel = koinViewModel {
        parametersOf(userId)
    }
}
```

## Advanced Patterns

### Parameterized Injection
```kotlin
// Define with parameters
val userModule = module {
    factory { (userId: String) ->
        GetUserUseCase(userId, get())
    }
}

// Inject with parameters
class MyViewModel : ViewModel() {
    fun loadUser(userId: String) {
        val useCase: GetUserUseCase = get { parametersOf(userId) }
    }
}
```

### Bindings & Interfaces
```kotlin
val dataModule = module {
    // Bind interface to implementation
    single<UserRepository> { UserRepositoryImpl(get()) }
    
    // Multiple bindings
    single { UserRepositoryImpl(get()) } bind UserRepository::class
    
    // Bind to multiple interfaces
    single { DataSourceImpl() } binds arrayOf(
        LocalDataSource::class,
        RemoteDataSource::class
    )
}
```

### Lazy Modules
```kotlin
// Load modules on demand
val featureModule = module {
    single { FeatureRepository(get()) }
}

// In Application
startKoin {
    modules(coreModule)
}

// Later, when feature is accessed
loadKoinModules(featureModule)

// Unload when done
unloadKoinModules(featureModule)
```

### Qualifiers
```kotlin
val networkModule = module {
    single(named("base_url")) { "https://api.example.com" }
    single(named("timeout")) { 30L }
    
    single<OkHttpClient> {
        OkHttpClient.Builder()
            .connectTimeout(get(named("timeout")), TimeUnit.SECONDS)
            .build()
    }
}
```

## Testing

### Unit Tests
```kotlin
class UserRepositoryTest : KoinTest {
    
    @get:Rule
    val koinTestRule = KoinTestRule.create {
        modules(testModule)
    }
    
    // Inject for testing
    private val repository: UserRepository by inject()
    
    @Test
    fun `test get user`() = runTest {
        val user = repository.getUser("123")
        assertEquals("John", user.name)
    }
}

// Test module with mocks
val testModule = module {
    single<UserApi> { mockk<UserApi>() }
    single<UserRepository> { UserRepositoryImpl(get()) }
}
```

### Mocking Dependencies
```kotlin
class ViewModelTest : KoinTest {
    
    private val mockRepository: UserRepository = mockk()
    
    @Before
    fun setup() {
        startKoin {
            modules(module {
                single<UserRepository> { mockRepository }
                viewModel { HomeViewModel(get()) }
            })
        }
    }
    
    @After
    fun tearDown() {
        stopKoin()
    }
    
    @Test
    fun `test load user`() {
        val viewModel: HomeViewModel by inject()
        // Test viewModel
    }
}
```

### Verify Modules
```kotlin
@Test
fun `verify koin modules`() {
    koinApplication {
        modules(appModule, dataModule, domainModule)
        checkModules()
    }
}
```

## Best Practices

### Module Organization
```kotlin
// ✅ Good - Organized by layer
val dataModule = module {
    single<UserApi> { provideUserApi(get()) }
    single<UserDao> { provideUserDao(get()) }
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
}

val domainModule = module {
    factory { GetUserUseCase(get()) }
    factory { UpdateUserUseCase(get()) }
}

val presentationModule = module {
    viewModel { HomeViewModel(get(), get()) }
    viewModel { ProfileViewModel(get()) }
}

// ❌ Bad - Everything in one module
val appModule = module {
    single<UserApi> { ... }
    single<UserRepository> { ... }
    factory { GetUserUseCase(get()) }
    viewModel { HomeViewModel(get()) }
}
```

### Prefer Single Over Factory
```kotlin
// ✅ Good - Use single for stateless dependencies
val dataModule = module {
    single<UserRepository> { UserRepositoryImpl(get()) }
    single<SettingsRepository> { SettingsRepositoryImpl(get()) }
}

// ❌ Bad - Factory for stateless (creates new instance every time)
val dataModule = module {
    factory<UserRepository> { UserRepositoryImpl(get()) }
}

// ✅ Good - Use factory for stateful or short-lived
val domainModule = module {
    factory { (userId: String) -> GetUserUseCase(userId, get()) }
}
```

### Avoid Context Leaks
```kotlin
// ❌ Bad - Holding Activity context in singleton
val appModule = module {
    single { MyManager(androidContext() as Activity) } // Leak!
}

// ✅ Good - Use Application context
val appModule = module {
    single { MyManager(androidContext()) }
}

// ✅ Good - Scope to activity
val appModule = module {
    scope<MainActivity> {
        scoped { ActivityScopedManager(getSource()) }
    }
}
```

## Migration from Hilt

### Hilt to Koin Mapping

| Hilt | Koin |
|------|------|
| `@Singleton` | `single { }` |
| `@HiltViewModel` | `viewModel { }` |
| `@Inject constructor()` | Constructor params with `get()` |
| `@Provides` | `single { }` or `factory { }` |
| `@Binds` | `single { Impl() } bind Interface::class` |
| `@Named("name")` | `named("name")` |
| `@ActivityScoped` | `scope<Activity> { scoped { } }` |

### Example Migration
```kotlin
// Hilt
@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    @Provides
    @Singleton
    fun provideUserRepository(
        api: UserApi,
        dao: UserDao
    ): UserRepository = UserRepositoryImpl(api, dao)
}

// Koin
val dataModule = module {
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
}
```

## Anti-Patterns

- **No get() in Module Definition**: Use lazy `get()` to avoid circular dependencies
- **No Context Leaks**: Don't hold Activity context in singletons
- **No Over-Scoping**: Use appropriate scope (single/factory/scoped)
- **No Reflection in Production**: Koin uses reflection, consider performance
- **No Missing Modules**: Always check modules are loaded

## Performance Tips

1. **Use single for stateless**: Avoid creating new instances
2. **Lazy loading**: Load modules only when needed
3. **Avoid deep dependency graphs**: Keep dependencies flat
4. **Profile startup**: Use `androidLogger(Level.ERROR)` in production

## Reference & Examples

For complete Koin examples and patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/language | kotlin/coroutines | jetpack-compose/architecture

> **💡 Alternative**: For compile-time DI with better performance, see Hilt/Dagger documentation.
