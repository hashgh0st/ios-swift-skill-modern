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

### Step 5: Migrate singleton and `private` property patterns

Singletons accessed via `@ObservedObject` simplify significantly:

```swift
// Before
@ObservedObject private var settings = SettingsStore.shared
@ObservedObject private var voiceStore = VoicePersonaStore.shared

// After — observation is automatic, no wrapper needed
var settings = SettingsStore.shared
var voiceStore = VoicePersonaStore.shared
```

If the view needs `$` bindings to the singleton's properties, use `@Bindable`:
```swift
@Bindable var settings = SettingsStore.shared
```

**The `private` and memberwise init trap:** In Swift, marking ANY stored property `private` on a struct makes the compiler-generated memberwise init `private`. With `@ObservedObject`, the property wrapper handled initialization internally so this wasn't visible. After migration, `private` on a plain `var` directly affects the struct's init visibility.

The rule depends on whether the struct has properties WITHOUT default values:

```swift
// SAFE — all properties have defaults, so the no-arg init MyView() works
// even though private makes the memberwise init private
struct SettingsView: View {
    private var store = SettingsStore.shared     // has default → no-arg init works
    private var authManager = AuthManager.shared // has default → fine

    var body: some View { ... }
}

// BROKEN — @Binding has no default, so the memberwise init is required
// but private on another property made it private
struct SubscriptionView: View {
    @Binding var settings: AppSettings          // no default → needs memberwise init
    private var store = SubscriptionStore.shared // ← this makes memberwise init private!

    var body: some View { ... }
}

// FIX — remove private, or write an explicit init
struct SubscriptionView: View {
    @Binding var settings: AppSettings
    var store = SubscriptionStore.shared         // removed private → memberwise init is internal

    var body: some View { ... }
}
```

**Before adding or keeping `private` on a migrated property in a View struct, check:** does this struct have ANY properties without default values (`@Binding`, `let` params, non-optional properties without `=`)? If yes, don't use `private` on stored properties unless you write an explicit `init`. If all properties have defaults, `private` is safe because the no-arg init still works.

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

## NSObject Subclasses and @Observable

`@Observable` **cannot be applied to NSObject subclasses.** The macro needs to synthesize stored properties for observation tracking, which conflicts with Objective-C's runtime. If you try, you'll get compiler errors.

This comes up when migrating classes that act as delegates for UIKit/AppKit/CoreBluetooth/etc. frameworks — those delegates must inherit from NSObject.

### The Delegate Extraction Pattern

The solution is to separate the `@Observable` state from the NSObject delegate:

```swift
// The public @Observable class — clean API, no NSObject
@Observable
@MainActor
final class SpeechManager {
    var isListening = false
    var transcript = ""
    var error: Error?

    private var delegate: SpeechDelegate?

    func startListening() {
        let delegate = SpeechDelegate(owner: self)
        self.delegate = delegate
        // configure and start the speech recognizer with this delegate...
        isListening = true
    }

    func stopListening() {
        isListening = false
        delegate = nil
    }
}

// Private NSObject subclass — handles Obj-C delegate callbacks only
private class SpeechDelegate: NSObject, SFSpeechRecognizerDelegate {
    weak var owner: SpeechManager?

    init(owner: SpeechManager) {
        self.owner = owner
        super.init()
    }

    func speechRecognizer(_ speechRecognizer: SFSpeechRecognizer, availabilityDidChange available: Bool) {
        Task { @MainActor in
            if !available {
                owner?.error = SpeechError.unavailable
            }
        }
    }
}
```

The key points:
- The `@Observable` class owns the state and the public API
- The NSObject delegate subclass is private, handles only Obj-C callbacks, and forwards results back to the owner via a `weak` reference
- Use `Task { @MainActor in }` in delegate callbacks to safely update the `@Observable` owner's state
- The delegate doesn't hold any observed state itself — it's just a bridge

This pattern applies to any framework that requires NSObject-based delegates: `CLLocationManager`, `CBCentralManager`, `AVCaptureSession`, `WCSession`, `SFSpeechRecognizer`, etc.

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

8. **`private` on a stored property breaking memberwise init** — in a View struct, marking a migrated property `private` makes the memberwise init private. If the struct has any properties without defaults (like `@Binding`), the struct becomes unconstructable from outside. Check for non-defaulted properties before using `private`.

9. **Trying to apply `@Observable` to an NSObject subclass** — the macro can't synthesize its backing storage on NSObject because of Obj-C runtime conflicts. Use the delegate extraction pattern: keep the `@Observable` class pure Swift, and use a private NSObject subclass for Obj-C delegate callbacks that forwards to the owner via weak reference.

## Performance Benefits

The key win: **granular observation**. With `ObservableObject`, changing `isLoading` redraws every view observing that object, even views that only display `name`. With `@Observable`, SwiftUI tracks exactly which properties each view reads in its `body` and only redraws when those properties change. This is automatic — you get it for free just by switching.

For large apps with complex view models, this eliminates a huge category of unnecessary redraws without having to manually split state across multiple `ObservableObject`s.
