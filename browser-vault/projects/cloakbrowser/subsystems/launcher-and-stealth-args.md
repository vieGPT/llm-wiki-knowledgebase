---
title: >-
    Launcher and stealth args
category: subsystems
tags: [launcher, chromium-flags, fingerprinting, playwright, cloakbrowser]
aliases: [buildArgs, stealth args, build_args]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/screen-viewport-consistency]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/profile-seed]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/webrtc-ip-spoofing]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/geoip-timezone-locale-matching]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/source-level-chromium-patches]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/widevine-cdm-seeding]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/browser-manager]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/js/src/args.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/README.md
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__init__.py
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md
  - repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/config.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/puppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/extension-loading.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/require.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/setup.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/widevine-integration.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_build_args.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_config.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_extension_loading.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/conftest.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_launch.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_launch_context.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_persistent_context.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/CloakLauncher.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Config.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/LaunchOptions.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/ViewportDefaults.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/BuildArgsTests.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/ViewportTests.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/FontWarningTests.cs
summary: >-
    How the wrapper turns launch options into a deduplicated Chromium argv and a Playwright launch: stealth defaults, overrides, GeoIP, proxy, license guards, and version-gated viewport behavior.
provenance:
  extracted: 0.84
  inferred: 0.13
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:42:16+01:00
updated: 2026-10-31T06:58:25+01:00
---

# Launcher and stealth args

The launcher builds the Chromium command line that switches on the binary's compiled-in fingerprint patches, then hands it to Playwright or Puppeteer. In the JS wrapper this happens in `args.ts::buildArgs`, which both backends share. The defaults and the version gates live in `config.ts`, which "mirrors Python cloakbrowser/config.py" (see [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]]). In Python, `browser.py` runs the pipeline behind six public entry points (`launch`, `launch_async`, `launch_context`, `launch_context_async`, `launch_persistent_context`, `launch_persistent_context_async`), each returning an ordinary Playwright `Browser` or `BrowserContext`; `config.py` (ast-extract god node, degree 28 as of 31/10/2026) holds constants, platform detection and version gates. The flags only select and configure patches that already exist in the binary. The wrapper injects no JavaScript. See [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]] and [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]].

## Launch pipeline (Python `launch()` order)

1. **Reject removed kwargs**: `backend=` raises `TypeError` (patchright removed in 0.4.0).
2. **Resolve binary**: `ensure_binary(license_key, browser_version, release_channel)`. See [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]].
3. **GeoIP**: `maybe_resolve_geoip()` returns `(timezone, locale, exit_ip)`. Raw `--fingerprint-timezone`, `--lang` or `--fingerprint-locale` in `args` count as explicit and are never overwritten. See [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]].
4. **Proxy routing**: `_resolve_proxy_config()` yields either a Playwright `proxy` kwarg or a `--proxy-server` flag. See [[projects/cloakbrowser/subsystems/proxy-handling|Proxy handling]].
5. **WebRTC IP**: `--fingerprint-webrtc-ip=auto` is replaced by the resolved proxy exit IP, or dropped with a warning. The GeoIP exit IP is appended unless the flag is already set. See [[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]].
6. **Persistent seed** (persistent contexts only): `persistent_seed_args()` prepends the profile's stored `--fingerprint`. See [[projects/cloakbrowser/subsystems/profile-seed|Profile seed]].
7. **Merge flags** with `build_args()`, then warn once if a Linux host spoofing Windows lacks the Windows font set. See [[projects/cloakbrowser/detection-vectors/font-fingerprinting|Font fingerprinting]].
8. **License env**: with a key, mint a denial-file path and build an env carrying `CLOAKBROWSER_LICENSE_KEY` to the browser process, merged with any user `env`.
9. **Launch** `chromium.launch(executable_path=…, args=…, ignore_default_args=IGNORE_DEFAULT_ARGS)`. On failure, map the Pro binary's license exit codes (76–79) to `CloakBrowserLicenseError`, stop the Playwright driver, else re-raise unchanged.
10. **Post-launch wraps**, in a deliberate order: `close()` also stops Playwright; license guards; one-time warning that Playwright's request client bypasses a browser-level proxy (#579); default `no_viewport`; humanize patching (see [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]]).

## Argument priority and deduplication

- `buildArgs` keys every flag by the text before `=` and keeps the last writer. The priority order is: **stealth defaults < user `args` < dedicated options** (`timezone`, `locale`). Setting `DEBUG=cloakbrowser` logs each override. Python's `build_args()` applies the same rule (introduced 0.3.9, so user values win instead of passing two copies to Chromium).
- `stealthArgs: false` drops the defaults entirely.
- `timezone` becomes `--fingerprint-timezone=<tz>`. `locale` sets both `--lang` and `--fingerprint-locale`. These are binary flags, not CDP emulation, so there is no detectable emulation traffic and the timezone applies process-wide to every context. See [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]].
- `extensionPaths` are resolved to absolute paths and emitted twice, as `--load-extension` and as `--disable-extensions-except`.

## Default stealth args

- The base set is `--no-sandbox` plus `--fingerprint=<seed>`. The seed is random in the range 10000 to 99999 on each launch unless the caller pins one, for example `--fingerprint=12345` or a persistent profile seed (see [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]] and [[projects/cloakbrowser/subsystems/profile-seed|Profile seed]]).
- The platform persona is `--fingerprint-platform=windows` on Linux and Windows hosts, and `--fingerprint-platform=macos` on macOS. Screen and window size come from the real display, not from this flag ("verified: identical across seeds"). See [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]].
- `IGNORE_DEFAULT_ARGS` strips two Playwright defaults that leak automation:
  - `--enable-automation` exposes `navigator.webdriver = true`.
  - `--enable-unsafe-swiftshader` forces SwiftShader software WebGL, whose renderer string no real user has.

## Conditional flags

- **`--ignore-gpu-blocklist`** is added in headed mode on every platform, because Chromium blocks WebGL on software GPUs under Docker/Xvfb (issue #56). It is also added on Windows in every mode, because the GPU blocklist disables WebGPU on the Microsoft Basic Render Driver.
- **`MediaRouter`**: when `--fingerprint-windows-font-metrics` is present, `MediaRouter` is merged into the existing `--enable-features` value. Playwright's default args switch it off, but stock Chrome ships it on. The aim is a feature set that looks like a stock browser, not a test harness.
- **`--start-maximized`** is added automatically when the caller sets no viewport, no `defaultViewport` and no window-geometry flag. The rationale: "real Chrome overwhelmingly runs maximized". In Python it is additionally gated by `binary_supports_maximized_window`, and `launch_context()` suppresses it when the caller chose a viewport.

## Version-gated behavior

Three capability gates compare the binary version that will actually launch against a threshold. Resolution order: a declared pin first, then a Pro build when a license resolves, then the free per-platform default.

| Gate | Threshold (as of 26/10/2026, commit b1a00ed) | Why |
|---|---|---|
| `binarySupportsHeadlessNoViewport` | `148.0.7778.215.4` | Newer binaries report coherent headless dimensions without an emulated viewport. Older ones get `DEFAULT_VIEWPORT` 1920x947, which models maximized Chrome on a 1080p Windows screen minus about 85 px of browser UI |
| `binarySupportsMaximizedWindow` | same threshold, separate name so the two can diverge | Below the threshold, a maximized headless window with a 1280x720 CDP viewport gives `outerWidth < innerWidth`, which is a bot tell |
| `binarySupportsHttpProxyInlineAuth` | per platform: linux-x64 and windows-x64 `146.0.7680.177.5`; linux-arm64 and macOS `148.0.7778.215.3` | Older binaries treat `user:pass@` in `--proxy-server` as an invalid host and drop authentication (#182). Below the floor, credentials go through Playwright's proxy dict instead. See [[projects/cloakbrowser/subsystems/proxy-handling\|Proxy handling]] |

All three gates **fail safe** (feature off) for a local override binary (`CLOAKBROWSER_BINARY_PATH`) with no declared version, for an unresolvable Pro build, and for a malformed version string. One exception in Python: `binary_supports_http_proxy_inline_auth` returns **true** for a Pro license with no cached Pro marker, reasoning that the server's latest Pro build is above every floor by construction. The JS gate does **not** match: `getEffectiveVersion(pro=true)` deliberately returns null when no cached Pro marker and binary exist, and the gate then returns false (B5 batch, `config.ts` as of commit b1a00ed). On the JS launch paths `ensureBinary` runs before `resolveProxyConfig`, so the marker normally exists by the time the gate is asked. The difference is probably invisible in practice. ^[inferred]

## Viewport wrapping

Playwright's default emulated 1280×720 viewport inside a larger real window yields `outerWidth < innerWidth`, a physically impossible window. `launch()` returns a raw Playwright `Browser`, so the launcher wraps its `new_page()` / `new_context()` to `setdefault(no_viewport=True)` for headed launches and for headless launches on binaries past the no-viewport gate; an explicit `viewport` from the caller is never overridden. When a caller passes both an explicit `viewport` and `no_viewport` via kwargs, the viewport is dropped because Playwright rejects both. See [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen/viewport consistency]].

## License guards (Python)

A post-handshake license denial, for example going over the concurrent-session cap, kills the browser without a launch failure, so the user's next call would see a bare `TargetClosedError`. The launcher wraps every public method of the browser, the context and any already-open pages. On any exception, and after every successful call, the wrapper checks the denial file the binary writes and raises `CloakBrowserLicenseError`. The binary writes that file the moment it is over the cap but keeps serving blank responses for about a second, so checking only on error would be too late. `new_page` / `new_context` results are guarded recursively; `close`, `on`, `once`, `remove_listener` are never wrapped (teardown and Playwright's own event dispatch). Guards are stored as a non-descriptor `_Guarded` callable so humanize can copy them without re-binding (Python 3.14 made `functools.partial` a descriptor, #488). On persistent contexts humanize is applied *before* the guard, because a guard stored on the instance would shadow humanize's class-level patch and silently disable it. See [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].

## Entry-point differences

- `launch_context()` resolves GeoIP once before calling `launch()`, avoiding double resolution; closing the returned context also closes the browser.
- `launch_persistent_context()` additionally seeds the profile fingerprint and the Widevine hint file (see [[projects/cloakbrowser/subsystems/widevine-cdm-seeding|Widevine CDM seeding]]). `user_agent`, viewport and `color_scheme` become context kwargs, while timezone and locale stay binary flags.
- Async variants run the binary/license check and GeoIP/WebRTC lookups in a worker thread (0.6.3), so a slow proxy no longer stalls the caller's event loop for up to 20 s.
- `timezone_id` is accepted as an alias of `timezone`.

## JS Playwright wrapper specifics (`playwright.ts`, B5 batch)

- **Composable helpers**: `buildLaunchOptions()` returns the full `chromium.launch()` options without starting Chromium, `buildContextOptions()` the stealth-safe `newContext()` options, and `humanizeBrowser()` applies humanize to an existing `Browser`. Together they let custom Playwright builds or other wrappers reuse the pipeline.
- **CDP emulation guard**: `contextOptions.locale` and `contextOptions.timezoneId` are stripped with a warning, because Playwright implements both with detectable CDP emulation. The top-level `locale`/`timezone` fields route through binary flags instead.
- **Effective headless**: Playwright spreads `launchOptions` last, so a raw `launchOptions.headless` beats the top-level field. The viewport logic reads the same effective value, so a headed browser never gets a fixed viewport by accident. The Puppeteer wrapper resolves the opposite way, matching its own spread order. See [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].
- **No double work in `launchContext()`**: it resolves GeoIP and WebRTC once, then calls the inner `launch()` with `geoip: false` and `humanize: false`, and applies `patchContext` itself, so humanize patches exactly once.
- **Test-pinned behavior** (`js/tests/config.test.ts`): user `--fingerprint` and `--fingerprint-platform` override defaults; the `timezone`/`locale` options override raw user flags; timezone/locale flags are injected even with `stealthArgs: false`; output never contains duplicate flag keys. `--start-maximized` is added only above the gate, is suppressed by a user `--window-size` or an explicit viewport, and is never doubled. The headless-no-viewport gate is off at `148.0.7778.215.3`, described in the test as the current live Pro version, and on at `.4`. The version pin rejects unsafe values and non-ASCII digits, for parity with Python/.NET.

## .NET launcher specifics (`CloakLauncher.cs`, batch D)

`CloakLauncher` is a "Direct port of Python `cloakbrowser/browser.py`" (commit b1a00ed). Because .NET Playwright is async-only, it offers just the three async entry points. `Config.cs` carries the same constants and gates as `config.py`. See [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

- **Same `BuildArgs`.** It dedups by flag key with priority stealth defaults < user args < dedicated `Timezone`/`Locale`, adds `--ignore-gpu-blocklist` headed or on Windows, merges `MediaRouter` into `--enable-features` when `--fingerprint-windows-font-metrics` is present, makes extension paths absolute, and adds `--start-maximized` only above the gate and when no window geometry was chosen. `BuildArgsTests` and `ViewportTests` (the latter a port of upstream 50bf14b's Python/JS tests) pin these rules.
- **Owning handles instead of Playwright objects.** The launchers return `CloakBrowserHandle`/`CloakContextHandle`, which own the `IPlaywright` instance. A failure after launch (while building the handle) closes the browser and disposes Playwright. Only the launch call itself is mapped to license errors, "mirrors Python/JS".
- **Headed no-viewport default** is applied in one place (`ViewportDefaults`), shared by the handle and the humanized browser wrapper. It stands in for Python's monkey-patched `_default_no_viewport`. `ResolveContextViewport` ports `_resolve_context_viewport`: explicit `NoViewport` beats `Viewport`; headless below the gate gets 1920x947.
- **`LaunchContextAsync`** resolves GeoIP once, then calls `LaunchAsync` with `GeoIp = false` and `SuppressMaximize` set when a viewport was chosen. It creates the context through the raw browser so the context handle guards it only once.
- **Persistent path.** It prepends the stored `--fingerprint=<seed>` (`Config.PersistentSeedArgs`, publishing atomically with a P/Invoked `link()` on Unix), seeds the Widevine hint, and passes user agent, viewport and color scheme as context options. Timezone and locale stay binary flags. See [[projects/cloakbrowser/subsystems/profile-seed|Profile seed]].
- **Windows-font warning** is the same once-per-environment check, using an in-process flag plus the `.font_warning_shown` marker and `CLOAKBROWSER_SUPPRESS_FONT_WARNING`. The `fc-list` probe drains both pipes concurrently and is bounded at 5 s. `FontWarningTests` pins the hang case and the 20-font macOS set.
- **No CDP emulation guard.** `LaunchContextOptions` simply has no locale/timezone context field, so it cannot route them through CDP emulation. ^[inferred]

## Pinned by tests (B6, JS launch suite)

`js/tests/launch.test.ts`, `puppeteer.test.ts`, `extension-loading.test.ts`, `require.test.ts` and `widevine-integration.test.ts` (commit b1a00ed) mostly use a mocked `playwright-core`/`puppeteer-core` and confirm the pipeline above:

- **Binary flags, not CDP.** `launchContext({ timezone })` puts `--fingerprint-timezone=...` on the launch argv and leaves `timezoneId` out of `newContext()`. Persistent contexts and both Puppeteer entry points emit `--fingerprint-timezone` and `--lang` the same way. `contextOptions.locale` and `contextOptions.timezoneId` are stripped with one warning each, in both `launchContext`/`launchPersistentContext` and `buildContextOptions`.
- **Option precedence.** Top-level `userAgent`, `viewport` and `colorScheme` beat the same keys inside `contextOptions`. Harmless keys (`storageState`, `permissions`, `extraHTTPHeaders`) pass through, and `launchOptions` fields such as `timeout` or `slowMo` reach the framework call.
- **Viewport defaults.** In the test environment (free binary below the no-viewport gate), headless contexts get `DEFAULT_VIEWPORT`, headed ones `null`, and an explicit viewport always wins. The decision reads the effective headless value: `launchOptions.headless` for Playwright and the top-level `headless` for Puppeteer. `setup.ts` isolates the cache directory because a developer's real Pro license or marker would flip this "headless no-viewport shim".
- **Defaults and opt-out.** Every launch carries `--fingerprint=<seed>` and `--no-sandbox` unless `stealthArgs: false` removes the seed. `ignoreDefaultArgs` contains `--enable-automation`. Extra user args are merged in. `extensionPaths` produce absolute `--load-extension` and `--disable-extensions-except` flags.
- **Composable helpers.** `buildLaunchOptions`, `buildContextOptions` and `humanizeBrowser` are exported from the package entry and start no browser. `humanizeBrowser` leaves the browser untouched with `humanize: false`.
- **Lifecycle and errors.** Closing a `launchContext()` context also closes its browser. A launch rejected with license exit code 77 surfaces as `CloakBrowserLicenseError`, and other launch errors are re-thrown unchanged.
- **Persistent-path wiring.** Both persistent launchers call `seedWidevineHint(userDataDir, binaryPath)` with the resolved binary path; the test exists so that removing the call fails CI. See [[projects/cloakbrowser/subsystems/widevine-cdm-seeding|Widevine CDM seeding]].
- **Packaging.** The built `dist/` loads through the package's exports map under ESM `import`, and under CommonJS `require()` where Node supports `require(esm)`, for `cloakbrowser`, `cloakbrowser/puppeteer` and `cloakbrowser/human`.
- **Live stealth smoke test.** It runs only with `CLOAKBROWSER_BINARY_PATH` set: a headless launch must report a falsy `navigator.webdriver` and a non-empty `navigator.plugins`. See [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]].

## Python test-pinned behavior (B7 batch)

`tests/test_build_args.py`, `tests/test_config.py` and `tests/test_extension_loading.py` (commit b1a00ed) mirror the JS pins above for `build_args()` and `config.py`:

- **Priority and dedup**: a user `--fingerprint` or `--fingerprint-platform` replaces the default. The `timezone`/`locale` parameters replace raw `--fingerprint-timezone`, `--lang` and `--fingerprint-locale`. No flag key appears twice. Valueless flags (`--disable-gpu`, `--no-zygote`) pass through, and `--no-sandbox` is always present. Each override is logged at DEBUG on the `cloakbrowser` logger.
- **Timezone/locale are not stealth args**: they are injected even with `stealth_args=False`, which drops only the `--fingerprint=` seed defaults. No timezone or locale flag appears when neither is set.
- **`timezone_id` alias**: it is popped from kwargs, promoted to `timezone`, and loses to an explicit `timezone`.
- **WebRTC**: `--fingerprint-webrtc-ip` is never added by default. `auto` is replaced by the proxy exit IP, and an explicit IP is left untouched.
- **`--start-maximized`**: added only when the `start_maximized` gate is passed in as on, suppressed by a user `--window-size`, and never duplicated.
- **Extensions**: `extension_paths` become absolute paths in both `--load-extension` and `--disable-extensions-except`.
- **Defaults**: two calls give different random seeds (range 10000 to 99999). The persona is `windows` on Linux and `macos` on macOS. **No GPU vendor/renderer flags** are emitted, because the binary derives the GPU from seed plus platform.
- **Gates**: see [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]]. The test comments call the no-viewport threshold "an unshipped version" and `148.0.7778.215.3` the live Pro build. That conflicts with the README's Pro 155 builds. ^[ambiguous]
- **Isolation**: `tests/conftest.py` gives each unit test a temporary `CLOAKBROWSER_CACHE_DIR`, because a real `license.key` or cached Pro marker resolves the machine as Pro and flips version-gated defaults (headless `no_viewport`, inline proxy auth).

## Pinned by Python tests (B8 batch)

`tests/test_launch.py`, `tests/test_launch_context.py` and `tests/test_persistent_context.py` (mock-based except the basic launch tests) confirm the pipeline above:

- **Removed kwarg.** `backend=` raises a clear `TypeError` before anything else runs.
- **Binary flags, not CDP.** `locale` and `timezone` (and the `timezone_id` alias) go to `launch()` as binary flags and never to `new_context()`. The tests give the reason: `--fingerprint-timezone` is read from the command line by the renderer, so it applies to all contexts, while context emulation would be detectable. GeoIP-filled values follow the same route.
- **Kwargs split.** Extra kwargs (for example `record_video_dir`, or `storage_state` in the async path) go to `new_context()` and never to `launch()`. `user_agent` and `color_scheme` are context options.
- **Viewport.** A headless `launch_context()` without a viewport gets `DEFAULT_VIEWPORT` (on the mocked default binary). A headed one gets `no_viewport=True` and no `viewport`. A caller viewport wins. If a caller passes both `viewport=` and `no_viewport=True`, only `no_viewport` reaches Playwright. `_default_no_viewport` (sync and async) never overrides an explicit viewport.
- **Maximize parity with JS/.NET.** No viewport means `launch()` may auto-maximize. An explicit `viewport` or `no_viewport` counts as a chosen geometry and suppresses `--start-maximized`.
- **Cleanup.** Closing the context closes the browser (or stops Playwright for persistent contexts). If `new_context()` raises, including an `asyncio.CancelledError`, the browser is still closed; cleanup catches `BaseException` to avoid leaking browser processes.
- **Event loop.** All three async launchers run the binary/license check, GeoIP and WebRTC lookups off the event loop. A heartbeat coroutine must keep ticking while each mocked lookup sleeps.
- **Order on persistent contexts.** Humanize is patched first, then the license guard is installed on the context and on every page already open. Widevine seeding receives the profile path.

## Design notes

- Headed launches use no viewport, so the page follows the real window. Emulating a viewport on top of a real window would break `outerWidth >= innerWidth` coherence. ^[inferred]
- Version gating lets one wrapper release drive several binary generations, free and Pro, across platforms without emitting flags that an older binary would misread. ^[inferred]
- The history shows the wrapper shedding stealth responsibilities to the binary: hardcoded GPU strings and `--disable-blink-features=AutomationControlled` were removed in 0.3.21, and CDP locale/timezone emulation was replaced by binary flags in 0.3.12. ^[inferred]

## Open Questions

- Why `--no-sandbox` is always set, beyond running as root in Docker, is not explained in these sources.

## Related

- [[projects/cloakbrowser-manager/subsystems/browser-manager|Browser manager (Manager)]] — CloakBrowser-Manager calls the persistent-context launcher once per profile with per-profile flags.

## Sources

- `repo:github.com/CloakHQ/CloakBrowser` at commit `b1a00ed`: `js/src/args.ts`, `js/src/config.ts`, `js/README.md`
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__init__.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/README.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser` at commit `b1a00ed`: `js/src/playwright.ts`, `js/src/puppeteer.ts`, `js/tests/config.test.ts` (B5 batch)
- `repo:github.com/CloakHQ/CloakBrowser` at commit `b1a00ed`: `js/tests/launch.test.ts`, `js/tests/puppeteer.test.ts`, `js/tests/extension-loading.test.ts`, `js/tests/require.test.ts`, `js/tests/setup.ts`, `js/tests/widevine-integration.test.ts` (B6 batch)
- `repo:github.com/CloakHQ/CloakBrowser/tests/test_build_args.py`, `tests/test_config.py`, `tests/test_extension_loading.py`, `tests/conftest.py` (commit `b1a00ed`, B7 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_launch.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_launch_context.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_persistent_context.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/CloakLauncher.cs, Config.cs, LaunchOptions.cs, ViewportDefaults.cs; dotnet/tests/CloakBrowser.Tests/BuildArgsTests.cs, ViewportTests.cs, FontWarningTests.cs (commit b1a00ed, batch D)
