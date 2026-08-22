# Windows desktop-integration feasibility

_Resolves [#3](https://github.com/scwlkr/scary-gary/issues/3). Researched 2026-08-21 against Microsoft documentation and first-party Microsoft engineering guidance._

## Decision

Every planned Gary behavior is feasible for Windows 10/11 without administration, network access, filesystem tricks, or control of another process. The reliable foundation is a normal per-user desktop process with small, borderless layered Win32 overlay windows. Gary should be rendered above the ordinary desktop, should never take focus, and should pass all pointer input through.

Three effects must be honest simulations rather than shell integration:

- **Sit on Start:** anchor Gary to the taskbar/work-area edge. Windows documents the taskbar rectangle, but not a stable Start-button rectangle.
- **Crawl behind windows:** clip Gary against observed window bounds. Do not reparent Gary into another process or continually rewrite third-party z-order.
- **Steal desktop icons:** animate a bundled generic fake icon. Do not enumerate, hide, move, rename, or copy a user's real desktop items.

This keeps the desktop visually haunted while preserving the product rule that Gary only _looks_ dangerous.

## Supported overlay foundation

A top-level `WS_EX_LAYERED` window supports per-pixel alpha and efficient animation through `UpdateLayeredWindow`. On layered windows, alpha-zero pixels pass mouse input through; adding `WS_EX_TRANSPARENT` passes mouse events through regardless of shape. Microsoft recommends combining `WS_EX_TRANSPARENT` with `WS_EX_LAYERED` for top-level hit testing ([Window Features](https://learn.microsoft.com/en-us/windows/win32/winmsg/window-features), [DWM best practices](https://learn.microsoft.com/en-us/windows/win32/dwm/bestpractices-ovw)).

Use these window properties for every Gary surface:

- `WS_EX_LAYERED | WS_EX_TRANSPARENT | WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW`.
- `WS_EX_TOPMOST` only while Gary is meant to be visible above ordinary app windows.
- `SetWindowPos(..., SWP_NOACTIVATE, ...)` for movement, so animation does not activate Gary ([SetWindowPos](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowpos)).
- A transparent, undecorated popup with bounds kept near the currently visible sprite. Large monitor-sized windows are reserved for the brief Gary Event.

`WS_EX_NOACTIVATE` prevents click activation, while `WS_EX_TOOLWINDOW` omits the window from the taskbar and Alt+Tab. A topmost window is guaranteed above **non-topmost** windows, not above every other topmost surface ([Extended Window Styles](https://learn.microsoft.com/en-us/windows/win32/winmsg/extended-window-styles), [Window z-order](https://learn.microsoft.com/en-us/windows/win32/winmsg/window-features)). Consequently, Gary must never fight Task Manager, another topmost app, UAC, the lock screen, or an exclusive-full-screen application. Losing visual precedence is the safe fallback.

## Effect-by-effect feasibility

| Effect | v0.1 decision | Supported mechanism and safest fallback |
| --- | --- | --- |
| Walk on taskbar | **Supported approximation** | Walk along the edge between `rcWork` and `rcMonitor`, or the system taskbar rectangle when available. If no reserved strip is discoverable (for example, auto-hide), walk along the monitor's bottom edge instead. |
| Sit on Start | **Visual simulation** | Use a taskbar-edge anchor at the expected Start region; never depend on Explorer child-window class names. Exact placement is best effort because the documented appbar API exposes the system taskbar rectangle, not the Start button. Fall back to sitting on the nearest taskbar corner/edge. |
| Sleep on taskbar | **Supported approximation** | Same taskbar-edge geometry as walking, with a static/looping sprite. Re-anchor after shell, display, or settings changes. |
| Screen-edge peek | **Supported** | Choose an enumerated monitor and move a tightly bounded layered window partly outside its `rcMonitor`; signed virtual-screen coordinates support monitors left/above the primary display. |
| Crawl behind windows | **Visual simulation** | Observe eligible top-level window bounds and clip Gary's own pixels/region as he crosses an edge. If the target moves, closes, minimizes, becomes cloaked, or cannot be measured, finish the hide animation at its nearest edge and later reappear elsewhere. Never parent to Explorer/another app or alter the target's z-order. |
| Drag garbage | **Supported illusion** | Animate bundled garbage sprites inside Gary-owned overlays. No desktop file is created or moved, and the pointer remains usable because the overlay is click-through. |
| Stare at, follow, or flee cursor | **Supported** | Poll `GetCursorPos`, which returns screen coordinates, and steer Gary without installing a global mouse hook ([GetCursorPos](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getcursorpos)). If the input desktop is unavailable, stop reacting rather than retrying aggressively. |
| Sprint across screen | **Supported** | Move/animate the same small overlay across one monitor at a time. Clamp the path to that monitor and recompute if the display topology changes. |
| Fall over / hold signs | **Supported** | Sprite/state changes wholly inside Gary's overlay; no desktop integration is needed. |
| Fake icon theft | **Visual simulation only** | Spawn a clearly generic, bundled fake icon as a Gary-owned sprite, then have Gary carry it away. Never query item names/images or mutate Explorer's folder view. If the illusion would overlap real content, abandon that event. |
| Screen flicker | **Visual simulation only** | Briefly animate opacity/color in a click-through monitor-sized overlay. Do not change display modes, monitor settings, gamma, wallpaper, or another app. If Gary cannot cover a surface, leave it uncovered. |
| Full-screen Gary face | **Supported on the ordinary desktop** | Show a borderless, topmost, non-activating, click-through overlay on the cursor/foreground monitor for a fixed short duration, then destroy it. Do not request exclusive full screen, change resolution, capture input, or attempt to cover secure desktops. Other topmost windows may remain above it. |
| Scream / Gary Event silence | **Supported with a limit** | Play a bundled local asset in Gary's process-specific audio session and control only that session. “Silence” means Gary stops his own audio; it cannot mute other applications or the system. Windows explicitly advises apps not to control unrelated sessions ([Audio Sessions](https://learn.microsoft.com/en-us/windows/win32/coreaudio/audio-sessions), [Session Volume Controls](https://learn.microsoft.com/en-us/windows/win32/coreaudio/session-volume-controls)). Honor Windows mixer mute/volume. |

## Geometry and shell constraints

### Taskbar and Explorer

`SHAppBarMessage(ABM_GETTASKBARPOS)` returns the **system taskbar** bounding rectangle. Microsoft recommends `GetMonitorInfo` when the real need is usable work area not covered by the taskbar or other appbars ([SHAppBarMessage](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-shappbarmessage)). Therefore:

1. Enumerate monitors and retain both `MONITORINFO.rcMonitor` and `rcWork`; both use virtual-screen coordinates and can be negative ([MONITORINFO](https://learn.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-monitorinfo)).
2. Use `ABM_GETTASKBARPOS` only to improve placement around the system taskbar. Do not treat it as complete geometry for every secondary-monitor taskbar or third-party appbar.
3. Infer a safe walk/sleep edge from `rcWork` versus `rcMonitor`; if ambiguous, use the monitor edge.
4. Never search for `Shell_TrayWnd`, `SHELLDLL_DefView`, `SysListView32`, or other Explorer implementation classes.

Explorer broadcasts the registered `TaskbarCreated` message when it creates/recreates the taskbar; Windows 10 can also broadcast it on primary-display DPI changes ([Taskbar creation notification](https://learn.microsoft.com/en-us/windows/win32/shell/taskbar#taskbar-creation-notification)). On that message, discard cached taskbar geometry and requery. Also requery on `WM_SETTINGCHANGE` because consumers should reload used system settings when it arrives ([WM_SETTINGCHANGE](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-settingchange)). Gary's windows must remain independent of Explorer, so an Explorer restart can cause temporary fallback placement but cannot kill or orphan Gary.

### Multiple monitors and DPI

Call `EnumDisplayMonitors(NULL, NULL, ...)` and maintain one geometry record per monitor. A display-change notification can invalidate every `HMONITOR`, so discard and rebuild the collection on `WM_DISPLAYCHANGE` ([EnumDisplayMonitors](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumdisplaymonitors), [HMONITOR lifetime](https://learn.microsoft.com/en-us/windows/win32/gdi/hmonitor-and-the-device-context)). Never assume the primary display begins at `(0,0)` or use unsigned coordinate extraction.

Declare `PerMonitorV2` DPI awareness in the executable manifest before any window is created; Microsoft recommends the manifest over setting awareness programmatically ([default DPI awareness](https://learn.microsoft.com/en-us/windows/win32/hidpi/setting-the-default-dpi-awareness-for-a-process)). Handle `WM_DPICHANGED`, rebuild the scaled sprite/render target, and accept the suggested window rectangle ([WM_DPICHANGED](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged)). This prevents mismatched cursor, monitor, and occlusion coordinates when monitors use different scale factors.

### Other windows, occlusion, and z-order

`EnumWindows` is the supported way to enumerate top-level desktop-app windows and is safer than walking handles with `GetWindow` ([EnumWindows](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumwindows)). For a crawl target, reject Gary's HWNDs, invisible/minimized windows, zero-area rectangles, tool surfaces, and DWM-cloaked windows. Read visible frame bounds using `DwmGetWindowAttribute(DWMWA_EXTENDED_FRAME_BOUNDS)` and use `DWMWA_CLOAKED` to exclude shell/virtual-desktop-hidden surfaces ([DWM attributes](https://learn.microsoft.com/en-us/windows/win32/api/dwmapi/ne-dwmapi-dwmwindowattribute)). `GetWindowRect` is DPI-virtualized and may include invisible resize borders, so it is only a fallback ([GetWindowRect](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getwindowrect)).

True “behind exactly this foreign window but above everything below it” is not stable: activation continuously changes the shared z-order, and the topmost band is separate. The supported, deterministic illusion is to keep Gary in his own overlay and clip his visible region. `SetWindowRgn` guarantees pixels outside Gary's window region are not displayed ([SetWindowRgn](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowrgn)); renderer-side clipping is also acceptable.

### Virtual desktops

Windows assigns every top-level window to a virtual desktop and hides it when that desktop is inactive. The public `IVirtualDesktopManager` can query whether a window is on the current desktop and can get/move a window's desktop ID ([IVirtualDesktopManager](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nn-shobjidl_core-ivirtualdesktopmanager)). Its documented contract does not provide desktop-switch notifications.

For v0.1, Gary is **desktop-local**: his visible HWND may disappear when the user switches virtual desktops and return when they switch back. Do not call undocumented virtual-desktop interfaces, force a desktop switch, or promise simultaneous Gary overlays on every virtual desktop. Filter crawl targets with `IsWindowOnCurrentVirtualDesktop` when available ([method contract](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nf-shobjidl_core-ivirtualdesktopmanager-iswindowoncurrentvirtualdesktop)).

## Desktop icons: supported APIs exist, but Gary should not use them

Windows exposes `IFolderView::GetItemPosition` and `SelectAndPositionItems`; Microsoft explicitly identifies `SelectAndPositionItems` as the supported alternative to sending messages to Explorer's undocumented list-view implementation ([IFolderView](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nn-shobjidl_core-ifolderview), [Microsoft engineering guidance](https://devblogs.microsoft.com/oldnewthing/20211122-00/?p=105948)). That makes real icon access technically possible, not appropriate.

Scary Gary's contract is stricter than the platform's capability. v0.1 must not obtain the desktop folder view, enumerate PIDLs, inspect item names/images, reposition items, toggle Explorer redraw, or persist icon coordinates. A bundled fake icon provides the joke with no user-data access and remains correct through Explorer restarts, OneDrive-backed desktops, auto-arrange, and layout changes.

## Focus, input, and Garynap

The overlay foundation above keeps normal mouse input working and prevents focus theft. Use `GetAsyncKeyState(VK_ESCAPE)` only to observe the high-order “currently down” bit, with edge detection/debounce, so Esc can trigger Garynap without a global keyboard hook or registered system-wide hotkey. Microsoft documents the low-order “pressed since last call” bit as unreliable ([GetAsyncKeyState](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getasynckeystate)). Gary must not consume the keystroke; Esc continues to reach the foreground application.

If Gary's process is ended in Task Manager, all overlays and audio end immediately. Any future per-user sign-in launch is a separate lifecycle concern; desktop integration requires no watchdog, service, scheduled task, elevation, or hidden respawn.

## Acceptance probes for implementation

Before calling the illusions complete, verify on Windows 10 and Windows 11 that:

- the cursor can click, drag, scroll, and invoke taskbar controls through every ordinary Gary sprite and through the Gary Event overlay;
- focus and foreground HWND do not change when Gary appears, moves, or disappears;
- taskbar walking safely falls back under bottom/top/side, auto-hide, Explorer restart, and secondary-monitor configurations available in the test lab;
- cursor tracking, peeking, and sprinting stay on the correct monitor with negative coordinates and mixed 100%/150% DPI;
- crawl clipping fails closed when targets move, minimize, close, cloak, or use always-on-top;
- switching virtual desktops never forces a switch or leaks a stale crawl target;
- fake icon and garbage events do not read or mutate the actual Desktop folder/view;
- the Gary Event never changes system display/audio settings and yields to UAC, lock screen, Task Manager, exclusive full screen, and other higher surfaces.
