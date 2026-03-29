# ios-swift-modern

A Claude Code skill that enforces modern iOS/Swift best practices across all your projects.

## What It Does

When Claude Code detects an iOS/Swift task, this skill automatically loads conventions for Swift 6, iOS 17-18, and Xcode 16+ — so every file it writes or refactors follows current standards out of the box.

Covers:

- **@Observable** over `ObservableObject` (with full migration guide)
- **async/await** over GCD and completion handlers
- **SwiftData** over Core Data for new projects
- **NavigationStack**, `.task`, `@Environment`, `@Bindable`
- **MVVM** architecture with clean separation
- **Access control**, error handling, naming conventions
- Phased refactoring workflow (audit → file-by-file → cleanup)

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
├── SKILL.md                           # Core standards + decision guide
└── references/
    ├── observable-migration.md        # ObservableObject → @Observable
    ├── swift-concurrency.md           # async/await, actors, Sendable
    └── swiftui-patterns.md            # Navigation, state, view composition
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
