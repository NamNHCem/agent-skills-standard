---
name: Kotlin Best Practices
description: Idiomatic Kotlin coding standards and conventions.
metadata:
  labels: [kotlin, best-practices, conventions, code-style]
  triggers:
    files: ['**/*.kt']
    keywords: [kotlin, class, function, property]
---

# Kotlin Best Practices

## **Priority: P1 (OPERATIONAL)**

Idiomatic Kotlin patterns for clean, maintainable code.

## Naming Conventions

### Classes & Interfaces
- **PascalCase**: `UserRepository`, `DataSource`
- **Interfaces**: No `I` prefix (e.g., `Repository` not `IRepository`)
- **Sealed Classes**: Suffix with type (e.g., `UiState`, `Result`)

### Functions & Properties
- **camelCase**: `getUserById()`, `isValid`
- **Boolean Properties**: Prefix with `is`, `has`, `can` (e.g., `isLoading`, `hasError`)
- **Extension Functions**: Descriptive names (e.g., `String.toTitleCase()`)

### Constants
- **UPPER_SNAKE_CASE**: `const val MAX_RETRY_COUNT = 3`
- **Companion Object**: Place constants in companion object
  ```kotlin
  companion object {
      const val TAG = "MainActivity"
      private const val REQUEST_CODE = 100
  }
  ```

## Code Organization

### File Structure
```text
package com.example.feature

// Imports (grouped and sorted)
import android.os.Bundle
import androidx.lifecycle.ViewModel

// Top-level constants/functions (if any)
private const val DEFAULT_TIMEOUT = 5000L

// Main class
class FeatureViewModel : ViewModel() { ... }

// Extension functions (related to main class)
private fun String.sanitize() = this.trim()
```

### Class Member Order
1. Companion object
2. Properties (public → private)
3. Init blocks
4. Constructors
5. Public methods
6. Private methods
7. Inner/nested classes

## Null Safety

### Safe Call Operator (?.)
```kotlin
// ✅ Good - Safe call
val length = user?.name?.length

// ❌ Bad - Potential NPE
val length = user.name.length // Crashes if user or name is null
```

### Elvis Operator (?:)
```kotlin
// ✅ Good - Provide default
val name = user?.name ?: "Unknown"
val count = items?.size ?: 0

// ✅ Good - Early return
fun processUser(user: User?) {
    val validUser = user ?: return
    // Work with non-null user
}
```

### Avoid Not-Null Assertion (!!)
```kotlin
// ❌ Bad - Can crash
val name = user!!.name // Only use if 100% certain

// ✅ Good - Safe alternatives
val name = user?.name ?: "Unknown"
val name = requireNotNull(user?.name) { "User name is required" }
```

### Let for Null Checks
```kotlin
// ✅ Good - Execute only if non-null
user?.let { validUser ->
    println("User: ${validUser.name}")
    saveUser(validUser)
}

// ❌ Bad - Unnecessary null check
if (user != null) {
    println("User: ${user.name}")
    saveUser(user)
}
```

### Nullable Types Best Practices
```kotlin
// ✅ Good - Prefer non-nullable
data class User(
    val id: String,
    val name: String,
    val email: String? = null // Only nullable if truly optional
)

// ❌ Bad - Unnecessary nullability
data class User(
    val id: String?,
    val name: String?,
    val email: String?
)
```

### Safe Casts (as?)
```kotlin
// ✅ Good - Safe cast
val user = obj as? User
user?.let { processUser(it) }

// ❌ Bad - Unsafe cast
val user = obj as User // Can throw ClassCastException
```

## Immutability

### Prefer Val Over Var
```kotlin
// ✅ Good
val userId = getUserId()
val items = mutableListOf<Item>() // Mutable reference to mutable list

// ❌ Avoid
var userId = getUserId() // Unless reassignment needed
```

### Immutable Collections
```kotlin
// ✅ Good - Expose immutable
class Repository {
    private val _items = mutableListOf<Item>()
    val items: List<Item> get() = _items.toList()
}

// ❌ Bad - Exposes mutable state
class Repository {
    val items = mutableListOf<Item>()
}
```

## Function Design

### Single Expression Functions
```kotlin
// ✅ Good
fun double(x: Int) = x * 2

// ❌ Verbose
fun double(x: Int): Int {
    return x * 2
}
```

### Named Arguments
```kotlin
// ✅ Good - Clear intent
createUser(
    name = "John",
    email = "john@example.com",
    age = 30
)

// ❌ Unclear
createUser("John", "john@example.com", 30)
```

### Default Parameters
```kotlin
// ✅ Good - Avoid overloads
fun fetchData(
    limit: Int = 10,
    offset: Int = 0,
    sortBy: String = "date"
) { ... }

// ❌ Avoid multiple overloads
fun fetchData() = fetchData(10, 0, "date")
fun fetchData(limit: Int) = fetchData(limit, 0, "date")
```

## String Templates

```kotlin
// ✅ Good
val message = "User $name has $count items"
val complex = "Result: ${calculateResult()}"

// ❌ Avoid concatenation
val message = "User " + name + " has " + count + " items"
```

## When Expressions

### Exhaustive When
```kotlin
// ✅ Good - Exhaustive
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val message: String) : Result()
}

fun handle(result: Result) = when (result) {
    is Result.Success -> showData(result.data)
    is Result.Error -> showError(result.message)
    // Compiler ensures all cases covered
}
```

### Prefer When Over If-Else Chains
```kotlin
// ✅ Good
val color = when (status) {
    Status.ACTIVE -> Color.GREEN
    Status.PENDING -> Color.YELLOW
    Status.INACTIVE -> Color.RED
}

// ❌ Verbose
val color = if (status == Status.ACTIVE) {
    Color.GREEN
} else if (status == Status.PENDING) {
    Color.YELLOW
} else {
    Color.RED
}
```

## Destructuring

```kotlin
// ✅ Good
data class User(val name: String, val email: String)
val (name, email) = getUser()

// Map iteration
for ((key, value) in map) {
    println("$key -> $value")
}
```

## Anti-Patterns

- **No Magic Numbers**: Use named constants
- **No Deep Nesting**: Extract functions, use early returns
- **No Unnecessary Null Checks**: Leverage Kotlin's null safety
- **No Java-Style Getters/Setters**: Use properties
- **No Semicolons**: Kotlin doesn't require them

## Reference & Examples

For detailed examples and advanced patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/language | kotlin/coroutines | jetpack-compose/ui-components
