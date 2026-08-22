---
status: accepted
---

# Publish one signed, inspectable, explicitly enabled Gary

Scary Gary v0.1 will remain public under the MIT License and ship from a versioned GitHub Release as one Authenticode-signed and timestamped `gary.exe` plus `SHA256SUMS.txt`; GitHub's generated source archives are sufficient, so v0.1 will not add a redundant binary ZIP. This spends signing effort and reveals the prank category because a default-scary executable that can start at sign-in must be easier to inspect, authenticate, stop, and remove than it is to persist by accident.

## Consequences

- A final v0.1 release is blocked until its executable is signed and timestamped. An unsigned build may be labeled and published only as a preview, with no promise that Windows SmartScreen will accept either build without a warning.
- MIT covers repository-owned code, documentation, art, and sound. Every media asset and dependency needs recorded provenance and redistribution rights before it may ship.
- Published PowerShell and `curl.exe` one-liners download a pinned version but never execute it. Running Gary directly is a non-persistent Portable Run; `gary.exe --enable-startup` and `gary.exe --disable-startup` control the per-user Gary Startup promise, and `gary.exe --remove` disables Startup and clears Gary-owned state while leaving the user-owned executable. Issue #8 owns the lifecycle mechanics behind those commands.
- Supported editions are the x64 Windows 11 and Windows 10 editions still supported by both Microsoft and .NET 10 when a release ships. Consumer Windows 10 22H2 x64 remains a required, tested, best-effort compatibility target because it is outside current Microsoft and .NET support.
- The README and every release disclose before download that Gary is a harmless desktop prank with a rare default-on full-screen visual-and-sound scare, no administrator/runtime-network/telemetry/private-file behavior, ordinary Task Manager stopping, and documented removal. Exact Gary Event choreography remains behind an explicit spoiler boundary.
- Release acceptance must verify the signature and timestamp, SHA-256 manifest, clean-machine run, command documentation, pinned .NET 10 source build, disclosure, and removal behavior; checksums prove artifact identity, not publisher identity.
