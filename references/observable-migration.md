# @Observable Migration Guide

## Overview

Apple introduced the `@Observable` macro in iOS 17 (Swift 5.9) via the Observation framework. It replaces the `ObservableObject` protocol + `@Published` pattern with automatic, granular property tracking. Views only redraw when the specific properties they read change — not when any `@Published` property changes.

## Side-by-Side Comparison

### Old Way (ObservableObject)

```swift
import Combine

class UserViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var email: String = ""
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?
}

struct ProfileView: View {
    @StateObject var viewModel = UserViewModel()          // view owns lifecycle
    // OR
    @ObservedObject var viewModel: UserViewModel          // passed in
    // OR via environment:
    @EnvironmentObject var viewModel: UserViewModel       // injected with .environmentObject()

    var body: some View {
        // ANY @Published change redraws this ENTIRE view
        Text(viewModel.name)
    }
}
```

### New Way (@Observable)

```swift
import Observation

@Observable
final class UserViewModel {
    var name: String = ""
    var email: String = ""
    var isLoading: Bool = false
    var errorMessage: String?
}

struct ProfileView: View {
    @State var viewModel = UserViewModel()                // view owns lifecycle
    // OR
    var viewModel: UserViewModel                          // passed in (plain property)
    // OR via environment:
    @Environment(UserViewModel.self) var viewModel        // injected with .environment()

    var body: some View {
        // ONLY redraws when 'name' changes — not when isLoading or email change
        Text(viewModel.name)
    }
}
```

## Migration Checklist

For each `ObservableObject` class:

### Step 1: Convert the class declaration
```swift
// Before
class MyViewModel: ObservableObject {
// After
@Observable
final class MyViewModel {
```
Add `import Observation` at the top of the file if the compiler requires it.

### Step 2: Remove @Published from all properties
```swift
// Before
@Published var items: [Item] = []
@Published var isLoading = false
// After
var items: [Item] = []
var isLoading = false
```
The `@Observable` macro tracks all stored properties automatically.

### Step 3: Mark non-observed properties with @ObservationIgnored
Any property that should NOT trigger view updates:
```swift
@ObservationIgnored
private var cancellables: Set<AnyCancellable> = []

@ObservationIgnored
private var cache: [String: Data] = [:]
```

### Step 4: Update property wrappers in all consuming views

| Old | New | When |
|-----|-----|------|
| `@StateObject var vm = VM()` | `@State var vm = VM()` | View creates and owns the object |
| `@ObservedObject var vm: VM` | `var vm: VM` | Object passed in from parent |
| `@EnvironmentObject var vm: VM` | `@Environment(VM.self) var vm` | Object injected via environment |

### Step 5: Update environment injection sites
```swift
// Before
.environmentObject(viewModel)
// After
.environment(viewModel)
```

### Step 6: Handle Combine integration (if present)
This is the tricky part. If the old `ObservableObject` had:

**Custom `objectWillChange` publishers** — These don't exist in `@Observable`. The observation system handles change notification automatically. If you had custom throttling or debouncing on `objectWillChange`, you need to rethink the approach (often `.debounce` on the async side, or `withObservationTracking` for advanced cases).

**`$property` publisher references** — `@Observable` properties don't have Combine publishers. If other code subscribes to `$name` or `$isLoading`, you need to either:
- Convert the subscriber to use `withObservationTracking` (complex)
- Keep a Combine `@Published` property alongside `@Observable` temporarily (bridge)
- Refactor the subscriber to use `onChange(of:)` in SwiftUI or an `AsyncStream`

**Flag these for manual review** rather than auto-converting if you find them.

### Step 7: Build and test
Build the project after each file. The compiler will catch most issues — missing imports, wrong property wrapper types, etc.

## Bindings with @Observable

With `ObservableObject`, you used `$viewModel.property` for bindings. With `@Observable`, use `@Bindable`:

```swift
struct EditProfileView: View {
    @Bindable var viewModel: ProfileViewModel

    var body: some View {
        TextField("Name", text: $viewModel.name)
        Toggle("Notifications", isOn: $viewModel.notificationsEnabled)
    }
}
```

If the view owns the object with `@State`, bindings work directly:
```swift
struct EditProfileView: View {
    @State var viewModel = ProfileViewModel()

    var body: some View {
        TextField("Name", text: $viewModel.name)  // works directly
    }
}
```

## Common Mistakes

1. **Forgetting `@State` when the view owns the object** — if you just write `var viewModel = ProfileViewModel()` without `@State`, a new instance gets created on every view re-render.

2. **Using `@ObservedObject` or `@StateObject` with `@Observable`** — these are for `ObservableObject` only. The compiler may not always error clearly on this.

3. **Assuming `@Observable` works below iOS 17** — it doesn't. If your deployment target is iOS 16 or below, stick with `ObservableObject`.

4. **Mixing `@Observable` and `ObservableObject`** — a class can't be both. During migration you can have some classes converted and others not, but each individual class must be one or the other.

5. **Not adding `@Bindable`** — when you need to create bindings to an `@Observable` object that was passed in (not owned via `@State`), you need `@Bindable`.

## Performance Benefits

The key win: **granular observation**. With `ObservableObject`, changing `isLoading` redraws every view observing that object, even views that only display `name`. With `@Observable`, SwiftUI tracks exactly which properties each view reads in its `body` and only redraws when those properties change. This is automatic — you get it for free just by switching.

For large apps with complex view models, this eliminates a huge category of unnecessary redraws without having to manually split state across multiple `ObservableObject`s.
