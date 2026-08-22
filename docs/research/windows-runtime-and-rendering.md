# Windows runtime and rendering architecture

Research snapshot: 2026-08-21. Sources are current Microsoft documentation and first-party project documentation/source.

## Decision

Build Scary Gary v0.1 as an **unpackaged, self-contained WPF application on .NET 10 LTS for `win-x64`**, distributed as one `gary.exe`, with a small, explicit Win32 interop layer.

Use one small borderless transparent WPF `Window` per independent Gary/sign/junk overlay and create monitor-sized overlay windows only for a Gary Event. Drive sprite motion from elapsed time on `CompositionTarget.Rendering`; use transforms rather than layout churn. Get each window's `HWND` through `WindowInteropHelper`, then use Win32 only for cursor/monitor geometry, Z-order, no-activate behavior, and dynamic click-through.

.NET 10 is an active LTS release through November 2028, and WPF is part of its Windows Desktop runtime ([support policy](https://dotnet.microsoft.com/en-us/platform/support/policy), [WPF `WindowInteropHelper` applies through Windows Desktop 10](https://learn.microsoft.com/en-us/dotnet/api/system.windows.interop.windowinterophelper?view=windowsdesktop-10.0)). This is the shortest maintained route to Gary's defining feature: a per-pixel transparent, non-rectangular desktop creature.

## Feasibility matrix

Legend: **Yes** = direct and suitable; **Interop** = feasible through documented Windows APIs; **Weak** = feasible but a poor fit; **Blocker** = no supported first-class path for Gary's requirement.

| Required capability | WPF / .NET 10 | WinUI 3 / Windows App SDK 1.8 | WinForms / .NET 10 |
| --- | --- | --- | --- |
| Transparent shaped windows | **Yes.** `WindowStyle=None`, `AllowsTransparency=True`, and a transparent background are the documented non-rectangular-window path ([WPF windows](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/windows/), [`AllowsTransparency`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.window.allowstransparency?view=windowsdesktop-10.0)). | **Blocker.** WinUI's own repository still records top-level/island output transparency as unsupported; layered-window attributes do not fix its composition surface ([open proposal](https://github.com/microsoft/microsoft-ui-xaml/issues/11134)). | **Weak.** `TransparencyKey` gives color-key transparency, not the clean per-pixel alpha needed by antialiased PNG sprites ([API](https://learn.microsoft.com/en-us/dotnet/api/system.windows.forms.form.transparencykey?view=windowsdesktop-10.0)). A custom layered-window renderer would erase WinForms' simplicity advantage. |
| Dynamic click-through | **Interop.** Toggle `WS_EX_TRANSPARENT` on the Gary HWND while passive and remove it when interaction is intended. Windows documents that mouse events pass to windows underneath a layered window with that style ([layered windows](https://learn.microsoft.com/en-us/windows/win32/winmsg/window-features)). | **Weak.** The HWND is available, but input pass-through does not solve the unsupported transparent output surface ([retrieve HWND](https://learn.microsoft.com/en-us/windows/apps/develop/ui/retrieve-hwnd)). | **Interop.** Same Win32 technique, with the alpha-rendering limitation above. |
| High-frequency animation | **Yes.** `CompositionTarget.Rendering` is called once per rendered frame; keep overlay surfaces tight and animate transforms ([rendering event](https://learn.microsoft.com/en-us/dotnet/api/system.windows.media.compositiontarget.rendering?view=windowsdesktop-10.0), [WPF performance guidance](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/optimizing-performance-other-recommendations)). | **Yes, strongest animation system.** Independent animations run on a separate composition thread ([WinUI animation guidance](https://learn.microsoft.com/en-us/windows/apps/develop/performance/optimize-animations-and-media)), but the transparency blocker makes that advantage unusable for v0.1. | **Weak.** Timers plus double buffering are viable for crude motion, but this is the least natural smooth-alpha sprite path ([double-buffered graphics](https://learn.microsoft.com/en-us/dotnet/desktop/winforms/advanced/double-buffered-graphics)). |
| Multiple overlays | **Yes.** Each WPF `Window` has a top-level HWND, and WPF provides direct hooks for otherwise-unmapped Win32 messages ([WPF/Win32 interop](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/wpf-and-win32-interoperation), [`WindowInteropHelper`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.interop.windowinterophelper?view=windowsdesktop-10.0)). | **Yes.** WinUI supports multiple `Window`/`AppWindow` instances on one UI thread ([multiple windows](https://learn.microsoft.com/en-us/windows/apps/develop/ui/multiple-windows)). | **Yes.** Multiple forms are straightforward, subject to the rendering weakness above. |
| Full-screen presentation | **Yes.** Create one borderless overlay per monitor and place it with `SetWindowPos`; monitor and work rectangles are available in virtual-screen coordinates ([`MONITORINFO`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-monitorinfo), [`SetWindowPos`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowpos)). | **Yes.** `AppWindowPresenter` supports full-screen presentation ([windowing overview](https://learn.microsoft.com/en-us/windows/apps/develop/ui/windowing-overview)). | **Yes.** Borderless monitor-sized forms plus the same Win32 geometry. |
| Audio | **Yes.** For a one-file build, embed short WAV assets and play their streams asynchronously with `SoundPlayer`; it explicitly supports embedded resources/streams ([`SoundPlayer`](https://learn.microsoft.com/en-us/dotnet/api/system.media.soundplayer?view=windowsdesktop-10.0)). | **Yes.** `MediaPlayer` supports local files and memory streams ([media playback](https://learn.microsoft.com/en-us/windows/apps/develop/media-playback/play-audio-and-video-with-mediaplayer)). | **Yes.** The same `SoundPlayer` path. |
| Per-monitor DPI | **Interop.** Declare PerMonitorV2 awareness, handle WPF `DpiChanged`/`WM_DPICHANGED`, and keep Win32 pixel coordinates separate from WPF DIPs ([WPF DPI guidance](https://learn.microsoft.com/en-us/windows/win32/hidpi/declaring-managed-apps-dpi-aware), [`DpiChanged`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.window.dpichanged?view=windowsdesktop-10.0)). This must be tested; Microsoft notes WPF does not handle every per-monitor case automatically ([desktop guidance](https://learn.microsoft.com/en-us/windows/apps/get-started/best-practices#dpi-awareness)). | **Yes.** WinUI automatically scales for each display ([desktop guidance](https://learn.microsoft.com/en-us/windows/apps/get-started/best-practices#dpi-awareness)). | **Interop.** PerMonitorV2 is possible but requires explicit configuration and testing. |
| Cursor tracking | **Interop.** Poll `GetCursorPos` in screen coordinates; no global hook is needed ([API](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getcursorpos)). | **Interop.** Same API through the retrieved HWND/native boundary. | **Interop.** Same API. |
| Native interop | **Yes.** `WindowInteropHelper` exposes the HWND and `HwndSource.AddHook` receives native messages ([interop API](https://learn.microsoft.com/en-us/dotnet/api/system.windows.interop.windowinterophelper?view=windowsdesktop-10.0)). | **Yes.** `WindowNative.GetWindowHandle` exposes the HWND ([retrieve HWND](https://learn.microsoft.com/en-us/windows/apps/develop/ui/retrieve-hwnd)). | **Yes.** Forms are HWND-backed. |
| Per-user Startup registration | **Yes; stack-neutral.** An explicit install command can add `gary.exe` to `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`; uninstall removes it. Windows runs `Run` entries at each sign-in, possibly delayed, with no machine-wide key or administrator privilege needed ([Run keys](https://learn.microsoft.com/en-us/windows/win32/setupapi/run-and-runonce-registry-keys)). | **Yes only in the unpackaged model** recommended for one-file distribution; use the same Win32 registration. | **Yes.** Same registration. |
| Crash recovery | **Partial; stack-neutral.** `RegisterApplicationRestart` can offer the user a restart after an unhandled exception or hang, and Windows suppresses restart loops for apps that ran under 60 seconds. It is consent-based, not an unattended watchdog ([API semantics](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-registerapplicationrestart)). The HKCU entry starts the app again at the next sign-in. | **Partial.** Same Windows API and limits. | **Partial.** Same Windows API and limits. |
| Self-contained one-file distribution | **Yes, best fit.** .NET single-file publishing is RID-specific and can bundle native runtime libraries with `IncludeNativeLibrariesForSelfExtract`; embed Gary's PNG/WAV assets and PDB ([single-file deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/single-file/overview)). | **Yes, but heavier.** Only unpackaged, self-contained WinUI apps support it; Windows App SDK requires full-content extraction and a larger bundled runtime ([WinUI unpackaged deployment](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/unpackage-winui-app#single-file-exe)). | **Yes.** Uses the same simpler .NET single-file model as WPF. |

## Packaging model

Use these release properties as the baseline:

```xml
<TargetFramework>net10.0-windows</TargetFramework>
<UseWPF>true</UseWPF>
<RuntimeIdentifier>win-x64</RuntimeIdentifier>
<SelfContained>true</SelfContained>
<PublishSingleFile>true</PublishSingleFile>
<IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
<DebugType>embedded</DebugType>
```

This means **one file to distribute**, not necessarily zero extraction: bundled native libraries can be extracted under `%TEMP%/.net` at launch ([single-file native libraries](https://learn.microsoft.com/en-us/dotnet/core/deploying/single-file/overview#native-libraries)). Do not enable trimming for v0.1; WPF's first-party tracker still marks trimming compatibility unresolved ([dotnet/wpf #3811](https://github.com/dotnet/wpf/issues/3811)). Compression is optional only after measuring its startup cost ([single-file compression](https://learn.microsoft.com/en-us/dotnet/core/deploying/single-file/overview#compress-assemblies-in-single-file-apps)).

One current first-party report describes WPF single-file/BAML startup regressions in SDK 10.0.200/10.0.202 ([dotnet/wpf #11678](https://github.com/dotnet/wpf/issues/11678)). It is open and untriaged, so treat it as a validation risk rather than a proven platform-wide defect: pin the SDK version that passes the publish smoke test and run the published EXE from paths with spaces before accepting the foundation.

## Architecture boundaries

- **WPF owns pixels and assets.** Transparent `Window`s contain only sprite/sign visuals. Keep assets embedded so the executable is portable.
- **Win32 owns desktop facts.** A narrow adapter owns HWND styles, `GetCursorPos`, monitor/work-area rectangles, DPI conversion, and Z-order. `MONITORINFO.rcWork` provides the taskbar-reserved work area without reading desktop files.
- **One coordinate authority.** Simulation state uses virtual-screen physical pixels. Convert at the WPF window edge using that window's current DPI; never mix `Window.Left`/`Top` DIPs with Win32 pixels implicitly.
- **Input mode is explicit.** Passive overlays are no-activate and click-through; interactive moments temporarily remove click-through. Transparent pixels should never create an invisible input wall.
- **Crash handling is not persistence.** Catch/log recoverable failures and use Windows restart registration as best effort. Automatic unattended recovery would be a separate supervisor design, not a capability supplied by the selected UI runtime.

## Alternatives screened out

- **WinUI 3** is maintained, runs on Windows 10 1809 and later, and has the best compositor ([Windows App SDK overview](https://learn.microsoft.com/en-us/windows/apps/windows-app-sdk/)). Its unsupported transparent top-level surface is nevertheless a direct conflict with Gary's core rendering requirement.
- **WinForms** is maintained and fast to scaffold, but color-key transparency degrades Gary's PNG edges; solving that with a custom layered renderer forfeits the simplicity benefit.
- **Avalonia 12** is credible and supports transparent Windows backgrounds, render-thread animation, per-monitor scaling, and HWND interop ([Windows backend](https://docs.avaloniaui.net/docs/platform-specific-guides/windows), [composition animation](https://docs.avaloniaui.net/docs/graphics-animation/composition-animations)). Its own docs say transparent click-through still requires native platform APIs ([window guidance](https://docs.avaloniaui.net/docs/how-to/window-how-to#window-transparency)). Cross-platform support is not a v0.1 goal, so the extra framework/compositor is cost without product value.
- **Raw Win32 + Direct2D** offers maximum control and a true native single EXE ([Direct2D overview](https://learn.microsoft.com/en-us/windows/win32/direct2d/direct2d-overview)), but would require substantially more windowing, rendering, resource, and lifetime code. Keep it only as an escalation if a measured WPF overlay spike misses performance targets.

## Acceptance evidence for the foundation

Before building the behavior systems, require a tiny technical spike to prove all high-risk seams together:

1. Release publish produces only `gary.exe`, and that file starts on a clean Windows 11 x64 machine with no .NET installation.
2. Two animated transparent overlays preserve PNG alpha edges and can change Z-order independently.
3. Click-through can be toggled at runtime without focus theft; the underlying app receives clicks while Gary is passive.
4. Cursor following and taskbar/work-area positioning remain correct on two monitors with mixed DPI and negative virtual-screen coordinates.
5. A monitor-sized overlay presents and disappears cleanly, embedded WAV playback works, process exit is clean, and the per-user Startup entry can be added and removed without elevation.

Windows 10 needs an explicit support statement: as of this snapshot, .NET 10 officially supports only still-supported Windows 10 LTSC/Enterprise releases, not consumer Windows 10 22H2 ([current Windows support table](https://learn.microsoft.com/en-us/dotnet/core/install/windows#supported-versions)). A consumer Windows 10 build may still run, but it must be described and tested as best-effort rather than Microsoft-supported.
