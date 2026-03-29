# Swift Concurrency Guide

## Overview

Swift Concurrency (async/await, actors, structured concurrency) is the standard concurrency model for new Swift code. It replaces GCD (DispatchQueue), completion handlers, and most Combine usage for async operations.

## async/await Basics

### Converting Completion Handlers

```swift
// Old — completion handler
func fetchUser(id: String, completion: @escaping (Result<User, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, response, error in
        if let error { completion(.failure(error)); return }
        // decode...
        completion(.success(user))
    }.resume()
}

// New — async/await
func fetchUser(id: String) async throws -> User {
    let (data, response) = try await URLSession.shared.data(from: url)
    guard let http = response as? HTTPURLResponse,
          (200...299).contains(http.statusCode) else {
        throw NetworkError.invalidResponse
    }
    return try JSONDecoder().decode(User.self, from: data)
}
```

### Calling Async Functions

From an async context, just `await`:
```swift
let user = try await fetchUser(id: "123")
```

From a synchronous context, wrap in `Task`:
```swift
Task {
    let user = try await fetchUser(id: "123")
    self.user = user
}
```

In SwiftUI, use `.task`:
```swift
var body: some View {
    List(items) { item in ItemRow(item: item) }
        .task {
            await viewModel.loadItems()
        }
}
```

The `.task` modifier automatically cancels when the view disappears — unlike `.onAppear` + `Task {}` where you'd need to manage cancellation manually.

## @MainActor

Use `@MainActor` on any type or function that updates UI state. This guarantees execution on the main thread without manual dispatching.

```swift
@Observable
@MainActor
final class ProfileViewModel {
    var user: User?
    var isLoading = false

    func load() async {
        isLoading = true                    // guaranteed main thread
        defer { isLoading = false }
        user = try? await service.fetchUser()  // await suspends, resumes on main
    }
}
```

Apply `@MainActor` to:
- All ViewModels
- Any class/struct whose properties are read by SwiftUI views
- Individual functions that update `@Observable` / `@Published` state

Don't apply `@MainActor` to:
- Network services, data processing, file I/O (these should run off the main thread)
- Model types that are just data containers

## Actors

For shared mutable state that needs thread safety without blocking the main thread:

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL) -> UIImage? {
        cache[url]
    }

    func store(_ image: UIImage, for url: URL) {
        cache[url] = image
    }
}

// Usage — must await because actor-isolated
let cached = await imageCache.image(for: url)
```

Use `actor` when:
- Multiple tasks access shared mutable state
- You'd otherwise use a lock, serial queue, or `os_unfair_lock`

Don't use `actor` when:
- The state is only accessed from one context (just use a class or struct)
- The type is a ViewModel (use `@MainActor` instead)

## Structured Concurrency

### TaskGroup — parallel work with collected results

```swift
func fetchAllProfiles(ids: [String]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask {
                try await self.fetchUser(id: id)
            }
        }
        var users: [User] = []
        for try await user in group {
            users.append(user)
        }
        return users
    }
}
```

### async let — parallel work with known shape

```swift
func loadDashboard() async throws -> Dashboard {
    async let user = fetchUser()
    async let orders = fetchOrders()
    async let notifications = fetchNotifications()

    return try await Dashboard(
        user: user,
        orders: orders,
        notifications: notifications
    )
}
```

Use `async let` when you know exactly how many concurrent tasks you need. Use `TaskGroup` when the count is dynamic.

## AsyncStream / AsyncSequence

Replace Combine publishers for streaming values:

```swift
// Producing a stream
func locationUpdates() -> AsyncStream<CLLocation> {
    AsyncStream { continuation in
        let manager = CLLocationManager()
        // set up delegate that calls:
        // continuation.yield(location)
        // continuation.finish() when done

        continuation.onTermination = { _ in
            manager.stopUpdatingLocation()
        }
    }
}

// Consuming a stream
for await location in locationUpdates() {
    updateMap(with: location)
}
```

## Sendable

`Sendable` marks types as safe to pass across concurrency boundaries. The compiler enforces this with strict concurrency checking.

```swift
// Value types (struct, enum) are implicitly Sendable if all stored properties are Sendable
struct User: Sendable {
    let id: String
    let name: String
}

// Classes must be explicitly marked and must be final with immutable stored properties
// OR use actor instead
final class Config: Sendable {
    let apiKey: String
    let baseURL: URL
}

// For closures crossing actor boundaries
func process(completion: @Sendable () -> Void) { ... }
```

Enable strict concurrency checking in Xcode build settings (`SWIFT_STRICT_CONCURRENCY = complete`) to catch violations at compile time.

## Migration from GCD

| GCD Pattern | Swift Concurrency Replacement |
|---|---|
| `DispatchQueue.main.async { }` | `@MainActor` on the type or function |
| `DispatchQueue.global().async { }` | `Task { }` or `Task.detached { }` |
| Serial queue for synchronization | `actor` |
| `DispatchGroup` | `TaskGroup` or `async let` |
| `DispatchSemaphore` | `AsyncStream` + `continuation` or `CheckedContinuation` |
| `DispatchQueue.main.asyncAfter` | `try await Task.sleep(for: .seconds(n))` on `@MainActor` |

## Migration from Combine

| Combine Pattern | Swift Concurrency Replacement |
|---|---|
| `Publisher` / `AnyPublisher` | `AsyncSequence` / `AsyncStream` |
| `.sink { }` | `for await value in stream { }` |
| `.map`, `.filter` on publishers | `.map`, `.filter` on `AsyncSequence` |
| `PassthroughSubject` | `AsyncStream` with stored `continuation` |
| `@Published` + `$property` sink | `@Observable` property (SwiftUI) or `withObservationTracking` |
| `Future` | `async` function |

Combine is still fine for complex reactive pipelines or when targeting below iOS 17. But for new code, Swift Concurrency should be the default for async operations and `@Observable` for state observation.

## Common Mistakes

1. **Creating `Task {}` in a loop without limiting concurrency** — this fires all tasks at once. Use `TaskGroup` with a semaphore pattern or limit batch size.

2. **Forgetting `.task` cancels on view disappear** — if you need work to continue after navigation, use a service-layer task, not `.task` on the view.

3. **Using `Task.detached` unnecessarily** — `Task.detached` opts out of actor context inheritance. Most of the time regular `Task {}` is correct.

4. **Not handling cancellation** — check `Task.isCancelled` or use `try Task.checkCancellation()` in long-running loops.

5. **Blocking an actor** — never do synchronous heavy work inside an actor. Offload CPU-intensive work to a detached task or nonisolated method.
