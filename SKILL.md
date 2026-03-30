---
name: ios-swift-modern
description: Modern iOS/Swift development standards and patterns for new and existing projects. Use this skill whenever the user asks to create, scaffold, refactor, or review an iOS app, Swift package, SwiftUI view, Xcode project, or any Apple platform code (macOS, watchOS, visionOS, tvOS). Also trigger when the user mentions Swift, SwiftUI, UIKit, Xcode, @Observable, async/await in a Swift context, MVVM for iOS, Core Data, SwiftData, Liquid Glass, or any Apple framework. Use this even for quick one-file Swift tasks to ensure modern conventions are followed. If the user says "make me an app" or "build an iOS app" or "new Xcode project" — use this skill.
---

# Modern iOS/Swift Development Skill

This skill ensures all iOS and Apple platform code follows current best practices as of **Swift 6.3, iOS 26 (Liquid Glass), Xcode 26.4, and macOS Tahoe 26**. Apple moved to year-based versioning in 2025 — iOS went from 18 directly to 26, and all platforms now share the same version number.

**Current versions (March 2026):** iOS 26.4, Xcode 26.4, Swift 6.3, macOS Tahoe 26.4

**App Store requirement:** Starting April 2026, all submissions must be built with the iOS 26 SDK.

Read the appropriate reference files in `references/` based on what the task requires. Don't read all of them upfront — only pull in what's relevant.

## Quick Decision Guide

Before writing any code, determine:

1. **Deployment target** — This drives everything. If iOS 26 (new projects), use Liquid Glass design, `@Observable`, `SwiftData`, all modern APIs. If iOS 17+, use `@Observable` and `SwiftData` but skip Liquid Glass APIs. If below iOS 17, use `ObservableObject`, Core Data.
2. **UI framework** — SwiftUI (default for new projects) or UIKit (only if the project already uses it or has a specific requirement). Note: SwiftUI in iOS 26 gained native `WebView` and rich text `TextEditor` support.
3. **Architecture** — MVVM is the default. Don't introduce TCA, VIPER, or other patterns unless the user specifically requests them.
4. **Concurrency model** — Swift Concurrency (async/await, actors) is the default. No GCD in new code. Swift 6.2+ adds the `@concurrent` attribute for more granular concurrency control.
5. **Design language** — iOS 26 introduces Liquid Glass. Apps rebuilt with Xcode 26 SDK automatically get Liquid Glass on standard controls. Custom adoption uses `.glassEffect()` and `GlassEffectContainer`.

## Project Structure

New projects should follow this layout:

```
AppName/
├── App/
│   ├── AppNameApp.swift          (@main entry point)
│   └── ContentView.swift         (root view / navigation)
├── Models/
├── ViewModels/
├── Views/
│   └── FeatureName/              (group views by feature)
├── Services/
├── Utilities/
│   └── Extensions/
└── Resources/
    ├── Assets.xcassets
    └── Localizable.xcstrings
```

One primary type per file. File names match the type name.

## Patterns To Recognize and Leave Alone

During refactoring, these are good practices — don't "fix" them:

### Debug-gated print() statements
`print()` calls wrapped in `#if DEBUG` are intentional and should be preserved in existing codebases. In release builds, bare `print()` writes to stderr which degrades performance and can leak information. If a codebase already uses `#if DEBUG` print() alongside `os.Logger`, don't refactor the prints into Logger calls — they're serving as throwaway/verbose debug output that the developer intentionally excluded from structured logging.

However, for **new code**, prefer `os.Logger` even for debug-level output (see the Logging section below). The `#if DEBUG` print() pattern is a valid legacy approach but Logger is strictly better for anything you'd want to keep long-term.

### Caseless enums as namespaces
A `case`-less `enum` used to hold `static let` constants is the Swift-idiomatic namespace pattern. Unlike a `struct`, it can't be accidentally instantiated. Don't convert these to structs, don't add cases, and don't add `private init()` — the absence of cases already prevents instantiation.

```swift
// This is correct — leave it alone
enum API {
    static let baseURL = URL(string: "https://api.example.com")!
    static let timeout: TimeInterval = 30
}

enum AnimationDuration {
    static let short: CGFloat = 0.15
    static let medium: CGFloat = 0.3
    static let long: CGFloat = 0.5
}
```

Only keys shared across 2+ files belong in a shared namespace enum. File-private constants should stay local to avoid over-centralizing.

### Other patterns to preserve
- Well-structured `// MARK: -` sections — don't reorganize them arbitrarily
- Intentional `@unchecked Sendable` conformance with documented reasoning
- Custom `Equatable`/`Hashable` implementations that deliberately exclude properties
- `nonisolated` annotations that exist to avoid unnecessary actor hops
- Existing unit test structure and naming conventions

## Swift Language Standards

### Access Control
Mark everything as restrictive as possible. Default to `private` and only widen as needed. Mark classes `final` unless designed for inheritance.

### Optionals
No force unwraps except clear invariants (`@IBOutlet`). Use `guard let` for early returns, `if let` for branching, nil coalescing for defaults.

### Error Handling
Use typed errors with custom `Error` enums. Never silently swallow errors with `try?` unless you explicitly log the failure.

```swift
enum NetworkError: Error, LocalizedError {
    case invalidResponse(statusCode: Int)
    case decodingFailed(underlying: Error)
    case unauthorized

    var errorDescription: String? {
        switch self {
        case .invalidResponse(let code): "Server returned status \(code)"
        case .decodingFailed: "Failed to parse response"
        case .unauthorized: "Authentication required"
        }
    }
}
```

### Enums Over Strings
No stringly-typed keys, notification names, user defaults keys, or identifiers.

```swift
enum UserDefaultsKey: String {
    case hasCompletedOnboarding
    case preferredTheme
}
```

### Value Types
Prefer `struct` over `class` unless reference semantics are explicitly needed (ViewModels are the main exception with `@Observable`).

### Logging

**`os.Logger` is the default for all new code** — both production and debug logging. It's superior to `#if DEBUG print()` in every way:

1. **Privacy by default** — Logger auto-redacts dynamic content in release builds. You explicitly opt-in with `privacy: .public` for non-sensitive data. No risk of leaking user data.
2. **Always available** — Unlike `#if DEBUG print()`, Logger messages are always captured but only visible with a debugger or Console.app attached. This means you can diagnose production issues on user devices without needing a debug build.
3. **Structured categories** — Subsystem + category lets you filter logs in Console.app by specific area (e.g., show only network logs, or only storage logs), which is invaluable for debugging specific subsystems.
4. **Log levels** — `.debug`, `.info`, `.error`, `.fault` let the system manage retention and visibility. Debug-level messages have minimal performance impact and are only persisted when logging is actively observed.

```swift
import os

// Define categories as static Logger instances for structured filtering
extension Logger {
    private static let subsystem = Bundle.main.bundleIdentifier!

    static let network = Logger(subsystem: subsystem, category: "Network")
    static let auth = Logger(subsystem: subsystem, category: "Auth")
    static let storage = Logger(subsystem: subsystem, category: "Storage")
    static let cloudKit = Logger(subsystem: subsystem, category: "CloudKit")
    static let memory = Logger(subsystem: subsystem, category: "Memory")
}

// Usage — note privacy controls on dynamic content
Logger.network.debug("Request started: \(endpoint, privacy: .public)")
Logger.network.info("Response: \(statusCode, privacy: .public) in \(duration)s")
Logger.network.error("Failed: \(error.localizedDescription, privacy: .public)")

// Sensitive data is auto-redacted by default — no privacy annotation needed
Logger.auth.info("Token refreshed for user \(userId)")  // userId is redacted in release
```

**When `#if DEBUG print()` is still acceptable:**
- Throwaway debug output you intend to remove before committing
- Extremely verbose dump output (full JSON payloads, etc.) that would pollute structured logs

**Never use bare `print()` without `#if DEBUG`** — it writes to stderr in release, degrades performance, and can leak information.

## Observation — @Observable vs ObservableObject

→ **Read `references/observable-migration.md` for full migration guide and side-by-side comparisons.**

The rule: iOS 17+ → `@Observable`. Below iOS 17 → `ObservableObject`.

Quick summary for new iOS 17+/26 code:

```swift
import Observation

@Observable
@MainActor
final class ProfileViewModel {
    var user: User?
    var isLoading = false
    var errorMessage: String?

    @ObservationIgnored
    private var internalCache: [String: Any] = [:]
}

// In views — no special property wrappers needed in most cases:
struct ProfileView: View {
    @State var viewModel = ProfileViewModel()        // view owns lifecycle
    // OR
    var viewModel: ProfileViewModel                  // passed in from parent
    // OR
    @Environment(ProfileViewModel.self) var viewModel // from environment
}
```

## Swift Concurrency

→ **Read `references/swift-concurrency.md` for async/await patterns, actors, Sendable, and migration from GCD/Combine.**

Key rules:
- `async/await` over completion handlers — always
- `@MainActor` on ViewModels and anything touching UI state
- `actor` for shared mutable state needing thread safety
- `Task {}` to bridge sync → async (`.task` modifier, button actions)
- `AsyncStream` / `AsyncSequence` to replace Combine publishers where practical
- Enable strict concurrency checking in build settings
- Swift 6.2+ introduces `@concurrent` for marking functions that should run concurrently off the caller's actor

```swift
@Observable
@MainActor
final class OrderViewModel {
    var orders: [Order] = []
    var isLoading = false

    private let service: OrderService

    init(service: OrderService = .init()) {
        self.service = service
    }

    func loadOrders() async {
        isLoading = true
        defer { isLoading = false }
        do {
            orders = try await service.fetchOrders()
        } catch {
            // handle error
        }
    }
}
```

## SwiftUI Patterns

→ **Read `references/swiftui-patterns.md` for navigation, state management, Liquid Glass, modifiers, and common patterns.**

Key conventions:
- `NavigationStack` (never the deprecated `NavigationView`)
- `.navigationDestination(for:)` for type-safe navigation
- `.task` modifier for async work on appear (replaces `.onAppear` + `Task {}`)
- `@Environment(\.dismiss)` to dismiss views
- Keep view bodies under ~40 lines — extract subviews aggressively (this is a compiler requirement, not just style)
- `ViewModifier` for reusable styling
- Prefer `.sensoryFeedback()` (iOS 17+) over UIKit haptics
- **iOS 26:** Liquid Glass automatically applies to standard controls when built with Xcode 26 SDK. Use `.glassEffect()` for custom Liquid Glass surfaces.

## Data Persistence

iOS 17+ new projects → **SwiftData**. Existing Core Data projects → keep Core Data unless asked to migrate.

```swift
@Model
final class Expense {
    var title: String
    var amount: Double
    var date: Date
    var category: Category?

    init(title: String, amount: Double, date: Date = .now) {
        self.title = title
        self.amount = amount
        self.date = date
    }
}
```

Simple key-value → `@AppStorage` or a typed `UserDefaults` wrapper with enum keys.

## Networking

`URLSession` with async/await. No third-party HTTP libraries unless requested.

```swift
actor NetworkService {
    private let session: URLSession
    private let decoder: JSONDecoder

    init(session: URLSession = .shared, decoder: JSONDecoder = .init()) {
        self.session = session
        self.decoder = decoder
    }

    func fetch<T: Decodable>(_ type: T.Type, from endpoint: Endpoint) async throws -> T {
        let (data, response) = try await session.data(for: endpoint.request)
        guard let http = response as? HTTPURLResponse,
              (200...299).contains(http.statusCode) else {
            throw NetworkError.invalidResponse(statusCode: (response as? HTTPURLResponse)?.statusCode ?? 0)
        }
        return try decoder.decode(T.self, from: data)
    }
}
```

## Things To Never Use in New Code

- `DispatchQueue.main.async` → use `@MainActor`
- `NotificationCenter` for internal app communication → use `@Observable` properties or `AsyncStream`
- Completion handler patterns → use `async/await`
- `AnyView` → restructure with `@ViewBuilder` or concrete types
- `NavigationView` → use `NavigationStack`
- `ObservableObject` + `@Published` in iOS 17+ → use `@Observable`
- Force unwraps / implicitly unwrapped optionals (except `@IBOutlet`)
- `Any` / `AnyObject` when a protocol or generic works
- Singleton pattern for services → use dependency injection
- `@UIApplicationDelegateAdaptor` unless you genuinely need UIKit lifecycle hooks
- Bare `print()` outside of `#if DEBUG` — use `os.Logger` instead

## Naming Conventions

- Types: `PascalCase` — `UserProfileView`, `OrderService`
- Functions/properties: `camelCase` — `fetchUserProfile()`, `isLoading`
- Constants: `camelCase` — `let maxRetryCount = 3`
- Protocols: Descriptive — `ProfileFetching`, `AuthenticationProviding`
- Files: Match the primary type — `UserProfileView.swift`
- Extensions: `TypeName+Context.swift` — `Date+Formatting.swift`

## Refactoring Existing Projects

When refactoring (not starting fresh):

1. **Audit first** — read the entire codebase, produce a report, then STOP and wait for approval before changing anything
2. **File-by-file** — dependencies first, leaf files first. Build after every change.
3. **Preserve functionality** — never change behavior unless asked
4. **Git commit after each file** — `refactor: convert NetworkManager to async/await`
5. **Final cleanup** — dead code, consistent organization, `// MARK: -` sections
