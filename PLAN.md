# Plan: Enhanced Vertical Tab Contents with Claude Hooks

## Overview

Enhance the vertical sidebar tabs to show rich agent session context (name, current task, status) via Claude Code hooks, display horizontal sub-tab indicators with clickable navigation, and fix three additional bugs (zoom, cmd+click, duplicate notifications).

---

## Part 1: Agent Session Data Model & Hook Enhancements

### 1A. Extend the Claude wrapper hooks to capture richer data

**File: `Resources/bin/claude`**

The existing wrapper already injects `SessionStart`, `Stop`, and `Notification` hooks. We need to add two more hook events to capture ongoing task/status info:

- **`PreToolUse`** hook — fires before each tool call. We can use this to set status to "Working" and capture the tool name as context.
- **`PostToolUse`** hook — fires after each tool call completes.
- **`SubagentStart`** / **`SubagentStop`** — if available, to track sub-agent activity.

Update the `HOOKS_JSON` to include:
```json
{
  "hooks": {
    "SessionStart": [{"matcher":"","hooks":[{"type":"command","command":"cmux claude-hook session-start","timeout":10}]}],
    "Stop": [{"matcher":"","hooks":[{"type":"command","command":"cmux claude-hook stop","timeout":10}]}],
    "Notification": [{"matcher":"","hooks":[{"type":"command","command":"cmux claude-hook notification","timeout":10}]}],
    "PreToolUse": [{"matcher":"","hooks":[{"type":"command","command":"cmux claude-hook tool-start","timeout":10}]}],
    "PostToolUse": [{"matcher":"","hooks":[{"type":"command","command":"cmux claude-hook tool-end","timeout":10}]}]
  }
}
```

### 1B. Extend the Workspace model to store agent session data

**File: `Sources/Workspace.swift`**

Add new published properties to the `Workspace` class:

```swift
// Agent session tracking (per-surface)
@Published var agentSessions: [UUID: AgentSessionInfo] = [:]  // keyed by panelId

struct AgentSessionInfo {
    var sessionName: String?       // auto-detected or manual
    var currentTask: String?       // last user prompt / current task description
    var status: AgentStatus        // working, thinking, waiting, idle, stopped
    var agentType: AgentType       // claude, codex, terminal
    var lastUpdated: Date
}

enum AgentStatus: String {
    case working    // actively using tools
    case thinking   // processing (between tool calls)
    case waiting    // waiting for user input (notification hook)
    case idle       // session started but no recent activity
    case stopped    // session ended
}

enum AgentType: String {
    case claude
    case codex
    case terminal
}
```

### 1C. Add new CLI subcommands for richer hook data

**File: `CLI/cmux.swift`**

Add `claude-hook tool-start` and `claude-hook tool-end` subcommands that:
1. Parse the JSON from stdin (contains `tool_name`, `session_id`)
2. Look up the workspace+surface via session store
3. Update `agentSessions[panelId].status` and `agentSessions[panelId].currentTask`
4. Send the update to the app via socket command `report_agent_status`

Modify existing `session-start` to also set `agentSessions[panelId]` with initial state.
Modify existing `notification` to set status to `.waiting` with the notification text as context.
Modify existing `stop` to set status to `.stopped`.

### 1D. Add socket command handler for agent status updates

**File: `Sources/TerminalController.swift`**

Add a new socket command `report_agent_status` that:
- Receives: `workspace_id`, `surface_id`, `status`, `task`, `session_name`
- Updates `workspace.agentSessions[panelId]` on main queue via async
- Follows the existing `report_*` telemetry pattern (off-main parsing, minimal main-queue mutation)

### 1E. Session persistence

**File: `Sources/SessionPersistence.swift`**

Add `agentSessions` to the workspace snapshot so session info survives app restarts.

---

## Part 2: Vertical Tab UI — Display Agent Session Info

### 2A. Show agent session info in TabItemView

**File: `Sources/ContentView.swift` (TabItemView)**

Below the existing title row, add a new section that shows (when agent sessions exist):

```
┌──────────────────────────────────┐
│ 🟢 myproject/main               │  ← existing title (Name)
│ Claude: "Fix login bug"         │  ← agent session name + current task
│ ● Working (Edit tool)           │  ← status indicator
│ main · ~/projects/myapp         │  ← existing branch/directory
└──────────────────────────────────┘
```

Implementation:
- Add a computed property `agentSessionSummary` that aggregates agent sessions across all panels
- Show the most relevant/active session's task and status
- Use colored dot indicators: green=working, yellow=thinking, blue=waiting, gray=idle/stopped
- Add `@AppStorage("sidebarShowAgentSessions")` toggle (default: true)
- Update the `==` function on `TabItemView` to include the new fields for Equatable conformance

### 2B. Allow manual session naming

Add a context menu option on the agent session row: "Rename Session" that lets users set a custom name. Store in `agentSessions[panelId].sessionName`. Also support setting via:
- Socket command: `cmux set-agent-name --workspace <id> --surface <id> "my session name"`
- The existing `set-status` pattern

---

## Part 3: Sub-Tab (Horizontal Tab) Indicators

### 3A. Show horizontal tab count and types in sidebar

**File: `Sources/ContentView.swift` (TabItemView)**

Each workspace (vertical tab) can have multiple panels (horizontal tabs). Add a row showing:

```
│ 📟 Terminal  🤖 Claude (working)  🤖 Codex │
```

Implementation:
- Query `tab.sidebarOrderedPanelIds()` to get all panels
- For each panel, check `tab.agentSessions[panelId]` for type/status
- Render as a horizontal row of small pill indicators
- Each pill shows: icon + short label + status dot

### 3B. Make sub-tab indicators clickable

Each pill indicator acts as a button that:
1. Switches the workspace's focused pane to that panel: `tab.focusPanel(panelId)`
2. Also selects the workspace if not already selected: `tabManager.selectTab(tab)`

This gives direct sidebar-to-panel navigation without needing to cycle through horizontal tabs.

### 3C. Visual treatment

- Active/focused panel pill gets a subtle highlight
- Panels with notifications get a small unread badge
- Agent panels show their status color on the pill
- Keep pills compact (icon + 1-word label) to fit the sidebar width

---

## Part 4: Bug Fixes

### 4A. Fix Zoom (Cmd+/Cmd-)

**Investigation area: `Sources/Workspace.swift`, `Sources/GhosttyTerminalView.swift`**

The `toggleSplitZoom` function in `Workspace.swift` calls `bonsplitController.togglePaneZoom(inPane:)`. Need to investigate what specifically is broken — likely a focus reconciliation issue after zoom toggle, or the zoom state not being properly restored after split operations.

Steps:
1. Read the full `toggleSplitZoom` implementation and trace the call path
2. Check if `reconcileTerminalPortalVisibilityForCurrentRenderedLayout` is correctly showing/hiding portals
3. Look for any recent regressions in the zoom handling
4. Fix and test

### 4B. Fix Cmd+Click opening files in built-in browser

**File: `Sources/GhosttyTerminalView.swift`** — `resolveTerminalOpenURLTarget()` and `GHOSTTY_ACTION_OPEN_URL` handler

Current behavior: `BrowserLinkOpenSettings.openTerminalLinksInCmuxBrowser()` defaults to `true`, which routes file:// and http:// URLs to the embedded browser.

The fix has two parts:
1. **For file paths**: Detect when the URL is a local file path (file:// scheme, or a path like `/foo/bar.rs:42`). For local files, always open in the system default editor (e.g., Zed) via `NSWorkspace.shared.open()` instead of the embedded browser.
2. **Add a setting**: `openLocalFilesInExternalEditor` (default: true) in `BrowserLinkOpenSettings` so users can control this behavior.

Specifically, in the `GHOSTTY_ACTION_OPEN_URL` handler around line 2200, before checking `BrowserLinkOpenSettings.openTerminalLinksInCmuxBrowser()`, add a check:
```swift
if target.url.isFileURL || isLocalFilePath(urlString) {
    return performOnMain {
        NSWorkspace.shared.open(target.url)
    }
}
```

### 4C. Fix duplicate notifications on app focus

**File: `Sources/AppDelegate.swift`** — `applicationDidBecomeActive()`
**File: `Sources/TerminalNotificationStore.swift`**

Current behavior: When clicking back to the app, the user sees the same notification again. This is because:

1. `applicationDidBecomeActive` calls `notificationStore.markRead()` for the selected tab
2. But the macOS notification center still has the delivered notification banner
3. `willPresent` delegate always returns `[.banner, .list, .sound]` even when the app is active

The fix:
- In `userNotificationCenter(_:willPresent:withCompletionHandler:)` at line 9639: check if the notification's tab is currently the active/selected tab and the app is active. If so, suppress the banner (return empty options or just `.list`).
- In `applicationDidBecomeActive`: after marking notifications read, also remove delivered notifications from the notification center for the active tab:
  ```swift
  center.removeDeliveredNotificationsOffMain(withIdentifiers: readNotificationIds)
  ```
- Investigate whether the notification hook is being re-triggered on focus (the Claude `Notification` hook might fire again when the terminal regains focus).

---

## Part 5: Settings UI

### 5A. Add settings for new features

**File: `Sources/NotificationsPage.swift` or equivalent settings view**

Add toggles for:
- "Show agent session info in sidebar" (default: on)
- "Show sub-tab indicators in sidebar" (default: on)
- "Open local files in external editor" (default: on)

---

## Implementation Order

1. **Part 4C** — Fix duplicate notifications (smallest, highest user annoyance)
2. **Part 4B** — Fix cmd+click file opening (small, high impact)
3. **Part 4A** — Fix zoom (needs investigation)
4. **Part 1B** — Add data model for agent sessions
5. **Part 1C + 1D** — CLI + socket commands for agent status
6. **Part 1A** — Update Claude wrapper hooks
7. **Part 2A** — Show session info in sidebar
8. **Part 3A + 3B** — Sub-tab indicators with click navigation
9. **Part 1E** — Session persistence
10. **Part 2B** — Manual session naming
11. **Part 5A** — Settings UI

---

## Key Risks & Considerations

- **TabItemView typing latency**: Per CLAUDE.md, this view uses `Equatable` conformance to skip re-evaluation. Any new `@ObservedObject` or `@Binding` must be reflected in the `==` function. Agent session data should be passed as precomputed `let` parameters, not read from `tabManager` in the body.
- **Socket threading**: Agent status updates are high-frequency telemetry. Must follow the socket command threading policy — parse off-main, coalesce, then `DispatchQueue.main.async` for minimal UI mutation.
- **Localization**: All new user-facing strings must use `String(localized:defaultValue:)` per CLAUDE.md.
- **Hook timeout**: Claude Code hooks have a 10-second timeout. The `tool-start`/`tool-end` hooks must complete quickly (they just fire-and-forget a socket command).
- **Codex support**: The wrapper is currently Claude-specific. Codex integration would need a separate wrapper or detection of the `codex` binary. For now, we can detect Codex sessions by checking if the process name is `codex` and setting `agentType` accordingly.
