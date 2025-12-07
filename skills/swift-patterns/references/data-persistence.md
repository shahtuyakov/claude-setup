# Data Persistence

## When to Use What

| Storage | Use When | iOS Version |
|---------|----------|-------------|
| SwiftData | Structured data, relationships | iOS 17+ |
| Core Data | Legacy apps, iOS 15 support | iOS 3+ |
| UserDefaults | Simple settings, preferences | All |
| Keychain | Sensitive data (tokens, passwords) | All |
| File System | Documents, large files, exports | All |

## SwiftData (iOS 17+ Recommended)

### Model Definition

```swift
import SwiftData

@Model
class User {
    var id: UUID
    var name: String
    var email: String
    var createdAt: Date

    // Relationships
    @Relationship(deleteRule: .cascade)
    var posts: [Post] = []

    init(name: String, email: String) {
        self.id = UUID()
        self.name = name
        self.email = email
        self.createdAt = Date()
    }
}

@Model
class Post {
    var id: UUID
    var title: String
    var content: String
    var createdAt: Date

    // Inverse relationship
    var author: User?

    init(title: String, content: String, author: User? = nil) {
        self.id = UUID()
        self.title = title
        self.content = content
        self.createdAt = Date()
        self.author = author
    }
}
```

### Model Inheritance (iOS 26+)

```swift
@Model
class Vehicle {
    var id: UUID
    var brand: String
    var year: Int

    init(brand: String, year: Int) {
        self.id = UUID()
        self.brand = brand
        self.year = year
    }
}

@Model
class Car: Vehicle {
    var numberOfDoors: Int

    init(brand: String, year: Int, numberOfDoors: Int) {
        self.numberOfDoors = numberOfDoors
        super.init(brand: brand, year: year)
    }
}
```

### Container Setup

```swift
import SwiftUI
import SwiftData

@main
struct MyApp: App {
    var sharedModelContainer: ModelContainer = {
        let schema = Schema([User.self, Post.self])
        let config = ModelConfiguration(
            schema: schema,
            isStoredInMemoryOnly: false,
            cloudKitDatabase: .none  // or .private for iCloud sync
        )
        do {
            return try ModelContainer(for: schema, configurations: [config])
        } catch {
            fatalError("Could not create ModelContainer: \(error)")
        }
    }()

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(sharedModelContainer)
    }
}
```

### CRUD Operations

```swift
struct UserListView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \User.createdAt, order: .reverse) private var users: [User]

    var body: some View {
        List {
            ForEach(users) { user in
                Text(user.name)
            }
            .onDelete(perform: deleteUsers)
        }
    }

    private func addUser() {
        let user = User(name: "New User", email: "new@example.com")
        modelContext.insert(user)
        // Auto-saves
    }

    private func deleteUsers(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(users[index])
        }
    }
}
```

### Query with Predicate

```swift
// Static predicate
@Query(filter: #Predicate<User> { $0.name.contains("John") })
private var johns: [User]

// Dynamic predicate
struct UserSearchView: View {
    @State private var searchText = ""

    var body: some View {
        UserList(searchText: searchText)
    }
}

struct UserList: View {
    @Query private var users: [User]

    init(searchText: String) {
        let predicate = #Predicate<User> {
            searchText.isEmpty || $0.name.localizedStandardContains(searchText)
        }
        _users = Query(filter: predicate, sort: \User.name)
    }

    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}
```

### Background Operations with @ModelActor

```swift
@ModelActor
actor DataManager {
    func importUsers(_ userData: [UserDTO]) throws {
        for dto in userData {
            let user = User(name: dto.name, email: dto.email)
            modelContext.insert(user)
        }
        try modelContext.save()
    }

    func fetchUserCount() throws -> Int {
        let descriptor = FetchDescriptor<User>()
        return try modelContext.fetchCount(descriptor)
    }
}

// Usage
let manager = DataManager(modelContainer: container)
try await manager.importUsers(userData)
```

## Core Data (Legacy iOS 15+)

### Model Setup

```swift
// DataController.swift
class DataController: ObservableObject {
    let container: NSPersistentContainer

    init() {
        container = NSPersistentContainer(name: "Model")
        container.loadPersistentStores { _, error in
            if let error {
                fatalError("Core Data failed: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }

    func save() {
        let context = container.viewContext
        if context.hasChanges {
            try? context.save()
        }
    }
}
```

### Usage in SwiftUI

```swift
@main
struct MyApp: App {
    @StateObject private var dataController = DataController()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.managedObjectContext, dataController.container.viewContext)
        }
    }
}

struct UserListView: View {
    @Environment(\.managedObjectContext) private var viewContext
    @FetchRequest(
        sortDescriptors: [NSSortDescriptor(keyPath: \User.createdAt, ascending: false)]
    ) private var users: FetchedResults<User>

    var body: some View {
        List(users) { user in
            Text(user.name ?? "")
        }
    }
}
```

## Keychain (Secure Storage)

### Using KeychainAccess Package

```swift
import KeychainAccess

class TokenManager {
    private let keychain = Keychain(service: "com.myapp.tokens")
        .accessibility(.whenUnlockedThisDeviceOnly)

    var accessToken: String? {
        get { try? keychain.get("accessToken") }
        set {
            if let value = newValue {
                try? keychain.set(value, key: "accessToken")
            } else {
                try? keychain.remove("accessToken")
            }
        }
    }

    var refreshToken: String? {
        get { try? keychain.get("refreshToken") }
        set {
            if let value = newValue {
                try? keychain.set(value, key: "refreshToken")
            } else {
                try? keychain.remove("refreshToken")
            }
        }
    }

    func clearAll() {
        try? keychain.removeAll()
    }
}
```

### Native Keychain

```swift
import Security

struct KeychainHelper {
    static func save(_ data: Data, service: String, account: String) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: data
        ]

        SecItemDelete(query as CFDictionary)
        return SecItemAdd(query as CFDictionary, nil) == errSecSuccess
    }

    static func read(service: String, account: String) -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)
        return result as? Data
    }

    static func delete(service: String, account: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account
        ]
        SecItemDelete(query as CFDictionary)
    }
}
```

## UserDefaults

### Type-Safe Wrapper

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    let container: UserDefaults = .standard

    var wrappedValue: T {
        get { container.object(forKey: key) as? T ?? defaultValue }
        set { container.set(newValue, forKey: key) }
    }
}

// Usage
enum Settings {
    @UserDefault(key: "hasCompletedOnboarding", defaultValue: false)
    static var hasCompletedOnboarding: Bool

    @UserDefault(key: "preferredTheme", defaultValue: "system")
    static var preferredTheme: String
}
```

### App Storage in SwiftUI

```swift
struct SettingsView: View {
    @AppStorage("notificationsEnabled") private var notificationsEnabled = true
    @AppStorage("fontSize") private var fontSize = 14.0

    var body: some View {
        Form {
            Toggle("Notifications", isOn: $notificationsEnabled)
            Slider(value: $fontSize, in: 12...24)
        }
    }
}
```

## File System

```swift
class FileStorageManager {
    private let fileManager = FileManager.default

    var documentsDirectory: URL {
        fileManager.urls(for: .documentDirectory, in: .userDomainMask)[0]
    }

    func save<T: Encodable>(_ object: T, filename: String) throws {
        let url = documentsDirectory.appendingPathComponent(filename)
        let data = try JSONEncoder().encode(object)
        try data.write(to: url)
    }

    func load<T: Decodable>(_ type: T.Type, filename: String) throws -> T {
        let url = documentsDirectory.appendingPathComponent(filename)
        let data = try Data(contentsOf: url)
        return try JSONDecoder().decode(type, from: data)
    }

    func delete(filename: String) throws {
        let url = documentsDirectory.appendingPathComponent(filename)
        try fileManager.removeItem(at: url)
    }
}
```

## SwiftData vs Core Data Decision

| Factor | Choose SwiftData | Choose Core Data |
|--------|------------------|------------------|
| iOS target | iOS 17+ only | iOS 15+ needed |
| New project | Yes | No |
| Complex migrations | No | Yes |
| Existing Core Data | Migrate gradually | Keep |
| CloudKit sync | Both work | More mature |
| Performance critical | Test first | Proven |

## Best Practices

1. **Use SwiftData for new iOS 17+ projects**
2. **Store sensitive data in Keychain** - Never UserDefaults
3. **Use @ModelActor for background work** - Avoid main thread blocking
4. **Test SwiftData on iOS 18** - Known issues exist
5. **Keep models simple** - Avoid complex computed properties in @Model
