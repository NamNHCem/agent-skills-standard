---
name: SwiftUI Architecture & State
description: Best practices for SwiftUI state management and MVVM architecture.
metadata:
  labels: [swiftui, ios, architecture, mvvm, state]
  triggers:
    files: ['**/*.swift']
    keywords: [SwiftUI, View, State, Binding, ObservedObject, StateObject, EnvironmentObject]
---

# SwiftUI Architecture & State

## **Priority: P0 (CRITICAL)**

State management is the single most common source of bugs in SwiftUI. Follow these strict rules to ensure predictable UI updates.

## The Source of Truth Rules

### 1. `@State` (Private View State)
Use only for **transient UI state** strictly local to a single View (e.g., toggle ON/OFF, scroll position, local input).
**Always mark as private.**

```swift
struct LoginView: View {
    @State private var email: String = "" // ✅ Correct
    @State private var isHighligted: Bool = false
}
```

### 2. `@StateObject` (ViewModel Owner)
Use when the View **owns/creates** the ObservableObject. This ensures the object survives View redraws.

```swift
struct UserProfileView: View {
    // ✅ View owns this ViewModel instance
    // It will NOT be recreated when View redraws
    @StateObject private var viewModel = UserViewModel()
    
    var body: some View { ... }
}
```

### 3. `@ObservedObject` (Dependency ViewModel)
Use when the View **receives** an existing ObservableObject from a parent. The View does NOT own it.

```swift
struct SubView: View {
    // ✅ Passed from parent
    @ObservedObject var viewModel: UserViewModel
}
```

**⚠️ DANGER ZONE**:
Never initialize a ViewModel with `@ObservedObject`.
```swift
// ❌ CRITICAL BUG: Re-inits on every redraw, losing state!
@ObservedObject var viewModel = UserViewModel() 
```

### 4. `@Binding` (Two-Way Connection)
Use to allow a child view to modify a parent's `@State` without owning it.

```swift
struct ToggleButton: View {
    @Binding var isOn: Bool // ✅ Connects to parent's state
    
    var body: some View {
        Toggle("Switch", isOn: $isOn)
    }
}
```

## MVVM Architecture Pattern

### ViewModel Structure
ViewModels must be `MainActor` to ensure UI updates happen on the main thread safely.

```swift
@MainActor
class LoginViewModel: ObservableObject {
    // Published properties trigger UI updates
    @Published var email = ""
    @Published var password = ""
    @Published var state: ViewState = .idle
    
    private let authService: AuthService
    
    init(authService: AuthService) {
        self.authService = authService
    }
    
    func login() async {
        state = .loading
        do {
            let user = try await authService.login(email, password)
            state = .success(user)
        } catch {
            state = .error(error.localizedDescription)
        }
    }
}
```

### View Implementation
Views should be dumb and purely reactive to ViewModel state.

```swift
struct LoginView: View {
    @StateObject private var viewModel = LoginViewModel(authService: .shared)
    
    var body: some View {
        VStack {
            TextField("Email", text: $viewModel.email)
            SecureField("Password", text: $viewModel.password)
            
            Button("Login") {
                Task { await viewModel.login() }
            }
            .disabled(viewModel.state == .loading)
            
            if case .loading = viewModel.state {
                ProgressView()
            }
        }
    }
}
```

## Anti-Patterns to Avoid

### ❌ Heavy Logic in Views
Do not put networking, formatting, or heavy computation inside `var body`.

### ❌ Unnecessary `@EnvironmentObject`
Avoid using EnvironmentObject for everything (Global State). It makes views hard to reuse and test since dependencies are implicit. Favor explicit injection via `@ObservedObject` or constructor injection.

### ❌ Ignoring View Equatable
For complex lists, making views explicitly `Equatable` and using `.equatable()` can prevent massive over-rendering.
