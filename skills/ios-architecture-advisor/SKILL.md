---
name: ios-architecture-advisor
description: Architectural design, modularization, and structural patterns for iOS applications. Covers Swift Package Manager (SPM) local packages, clean MVVM, The Composable Architecture (TCA), unidirectional data flow, dependency injection, and decoupling logic from UI.
---

# iOS Architecture & Modularization Advisor

This skill provides architectural guidance for structuring scalable, testable, and maintainable Apple platform applications. It is grounded in Apple's official engineering patterns (e.g., *Backyard Birds, Food Truck*) and industry-standard practices from Point-Free.

---

## 1. Core Principles

1. **Unidirectional Data Flow (UDF):** State flows down; events/actions flow up. Views never perform direct business logic, raw network requests, or direct database mutations inside `body`.
2. **Interface / Implementation Separation:** High-level feature modules depend on abstract interfaces (protocols or client structs), never on concrete third-party or infrastructure implementations.
3. **Isolate Side Effects:** Any non-deterministic operation (networking, disk I/O, system date, location, random numbers) must be injected as a dependency so that features remain purely unit-testable.
4. **Compile-time Isolation via Modularization:** Break monolithic application targets into fine-grained local Swift packages using Swift Package Manager (SPM).

---

## 2. Modular Architecture with Local SPM Packages

For applications exceeding 10 screens or multiple developers, decouple into a layered modular hierarchy:

```text
MyApp (App Target - Composition Root)
 ├── Feature Packages (FeatureHome, FeatureAuth, FeatureCheckout)
 │    └── Depends on: UIComponents, Model, CoreInterfaces
 ├── Core Packages (NetworkClient, AnalyticsClient, DatabaseClient)
 │    └── Implements: CoreInterfaces
 └── Shared Libraries (UIComponents, DesignSystem, FoundationExtensions)
```

### SPM Package Manifest Best Practice
```swift
// Package.swift inside /Packages/FeatureHome
let package = Package(
    name: "FeatureHome",
    platforms: [.iOS(.v18), .macOS(.v15)],
    products: [
        .library(name: "FeatureHome", targets: ["FeatureHome"])
    ],
    dependencies: [
        .package(path: "../CoreInterfaces"),
        .package(path: "../UIComponents")
    ],
    targets: [
        .target(
            name: "FeatureHome",
            dependencies: ["CoreInterfaces", "UIComponents"]
        ),
        .testTarget(
            name: "FeatureHomeTests",
            dependencies: ["FeatureHome"]
        )
    ]
)
```

---

## 3. Dependency Injection Pattern (Interface Clients)

Prefer value-type **Client Structs** over heavyweight Protocol trees. Client structs enable lightweight test mocking without protocol pollution:

```swift
// 1. Definition in CoreInterfaces
public struct UserClient: Sendable {
    public var fetchCurrentUser: @Sendable () async throws -> User
    public var updateProfile: @Sendable (UserProfile) async throws -> Void

    public init(
        fetchCurrentUser: @escaping @Sendable () async throws -> User,
        updateProfile: @escaping @Sendable (UserProfile) async throws -> Void
    ) {
        self.fetchCurrentUser = fetchCurrentUser
        self.updateProfile = updateProfile
    }
}

// 2. Live Implementation in CoreImplementations
extension UserClient {
    public static func live(network: HTTPClient) -> Self {
        UserClient(
            fetchCurrentUser: { try await network.get("/user/me") },
            updateProfile: { profile in try await network.post("/user/profile", body: profile) }
        )
    }
}

// 3. Mock Implementation for Tests & SwiftUI Previews
extension UserClient {
    public static let mock = UserClient(
        fetchCurrentUser: { .stub(id: "1", name: "Jane Doe") },
        updateProfile: { _ in }
    )
}
```

---

## 4. Modern SwiftUI MVVM / Observable State Pattern

When using MVVM with `@Observable` (iOS 17+ / Swift 6):

```swift
// AVOID: Exposing unconstrained mutable properties that allow Views to mutate internal state directly.
// AVOID: Performing async work without task lifetime management.

// PREFER: Explicit State and Actions with MainActor isolation.
@Observable
@MainActor
public final class HomeViewModel {
    public enum ViewState: Sendable {
        case idle
        case loading
        case loaded([FeedItem])
        case failure(String)
    }

    public private(set) var state: ViewState = .idle
    private let feedClient: FeedClient

    public init(feedClient: FeedClient) {
        self.feedClient = feedClient
    }

    public func onAppear() async {
        guard case .idle = state else { return }
        state = .loading
        do {
            let items = try await feedClient.fetchItems()
            state = .loaded(items)
        } catch {
            state = .failure(error.localizedDescription)
        }
    }
}
```

---

## 5. Architectural Anti-Patterns Checklist

Before signing off on an architectural change, verify:
- [ ] **No Singleton Sprawl:** Avoid `NetworkManager.shared` or `Database.shared` scattered in views. Route through Environment or dependencies.
- [ ] **No Business Logic in Views:** If a SwiftUI view file has `URLSession`, formatting logic, or multi-step async branching, move it to a dedicated model or client.
- [ ] **No Circular Package Dependencies:** Check package dependency graphs to ensure Feature A does not depend on Feature B if B also depends on A. Use a shared Navigation or Coordinator protocol.
- [ ] **Strict Sendable Compliance:** Ensure all models passed across module boundaries conform to `Sendable`.
