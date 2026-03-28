# Notchy — Codebase Study Guide

A deep-dive reference for understanding how Notchy works, with links to relevant documentation and learning resources.

---

## What Notchy Is

A macOS menu bar app that embeds a floating terminal panel anchored to the MacBook notch. When you hover over the notch or click the menu bar icon, a panel appears with terminal sessions that auto-`cd` into detected Xcode projects and launch `claude`.

---

## File Map

```
Notchy/
├── BotdockApp.swift          Entry point (@main SwiftUI app)
├── AppDelegate.swift         App lifecycle, hotkeys, hover coordination
├── NotchWindow.swift         Notch pill UI, animations, mouse tracking
├── TerminalPanel.swift       Floating panel window (NSPanel)
├── PanelContentView.swift    SwiftUI layout: tabs + terminal area
├── SessionTabBar.swift       Tab bar with status indicators
├── TerminalSessionView.swift NSViewRepresentable bridge to terminal
├── SessionStore.swift        Central @Observable state machine
├── TerminalManager.swift     Terminal creation, pooling, status detection
├── TerminalSession.swift     Session model + TerminalStatus enum
├── XcodeDetector.swift       AppleScript + CGWindow project detection
├── CheckpointManager.swift   Git snapshot operations
└── BotFaceView.swift         Claude icon in notch pill
```

---

## Architecture Overview

### App Lifecycle

- **`BotdockApp.swift`** is the SwiftUI `@main` entry. Its body is intentionally empty — it uses `@NSApplicationDelegateAdaptor` to delegate everything to `AppDelegate`. All UI lives in native `NSWindow`/`NSPanel` objects, not SwiftUI scenes.
- **`AppDelegate`** is the coordinator. It creates the `NSStatusItem`, the `TerminalPanel`, and the `NotchWindow`. It also registers a global hotkey (backtick, keyCode 50) and manages the dual interaction model (hover vs. click).

**Learning resources:**
- [NSApplicationDelegate — Apple Docs](https://developer.apple.com/documentation/appkit/nsapplicationdelegate)
- [NSStatusItem — Apple Docs](https://developer.apple.com/documentation/appkit/nsstatusitem)
- [SwiftUI App lifecycle (WWDC 2020)](https://developer.apple.com/videos/play/wwdc2020/10037/)

---

### The Notch — `NotchWindow.swift`

An always-visible `NSPanel` positioned over the MacBook notch. Key behaviors:

- Detects notch dimensions via `NSScreen.auxiliaryTopLeftArea` / `auxiliaryTopRightArea` (macOS 12+), with a fallback of 180pt × menu bar height.
- Tracks global mouse position and triggers the panel open on hover.
- Draws a pill shape (`NotchPillView`) with curved "ear" protrusions on hover.
- Animates expand/collapse using **`CVDisplayLink`** for frame-by-frame control at the display's refresh rate.
- **Expand easing:** spring/bounce (12Hz frequency, 0.4 damping).
- **Collapse easing:** ease-out cubic, debounced 500ms to avoid flicker.
- Shows Claude face + status icon (spinner / warning / checkmark) via `NotchPillContent` (SwiftUI inside the panel).

**Status display priority:** `.taskCompleted` > `.waitingForInput` > `.working` > `.idle`

**Learning resources:**
- [NSScreen auxiliaryTopLeftArea — Apple Docs](https://developer.apple.com/documentation/appkit/nsscreen/3882915-auxiliarytopleftarea)
- [CVDisplayLink — Apple Docs](https://developer.apple.com/documentation/corevideo/cvdisplaylink-k0k)
- [NSPanel — Apple Docs](https://developer.apple.com/documentation/appkit/nspanel)
- [Spring animations explained (objc.io)](https://www.objc.io/issues/12-animations/spring-animations/)

---

### State Management — `SessionStore.swift`

A `@Observable` singleton and the single source of truth for the entire app. It manages:

| Responsibility | Detail |
|---|---|
| Session list | `[TerminalSession]` + `activeSessionId: UUID?` |
| Xcode detection | Polls every 5s when pinned, async detect + auto-switch |
| Status transitions | 3s debounce before `.working → .taskCompleted` |
| Sleep prevention | `IOPMAssertion` engaged while any session is `.working` |
| Sound playback | Throttled to 1s between sounds |
| Persistence | Encodes sessions to `UserDefaults` as `PersistedSession` |
| Checkpoint tracking | Stores last checkpoint per project |

**Dismissal tracking:** `dismissedProjects: [String: Bool]` prevents recreating tabs the user deliberately closed.

**Learning resources:**
- [Swift Observation framework (@Observable) — Apple Docs](https://developer.apple.com/documentation/observation)
- [IOPMAssertion (sleep prevention) — Apple Docs](https://developer.apple.com/documentation/iokit/iopmlib_h/iopmassertion)
- [@Observable vs ObservableObject (WWDC 2023)](https://developer.apple.com/videos/play/wwdc2023/10149/)
- [UserDefaults — Apple Docs](https://developer.apple.com/documentation/foundation/userdefaults)

---

### Terminal Management — `TerminalManager.swift`

A singleton owning `[UUID: LocalProcessTerminalView]` — terminals are created on demand and cached.

**`ClickThroughTerminalView`** subclasses SwiftTerm's `LocalProcessTerminalView` and adds:
- Buffer reading on every `dataReceived` (debounced 150ms)
- Pattern matching on the last 20 visible lines to classify status
- Arrow key interception (replaces kitty protocol with standard VT100)
- Drag-to-paste with shell escaping

**Status detection patterns:**

| Status | Detection |
|---|---|
| `.working` | Token counter line or `"esc to interrupt"` |
| `.waitingForInput` | `"❯ <digit>"` user prompt or `"Esc to cancel"` |
| `.interrupted` | `"Interrupted"` in output |
| `.idle` | Default / fallback |

**Terminal startup sequence:**
1. Spawn login shell (`--login`)
2. Send `cd <path> && clear && claude`
3. Set font (11pt monospace), colors (dark bg, light fg)

**Learning resources:**
- [SwiftTerm (migueldeicaza/SwiftTerm)](https://github.com/migueldeicaza/SwiftTerm)
- [LocalProcessTerminalView — SwiftTerm Docs](https://migueldeicaza.github.io/SwiftTermDocs/documentation/swiftterm/localprocessterminalview)
- [NSViewRepresentable — Apple Docs](https://developer.apple.com/documentation/swiftui/nsviewrepresentable)
- [VT100 escape codes reference](https://vt100.net/docs/vt100-ug/chapter3.html)

---

### Floating Panel — `TerminalPanel.swift`

A borderless, non-activating `NSPanel` that hosts `PanelContentView`.

| Feature | Detail |
|---|---|
| Positioning | Below status item or notch (`showPanel(below:)`) |
| Collapse | Toggles between full height and 44pt (tab bar only) |
| Opacity | 1.0 when key, 0.8 when collapsed+unfocused, clear bg when expanded+unfocused |
| Auto-hide | Hides on resign-key unless pinned |
| Cmd+S | Create checkpoint for active session |
| Cmd+T | Create a new plain terminal session |

**Learning resources:**
- [NSPanel — Apple Docs](https://developer.apple.com/documentation/appkit/nspanel)
- [NSWindow styling options — Apple Docs](https://developer.apple.com/documentation/appkit/nswindow/stylemask)
- [performKeyEquivalent — Apple Docs](https://developer.apple.com/documentation/appkit/nsresponder/performkeyequivalent(_:))

---

### Xcode Detection — `XcodeDetector.swift`

Two strategies, in order:

1. **AppleScript** — queries Xcode for all open workspace document paths. Returns `"Name|||Path"` pairs joined by `":::"`.
2. **CGWindowList fallback** — filters Xcode windows by bundle ID / PID, extracts project name from the window title (before `—` or `–`).

Result is an `XcodeProject` struct: `name`, `path`, `directoryPath`.

**Learning resources:**
- [AppleScript Language Guide — Apple Docs](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/introduction/ASLR_intro.html)
- [CGWindowListCopyWindowInfo — Apple Docs](https://developer.apple.com/documentation/coregraphics/1455137-cgwindowlistcopywindowinfo)
- [NSAppleScript — Apple Docs](https://developer.apple.com/documentation/foundation/nsapplescript)
- [Scripting Bridge vs NSAppleScript (objc.io)](https://www.objc.io/issues/14-mac/scripting-bridge/)

---

### Checkpoints — `CheckpointManager.swift`

Git snapshots stored as custom refs. The key trick: uses a **temporary `GIT_INDEX_FILE`** so `git add -A` doesn't disturb the user's real staging area.

**Checkpoint creation flow:**
```
git add -A          (into temp index)
git write-tree      → tree SHA
git commit-tree     → detached commit SHA
git update-ref refs/Notchy-snapshots/<project>/<timestamp> <commit>
```

**Restore:**
```
git checkout <commitHash> -- .
```

**Ref format:** `refs/Notchy-snapshots/<projectName>/<ISO8601-timestamp>`

**Learning resources:**
- [Git internals — Pro Git Book (free)](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [git write-tree — Git Docs](https://git-scm.com/docs/git-write-tree)
- [git commit-tree — Git Docs](https://git-scm.com/docs/git-commit-tree)
- [git update-ref — Git Docs](https://git-scm.com/docs/git-update-ref)
- [GIT_INDEX_FILE environment variable — Git Docs](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables)

---

## Data Flow Diagrams

### Notch Hover → Panel Open
```
Mouse enters notch rect
  → NotchWindow.checkMouse()
  → AppDelegate.notchHovered()
  → TerminalPanel.showPanelBelowNotch()
  → XcodeDetector.queryAll()
  → SessionStore.applyDetectedProjects()
  → SessionTabBar renders tabs

Mouse exits (+15pt margin)
  → hoverHideTimer fires (60ms debounce)
  → TerminalPanel.hidePanel()
```

### Terminal Status Detection
```
User types → shell output arrives
  → ClickThroughTerminalView.dataReceived()
  → debounceTimer fires (150ms)
  → evaluateStatus() reads last 20 buffer lines
  → pattern match → TerminalStatus
  → SessionStore.updateTerminalStatus()
      ├── NotchWindow animates (expand/status icon)
      ├── Sound plays (throttled 1s)
      ├── Sleep prevention toggled
      └── Panel auto-expands if waitingForInput + pinned
```

### Checkpoint (Cmd+S)
```
User presses Cmd+S
  → TerminalPanel.performKeyEquivalent()
  → SessionStore.createCheckpointForActiveSession()
  → background thread:
      CheckpointManager.createCheckpoint()
        ├── temp GIT_INDEX_FILE
        ├── git add -A
        ├── git write-tree → treeSHA
        ├── git commit-tree → commitSHA
        └── git update-ref refs/Notchy-snapshots/…
  → SessionStore.lastCheckpoint updated
  → PanelContentView shows "Checkpoint Saved" banner
```

---

## Key Design Patterns

| Pattern | Where used | Why |
|---|---|---|
| `@Observable` singleton | `SessionStore`, `TerminalManager` | Reactive state without manual `objectWillChange` |
| Lazy terminal pooling | `TerminalManager` | Terminals are expensive; create on demand, cache by UUID |
| Debouncing | Status checks (150ms), hover hide (60ms), collapse (500ms) | Prevent flickering from rapid state changes |
| `CVDisplayLink` animation | `NotchWindow` | Frame-accurate 60fps animations tied to display refresh |
| Dual detection fallback | `XcodeDetector` | AppleScript can fail; CGWindow titles are a reliable fallback |
| Temporary git index | `CheckpointManager` | Snapshots without polluting user's `git status` |
| `NSViewRepresentable` | `TerminalSessionView` | Bridge AppKit terminal view into SwiftUI layout |
| `generation` field | `TerminalSession` | Trigger terminal recreation without changing session identity |

---

## macOS-Specific APIs Used

| API | Used in | Purpose |
|---|---|---|
| `NSStatusItem` | `AppDelegate` | Menu bar icon |
| `NSPanel` | `TerminalPanel`, `NotchWindow` | Floating windows that don't steal focus |
| `NSScreen.auxiliaryTopLeftArea` | `NotchWindow` | Notch geometry (macOS 12+) |
| `CVDisplayLink` | `NotchWindow` | Display-sync animation |
| `IOPMAssertion` | `SessionStore` | Prevent system sleep |
| `CGWindowListCopyWindowInfo` | `XcodeDetector` | Read window titles of other apps |
| `NSAppleScript` | `XcodeDetector` | Query Xcode for open projects |
| `NSEventMonitor` | `AppDelegate` | Global keyboard/mouse monitoring |

---

## Further Learning

### SwiftUI + AppKit Integration
- [Mixing SwiftUI with AppKit (WWDC 2019)](https://developer.apple.com/videos/play/wwdc2019/231/)
- [NSViewRepresentable — Apple Docs](https://developer.apple.com/documentation/swiftui/nsviewrepresentable)
- [SwiftUI for Mac (WWDC 2020)](https://developer.apple.com/videos/play/wwdc2020/10041/)

### Swift Concurrency (used throughout for async detection)
- [Swift Concurrency — Swift Docs](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Meet async/await (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)
- [Actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)

### macOS App Architecture
- [Mac Catalyst and AppKit interop (WWDC 2022)](https://developer.apple.com/videos/play/wwdc2022/10076/)
- [Build a productivity app for Mac (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10323/)

### Terminal Emulation
- [SwiftTerm GitHub](https://github.com/migueldeicaza/SwiftTerm)
- [xterm.js (JS reference implementation — great for concepts)](https://xtermjs.org/)
- [VT100 terminal spec](https://vt100.net/docs/vt100-ug/contents.html)

### Git Internals
- [Pro Git Book — Git Objects (free)](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Pro Git Book — Git References (free)](https://git-scm.com/book/en/v2/Git-Internals-Git-References)
