# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Infinity Terminal is a native macOS terminal: a horizontally scrolling grid of
terminals, always two rows tall and N columns wide. Swift 5.9 / AppKit + SwiftUI,
built on [SwiftTerm](https://github.com/migueldeicaza/SwiftTerm). `AGENTS.md`
holds the style and naming conventions; this file covers commands and the
architecture that spans multiple files.

## Commands

```sh
swift build                    # debug → .build/debug/InfinityTerminal
swift build -c release         # release → .build/release/InfinityTerminal
./build-app.sh                 # release build wrapped in .build/InfinityTerminal.app (ad-hoc signed if no Developer ID)
./build-app.sh --dmg           # + DMG, notarized if APPLE_ID / APPLE_PASSWORD / APPLE_TEAM are set (env or .env)
```

- macOS only (AppKit, SwiftTerm, `proc_pidinfo`); it cannot be compiled or run on Linux.
- There is no test target and no linter. If tests are added, use `swift test` with a `.testTarget` under `Tests/InfinityTerminalTests/`.
- `swift build` after editing `Package.swift` re-resolves SwiftTerm; `Package.resolved` is committed.

## Release flow

1. Bump `VERSION` and `BUILD_NUMBER` in `build-app.sh` (the only place the version lives; `Info.plist` is generated from it).
2. Write `RELEASE_NOTES.md` as `- ` bullets. The release workflow uses it as the GitHub release body, and `update-web.yml` parses those bullets (`**bold**` → `<strong>`, `` `code` `` → `<code>`) into the changelog on infinityterminal.com.
3. Tag `v<VERSION>` and push. `.github/workflows/release.yml` fails if the tag doesn't match `VERSION`.
4. When adding or changing a keyboard shortcut, update all three places: the key handler in `main.swift`, the list in `Views/ShortcutsView.swift`, and the tables in `README.md`.

## Architecture

### Bootstrap and global event routing (`main.swift`)

There is no SwiftUI `App`. `main.swift` builds an `NSApplication` by hand, creates a
`CenteredTrafficLightWindow`, and hosts `ContentView` in an `NSHostingView`. The
`AppDelegate` owns three `NSEvent.addLocalMonitorForEvents` monitors that do work
SwiftUI cannot, because SwiftTerm's `LocalProcessTerminalView` has focus and most of
its methods (`scrollWheel`, `mouseDown`, `becomeFirstResponder`) are `public` but not
`open`:

- **keyDown** – every app shortcut (⌘⇧arrows, ⌃⇥, ⌥⌘arrows, ⌘/, ⌘⌥O, font size, ⌥-arrow / ⌥⌫ word motions, ⌘C/⌘V). Returning `nil` consumes the event. `onKeyPress` never fires while a terminal is first responder.
- **scrollWheel** – direction-locked per gesture. Horizontal → forwarded to the outermost `NSScrollView` (the SwiftUI grid). Vertical on the alternate buffer → `InfinityTerminalNSView.forwardWheelToAltBuffer` sends xterm wheel events or arrow keys to the child, since SwiftTerm has no scrollback there.
- **leftMouseDown / Up** – sets `gridModel.activeSessionID` from the hit-tested pane, and temporarily disables mouse reporting for a modifier-free drag so text selection works inside TUIs that grab the mouse (restored on mouse-up).

The delegate talks to `ContentView` only through `Notification.Name` constants
defined at the bottom of `main.swift` (`jumpToHome`, `scrollColumnLeft`,
`focusColumnNext`, `focusPaneUp`, …). `ContentView` subscribes with `.onReceive`
and does the scrolling/focusing.

### Model (`Models/`)

`TerminalGridModel` (shared via `@EnvironmentObject`) → `[TerminalColumn]` →
`[TerminalSession]` (1 or 2 per column). All three are `ObservableObject` classes
with stable `UUID` ids. Every `ForEach` is keyed by those ids so SwiftUI moves views
in place when panes swap or columns reorder, which is what keeps the underlying
NSViews and their shell processes alive.

- Focus is `activeSessionID`. `focusTarget(inColumn:preferringRow:)` encodes the rules for where keyboard navigation lands (maximized pane > same row > last-used pane in that column > top). Keyboard navigation moves focus; trackpad scrolling and minimap drags deliberately do not.
- `refreshPane` replaces a session with a brand-new `TerminalSession`. A new id means SwiftUI calls `makeNSView` again and a fresh shell starts. This is also what happens when a shell exits.
- `maximizedSessionID` on a column collapses the other pane to a 30pt strip; the hidden pane's shell keeps running.

### Persistence (`Models/SessionSnapshot.swift`, `TerminalGridModel`)

Live objects hold non-Codable state, so they are projected into `GridSnapshot`
(columns → sessions with color/cwd/name, font size, active pane, scroll offset,
Option-as-Meta). It is written to
`~/Library/Application Support/Infinity Terminal/session.json`. Call
`scheduleSave()` after any model mutation (500 ms debounce); `saveNow()` runs on
quit. New snapshot fields must be `Optional` so older files still decode; bump
`GridSnapshot.currentVersion` only for incompatible changes, which discards old
sessions.

### PTY lifecycle (`Views/TerminalPaneView.swift`)

`TerminalPaneView` is an `NSViewRepresentable` whose `makeNSView` returns a
throwaway `TerminalSlot`, but the real `InfinityTerminalNSView` (a
`LocalProcessTerminalView` subclass) is cached on `session.cachedTermView` and
re-parented into whatever slot SwiftUI creates. That cache is the reason a pane can
move between columns without restarting its shell. Other things the subclass owns:

- `dataReceived` override: keeps the viewport where the user scrolled and keeps a selection highlighted while output streams (SwiftTerm would otherwise snap to bottom and clear the selection on every chunk). Scrollback is raised to 50,000 lines.
- Process exit is detected with a `DispatchSourceProcess` on the shell PID, not SwiftTerm's delegate, which proved unreliable under SwiftUI.
- Working directory comes from the OSC 7 delegate callback plus a 30 s `proc_pidinfo` poll for shells that don't emit OSC 7; it feeds session restore so panes reopen in the same directory.
- `optionAsMetaKey` defaults to off (Terminal.app behavior) and is driven app-wide from `gridModel.useOptionAsMetaKey` through `updateNSView`.

### Layout (`Views/ContentView.swift`, `Views/TerminalColumnView.swift`)

The grid uses custom `Layout` types (`GridStripLayout` for the column strip,
`ColumnPanesLayout` for the panes in a column) instead of `HStack`/`VStack`. Plain
stacks never proposed a concrete size to the `NSViewRepresentable`, so SwiftTerm
stayed at a 0×0 grid. Column width is `max(400, windowWidth / 2)`. The grid's
`NSScrollView` is captured via `ScrollViewSpy` for instant (non-animated) scrolls
and scroll-offset restore on launch.

The toolbar is 38pt tall; `CenteredTrafficLightWindow.toolbarHeight` in
`main.swift` and the `.frame(height: 38)` in `ToolbarView` must stay in sync, as
must the `toolbar` constant in `ContentView.scrollGrid`.

### Resources

Use `Bundle.appResources` (in `ResourceBundle.swift`) rather than `Bundle.module`;
the SPM-generated accessor calls `fatalError` when the app is launched from
`/Applications`.
