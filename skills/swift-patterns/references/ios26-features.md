# iOS 26 Features

## Liquid Glass Design System

### Automatic Adoption

When you compile with Xcode 26, your app automatically adopts Liquid Glass:
- TabView gets new liquid tab bar
- NavigationSplitView becomes liquid glass sidebar
- Toolbars float with glass effect
- Sheets have inset liquid glass background

### Opting Out (Grace Period)

```xml
<!-- Info.plist -->
<key>UIDesignRequiresCompatibility</key>
<true/>
```

This disables Liquid Glass for one year.

### Custom Glass Effects

```swift
// Apply glass effect to any view
Text("Hello")
    .glassEffect(.regular, in: RoundedRectangle(cornerRadius: 12))

// Tinted glass
Button("Action") { }
    .buttonStyle(.borderedProminent)
    .tint(.blue)  // Tints the glass
```

### Toolbar Enhancements

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            ScrollView {
                // Content
            }
            .toolbar {
                // New: ToolbarSpacer for grouping
                ToolbarItem(placement: .primaryAction) {
                    Button("Save", systemImage: "checkmark") { }
                }

                ToolbarSpacer(.fixed)

                ToolbarItem(placement: .primaryAction) {
                    Button("Share", systemImage: "square.and.arrow.up") { }
                }
            }
            // New: Scroll edge effect
            .toolbarScrollEdgeEffect(.automatic)
        }
    }
}
```

## Foundation Models (On-Device AI)

### Basic Text Generation

```swift
import FoundationModels

// Check availability
guard LanguageModelSession.isAvailable else {
    print("Foundation Models not available")
    return
}

// Create session
let session = LanguageModelSession()

// Generate text
let response = try await session.respond(to: "Summarize this article: \(articleText)")
print(response.content)
```

### Streaming Response

```swift
let session = LanguageModelSession()

for try await partial in session.streamResponse(to: prompt) {
    print(partial.content, terminator: "")
}
```

### Structured Output with @Generable

```swift
import FoundationModels

@Generable
struct MovieReview {
    @Guide("A rating from 1 to 5 stars")
    var rating: Int

    @Guide("A brief summary of the review sentiment")
    var summary: String

    @Guide("Key points mentioned in the review")
    var keyPoints: [String]
}

// Generate structured data
let session = LanguageModelSession()
let review: MovieReview = try await session.respond(
    to: "Analyze this movie review: \(reviewText)"
)

print("Rating: \(review.rating)/5")
print("Summary: \(review.summary)")
```

### Specialized Adapters

```swift
// Use specialized model for content tagging
let session = LanguageModelSession(
    model: SystemLanguageModel(useCase: .contentTagging)
)

@Generable
struct ContentTags {
    var category: String
    var tags: [String]
    var sentiment: String
}

let tags: ContentTags = try await session.respond(to: content)
```

### Using Tools

```swift
import FoundationModels

struct WeatherTool: Tool {
    static let name = "getWeather"
    static let description = "Get current weather for a location"

    struct Input: Codable {
        let location: String
    }

    struct Output: Codable {
        let temperature: Int
        let condition: String
    }

    func call(input: Input) async throws -> Output {
        // Fetch actual weather
        let weather = try await WeatherService.fetch(location: input.location)
        return Output(temperature: weather.temp, condition: weather.condition)
    }
}

let session = LanguageModelSession(tools: [WeatherTool()])
let response = try await session.respond(to: "What's the weather in San Francisco?")
```

## SwiftUI iOS 26 Updates

### Native WebView

```swift
import SwiftUI
import WebKit

struct WebContentView: View {
    var body: some View {
        WebView(url: URL(string: "https://apple.com")!)
            .ignoresSafeArea()
    }
}
```

### Rich Text Editing

```swift
struct RichTextEditor: View {
    @State private var attributedText = AttributedString("Hello, World!")

    var body: some View {
        TextEditor(text: $attributedText)
            .textEditorStyle(.richText)  // New in iOS 26
    }
}
```

### @Animatable Macro

```swift
// Before iOS 26
struct OldAnimatableView: View, Animatable {
    var progress: Double

    var animatableData: Double {
        get { progress }
        set { progress = newValue }
    }

    var body: some View {
        // ...
    }
}

// iOS 26 with @Animatable
@Animatable
struct AnimatedProgressView: View {
    var progress: Double  // Automatically animatable

    var body: some View {
        Circle()
            .trim(from: 0, to: progress)
            .stroke(lineWidth: 4)
    }
}
```

### Chart3D

```swift
import Charts

struct Sales3DChart: View {
    var body: some View {
        Chart3D(data) { item in
            BarMark3D(
                x: .value("Month", item.month),
                y: .value("Product", item.product),
                z: .value("Sales", item.sales)
            )
            .foregroundStyle(by: .value("Category", item.category))
        }
        .chartRotation3D(angle: .degrees(45))
    }
}
```

### Background Extension Effect

```swift
Image("hero")
    .resizable()
    .aspectRatio(contentMode: .fill)
    .backgroundExtensionEffect()  // Extends and blurs beyond bounds
```

### Label Spacing

```swift
Label("Settings", systemImage: "gear")
    .labelIconToTitleSpacing(12)  // New in iOS 26
```

## SwiftData iOS 26

### Model Inheritance

```swift
import SwiftData

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
    var fuelType: FuelType

    init(brand: String, year: Int, doors: Int, fuel: FuelType) {
        self.numberOfDoors = doors
        self.fuelType = fuel
        super.init(brand: brand, year: year)
    }
}

@Model
class Motorcycle: Vehicle {
    var engineCC: Int

    init(brand: String, year: Int, engineCC: Int) {
        self.engineCC = engineCC
        super.init(brand: brand, year: year)
    }
}

// Query all vehicles (includes cars and motorcycles)
@Query var allVehicles: [Vehicle]

// Query only cars
@Query var cars: [Car]
```

### Enum in Predicates (Fixed)

```swift
enum Status: String, Codable {
    case pending, approved, rejected
}

@Model
class Request {
    var status: Status
    // ...
}

// Now works in iOS 26 (was broken before)
@Query(filter: #Predicate<Request> { $0.status == .approved })
var approvedRequests: [Request]
```

## UIKit iOS 26

### Swift Observable Support

```swift
// Automatic tracking in UIKit (iOS 26)
class MyViewController: UIViewController {
    var viewModel = MyViewModel()  // @Observable class

    override func viewDidLoad() {
        super.viewDidLoad()
    }

    // Called automatically when observed properties change
    override func updateProperties() {
        titleLabel.text = viewModel.title
        subtitleLabel.text = viewModel.subtitle
    }
}

// Back-deploy to iOS 18
// Add to Info.plist:
// UIObservationTrackingEnabled = YES
```

### New updateProperties() Lifecycle

```swift
class CustomView: UIView {
    var configuration: Configuration? {
        didSet { setNeedsUpdateProperties() }
    }

    override func updateProperties() {
        super.updateProperties()
        // Update view based on configuration
        backgroundColor = configuration?.backgroundColor
        layer.cornerRadius = configuration?.cornerRadius ?? 0
    }
}
```

### Fluid Animations

```swift
// Interruptible animations with flushUpdates
UIView.animate(withDuration: 0.3, options: [.flushUpdates]) {
    // Pending layout updates are flushed before animation
    self.view.frame = newFrame
}
```

### iPad Menu Bar

```swift
// Menu bar support (macOS-style on iPad)
override func buildMenu(with builder: UIMenuBuilder) {
    super.buildMenu(with: builder)

    let customMenu = UIMenu(
        title: "Custom",
        children: [
            UIAction(title: "Action 1") { _ in },
            UIAction(title: "Action 2") { _ in }
        ]
    )

    builder.insertSibling(customMenu, afterMenu: .file)
}
```

## Other APIs

### Declared Age Range

```swift
import DeclaredAgeRange

// Check user's declared age range
let ageRange = DeclaredAgeRange.shared

switch ageRange.category {
case .child:
    showChildFriendlyContent()
case .teen:
    showTeenContent()
case .adult:
    showAllContent()
case .unknown:
    requestAgeVerification()
}
```

### Call Translation

```swift
import CallTranslation

// Enable live translation in communication apps
let translator = CallTranslator()
translator.start(sourceLanguage: .english, targetLanguage: .spanish)

// Receive translated text
translator.onTranslation = { translatedText in
    displaySubtitle(translatedText)
}
```

## Migration Checklist

1. **Recompile with Xcode 26** - Get Liquid Glass automatically
2. **Test UI thoroughly** - New design may affect layouts
3. **Adopt Foundation Models** - For AI features
4. **Update SwiftData models** - Use inheritance if needed
5. **Enable UIObservationTracking** - For UIKit apps
6. **Test on iOS 26 devices** - Check for visual issues
7. **Plan for April 2026** - SDK requirement deadline
