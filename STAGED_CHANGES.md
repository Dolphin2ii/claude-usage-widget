# Staged Changes (Local Only)

Changes accumulating here are waiting to be merged into `develop`.
This file is gitignored — it never gets pushed to the remote repo.

---

## Branches Staged

| Branch | Description |
|--------|-------------|
| `fix/weekly-tray-click-duplicate` | Tray icon restore fixes |
| `fix/minimize-disappears-from-taskbar` | Minimize/restore regression on Windows (v1.7.2) |

---

## Changes

### fix/weekly-tray-click-duplicate

**Tray icon — duplicate click handler removed**
The `weeklyTray` click handler was registered twice in `createTray()`. When the
window was hidden and the weekly tray icon was clicked, both handlers fired in
sequence: the first restored the window, the second immediately hid it again,
causing the window to blink on screen and disappear.
Fix: removed the duplicate handler.

**Tray icon — double-blink on restore fixed (Windows)**
On Windows, showing a hidden transparent + frameless + alwaysOnTop window caused
two visible render flashes before the window settled. This is a Windows DWM
layered-window artifact: the compositor does an initial render pass, then a second
pass when Electron re-asserts the alwaysOnTop z-order.
Fix: added `showMainWindowClean()` helper that sets opacity to 0, calls `show()`,
then restores opacity to 1 after ~50ms once the DWM has composited the window.
Applies to Windows only (`process.platform === 'win32'`). macOS and Linux are
unaffected and continue to use a plain `show()` call.
Applied to both tray click handlers and the "Show Widget" context menu item.

---

### fix/minimize-disappears-from-taskbar

Reported by users on Windows 10 and Windows 11: clicking the minimize button
caused the app to disappear from both the taskbar and the system tray, making
it completely inaccessible. Introduced in v1.7.2, not present in v1.7.1.
Three interdependent bugs combined to produce the symptom.

**Bug 1 — Tray icons destroyed after every data poll**
`updateTrayIcon()` destroyed both `sessionTray` and `weeklyTray` whenever
`showTrayStats` was `false` (the default setting). This ran on every data poll,
so within seconds of startup both tray icons were silently gone. The
`minimize-window` handler relies on the tray icon as the restore path when the
window is hidden, so with no tray icon the app became completely inaccessible.
Fix: only destroy `weeklyTray` when stats are disabled. `sessionTray` is kept
alive as a static icon and always serves as the restore path.

**Bug 2 — Minimize always called `hide()` on Windows regardless of settings**
The `minimize-window` IPC handler always called `mainWindow.hide()` on
Windows, which removes the window from the taskbar entirely. This was designed
to work alongside a persistent tray icon (as in v1.7.1), but Bug 1 had already
destroyed the tray. The "Hide from Taskbar" (`minimizeToTray`) setting existed
but was never consulted.
Fix: check `minimizeToTray` before deciding. When off (the default), call
`mainWindow.minimize()` so the window stays in the taskbar like a normal
Windows application. When on, call `hide()` as before — the tray icon (kept
alive by Bug 1's fix) provides the restore path.

**Bug 3 — `showMainWindowClean()` left the window unfocused and unresponsive**
After restoring the window via the tray icon, UI elements (e.g. the settings
cog) were unresponsive to clicks. `showMainWindowClean()` used an opacity trick
(`setOpacity(0)` → `show()` → 50 ms → `setOpacity(1)`) to suppress a DWM
blink artifact, but never called `focus()`. Without focus, Windows does not
route mouse events to the window. Additionally, the `setOpacity` sequence left
the window in a layered state that further interfered with input routing.
Fix: removed the opacity trick and replaced it with `show()` + `focus()`
directly, matching the behaviour of v1.7.1. This supersedes the
`showMainWindowClean()` change introduced in `fix/weekly-tray-click-duplicate`
— the DWM double-blink that change targeted is acceptable compared to an
unresponsive window.

---

*Add new entries above this line as additional branches are staged.*
