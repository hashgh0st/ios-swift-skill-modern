# SwiftUI Patterns & Best Practices (iOS 26 / Xcode 26.4 / Swift 6.3)

## Liquid Glass (iOS 26+)

iOS 26 introduces Liquid Glass — the biggest design overhaul since iOS 7. It's a dynamic translucent material that refracts content below it, reflects light around it, and has a lensing effect along edges. It applies across iOS, iPadOS, macOS Tahoe, watchOS, tvOS, and visionOS.

### Automatic Adoption
Simply rebuilding with the Xcode 26 SDK gives you Liquid Glass on standard controls automatically — toolbars, tab bars, navigation bars, sheets, segmented pickers, toggles, and sliders all update without code changes.

### Custom Liquid Glass Surfaces
For custom views, use `.glassEffect()`:

```swift
Button("Action") { doSomething() }
    .glassEffect()                          // basic glass

Button("Tinted") { doSomething() }
    .glassEffect(.regular.tint(.purple))    // with color tint

Button("Interactive") { doSomething() }
    .glassEffect(.regular.interactive())    // scales and bounces on touch
```

### GlassEffectContainer — Morphing and Grouping
Wrap glass elements in `GlassEffectContainer` to blend overlapping shapes and enable morphing transitions:

```swift
GlassEffectContainer {
    HStack(spacing: 20) {
        Button("Home") { }
            .glassEffect()
        Button("Search") { }
            .glassEffect()
        Button("Profile") { }
            .glassEffect()
    }
}
```

Use `.glassEffectID(_:in:)` with a `@Namespace` for morphing animations between states.

### Liquid Glass Design Principles
- Glass belongs on the **navigation layer** — controls and chrome that float above content. Don't apply glass to content itself.
- Partial-height sheets are inset by default with a glass background in iOS 26. If you previously used `.presentationBackground()` to customize sheets, consider removing it to let the new material work.
- Toolbar items now float on a glass surface and are automatically grouped. Use `ToolbarSpacer` with fixed spacing to visually separate related action groups.
- Tab bars shrink on scroll and expand when scrolling back up.
- If you used custom `.presentationBackground()` on sheets, consider removing it to let the new Liquid Glass material shine.

### Layered App Icons
iOS 26 requires multi-layer icons for Liquid Glass rendering. Use Icon Composer in Xcode 26 to create layered icons from a single design — they render in light, dark, tinted, and the new "clear" appearance modes.

## New in SwiftUI for iOS 26

### Native WebView
SwiftUI now has a built-in `WebView` for displaying HTML/CSS/JS content. Associate it with a `WebPage` (`@Observable` class) for full navigation control:

```swift
struct BrowserView: View {
    @State private var page = WebPage()

    var body: some View {
        WebView(page)
            .onAppear { page.url = URL(string: "https://example.com") }
    }
}
```

### Rich Text TextEditor
`TextEditor` now supports `AttributedString` for rich text editing:

```swift
struct RichEditorView: View {
    @State private var text = AttributedString()

    var body: some View {
        TextEditor(text: $text)
    }
}
```

### @Animatable Macro
New macro that auto-synthesizes `Animatable` conformance — no more manual `animatableData` implementations:

```swift
@Animatable
struct PulseModifier: ViewModifier {
    var scale: CGFloat  // automatically animatable
    // ...
}
```

## Navigation

### NavigationStack (iOS 16+)

Always use `NavigationStack`, never the deprecated `NavigationView`.

```swift
struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            OrderListView()
                .navigationDestination(for: Order.self) { order in
                    OrderDetailView(order: order)
                }
                .navigationDestination(for: User.self) { user in
                    UserProfileView(user: user)
                }
        }
    }
}
```

Navigate programmatically by appending to the path:
```swift
path.append(selectedOrder)   // pushes OrderDetailView
path.removeLast()            // pops
path.removeLast(path.count)  // pops to root
```

For simple cases without programmatic navigation, omit the `path` binding:
```swift
NavigationStack {
    List(items) { item in
        NavigationLink(value: item) {
            ItemRow(item: item)
        }
    }
    .navigationDestination(for: Item.self) { item in
        ItemDetailView(item: item)
    }
}
```

### Tab Navigation

```swift
struct MainTabView: View {
    @State private var selectedTab: Tab = .home

    var body: some View {
        TabView(selection: $selectedTab) {
            HomeView()
                .tabItem { Label("Home", systemImage: "house") }
                .tag(Tab.home)
            SearchView()
                .tabItem { Label("Search", systemImage: "magnifyingglass") }
                .tag(Tab.search)
            ProfileView()
                .tabItem { Label("Profile", systemImage: "person") }
                .tag(Tab.profile)
        }
    }
}

enum Tab: Hashable {
    case home, search, profile
}
```

### Dismissing Views

```swift
struct DetailView: View {
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        Button("Done") { dismiss() }
    }
}
```

## State Management

### Which Property Wrapper When

| Wrapper | Use When |
|---------|----------|
| `@State` | View-local value type OR view-owned `@Observable` object |
| `var` (plain) | `@Observable` object passed in from parent |
| `@Binding` | Two-way connection to parent's state |
| `@Bindable` | Need bindings to a passed-in `@Observable` object's properties |
| `@Environment(\.key)` | System values (colorScheme, dismiss, locale) |
| `@Environment(MyType.self)` | `@Observable` object from `.environment()` |

### State Initialization

Create `@State` objects with default values in the property declaration, not in `init`:
```swift
// Good
struct OrderView: View {
    @State var viewModel = OrderViewModel()
}

// Acceptable — when you need to inject dependencies
struct OrderView: View {
    @State var viewModel: OrderViewModel

    init(service: OrderService) {
        _viewModel = State(initialValue: OrderViewModel(service: service))
    }
}
```

### Derived State

Don't store derived values as separate state. Use computed properties:
```swift
@Observable
final class CartViewModel {
    var items: [CartItem] = []

    // Good — computed from source of truth
    var total: Decimal {
        items.reduce(0) { $0 + $1.price * Decimal($1.quantity) }
    }

    var isEmpty: Bool { items.isEmpty }
}
```

## View Composition

### Keep Views Small — This Is a Compiler Requirement, Not Just Style

The Swift compiler uses bidirectional type inference. In large view bodies with long chains of generic modifiers (`.sheet()`, `.onChange()`, `.onReceive()`, `.navigationDestination()`), the number of type candidates grows exponentially. This causes the type-checker to timeout, producing the infamous `The compiler is unable to type-check this expression in reasonable time` error.

Extracting sub-expressions into computed properties, separate structs, or ViewModifiers gives the compiler smaller chunks to infer, solving the timeout. This is why Apple's own SwiftUI tutorials keep view bodies short — it's not just readability, it's a real compiler constraint.

**Rule of thumb**: if a view body exceeds ~40 lines or has more than 4-5 chained modifiers like `.sheet`, `.onChange`, `.alert`, `.confirmationDialog`, break it up.

```swift
struct OrderDetailView: View {
    let order: Order

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                OrderHeaderSection(order: order)
                OrderItemsSection(items: order.items)
                OrderTotalSection(total: order.total)
            }
            .padding()
        }
    }
}

// Each section is its own struct — simple, testable, and keeps the type-checker happy
private struct OrderHeaderSection: View {
    let order: Order

    var body: some View {
        VStack(alignment: .leading) {
            Text(order.title).font(.headline)
            Text(order.date.formatted()).foregroundStyle(.secondary)
        }
    }
}
```

### Fixing Type-Checker Timeouts

If the compiler chokes on a large view body, these are the fixes in order of preference:

1. **Extract subviews** — move logical sections into their own `View` structs
2. **Extract computed properties** — pull complex modifier chains into `private var someSection: some View`
3. **Extract ViewModifiers** — if multiple views share the same modifier chain
4. **Add explicit type annotations** — `let x: SomeType = expression` helps the compiler narrow candidates

```swift
// Before — compiler timeout risk
var body: some View {
    VStack {
        // 200 lines of nested views with sheets, alerts, onChange...
    }
    .sheet(item: $editItem) { ... }
    .sheet(item: $shareItem) { ... }
    .alert("Error", isPresented: $showError) { ... }
    .confirmationDialog("Delete?", isPresented: $showDelete) { ... }
    .onChange(of: searchText) { ... }
    .onReceive(timer) { ... }
}

// After — compiler handles each piece independently
var body: some View {
    VStack {
        headerSection
        contentSection
        footerSection
    }
    .modifier(OrderSheets(editItem: $editItem, shareItem: $shareItem))
    .modifier(OrderAlerts(showError: $showError, showDelete: $showDelete))
}

private var headerSection: some View { ... }
private var contentSection: some View { ... }
private var footerSection: some View { ... }
```

### Why AnyView Kills Performance — Use @ViewBuilder Instead

`AnyView` type-erases the view, which destroys SwiftUI's diffing optimization. SwiftUI's rendering engine compares old and new view trees to determine what changed — with concrete types it can do this efficiently. `AnyView` hides the concrete type, so SwiftUI can't compare the trees and falls back to tearing down and fully rebuilding the subtree on every state change.

With `@ViewBuilder`, SwiftUI sees the concrete types through `_ConditionalContent` and can diff efficiently — it knows which branch changed and only updates that branch.

```swift
// BAD — AnyView destroys diffing, full rebuild every time
func makeView(for item: Item) -> AnyView {
    if item.isPremium {
        return AnyView(PremiumItemView(item: item))
    } else {
        return AnyView(BasicItemView(item: item))
    }
}

// GOOD — @ViewBuilder preserves concrete types via _ConditionalContent
@ViewBuilder
func makeView(for item: Item) -> some View {
    if item.isPremium {
        PremiumItemView(item: item)
    } else {
        BasicItemView(item: item)
    }
}

// ALSO GOOD — Group achieves the same thing
func makeView(for item: Item) -> some View {
    Group {
        if item.isPremium {
            PremiumItemView(item: item)
        } else {
            BasicItemView(item: item)
        }
    }
}
```

If you find `AnyView` in a codebase, it's almost always replaceable with `@ViewBuilder`, `Group`, or a restructured view hierarchy. The only legitimate use case was heterogeneous collections, and even that is better handled with an enum of view types.

### ViewModifier for Reusable Styling

```swift
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.background, in: RoundedRectangle(cornerRadius: 12))
            .shadow(color: .black.opacity(0.05), radius: 4, y: 2)
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}

// Usage
Text("Hello").cardStyle()
```

### @ViewBuilder for Conditional Content

```swift
@ViewBuilder
private func statusView(for order: Order) -> some View {
    switch order.status {
    case .pending:
        Label("Pending", systemImage: "clock")
            .foregroundStyle(.orange)
    case .shipped:
        Label("Shipped", systemImage: "shippingbox")
            .foregroundStyle(.blue)
    case .delivered:
        Label("Delivered", systemImage: "checkmark.circle")
            .foregroundStyle(.green)
    }
}
```

## Async Data Loading

### The .task Modifier

Preferred over `.onAppear` + `Task {}` because it automatically cancels when the view disappears.

```swift
struct OrderListView: View {
    @State var viewModel = OrderListViewModel()

    var body: some View {
        Group {
            if viewModel.isLoading {
                ProgressView()
            } else {
                List(viewModel.orders) { order in
                    OrderRow(order: order)
                }
            }
        }
        .task {
            await viewModel.loadOrders()
        }
    }
}
```

### Refreshable

```swift
List(viewModel.orders) { order in
    OrderRow(order: order)
}
.refreshable {
    await viewModel.loadOrders()
}
```

### Searchable

```swift
struct ItemListView: View {
    @State var viewModel = ItemListViewModel()

    var body: some View {
        List(viewModel.filteredItems) { item in
            ItemRow(item: item)
        }
        .searchable(text: $viewModel.searchText, prompt: "Search items")
    }
}
```

For `@Observable` objects, use `@Bindable` if needed:
```swift
struct ItemListView: View {
    @Bindable var viewModel: ItemListViewModel

    var body: some View {
        List(viewModel.filteredItems) { item in
            ItemRow(item: item)
        }
        .searchable(text: $viewModel.searchText)
    }
}
```

## Lists & Performance

### Lazy Stacks for Large Content

```swift
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```

### Identifiable Conformance

Always conform your data models to `Identifiable` for use in `ForEach` and `List`:
```swift
struct Order: Identifiable {
    let id: UUID
    var title: String
    var amount: Decimal
}
```

## Sheets, Alerts, Confirmations

### Sheets with item binding

```swift
struct OrderListView: View {
    @State private var selectedOrder: Order?

    var body: some View {
        List(orders) { order in
            Button(order.title) { selectedOrder = order }
        }
        .sheet(item: $selectedOrder) { order in
            OrderDetailView(order: order)
        }
    }
}
```

In iOS 26, partial-height sheets are automatically inset with a Liquid Glass background. At smaller heights the bottom edges pull in, nesting in the curved edges of the display.

### Confirmation Dialogs

```swift
.confirmationDialog("Delete order?", isPresented: $showDeleteConfirmation) {
    Button("Delete", role: .destructive) { deleteOrder() }
    Button("Cancel", role: .cancel) { }
}
```

In iOS 26, dialogs automatically morph out of the buttons that present them.

### Alerts

```swift
.alert("Error", isPresented: $showError) {
    Button("OK") { }
} message: {
    Text(errorMessage)
}
```

## Haptics (iOS 17+)

Prefer `.sensoryFeedback()` over UIKit haptics:
```swift
Button("Add to Cart") { addItem() }
    .sensoryFeedback(.success, trigger: cartItemCount)
```

## Common Anti-Patterns

1. **Type-checker timeout from large view bodies** — this is the #1 SwiftUI build issue. Long chains of `.sheet()`, `.onChange()`, `.onReceive()` in a single body cause exponential type inference. The fix is always to extract subviews, computed properties, or ViewModifiers. If you see `unable to type-check this expression in reasonable time`, the view body is too large.

2. **`AnyView` type erasure** — destroys SwiftUI's diffing. `AnyView` hides concrete types so SwiftUI can't compare old/new view trees and falls back to full teardown/rebuild. With `@ViewBuilder`, SwiftUI sees concrete types through `_ConditionalContent` and diffs efficiently. Replace every `AnyView` with `@ViewBuilder` or `Group`.

3. **Business logic in view body** — views should only describe UI. Move logic to ViewModels.

4. **Deep nesting** — if your view has 5+ levels of nesting, extract subviews.

5. **Creating objects in body** — never `let vm = ViewModel()` inside `body`. It recreates every render. Use `@State`.

6. **Ignoring `Identifiable`** — using `\.self` for `id` in `ForEach` with value types that aren't truly unique causes rendering bugs.

7. **Giant `@Observable` objects** — if a ViewModel has 20+ properties, split it by feature area.

8. **Over-applying `.glassEffect()`** — Liquid Glass is for the navigation/control layer, not content. Don't put glass on text, images, or content cards. Let system controls adopt it automatically and only use `.glassEffect()` on custom chrome.
