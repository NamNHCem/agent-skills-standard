---
name: Jetpack Compose UI Components
description: Best practices for building reusable and performant Compose UI components.
metadata:
  labels: [compose, ui, components, modifiers, theming]
  triggers:
    files: ['**/*Screen.kt', '**/*Component.kt']
    keywords: [Composable, Modifier, remember, LaunchedEffect, derivedStateOf]
---

# Jetpack Compose UI Components

## **Priority: P1 (OPERATIONAL)**

Building reusable, performant, and maintainable UI components with Jetpack Compose.

## Composable Functions

### Naming Conventions
```kotlin
// ✅ Good - PascalCase for Composables
@Composable
fun UserCard(user: User, onClick: () -> Unit) { }

// ✅ Good - Lowercase for non-UI functions
@Composable
fun rememberUserState(): UserState { }

// ❌ Bad - Inconsistent naming
@Composable
fun userCard() { } // Should be PascalCase
```

### Function Signatures
```kotlin
// ✅ Good - Modifier first, events last
@Composable
fun CustomButton(
    text: String,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    onClick: () -> Unit
) {
    Button(
        onClick = onClick,
        enabled = enabled,
        modifier = modifier
    ) {
        Text(text)
    }
}
```

## State Management

### Remember & MutableState
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

### RememberSaveable (Survives Configuration Changes)
```kotlin
@Composable
fun SearchBar() {
    var query by rememberSaveable { mutableStateOf("") }
    
    TextField(
        value = query,
        onValueChange = { query = it }
    )
}
```

### DerivedStateOf (Computed State)
```kotlin
@Composable
fun FilteredList(items: List<Item>, query: String) {
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
```

## Side Effects

### LaunchedEffect (Coroutine Lifecycle)
```kotlin
@Composable
fun UserScreen(userId: String, viewModel: UserViewModel) {
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }
}
```

### DisposableEffect (Cleanup)
```kotlin
@Composable
fun EventListener() {
    DisposableEffect(Unit) {
        val listener = EventListener { /* handle event */ }
        EventBus.register(listener)
        
        onDispose {
            EventBus.unregister(listener)
        }
    }
}
```

### SideEffect (Synchronize State)
```kotlin
@Composable
fun AnalyticsScreen(screenName: String) {
    SideEffect {
        analytics.logScreenView(screenName)
    }
}
```

## Modifiers

### Modifier Order (Matters!)
```kotlin
// ✅ Good - Correct order
Box(
    modifier = Modifier
        .size(100.dp)           // 1. Size
        .padding(16.dp)         // 2. Padding
        .background(Color.Blue) // 3. Background
        .clickable { }          // 4. Interactions
)

// ❌ Bad - Wrong order
Box(
    modifier = Modifier
        .clickable { }          // Clickable area too small!
        .padding(16.dp)
        .size(100.dp)
)
```

### Custom Modifiers
```kotlin
fun Modifier.shimmerEffect(): Modifier = composed {
    var targetValue by remember { mutableStateOf(0f) }
    
    LaunchedEffect(Unit) {
        while (true) {
            animate(0f, 1000f, animationSpec = tween(1000)) { value, _ ->
                targetValue = value
            }
        }
    }
    
    this.background(
        Brush.horizontalGradient(
            colors = listOf(Color.Gray, Color.LightGray, Color.Gray),
            startX = targetValue - 1000f,
            endX = targetValue
        )
    )
}
```

## Lists & Performance

### LazyColumn/LazyRow
```kotlin
@Composable
fun UserList(users: List<User>, onUserClick: (User) -> Unit) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            items = users,
            key = { it.id } // ✅ Important for performance
        ) { user ->
            UserCard(
                user = user,
                onClick = { onUserClick(user) }
            )
        }
    }
}
```

### LazyVerticalGrid
```kotlin
@Composable
fun PhotoGrid(photos: List<Photo>) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 128.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(photos, key = { it.id }) { photo ->
            PhotoItem(photo)
        }
    }
}
```

## Theming

### Material 3 Theme
```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) {
        darkColorScheme(
            primary = Purple80,
            secondary = PurpleGrey80,
            tertiary = Pink80
        )
    } else {
        lightColorScheme(
            primary = Purple40,
            secondary = PurpleGrey40,
            tertiary = Pink40
        )
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

### Custom Theme Values
```kotlin
data class AppColors(
    val success: Color,
    val warning: Color,
    val error: Color
)

val LocalAppColors = staticCompositionLocalOf {
    AppColors(
        success = Color.Green,
        warning = Color.Yellow,
        error = Color.Red
    )
}

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    CompositionLocalProvider(LocalAppColors provides AppColors(...)) {
        MaterialTheme(content = content)
    }
}

// Usage
val appColors = LocalAppColors.current
Text("Success", color = appColors.success)
```

## Animations

### AnimatedVisibility
```kotlin
@Composable
fun ExpandableCard(expanded: Boolean) {
    AnimatedVisibility(
        visible = expanded,
        enter = fadeIn() + expandVertically(),
        exit = fadeOut() + shrinkVertically()
    ) {
        Text("Expanded content")
    }
}
```

### Animate*AsState
```kotlin
@Composable
fun PulsingButton(enabled: Boolean) {
    val scale by animateFloatAsState(
        targetValue = if (enabled) 1f else 0.9f,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy)
    )
    
    Button(
        onClick = { },
        modifier = Modifier.scale(scale)
    ) {
        Text("Click me")
    }
}
```

## Preview

### Multiple Previews
```kotlin
@Preview(name = "Light Mode", showBackground = true)
@Preview(name = "Dark Mode", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
private fun UserCardPreview() {
    AppTheme {
        UserCard(
            user = User("John Doe", "john@example.com"),
            onClick = { }
        )
    }
}
```

### Preview Parameters
```kotlin
class UserPreviewParameterProvider : PreviewParameterProvider<User> {
    override val values = sequenceOf(
        User("John", "john@example.com"),
        User("Jane", "jane@example.com")
    )
}

@Preview
@Composable
fun UserCardPreview(
    @PreviewParameter(UserPreviewParameterProvider::class) user: User
) {
    UserCard(user = user, onClick = { })
}
```

## Anti-Patterns

- **No Side Effects in Composition**: Use LaunchedEffect/SideEffect
- **No Heavy Computation in Composables**: Use remember/derivedStateOf
- **No Missing Keys in Lists**: Always provide `key` parameter
- **No Nested LazyColumns**: Use single LazyColumn with different item types
- **No Modifier Reuse**: Create new Modifier chains for each component

## Reference & Examples

For advanced component patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

jetpack-compose/state-management | jetpack-compose/performance | jetpack-compose/architecture
