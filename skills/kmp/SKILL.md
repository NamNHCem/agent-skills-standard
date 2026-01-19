---
name: Kotlin Multiplatform (KMP)
description: Share Kotlin code across Android, iOS, Web, Desktop, and Server platforms.
metadata:
  labels: [kotlin, kmp, multiplatform, cross-platform, ios, android]
  triggers:
    files: ['**/commonMain/**/*.kt', '**/androidMain/**/*.kt', '**/iosMain/**/*.kt', 'build.gradle.kts']
    keywords: [kotlin-multiplatform, commonMain, expect, actual, iosMain, androidMain]
---

# Kotlin Multiplatform (KMP)

## **Priority: P0 (CRITICAL)**

Share business logic, networking, and data layers across multiple platforms while keeping UI platform-specific.

## Architecture Overview

### Typical KMP Project Structure
```
shared/
├── commonMain/          # Shared code (all platforms)
│   ├── kotlin/
│   │   ├── domain/     # Business logic
│   │   ├── data/       # Repositories, data sources
│   │   └── util/       # Utilities
│   └── resources/
├── androidMain/         # Android-specific
│   └── kotlin/
├── iosMain/            # iOS-specific
│   └── kotlin/
├── desktopMain/        # Desktop-specific (optional)
│   └── kotlin/
└── jsMain/             # Web-specific (optional)
    └── kotlin/
```

### What to Share
✅ **Share**:
- Business logic (use cases, domain models)
- Data layer (repositories, data sources)
- Networking (API clients)
- Database (SQLDelight, Realm)
- State management (ViewModels with KMP)
- Utilities (date formatting, validation)

❌ **Don't Share**:
- UI code (use Compose Multiplatform for that)
- Platform-specific APIs (camera, sensors)
- Navigation (platform-specific)

## Setup

### Gradle Configuration
```kotlin
// shared/build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.20"
    kotlin("native.cocoapods") version "1.9.20" // For iOS
    id("com.android.library")
    kotlin("plugin.serialization") version "1.9.20"
}

kotlin {
    // Android target
    androidTarget {
        compilations.all {
            kotlinOptions {
                jvmTarget = "17"
            }
        }
    }
    
    // iOS targets
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "shared"
            isStatic = true
        }
    }
    
    // Common dependencies
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
                implementation("io.ktor:ktor-client-core:2.3.7")
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-android:2.3.7")
            }
        }
        
        val iosMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-darwin:2.3.7")
            }
        }
        
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
    }
}
```

## Expect/Actual Pattern

### Platform-Specific Implementations

#### Common (Interface)
```kotlin
// commonMain/kotlin/Platform.kt
expect class Platform() {
    val name: String
}

expect fun getPlatform(): Platform
```

#### Android (Implementation)
```kotlin
// androidMain/kotlin/Platform.android.kt
actual class Platform actual constructor() {
    actual val name: String = "Android ${android.os.Build.VERSION.SDK_INT}"
}

actual fun getPlatform(): Platform = Platform()
```

#### iOS (Implementation)
```kotlin
// iosMain/kotlin/Platform.ios.kt
import platform.UIKit.UIDevice

actual class Platform actual constructor() {
    actual val name: String = 
        UIDevice.currentDevice.systemName() + " " + 
        UIDevice.currentDevice.systemVersion
}

actual fun getPlatform(): Platform = Platform()
```

### Common Use Cases

#### UUID Generation
```kotlin
// commonMain
expect fun randomUUID(): String

// androidMain
import java.util.UUID
actual fun randomUUID(): String = UUID.randomUUID().toString()

// iosMain
import platform.Foundation.NSUUID
actual fun randomUUID(): String = NSUUID().UUIDString()
```

#### Date Formatting
```kotlin
// commonMain
expect class DateFormatter() {
    fun format(timestamp: Long, pattern: String): String
}

// androidMain
import java.text.SimpleDateFormat
import java.util.Date

actual class DateFormatter {
    actual fun format(timestamp: Long, pattern: String): String {
        val sdf = SimpleDateFormat(pattern)
        return sdf.format(Date(timestamp))
    }
}

// iosMain
import platform.Foundation.*

actual class DateFormatter {
    actual fun format(timestamp: Long, pattern: String): String {
        val date = NSDate.dateWithTimeIntervalSince1970(timestamp / 1000.0)
        val formatter = NSDateFormatter()
        formatter.dateFormat = pattern
        return formatter.stringFromDate(date)
    }
}
```

## Shared Business Logic

### Repository Pattern
```kotlin
// commonMain/kotlin/data/UserRepository.kt
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun updateUser(user: User): Result<Unit>
}

class UserRepositoryImpl(
    private val api: UserApi,
    private val database: UserDatabase
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> {
        return try {
            val user = api.getUser(id)
            database.saveUser(user)
            Result.success(user)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

### Use Cases
```kotlin
// commonMain/kotlin/domain/GetUserUseCase.kt
class GetUserUseCase(
    private val repository: UserRepository
) {
    suspend operator fun invoke(userId: String): Result<User> {
        return repository.getUser(userId)
    }
}
```

### Shared ViewModel (with KMP)
```kotlin
// commonMain/kotlin/presentation/HomeViewModel.kt
class HomeViewModel(
    private val getUserUseCase: GetUserUseCase
) {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            when (val result = getUserUseCase(userId)) {
                is Result.Success -> _uiState.value = UiState.Success(result.data)
                is Result.Failure -> _uiState.value = UiState.Error(result.message)
            }
        }
    }
}
```

## Networking with Ktor

### API Client
```kotlin
// commonMain/kotlin/data/remote/UserApi.kt
interface UserApi {
    suspend fun getUser(id: String): User
    suspend fun updateUser(user: User): User
}

class UserApiImpl(
    private val httpClient: HttpClient
) : UserApi {
    override suspend fun getUser(id: String): User {
        return httpClient.get("https://api.example.com/users/$id").body()
    }
    
    override suspend fun updateUser(user: User): User {
        return httpClient.put("https://api.example.com/users/${user.id}") {
            contentType(ContentType.Application.Json)
            setBody(user)
        }.body()
    }
}

// HttpClient factory
fun createHttpClient(): HttpClient {
    return HttpClient {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                isLenient = true
            })
        }
        install(Logging) {
            level = LogLevel.INFO
        }
    }
}
```

## Database with SQLDelight

### Setup
```kotlin
// shared/build.gradle.kts
plugins {
    id("app.cash.sqldelight") version "2.0.1"
}

sqldelight {
    databases {
        create("AppDatabase") {
            packageName.set("com.example.db")
        }
    }
}
```

### Schema
```sql
-- commonMain/sqldelight/com/example/db/User.sq
CREATE TABLE User (
    id TEXT NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL
);

selectAll:
SELECT * FROM User;

selectById:
SELECT * FROM User WHERE id = ?;

insert:
INSERT OR REPLACE INTO User(id, name, email)
VALUES (?, ?, ?);

deleteById:
DELETE FROM User WHERE id = ?;
```

### Usage
```kotlin
// commonMain/kotlin/data/local/UserDatabase.kt
class UserDatabase(driver: SqlDriver) {
    private val database = AppDatabase(driver)
    private val queries = database.userQueries
    
    fun getAllUsers(): List<User> {
        return queries.selectAll().executeAsList()
    }
    
    fun getUserById(id: String): User? {
        return queries.selectById(id).executeAsOneOrNull()
    }
    
    fun saveUser(user: User) {
        queries.insert(user.id, user.name, user.email)
    }
}

// Platform-specific driver
// androidMain
fun createDriver(context: Context): SqlDriver {
    return AndroidSqliteDriver(AppDatabase.Schema, context, "app.db")
}

// iosMain
fun createDriver(): SqlDriver {
    return NativeSqliteDriver(AppDatabase.Schema, "app.db")
}
```

## Dependency Injection

### Koin for KMP
```kotlin
// commonMain/kotlin/di/CommonModule.kt
val commonModule = module {
    single { createHttpClient() }
    single<UserApi> { UserApiImpl(get()) }
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
    factory { GetUserUseCase(get()) }
}

// androidMain/kotlin/di/AndroidModule.kt
val androidModule = module {
    single { createDriver(androidContext()) }
    single { UserDatabase(get()) }
}

// iosMain/kotlin/di/IosModule.kt
val iosModule = module {
    single { createDriver() }
    single { UserDatabase(get()) }
}

// Initialize
// Android
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            modules(commonModule, androidModule)
        }
    }
}

// iOS (in Swift)
func initKoin() {
    KoinKt.doInitKoin { koin in
        koin.modules([commonModule, iosModule])
    }
}
```

## iOS Integration

### Swift Interop
```swift
// iOS App
import shared

class UserViewModel: ObservableObject {
    @Published var user: User?
    @Published var isLoading = false
    
    private let getUserUseCase: GetUserUseCase
    
    init(getUserUseCase: GetUserUseCase) {
        self.getUserUseCase = getUserUseCase
    }
    
    func loadUser(userId: String) {
        isLoading = true
        
        getUserUseCase.invoke(userId: userId) { result, error in
            DispatchQueue.main.async {
                self.isLoading = false
                if let user = result as? User {
                    self.user = user
                }
            }
        }
    }
}
```

### Flow to Combine
```kotlin
// iosMain/kotlin/util/FlowExtensions.kt
fun <T> Flow<T>.asPublisher(): AnyPublisher<T, Never> {
    return FlowPublisher(flow: self).eraseToAnyPublisher()
}
```

## Testing

### Common Tests
```kotlin
// commonTest/kotlin/GetUserUseCaseTest.kt
class GetUserUseCaseTest {
    private val mockRepository = mockk<UserRepository>()
    private val useCase = GetUserUseCase(mockRepository)
    
    @Test
    fun `test get user success`() = runTest {
        val user = User("1", "John", "john@example.com")
        coEvery { mockRepository.getUser("1") } returns Result.success(user)
        
        val result = useCase("1")
        
        assertTrue(result.isSuccess)
        assertEquals(user, result.getOrNull())
    }
}
```

## Best Practices

### 1. Keep UI Platform-Specific
```kotlin
// ❌ Bad - Trying to share UI
// commonMain
@Composable
fun UserScreen() { ... } // Won't work well cross-platform

// ✅ Good - Share ViewModel, keep UI separate
// commonMain
class UserViewModel { ... }

// androidMain (Jetpack Compose)
@Composable
fun UserScreen(viewModel: UserViewModel) { ... }

// iosMain (SwiftUI)
// struct UserView: View { ... }
```

### 2. Use Interfaces for Platform APIs
```kotlin
// ✅ Good
// commonMain
interface ImageLoader {
    suspend fun loadImage(url: String): ByteArray
}

// androidMain
class AndroidImageLoader : ImageLoader {
    override suspend fun loadImage(url: String): ByteArray {
        // Use Coil/Glide
    }
}

// iosMain
class IosImageLoader : ImageLoader {
    override suspend fun loadImage(url: String): ByteArray {
        // Use native iOS APIs
    }
}
```

### 3. Minimize expect/actual
```kotlin
// ❌ Bad - Too many expect/actual
expect fun formatDate(...)
expect fun parseJson(...)
expect fun encodeBase64(...)

// ✅ Good - Use multiplatform libraries
// Use kotlinx-datetime, kotlinx-serialization, etc.
```

## Anti-Patterns

- **No UI in commonMain**: Keep UI platform-specific
- **No Platform-Specific Code in Common**: Use expect/actual
- **No Blocking Calls**: Use suspend functions
- **No Missing iOS Framework**: Configure CocoaPods properly
- **No Ignoring Memory Model**: Be aware of iOS memory model

## Reference & Examples

For complete KMP examples and patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/coroutines | kotlin/koin-di | compose-multiplatform

> **💡 Next Step**: For shared UI across platforms, see [compose-multiplatform](../../compose-multiplatform/SKILL.md).
