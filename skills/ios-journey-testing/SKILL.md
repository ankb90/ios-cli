---
name: ios-journey-testing
description: Plan, execute, and verify end-to-end user journeys on iOS Simulators using natural language scenarios, layout inspection, and visual screenshot verification.
---

# iOS Journey Testing

Journey testing validates multi-step user scenarios on an iOS simulator from start to finish. It combines UI hierarchy inspection, accurate `hitPoint` touch synthesis, and screenshot capture to confirm that user flows work as intended.

---

## 1. Journey Structure

Every journey is defined by:
- **Title & Objective**: What user flow is being validated (e.g., "Onboarding & Sign-up flow", "Add to Cart and Checkout").
- **Initial State**: Pre-conditions (app installed, simulator booted, clean state).
- **Steps**: Ordered user actions (tap, scroll, type, wait).
- **Assertions**: Verifiable expectations for each step (element visibility, text content, navigation).
- **Evidence**: Visual screenshot and hierarchy snapshot captured at each checkpoint.

---

## 2. Execution Workflow

### Step 1: Session & Application Launch
1. Ensure the simulator is running and the target app is installed:
   ```text
   Tool: call_mcp_tool -> ServerName: "mobile-build", ToolName: "build_run_sim"
   ```
2. Start an interaction session:
   - For Xcode-backed sessions: Call `DeviceInteractionStartWorkspaceSession` or `DeviceInteractionStartSession`.

### Step 2: Observe Initial Screen
1. Inspect UI elements:
   - Call `snapshot_ui` (from `mobile-build`) OR `DeviceInteractionSynthesize` (from `xcode`).
2. Capture baseline screenshot:
   - Call `screenshot` from `mobile-build`.

### Step 3: Perform Action & Verify
For each user action in the journey:
1. **Locate Element**: Find the target element in the UI hierarchy snapshot. Note its `hitPoint` coordinates (preferred) or identifier/label.
2. **Execute Interaction**:
   - Tap: Synthesize tap event at `{ x, y }` using `DeviceInteractionSynthesize`.
   - Text input: Enter text into active field.
   - Gesture: Synthesize swipe / scroll if the element is off-screen.
3. **Wait for State Transition**: Allow UI animation or network request to complete.
4. **Assert Outcome**:
   - Re-capture UI hierarchy.
   - Confirm expected element exists, has correct label/value, or previous screen has dismissed.
5. **Capture Step Screenshot**:
   - Call `screenshot` to record visual proof.

### Step 4: Session Teardown
- **Always** call `DeviceInteractionEndSession` when all steps complete.

---

## 3. Handling Multi-Window & System Dialogs

- When permission dialogs (e.g. Notification, Location) or system sheets appear, inspect the element hierarchy for `activationBundleId`.
- If an element belongs to `com.apple.springboard` or a system alert, target the interaction with that specific `activationBundleId`.

---

## 4. Journey Test Report Template

At the conclusion of a journey, generate a structured markdown report:

```markdown
# 🧪 iOS Journey Test Report: [Journey Name]

- **Status**: ✅ PASSED / ❌ FAILED
- **Target**: [App Name] on [Simulator Name] (iOS [Version])
- **Timestamp**: [ISO 8601 Timestamp]

### Step Execution Summary

| Step # | Action | Expected Outcome | Actual Result | Status |
| :---: | :--- | :--- | :--- | :---: |
| 1 | Launch app and view welcome screen | "Get Started" button visible | Button found at hitPoint {195, 720} | ✅ Pass |
| 2 | Tap "Get Started" | Navigate to Login/Signup | Login form displayed | ✅ Pass |
| 3 | Enter credentials and tap "Submit" | Main Dashboard loads | Dashboard header visible | ✅ Pass |

### Step Screenshots & Evidence

#### Step 1: Welcome Screen
![Step 1 Screenshot](file:///path/to/step1.png)

#### Step 2: Login Screen
![Step 2 Screenshot](file:///path/to/step2.png)

### Issues & Observations
- [Note any visual defects, unexpected alignment, or slow transitions observed]
```
