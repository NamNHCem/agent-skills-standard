---
name: Hilt Dependency Injection
description: Compile-time dependency injection for Android with Dagger-based annotation processing.
metadata:
  labels: [kotlin, hilt, dagger, dependency-injection, di, android]
  triggers:
    files: ['**/di/*.kt', '**/hilt/*.kt', '**/*Module.kt']
    keywords: [HiltAndroidApp, HiltViewModel, Module, InstallIn, Provides, Binds, Inject]
---

# Hilt Dependency Injection

## **Priority: P1 (OPERATIONAL)**

Compile-time dependency injection built on top of Dagger for Android applications with reduced boilerplate and better performance.

## Why Hilt?

### Hilt vs Koin
- **Hilt**: Annotation-based, compile-time DI, better performance, compile-time safety
- **Koin**: Pure Kotlin DSL, runtime DI, no code generation, easier learning curve

**Use Hilt when**:
- Android-only projects
- Need compile-time safety
- Large-scale enterprise apps
- Performance is critical
- Team familiar with Dagger

**Use Koin when**:
- Kotlin Multiplatform projects
- Simpler DI needs
- Faster build times preferred
- Team prefers DSL over annotations

## Setup

### Gradle Dependencies
```kotlin
// Project build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android") version "2.50" apply false
}

// App build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android")
    id("kotlin-kapt")
}

dependencies {
    // Hilt
    implementation("com.google.dagger:hilt-android:2.50")
    kapt("com.google.dagger:hilt-android-compiler:2.50")
    
    // Hilt ViewModel
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")
    
    // Hilt Testing
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.50")
    kaptAndroidTest("com.google.dagger:hilt-android-compiler:2.50")
}

// Enable Hilt
kapt {
    correctErrorTypes = true
}
```

### Application Setup
```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

### AndroidManifest.xml
```xml
<application
    android:name=".MyApplication"
    ...>
</application>
```

## Component Hierarchy

### Hilt Components
```
SingletonComponent (Application)
    ↓
ActivityRetainedComponent
    ↓
ViewModelComponent
    ↓
ActivityComponent
    ↓
FragmentComponent
    ↓
ViewComponent
    ↓
ViewWithFragmentComponent
    ↓
ServiceComponent
```

### Component Scopes
- `@Singleton` - Application lifetime
- `@ActivityRetainedScoped` - Survives configuration changes
- `@ViewModelScoped` - ViewModel lifetime
- `@ActivityScoped` - Activity lifetime
- `@FragmentScoped` - Fragment lifetime
- `@ServiceScoped` - Service lifetime

## Module Definition

### Basic Module
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
    
    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi {
        return retrofit.create(UserApi::class.java)
    }
}
```

### Binds (Interface to Implementation)
```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

### Provides + Binds in Same Module
```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
    
    companion object {
        @Provides
        @Singleton
        fun provideDatabase(
            @ApplicationContext context: Context
        ): AppDatabase {
            return Room.databaseBuilder(
                context,
                AppDatabase::class.java,
                "app_database"
            ).build()
        }
    }
}
```

## Injection Patterns

### Constructor Injection (Recommended)
```kotlin
// ✅ Good - Constructor injection
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    private val dao: UserDao
) : UserRepository {
    override suspend fun getUser(id: String): User {
        return api.getUser(id)
    }
}
```

### Field Injection (Activities/Fragments)
```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    @Inject
    lateinit var analytics: Analytics
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // analytics is injected before onCreate
    }
}
```

### ViewModel Injection
```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            val user = getUserUseCase(userId)
            // ...
        }
    }
}

// Usage in Activity
@AndroidEntryPoint
class HomeActivity : AppCompatActivity() {
    private val viewModel: HomeViewModel by viewModels()
}

// Usage in Compose
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    // ...
}
```

## Qualifiers

### Named Qualifiers
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    @Named("auth")
    fun provideAuthOkHttpClient(
        authInterceptor: AuthInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .build()
    }
    
    @Provides
    @Singleton
    @Named("public")
    fun providePublicOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().build()
    }
    
    @Provides
    @Singleton
    fun provideAuthApi(
        @Named("auth") okHttpClient: OkHttpClient
    ): AuthApi {
        return createRetrofit(okHttpClient).create(AuthApi::class.java)
    }
}

// Usage
class MyRepository @Inject constructor(
    @Named("auth") private val authClient: OkHttpClient,
    @Named("public") private val publicClient: OkHttpClient
)
```

### Custom Qualifiers
```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptor

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LoggingInterceptor

@Module
@InstallIn(SingletonComponent::class)
object InterceptorModule {
    
    @Provides
    @AuthInterceptor
    fun provideAuthInterceptor(): Interceptor {
        return Interceptor { chain ->
            val request = chain.request().newBuilder()
                .addHeader("Authorization", "Bearer token")
                .build()
            chain.proceed(request)
        }
    }
    
    @Provides
    @LoggingInterceptor
    fun provideLoggingInterceptor(): Interceptor {
        return HttpLoggingInterceptor().apply {
            level = HttpLoggingInterceptor.Level.BODY
        }
    }
}
```

## Android Components

### Activity
```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    private val viewModel: MainViewModel by viewModels()
    
    @Inject
    lateinit var analytics: Analytics
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MainScreen(viewModel)
        }
    }
}
```

### Fragment
```kotlin
@AndroidEntryPoint
class HomeFragment : Fragment() {
    
    private val viewModel: HomeViewModel by viewModels()
    
    // Shared ViewModel with Activity
    private val sharedViewModel: SharedViewModel by activityViewModels()
    
    @Inject
    lateinit var userRepository: UserRepository
}
```

### Compose
```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    HomeContent(uiState)
}

// With navigation
@Composable
fun DetailScreen(
    viewModel: DetailViewModel = hiltViewModel()
) {
    // ViewModel scoped to navigation entry
}
```

### Service
```kotlin
@AndroidEntryPoint
class MyService : Service() {
    
    @Inject
    lateinit var repository: UserRepository
    
    override fun onBind(intent: Intent?): IBinder? {
        return null
    }
}
```

## Advanced Patterns

### AssistedInject (Runtime Parameters)
```kotlin
class UserDetailViewModel @AssistedInject constructor(
    @Assisted private val userId: String,
    private val repository: UserRepository
) : ViewModel() {
    
    @AssistedFactory
    interface Factory {
        fun create(userId: String): UserDetailViewModel
    }
}

// Usage
@AndroidEntryPoint
class DetailActivity : AppCompatActivity() {
    
    @Inject
    lateinit var viewModelFactory: UserDetailViewModel.Factory
    
    private val viewModel by lazy {
        viewModelFactory.create(intent.getStringExtra("userId")!!)
    }
}
```

### Multi-Bindings (Set/Map)
```kotlin
// Set Multi-binding
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {
    
    @Binds
    @IntoSet
    abstract fun bindFirebaseAnalytics(
        impl: FirebaseAnalyticsImpl
    ): AnalyticsProvider
    
    @Binds
    @IntoSet
    abstract fun bindMixpanelAnalytics(
        impl: MixpanelAnalyticsImpl
    ): AnalyticsProvider
}

class AnalyticsManager @Inject constructor(
    private val providers: Set<@JvmSuppressWildcards AnalyticsProvider>
) {
    fun logEvent(event: String) {
        providers.forEach { it.log(event) }
    }
}

// Map Multi-binding
@MapKey
annotation class ViewModelKey(val value: KClass<out ViewModel>)

@Module
@InstallIn(ViewModelComponent::class)
abstract class ViewModelModule {
    
    @Binds
    @IntoMap
    @ViewModelKey(HomeViewModel::class)
    abstract fun bindHomeViewModel(
        viewModel: HomeViewModel
    ): ViewModel
}
```

### Entry Points (Access from Non-Android Classes)
```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface RepositoryEntryPoint {
    fun userRepository(): UserRepository
}

// Usage in non-Android class
class MyWorker(
    context: Context,
    params: WorkerParameters
) : Worker(context, params) {
    
    override fun doWork(): Result {
        val appContext = applicationContext
        val hiltEntryPoint = EntryPointAccessors.fromApplication(
            appContext,
            RepositoryEntryPoint::class.java
        )
        val repository = hiltEntryPoint.userRepository()
        
        // Use repository
        return Result.success()
    }
}
```

## Testing

### Setup Test Application
```kotlin
@HiltAndroidTest
class HomeViewModelTest {
    
    @get:Rule
    var hiltRule = HiltAndroidRule(this)
    
    @Inject
    lateinit var repository: UserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun testLoadUser() = runTest {
        // Test with injected dependencies
    }
}
```

### Replace Bindings for Tests
```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [DataModule::class]
)
object FakeDataModule {
    
    @Provides
    @Singleton
    fun provideFakeUserRepository(): UserRepository {
        return FakeUserRepository()
    }
}
```

### Custom Test Runner
```kotlin
class HiltTestRunner : AndroidJUnitRunner() {
    override fun newApplication(
        cl: ClassLoader?,
        className: String?,
        context: Context?
    ): Application {
        return super.newApplication(cl, HiltTestApplication::class.java.name, context)
    }
}

// In build.gradle.kts
android {
    defaultConfig {
        testInstrumentationRunner = "com.example.HiltTestRunner"
    }
}
```

## Best Practices

### Module Organization
```kotlin
// ✅ Good - Organized by layer
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    // Network-related dependencies
}

@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    // Data layer bindings
}

@Module
@InstallIn(SingletonComponent::class)
abstract class DomainModule {
    // Domain layer bindings
}

// ❌ Bad - Everything in one module
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    // Too many unrelated dependencies
}
```

### Prefer Binds Over Provides
```kotlin
// ✅ Good - Use @Binds for interfaces
@Binds
abstract fun bindUserRepository(
    impl: UserRepositoryImpl
): UserRepository

// ❌ Bad - Unnecessary @Provides
@Provides
fun provideUserRepository(
    impl: UserRepositoryImpl
): UserRepository = impl
```

### Avoid Context Leaks
```kotlin
// ❌ Bad - Holding Activity context in singleton
@Provides
@Singleton
fun provideManager(activity: Activity): Manager {
    return Manager(activity) // Leak!
}

// ✅ Good - Use Application context
@Provides
@Singleton
fun provideManager(
    @ApplicationContext context: Context
): Manager {
    return Manager(context)
}
```

## Migration from Koin

### Koin to Hilt Mapping

| Koin | Hilt |
|------|------|
| `single { }` | `@Provides @Singleton` |
| `factory { }` | `@Provides` (no scope) |
| `viewModel { }` | `@HiltViewModel` |
| `get()` | `@Inject constructor()` |
| `named("name")` | `@Named("name")` |
| `scope<Activity>` | `@ActivityScoped` |

### Example Migration
```kotlin
// Koin
val dataModule = module {
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
}

// Hilt
@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

## Anti-Patterns

- **No Field Injection in ViewModels**: Use constructor injection
- **No @Provides for Simple Bindings**: Use @Binds
- **No Singleton for Stateful Objects**: Use appropriate scope
- **No Missing @AndroidEntryPoint**: Required for injection
- **No Circular Dependencies**: Refactor architecture

## Performance Tips

1. **Use @Binds**: Faster than @Provides
2. **Minimize Singleton Scope**: Use narrower scopes when possible
3. **Lazy Injection**: Use `Provider<T>` for lazy initialization
4. **Avoid Reflection**: Hilt uses code generation (fast)

## Reference & Examples

For complete Hilt examples and patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/language | kotlin/coroutines | jetpack-compose/architecture

> **💡 Alternative**: For runtime DI with simpler setup, see [kotlin/koin-di](../koin-di/SKILL.md).
