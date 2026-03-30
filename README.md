# ios-swift-modern

A Claude Code skill for refactoring, reviewing, and maintaining existing iOS/Swift projects with current best practices and battle-tested migration patterns.

## What It Does

When you ask Claude Code to refactor, review, or modernize an existing iOS codebase, this skill loads conventions for **Swift 6.3, iOS 26, Xcode 26.4** — plus migration pitfalls learned from real refactors that Claude doesn't know by default.

**For building new apps from scratch, use [`ios26-new-project`](../ios26-new-project/) instead.**

**Standards & Architecture:**
- **@Observable** over `ObservableObject` (with full migration guide and 9 known pitfalls)
- **async/await** over GCD and completion handlers (with Combine migration tables)
- **SwiftData** over Core Data for new projects
- **NavigationStack**, `.task`, `@Environment`, `@Bindable`
- **os.Logger** for production logging, `#if DEBUG`-gated `print()` for dev
- **Default MainActor isolation** (SE-0466) — guidance for enabling on existing projects
- **Liquid Glass** — automatic on standard controls, `.glassEffect()` for custom

**Migration Gotchas (learned the hard way):**
- `@Published` removal silently widening `private(set)` write access
- `private` on stored properties breaking struct memberwise inits (especially with `@Binding`)
- Singleton `@ObservedObject` → plain `var` and when you need `@Bindable` for bindings
- Observation notification coalescing — avoiding multiple re-renders from sequential `await` assignments
- SwiftUI type-checker timeouts from large view bodies with chained modifiers
- NSObject subclasses can't use `@Observable` — use delegate extraction pattern
- NotificationCenter async sequences replacing Combine `.publisher(for:)`
- AnyView destroying SwiftUI's `_ConditionalContent` diffing

**Patterns It Knows to Leave Alone:**
- `#if DEBUG`-gated `print()` statements
- Caseless enum namespaces (`enum Constants` over `struct`)
- Intentional `@unchecked Sendable`, custom `Equatable`/`Hashable`, `nonisolated` annotations

**Refactoring Workflow:**
- Phased approach: audit → wait for approval → file-by-file with builds → cleanup
- Git commit after each file with descriptive messages

## Install

```bash
# Project-level (one project)
mkdir -p /path/to/your/project/.claude/skills
ln -s /Users/user/Projects/ios-swift-modern /path/to/your/project/.claude/skills/ios-swift-modern

# User-level (all projects)
mkdir -p ~/.claude/skills
ln -s /Users/user/Projects/ios-swift-modern ~/.claude/skills/ios-swift-modern
```

## Structure

```
ios-swift-modern/
├── SKILL.md                           # Core standards, decision guide, patterns to preserve
└── references/
    ├── observable-migration.md        # ObservableObject → @Observable (9 pitfalls)
    ├── swift-concurrency.md           # async/await, actors, Sendable, NotificationCenter async
    └── swiftui-patterns.md            # Liquid Glass, navigation, state, type-checker fixes
```

Progressive disclosure — Claude reads `SKILL.md` first, then pulls in only the reference file relevant to the task.

## Usage

```
refactor this project
```

Claude will audit the codebase, present a report, and wait for your approval before touching anything.

## License

MIT
