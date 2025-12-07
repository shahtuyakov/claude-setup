---
name: swift-patterns
description: Swift and SwiftUI development patterns for building native iOS/iPadOS applications. Use when implementing views, view models, data persistence, networking, concurrency, and navigation. Covers Swift 6.2, SwiftUI, SwiftData, iOS 26 features including Liquid Glass and Foundation Models.
---

# Swift Patterns

Modern Swift and SwiftUI patterns for iOS development.

## iOS Version Selection

| Target | Best For | Stack |
|--------|----------|-------|
| iOS 26+ | New apps, latest features | SwiftUI + SwiftData + @Observable + Foundation Models |
| iOS 17+ | Broad compatibility with modern APIs | SwiftUI + SwiftData + @Observable |
| iOS 15+ | Maximum device coverage | SwiftUI + Core Data + ObservableObject |

## Reference Files

| Topic | Load | Use When |
|-------|------|----------|
| Project structure | `references/project-structure.md` | Setting up or organizing code |
| SwiftUI views | `references/swiftui-patterns.md` | Building UI components |
| State management | `references/state-management.md` | @Observable, ObservableObject, @State |
| Data persistence | `references/data-persistence.md` | SwiftData, Core Data, Keychain |
| Networking | `references/networking.md` | URLSession, API clients |
| Concurrency | `references/concurrency.md` | async/await, actors, tasks |
| Navigation | `references/navigation.md` | NavigationStack, coordinators |
| iOS 26 features | `references/ios26-features.md` | Liquid Glass, Foundation Models |

## Quick Start Patterns

### Basic SwiftUI View (iOS 17+)

```swift
import SwiftUI

struct UserProfileView: View {
    @State private var viewModel = UserProfileViewModel()

    var body: some View {
        Group {
            if viewModel.isLoading {
                ProgressView()
            } else if let user = viewModel.user {
                VStack {
                    Text(user.name)
                        .font(.title)
                    Text(user.email)
                        .foregroundStyle(.secondary)
                }
            } else if let error = viewModel.error {
                ContentUnavailableView(
                    "Error",
                    systemImage: "exclamationmark.triangle",
                    description: Text(error.localizedDescription)
                )
            }
        }
        .task {
            await viewModel.loadUser()
        }
    }
}
```

### Observable ViewModel (iOS 17+)

```swift
import Foundation

@Observable
class UserProfileViewModel {
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

### SwiftData Model (iOS 17+)

```swift
import SwiftData

@Model
class User {
    var id: UUID
    var name: String
    var email: String
    var createdAt: Date

    init(id: UUID = UUID(), name: String, email: String) {
        self.id = id
        self.name = name
        self.email = email
        self.createdAt = Date()
    }
}
```

### Async API Call

```swift
func fetchUsers() async throws -> [User] {
    let url = URL(string: "https://api.example.com/users")!
    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 200 else {
        throw APIError.invalidResponse
    }

    return try JSONDecoder().decode([User].self, from: data)
}
```

## Architecture Patterns

| Pattern | Best For | Complexity |
|---------|----------|------------|
| MVVM | Most apps | Medium |
| TCA | Large teams, complex state | High |
| Coordinator | Complex navigation | Medium |

## Essential Packages

| Package | Purpose |
|---------|---------|
| SwiftData | Persistence (iOS 17+) |
| SwiftUI | UI framework |
| Foundation | Networking, dates, etc. |
| Swift Testing | Modern testing |
| KeychainAccess | Secure storage |

## Swift 6.2 Key Features

1. **Approachable Concurrency** - Main thread by default
2. **@concurrent** - Explicit background execution
3. **Typed throws** - Better error handling
4. **@Animatable macro** - Simplified animations

## Code Quality Rules

1. **@Observable over ObservableObject** - For iOS 17+
2. **Structured concurrency** - Use async/await, not callbacks
3. **SwiftData over Core Data** - For iOS 17+ new projects
4. **Swift Testing over XCTest** - For new test code
5. **SPM over CocoaPods** - For dependency management
6. **Value types** - Prefer structs over classes when possible
