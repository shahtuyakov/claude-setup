# iOS Patterns

## Quick Decision Matrix

| If you need... | Use | Avoid |
|----------------|-----|-------|
| UI framework | SwiftUI (iOS 14+) | UIKit for new projects |
| Architecture | MVVM with SwiftUI | MVC (doesn't scale) |
| Local persistence | Core Data / SwiftData | UserDefaults (for large data) |
| Networking | URLSession + async/await | Alamofire (unless complex needs) |
| Dependency injection | Native / Factory pattern | Heavy DI frameworks |
| Offline-first | Core Data + sync logic | Assuming always online |

---

## SwiftUI + MVVM Architecture

| Aspect | Details |
|--------|---------|
| **Best for** | Modern iOS apps, declarative UI, reactive updates |
| **Pros** | Less code, preview support, native Apple direction, Combine integration |
| **Cons** | iOS 14+ only, some UIKit features missing, learning curve |
| **When to use** | New projects, iOS 14+ target, modern Apple ecosystem |
| **When to avoid** | iOS 13 support needed, complex custom UI (consider UIKit) |

### MVVM Structure
```
App/
├── Models/           # Data models, Codable structs
├── Views/            # SwiftUI views
├── ViewModels/       # ObservableObject classes
├── Services/         # API, persistence, business logic
├── Repositories/     # Data access abstraction
└── Utilities/        # Extensions, helpers
```

### ViewModel Pattern
```swift
@MainActor
class UserViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var error: Error?

    private let repository: UserRepository

    func loadUsers() async {
        isLoading = true
        defer { isLoading = false }

        do {
            users = try await repository.getUsers()
        } catch {
            self.error = error
        }
    }
}
```

---

## Core Data / SwiftData

| Aspect | Details |
|--------|---------|
| **Best for** | Complex data models, relationships, offline-first |
| **Pros** | Apple native, powerful queries, iCloud sync option, migration support |
| **Cons** | Learning curve, boilerplate, can be complex |
| **When to use** | Offline-first apps, complex data relationships, large datasets |
| **When to avoid** | Simple key-value storage (use UserDefaults), very simple apps |

### Core Data vs SwiftData

| Feature | Core Data | SwiftData (iOS 17+) |
|---------|-----------|---------------------|
| Setup complexity | High | Low |
| Swift native | Partial | Full |
| Macro-based | No | Yes |
| Migration | Manual | Automatic (simple cases) |
| Maturity | Very mature | New |

### Offline-First Pattern
```
1. App loads → Fetch from Core Data (instant)
2. Background → API request for updates
3. API response → Update Core Data
4. Core Data change → UI updates automatically
5. No network → App works with cached data
6. Network returns → Sync pending changes
```

---

## Networking with async/await

| Aspect | Details |
|--------|---------|
| **Best for** | All network requests in modern iOS |
| **Pros** | Native, clean syntax, structured concurrency, cancellation |
| **Cons** | iOS 13+ (full support iOS 15+) |
| **When to use** | All new iOS networking code |

### API Client Pattern
```swift
class APIClient {
    private let session: URLSession
    private let baseURL: URL
    private let tokenProvider: TokenProvider

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        var request = endpoint.urlRequest(baseURL: baseURL)
        request.setValue("Bearer \(tokenProvider.accessToken)", forHTTPHeaderField: "Authorization")

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        switch httpResponse.statusCode {
        case 200...299:
            return try JSONDecoder().decode(T.self, from: data)
        case 401:
            // Trigger token refresh
            throw APIError.unauthorized
        default:
            throw APIError.serverError(httpResponse.statusCode)
        }
    }
}
```

### Token Refresh Pattern
```swift
func requestWithRefresh<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
    do {
        return try await request(endpoint)
    } catch APIError.unauthorized {
        try await tokenProvider.refreshToken()
        return try await request(endpoint)
    }
}
```

---

## Keychain Storage

| Store In | Data Type |
|----------|-----------|
| Keychain | Tokens, passwords, sensitive data |
| UserDefaults | Settings, preferences, flags |
| Core Data | Structured app data |
| File System | Documents, media, large files |

### Keychain Wrapper
```swift
class KeychainManager {
    static let shared = KeychainManager()

    func save(_ data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        SecItemDelete(query as CFDictionary)
        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else { throw KeychainError.saveFailed }
    }

    func load(for key: String) throws -> Data? {
        // Implementation
    }

    func delete(for key: String) throws {
        // Implementation
    }
}
```

---

## Dependency Injection

| Approach | Complexity | Best For |
|----------|------------|----------|
| Constructor injection | Low | Simple apps, clear dependencies |
| Environment objects | Low | SwiftUI view hierarchy |
| Factory pattern | Medium | Testable, scalable apps |
| Swinject/Resolver | High | Large enterprise apps |

### Factory Pattern (Recommended)
```swift
class DependencyContainer {
    static let shared = DependencyContainer()

    lazy var apiClient: APIClient = {
        APIClient(session: .shared, baseURL: Config.apiURL, tokenProvider: tokenProvider)
    }()

    lazy var tokenProvider: TokenProvider = {
        KeychainTokenProvider()
    }()

    lazy var userRepository: UserRepository = {
        UserRepositoryImpl(apiClient: apiClient, coreData: coreDataStack)
    }()
}
```

---

## App Lifecycle & Background

| State | Can Do | Typical Duration |
|-------|--------|------------------|
| Active | Everything | Unlimited |
| Background | Limited tasks | ~30 seconds |
| Suspended | Nothing | Until terminated |

### Background Tasks
```swift
// Request extended background time
func applicationDidEnterBackground(_ application: UIApplication) {
    let taskID = application.beginBackgroundTask {
        // Cleanup if time expires
    }

    // Do work

    application.endBackgroundTask(taskID)
}
```

### Background Refresh
```swift
// In AppDelegate
func application(_ application: UIApplication,
                 performFetchWithCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
    Task {
        do {
            let hasNewData = try await syncService.sync()
            completionHandler(hasNewData ? .newData : .noData)
        } catch {
            completionHandler(.failed)
        }
    }
}
```

---

## Error Handling Pattern

```swift
enum AppError: LocalizedError {
    case network(NetworkError)
    case database(DatabaseError)
    case auth(AuthError)
    case unknown(Error)

    var errorDescription: String? {
        switch self {
        case .network(let error): return error.localizedDescription
        case .database: return "Failed to save data"
        case .auth: return "Authentication failed"
        case .unknown: return "Something went wrong"
        }
    }
}

// In ViewModel
@Published var alertItem: AlertItem?

func handleError(_ error: Error) {
    let appError = error as? AppError ?? .unknown(error)
    alertItem = AlertItem(title: "Error", message: appError.localizedDescription)
}
```

---

## iOS Architecture Checklist

- [ ] SwiftUI + MVVM architecture
- [ ] Core Data / SwiftData for persistence
- [ ] async/await for networking
- [ ] Keychain for sensitive data
- [ ] Dependency injection pattern chosen
- [ ] Error handling strategy
- [ ] Offline-first capability
- [ ] Background refresh configured
- [ ] Token refresh flow implemented
- [ ] Biometric authentication (if needed)
- [ ] Push notifications setup
