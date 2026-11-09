# Apple Platform & iOS Agent Guidelines

When developing, testing, or interacting with Apple platform applications (iOS, iPadOS, macOS, watchOS, visionOS), agents must strictly adhere to the following directives:

## 1. Preflight Session & Destination Management
- **Mandatory Preflight:** Before executing the first build, run, or test tool call in a session, call `session_show_defaults` from `mobile-build` or check `XcodeListRunDestinations` from `xcode` to confirm the active workspace, scheme, and destination simulator.
- **No Speculative Targets:** Never guess scheme names or simulator identifiers without verifying them via discovery tools (`discover_projs`, `list_schemes`, `list_sims`).

## 2. Tooling Priority: MCP over Raw Shell Commands
- **Prefer MCP Tools:** Always use `mobile-build` and `xcode` MCP tools over executing raw shell commands (`xcodebuild`, `xcrun simctl`) to avoid process timeouts, DerivedData lock contention, and hanging background daemons.
- Only fall back to shell commands when an operation is explicitly unsupported by available MCP tools.

## 3. UI Automation & Device Resource Lifecycle
- **HitPoint Accuracy:** In UI interaction sessions (`DeviceInteractionSynthesize`), ALWAYS prioritize calculated `hitPoint` coordinates over estimated visual coordinates.
- **Multi-Window Contexts:** Inspect element hierarchies for `activationBundleId` when handling system alerts, notification banners, or multi-window environments.
- **Mandatory Session Teardown:** ALWAYS call `DeviceInteractionEndSession` immediately after finishing UI verification workflows to free memory, simulator processes, and system threads.

## 4. Component-Level SwiftUI Previews
- For isolated SwiftUI component modifications, prioritize the `RenderPreview` tool from `xcode` MCP over full application compilation and simulator execution.

## 5. Engineering Standards: Concurrency & Memory Safety
- **Swift 6 Concurrency:** Enforce strict concurrency safety. Never apply `@unchecked Sendable` as a shortcut to suppress compiler diagnostics; use proper actor isolation (`@MainActor`), client structs, or value semantics.
- **Retain Cycle Prevention:** Ensure escaping closures and asynchronous tasks capture `[weak self]` when retaining instances, and bind asynchronous task lifecycles directly to the view lifecycle (e.g., `.task`).
