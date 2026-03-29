# SwiftUI Patterns & Best Practices

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

### Confirmation Dialogs

```swift
.confirmationDialog("Delete order?", isPresented: $showDeleteConfirmation) {
    Button("Delete", role: .destructive) { deleteOrder() }
    Button("Cancel", role: .cancel) { }
}
```

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

2. **`AnyView` erasure** — kills SwiftUI's diffing performance. Use `@ViewBuilder`, `Group`, or concrete conditional views.

3. **Business logic in view body** — views should only describe UI. Move logic to ViewModels.

4. **Deep nesting** — if your view has 5+ levels of nesting, extract subviews.

5. **Creating objects in body** — never `let vm = ViewModel()` inside `body`. It recreates every render. Use `@State`.

6. **Ignoring `Identifiable`** — using `\.self` for `id` in `ForEach` with value types that aren't truly unique causes rendering bugs.

7. **Giant `@Observable` objects** — if a ViewModel has 20+ properties, split it by feature area.
