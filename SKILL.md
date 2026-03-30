---
name: ios-swift-modern
description: Modern iOS/Swift standards for refactoring, reviewing, and maintaining existing projects. ALWAYS use this skill when the user asks to refactor, clean up, review, fix, update, or modernize an existing iOS app, Swift codebase, or Xcode project. Trigger on ANY of these — "refactor", "clean up", "modernize", "migrate", "update to iOS 26", "fix this code", "review my code", "convert to @Observable", "migrate to SwiftData", "remove ObservableObject", "update to async/await", or any request to improve existing Swift/SwiftUI code. Also trigger on quick one-file Swift tasks, code review requests, or when the user pastes Swift code and asks questions about it. This skill knows critical iOS 26 conventions including default MainActor isolation (SE-0466), @Observable migration pitfalls (private(set) widening, memberwise init breakage, NSObject delegate extraction), and battle-tested patterns learned from real refactors. Without this skill, refactoring advice will use outdated patterns and miss known migration landmines. For building NEW apps from scratch, prefer ios26-new-project instead.
---

# Modern iOS/Swift Development Skill (Refactoring & Maintenance)

This skill ensures all iOS and Apple platform code follows current best practices as of **Swift 6.3, iOS 26 (Liquid Glass), Xcode 26.4, and macOS Tahoe 26**. Apple moved to year-based versioning in 2025 — iOS went from 18 directly to 26, and all platforms now share the same version number.

**Current versions (March 2026):** iOS 26.4, Xcode 26.4, Swift 6.3, macOS Tahoe 26.4

**App Store requirement:** Starting April 2026, all submissions must be built with the iOS 26 SDK.

This skill is optimized for **refactoring existing projects** and **reviewing/maintaining Swift code**. For building new apps from scratch, use the `ios26-new-project` skill instead.

Read the appropriate reference files in `references/` based on what the task requires. Don't read all of them upfront — only pull in what's relevant.

## Quick Decision Guide

Before writing any code, determine:

1. **Deployment target** — This drives everything. If iOS 26 (new projects), use Liquid Glass design, `@Observable`, `SwiftData`, all modern APIs. If iOS 17+, use `@Observable` and `SwiftData` but skip Liquid Glass APIs. If below iOS 17, use `ObservableObject`, Core Data.
2. **UI framework** — SwiftUI (default for new projects) or UIKit (only if the project already uses it or has a specific requirement). Note: SwiftUI in iOS 26 gained native `WebView` and rich text `TextEditor` support.
3. **Architecture** — MVVM is the default. Don't introduce TCA, VIPER, or other patterns unless the user specifically requests them.
4. **Concurrency model** — Swift Concurrency (async/await, actors) is the default. No GCD in new code. Swift 6.2+ adds the `@concurrent` attribute for more granular concurrency control.
5. **Design language** — iOS 26 introduces Liquid Glass. Apps rebuilt with Xcode 26 SDK automatically get Liquid Glass on standard controls. Custom adoption uses `.glassEffect()` and `GlassEffectContainer`.
6. **Default Actor Isolation** — New Xcode 26 projects default to `@MainActor` isolation (SE-0466). If refactoring toward this, enable the build setting and add `nonisolated` / `@concurrent` where needed. Existing projects default to `nonisolated` and require manual `@MainActor`.

## Default MainActor Isolation (iOS 26 / Swift 6.2+)

Xcode 26 new projects default ALL code to `@MainActor`. This is a build setting ("Default Actor Isolation") controlled by SE-0466. When refactoring an existing project toward iOS 26:

**If enabling default MainActor isolation on an existing project:**
- Remove manual `@MainActor` from ViewModels — it's now redundant
- Add `nonisolated` to data models, `Codable` structs, and types that cross isolation boundaries
- Add `nonisolated` to ALL SwiftData `@Model` classes (critical — without it they get MainActor isolation and break in background contexts)
- Add `@concurrent` to async functions that should run off the main thread
- SPM packages/libraries should keep `nonisolated` default — they need to be usable from any isolation domain
- `actor` types are unaffected — they have their own isolation domain

**If keeping the existing nonisolated default:**
- Continue using explicit `@MainActor` on ViewModels
- No changes needed to data models
- This is the safer path for incremental migration

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
enum API {
    static let baseURL = URL(string: "https://api.example.com")!
    static let timeout: TimeInterval = 30
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

### Enums Over Strings
No stringly-typed keys, notification names, user defaults keys, or identifiers.

### Value Types
Prefer `struct` over `class` unless reference semantics are explicitly needed (ViewModels are the main exception with `@Observable`).

### Logging

**`os.Logger` is the default for all new code** — both production and debug logging:

1. **Privacy by default** — Logger auto-redacts dynamic content in release builds.
2. **Always available** — captured even without `#if DEBUG`, visible in Console.app.
3. **Structured categories** — filter by subsystem + category.
4. **Log levels** — `.debug`, `.info`, `.error`, `.fault`.

```swift
import os

extension Logger {
    private static let subsystem = Bundle.main.bundleIdentifier!
    static let network = Logger(subsystem: subsystem, category: "Network")
    static let auth = Logger(subsystem: subsystem, category: "Auth")
    static let storage = Logger(subsystem: subsystem, category: "Storage")
}

Logger.network.debug("Request started: \(endpoint, privacy: .public)")
Logger.network.error("Failed: \(error.localizedDescription, privacy: .public)")
```

## Observation — @Observable vs ObservableObject

→ **Read `references/observable-migration.md` for full migration guide, pitfalls, and side-by-side comparisons.**

The rule: iOS 17+ → `@Observable`. Below iOS 17 → `ObservableObject`.

The migration guide covers 9 known pitfalls including: `@Published` hiding `private(set)` intent, `private` breaking memberwise inits, singleton access patterns, NSObject delegate extraction, observation notification coalescing, and more.

## Swift Concurrency

→ **Read `references/swift-concurrency.md` for async/await patterns, actors, Sendable, NotificationCenter async sequences, and migration from GCD/Combine.**

Key rules:
- `async/await` over completion handlers — always
- `@MainActor` on ViewModels and anything touching UI state (unless using default MainActor isolation)
- `actor` for shared mutable state needing thread safety
- `Task {}` to bridge sync → async (`.task` modifier, button actions)
- `AsyncStream` / `AsyncSequence` to replace Combine publishers where practical
- `NotificationCenter.default.notifications(named:)` for system notifications (auto-cleanup on Task cancellation)
- Enable strict concurrency checking in build settings
- Swift 6.2+ introduces `@concurrent` for marking functions that should run concurrently off the caller's actor

## SwiftUI Patterns

→ **Read `references/swiftui-patterns.md` for navigation, state management, Liquid Glass, type-checker fixes, AnyView, modifiers, and common patterns.**

Key conventions:
- `NavigationStack` (never the deprecated `NavigationView`)
- `.navigationDestination(for:)` for type-safe navigation
- `.task` modifier for async work on appear
- Keep view bodies under ~40 lines — this is a compiler constraint (type-checker timeout), not just style
- Never use `AnyView` — it destroys SwiftUI's diffing via `_ConditionalContent`
- **iOS 26:** Liquid Glass automatically applies when built with Xcode 26 SDK

## Data Persistence

iOS 17+ new projects → **SwiftData**. Existing Core Data projects → keep Core Data unless asked to migrate.

**If using default MainActor isolation:** add `nonisolated` to all `@Model` classes.

## Networking

`URLSession` with async/await. No third-party HTTP libraries unless requested.

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