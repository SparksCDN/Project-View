# Cross-platform desktop stacks for a read-only GitHub issue client

Research for [#2](https://github.com/SparksCDN/Project-View/issues/2). Feeds the decision in
[#5](https://github.com/SparksCDN/Project-View/issues/5) — **this document deliberately does not pick a winner.**

All facts gathered **2026-08-12**. Version numbers move fast (Electron ships weekly; Wails v3 moved two beta
releases during this research). Re-check anything version-sensitive before acting on it.

---

## 1. How to read this document

Claims are graded:

- **Measured** — I ran the command or fetched the byte count on this machine today. Reproducible.
- **Primary** — stated in the vendor's own docs, repo, or release notes. URL given.
- **Third-party** — someone else's blog or benchmark. Treated as weak.
- **Unverified** — I could not establish it from a source I trust. Said so rather than guessed.

The largest honest gap in this report is **§7 (runtime performance)**. Startup time, idle memory and packaged
bundle size are the numbers everyone quotes and almost nobody sources. No vendor here publishes measured
figures for any of them. See §7 for what that means.

---

## 2. Verified starting state of this machine

Measured on 2026-08-12:

| Fact | Value |
|---|---|
| OS | macOS 26.5.2 (build 25F84) |
| Architecture | `arm64` (Apple Silicon) |
| Homebrew | present, `/opt/homebrew/bin/brew` |
| Python | 3.14.3, `/Library/Frameworks/Python.framework/Versions/3.14` |
| Swift | 6.0.3 (ruled out — not cross-platform) |
| Xcode | 16.2 (Build 16C5032a), command-line tools present |
| Node / npm / pnpm / bun | **absent** |
| Rust / cargo | **absent** |
| Go | **absent** |
| .NET | **absent** |

Xcode being already installed matters more than it looks. Every one of these six stacks needs the Xcode
command-line tools to produce a macOS build, and several need full Xcode to sign. That cost is already paid.

### Toolchain download sizes (measured today, via HTTP `Content-Length`)

| Toolchain | Artefact | Size |
|---|---|---|
| Node 24.19.0 LTS | `node-v24.19.0-darwin-arm64.tar.gz` | 52,234,372 B ≈ **50 MB** |
| Go 1.26.5 | `go1.26.5.darwin-arm64.tar.gz` | **61.7 MB** |
| .NET SDK 10.0.400 | `dotnet-sdk-10.0.400-osx-arm64.pkg` | 225,591,234 B ≈ **215 MB** |
| Flutter 3.47.0 | `flutter_macos_arm64_3.47.0-stable.zip` | 2,257,377,039 B ≈ **2.10 GiB** |
| PySide6 6.11.1 | `PySide6-Essentials` universal2 wheel | **105.2 MB** |
| PySide6 6.11.1 | `PySide6-Addons` universal2 wheel (optional) | **316.3 MB** |
| Rust | rustup 1.29.0 → stable channel 2026-07-16 (rustc 1.97.1, cargo 0.98.0) | size not published as a single artefact — **unverified** |

Sources: [nodejs.org/dist](https://nodejs.org/dist/index.json),
[go.dev/dl](https://go.dev/dl/?mode=json),
[.NET 10 release metadata](https://builds.dotnet.microsoft.com/dotnet/release-metadata/10.0/releases.json),
[Flutter macOS releases JSON](https://storage.googleapis.com/flutter_infra_release/releases/releases_macos.json),
[PySide6-Essentials on PyPI](https://pypi.org/pypi/PySide6-Essentials/json),
[PySide6-Addons on PyPI](https://pypi.org/pypi/PySide6-Addons/json),
[Rust stable channel manifest](https://static.rust-lang.org/dist/channel-rust-stable.toml).

Flutter's 2.1 GiB SDK is **40× the Node download and 10× the .NET SDK**. It is the single largest install-cost
outlier in the field by a wide margin.

### Homebrew availability (measured with `brew info --json=v2`)

`node` 26.7.0 (formula), `go` 1.26.5 (formula), `rustup` 1.29.0 (formula), `dotnet-sdk` 10.0.400 (cask),
`flutter` 3.47.0 (cask, not deprecated). Every toolchain here is one `brew install` away. Note Homebrew's
`node` formula tracks *Current* (26.x), not LTS (24.x) — Electron's `package.json` declares
`engines: { node: ">= 22.12.0" }`
([registry.npmjs.org/electron/43.4.0](https://registry.npmjs.org/electron/43.4.0)), so either works.

---

## 3. Costs every candidate shares

Worth stating once so it does not distort the comparison.

- **Apple Developer Program, $99/year, for macOS distribution.** Tauri's docs state it plainly: an Apple
  Developer account "which is either paid (99$ per year) or on the free plan", and the free plan
  **cannot notarize**. ([Tauri macOS signing](https://v2.tauri.app/distribute/sign/macos/)) This applies
  identically to Electron, Wails, Flutter, Avalonia and PySide6 — anything shipped outside the App Store to
  another Mac needs Developer ID signing + notarisation, or the recipient meets Gatekeeper.
  For a personal tool run only on the machine that built it, ad-hoc signing suffices and this cost is $0.
- **Windows code signing.** All stacks need it to avoid SmartScreen warnings. I did not verify current
  certificate pricing — **unverified**.
- **Linux is unsigned** in practice across all six.
- **A GitHub API client.** Availability differs sharply by language; see §9.

---

## 4. The six candidates

### 4.1 Electron

**Current state.** Stable **43.4.0**, published 2026-08-11. 44.0.0-beta.3 in beta. Supported lines today:
43.4.0, 42.9.0, 41.10.5 — Electron supports "the latest three stable major versions" on an **8-week major
cadence**, with weekly-ish patch releases across all three lines. Backwards compatibility is held for a
minimum of two majors.
([release timelines](https://www.electronjs.org/docs/latest/tutorial/electron-timelines),
[releases API](https://api.github.com/repos/electron/electron/releases))

**Setup from baseline.** Install Node (brew or tarball, ~50 MB), then `npm i -D electron`. That is the whole
toolchain — Xcode CLT is already present. **Shortest path in the field.** Rough time: minutes plus npm install.

**Distribution.** Electron's own docs recommend **Electron Forge**
([application distribution](https://www.electronjs.org/docs/latest/tutorial/application-distribution)).
electron-builder is the widely-used alternative (14,640 stars, pushed 2026-08-12).
The prebuilt runtime for v43.4.0 is **116 MB compressed** for `darwin-arm64`, 119 MB `linux-x64`,
137 MB `win32-x64` (measured from GitHub release asset sizes). A packaged `.app` is larger than the
compressed zip; the unpacked figure is **unverified** — Electron's docs discuss ASAR and rebranding but say
nothing about bundle size.

**Linux.** Full Chromium ships with the app, so Linux renders **identically** to macOS and Windows. Given that
Linux will be built for but never tested, this is the strongest Linux-risk position of any candidate.
The caveat: **`autoUpdater` does not support Linux at all** — "Currently, only macOS and Windows are
supported", with the docs directing you to the distro package manager instead.
([autoUpdater](https://www.electronjs.org/docs/latest/api/auto-updater))

**Boring parts — the best-covered of the six.**

| Need | Electron answer | First-party? |
|---|---|---|
| HTTP | `net` module, on Chromium's network stack — gets system proxy, WPAD/PAC, HTTPS tunnelling, and NTLM/Kerberos/negotiate proxy auth for free ([net](https://www.electronjs.org/docs/latest/api/net)) | Yes |
| Credentials | `safeStorage` — **macOS Keychain**, **Windows DPAPI**, Linux kwallet 4–6 / gnome-libsecret ([safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage)) | Yes |
| Auto-update | `autoUpdater` via Squirrel.Mac / Squirrel.Windows / MSIX. macOS **requires** a signed app | Yes (macOS + Windows only) |
| System theme | `nativeTheme` — `shouldUseDarkColors`, `updated` event, plus high-contrast and reduced-transparency signals ([nativeTheme](https://www.electronjs.org/docs/latest/api/native-theme)) | Yes |

Two caveats on `safeStorage`: on Linux, "If no secret store is available, items stored in using the
`safeStorage` API will be **unprotected**" — encrypted with a hardcoded plaintext password, detectable via
`getSelectedStorageBackend()` returning `basic_text`. On Windows, DPAPI protects against other *users* but
"not from other apps running in the same userspace".

**Ecosystem.** 122,472 stars; 23,132 repos tagged `electron`; **22,297,787 npm downloads last month**;
15,275 Stack Overflow questions. Squirrel.Windows itself was last pushed **2024-07-24** and looks dormant,
though Electron 43 also supports MSIX updates.

---

### 4.2 Tauri

**Current state.** Stable **2.11.5**, published 2026-07-01; `tauri-cli` 2.11.4. No v3. Cadence is roughly
monthly patch/minor across a large workspace of crates.
([releases API](https://api.github.com/repos/tauri-apps/tauri/releases))

**Setup from baseline.** Rust via rustup, plus Xcode or the CLT (present). **Node is explicitly optional** —
the docs say Node.js is "only needed for JavaScript frontend frameworks"
([prerequisites](https://v2.tauri.app/start/prerequisites/)), and both `create-tauri-app` (4.7.3) and
`tauri-cli` (2.11.4) are installable from crates.io. A Node-free vanilla-HTML path genuinely exists.
Realistically, though, a UI worth building implies a JS framework implies Node — budget for both.
Rust compile times on first build are the real cost here, not the download.

**Distribution.** deb, RPM, AppImage, Flatpak, Snap, AUR on Linux; DMG / App Store on macOS; MSI / NSIS /
Microsoft Store on Windows ([distribute](https://v2.tauri.app/distribute/)). Tauri's size docs claim it
"by default provides very small binaries" and offer LTO/`opt-level='s'`/`strip` recipes, but
**publish no concrete numbers** ([app size](https://v2.tauri.app/concept/size/)).

**Linux — the weakest point, and Tauri documents it themselves.** Tauri uses the OS webview: WKWebView on
macOS, WebView2 (Chromium) on Windows, **WebKitGTK 4.1 on Linux**. Three different engines, one of which is
the untested platform. Their own AppImage guide warns:

> "Core libraries such as glibc frequently break compatibility with older systems... Building on a newer base
> system can raise the minimum glibc version required by your app, so when running on an older system, you may
> face a runtime error like `/usr/lib/libc.so.6: version 'GLIBC_2.33' not found`."

The recommended mitigation is to build in Docker or GitHub Actions on Ubuntu 22.04 / Debian 12 — the oldest
base that still carries `libwebkit2gtk-4.1-dev`. ([AppImage](https://v2.tauri.app/distribute/appimage/))
For "build it but never test it", CSS/JS that works in WKWebView and WebView2 may still break in WebKitGTK,
and nobody would find out.

**Boring parts.**

| Need | Tauri answer | First-party? |
|---|---|---|
| HTTP | `plugin-http` — re-exports `reqwest` in Rust, Web-standard `fetch` in JS. Requires per-origin allowlisting in the capability files; the default permission set pre-authorises **no** origins ([http plugin](https://v2.tauri.app/plugin/http-client/)) | Yes (official plugin) |
| Credentials | **No OS-keychain plugin.** Official plugin list is `autostart, barcode-scanner, biometric, cli, clipboard-manager, deep-link, dialog, fs, geolocation, global-shortcut, haptics, http, localhost, log, nfc, notification, opener, os, persisted-scope, positioner, process, shell, single-instance, sql, store, stronghold, updater, upload, websocket, window-state`. The closest is **Stronghold** — an IOTA-engine encrypted vault file requiring its own password, **not** Keychain/DPAPI/Secret Service ([stronghold](https://v2.tauri.app/plugin/stronghold/)); `store` is key-value persistence, not secure. See §8. | **No** |
| Auto-update | `plugin-updater` — AppImage on Linux, `.tar.gz` app bundle on macOS, MSI/NSIS on Windows, all with minisign-style signatures. **Linux is supported**, unlike Electron ([updater](https://v2.tauri.app/plugin/updater/)) | Yes |
| System theme | `tauri::Theme` in **core**, not a plugin — `Light`/`Dark`, available on macOS, Windows, Linux, Android, iOS ([docs.rs](https://docs.rs/tauri/latest/tauri/enum.Theme.html)) | Yes |

**Ecosystem.** 110,152 stars; 10,537 repos tagged `tauri`; 25,458,073 all-time crates.io downloads
(9,591,536 recent); 8,051,066 npm downloads last month for `@tauri-apps/cli`. Only **469** Stack Overflow
questions — see the caveat in §10.

---

### 4.3 Wails

**Current state — the most volatile in the field.** Stable **v2.14.0**, 2026-08-10. But
**v3.0.0-beta.0 landed 2026-08-02, ten days before this research**, and by 2026-08-12 it was already at
**beta.8** — eight betas in eleven days. The release notes are candid:

> "The desktop API is stable, but this is still a prerelease: test thoroughly before using it in a critical
> production deployment."

([v3.0.0-beta.0 release notes](https://github.com/wailsapp/wails/releases/tag/v3.0.0-beta.0))

This is a genuine fork in the road. v2 is stable but is the outgoing version. v3 has the better architecture
and features but is eleven days into beta with a v2→v3 migration guide that assumes a deliberate port.

**Setup from baseline.** Go (61.7 MB) + `go install github.com/wailsapp/wails/v3/cmd/wails3@latest` +
`wails3 setup`. npm is "optional... needed by most of the bundled templates". Xcode CLT already present.
Note a **documentation conflict**: the installation page says "Go (At least 1.24)" while the quick-start page
and the v3 status page both say **Go 1.25 or later**. Assume 1.25.
([installation](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/getting-started/installation.mdx),
[status/roadmap](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/status.mdx))
`wails3 setup` itself is flagged **experimental** and "primarily tested on Linux".

**Distribution.** `wails3 sign` / `wails3 task darwin:sign:notarize` wrap `codesign` and `notarytool`, with
`wails3 setup signing` as an interactive credential wizard. Cross-compiled macOS binaries are unsigned and
must be signed on a Mac.
([macOS build guide](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/guides/build/macos.mdx))
Bundle size: **unverified** — no published figures.

**Linux — a fresh discontinuity.** The v3 beta compatibility promise states Linux needs
**GTK4 + WebKitGTK 6.0 by default**, with "GTK3 + WebKit2GTK 4.1 remains a `-tags gtk3` legacy option through
v3.0.x and is **removed in v3.1**". GTK4/WebKitGTK 6.0 is newer than what Tauri targets (4.1), so Wails v3
raises the minimum Linux distro floor and then removes the fallback one minor version later. Supported
platform line is "Ubuntu 24.04 AMD64/ARM64 (other Linux may work too!)". Same three-different-webviews problem
as Tauri, on a shorter distro leash.

**Boring parts.**

| Need | Wails answer | First-party? |
|---|---|---|
| HTTP | Go's `net/http` standard library | Yes (language) |
| Credentials | **Nothing.** No keychain/secret feature anywhere in the v3 docs tree. Community Go libraries exist (`zalando/go-keyring`, `99designs/keyring`) — not evaluated here | **No** |
| Auto-update | **Surprisingly strong.** `app.Updater` is a built-in state machine with pluggable providers — **GitHub Releases**, keygen.sh, Sparkle AppCast, or a Wails Update Manifest — public-key verification, atomic binary swap, and a themeable default update window ([updater guide](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/guides/updater.mdx)) | Yes, v3 only, and the best in the field |
| System theme | Not found as a documented feature page in the v3 docs tree; the framework exposes window/system events but I could not confirm a documented theme API — **unverified** | Unclear |

**Ecosystem — the thinnest here.** 35,788 stars but only **862** repos tagged `wails` and **26** Stack Overflow
questions. For the "AI assistance must be strong" constraint this is the number to weigh: there is an order of
magnitude less public Wails material than Tauri, and two orders less than Electron — and most of what exists
is about v2, which v3 changes substantially.

---

### 4.4 Flutter desktop

**Current state.** Stable **3.47.0**, released **2026-08-12** (Dart 3.13.0); beta 3.48.0. Roughly monthly
stable releases. ([releases_macos.json](https://storage.googleapis.com/flutter_infra_release/releases/releases_macos.json))

**Setup from baseline.** The **2.10 GiB SDK download** dominates. `brew install --cask flutter` (3.47.0) or
the zip. Xcode CLT required; full Xcode required to sign and archive. Flutter's docs warn that Intel macOS is
deprecated and "Future Flutter releases will require Apple Silicon" — irrelevant here (already arm64), but it
signals where Flutter's desktop attention is.
([manual install](https://docs.flutter.dev/install/manual))
Windows builds additionally need Visual Studio; Linux builds need the GTK/clang/ninja toolchain (see below).

**Distribution.** Flutter documents full support for compiling native Windows, macOS and Linux desktop apps
([desktop support](https://docs.flutter.dev/platform-integration/desktop)). Bundle size: **unverified**.

**A macOS gotcha that will bite a GitHub client on day one.** Flutter macOS builds are "configured by default
to be signed and **sandboxed**". Outbound network requests require the
`com.apple.security.network.client` entitlement, or you get:

> `flutter: SocketException: Connection failed (OS Error: Operation not permitted, errno = 1), address = example.com, port = 443`

Entitlements live in `macos/Runner/Runner-DebugProfile.entitlements` *and*
`Runner-Release.entitlements`, and both must be edited.
([building macOS apps](https://docs.flutter.dev/platform-integration/macos/building))
This is a five-minute fix once you know, and an afternoon of confusion if you don't.

**Linux.** Documented runtime deps are modest — `libgtk-3-0 libblkid1 liblzma5` — with `ldd` on the built
bundle as the way to get the real list. Snap and deb/rpm packaging are documented; Flatpak is not.
([building Linux apps](https://docs.flutter.dev/platform-integration/linux/building))
Flutter renders everything itself via Impeller/Skia, so **the UI looks identical on all three platforms** —
which is either the point or the problem, depending on whether you want native-feeling chrome. Like Electron,
this means untested Linux is low-risk visually; unlike Electron, nothing about it will look like a GTK app.

**Boring parts.**

| Need | Flutter answer | First-party? |
|---|---|---|
| HTTP | `package:http` 1.6.0, published from `dart-lang/http` — Dart team owned ([pub.dev](https://pub.dev/packages/http)) | Yes |
| Credentials | `flutter_secure_storage` 11.0.0 (2026-08-06), `mogol/flutter_secure_storage` — popular, actively maintained, **community** | **No** |
| Auto-update | Nothing first-party found | **No** |
| System theme | `MediaQuery.platformBrightness` / `ThemeMode.system` — core framework | Yes |
| **Markdown rendering** | **`flutter_markdown` is discontinued**, marked `isDiscontinued: true`, `replacedBy: flutter_markdown_plus`, last release 2025-05-06 ([pub.dev API](https://pub.dev/api/packages/flutter_markdown)) | **No — Google dropped it** |

That last row matters more than it looks for *this* app. A GitHub issue client renders GitHub-Flavoured
Markdown constantly — issue bodies, comments, the map's Notes section. Google discontinued the official
Flutter Markdown package and handed it to a community fork.

**Ecosystem.** By far the largest raw numbers: 178,363 stars, 77,923 repos tagged `flutter`,
178,672 Stack Overflow questions. But **the overwhelming majority of that is mobile**. Desktop-specific
Flutter material is a small slice of a very large pie, and I have no way to size that slice —
**unverified**.

---

### 4.5 Avalonia

**Current state — note the two live lines.** **12.1.1** (2026-07-29) is current;
**11.3.20** (2026-08-10) is still receiving releases, so 11.x is actively maintained alongside 12.x.
Avalonia 12.0 shipped **2026-04-07**; 12.1 on **2026-07-08**. Release cadence is brisk on both lines.
([releases API](https://api.github.com/repos/AvaloniaUI/Avalonia/releases))

**A sourcing conflict I could not resolve.** The docs' supported-platforms page gives the desktop minimum as
**.NET 8.0** ([supported platforms](https://docs.avaloniaui.net/docs/supported-platforms)), while the
Avalonia 12 launch blog was read as saying the minimum target moves to **.NET 10**
([Avalonia 12 blog](https://avaloniaui.net/blog/avalonia-12)). A third summary of the same blog said .NET 8.
**Treat the minimum as unresolved; verify before relying on it.** It does not change the install cost here —
you'd install .NET 10 LTS regardless.

**Setup from baseline.** .NET SDK 10.0.400 (215 MB pkg, or `brew install --cask dotnet-sdk`), then the
Avalonia templates. .NET 10 is **LTS, active, supported to 2028-11-14**
([.NET 10 release metadata](https://builds.dotnet.microsoft.com/dotnet/release-metadata/10.0/releases.json)) —
the longest support runway of any toolchain in this comparison.

**Distribution — the most manual macOS story here.** Avalonia's own macOS guide walks you through hand-building
the `.app` directory structure, hand-writing `Info.plist` (`CFBundleExecutable`, `CFBundleIdentifier`,
`NSHighResolutionCapable`, …), generating `.icns`, then signing and notarising
([macOS deployment](https://github.com/AvaloniaUI/avalonia-docs/blob/main/docs/deployment/macos.mdx)).
The docs recommend **Parcel**, Avalonia's own tool, to automate all of it. Parcel's free "Essentials" tier is
**Community — non-commercial projects only**; Plus is $17/month and Pro $49/month
([pricing](https://avaloniaui.net/pricing)). A personal tool qualifies for Community, but note that the
smooth path runs through a product with a paywall, and the free fallback is the manual procedure above.
The framework core itself is **MIT** and free.

**Linux — genuinely the strongest Linux story in this report.** Avalonia 12.0 shipped "the first .NET UI
framework to ship a **native Linux accessibility backend**" (AT-SPI2), and 12.1 graduated a **native Wayland
backend** out of private preview — talking the Wayland protocol directly rather than going through Xwayland,
though it still needs explicit opt-in via the `Avalonia.Wayland` package and `UseWayland()`, and is still
labelled experimental ([12.1 release notes](https://avaloniaui.net/blog/release-12-1)).
Tier 1/2 support for Ubuntu 25.x, Fedora 43, Debian 13 on x64/ARM64. Avalonia draws its own widgets, so
rendering is consistent cross-platform. Caveat: "Custom glibc builds may be required for non-standard
distributions."

**Boring parts.**

| Need | Avalonia answer | First-party? |
|---|---|---|
| HTTP | `System.Net.Http.HttpClient` — .NET base class library | Yes (platform) |
| Credentials | **Nothing.** No keychain/credential/secret page anywhere in the Avalonia docs tree, and the .NET BCL has no cross-platform credential store. Would need a third-party library | **No** |
| Auto-update | Nothing first-party. **Velopack** (2,262 stars, pushed 2026-08-09) is the usual answer — third-party | **No** |
| System theme | `ActualThemeVariant` / `RequestedThemeVariant`; "By default, Avalonia inherits the theme variant set by the system-wide user preference", and both built-in themes ship Dark and Light variants ([theme variants](https://docs.avaloniaui.net/docs/guides/styles-and-resources/how-to-use-theme-variants)). Per-OS detection coverage not spelled out — partially **unverified** | Yes |

**Ecosystem.** 31,310 stars; 20,528,929 total NuGet downloads for `Avalonia` (12,121,118 for
`Avalonia.Desktop`); 1,493 repos tagged `avalonia`; ~1,057 Stack Overflow questions across the `avalonia`
and `avaloniaui` tags. Smaller than Electron/Tauri/Flutter, larger than Wails. The wider **C#/.NET** corpus
that a model can draw on is enormous, but Avalonia's XAML dialect specifically is not.

---

### 4.6 PySide6 / Qt

**Current state.** **PySide6 6.11.1**, published 2026-05-13 — Qt's own Python bindings, from The Qt Company.
Tracks Qt 6.x releases. This is the only candidate needing **zero new language toolchain**: Python 3.14.3 is
already here, and `requires_python` is `>=3.10,<3.15` with an explicit `Programming Language :: Python :: 3.14`
classifier and `cp310-abi3` wheels, so **3.14.3 is supported today**
([PySide6 on PyPI](https://pypi.org/pypi/PySide6/json)).

**Setup from baseline — the shortest of all six.** `pip install PySide6-Essentials` — 105 MB, one command,
no new runtime, no compiler, no Xcode step beyond what's installed. The full `PySide6` meta-package pulls
Addons too (+316 MB) but a GitHub client needs only Essentials (QtCore/QtGui/QtWidgets/QtNetwork).

**The licence is the headline, and it is the one thing here that could disqualify the stack outright.**
PyPI metadata gives the licence as `LGPL-3.0-only OR GPL-2.0-only OR GPL-3.0-only`. There is **no permissive
option** — the alternative is a commercial licence from The Qt Company. For a personal, read-only,
non-distributed tool this is a non-issue. If the app is ever distributed as a frozen single-file binary,
LGPLv3's relinking obligations become a real question. I fetched Qt's licensing page and it did **not** state
the obligations clearly enough to summarise responsibly — **treat the distribution implications as unverified
and worth ten minutes of the author's own reading** if distribution ever becomes real.
([Qt for Python licences](https://doc.qt.io/qtforpython-6/licenses.html))

**Distribution.** `pyside6-deploy` has shipped since PySide6 6.4 and targets Windows, Linux and macOS; it
drives **Nuitka** underneath, installing `nuitka`, `ordered_set` and `zstandard` as part of deployment
([pyside6-deploy](https://doc.qt.io/qtforpython-6/deployment/deployment-pyside6-deploy.html),
[Qt for Python & Nuitka](https://doc.qt.io/qtforpython-6/deployment/deployment-nuitka.html)).
Both backends support Python 3.14: Nuitka 4.1.3 and PyInstaller 6.22.0 both carry the 3.14 classifier.
Resulting bundle size and macOS signing behaviour under `pyside6-deploy`: **unverified** — I could not get
concrete figures or a signing procedure out of the official deployment docs.

**Linux.** Qt on Linux is about as mature as desktop UI gets — decades of use, `manylinux_2_34_x86_64` and
`manylinux_2_39_aarch64` wheels published. Qt draws its own widgets, so rendering is consistent. The
practical Linux risk is not "does it work" but "does the frozen bundle work on the target's glibc" — the same
glibc floor problem Tauri documents, and **unverified** for Qt here.

**Boring parts.**

| Need | PySide6 answer | First-party? |
|---|---|---|
| HTTP | `QtNetwork` / `QNetworkAccessManager` — part of Essentials. (Python's `urllib`/`httpx` also available) | Yes |
| Credentials | **Nothing in Qt.** The standard answer is the `keyring` package (25.7.0, `jaraco/keyring`) which does wrap macOS Keychain / Windows Credential Manager / Secret Service — but it is **community**, not Qt | **No** |
| Auto-update | Nothing first-party | **No** |
| System theme | `QStyleHints::colorScheme()`, added in **Qt 6.5**, with a `colorSchemeChanged` signal; "follows the system's default color scheme" automatically and reacts to changes. Docs note *overriding* is not supported on all platforms ([QStyleHints](https://doc.qt.io/qt-6/qstylehints.html)) | Yes |
| Markdown | `QTextDocument::setMarkdown()` is built into Qt — relevant given §4.4's Flutter finding | Yes |

**Ecosystem — read carefully, the tags mislead.** `pyside6` has only 1,075 Stack Overflow questions and
3,228 tagged repos, but the `qt` tag has **86,290** questions and `pyqt5` has **14,701**. Qt's C++ API surface
is what PySide6 binds, and PyQt5/PyQt6 are near-identical APIs, so the *effective* corpus a model can draw on
is far larger than the PySide6 tag suggests. Conversely, much of that corpus is C++ or PyQt5-era and needs
translation. Net: **genuinely ambiguous** for the AI-assistance constraint, in a way the raw numbers hide.

---

## 5. Comparison table — setup and distribution

| | Electron | Tauri | Wails | Flutter | Avalonia | PySide6 |
|---|---|---|---|---|---|---|
| **Stable version (2026-08-12)** | 43.4.0 | 2.11.5 | v2.14.0 (v3 in beta) | 3.47.0 | 12.1.1 (11.3.20 also live) | 6.11.1 |
| **New toolchain to install** | Node | Rust (+Node in practice) | Go (+npm for templates) | Flutter SDK | .NET 10 SDK | **none** |
| **Download to install** | ~50 MB | rustup (size unverified) | 61.7 MB | **2.10 GiB** | 215 MB | 105 MB |
| **Language** | JS/TS | Rust + JS/TS | Go + JS/TS | Dart | C# | Python |
| **UI rendering** | Chromium (bundled) | OS webview (3 engines) | OS webview (3 engines) | own (Impeller/Skia) | own | own (Qt) |
| **Cross-OS render consistency** | high | **low** | **low** | high | high | high |
| **macOS signing tooling** | via Forge/builder | documented | `wails3 sign` + wizard | via Xcode | manual, or Parcel (paywalled tiers) | unverified |
| **Runtime bundled in app** | 116 MB zip (arm64) | small (unquantified) | unverified | unverified | unverified | unverified |
| **Licence** | MIT | MIT/Apache-2.0 | MIT | BSD-3 | MIT (tooling tiered) | **LGPL/GPL only** |

## 6. Comparison table — the boring parts

Legend: **1st** = first-party/official · **3rd** = community · **—** = nothing · **?** = unverified

| | Electron | Tauri | Wails | Flutter | Avalonia | PySide6 |
|---|---|---|---|---|---|---|
| HTTP client | **1st** (`net`, Chromium stack) | **1st** (`plugin-http`) | **1st** (Go stdlib) | **1st** (`package:http`) | **1st** (`HttpClient`) | **1st** (`QtNetwork`) |
| OS keychain / secret store | **1st** (`safeStorage`) | — (Stronghold ≠ keychain) | — | 3rd (`flutter_secure_storage`) | — | 3rd (`keyring`) |
| Auto-update | **1st**, mac+win only, **no Linux** | **1st**, incl. Linux | **1st** (v3 only), GitHub Releases provider | — | 3rd (Velopack) | — |
| System theme | **1st** (`nativeTheme`) | **1st** (`tauri::Theme`, core) | ? | **1st** (`platformBrightness`) | **1st** (`ActualThemeVariant`) | **1st** (`QStyleHints`, Qt 6.5+) |
| Markdown rendering | 3rd, huge JS choice | 3rd, huge JS choice | 3rd, huge JS choice | 3rd, **official pkg discontinued** | Pro tier, or 3rd | **1st** (`setMarkdown`) |
| Official GitHub API client | **1st** (octokit.js) | 3rd (octocrab) | quasi-1st (google/go-github) | 3rd (github.dart) | **1st** (octokit.net) | 3rd (PyGithub) |

## 7. Runtime performance — what I could not verify

**No vendor in this comparison publishes measured startup time, idle memory, or packaged bundle size.**
Tauri's own size page explicitly gives techniques but no numbers. Electron's distribution page does not
mention size at all. Wails, Flutter, Avalonia and Qt publish nothing comparable.

Searching turns up a dense layer of 2026 SEO comparison articles quoting figures like "Tauri 42 MB idle vs
Electron 168 MB", "Tauri 380 ms cold start vs Electron 1,420 ms", and elsewhere "Tauri ~1.4 s vs Electron
~3.2 s". These are **third-party, mutually inconsistent** (the two startup pairs differ by ~4×), and none
trace to a reproducible methodology. Sources include
[rustify.rs](https://rustify.rs/articles/rust-tauri-vs-electron-2026),
[pkgpulse.com](https://www.pkgpulse.com/guides/electron-vs-tauri-2026) and
[gethopp.app](https://www.gethopp.app/blog/tauri-vs-electron).

What **is** measurable and true: Electron's `darwin-arm64` runtime is a **116 MB compressed** download that
ships inside every app. Tauri and Wails ship no browser engine. Flutter and Avalonia and Qt ship their own
renderer. The *direction* of the size and memory differences is not in doubt; the *magnitudes* circulating
online are not sourced.

**If the numbers matter to the decision in #5, the honest move is to build the same hello-world in the two
finalists and measure on this machine.** That is a prototype ticket, not a research one.

## 8. Secure credential storage — the sharpest differentiator found

This app needs to hold a GitHub token. The field splits three ways, and it does not split the way the
popularity rankings do:

1. **Electron alone has first-party OS-keychain storage.** `safeStorage` maps directly to macOS Keychain,
   Windows DPAPI, and kwallet/gnome-libsecret on Linux — with a documented, queryable failure mode when no
   Linux secret store exists.
2. **Flutter and PySide6 have well-maintained community wrappers** (`flutter_secure_storage` 11.0.0, released
   six days before this research; `keyring` 25.7.0) that reach the same OS stores.
3. **Tauri, Wails and Avalonia have nothing first-party.** Tauri's Stronghold is the nearest miss and is
   easy to mistake for the answer: it is an IOTA-engine **encrypted vault file with its own password**, with
   no documented OS-keychain integration at all. Choosing Tauri or Wails or Avalonia means picking and
   vetting a third-party crate/module/NuGet for the one security-sensitive thing this app does.

There is a legitimate counter-argument the decision conversation should hear: this app could simply shell out
to `gh auth token` and never store a credential itself, given `gh` 2.97.0 is already authenticated on this
machine. That would neutralise the entire differentiator — but it also couples the app to the `gh` CLI, which
is a design decision for #5, not a research finding.

## 9. GitHub API client availability by language

Verified 2026-08-12 via the GitHub API. All actively pushed within the last three weeks; none archived.

| Language | Library | Owner | Stars | Status |
|---|---|---|---|---|
| JS/TS | `octokit/octokit.js` | **GitHub official** | 7,830 | pushed 2026-08-12 |
| C# | `octokit/octokit.net` | **GitHub official** | 2,856 | pushed 2026-08-10 |
| Go | `google/go-github` | Google (not GitHub) | 11,281 | pushed 2026-08-08 |
| Python | `PyGithub/PyGithub` | community | 7,759 | pushed 2026-08-06 |
| Rust | `XAMPPRocky/octocrab` | community | 1,429 | pushed 2026-07-24 |
| Dart | `SpinlockLabs/github.dart` (`github` 9.26.0) | community | — | published 2026-07-28 |

Only the two Octokit libraries are maintained by GitHub itself. That said, for a **read-only** client hitting
the REST API, a plain HTTP client plus `Bearer` header is entirely sufficient — no SDK is strictly required.
The map's "generic core, skills-aware layer on top" preference may even argue *against* a heavy typed SDK.

## 10. Ecosystem depth — and why the usual proxies mislead here

The map's hardest constraint is "mainstream enough that AI assistance is strong". Every available proxy is
flawed; here are all of them, with the flaws stated.

| | Stars | Repos w/ topic | Package downloads | SO questions |
|---|---|---|---|---|
| Flutter | 178,363 | 77,923 | — | 178,672 |
| Electron | 122,472 | 23,132 | 22.3M/month (npm) | 15,275 |
| Tauri | 110,152 | 10,537 | 9.6M recent (crates) · 8.1M/month (npm CLI) | 469 |
| Wails | 35,788 | 862 | — | 26 |
| Avalonia | 31,310 | 1,493 | 20.5M total (NuGet) | ~1,057 |
| PySide6 | — | 3,228 | rate-limited | 1,075 (but `qt` = 86,290, `pyqt5` = 14,701) |

**Stack Overflow counts are the most misleading column on this page.** SO question volume collapsed after
2023, so any framework that grew mainly *after* that point is structurally under-counted. Tauri at 469
questions versus Electron's 15,275 does not mean Tauri has 3% of Electron's material — Tauri has 9.6M recent
crates.io downloads and 10,537 tagged repos. Read SO as "depth of pre-2023 archive", not "current popularity".

**Flutter's 178k questions are overwhelmingly mobile**, and cannot be taken as desktop depth.

**PySide6's tag count badly understates it** — `qt` (86,290) and `pyqt5` (14,701) cover a near-identical API
surface, though much of it in C++ or an older Python binding.

**Wails is the clear outlier on every axis**: 862 tagged repos and 26 SO questions, and the v3 rewrite means
much existing material describes an API that is changing.

## 11. Findings that most constrain the choice

Stated neutrally, in rough order of how much they should move the decision:

1. **The webview stacks trade "small" for "three rendering engines".** Tauri and Wails ship no browser, which
   is their whole pitch — but that means WKWebView on macOS, Chromium on Windows, WebKitGTK on Linux.
   Combined with the map's "Linux is built for but never tested" position, this is a compounding risk:
   the untested platform is also the one with the least-common engine. Electron, Flutter, Avalonia and Qt all
   render identically everywhere.
2. **Wails v3 is eleven days into beta and moving hourly** (beta.0 → beta.8 in eleven days). Choosing Wails
   means choosing between a stable-but-outgoing v2 and a better-but-prerelease v3. Its v3 auto-updater is the
   best in the field, and its ecosystem is the thinnest by an order of magnitude.
3. **Flutter's 2.10 GiB SDK is the largest install cost by ~10×**, its official Markdown package is
   discontinued, and its macOS sandbox default silently blocks all network calls until an entitlement is added.
   None of these are dealbreakers; all three are surprises.
4. **Only Electron has first-party OS-keychain storage.** Tauri's Stronghold looks like the answer and is not.
   Unless the app delegates auth to `gh` entirely (§8).
5. **PySide6 costs literally nothing to install** — Python 3.14.3 is already here, 3.14 is supported, one
   `pip install`. It is the only zero-toolchain option. Its licence is LGPL/GPL-only with no permissive
   alternative, which is irrelevant for a personal tool and potentially material if distribution ever happens.
6. **Avalonia has the best Linux story** (native AT-SPI2 accessibility, native Wayland as of 12.1) and the
   longest support runway (.NET 10 LTS to 2028-11-14) — and the most manual macOS packaging, with the
   automated path behind a tiered product.
7. **Electron is the shortest path to a working app** from this exact baseline: one 50 MB install, first-party
   coverage of every boring part except Linux auto-update, official GitHub SDK, and the largest current
   ecosystem after Flutter's mobile-inflated numbers.

## 12. Open questions this research could not close

- **Packaged bundle size, cold start, and idle memory** for anything except Electron's runtime zip. No primary
  sources exist. Needs measurement (§7).
- **Avalonia's true minimum .NET target** — docs and launch blog disagree (§4.5).
- **Wails' minimum Go version** — docs disagree internally, 1.24 vs 1.25 (§4.3).
- **Wails' system-theme API** — no documented feature page found.
- **PySide6 LGPLv3 obligations** for a frozen/distributed binary — Qt's own licensing page was not specific
  enough to summarise responsibly.
- **`pyside6-deploy` output size and macOS signing behaviour** — not documented.
- **Rust toolchain install footprint** — not published as a single figure.
- **How much of Flutter's enormous corpus is desktop-specific** — unmeasurable with available data.

---

## Sources

Primary — vendor docs, repos, release notes, package registries:

- Electron: [release timelines](https://www.electronjs.org/docs/latest/tutorial/electron-timelines) · [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage) · [autoUpdater](https://www.electronjs.org/docs/latest/api/auto-updater) · [nativeTheme](https://www.electronjs.org/docs/latest/api/native-theme) · [net](https://www.electronjs.org/docs/latest/api/net) · [application distribution](https://www.electronjs.org/docs/latest/tutorial/application-distribution) · [releases API](https://api.github.com/repos/electron/electron/releases) · [npm metadata](https://registry.npmjs.org/electron/43.4.0)
- Tauri: [prerequisites](https://v2.tauri.app/start/prerequisites/) · [plugins](https://v2.tauri.app/plugin/) · [http plugin](https://v2.tauri.app/plugin/http-client/) · [updater plugin](https://v2.tauri.app/plugin/updater/) · [stronghold plugin](https://v2.tauri.app/plugin/stronghold/) · [app size](https://v2.tauri.app/concept/size/) · [distribute](https://v2.tauri.app/distribute/) · [AppImage](https://v2.tauri.app/distribute/appimage/) · [macOS signing](https://v2.tauri.app/distribute/sign/macos/) · [tauri::Theme](https://docs.rs/tauri/latest/tauri/enum.Theme.html) · [plugins-workspace](https://github.com/tauri-apps/plugins-workspace/tree/v2/plugins)
- Wails: [v3.0.0-beta.0 release notes](https://github.com/wailsapp/wails/releases/tag/v3.0.0-beta.0) · [installation](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/getting-started/installation.mdx) · [quick-start installation](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/quick-start/installation.mdx) · [status / beta compatibility promise](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/status.mdx) · [updater guide](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/guides/updater.mdx) · [macOS build guide](https://github.com/wailsapp/wails/blob/master/docs/src/content/docs/guides/build/macos.mdx) · [v3 beta announcement](https://v3.wails.io/blog/wails-v3-beta/)
- Flutter: [releases_macos.json](https://storage.googleapis.com/flutter_infra_release/releases/releases_macos.json) · [manual install](https://docs.flutter.dev/install/manual) · [desktop support](https://docs.flutter.dev/platform-integration/desktop) · [building macOS apps](https://docs.flutter.dev/platform-integration/macos/building) · [building Linux apps](https://docs.flutter.dev/platform-integration/linux/building) · [package:http](https://pub.dev/packages/http) · [flutter_markdown (discontinued)](https://pub.dev/packages/flutter_markdown) · [flutter_secure_storage](https://pub.dev/packages/flutter_secure_storage)
- Avalonia: [supported platforms](https://docs.avaloniaui.net/docs/supported-platforms) · [theme variants](https://docs.avaloniaui.net/docs/guides/styles-and-resources/how-to-use-theme-variants) · [macOS deployment](https://github.com/AvaloniaUI/avalonia-docs/blob/main/docs/deployment/macos.mdx) · [Avalonia 12 blog](https://avaloniaui.net/blog/avalonia-12) · [Avalonia 12.1 blog](https://avaloniaui.net/blog/release-12-1) · [pricing](https://avaloniaui.net/pricing) · [releases API](https://api.github.com/repos/AvaloniaUI/Avalonia/releases)
- Qt / PySide6: [PySide6 on PyPI](https://pypi.org/pypi/PySide6/json) · [PySide6-Essentials](https://pypi.org/pypi/PySide6-Essentials/json) · [PySide6-Addons](https://pypi.org/pypi/PySide6-Addons/json) · [QStyleHints](https://doc.qt.io/qt-6/qstylehints.html) · [licences](https://doc.qt.io/qtforpython-6/licenses.html) · [pyside6-deploy](https://doc.qt.io/qtforpython-6/deployment/deployment-pyside6-deploy.html) · [Nuitka backend](https://doc.qt.io/qtforpython-6/deployment/deployment-nuitka.html) · [keyring](https://pypi.org/pypi/keyring/json)
- Toolchains: [nodejs.org/dist](https://nodejs.org/dist/index.json) · [go.dev/dl](https://go.dev/dl/?mode=json) · [rust stable manifest](https://static.rust-lang.org/dist/channel-rust-stable.toml) · [.NET 10 releases](https://builds.dotnet.microsoft.com/dotnet/release-metadata/10.0/releases.json) · [.NET download page](https://dotnet.microsoft.com/en-us/download/dotnet)
- GitHub API clients: [octokit.js](https://github.com/octokit/octokit.js) · [octokit.net](https://github.com/octokit/octokit.net) · [go-github](https://github.com/google/go-github) · [PyGithub](https://github.com/PyGithub/PyGithub) · [octocrab](https://github.com/XAMPPRocky/octocrab) · [github.dart](https://pub.dev/packages/github)
- Tooling: [Electron Forge](https://github.com/electron/forge) · [electron-builder](https://github.com/electron-userland/electron-builder) · [Squirrel.Windows](https://github.com/Squirrel/Squirrel.Windows) · [Velopack](https://github.com/velopack/velopack) · [Nuitka](https://pypi.org/pypi/Nuitka/json) · [PyInstaller](https://pypi.org/pypi/pyinstaller/json)

Third-party, cited only to show what circulates and why it is not trusted (§7):
[rustify.rs](https://rustify.rs/articles/rust-tauri-vs-electron-2026) ·
[pkgpulse.com](https://www.pkgpulse.com/guides/electron-vs-tauri-2026) ·
[gethopp.app](https://www.gethopp.app/blog/tauri-vs-electron)
