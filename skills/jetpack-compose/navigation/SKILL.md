---
name: Jetpack Compose Navigation
description: Type-safe navigation patterns with Compose Navigation.
metadata:
  labels: [compose, navigation, navigation-compose, deep-links]
  triggers:
    files: ['**/*Navigation.kt', '**/*NavHost.kt']
    keywords: [NavHost, NavController, composable, navigate, popBackStack]
---

# Jetpack Compose Navigation

## **Priority: P1 (OPERATIONAL)**

Type-safe navigation implementation with Jetpack Compose Navigation.

## Basic Setup

### NavHost Configuration
```kotlin
@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController(),
    startDestination: String = Screen.Home.route
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable(Screen.Home.route) {
            HomeScreen(
                onNavigateToDetails = { id ->
                    navController.navigate(Screen.Details.createRoute(id))
                }
            )
        }
        
        composable(
            route = Screen.Details.route,
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")
            DetailsScreen(
                id = id,
                onNavigateBack = { navController.popBackStack() }
            )
        }
    }
}
```

### Sealed Class Routes
```kotlin
sealed class Screen(val route: String) {
    data object Home : Screen("home")
    data object Profile : Screen("profile")
    data object Details : Screen("details/{id}") {
        fun createRoute(id: String) = "details/$id"
    }
    data object Settings : Screen("settings?tab={tab}") {
        fun createRoute(tab: String? = null) = 
            if (tab != null) "settings?tab=$tab" else "settings"
    }
}
```

## Type-Safe Arguments

### Required Arguments
```kotlin
composable(
    route = "user/{userId}/post/{postId}",
    arguments = listOf(
        navArgument("userId") { type = NavType.StringType },
        navArgument("postId") { type = NavType.IntType }
    )
) { backStackEntry ->
    val userId = backStackEntry.arguments?.getString("userId")!!
    val postId = backStackEntry.arguments?.getInt("postId")!!
    
    PostScreen(userId, postId)
}
```

### Optional Arguments
```kotlin
composable(
    route = "search?query={query}",
    arguments = listOf(
        navArgument("query") {
            type = NavType.StringType
            nullable = true
            defaultValue = null
        }
    )
) { backStackEntry ->
    val query = backStackEntry.arguments?.getString("query")
    SearchScreen(initialQuery = query)
}
```

## Navigation Actions

### Navigate with Arguments
```kotlin
// Simple navigation
navController.navigate(Screen.Profile.route)

// With arguments
navController.navigate(Screen.Details.createRoute("123"))

// With options
navController.navigate(Screen.Settings.route) {
    popUpTo(Screen.Home.route) { inclusive = false }
    launchSingleTop = true
}
```

### Pop Back Stack
```kotlin
// Pop current screen
navController.popBackStack()

// Pop to specific destination
navController.popBackStack(Screen.Home.route, inclusive = false)

// Pop with result
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("result", data)
navController.popBackStack()
```

## Deep Links

### Deep Link Configuration
```kotlin
composable(
    route = Screen.Details.route,
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "myapp://details/{id}"
        },
        navDeepLink {
            uriPattern = "https://myapp.com/details/{id}"
        }
    )
) { backStackEntry ->
    DetailsScreen(backStackEntry.arguments?.getString("id"))
}
```

### Manifest Configuration
```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" android:host="details" />
    </intent-filter>
</activity>
```

## Nested Navigation

### Nested NavHost
```kotlin
@Composable
fun MainNavHost(navController: NavHostController) {
    NavHost(navController, startDestination = "main") {
        composable("main") {
            MainScreen(
                onNavigateToAuth = { navController.navigate("auth") }
            )
        }
        
        navigation(startDestination = "login", route = "auth") {
            composable("login") { LoginScreen() }
            composable("register") { RegisterScreen() }
        }
    }
}
```

## Bottom Navigation

### Bottom Nav Integration
```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route
    
    Scaffold(
        bottomBar = {
            NavigationBar {
                bottomNavItems.forEach { item ->
                    NavigationBarItem(
                        selected = currentRoute == item.route,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) }
                    )
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = Screen.Home.route,
            modifier = Modifier.padding(paddingValues)
        ) {
            // Destinations
        }
    }
}
```

## Anti-Patterns

- **No NavController in ViewModels**: Pass as parameter to Composables
- **No String Routes Directly**: Use sealed classes
- **No Ignoring Back Stack**: Handle back properly
- **No Deep Nesting**: Keep navigation hierarchy flat

## Reference & Examples

See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

jetpack-compose/architecture | jetpack-compose/state-management
