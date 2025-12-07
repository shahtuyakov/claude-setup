# Project Structure

## SPM-Based Modular Architecture (Recommended)

```
MyApp/
├── Package.swift                    # Root package manifest
├── App/
│   └── MyApp/
│       ├── MyAppApp.swift          # @main entry point
│       ├── ContentView.swift
│       └── Assets.xcassets
├── Sources/
│   ├── Features/                   # Feature modules
│   │   ├── Auth/
│   │   │   ├── Package.swift
│   │   │   └── Sources/
│   │   │       ├── Views/
│   │   │       │   ├── LoginView.swift
│   │   │       │   └── RegisterView.swift
│   │   │       ├── ViewModels/
│   │   │       │   └── AuthViewModel.swift
│   │   │       └── Models/
│   │   │           └── AuthState.swift
│   │   ├── Home/
│   │   │   └── ...
│   │   └── Profile/
│   │       └── ...
│   ├── Core/                       # Shared core modules
│   │   ├── Networking/
│   │   │   ├── Package.swift
│   │   │   └── Sources/
│   │   │       ├── APIClient.swift
│   │   │       ├── Endpoint.swift
│   │   │       └── APIError.swift
│   │   ├── Persistence/
│   │   │   └── Sources/
│   │   │       ├── DataController.swift
│   │   │       └── Models/
│   │   └── Common/
│   │       └── Sources/
│   │           ├── Extensions/
│   │           └── Utilities/
│   └── UI/                         # Shared UI components
│       └── Components/
│           └── Sources/
│               ├── Buttons/
│               ├── Cards/
│               └── Forms/
├── Tests/
│   ├── AuthTests/
│   ├── NetworkingTests/
│   └── ...
└── Resources/
    └── Localizable.xcstrings
```

## Simple SwiftUI App Structure

```
MyApp/
├── MyApp.xcodeproj
├── MyApp/
│   ├── MyAppApp.swift              # @main entry
│   ├── ContentView.swift
│   ├── Features/
│   │   ├── Auth/
│   │   │   ├── Views/
│   │   │   │   ├── LoginView.swift
│   │   │   │   └── RegisterView.swift
│   │   │   └── AuthViewModel.swift
│   │   ├── Home/
│   │   │   ├── HomeView.swift
│   │   │   └── HomeViewModel.swift
│   │   └── Profile/
│   │       └── ...
│   ├── Core/
│   │   ├── Networking/
│   │   │   ├── APIClient.swift
│   │   │   └── Endpoints.swift
│   │   ├── Persistence/
│   │   │   └── DataController.swift
│   │   └── Extensions/
│   │       └── View+Extensions.swift
│   ├── Models/
│   │   ├── User.swift
│   │   └── Post.swift
│   ├── Components/
│   │   ├── PrimaryButton.swift
│   │   └── LoadingView.swift
│   ├── Resources/
│   │   └── Localizable.xcstrings
│   └── Assets.xcassets
├── MyAppTests/
└── MyAppUITests/
```

## File Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Views | PascalCase + View | `UserProfileView.swift` |
| ViewModels | PascalCase + ViewModel | `UserProfileViewModel.swift` |
| Models | PascalCase | `User.swift` |
| Extensions | Type+Category | `View+Loading.swift` |
| Protocols | PascalCase + able/ing | `Loadable.swift` |
| Utilities | PascalCase | `DateFormatter.swift` |

## Package.swift Example

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "MyApp",
    platforms: [
        .iOS(.v17),
        .macOS(.v14)
    ],
    products: [
        .library(name: "Features", targets: ["Auth", "Home", "Profile"]),
        .library(name: "Core", targets: ["Networking", "Persistence", "Common"]),
        .library(name: "UI", targets: ["Components"]),
    ],
    dependencies: [
        .package(url: "https://github.com/kishikawakatsumi/KeychainAccess.git", from: "4.2.2"),
    ],
    targets: [
        // Feature modules
        .target(
            name: "Auth",
            dependencies: ["Networking", "Components", "Common"],
            path: "Sources/Features/Auth"
        ),
        .target(
            name: "Home",
            dependencies: ["Networking", "Components"],
            path: "Sources/Features/Home"
        ),
        .target(
            name: "Profile",
            dependencies: ["Networking", "Persistence", "Components"],
            path: "Sources/Features/Profile"
        ),

        // Core modules
        .target(
            name: "Networking",
            dependencies: ["Common"],
            path: "Sources/Core/Networking"
        ),
        .target(
            name: "Persistence",
            dependencies: ["Common"],
            path: "Sources/Core/Persistence"
        ),
        .target(
            name: "Common",
            path: "Sources/Core/Common"
        ),

        // UI module
        .target(
            name: "Components",
            dependencies: ["Common"],
            path: "Sources/UI/Components"
        ),

        // Tests
        .testTarget(
            name: "AuthTests",
            dependencies: ["Auth"]
        ),
        .testTarget(
            name: "NetworkingTests",
            dependencies: ["Networking"]
        ),
    ]
)
```

## Module Dependencies

```
┌─────────────────────────────────────────────────┐
│                      App                         │
├─────────────────────────────────────────────────┤
│   Auth    │    Home    │    Profile    │ ...    │  ← Features
├───────────┴────────────┴───────────────┴────────┤
│   Components   │   Networking   │   Persistence  │  ← Core/UI
├────────────────┴────────────────┴────────────────┤
│                     Common                        │  ← Utilities
└──────────────────────────────────────────────────┘
```

**Rules:**
- Features can depend on Core and UI
- Features should NOT depend on each other
- Core modules can depend on Common
- Common has no internal dependencies

## App Entry Point

```swift
// MyAppApp.swift
import SwiftUI
import SwiftData

@main
struct MyAppApp: App {
    var sharedModelContainer: ModelContainer = {
        let schema = Schema([User.self, Post.self])
        let config = ModelConfiguration(schema: schema, isStoredInMemoryOnly: false)
        return try! ModelContainer(for: schema, configurations: [config])
    }()

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(sharedModelContainer)
    }
}
```

## Environment Setup

```swift
// AppEnvironment.swift
enum AppEnvironment {
    case development
    case staging
    case production

    static var current: AppEnvironment {
        #if DEBUG
        return .development
        #else
        return .production
        #endif
    }

    var baseURL: URL {
        switch self {
        case .development:
            return URL(string: "https://dev-api.example.com")!
        case .staging:
            return URL(string: "https://staging-api.example.com")!
        case .production:
            return URL(string: "https://api.example.com")!
        }
    }
}
```

## Info.plist Keys

| Key | Purpose |
|-----|---------|
| `NSFaceIDUsageDescription` | Face ID permission |
| `NSCameraUsageDescription` | Camera permission |
| `NSPhotoLibraryUsageDescription` | Photos permission |
| `UIDesignRequiresCompatibility` | Disable Liquid Glass (iOS 26) |
| `UIObservationTrackingEnabled` | Enable Observable in UIKit (iOS 18) |

## Xcode Project Settings

| Setting | Recommended Value |
|---------|-------------------|
| iOS Deployment Target | 17.0+ |
| Swift Language Version | 6.0 |
| Strict Concurrency | Complete |
| Build Libraries for Distribution | Yes (for SPM) |
