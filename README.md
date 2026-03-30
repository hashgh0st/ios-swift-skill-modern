# ios-swift-modern

A Claude Code skill that enforces modern iOS/Swift best practices across all your projects — including migration gotchas learned from real refactors.

## What It Does

When Claude Code detects an iOS/Swift task, this skill automatically loads conventions for **Swift 6.3, iOS 26 (Liquid Glass), Xcode 26.4, and macOS Tahoe 26** — so every file it writes or refactors follows current standards out of the box.

**Standards & Architecture:**
- **Liquid Glass** design language — `.glassEffect()`, `GlassEffectContainer`, layered icons, when to apply and when not to
- **@Observable** over `ObservableObject` (with full migration guide and known pitfalls)
- **async/await** over GCD and completion handlers (with Combine migration tables)
- **SwiftData** over Core Data for new projects
- **NavigationStack**, `.task`, `@Environment`, `@Bindable`
- New iOS 26 SwiftUI APIs: native `WebView`, rich text `TextEditor`, `@Animatable` macro
- **MVVM** architecture with clean separation
- **Access control**, error handling, naming conventions
- **os.Logger** for production logging, `#if DEBUG`-gated `print()` for dev

**Migration Gotchas (learned the hard way):**
- `@Published` removal silently widening `private(set)` write access
- `private` on stored properties breaking struct memberwise inits (especially with `@Binding`)
- Singleton `@ObservedObject` → plain `var` and when you need `@Bindable` for bindings
- Observation notification coalescing — avoiding multiple re-renders from sequential `await` assignments
- SwiftUI type-checker timeouts from large view bodies with chained modifiers

**Patterns It Knows to Leave Alone:**
- `#if DEBUG`-gated `print()` statements
- Caseless enum namespaces (`enum Constants` over `struct`)
- Intentional `@unchecked Sendable`, custom `Equatable`/`Hashable`, `nonisolated` annotations

**Refactoring Workflow:**
- Phased approach: audit → wait for approval → file-by-file with builds → cleanup
- Git commit after each file with descriptive messages

## Install

Copy the `ios-swift-modern/` folder into your Claude Code skills directory:

```bash
cp -r ios-swift-modern/ ~/.claude/skills/ios-swift-modern
```

Or symlink it:

```bash
ln -s /path/to/ios-swift-modern ~/.claude/skills/ios-swift-modern
```

## Structure

```
ios-swift-modern/
├── SKILL.md                           # Core standards, decision guide, patterns to preserve
└── references/
    ├── observable-migration.md        # ObservableObject → @Observable (with pitfall guide)
    ├── swift-concurrency.md           # async/await, actors, Sendable, GCD/Combine migration
    └── swiftui-patterns.md            # Liquid Glass, navigation, state, view composition, type-checker fixes
```

The skill uses progressive disclosure — Claude reads `SKILL.md` first, then pulls in only the reference file relevant to the task.

## Usage

Just work normally. The skill triggers on any Swift/iOS/SwiftUI/Xcode task. For refactors, say:

```
refactor this project
```

Claude will audit the codebase, present a report, and wait for your approval before touching anything.

## License

MIT
