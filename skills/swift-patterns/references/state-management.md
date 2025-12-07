# State Management

## When to Use What

| Property Wrapper | Use When | iOS Version |
|-----------------|----------|-------------|
| `@State` | View-local value types | All |
| `@Binding` | Child needs to mutate parent's state | All |
| `@Observable` class | Shared reference type, ViewModel | iOS 17+ |
| `@Environment` | Dependency injection | All |
| `@Bindable` | Bindings from @Observable | iOS 17+ |
| `ObservableObject` | Legacy shared state | iOS 13+ |
| `@StateObject` | Create ObservableObject | iOS 14+ |
| `@EnvironmentObject` | Inject ObservableObject | iOS 13+ |

## @Observable (iOS 17+ Recommended)

### Basic Observable Class

```swift
import Foundation

@Observable
class UserViewModel {
    var user: User?
    var isLoading = false
    var error: Error?

    func loadUser() async {
        isLoading = true
        defer { isLoading = false }

        do {
            user = try await APIClient.shared.fetchCurrentUser()
        } catch {
            self.error = error
        }
    }
}
```

### Using in View

```swift
struct UserProfileView: View {
    // Use @State for @Observable classes
    @State private var viewModel = UserViewModel()

    var body: some View {
        VStack {
            if viewModel.isLoading {
                ProgressView()
            } else if let user = viewModel.user {
                Text(user.name)
            }
        }
        .task {
            await viewModel.loadUser()
        }
    }
}
```

### Passing Observable to Child

```swift
struct ParentView: View {
    @State private var viewModel = UserViewModel()

    var body: some View {
        // Just pass directly - no wrapper needed
        ChildView(viewModel: viewModel)
    }
}

struct ChildView: View {
    var viewModel: UserViewModel  // No property wrapper

    var body: some View {
        Text(viewModel.user?.name ?? "")
    }
}
```

### @Bindable for Bindings

```swift
struct EditUserView: View {
    @Bindable var viewModel: UserViewModel

    var body: some View {
        // Create bindings to @Observable properties
        TextField("Name", text: $viewModel.userName)
    }
}

// Parent passes @Observable instance
EditUserView(viewModel: viewModel)
```

### Environment with @Observable

```swift
// Define
@Observable
class AppState {
    var isAuthenticated = false
    var currentUser: User?
}

// Inject in App
@main
struct MyApp: App {
    @State private var appState = AppState()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(appState)
        }
    }
}

// Use in any view
struct SomeView: View {
    @Environment(AppState.self) private var appState

    var body: some View {
        if appState.isAuthenticated {
            // Show authenticated content
        }
    }
}
```

## @State (Local Value State)

```swift
struct CounterView: View {
    @State private var count = 0
    @State private var user = User(name: "John")  // Value type

    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") {
                count += 1
            }
        }
    }
}
```

## @Binding (Two-Way Data Flow)

```swift
struct ToggleRow: View {
    let title: String
    @Binding var isOn: Bool

    var body: some View {
        Toggle(title, isOn: $isOn)
    }
}

// Parent
struct SettingsView: View {
    @State private var notificationsEnabled = true

    var body: some View {
        ToggleRow(
            title: "Notifications",
            isOn: $notificationsEnabled
        )
    }
}
```

## ObservableObject (Legacy iOS 13+)

### Definition

```swift
import Combine

class LegacyViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false
    @Published var error: Error?

    func loadItems() async {
        await MainActor.run { isLoading = true }
        defer { Task { @MainActor in isLoading = false } }

        do {
            let items = try await APIClient.shared.fetchItems()
            await MainActor.run { self.items = items }
        } catch {
            await MainActor.run { self.error = error }
        }
    }
}
```

### Usage with @StateObject

```swift
struct ItemListView: View {
    @StateObject private var viewModel = LegacyViewModel()

    var body: some View {
        List(viewModel.items) { item in
            Text(item.name)
        }
        .task {
            await viewModel.loadItems()
        }
    }
}
```

### @EnvironmentObject

```swift
// Inject
ContentView()
    .environmentObject(appState)

// Use
struct ChildView: View {
    @EnvironmentObject var appState: AppState

    var body: some View {
        Text(appState.userName)
    }
}
```

## Comparison: @Observable vs ObservableObject

| Feature | @Observable | ObservableObject |
|---------|-------------|------------------|
| iOS Version | 17+ | 13+ |
| Property annotation | None (automatic) | @Published |
| View wrapper | @State | @StateObject |
| Child view wrapper | None | @ObservedObject |
| Environment wrapper | @Environment | @EnvironmentObject |
| Performance | Better (fine-grained) | Worse (any change rerenders) |
| Combine integration | Manual | Built-in |

## App-Wide State Pattern

```swift
// AppState.swift
@Observable
class AppState {
    // Auth
    var isAuthenticated = false
    var currentUser: User?

    // UI
    var selectedTab: Tab = .home

    // Feature flags
    var featureFlags: FeatureFlags = .default

    func logout() {
        isAuthenticated = false
        currentUser = nil
    }
}

// App.swift
@main
struct MyApp: App {
    @State private var appState = AppState()

    var body: some Scene {
        WindowGroup {
            if appState.isAuthenticated {
                MainTabView()
                    .environment(appState)
            } else {
                LoginView()
                    .environment(appState)
            }
        }
    }
}
```

## Derived/Computed State

```swift
@Observable
class CartViewModel {
    var items: [CartItem] = []

    // Computed properties work automatically
    var totalItems: Int {
        items.reduce(0) { $0 + $1.quantity }
    }

    var totalPrice: Decimal {
        items.reduce(0) { $0 + ($1.price * Decimal($1.quantity)) }
    }

    var isEmpty: Bool {
        items.isEmpty
    }
}
```

## Best Practices

1. **Use @Observable for iOS 17+** - Simpler, better performance
2. **Keep ViewModels focused** - One ViewModel per feature/screen
3. **Avoid deep nesting** - Use environment for widely shared state
4. **Prefer value types** - Use structs for models when possible
5. **Isolate side effects** - Keep async work in ViewModels, not Views
6. **Use @Bindable sparingly** - Only when you need bindings to @Observable
