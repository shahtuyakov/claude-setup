# SwiftUI Patterns

## Basic View Structure

```swift
import SwiftUI

struct UserCardView: View {
    let user: User

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: user.avatarURL) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Circle()
                    .fill(.gray.opacity(0.3))
            }
            .frame(width: 50, height: 50)
            .clipShape(Circle())

            VStack(alignment: .leading, spacing: 4) {
                Text(user.name)
                    .font(.headline)
                Text(user.email)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
        .background(.background, in: RoundedRectangle(cornerRadius: 12))
    }
}
```

## View with ViewModel (iOS 17+)

```swift
struct UserListView: View {
    @State private var viewModel = UserListViewModel()

    var body: some View {
        List {
            ForEach(viewModel.users) { user in
                UserCardView(user: user)
            }
        }
        .overlay {
            if viewModel.isLoading {
                ProgressView()
            }
        }
        .overlay {
            if viewModel.users.isEmpty && !viewModel.isLoading {
                ContentUnavailableView(
                    "No Users",
                    systemImage: "person.3",
                    description: Text("Users will appear here")
                )
            }
        }
        .task {
            await viewModel.loadUsers()
        }
        .refreshable {
            await viewModel.loadUsers()
        }
    }
}
```

## Loading, Error, Empty States

```swift
struct AsyncContentView<Content: View, T>: View {
    let phase: LoadingPhase<T>
    @ViewBuilder let content: (T) -> Content

    var body: some View {
        switch phase {
        case .idle:
            Color.clear
        case .loading:
            ProgressView()
        case .success(let data):
            content(data)
        case .failure(let error):
            ContentUnavailableView(
                "Error",
                systemImage: "exclamationmark.triangle",
                description: Text(error.localizedDescription)
            )
        case .empty:
            ContentUnavailableView.search
        }
    }
}

enum LoadingPhase<T> {
    case idle
    case loading
    case success(T)
    case failure(Error)
    case empty
}
```

## Reusable Components

### Primary Button

```swift
struct PrimaryButton: View {
    let title: String
    let isLoading: Bool
    let action: () -> Void

    init(_ title: String, isLoading: Bool = false, action: @escaping () -> Void) {
        self.title = title
        self.isLoading = isLoading
        self.action = action
    }

    var body: some View {
        Button(action: action) {
            Group {
                if isLoading {
                    ProgressView()
                        .tint(.white)
                } else {
                    Text(title)
                }
            }
            .frame(maxWidth: .infinity)
            .frame(height: 50)
        }
        .buttonStyle(.borderedProminent)
        .disabled(isLoading)
    }
}
```

### Form Field

```swift
struct FormField: View {
    let title: String
    @Binding var text: String
    var error: String?
    var isSecure: Bool = false

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(title)
                .font(.subheadline)
                .foregroundStyle(.secondary)

            Group {
                if isSecure {
                    SecureField("", text: $text)
                } else {
                    TextField("", text: $text)
                }
            }
            .textFieldStyle(.roundedBorder)

            if let error {
                Text(error)
                    .font(.caption)
                    .foregroundStyle(.red)
            }
        }
    }
}
```

## Environment Values

### Custom Environment Key

```swift
// Define
private struct APIClientKey: EnvironmentKey {
    static let defaultValue: APIClient = .shared
}

extension EnvironmentValues {
    var apiClient: APIClient {
        get { self[APIClientKey.self] }
        set { self[APIClientKey.self] = newValue }
    }
}

// Inject
ContentView()
    .environment(\.apiClient, APIClient.shared)

// Use
struct SomeView: View {
    @Environment(\.apiClient) private var apiClient

    var body: some View {
        // Use apiClient
    }
}
```

### Common Environment Values

```swift
@Environment(\.dismiss) private var dismiss
@Environment(\.colorScheme) private var colorScheme
@Environment(\.horizontalSizeClass) private var sizeClass
@Environment(\.dynamicTypeSize) private var dynamicTypeSize
@Environment(\.modelContext) private var modelContext  // SwiftData
```

## View Modifiers

### Custom Modifier

```swift
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.background)
            .clipShape(RoundedRectangle(cornerRadius: 12))
            .shadow(color: .black.opacity(0.1), radius: 5, y: 2)
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}

// Usage
Text("Hello")
    .cardStyle()
```

### Loading Overlay

```swift
extension View {
    func loading(_ isLoading: Bool) -> some View {
        overlay {
            if isLoading {
                ZStack {
                    Color.black.opacity(0.3)
                    ProgressView()
                        .tint(.white)
                }
                .ignoresSafeArea()
            }
        }
    }
}
```

## Lists and Grids

### List with Swipe Actions

```swift
List {
    ForEach(items) { item in
        ItemRow(item: item)
            .swipeActions(edge: .trailing) {
                Button(role: .destructive) {
                    deleteItem(item)
                } label: {
                    Label("Delete", systemImage: "trash")
                }

                Button {
                    editItem(item)
                } label: {
                    Label("Edit", systemImage: "pencil")
                }
                .tint(.orange)
            }
    }
}
```

### Lazy Grid

```swift
let columns = [
    GridItem(.adaptive(minimum: 150), spacing: 16)
]

ScrollView {
    LazyVGrid(columns: columns, spacing: 16) {
        ForEach(items) { item in
            ItemCard(item: item)
        }
    }
    .padding()
}
```

## Sheets and Alerts

### Sheet Presentation

```swift
struct ParentView: View {
    @State private var showSheet = false
    @State private var selectedItem: Item?

    var body: some View {
        Button("Show Sheet") {
            showSheet = true
        }
        .sheet(isPresented: $showSheet) {
            SheetContent()
                .presentationDetents([.medium, .large])
                .presentationDragIndicator(.visible)
        }
        .sheet(item: $selectedItem) { item in
            ItemDetail(item: item)
        }
    }
}
```

### Confirmation Dialog

```swift
.confirmationDialog(
    "Are you sure?",
    isPresented: $showConfirmation,
    titleVisibility: .visible
) {
    Button("Delete", role: .destructive) {
        deleteItem()
    }
    Button("Cancel", role: .cancel) {}
}
```

### Alert

```swift
.alert("Error", isPresented: $showError) {
    Button("OK", role: .cancel) {}
    Button("Retry") {
        retry()
    }
} message: {
    Text(errorMessage)
}
```

## Search

```swift
struct SearchableListView: View {
    @State private var searchText = ""
    @State private var items: [Item] = []

    var filteredItems: [Item] {
        if searchText.isEmpty {
            return items
        }
        return items.filter { $0.name.localizedCaseInsensitiveContains(searchText) }
    }

    var body: some View {
        List(filteredItems) { item in
            Text(item.name)
        }
        .searchable(text: $searchText, prompt: "Search items")
        .searchSuggestions {
            ForEach(suggestions, id: \.self) { suggestion in
                Text(suggestion)
                    .searchCompletion(suggestion)
            }
        }
    }
}
```

## Accessibility

```swift
Image(systemName: "heart.fill")
    .accessibilityLabel("Favorite")

Button(action: toggleFavorite) {
    Image(systemName: isFavorite ? "heart.fill" : "heart")
}
.accessibilityLabel(isFavorite ? "Remove from favorites" : "Add to favorites")
.accessibilityHint("Double tap to toggle")

VStack {
    Text("Price")
    Text("$99.99")
}
.accessibilityElement(children: .combine)
```

## Animations

```swift
// Implicit
Text("Hello")
    .scaleEffect(isExpanded ? 1.2 : 1.0)
    .animation(.spring, value: isExpanded)

// Explicit
withAnimation(.easeInOut(duration: 0.3)) {
    isExpanded.toggle()
}

// Transition
if showDetail {
    DetailView()
        .transition(.move(edge: .bottom).combined(with: .opacity))
}

// Matched Geometry
@Namespace private var animation

// Source
Image(item.image)
    .matchedGeometryEffect(id: item.id, in: animation)

// Destination
Image(selectedItem.image)
    .matchedGeometryEffect(id: selectedItem.id, in: animation)
```

## Preview Providers

```swift
#Preview {
    UserCardView(user: .preview)
}

#Preview("Dark Mode") {
    UserCardView(user: .preview)
        .preferredColorScheme(.dark)
}

#Preview("Large Text") {
    UserCardView(user: .preview)
        .dynamicTypeSize(.xxxLarge)
}

// Preview data
extension User {
    static var preview: User {
        User(name: "John Doe", email: "john@example.com")
    }
}
```
