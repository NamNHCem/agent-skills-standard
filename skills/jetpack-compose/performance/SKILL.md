---
name: Jetpack Compose Performance
description: Performance optimization techniques for Jetpack Compose.
metadata:
  labels: [compose, performance, recomposition, stability]
  triggers:
    files: ['**/*.kt']
    keywords: [Composable, remember, derivedStateOf, Stable, Immutable]
---

# Jetpack Compose Performance

## **Priority: P1 (OPERATIONAL)**

Optimization techniques for smooth 60fps Compose UIs.

## Recomposition Optimization

### Stable & Immutable Annotations
```kotlin
// ✅ Good - Mark stable data classes
@Immutable
data class User(val name: String, val email: String)

@Stable
class UserState(initialUser: User) {
    var user by mutableStateOf(initialUser)
}

// Compose can skip recomposition if User hasn't changed
@Composable
fun UserCard(user: User) { }
```

### Remember Expensive Calculations
```kotlin
@Composable
fun FilteredList(items: List<Item>, query: String) {
    // ✅ Good - Cached computation
    val filteredItems = remember(items, query) {
        items.filter { it.name.contains(query, ignoreCase = true) }
    }
    
    // ❌ Bad - Recomputes every recomposition
    val filteredItems = items.filter { it.name.contains(query) }
}
```

### DerivedStateOf for Computed Values
```kotlin
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    // ✅ Good - Only recomposes when visibility changes
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }
    
    AnimatedVisibility(visible = showButton) {
        FloatingActionButton(onClick = { /* scroll to top */ }) {
            Icon(Icons.Default.ArrowUpward, null)
        }
    }
}
```

## Lambda Optimization

### Remember Lambdas
```kotlin
@Composable
fun ItemList(items: List<Item>, onItemClick: (Item) -> Unit) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            // ✅ Good - Lambda is stable
            val onClick = remember(item) { { onItemClick(item) } }
            ItemRow(item, onClick)
        }
    }
}
```

## Key Parameter in Lists

### Always Provide Keys
```kotlin
@Composable
fun UserList(users: List<User>) {
    LazyColumn {
        // ✅ Good - Stable keys prevent unnecessary recomposition
        items(
            items = users,
            key = { it.id }
        ) { user ->
            UserCard(user)
        }
        
        // ❌ Bad - No key, items recompose on list changes
        items(users) { user ->
            UserCard(user)
        }
    }
}
```

## Avoid Unnecessary Recomposition

### Split Composables
```kotlin
// ✅ Good - Only Counter recomposes
@Composable
fun Screen() {
    Header() // Doesn't recompose
    Counter() // Recomposes when count changes
    Footer() // Doesn't recompose
}

@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// ❌ Bad - Entire Screen recomposes
@Composable
fun ScreenBad() {
    var count by remember { mutableStateOf(0) }
    Column {
        Header()
        Button(onClick = { count++ }) {
            Text("Count: $count")
        }
        Footer()
    }
}
```

## Modifier Reuse

### Don't Create Modifiers in Loops
```kotlin
@Composable
fun ItemList(items: List<Item>) {
    // ❌ Bad - Creates new modifier for each item
    LazyColumn {
        items(items) { item ->
            Text(
                text = item.name,
                modifier = Modifier.fillMaxWidth().padding(16.dp)
            )
        }
    }
    
    // ✅ Good - Reuse modifier
    val itemModifier = Modifier.fillMaxWidth().padding(16.dp)
    LazyColumn {
        items(items) { item ->
            Text(text = item.name, modifier = itemModifier)
        }
    }
}
```

## Defer State Reads

### Read State in Lambda
```kotlin
@Composable
fun DeferredReads() {
    var count by remember { mutableStateOf(0) }
    
    // ✅ Good - Only lambda recomposes
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
    
    // ❌ Bad - Entire composable recomposes
    val text = "Count: $count"
    Button(onClick = { count++ }) {
        Text(text)
    }
}
```

## Lambda-Based Modifiers

### Skip Composition/Layout Phases
```kotlin
// ✅ Good - Lambda-based modifiers skip composition/layout
@Composable
fun AnimatedBox() {
    var offset by remember { mutableStateOf(0f) }
    
    Box(
        modifier = Modifier
            .offset { IntOffset(offset.toInt(), 0) } // Only draw phase
            .graphicsLayer { translationX = offset } // Only draw phase
    )
}

// ❌ Bad - Triggers full recomposition
@Composable
fun AnimatedBoxBad() {
    var offset by remember { mutableStateOf(0f) }
    
    Box(
        modifier = Modifier.offset(offset.dp, 0.dp) // Full recomposition
    )
}
```

## Avoid Backward Writes

### Never Modify State After Reading
```kotlin
// ❌ Bad - Infinite recomposition
@Composable
fun BadExample() {
    var count by remember { mutableStateOf(0) }
    count++ // Never do this!
    Text("$count")
}

// ✅ Good - Update in callback
@Composable
fun GoodExample() {
    var count by remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("$count")
    }
}
```

## Baseline Profiles

### Generate Baseline Profile
```kotlin
// build.gradle.kts
plugins {
    id("androidx.baselineprofile") version "1.2.0"
}

dependencies {
    baselineProfile(project(":benchmark"))
}
```

## Anti-Patterns

- **No Unstable Parameters**: Mark data classes as @Immutable
- **No Heavy Work in Composition**: Use LaunchedEffect
- **No Missing Keys**: Always provide key in lists
- **No Inline Lambdas in Lists**: Remember them
- **No Reading State Too Early**: Defer to usage site

## Reference & Examples

See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

jetpack-compose/ui-components | jetpack-compose/state-management
