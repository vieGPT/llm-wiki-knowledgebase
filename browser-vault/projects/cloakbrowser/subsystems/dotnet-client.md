---
title: >-
    .NET client
category: subsystems
tags: [dotnet, csharp, playwright, wrapper-parity, humanize]
aliases: [CloakBrowser for .NET, CloakBrowser NuGet, HumanizedPage, CloakLauncher]
relationships:
  - target: "[[projects/cloakbrowser/architecture/python-js-wrapper-parity]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: derived_from
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: derived_from
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: derived_from
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: derived_from
  - target: "[[projects/cloakbrowser/techniques/geoip-timezone-locale-matching]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/humanized-input]]"
    type: implements
sources:
  - ~/code/CloakBrowser/dotnet/README.md
  - ~/code/CloakBrowser/dotnet/CloakBrowser.sln
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/CloakBrowser.csproj
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/CloakLauncher.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/LaunchOptions.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Handles.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Config.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/ProxyResolver.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/ViewportDefaults.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Download.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/License.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/GeoIp.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Widevine.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Diagnostics.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/CloakLog.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Human/HumanEngine.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Human/InjectedSource.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Human/Actionability.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Human/HumanPage.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Human/HumanConfig.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Wrappers/Humanize.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Wrappers/HumanizedPage.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser.Generators/InterfaceDelegationGenerator.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser.Cli/Program.cs
  - ~/code/CloakBrowser/dotnet/examples/CloakBrowser.Examples/Program.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/IsolatedCacheDir.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/BrowserFactAttribute.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/Wrappers/LicenseGuardCompositionTests.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/Wrappers/FakeProxy.cs
summary: >-
    The C# port (NuGet CloakBrowser 0.6.4, .NET 8, Microsoft.Playwright 1.64.1): async-only launcher, Roslyn-generated humanize decorators, DispatchProxy license guard.
provenance:
  extracted: 0.84
  inferred: 0.11
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T10:47:25+01:00
---

# .NET client

The `dotnet/` tree is a C# port of the Python wrapper, published as the `CloakBrowser` NuGet package (version 0.6.4, MIT, the same version as the Python and npm packages, as of 26/10/2026, commit b1a00ed). It targets `net8.0` and sits on `Microsoft.Playwright`. Most files open with "Direct port of Python `cloakbrowser/<module>.py`", and the shared constants match the Python ones: per-platform Chromium versions, the Ed25519 signing key, the seed range, the version gates. The structural differences come from the runtime. .NET Playwright is async-only and its interfaces cannot be monkey-patched, so the port uses a source-generated decorator layer for humanize and a runtime `DispatchProxy` for the license guard. This page was first built from `dotnet/README.md` alone and is now checked against the 76 `.cs` files; where the two disagree, the code wins (see [[#README vs code contradictions]]). Module-level parity is on [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].

## Solution layout

| Project | Kind | Role |
|---|---|---|
| `src/CloakBrowser` | library, `net8.0`, NuGet `CloakBrowser` | launcher, config, download, license, GeoIP, Widevine, humanize engine, wrappers |
| `src/CloakBrowser.Generators` | Roslyn source generator, `netstandard2.0` | emits the delegating members of the humanize decorators; referenced as an analyzer, not shipped as a runtime assembly |
| `src/CloakBrowser.Cli` | console exe, assembly `cloakbrowser-cli` | port of `__main__.py` |
| `tests/CloakBrowser.Tests` | xUnit 2.9 | about 400 `[Fact]`/`[Theory]` cases (as of commit b1a00ed), split into fake-based unit tests and real-browser tests |
| `examples/CloakBrowser.Examples` | console exe | named scenarios (`basic`, `humanize`, `context`, `persistent`, `proxy-geoip`, `visual`) plus manual probes (`bottest`, `webrtctest`, `trusted`, `timeout`, `repro549`) |

NuGet dependencies (as of 26/10/2026, commit b1a00ed): `Microsoft.Playwright` 1.64.1, `MaxMind.GeoIP2` 5.2.0, and `BouncyCastle.Cryptography` 2.4.0. BouncyCastle is there because .NET 8 has no built-in Ed25519, and the comment in the project file calls it the mirror of Python's `cryptography` and Node's `node:crypto`. `InternalsVisibleTo` opens internals to the tests and to the CLI.

## Public surface

- `CloakLauncher.LaunchAsync`, `LaunchContextAsync` and `LaunchPersistentContextAsync` mirror Python's `launch` family. There is no sync API because .NET Playwright has none.
- Options are classes (`LaunchOptions`, `LaunchContextOptions`) instead of keyword arguments. `Proxy` is typed `object` so it can take either a URL string or a `ProxySettings` object, matching Python's `str | ProxySettings`.
- The launchers return owning handles (`CloakBrowserHandle`, `CloakContextHandle`, both `IAsyncDisposable`) instead of raw Playwright objects. A handle owns the `IPlaywright` instance and disposes it on close. `Browser`/`Context` give the (possibly humanized, license-guarded) Playwright object; `RawBrowser`/`RawContext` give the untouched one.
- The launch pipeline follows Python `launch()` order: ensure binary → GeoIP → proxy resolution → WebRTC `auto` replacement → exit-IP flag → `BuildArgs` → Windows-font warning → denial file → Playwright launch with `IgnoreDefaultArgs` (`--enable-automation`, `--enable-unsafe-swiftshader`). See [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]].
- `CloakLog` is a small static logging facade (stderr by default, settable level and sink). The wrappers in the other languages use their language's logger. ^[inferred]

## Two wrapping mechanisms

The port needs two different interception techniques, and the order in which they compose is pinned by tests.

**Humanize: compile-time decorators.** `HumanizedPage`, `HumanizedFrame`, `HumanizedLocator`, `HumanizedFrameLocator`, `HumanizedElementHandle`, `HumanizedMouse`, `HumanizedKeyboard`, `HumanizedBrowserContext` and `HumanizedBrowser` are `partial` classes marked `[GenerateInterfaceDelegation(typeof(IPage))]` (and so on). The generator walks the interface and every interface it inherits, and emits a forwarding member for each member the class does not declare by hand. The hand-written ones are the humanized actions. Generated members carry a marker attribute so tests can tell them apart. The result is static typing and no reflection for the hundreds of forwarded members.

- Everything that returns an interactive object re-wraps it: `Mouse`, `Keyboard`, `Locator`/`GetBy*`, frames and frame events, element handles, `Context`, `NewPageAsync`. Contexts are memoized per raw context, so `page.Context` keeps reference identity.
- Some Playwright methods down-cast their argument to the concrete Playwright type and throw on a wrapper (#549: `AddLocatorHandlerAsync`, screenshot masks, `NewCDPSessionAsync`). The wrappers unwrap those arguments first. `Humanize.Unwrap(page)` and `Original` are the user-facing escape hatches.

**License guard: a runtime proxy.** `LicenseGuard` wraps `IBrowser`, `IBrowserContext` and `IPage` in a `System.Reflection.DispatchProxy` (`LicenseGuardProxy<T>`). After every guarded call, whether it failed or succeeded, the proxy checks the per-launch denial file and throws `CloakBrowserLicenseError` when a code is there. Returned pages and contexts are wrapped recursively. Property accessors are forwarded untouched, except `get_Pages` (a persistent context's existing pages get guarded) and `get_Context` (memoized). Handle arguments are unwrapped before the call. The guard is runtime reflection, unlike the humanize layer, because it must cover every method of three large interfaces uniformly. ^[inferred]

**Composition order.** Humanize wraps the raw object first, and the guard wraps the humanized object. `LicenseGuardCompositionTests` explains why: guarding the raw object would put Playwright's own methods in front of the humanized ones, "clicks would teleport", and "the Python and JS wrappers had exactly that bug".

## Humanize layer

- `HumanEngine` (split over `HumanEngine.cs` and `HumanEngine.Actions.cs`) is a port of the unified `engine.py` pipeline: resolve in a CDP isolated world with Playwright's own selector engine → state waits → whole-notch wheel scrolling → Fitts'-law stroke → hit-target check → press or type. It fails with `PlaywrightException`/`System.TimeoutException` and never falls back to stock actions. See [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]].
- `InjectedSource` loads Playwright's InjectedScript the same way Python does: as a string literal from the driver files that the NuGet package copies next to the app (`.playwright/package/lib`, `coreBundle.js` or `injectedScriptSource.js`). Search order: `PLAYWRIGHT_DRIVER_SEARCH_PATH`, the app base directory, the Playwright assembly directory, the NuGet cache layout. The install expression passes `sdkLanguage: "csharp"`.
- `PlaywrightInternals` reads private fields of Microsoft.Playwright by reflection: a locator's `_frame` and `_selector`, the page's `_timeoutSettings`, and an element handle's `Parent`/`Guid`. It throws a clear "Microsoft.Playwright layout changed" error when they are missing. This is the C# counterpart of Python's dependence on `Frame._timeout` and `ElementHandle._frame`. ^[inferred]
- Microsoft.Playwright joins frame-locator parts with an extra space (`enter-frame  >> `), so the engine splits on the marker with any surrounding whitespace.
- `HumanPage` (from `NewHumanPageAsync`) is an explicit API over the same engine: main-frame, non-strict selectors, options in `HumanActionOptions`.
- `Actionability` keeps the pre-engine public checks and still throws the typed `ElementNot…Error` classes, but runs on the engine, so every Playwright selector works there. `HumanScroll` (a standalone wheel helper that "mirrors the JS Puppeteer scroller") and `IsolatedWorld` are kept for compatibility and are not used by humanized actions. See [[projects/cloakbrowser/subsystems/humanize-errors|Humanize errors]].
- `HumanConfig` matches the Python dataclass field by field (Dhakal et al. keystroke timings, Fitts' law mouse parameters, `ScrollNotchPx` 0 = persona default). `With(...)` accepts snake_case or PascalCase keys, and ranges as tuples or 2-element sequences. It ignores unknown keys and never mutates the base config.

## Behaviors worth knowing

- **Headed viewport default.** On a headed launch, a page or context created without an explicit viewport gets `NoViewport`. `ViewportDefaults` is the single implementation, shared by the handle and the humanized browser. It mirrors Python's `_default_no_viewport`, because an emulated viewport on a real window yields `outerWidth < innerWidth`. On binaries at or above `148.0.7778.215.4` this extends to headless. See [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]].
- **Proxy parsing.** `ProxyResolver` hand-rolls a URL parser that copies Python `urlparse` behavior, including lowercasing the host, "so the assembled proxy URL/server string is byte-for-byte identical". SOCKS5 always goes inline in `--proxy-server`. Credentialed HTTP goes inline only above the per-platform `HttpProxyInlineAuthMinVersion` floor; otherwise it goes through Playwright's proxy object. See [[projects/cloakbrowser/subsystems/proxy-handling|Proxy handling]].
- **Locale and timezone** go to the binary as flags (`--lang`, `--fingerprint-locale`, `--fingerprint-timezone`), never through CDP emulation. See [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]].
- **Proxy not used by Playwright's request client.** The README warns that `APIRequest` and `route.FetchAsync()` do not use the browser's proxy and therefore send from the real IP. The code does nothing about this.
- **Timeouts.** One deadline per action (#307): the explicit timeout, else the page or context default (read by reflection), else 30 s. `Timeout = 0` means no limit.
- **Sync over async.** `License` makes blocking HTTP calls (`GetAwaiter().GetResult()`), and `EnsureBinaryAsync` moves license validation onto `Task.Run` so it does not block a UI thread.

## README vs code contradictions

Checked at commit b1a00ed (26/10/2026):

| README says | Code says |
|---|---|
| `Microsoft.Playwright` **1.49.0** | `CloakBrowser.csproj` pins **1.64.1** |
| Requirements list Playwright and GeoIP2 only | A third hard dependency, `BouncyCastle.Cryptography` 2.4.0, verifies the Ed25519 signatures |
| Binary carries "71 C++ fingerprint patches" | Not a code fact. The main README says 87 source-level patches, so 71 is most likely stale. ^[ambiguous] |
| CLI section shows `install`, `info`, `update`, `clear-cache` | `Program.cs` implements `login` and `logout` too, plus `doctor` as an alias for `info`, matching the README's feature matrix. The CLI section is incomplete. |
| `Download.cs` "(+ zip-slip guard)", and the test table claims a zip-slip test in `DownloadConfigTests.cs` | No explicit guard and no such test. Extraction calls `TarFile.ExtractToDirectory`/`ZipFile.ExtractToDirectory` directly, with the comment "Signature-verified before extraction, so unpack as-is". Traversal protection is left to the Ed25519 check and to the BCL extractors' own entry validation. ^[inferred] |
| Decorators use "no reflection on the hot path" | True for the generated forwarding members. The engine itself reads Playwright internals and action option objects by reflection, and the license guard is a reflection-based `DispatchProxy`. |
| "76 `.cs` files" (earlier gap note) | Confirmed: 76 tracked `.cs` files (41 under `src/`, 34 under `tests/`, 1 example), plus 5 project files and the solution |

## Testing approach

- **Fakes without a browser.** `FakeProxy` is a `DispatchProxy` fake for any interface. It records member accesses and returns defaults, so wrapper delegation, re-wrapping and the escape hatches are tested without Chromium.
- **Real-browser tests** use `[BrowserFact]`/`[BrowserTheory]`. They are reported as *skipped*, not silently passed, unless `CLOAKBROWSER_BINARY_PATH` is set. They serve the same fixture page as the Python and JS suites (`SiteServer`), and `HumanEngineBrowserTests` is a "Port of `js/tests/humanizeEngine.test.ts` / `tests/test_humanize_engine.py`".
- **Cache isolation.** A `[ModuleInitializer]` (`IsolatedCacheDir`) points `CLOAKBROWSER_CACHE_DIR` at a fresh temp directory before any test runs. It is a port of the Python `isolated_cache_dir` fixture, for the same reason: a real `license.key` would flip version-gated defaults. Real-browser runs opt out.
- **Env-mutating suites** share an `env-serial` xUnit collection to avoid races under xUnit's parallelism.
- **Test seams** are `internal static` delegates (`ValidateLicenseOverride`, `SignedManifestOverride`, `SigningPubkeysOverride`, `HomeDirOverride`, …). The comments call them "mirror the monkey-patching the Python/JS tests rely on".
- `tests/test_resolver_sources_match.py` (Python side) asserts that `Human/WorldHelpers.cs` embeds the same JS helper library as `world.py` and `worldHelpers.ts`.

## Open Questions

- The README's "71 patches" figure needs the binary changelog to settle; the main README's 87 is the more recent claim. ^[ambiguous]
- Whether .NET's built-in SOCKS support in `HttpClient` handles credentialed `socks5://` proxies for the GeoIP echo call. The code only logs "SOCKS5 proxy requires a SOCKS-capable transport" if building the handler throws. ^[ambiguous]
- The reflection on Microsoft.Playwright private fields (`_frame`, `_selector`, `_timeoutSettings`) fails loudly when those fields move. Nothing pins a maximum Playwright version, so a NuGet upgrade could break humanize at runtime. ^[inferred]

## Sources

Commit `b1a00ed` (26/10/2026), `~/code/CloakBrowser`:

- `dotnet/README.md`, `dotnet/CloakBrowser.sln`, `dotnet/src/CloakBrowser/CloakBrowser.csproj`
- `dotnet/src/CloakBrowser/`: `CloakLauncher.cs`, `LaunchOptions.cs`, `Handles.cs`, `Config.cs`, `ProxyResolver.cs`, `ViewportDefaults.cs`, `Download.cs`, `License.cs`, `GeoIp.cs`, `Widevine.cs`, `Diagnostics.cs`, `CloakLog.cs`
- `dotnet/src/CloakBrowser/Human/`: `HumanEngine.cs`, `InjectedSource.cs`, `Actionability.cs`, `HumanPage.cs`, `HumanConfig.cs`
- `dotnet/src/CloakBrowser/Wrappers/`: `Humanize.cs`, `HumanizedPage.cs`
- `dotnet/src/CloakBrowser.Generators/InterfaceDelegationGenerator.cs`, `dotnet/src/CloakBrowser.Cli/Program.cs`, `dotnet/examples/CloakBrowser.Examples/Program.cs`
- `dotnet/tests/CloakBrowser.Tests/`: `IsolatedCacheDir.cs`, `BrowserFactAttribute.cs`, `Wrappers/LicenseGuardCompositionTests.cs`, `Wrappers/FakeProxy.cs`
