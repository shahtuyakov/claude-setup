# Concurrency

## Swift 6.2 Approachable Concurrency

### Key Changes

| Before Swift 6.2 | Swift 6.2+ |
|------------------|------------|
| Explicit @MainActor required | Main thread by default |
| Async runs on global executor | Async runs in caller's context |
| Manual thread management | Use @concurrent for background |

### Default Behavior (Swift 6.2)

```swift
// This now runs on main thread by default
func loadData() async {
    let data = await fetchData()  // Runs in caller's context
    updateUI(with: data)          // Safe on main thread
}
```

### @concurrent for Background Work

```swift
// Explicitly run on background thread
@concurrent
func processLargeDataset(_ data: [Item]) async -> [ProcessedItem] {
    // Heavy computation runs off main thread
    return data.map { process($0) }
}

// Usage
let processed = await processLargeDataset(items)
// Back on main thread automatically
```

## Async/Await Basics

### Basic Async Function

```swift
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// Calling
Task {
    do {
        let user = try await fetchUser(id: "123")
        print(user.name)
    } catch {
        print(error)
    }
}
```

### Sequential vs Parallel

```swift
// Sequential - one after another
func loadSequentially() async throws -> (User, [Post]) {
    let user = try await fetchUser()      // Wait
    let posts = try await fetchPosts()    // Then wait
    return (user, posts)
}

// Parallel - concurrent execution
func loadInParallel() async throws -> (User, [Post]) {
    async let user = fetchUser()          // Start immediately
    async let posts = fetchPosts()        // Start immediately
    return try await (user, posts)        // Wait for both
}
```

### Task Groups

```swift
func fetchAllUsers(ids: [String]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask {
                try await fetchUser(id: id)
            }
        }

        var users: [User] = []
        for try await user in group {
            users.append(user)
        }
        return users
    }
}
```

## Tasks

### Creating Tasks

```swift
// Unstructured task (inherits actor context)
Task {
    await doSomething()
}

// Detached task (no inherited context)
Task.detached {
    await doBackgroundWork()
}

// Task with priority
Task(priority: .high) {
    await urgentWork()
}
```

### Task Cancellation

```swift
class ViewModel {
    private var loadTask: Task<Void, Never>?

    func startLoading() {
        loadTask = Task {
            do {
                // Check for cancellation periodically
                try Task.checkCancellation()

                let data = try await fetchData()

                // Check before UI update
                guard !Task.isCancelled else { return }
                updateUI(with: data)
            } catch is CancellationError {
                // Handle cancellation
            } catch {
                // Handle other errors
            }
        }
    }

    func stopLoading() {
        loadTask?.cancel()
        loadTask = nil
    }
}
```

### Task in SwiftUI

```swift
struct UserView: View {
    @State private var user: User?

    var body: some View {
        VStack {
            if let user {
                Text(user.name)
            }
        }
        .task {
            // Automatically cancelled when view disappears
            user = try? await fetchUser()
        }
        .task(id: userId) {
            // Re-runs when userId changes
            user = try? await fetchUser(id: userId)
        }
    }
}
```

## Actors

### Basic Actor

```swift
actor BankAccount {
    private var balance: Decimal = 0

    func deposit(_ amount: Decimal) {
        balance += amount
    }

    func withdraw(_ amount: Decimal) throws {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }
        balance -= amount
    }

    func getBalance() -> Decimal {
        balance
    }
}

// Usage (must await)
let account = BankAccount()
await account.deposit(100)
let balance = await account.getBalance()
```

### Global Actor

```swift
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
}

// Isolate to database actor
@DatabaseActor
class DatabaseManager {
    func save(_ data: Data) {
        // Always runs on DatabaseActor
    }
}

// Or individual functions
@DatabaseActor
func performDatabaseOperation() async {
    // Runs on DatabaseActor
}
```

### @MainActor

```swift
// Entire class on main actor
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    func loadItems() async {
        // Already on main actor
        let items = await fetchItems()
        self.items = items  // Safe UI update
    }
}

// Or individual functions
class Service {
    @MainActor
    func updateUI() {
        // Guaranteed main thread
    }
}
```

## Sendable

### Sendable Types

```swift
// Structs with Sendable properties are Sendable
struct User: Sendable {
    let id: UUID
    let name: String
}

// Classes must be final and immutable, or use @unchecked
final class ImmutableConfig: Sendable {
    let apiKey: String
    let baseURL: URL

    init(apiKey: String, baseURL: URL) {
        self.apiKey = apiKey
        self.baseURL = baseURL
    }
}

// Or use @unchecked with synchronization
final class ThreadSafeCache: @unchecked Sendable {
    private var cache: [String: Data] = [:]
    private let lock = NSLock()

    func get(_ key: String) -> Data? {
        lock.lock()
        defer { lock.unlock() }
        return cache[key]
    }
}
```

### Sendable Closures

```swift
// @Sendable closure can cross actor boundaries
func performAsync(_ operation: @Sendable @escaping () async -> Void) {
    Task {
        await operation()
    }
}
```

## AsyncSequence & AsyncStream

### Consuming AsyncSequence

```swift
// URLSession bytes
func downloadAndProcess(url: URL) async throws {
    let (bytes, _) = try await URLSession.shared.bytes(from: url)

    for try await byte in bytes {
        process(byte)
    }
}

// Notifications
func observeNotifications() async {
    let notifications = NotificationCenter.default.notifications(named: .userDidLogin)

    for await notification in notifications {
        handleLogin(notification)
    }
}
```

### Creating AsyncStream

```swift
func locationUpdates() -> AsyncStream<CLLocation> {
    AsyncStream { continuation in
        let manager = CLLocationManager()
        let delegate = LocationDelegate { location in
            continuation.yield(location)
        }

        manager.delegate = delegate
        manager.startUpdatingLocation()

        continuation.onTermination = { _ in
            manager.stopUpdatingLocation()
        }
    }
}

// Usage
for await location in locationUpdates() {
    updateMap(with: location)
}
```

### AsyncThrowingStream

```swift
func fetchPages() -> AsyncThrowingStream<Page, Error> {
    AsyncThrowingStream { continuation in
        Task {
            var page = 1
            while true {
                do {
                    let result = try await fetchPage(page)
                    if result.items.isEmpty {
                        continuation.finish()
                        break
                    }
                    continuation.yield(result)
                    page += 1
                } catch {
                    continuation.finish(throwing: error)
                    break
                }
            }
        }
    }
}
```

## Common Patterns

### Debounce

```swift
actor Debouncer {
    private var task: Task<Void, Never>?

    func debounce(delay: Duration, operation: @escaping @Sendable () async -> Void) {
        task?.cancel()
        task = Task {
            try? await Task.sleep(for: delay)
            guard !Task.isCancelled else { return }
            await operation()
        }
    }
}
```

### Timeout

```swift
func withTimeout<T>(
    seconds: Double,
    operation: @escaping @Sendable () async throws -> T
) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask {
            try await operation()
        }

        group.addTask {
            try await Task.sleep(for: .seconds(seconds))
            throw TimeoutError()
        }

        let result = try await group.next()!
        group.cancelAll()
        return result
    }
}
```

### Retry

```swift
func withRetry<T>(
    attempts: Int,
    delay: Duration = .seconds(1),
    operation: @escaping () async throws -> T
) async throws -> T {
    var lastError: Error?

    for attempt in 1...attempts {
        do {
            return try await operation()
        } catch {
            lastError = error
            if attempt < attempts {
                try await Task.sleep(for: delay)
            }
        }
    }

    throw lastError!
}
```

## Best Practices

1. **Use structured concurrency** - Prefer `async let` and task groups
2. **Cancel tasks properly** - Cancel in `deinit` or view disappear
3. **Check cancellation** - Use `Task.checkCancellation()` in long loops
4. **Use actors for shared state** - Not locks
5. **Mark closures @Sendable** - When crossing actor boundaries
6. **Use @MainActor for UI** - Not `DispatchQueue.main`
7. **Test concurrency** - Use async testing APIs
