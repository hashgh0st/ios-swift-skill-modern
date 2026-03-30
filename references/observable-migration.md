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

### Step 2: Remove @Published — but check access level first

**This is a critical step.** `@Published` silently hides access intent. Before removing it, check whether the original `@Published` property was effectively `private(set)`:

```swift
// Before — @Published made this writable only internally by convention
class SettingsViewModel: ObservableObject {
    @Published var theme: Theme = .system           // anyone could write
    @Published private(set) var user: User?         // only the class could write
    @Published var isLoading = false                // was this meant to be private(set)?
}

// After — you MUST preserve the write access that was intended
@Observable
final class SettingsViewModel {
    var theme: Theme = .system                      // still publicly writable — correct
    private(set) var user: User?                    // preserved — correct
    private(set) var isLoading = false              // added private(set) — the class controls this
}
```

If the original had `@Published var isLoading` but only the class ever set it, add `private(set)`. Removing `@Published` without checking this silently widens write access from "only internally writable" to "writable by anyone."

The `@Observable` macro tracks all stored properties automatically — you don't need `@Published` at all.

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

### Step 5: Migrate singleton access patterns
Singletons accessed via `@ObservedObject` simplify significantly:

```swift
// Before
@ObservedObject private var settings = SettingsStore.shared
@ObservedObject private var voiceStore = VoicePersonaStore.shared

// After — observation is automatic, no wrapper needed
var settings = SettingsStore.shared
var voiceStore = VoicePersonaStore.shared
```

Note: if the view needs `$` bindings to the singleton's properties, use `@Bindable`:
```swift
@Bindable var settings = SettingsStore.shared
```

Also watch out for `private` on these properties — with `@ObservedObject`, the wrapper handled initialization internally. With a plain `var`, marking it `private` can interfere with the view's memberwise init if the view is a struct. Remove `private` unless you have an explicit `init`.

### Step 6: Update environment injection sites
```swift
// Before
.environmentObject(viewModel)
// After
.environment(viewModel)
```

### Step 7: Handle Combine integration (if present)
This is the tricky part. If the old `ObservableObject` had:

**Custom `objectWillChange` publishers** — These don't exist in `@Observable`. The observation system handles change notification automatically. If you had custom throttling or debouncing on `objectWillChange`, you need to rethink the approach (often `.debounce` on the async side, or `withObservationTracking` for advanced cases).

**`$property` publisher references** — `@Observable` properties don't have Combine publishers. If other code subscribes to `$name` or `$isLoading`, you need to either:
- Convert the subscriber to use `withObservationTracking` (complex)
- Keep a Combine `@Published` property alongside `@Observable` temporarily (bridge)
- Refactor the subscriber to use `onChange(of:)` in SwiftUI or an `AsyncStream`

**Flag these for manual review** rather than auto-converting if you find them.

### Step 8: Build and test
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

## Performance: Coalescing Observation Notifications

With `@Observable`, each property mutation triggers a separate observation notification. If you set multiple properties across separate `await` boundaries, each one causes a view re-render:

```swift
// Bad — three separate notifications, three re-renders at startup
func loadInitialData() async {
    user = try? await fetchUser()           // notification 1 → re-render
    preferences = try? await fetchPrefs()   // notification 2 → re-render
    history = try? await fetchHistory()     // notification 3 → re-render
}

// Good — parallel fetch, single batch update, one re-render
func loadInitialData() async {
    async let u = fetchUser()
    async let p = fetchPrefs()
    async let h = fetchHistory()

    let (fetchedUser, fetchedPrefs, fetchedHistory) = await (
        try? u, try? p, try? h
    )

    // Single synchronous block — one observation notification
    user = fetchedUser
    preferences = fetchedPrefs
    history = fetchedHistory
}
```

This matters most at startup or when refreshing multiple data sources. The pattern is: do all your async work first, collect the results, then assign them all synchronously in one pass.

## Common Mistakes

1. **Forgetting `@State` when the view owns the object** — if you just write `var viewModel = ProfileViewModel()` without `@State`, a new instance gets created on every view re-render.

2. **Using `@ObservedObject` or `@StateObject` with `@Observable`** — these are for `ObservableObject` only. The compiler may not always error clearly on this.

3. **Assuming `@Observable` works below iOS 17** — it doesn't. If your deployment target is iOS 16 or below, stick with `ObservableObject`.

4. **Mixing `@Observable` and `ObservableObject`** — a class can't be both. During migration you can have some classes converted and others not, but each individual class must be one or the other.

5. **Not adding `@Bindable`** — when you need to create bindings to an `@Observable` object that was passed in (not owned via `@State`), you need `@Bindable`.

6. **Silently widening write access** — removing `@Published` without adding `private(set)` where the original intent was internal-only writes. Always check who was setting each property before removing `@Published`.

7. **Multiple async assignments causing multiple re-renders** — setting observed properties across separate `await` points triggers a notification per assignment. Coalesce by fetching in parallel and assigning synchronously in one pass.

## Performance Benefits

The key win: **granular observation**. With `ObservableObject`, changing `isLoading` redraws every view observing that object, even views that only display `name`. With `@Observable`, SwiftUI tracks exactly which properties each view reads in its `body` and only redraws when those properties change. This is automatic — you get it for free just by switching.

For large apps with complex view models, this eliminates a huge category of unnecessary redraws without having to manually split state across multiple `ObservableObject`s.
