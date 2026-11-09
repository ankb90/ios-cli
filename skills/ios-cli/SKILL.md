---
name: ios-cli
description: Core tools, commands, and workflows required to develop, build, run, inspect, and test for iOS and Apple platforms. Covers Xcode project discovery, simulator management, fast builds, UI layout inspection, screenshot capture, Apple documentation search, and journey testing.
---

# iOS CLI & Apple Developer Specialist

This skill provides unified instructions for building, running, inspecting, and testing applications across Apple platforms (iOS, iPadOS, macOS, watchOS, visionOS) using integrated MCP tools (`mobile-build` and `xcode`) and command-line utilities.

---

## 1. Session Setup & Project Discovery

Before running builds or tests in a session, always inspect or configure the session defaults to avoid passing repetitive parameters:

```text
1. Check active session defaults:
   Tool: call_mcp_tool -> ServerName: "mobile-build", ToolName: "session_show_defaults"

2. If project or scheme is missing, discover projects:
   Tool: call_mcp_tool -> ServerName: "mobile-build", ToolName: "discover_projs"
   Tool: call_mcp_tool -> ServerName: "mobile-build", ToolName: "list_schemes"

3. Set session defaults for the session:
   Tool: call_mcp_tool -> ServerName: "mobile-build", ToolName: "session_set_defaults"
   Arguments: { "project_path": "...", "scheme": "MyApp", "simulator_name": "iPhone 16" }
```

Alternatively, for Xcode workspace-centric workflows:
- Use `XcodeListWorkspaces`, `XcodeOpenWorkspace`, `XcodeListSchemes`, `XcodeSwitchScheme`.

---

## 2. Simulator Management

Manage iOS Simulators directly through `mobile-build` MCP:

- **List available simulators**:
  Call `list_sims` (returns simulator names, UDIDs, iOS runtimes, and boot states).
- **Boot simulator**:
  Call `boot_sim` with `simulator_name` (e.g., `"iPhone 16"`).
- **Open Simulator UI**:
  Call `open_sim` to bring the native macOS Simulator window to the front.

> **Tip:** `build_run_sim` will automatically boot and open the simulator frontend if needed.

---

## 3. Fast Incremental Build & Run

To build, install, and launch an iOS app on a simulator in one step:

```text
Tool: call_mcp_tool -> ServerName: "mobile-build", ToolName: "build_run_sim"
Arguments: {} (when session defaults are configured)
```

If specific overrides are needed:
- `scheme`: Target scheme name.
- `simulator_name`: Target simulator name.
- `configuration`: `"Debug"` or `"Release"`.

To stop a running app on the simulator:
- Call `stop_app_sim` with `bundle_id`.

---

## 4. UI Layout Inspection & Screen Analysis

Just like Android CLI's `layout` and `screen` commands, iOS provides deep UI inspection:

### A. Element Tree Snapshot
- Call `snapshot_ui` from `mobile-build` to extract the live element hierarchy (element types, labels, identifiers, values, and frames).

### B. Accurate Tap Coordinates & HitPoint Inspection
- Use Apple's `DeviceInteractionSynthesize` (from `xcode` MCP).
- This tool outputs calculated **`hitPoint`** coordinates for every visible element (e.g. `UIButton "Login", hitPoint: {125.0, 215.0}`).
- **Always prefer `hitPoint`** over estimated pixel positions for touch/tap gestures.

### C. Visual Screen Capture
- Call `screenshot` from `mobile-build` to capture the current simulator display to an image file.
- Call `record_sim_video` to record simulator video during test execution.

---

## 5. Journey Testing (End-to-End User Flows)

For testing multi-step user scenarios in natural language, refer to the **`ios-journey-testing`** skill:
1. Boot simulator and launch app (`build_run_sim` or `DeviceInteractionInstallAndRun`).
2. Perform step-by-step actions: Tap elements using `hitPoint`, enter text, swipe.
3. Verify intermediate UI states using `snapshot_ui` or `DeviceInteractionSynthesize`.
4. Capture screenshots as evidence at each milestone.
5. Generate a structured test report.

---

## 6. Official Apple Documentation Search

Search authoritative Apple Developer Documentation directly using Apple's official MCP tool:

```text
Tool: call_mcp_tool -> ServerName: "xcode", ToolName: "DocumentationSearch"
Arguments: { "query": "Swift concurrency async let TaskGroup" }
```

Use this tool for:
- API reference and signatures.
- Migration guides (Swift 5 to Swift 6, iOS 26/27 new APIs).
- Modern SwiftUI lifecycle, concurrency, and platform best practices.

---

## 7. SwiftUI Component Previews

For rapid component verification without building the entire application:
- Use the **`swiftui-preview`** skill.
- Calls `RenderPreview` from `xcode` MCP to render previews of specific views and inspect layouts.

---

## 8. Unit & Integration Testing

- Run tests on simulator:
  Call `test_sim` from `mobile-build`.
  Arguments: `{ "scheme": "MyAppTests", "test_plan": "..." }`
- Get coverage report:
  Call `get_coverage_report` and `get_file_coverage`.
- Run Xcode tests:
  Call `RunAllTests` or `RunSomeTests` from `xcode` MCP.
