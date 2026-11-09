---
name: ios-debugging-and-profiling
description: Deep debugging, LLDB diagnostics, crash log analysis, memory leak detection, and performance profiling for iOS apps. Use when diagnosing runtime crashes, analyzing stack traces, identifying retain cycles ([weak self]), or troubleshooting main-thread UI hitches and frame drops.
---

# iOS Deep Debugging, LLDB & Performance Profiling

This skill guides you through diagnosing runtime crashes, executing debugger commands via LLDB, resolving retain cycles and memory leaks, and eliminating main-thread UI hitches (frame drops).

---

## 1. Crash Log & Stack Trace Analysis

When an app crashes on a simulator or device, use the following investigative workflow:

### A. Fetch Logs via MCP Tools
1. Call `GetConsoleOutput` from `xcode` MCP to capture raw stdout/stderr.
2. Call `GetCrashIssueLogs` and `GetTopCrashIssues` from `xcode` MCP to inspect symbolicated crash reports.

### B. Common Crash Signatures & Root Causes

| Crash Code | Common Name | Typical Root Causes |
| :--- | :--- | :--- |
| **`EXC_BAD_ACCESS (SIGSEGV/SIGBUS)`** | Memory Access Violation | Dereferencing dangling pointer, force-unwrapping deallocated weak ref, or C interop buffer overflow. |
| **`SIGABRT`** | Abort Signal | Uncaught NSException, `fatalError()`, failed `precondition()`, or missing force-unwrapped optional (`!`). |
| **`EXC_BREAKPOINT (SIGTRAP)`** | Trap / Swift Runtime Error | Integer overflow, force unwrapping `nil`, index out of range in `Array`, or missing CoreData/SwiftData model attribute. |
| **`0x8badf00d` ("Ate bad food")** | Watchdog Termination | Main thread blocked for > 10 seconds during app launch or > 5 seconds in background. Caused by synchronous networking or synchronous disk I/O on Main Actor. |
| **`EXC_CRASH (SIGKILL)`** | Jetsam (OOM) | Memory limit exceeded by system. App terminated due to allocating too many high-resolution images or unbounded memory growth. |

---

## 2. LLDB Command Diagnostics (via `InvokeDebuggerCommand`)

Use LLDB through the `xcode` MCP tool `InvokeDebuggerCommand` to inspect live application state during a paused session or crash:

```text
Tool: call_mcp_tool -> ServerName: "xcode", ToolName: "InvokeDebuggerCommand"
Arguments: { "command": "bt" }
```

### Essential LLDB Commands:
- **`bt` / `thread backtrace all`**: Print call stack trace of the active thread (or all threads).
- **`frame select <number>`**: Jump to a specific stack frame to inspect its local variables.
- **`frame variable` (or `v`)**: Inspect local variables in the current stack frame without running expressions (fast and safe).
- **`po <expression>`**: Print object description (calls `CustomStringConvertible` / `debugDescription`).
- **`expr -l Swift -- <expression>`**: Dynamically execute Swift code in the target process.
- **`image lookup --address <hex_address>`**: Symbolicate a raw memory address to find the corresponding source file and line number.

---

## 3. Memory Leak & Retain Cycle Detection

### A. Closure Retain Cycles
Any closure that captures `self` strongly inside an object that directly or indirectly owns that closure creates a retain cycle:

```swift
// ANTI-PATTERN: Strong capture of self in an async stream or escaping closure
final class DataSyncService {
    var onDataUpdated: (() -> Void)?

    func startObserving() {
        NotificationCenter.default.addObserver(forName: .dataChanged, object: nil, queue: .main) { _ in
            self.refreshCache() // LEAK: NotificationCenter retains closure, closure retains self
        }
    }
}

// BEST PRACTICE: Weak capture and structured task cancellation
final class DataSyncService {
    private var observerTask: Task<Void, Never>?

    func startObserving() {
        observerTask?.cancel()
        observerTask = Task { [weak self] in
            for await _ in NotificationCenter.default.notifications(named: .dataChanged) {
                guard let self else { return }
                self.refreshCache()
            }
        }
    }

    deinit {
        observerTask?.cancel()
    }
}
```

### B. SwiftUI Task Lifetime Leaks
```swift
// ANTI-PATTERN: Detached Task in onAppear (outlives the view)
.onAppear {
    Task {
        await viewModel.startPolling() // Will continue running even after view dismissed!
    }
}

// BEST PRACTICE: Use SwiftUI .task modifier (automatically cancelled when view disappears)
.task {
    await viewModel.startPolling()
}
```

---

## 4. Main-Thread Hitch & Hang Elimination

Apple defines a **Hitch** as any frame that appears on screen later than scheduled (causing perceptible stutter/jank).

### Checklist for Smooth 120Hz / 60Hz Rendering:
1. **Never allocate formatters inside `body`:**
   ```swift
   // BAD: Re-creates DateFormatter on every render pass (very expensive)
   var body: some View {
       Text(DateFormatter().string(from: date))
   }
   // GOOD: Use modern formatted styles
   var body: some View {
       Text(date, style: .date)
   }
   ```
2. **Move heavy parsing and I/O to background tasks:**
   - Any JSON decoding (`JSONDecoder().decode(...)`), file reads, or cryptographic hashing must execute inside an actor or `Task.detached(priority: .userInitiated)`, hopping back to `@MainActor` only with the final view-ready model.
3. **Avoid redundant `@Observable` invalidations:**
   - Keep properties in `@Observable` classes finely scoped. Reading property `A` should not trigger re-renders when only unrelated property `B` changes.
