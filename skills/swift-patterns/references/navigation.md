# Navigation

## NavigationStack (iOS 16+)

### Basic Navigation

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            List(items) { item in
                NavigationLink(value: item) {
                    Text(item.name)
                }
            }
            .navigationDestination(for: Item.self) { item in
                ItemDetailView(item: item)
            }
            .navigationTitle("Items")
        }
    }
}
```

### Programmatic Navigation with Path

```swift
struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            List(items) { item in
                NavigationLink(value: item) {
                    Text(item.name)
                }
            }
            .navigationDestination(for: Item.self) { item in
                ItemDetailView(item: item)
            }
            .navigationDestination(for: User.self) { user in
                UserProfileView(user: user)
            }
        }
    }

    func navigateToItem(_ item: Item) {
        path.append(item)
    }

    func navigateToUser(_ user: User) {
        path.append(user)
    }

    func popToRoot() {
        path.removeLast(path.count)
    }

    func pop() {
        path.removeLast()
    }
}
```

### Multiple Destination Types

```swift
enum Destination: Hashable {
    case itemDetail(Item)
    case userProfile(User)
    case settings
}

struct ContentView: View {
    @State private var path: [Destination] = []

    var body: some View {
        NavigationStack(path: $path) {
            HomeView(path: $path)
                .navigationDestination(for: Destination.self) { destination in
                    switch destination {
                    case .itemDetail(let item):
                        ItemDetailView(item: item)
                    case .userProfile(let user):
                        UserProfileView(user: user)
                    case .settings:
                        SettingsView()
                    }
                }
        }
    }
}
```

## Coordinator Pattern

### Router Definition

```swift
// Router.swift
@Observable
class Router {
    var path = NavigationPath()

    enum Destination: Hashable {
        case itemDetail(Item)
        case userProfile(User)
        case settings
        case editProfile
    }

    func navigate(to destination: Destination) {
        path.append(destination)
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func popToRoot() {
        path.removeLast(path.count)
    }

    @ViewBuilder
    func view(for destination: Destination) -> some View {
        switch destination {
        case .itemDetail(let item):
            ItemDetailView(item: item)
        case .userProfile(let user):
            UserProfileView(user: user)
        case .settings:
            SettingsView()
        case .editProfile:
            EditProfileView()
        }
    }
}
```

### Router Integration

```swift
struct RootView: View {
    @State private var router = Router()

    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: Router.Destination.self) { destination in
                    router.view(for: destination)
                }
        }
        .environment(router)
    }
}

struct HomeView: View {
    @Environment(Router.self) private var router

    var body: some View {
        List {
            Button("Go to Settings") {
                router.navigate(to: .settings)
            }

            ForEach(items) { item in
                Button(item.name) {
                    router.navigate(to: .itemDetail(item))
                }
            }
        }
    }
}
```

### Deep Linking Support

```swift
@Observable
class Router {
    var path = NavigationPath()

    func handleDeepLink(_ url: URL) {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: true),
              let host = components.host else { return }

        // Clear existing path
        popToRoot()

        switch host {
        case "item":
            if let id = components.queryItems?.first(where: { $0.name == "id" })?.value {
                // Fetch item and navigate
                Task {
                    if let item = await fetchItem(id: id) {
                        navigate(to: .itemDetail(item))
                    }
                }
            }
        case "profile":
            navigate(to: .editProfile)
        case "settings":
            navigate(to: .settings)
        default:
            break
        }
    }
}

// In App
@main
struct MyApp: App {
    @State private var router = Router()

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(router)
                .onOpenURL { url in
                    router.handleDeepLink(url)
                }
        }
    }
}
```

## TabView

### Basic TabView

```swift
struct MainTabView: View {
    @State private var selectedTab = Tab.home

    enum Tab {
        case home, search, profile
    }

    var body: some View {
        TabView(selection: $selectedTab) {
            HomeView()
                .tabItem {
                    Label("Home", systemImage: "house")
                }
                .tag(Tab.home)

            SearchView()
                .tabItem {
                    Label("Search", systemImage: "magnifyingglass")
                }
                .tag(Tab.search)

            ProfileView()
                .tabItem {
                    Label("Profile", systemImage: "person")
                }
                .tag(Tab.profile)
        }
    }
}
```

### TabView with NavigationStack per Tab

```swift
struct MainTabView: View {
    @State private var selectedTab = Tab.home
    @State private var homeRouter = Router()
    @State private var searchRouter = Router()
    @State private var profileRouter = Router()

    var body: some View {
        TabView(selection: $selectedTab) {
            NavigationStack(path: $homeRouter.path) {
                HomeView()
                    .navigationDestination(for: Router.Destination.self) { dest in
                        homeRouter.view(for: dest)
                    }
            }
            .environment(homeRouter)
            .tabItem { Label("Home", systemImage: "house") }
            .tag(Tab.home)

            NavigationStack(path: $searchRouter.path) {
                SearchView()
                    .navigationDestination(for: Router.Destination.self) { dest in
                        searchRouter.view(for: dest)
                    }
            }
            .environment(searchRouter)
            .tabItem { Label("Search", systemImage: "magnifyingglass") }
            .tag(Tab.search)

            NavigationStack(path: $profileRouter.path) {
                ProfileView()
                    .navigationDestination(for: Router.Destination.self) { dest in
                        profileRouter.view(for: dest)
                    }
            }
            .environment(profileRouter)
            .tabItem { Label("Profile", systemImage: "person") }
            .tag(Tab.profile)
        }
    }
}
```

## Sheets & Full Screen Covers

### Sheet Presentation

```swift
struct ContentView: View {
    @State private var showSheet = false
    @State private var selectedItem: Item?

    var body: some View {
        List(items) { item in
            Button(item.name) {
                selectedItem = item
            }
        }
        .sheet(item: $selectedItem) { item in
            ItemDetailSheet(item: item)
                .presentationDetents([.medium, .large])
                .presentationDragIndicator(.visible)
        }
        .sheet(isPresented: $showSheet) {
            SettingsSheet()
        }
    }
}
```

### Full Screen Cover

```swift
struct ContentView: View {
    @State private var showOnboarding = true

    var body: some View {
        MainView()
            .fullScreenCover(isPresented: $showOnboarding) {
                OnboardingView(isPresented: $showOnboarding)
            }
    }
}
```

### Dismiss from Presented View

```swift
struct DetailSheet: View {
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                // Content
            }
            .navigationTitle("Detail")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") {
                        dismiss()
                    }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Save") {
                        save()
                        dismiss()
                    }
                }
            }
        }
    }
}
```

## NavigationSplitView (iPad/Mac)

```swift
struct SplitView: View {
    @State private var selectedItem: Item?
    @State private var columnVisibility = NavigationSplitViewVisibility.all

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            // Sidebar
            List(categories, selection: $selectedCategory) { category in
                Label(category.name, systemImage: category.icon)
            }
            .navigationTitle("Categories")
        } content: {
            // Content
            List(items, selection: $selectedItem) { item in
                Text(item.name)
            }
            .navigationTitle("Items")
        } detail: {
            // Detail
            if let item = selectedItem {
                ItemDetailView(item: item)
            } else {
                ContentUnavailableView(
                    "Select an Item",
                    systemImage: "square.dashed"
                )
            }
        }
        .navigationSplitViewStyle(.balanced)
    }
}
```

## Alert & Confirmation

```swift
struct ContentView: View {
    @State private var showDeleteAlert = false
    @State private var itemToDelete: Item?

    var body: some View {
        List {
            ForEach(items) { item in
                Text(item.name)
                    .swipeActions {
                        Button(role: .destructive) {
                            itemToDelete = item
                            showDeleteAlert = true
                        } label: {
                            Label("Delete", systemImage: "trash")
                        }
                    }
            }
        }
        .alert("Delete Item?", isPresented: $showDeleteAlert) {
            Button("Cancel", role: .cancel) {}
            Button("Delete", role: .destructive) {
                if let item = itemToDelete {
                    delete(item)
                }
            }
        } message: {
            Text("This action cannot be undone.")
        }
    }
}
```

## Navigation State Persistence

```swift
// Save navigation state
@Observable
class Router {
    var path = NavigationPath()

    private let pathKey = "navigationPath"

    func save() {
        guard let data = try? JSONEncoder().encode(path.codable) else { return }
        UserDefaults.standard.set(data, forKey: pathKey)
    }

    func restore() {
        guard let data = UserDefaults.standard.data(forKey: pathKey),
              let decoded = try? JSONDecoder().decode(
                NavigationPath.CodableRepresentation.self,
                from: data
              ) else { return }
        path = NavigationPath(decoded)
    }
}
```

## Best Practices

1. **Use NavigationStack** - Not deprecated NavigationView
2. **Value-based navigation** - NavigationLink(value:) over destination closures
3. **Coordinator for complex flows** - Centralize navigation logic
4. **One NavigationStack per tab** - Avoid nested stacks
5. **Handle deep links** - Support URL-based navigation
6. **Persist navigation state** - For app restoration
