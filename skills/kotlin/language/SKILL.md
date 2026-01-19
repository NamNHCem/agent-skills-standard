---
name: Kotlin Language Patterns
description: Modern Kotlin language features, idioms, and advanced patterns.
metadata:
  labels: [kotlin, language, coroutines, sealed-classes, data-classes]
  triggers:
    files: ['**/*.kt', '**/*.kts']
    keywords: [sealed, data class, object, companion, inline, reified, suspend]
---

# Kotlin Language Patterns

## **Priority: P0 (CRITICAL)**

Modern Kotlin language features for type-safe, concise, and expressive code.

## Core Language Features

### Data Classes & Sealed Classes
- **Data Classes**: Use for DTOs, Models, and Value Objects
  - Auto-generates `equals()`, `hashCode()`, `toString()`, `copy()`
  - Use `copy()` for immutable updates
- **Sealed Classes/Interfaces**: Use for restricted type hierarchies
  - Exhaustive `when` expressions
  - Perfect for State, Result, and Event modeling

### Null Safety
- **Nullable Types**: Use `?` operator explicitly
- **Safe Calls**: `?.` for null-safe access
- **Elvis Operator**: `?:` for default values
- **Non-null Assertion**: `!!` only when 100% certain (avoid in production)
- **Safe Casts**: `as?` instead of `as`

### Extension Functions
- **Use Cases**: Add functionality to existing classes without inheritance
- **Naming**: Use descriptive names (e.g., `String.toSnakeCase()`)
- **Scope**: Keep extensions close to usage or in dedicated `Extensions.kt`

### Scope Functions
- **`let`**: Transform nullable objects, scope limiting
- **`run`**: Object configuration and computation
- **`with`**: Multiple calls on same object
- **`apply`**: Object configuration (returns receiver)
- **`also`**: Side effects (returns receiver)

### Inline & Reified
- **`inline`**: Eliminate lambda overhead for higher-order functions
- **`reified`**: Access generic type at runtime (requires `inline`)
  ```kotlin
  inline fun <reified T> Gson.fromJson(json: String): T = 
      fromJson(json, T::class.java)
  ```

### Companion Objects
- **Use for**: Factory methods, constants, static-like members
- **Naming**: Use `@JvmStatic` for Java interop
- **Interfaces**: Can implement interfaces for polymorphism

## Collections & Sequences

### Collection Operations
- **Immutable by Default**: Use `listOf()`, `setOf()`, `mapOf()`
- **Mutable**: Only when necessary (`mutableListOf()`, etc.)
- **Transformations**: Prefer `map`, `filter`, `flatMap` over loops
- **Aggregations**: Use `fold`, `reduce`, `groupBy`, `partition`

### Sequences
- **Use for**: Large collections or chained operations
- **Lazy Evaluation**: Operations execute only when terminal operation called
  ```kotlin
  list.asSequence()
      .filter { it > 10 }
      .map { it * 2 }
      .toList() // Terminal operation
  ```

## Coroutines Basics

### Suspend Functions
- **Definition**: Functions that can be paused and resumed
- **Marking**: Use `suspend` modifier
- **Calling**: Only from other suspend functions or coroutine builders

### Coroutine Builders
- **`launch`**: Fire-and-forget (returns `Job`)
- **`async`**: Concurrent computation (returns `Deferred<T>`)
- **`runBlocking`**: Bridge blocking/non-blocking (tests, main functions)
- **`withContext`**: Switch dispatcher, return result

### Structured Concurrency
- **Scope**: Always use `CoroutineScope` (ViewModelScope, lifecycleScope)
- **Cancellation**: Respect cancellation, use `isActive` checks
- **Exception Handling**: Use `try-catch` or `CoroutineExceptionHandler`

## Anti-Patterns

- **No `!!` in Production**: Always handle nullability properly
- **No Mutable Collections as Public API**: Return immutable views
- **No Blocking in Coroutines**: Use `withContext(Dispatchers.IO)` for blocking calls
- **No Lateinit for Nullable Types**: Use nullable types instead
- **No Empty Catch Blocks**: Always log or handle exceptions

## Reference & Examples

For detailed code examples and advanced patterns:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/coroutines | kotlin/best-practices | jetpack-compose/architecture
