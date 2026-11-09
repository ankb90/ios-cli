---
name: swiftui-preview
description: Render and visually inspect SwiftUI #Preview macros and PreviewProvider definitions using Apple's official Xcode RenderPreview tool. Verify component layouts, variants, localizations, and widget timelines rapidly without launching the full app.
---

# SwiftUI Preview Specialist

This skill guides you through rendering and verifying SwiftUI component previews using Apple's official `RenderPreview` tool from the `xcode` MCP server.

---

## 1. When to Use

- After modifying a SwiftUI View or component to visually verify layout and styling.
- To test different preview variants (e.g., Light vs Dark mode, Dynamic Type sizes).
- To test localization (e.g., English vs German text expansion) in UI components.
- To inspect WidgetKit or Live Activity timelines.
- When full app compilation and simulator launch would be unnecessarily slow for a localized UI change.

---

## 2. Using `RenderPreview`

### Basic Invocation
```text
Tool: call_mcp_tool
ServerName: "xcode"
ToolName: "RenderPreview"
Arguments: {
  "sourceFilePath": "MyApp/Views/ProfileCardView.swift",
  "previewDefinitionIndexInFile": 0
}
```

### Parameters:
- `sourceFilePath` *(required)*: Path to the Swift source file relative to the project/workspace root (e.g., `"MyApp/Sources/Views/HeaderView.swift"`).
- `previewDefinitionIndexInFile` *(optional, default: 0)*: Zero-based index of the `#Preview` macro in the file (0 for first preview, 1 for second, etc.).
- `workspaceIdentifier` *(optional)*: Absolute path to the `.xcodeproj` or `.xcworkspace`.
- `previewLocalizationOverride` *(optional)*: Locale code (e.g., `"es"`, `"ja"`, `"fr"`) to preview localized text strings.
- `previewVariantOverrides` *(optional)*: Dictionary of variant group overrides (color scheme, orientation).
- `previewCanvasControlOverrides` *(optional)*:
  - `timelineIndex`: Index for timeline-based previews (Widgets, Live Activities).
  - `groupItemIndex`: Index for parameter-based `#Preview(traits: ...)`.
  - `toggleState`: Boolean state for toggles (e.g., Live Activity expanded state).

---

## 3. Best Practices for Previews

1. **Provide clear `#Preview` macros**:
   ```swift
   #Preview("Default State") {
       ProfileCardView(user: .mock)
   }

   #Preview("Dark Mode", traits: .darkMode) {
       ProfileCardView(user: .mock)
   }
   ```
2. **Review Render Results**: Inspect the output image and layout constraints returned by the tool to ensure labels are not clipped and paddings conform to Human Interface Guidelines (HIG).
3. **If render fails**:
   - Verify that the target scheme and destination are set correctly (`XcodeSwitchScheme`, `XcodeSwitchRunDestination`).
   - Check if preview mock data contains compilation errors.
