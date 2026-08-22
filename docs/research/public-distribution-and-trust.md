# Public distribution and trust research

_Current as of 2026-08-21. This note separates primary-source evidence from recommendations for [Issue #9](https://github.com/scwlkr/scary-gary/issues/9). It is technical/product guidance, not legal advice._

## Evidence

### GitHub releases and integrity

- A GitHub Release is based on a Git tag, can carry uploaded binaries, and automatically exposes source ZIP and tar archives for the tagged tree ([GitHub: About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)).
- Generated source archives are recreated on request. Their extracted files remain stable while the ref and repository name remain stable, but compression bytes can change; GitHub recommends release assets when archive security matters ([GitHub: Downloading source code archives](https://docs.github.com/en/repositories/working-with-files/using-files/downloading-source-code-archives)). They also cannot be checked with `gh release verify-asset` because they are generated on demand ([GitHub: Verifying release integrity](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity)).
- With immutable releases enabled, the published tag and uploaded assets cannot be changed, and GitHub creates a release attestation covering the tag, commit, and assets. GitHub recommends assembling a draft and publishing it only after every asset is attached ([GitHub: Immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases)). Release-asset API responses also expose a SHA-256 digest ([GitHub Releases REST API](https://docs.github.com/en/rest/releases/releases)).

### License and provenance

- A public GitHub repository without a license remains under default copyright; GitHub recommends a license file in the repository root. Making a repository private later does not retract forks or local copies already made ([GitHub: Licensing a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/licensing-a-repository)).
- The OSI-approved MIT text permits reuse subject to retaining its copyright and permission notice in copies or substantial portions, and its SPDX identifier is `MIT` ([OSI: MIT License](https://opensource.org/license/mit), [SPDX License List](https://spdx.org/licenses/)). GitHub's detected repository license does not account for dependency licenses or other notices ([GitHub Licenses REST API](https://docs.github.com/en/rest/licenses/licenses)).

### .NET 10 source build and single-file publish

- A self-contained publish carries the runtime, so the target PC does not need a separate .NET installation, but the publisher must rebuild to deliver later runtime security fixes. Single-file output is OS/architecture-specific; `win-x64`, `SelfContained`, and `PublishSingleFile` are therefore material inputs ([Microsoft: .NET publishing overview](https://learn.microsoft.com/en-us/dotnet/core/deploying/), [Microsoft: Single-file deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/single-file/overview)).
- Microsoft recommends setting `PublishSingleFile` in the project so build-time compatibility warnings run. Native runtime libraries require `IncludeNativeLibrariesForSelfExtract=true` to obtain one output executable; those native files then extract beneath `%TEMP%\.net` at launch. PDBs remain separate unless embedded with `DebugType=embedded` ([Microsoft: Single-file deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/single-file/overview)).
- Building requires a .NET 10 SDK; a self-contained consumer does not require a runtime. `global.json` controls SDK selection, while `dotnet restore --locked-mode` fails if a committed package lock would change ([Microsoft: `global.json`](https://learn.microsoft.com/en-us/dotnet/core/tools/global-json), [Microsoft: locked restore](https://learn.microsoft.com/en-us/nuget/reference/errors-and-warnings/nu1512)).

### Current Windows support

- Microsoft's current .NET 10 matrix supports x64 on Windows 11 26H1, 25H2, 24H2, and 23H2 Enterprise/Education, plus Windows 10 21H2, 1809, and 1607 LTSC/Enterprise editions. Microsoft explicitly says Windows 10 support is limited to LTSC and Enterprise editions ([Microsoft: Install .NET on Windows](https://learn.microsoft.com/en-us/dotnet/core/install/windows), [.NET 10 supported OS list](https://github.com/dotnet/core/blob/main/release-notes/10.0/supported-os.md)).
- Consequently, the roadmap's unqualified “Windows 10/11 x64” target cannot be an unqualified Microsoft-supported .NET 10 promise. Consumer Windows 10 22H2 is absent from the supported matrix even if the executable happens to run there.

### Authenticode, timestamps, and reputation

- Authenticode identifies the publisher and detects post-signing changes. Microsoft recommends SHA-256 file digests and an RFC 3161 SHA-256 timestamp (`/fd SHA256 /tr ... /td SHA256`); timestamping preserves signature validity after the signing certificate expires ([Microsoft: Time Stamping Authenticode Signatures](https://learn.microsoft.com/en-us/windows/win32/seccrypto/time-stamping-authenticode-signatures), [Microsoft: SignTool](https://learn.microsoft.com/en-us/windows/win32/seccrypto/signtool)).
- A valid OV/EV signature can still receive an initial SmartScreen warning. Unsigned and self-signed binaries do not carry transferable publisher reputation; EV no longer bypasses reputation checks. Microsoft recommends signing every release with a consistent identity, warns against modifying files after signing, and notes that Smart App Control can block unsigned files on Windows 11 ([Microsoft: SmartScreen reputation](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)).

### Safe download and prank-software expectations

- `Invoke-WebRequest -OutFile` downloads without executing, and `Get-FileHash -Algorithm SHA256` can compare the result with a published hash ([Microsoft: Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest), [Microsoft: Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)). Microsoft discourages download-to-`Invoke-Expression` patterns because they execute fetched text as code ([Microsoft: Avoid using Invoke-Expression](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/avoid-using-invoke-expression)).
- curl does not treat HTTP 4xx/5xx as failure by default; `--fail`, `--location`, HTTPS-only protocol restrictions, and an explicit output path provide safer download behavior. `--insecure` disables certificate verification and enables interception risk ([curl man page](https://curl.se/docs/manpage.html), [curl TLS FAQ](https://curl.se/docs/faq.html#Why_do_I_get_certificate_verify_failed)).
- Microsoft's unwanted-software criteria require prominent notice of purpose and activity, consent before installation, the ability to start/stop/revoke authorization, and straightforward install, disable, and removal paths. Hidden presence, misleading behavior, or poor removal can cause classification as unwanted software ([Microsoft: Malware and PUA criteria](https://learn.microsoft.com/en-us/unified-secops/criteria)).

## Recommendations for v0.1

### Release contract

1. Keep the repository public and add the verbatim MIT license as root `LICENSE`, naming the actual rights holder and year. Record creator/source/license provenance for every image, sound, font, and dependency; use `THIRD_PARTY_NOTICES` or per-asset licenses where MIT does not apply. Do not publish an asset until the project has the right to redistribute it.
2. Publish an immutable, version-tagged GitHub Release assembled as a draft. Attach final `gary.exe` and `SHA256SUMS.txt`. GitHub's generated source ZIP/tar are sufficient for ordinary source access; omit a redundant binary ZIP and attach a separate source archive only if stable archive bytes become a requirement.
3. Build, sign and timestamp the final EXE, verify its signature, then calculate hashes from those final bytes. Generate a GitHub artifact attestation when CI builds the release. Treat checksums as corruption/identity evidence from the release channel, not as a substitute for Authenticode publisher identity.
4. Make Authenticode signing with one stable, publicly trusted publisher identity a stable-v0.1 release gate. If signing is temporarily unavailable, publish only a prominently labeled unsigned prerelease, state that SmartScreen or Smart App Control may warn/block, and never instruct users to disable Defender or bypass policy.

### Source-build path

Commit a .NET 10 `global.json`, package lock, and the publish properties in the WPF project. Document a Windows CLI path equivalent to:

```powershell
git checkout <release-tag>
dotnet restore <project> -r win-x64 --locked-mode
dotnet test -c Release --no-restore
dotnet publish <project> -c Release -r win-x64 --self-contained --no-restore -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true
```

The release process should name the SDK version, source commit, output hash, and exact test matrix, and should republish after relevant .NET 10 security servicing. Upload only `gary.exe`; retain symbols as a separate debugging artifact if they are not embedded. Document that a local unsigned build is inspectable but will not be byte-identical to the signed release.

### Windows promise

Replace the map's broad platform statement with: **“Supported on x64 editions of Windows 11 and Windows 10 that Microsoft and .NET 10 currently support. Consumer Windows 10 compatibility is best-effort and listed per release.”** For v0.1 release notes, enumerate the exact Windows builds tested. This keeps Windows 10 in scope without implying Microsoft support for consumer 22H2.

### Download commands

Use a version-pinned HTTPS asset URL and a release-specific expected hash. Commands should download and verify but never execute, enable Startup, unblock, or suppress security warnings. Substitute the real tag and final hash at release time:

```powershell
$u='https://github.com/scwlkr/scary-gary/releases/download/<tag>/gary.exe'; $p=Join-Path $PWD 'gary.exe'; Invoke-WebRequest -Uri $u -OutFile $p -ErrorAction Stop; if ((Get-FileHash -LiteralPath $p -Algorithm SHA256).Hash -ne '<SHA256>') { Remove-Item -LiteralPath $p; throw 'gary.exe checksum mismatch' }
```

```powershell
curl.exe --fail --location --proto "=https" --proto-redir "=https" --tlsv1.2 --output gary.exe https://github.com/scwlkr/scary-gary/releases/download/<tag>/gary.exe
Get-FileHash -LiteralPath .\gary.exe -Algorithm SHA256
```

Do not offer `irm | iex`, `iwr | iex`, curl-to-shell, `/latest/` with a version-specific hash, `--insecure`, `Unblock-File`, or automated “Run anyway” instructions.

### Disclosure and removability

Preserve surprise by withholding exact action timing and Gary Event staging, not by hiding material capability. The repository, release page, and explicit Startup-enablement flow should disclose:

- this is a harmless desktop prank with audiovisual interruptions and a default-on rare scare;
- optional per-user Startup persistence is added only by `gary.exe --enable-startup`;
- no administrator access, private-file access, telemetry, runtime network service, security-tool interference, or watchdog resurrection;
- Gary remains visible in Task Manager and can be stopped there;
- `gary.exe --disable-startup` disables future sign-in launch without ending the current session;
- `gary.exe --remove` disables Startup, stops the current session, and clears Gary-owned state while deliberately leaving the user-owned portable executable in place;
- `gary.exe --rehearse-event` immediately rehearses the otherwise rare Gary Event; and
- use and Startup enablement are only for an account/device whose owner has authorized the prank.

There is no install/copy/self-delete command and no conventional uninstall entry: the downloaded executable remains a normal user-owned portable file. These commands implement the ownership and cleanup boundary settled by [Issue #8](https://github.com/scwlkr/scary-gary/issues/8); Issue #9 should name and disclose them without defining a second lifecycle model.
