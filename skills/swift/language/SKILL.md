---
name: Swift Language Best Practices
description: Modern Swift coding standards, optional handling, and idiomatic patterns.
metadata:
  labels: [swift, ios, best-practices]
  triggers:
    files: ['**/*.swift']
    keywords: [guard, if let, optional, struct, class, protocol]
---

# Swift Language Best Practices

## **Priority: P1 (OPERATIONAL)**

Idiomatic Swift patterns for clean, safe, and performant iOS applications.

## Optional Handling

### Prefer `guard let` over `if let` for Early Exit
Reduces nesting and keeps the "happy path" on the left edge.

```swift
// ✅ GOOD
func updateProfile(user: User?) {
    guard let user = user else {
        print("Error: No user found")
        return
    }
    
    // Use 'user' safely here
    display(user.name)
}

// ❌ BAD
func updateProfile(user: User?) {
    if let user = user {
        // Nested logic
        display(user.name)
    } else {
        print("Error: No user found")
    }
}
```

### Avoid Force Unwrapping (`!`)
Never use `!` unless app crashing is the desired behavior for that state (e.g., loaded assets or view outlets in correct lifecycle).

```swift
// ✅ GOOD
let url = URL(string: "https://api.com")
// Safe unwrap later or use guard

// ❌ BAD
let url = URL(string: "https://api.com")! // Crashes if string is invalid
```

## Struct vs Class

### Prefer Value Types (Struct) by Default
Use `struct` for data models, states, and simple logic containers. They are thread-safe (passed by copy) and performant (stack allocated).

```swift
// ✅ GOOD
struct UserState {
    var name: String
    var isLoggedIn: Bool
}

// Use Class for identity or long-lived objects
class UserManager: ObservableObject {
    @Published var state = UserState(...)
}
```

## Modern Concurrency (Async/Await)

### Replace Closures with Async/Await
```swift
// ✅ GOOD
func fetchUser() async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// ❌ BAD (Legacy Callback Hell)
func fetchUser(completion: @escaping (Result<User, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, error in
        // ... handling ...
    }.resume()
}
```

## Protocol Oriented Programming

### Use Protocols for Composition
Prefer composition over inheritance.

```swift
protocol Cacheable {
    func save()
}

protocol Loggable {
    func log()
}

// Capabilities composed via extensions
struct DataManager: Cacheable, Loggable { ... }
```

## Access Control

### Explicit Access Levels
- `private`: File-private access.
- `internal`: Module-wide access (default).
- `public`: API access.

```swift
// ✅ GOOD - Hide implementation details
class ViewModel {
    private var internalState: Int = 0
    public var stateDisplay: String { "\(internalState)" }
}
```
