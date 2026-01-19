---
name: Compose Multiplatform (CMP)
description: Build native UIs for Android, iOS, Desktop, and Web with a single Kotlin codebase using Jetpack Compose.
metadata:
  labels: [compose, multiplatform, cross-platform, ui, ios, android, desktop, web]
  triggers:
    files: ['**/commonMain/**/*.kt', '**/composeApp/**/*.kt']
    keywords: [Composable, compose-multiplatform, expect, actual, @Composable]
---

# Compose Multiplatform (CMP)

## **Priority: P0 (CRITICAL)**

Share UI code across Android, iOS, Desktop, and Web using Jetpack Compose with platform-specific customizations when needed.

## What is Compose Multiplatform?

**Compose Multiplatform** = Jetpack Compose + Kotlin Multiplatform
- Share **UI code** across platforms (not just business logic)
- Native performance on all platforms
- Platform-specific customizations available
- Single codebase for Android, iOS, Desktop, Web

### CMP vs Other Solutions

| Solution | UI Sharing | Performance | Platform Feel |
|----------|-----------|-------------|---------------|
| **CMP** | ✅ High | ✅ Native | ✅ Native |
| Flutter | ✅ High | ✅ Good | ⚠️ Custom |
| React Native | ✅ High | ⚠️ Bridge | ✅ Native |
| KMP only | ❌ No UI | ✅ Native | ✅ Native |

## Project Structure

### Typical CMP Project
```
composeApp/
├── commonMain/          # Shared UI + logic
│   ├── kotlin/
│   │   ├── ui/         # Composables
│   │   ├── viewmodel/  # ViewModels
│   │   ├── navigation/ # Navigation
│   │   └── theme/      # Theme, colors, typography
│   └── resources/      # Images, strings
├── androidMain/         # Android-specific
│   └── kotlin/
├── iosMain/            # iOS-specific
│   └── kotlin/
├── desktopMain/        # Desktop-specific
│   └── kotlin/
└── wasmJsMain/         # Web-specific
    └── kotlin/
```

## Setup

### Gradle Configuration
```kotlin
// composeApp/build.gradle.kts
plugins {
    kotlin("multiplatform")
    id("org.jetbrains.compose") version "1.5.11"
}

kotlin {
    androidTarget()
    
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "ComposeApp"
            isStatic = true
        }
    }
    
    jvm("desktop")
    
    js(IR) {
        browser()
    }
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material3)
                implementation(compose.ui)
                implementation(compose.components.resources)
                implementation(compose.components.uiToolingPreview)
                
                // Navigation
                implementation("org.jetbrains.androidx.navigation:navigation-compose:2.7.0-alpha03")
                
                // ViewModel
                implementation("org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose:2.8.0-alpha02")
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation("androidx.activity:activity-compose:1.8.2")
            }
        }
        
        val iosMain by getting
        val desktopMain by getting
    }
}
```

## Shared UI Components

### Basic Composables
```kotlin
// commonMain/kotlin/ui/HomeScreen.kt
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = koinViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Home") }
            )
        }
    ) { paddingValues ->
        when (uiState) {
            is UiState.Loading -> LoadingIndicator()
            is UiState.Success -> UserList(
                users = uiState.users,
                onUserClick = viewModel::onUserClick,
                modifier = Modifier.padding(paddingValues)
            )
            is UiState.Error -> ErrorView(
                message = uiState.message,
                onRetry = viewModel::retry
            )
        }
    }
}

@Composable
fun UserList(
    users: List<User>,
    onUserClick: (User) -> Unit,
    modifier: Modifier = Modifier
) {
    LazyColumn(modifier = modifier) {
        items(users, key = { it.id }) { user ->
            UserCard(
                user = user,
                onClick = { onUserClick(user) }
            )
        }
    }
}
```

### Platform-Specific UI

#### Expect/Actual for Platform UI
```kotlin
// commonMain
@Composable
expect fun PlatformSpecificButton(
    text: String,
    onClick: () -> Unit
)

// androidMain
@Composable
actual fun PlatformSpecificButton(
    text: String,
    onClick: () -> Unit
) {
    Button(onClick = onClick) {
        Text(text)
    }
}

// iosMain
@Composable
actual fun PlatformSpecificButton(
    text: String,
    onClick: () -> Unit
) {
    // iOS-specific styling
    Button(
        onClick = onClick,
        colors = ButtonDefaults.buttonColors(
            containerColor = MaterialTheme.colorScheme.primary
        )
    ) {
        Text(text)
    }
}
```

#### Platform Detection
```kotlin
// commonMain
enum class Platform {
    Android, iOS, Desktop, Web
}

expect fun getPlatform(): Platform

// Usage
@Composable
fun AdaptiveLayout() {
    when (getPlatform()) {
        Platform.Android, Platform.iOS -> MobileLayout()
        Platform.Desktop -> DesktopLayout()
        Platform.Web -> WebLayout()
    }
}
```

## Navigation

### Multiplatform Navigation
```kotlin
// commonMain/kotlin/navigation/Navigation.kt
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Profile : Screen("profile/{userId}") {
        fun createRoute(userId: String) = "profile/$userId"
    }
    object Settings : Screen("settings")
}

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = Screen.Home.route
    ) {
        composable(Screen.Home.route) {
            HomeScreen(
                onNavigateToProfile = { userId ->
                    navController.navigate(Screen.Profile.createRoute(userId))
                }
            )
        }
        
        composable(
            route = Screen.Profile.route,
            arguments = listOf(navArgument("userId") { type = NavType.StringType })
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getString("userId")
            ProfileScreen(
                userId = userId,
                onNavigateBack = { navController.popBackStack() }
            )
        }
        
        composable(Screen.Settings.route) {
            SettingsScreen()
        }
    }
}
```

## Resources

### Multiplatform Resources
```kotlin
// commonMain/composeResources/
├── drawable/
│   ├── ic_home.xml
│   └── logo.png
├── font/
│   └── roboto.ttf
└── values/
    └── strings.xml

// Usage
@Composable
fun ResourceExample() {
    Column {
        // Images
        Image(
            painter = painterResource(Res.drawable.logo),
            contentDescription = "Logo"
        )
        
        // Strings
        Text(stringResource(Res.string.app_name))
        
        // Fonts
        Text(
            text = "Custom Font",
            fontFamily = FontFamily(Font(Res.font.roboto))
        )
    }
}
```

## Theme

### Shared Theme
```kotlin
// commonMain/kotlin/theme/Theme.kt
private val LightColorScheme = lightColorScheme(
    primary = Color(0xFF6200EE),
    onPrimary = Color.White,
    secondary = Color(0xFF03DAC6),
    onSecondary = Color.Black,
    background = Color.White,
    onBackground = Color.Black
)

private val DarkColorScheme = darkColorScheme(
    primary = Color(0xFFBB86FC),
    onPrimary = Color.Black,
    secondary = Color(0xFF03DAC6),
    onSecondary = Color.Black,
    background = Color(0xFF121212),
    onBackground = Color.White
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}

// Platform-specific dark mode detection
expect fun isSystemInDarkTheme(): Boolean

// androidMain
actual fun isSystemInDarkTheme(): Boolean {
    return androidx.compose.foundation.isSystemInDarkTheme()
}

// iosMain
import platform.UIKit.UIScreen
actual fun isSystemInDarkTheme(): Boolean {
    return UIScreen.mainScreen.traitCollection.userInterfaceStyle == 
        UIUserInterfaceStyle.UIUserInterfaceStyleDark
}
```

## ViewModel

### Shared ViewModel
```kotlin
// commonMain/kotlin/viewmodel/HomeViewModel.kt
class HomeViewModel(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadUsers()
    }
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            when (val result = getUsersUseCase()) {
                is Result.Success -> _uiState.value = UiState.Success(result.data)
                is Result.Error -> _uiState.value = UiState.Error(result.message)
            }
        }
    }
    
    fun onUserClick(user: User) {
        // Handle click
    }
    
    fun retry() {
        loadUsers()
    }
}

sealed interface UiState {
    object Loading : UiState
    data class Success(val users: List<User>) : UiState
    data class Error(val message: String) : UiState
}
```

## Platform Entry Points

### Android
```kotlin
// androidMain/kotlin/MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            AppTheme {
                App()
            }
        }
    }
}
```

### iOS
```swift
// iosApp/iosApp.swift
import SwiftUI
import ComposeApp

@main
struct iOSApp: App {
    init() {
        KoinKt.doInitKoin()
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

struct ContentView: View {
    var body: some View {
        ComposeView()
            .ignoresSafeArea(.all)
    }
}

struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        MainViewControllerKt.MainViewController()
    }
    
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

```kotlin
// iosMain/kotlin/MainViewController.kt
fun MainViewController() = ComposeUIViewController {
    AppTheme {
        App()
    }
}
```

### Desktop
```kotlin
// desktopMain/kotlin/main.kt
fun main() = application {
    Window(
        onCloseRequest = ::exitApplication,
        title = "My App"
    ) {
        AppTheme {
            App()
        }
    }
}
```

### Web
```kotlin
// wasmJsMain/kotlin/main.kt
fun main() {
    ComposeViewport(document.body!!) {
        AppTheme {
            App()
        }
    }
}
```

## Platform-Specific Features

### Camera Access
```kotlin
// commonMain
expect class CameraController() {
    fun takePicture(onResult: (ByteArray) -> Unit)
}

// androidMain
actual class CameraController {
    actual fun takePicture(onResult: (ByteArray) -> Unit) {
        // Use CameraX
    }
}

// iosMain
actual class CameraController {
    actual fun takePicture(onResult: (ByteArray) -> Unit) {
        // Use UIImagePickerController
    }
}

// Usage in Composable
@Composable
fun CameraScreen() {
    val cameraController = remember { CameraController() }
    
    Button(onClick = {
        cameraController.takePicture { imageData ->
            // Handle image
        }
    }) {
        Text("Take Picture")
    }
}
```

## Best Practices

### 1. Adaptive Layouts
```kotlin
@Composable
fun AdaptiveUserList(users: List<User>) {
    val windowSize = rememberWindowSize()
    
    when (windowSize) {
        WindowSize.Compact -> {
            // Mobile: Single column
            LazyColumn {
                items(users) { user ->
                    UserCard(user)
                }
            }
        }
        WindowSize.Medium, WindowSize.Expanded -> {
            // Tablet/Desktop: Grid
            LazyVerticalGrid(
                columns = GridCells.Adaptive(minSize = 300.dp)
            ) {
                items(users) { user ->
                    UserCard(user)
                }
            }
        }
    }
}
```

### 2. Platform-Specific Styling
```kotlin
@Composable
fun PlatformButton(
    text: String,
    onClick: () -> Unit
) {
    val buttonColors = when (getPlatform()) {
        Platform.iOS -> ButtonDefaults.buttonColors(
            containerColor = Color(0xFF007AFF) // iOS blue
        )
        else -> ButtonDefaults.buttonColors()
    }
    
    Button(
        onClick = onClick,
        colors = buttonColors
    ) {
        Text(text)
    }
}
```

### 3. Minimize Platform-Specific Code
```kotlin
// ❌ Bad - Too much platform-specific code
@Composable
fun MyScreen() {
    when (getPlatform()) {
        Platform.Android -> AndroidScreen()
        Platform.iOS -> IosScreen()
        Platform.Desktop -> DesktopScreen()
    }
}

// ✅ Good - Shared UI with adaptive behavior
@Composable
fun MyScreen() {
    AdaptiveLayout {
        SharedContent()
    }
}
```

## Testing

### Shared UI Tests
```kotlin
// commonTest/kotlin/ui/HomeScreenTest.kt
class HomeScreenTest {
    
    @Test
    fun `test home screen displays users`() = runComposeUiTest {
        val users = listOf(
            User("1", "John"),
            User("2", "Jane")
        )
        
        setContent {
            HomeScreen(
                uiState = UiState.Success(users)
            )
        }
        
        onNodeWithText("John").assertIsDisplayed()
        onNodeWithText("Jane").assertIsDisplayed()
    }
}
```

## Anti-Patterns

- **No Platform-Specific UI in Common**: Use expect/actual
- **No Ignoring Platform Guidelines**: Adapt to platform UX
- **No Heavy Computation in Composables**: Use ViewModels
- **No Missing Resources**: Ensure resources exist for all platforms
- **No Hardcoded Sizes**: Use adaptive layouts

## Performance Tips

1. **Lazy Loading**: Use LazyColumn/LazyRow for lists
2. **Remember**: Cache expensive computations
3. **Stable Keys**: Provide keys for list items
4. **Avoid Recomposition**: Use derivedStateOf
5. **Platform Optimization**: Use platform-specific optimizations when needed

## Reference & Examples

For complete CMP examples and patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kmp | jetpack-compose/ui-components | jetpack-compose/architecture

> **💡 Prerequisite**: Requires understanding of [KMP](../kmp/SKILL.md) and [Jetpack Compose](../jetpack-compose/ui-components/SKILL.md).
