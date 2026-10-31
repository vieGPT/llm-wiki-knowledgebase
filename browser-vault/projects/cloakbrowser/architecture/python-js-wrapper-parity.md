---
title: >-
    Python/JS wrapper parity
category: architecture
tags: [python, typescript, wrapper, parity, cloakbrowser]
aliases: [wrapper parity, mirrored wrappers]
relationships:
  - target: "[[projects/cloakbrowser/architecture/wrapper-binary-split]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/js/package.json
  - repo:github.com/CloakHQ/CloakBrowser/js/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/args.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/download.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/geoip.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/fonts.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/cli.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/engine.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/fields.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/errors.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/examples/stealth-test.ts
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/license.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/widevine.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/injected.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/errors.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/README.md
  - repo:github.com/CloakHQ/CloakBrowser/tests/conftest.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_config.py
  - repo:github.com/CloakHQ/CloakBrowser/js/tsconfig.json
  - repo:github.com/CloakHQ/CloakBrowser/js/vitest.config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/widevine.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/tests/__init__.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/CloakBrowser.csproj
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/CloakLauncher.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Config.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/ProxyResolver.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Handles.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Download.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/License.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/GeoIp.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Widevine.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Human/HumanEngine.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Human/HumanScroll.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Human/WorldHelpers.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser.Generators/InterfaceDelegationGenerator.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser.Cli/Program.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/IsolatedCacheDir.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/LicenseTests.cs
summary: >-
    How the Python, TypeScript and C# wrappers mirror each other module by module, share one on-disk cache, and handle runtime differences (base64, NaN, buffers, sealed interfaces).
provenance:
  extracted: 0.79
  inferred: 0.17
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:42:16+01:00
updated: 2026-10-31T06:54:01+01:00
---

# Python/JS wrapper parity

CloakBrowser ships the same thin wrapper three times: Python (`cloakbrowser/`), TypeScript (`js/src/`) and a community .NET client (`dotnet/`). The TypeScript files open with headers such as "Mirrors Python cloakbrowser/config.py" or "Port of cloakbrowser/human/engine.py", and capability gates carry the note "Python, JS and .NET mirror this gate". Parity is kept by hand-maintained mirroring backed by tests. No code is generated from a shared spec. All three packages (PyPI, npm, NuGet) are at version 0.6.4 (as of 26/10/2026, commit b1a00ed). This page maps the correspondence and lists where the runtimes force deliberate differences. For why the wrappers can stay thin at all, see [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]].

## Module map

| Python (`cloakbrowser/`) | TypeScript (`js/src/`) | Notes |
|---|---|---|
| `config.py` | `config.ts` | Same constants: platform version map, Ed25519 signing keys ("Keep in parity with config.py"), seed range 10000 to 99999, `DEFAULT_VIEWPORT` 1920x947, `IGNORE_DEFAULT_ARGS`, `HEADLESS_NO_VIEWPORT_MIN_VERSION`, per-platform `HTTP_PROXY_INLINE_AUTH_MIN_VERSION` |
| `config.py::persistent_seed_args` | `profile-seed.ts` | Split out into its own module in JS |
| `browser.py` (1,861 lines) | `playwright.ts`, `args.ts`, `proxy.ts`, `fonts.ts` | JS splits the Python monolith: `build_args` becomes `args.ts::buildArgs`, the font probe becomes `fonts.ts`, proxy parsing becomes `proxy.ts` |
| `browser.py::maybe_resolve_geoip`, `_append_webrtc_exit_ip`, `_resolve_webrtc_args` | `geoip.ts::maybeResolveGeoip`, `appendWebrtcExitIp`, `resolveWebrtcArgs` | Same logic, placed in a different module |
| `download.py` | `download.ts` | Near-identical length (1,330 vs 1,328 lines) and function set, including `BinaryVerificationError` |
| `geoip.py` | `geoip.ts` | Different GeoIP readers (see below) |
| `license.py` | `license.ts` | |
| `widevine.py` | `widevine.ts` | JS reimplements `Path.is_file()` and `Path.resolve()` semantics |
| `__main__.py` | `cli.ts` | Same commands and `info` flags (`--quick`, `--json`, `--proxy`, `doctor` alias) |
| `human/{engine,fields,world,patch,injected,keyboard,mouse,config,errors}.py` | `human/*.ts` | JS adds `worldHelpers.ts` and a separate `human-puppeteer/` tree |
| — | `puppeteer.ts`, `human-puppeteer/` | JS only. The Python wrapper drives Playwright only ^[inferred] |
| `_version.py`, `serve` extra | — | The CDP server (`cloakserve`) is Python-side |

The lead's `ast-extract` hub table on [[projects/cloakbrowser/cloakbrowser|CloakBrowser]] shows the paired files with similar degrees, for example `download.ts`/`download.py` and `config.py`/`config.ts`.

## API naming

- Public API names follow each language's convention: `license_key` / `licenseKey`, `launch_persistent_context` / `launchPersistentContext`, `browser_version` / `browserVersion`.
- The humanize configuration keeps **snake_case keys in TypeScript** (`typing_delay`, `mouse_time_base`, `human_config`). Preset values are identical in both ports (default `typing_delay` 150 ms, careful 210 ms, `mistype_chance` 0.04). A humanize config object therefore moves unchanged between languages. ^[inferred]
- The error classes are the same set in all three ports. Python roots them in `RuntimeError`, JS in `Error`, and .NET defines them in `Human/Actionability.cs`. See [[projects/cloakbrowser/subsystems/humanize-errors|Humanize errors]].

## Shared on-disk state

All three wrappers share one cache directory (`~/.cloakbrowser`, or `CLOAKBROWSER_CACHE_DIR`). The file formats are kept compatible so the wrappers can coexist on one machine:

- `.last_update_check` stores **seconds**, not ms, "so the three wrappers share one cache dir without corrupting each other's rate limit". The welcome-banner marker uses the same seconds convention.
- The version markers are `latest_version_<platform>` (plus the legacy `latest_version` from before 0.3.0) and `latest_pro_version[_preview]_<platform>`. They are written atomically through a `.tmp` file and a rename.
- Binary directories are `chromium-<version>` and `chromium-<version>-pro`. Other shared files: `license.key` (mode 0600), `geoip/GeoLite2-City.mmdb` and `.font_warning_shown`.
- The hourly Pro version check in `license.py` keeps `.last_pro_version_check_[preview_]<platform>` plus a JSON resolution sidecar. Python reads the sidecar in snake_case with a camelCase fallback "for sidecars written by an older JS wrapper", and a missing `fallback` field defaults to requested ≠ resolved, which "mirrors JS and .NET". See [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].
- The Widevine hint file is serialized with compact separators and unescaped UTF-8 so Python's output byte-matches the JS wrapper's `JSON.stringify`; the wrappers skip the write when the content is already identical. See [[projects/cloakbrowser/subsystems/widevine-cdm-seeding|Widevine CDM seeding]].
- Humanize selector semantics are tied to the *installed Playwright client*, not to a copy shipped with the wrapper: the Python engine extracts Playwright's InjectedScript from the local driver bundle per version. The isolated-world helper library, however, is copied across ports, and `tests/test_resolver_sources_match.py` checks that the Python, JS and .NET copies stay byte-identical (per `dotnet/README.md`). See [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]].

## The .NET port

The root README calls the C# client "community-maintained", while its project file lists CloakHQ as the author. Its 76 `.cs` files were checked at commit b1a00ed. Most open with "Direct port of Python `cloakbrowser/<module>.py`", and many comments name the JS twin too. The port follows the Python module layout (one `browser.py`-like launcher), not the TypeScript split. Full detail is on [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

### Modules that mirror Python/JS

| .NET (`dotnet/src/CloakBrowser/`) | Python | Mirrored constants and behavior |
|---|---|---|
| `Config.cs` | `config.py` (+ `profile-seed.ts`) | Same per-platform Chromium versions, signing key, seed range 10000-99999, `DefaultViewport` 1920x947, `IgnoreDefaultArgs`, `HeadlessNoViewportMinVersion` `148.0.7778.215.4`, per-platform `HttpProxyInlineAuthMinVersion`; version pins are ASCII-digit regex only |
| `CloakLauncher.cs` + `ProxyResolver.cs` + `ViewportDefaults.cs` | `browser.py` | `BuildArgs` dedup and priority, `MediaRouter` re-enable, `--start-maximized` gate, GeoIP flag promotion, WebRTC `auto`, Windows/Office/macOS font tell lists (8/10/20) |
| `Handles.cs` (`LicenseGuard`) | `browser.py::_install_license_guard` / JS `installLicenseGuard` | Denial-file check after failing **and** succeeding calls |
| `Download.cs` | `download.py` | Pinned-key Ed25519 check, version binding, never-downgrade Pro routing, `PRO_MAJOR` "155" banner, seconds-based rate-limit markers |
| `License.cs` | `license.py` / `license.ts` | Key resolution order and env-injection rules, exit codes 76-79, 24 h validation cache, 1 h Pro version check with the snake_case/camelCase resolution sidecar |
| `GeoIp.cs` | `geoip.py` | Echo services, P3TERX mirror, 20 s timeout, per-process download lock (#458), `en-US` fallback (#591); also the unused `IsPrivateIp` |
| `Widevine.cs` | `widevine.py` | Compact JSON that byte-matches JS `JSON.stringify`; an empty `CLOAKBROWSER_WIDEVINE_CDM` is treated as invalid to match Python/JS |
| `Diagnostics.cs` + `CloakBrowser.Cli/Program.cs` | `__main__.py` | Same commands (`login`, `logout`, `install`, `info`/`doctor`, `update`, `clear-cache`), `info` mirrors the launch, `FormatSeats` "byte-identical" to Python/JS |
| `Human/*` | `human/*` | `HumanEngine` ← `engine.py`, `HumanWorld` ← `world.py`, `InjectedSource` ← `injected.py`, `HumanConfig` ← `config.py` (identical field defaults), `WorldHelpers.cs` byte-identical to `world.py` (checked by `tests/test_resolver_sources_match.py`) |
| `Wrappers/*` | `human/patch.py` | Same set of humanized members and escape hatches |

The test suite is ported along with the code: `LicenseTests.cs` is a "port of Python `tests/test_license.py` and JS `js/tests/license.test.ts`", `IsolatedCacheDir` ports the `conftest.py` cache-isolation fixture, and the real-browser engine tests use the same fixture page as the Python and JS suites.

### Where .NET diverges

- **Interception.** .NET Playwright interfaces cannot be monkey-patched. Humanize uses Roslyn-generated decorators (compile-time, statically typed), and the license guard uses a `DispatchProxy` (runtime reflection). The tests pin the composition order: humanize inside, guard outside. Guarding the raw object instead would put Playwright's own methods in front of the humanized ones, a bug the test comment says "the Python and JS wrappers had". The standalone `HumanScroll` helper is modelled on the JS Puppeteer scroller, not on Python.
- **Async only.** There is no sync API, unlike Python's sync and async surfaces. The launchers return owning handle objects (`CloakBrowserHandle`, `CloakContextHandle`) instead of raw Playwright objects.
- **Playwright internals.** Python reads `Frame._timeout`/`ElementHandle._frame` and gates on Playwright ≥ 1.53. .NET reads private Microsoft.Playwright fields (`_frame`, `_selector`, `_timeoutSettings`) by reflection, with no version gate. It pins Microsoft.Playwright 1.64.1 (as of commit b1a00ed).
- **Crypto and GeoIP dependencies.** These are hard NuGet references (BouncyCastle for Ed25519, MaxMind.GeoIP2), not optional extras or peers. `info` therefore treats a missing module as a broken deployment. SOCKS5 echo calls go through .NET's own `HttpClient` proxy support, with no `socksio`/`socks-proxy-agent` equivalent.
- **Wrapper update check.** It queries the NuGet flat-container index instead of PyPI or npm.
- **Atomic seed publish.** On Unix it P/Invokes libc `link()`, because `File.Move(overwrite: false)` checks and then renames, which lets concurrent callers replace each other ("seen on macOS"). On Windows a no-replace `File.Move` is atomic.
- **License error mapping.** .NET Playwright's launch failure message carries no exit code ("Target ... has been closed"), so `LicenseErrorFrom` falls back to the denial file even for launch failures.
- **Saved key permissions.** The Python and JS CLIs `chmod 0600` the saved `license.key`. The .NET `login` writes it with `File.WriteAllText` and sets no mode. ^[inferred]
- **Console output.** The .NET `info` prints ✓/✗/→ glyphs directly. It has no per-glyph ASCII fallback like Python's for legacy consoles, although the welcome banner is ASCII-only "to keep the three wrappers byte-parallel".
- **Zip-slip.** `dotnet/README.md` advertises an extra zip-slip guard in `Download.cs`. The code has none beyond the signature check and the BCL extractors (see [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]]). ^[ambiguous]

## Runtime differences handled explicitly

The code comments name each spot where a runtime would otherwise diverge:

- **Base64 strictness**: Node's `Buffer.from(..., "base64")` silently drops invalid characters. `verifySignature` therefore round-trips the decoded signature and rejects it on mismatch, "parity with Python's `base64.b64decode(validate=True)`".
- **Malformed versions**: `parseVersion` yields `NaN` instead of throwing. The JS gates check for `NaN` explicitly so they fail OFF, as Python and .NET do when they throw.
- **Subprocess buffers**: `fc-list` output can exceed Node's default `maxBuffer` of 1 MB. JS raises it to 16 MB because "Python/.NET have no such cap".
- **Optional dependencies**: GeoIP uses `geoip2` plus `socksio` (Python extra `geoip`) against `mmdb-lib` plus `socks-proxy-agent` (optional npm peer dependencies). Both read the same P3TERX GeoLite2-City mirror.
- **Seat display**: `formatSeats` is "kept identical in the Python and .NET wrappers", so all CLIs print the same seat strings.
- **Test parity**: `DEFAULT_GEOIP_TIMEOUT_MS` is exported "for parity testing", and `js/examples/stealth-test.ts` "Mirrors Python examples/stealth_test.py".

## Test infrastructure (B7 batch)

- **Parity named in the tests themselves**:
  - the viewport and maximize gates are marked "parity-critical: JS and .NET mirror this";
  - version pins reject Unicode digits "so the same input behaves identically across Python / JS / .NET";
  - the humanize error set is the "same compatibility set as JS errors.ts / .NET Actionability.cs".
  Most Python unit files have a JS twin under `js/tests/`, such as `test_build_args.py`/`config.test.ts` and `test_geoip.py`/`geoip.test.ts`. ^[inferred]
- **Python harness**: `tests/` is a package (an empty `__init__.py`), which lets a subprocess scenario import `tests.test_humanize_engine`. `tests/conftest.py` points every unit test at a fresh temporary `CLOAKBROWSER_CACHE_DIR`. Without it, a developer's real `license.key` or Pro marker would resolve the machine as Pro and flip version-gated defaults (headless `no_viewport`, inline proxy auth). Tests marked `slow` or `real_browser` opt out on purpose and use the developer's key and binary at a stable path. A fresh path per test would re-download the binary and re-prompt path-keyed firewalls (LuLu) for the ad-hoc-signed macOS build.
- **JS harness**: `tsconfig.json` compiles `src/` to `dist/` with `strict`, ES2022, `NodeNext` modules, declarations and source maps. `tests/` and `examples/` are excluded from the build. `vitest.config.ts` runs a shared `tests/setup.ts` first. Test files such as `widevine.test.ts` isolate `CLOAKBROWSER_CACHE_DIR` and redefine `process.platform` to cover Linux-only paths on any host.
- **Manual scripts**: `tests/*.mjs` import the built `js/dist` and run by hand against live sites, outside vitest.

## Packaging (as of 26/10/2026, commit b1a00ed)

- npm `cloakbrowser` 0.6.4 is an ESM package. Its subpath exports are `.` (Playwright), `./puppeteer` and `./human`, plus a `cloakbrowser` bin (`dist/cli.js`). It needs Node `^20.19.0 || >=22.12.0`.
- `tar` is the only hard dependency. `playwright-core` (>=1.53), `puppeteer-core` (>=21), `mmdb-lib` (>=2) and `socks-proxy-agent` (>=10) are all optional peers, so users install only the backend they use. See [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].
- On PyPI, `pyproject.toml` declares the optional extras `geoip`, `serve` and `dev`.
- `js/README.md` repeats large parts of the root README (test table, platform table, environment variables, troubleshooting). That duplication is a likely source of documentation drift. ^[inferred]

## Why mirror by hand

- Each wrapper has to feel native to its ecosystem: async Playwright in Node, sync and async in Python, NuGet in .NET. The real stealth logic lives in the binary, so the duplicated surface stays small. ^[inferred]
- The cost is drift risk. Constants such as signing keys, version floors and the Pro major (`PRO_MAJOR = "155"`) must be bumped in every port. The repository relies on comments and tests to catch misses. ^[inferred]

## Open Questions

- Python `geoip.py` defines `_is_private_ip`, but nothing in `cloakbrowser/` calls it, and the JS port has no equivalent (as of commit b1a00ed). `tests/test_geoip.py` unit-tests it, which keeps it alive without settling whether it is dead code or a missing JS guard. ^[ambiguous]
- Resolved (batch D, commit b1a00ed): the .NET client matches the Python constants and gates. It diverges in the ways listed under [[#Where .NET diverges]]. `GeoIp.IsPrivateIp` is as unused in C# as `_is_private_ip` is in Python; only `MiscTests` calls it, so the C# port copied the dead helper. ^[inferred]

## Sources

- `repo:github.com/CloakHQ/CloakBrowser` at commit `b1a00ed`: `js/package.json`, `js/README.md`, `js/src/config.ts`, `js/src/args.ts`, `js/src/download.ts`, `js/src/geoip.ts`, `js/src/fonts.ts`, `js/src/cli.ts`, `js/src/human/engine.ts`, `js/src/human/fields.ts`, `js/src/human/config.ts`, `js/src/human/errors.ts`, `js/examples/stealth-test.ts`; Python counterparts in `cloakbrowser/` consulted for comparison; `README.md`
- B1 additions at commit `b1a00ed`: `cloakbrowser/license.py`, `cloakbrowser/widevine.py`, `cloakbrowser/human/injected.py`, `cloakbrowser/human/errors.py`, `dotnet/README.md`
- B7 additions at commit `b1a00ed`: `tests/conftest.py`, `tests/__init__.py`, `tests/test_config.py`, `js/tsconfig.json`, `js/vitest.config.ts`, `js/tests/widevine.test.ts`
- Batch D (.NET source) at commit `b1a00ed`: `dotnet/src/CloakBrowser/CloakBrowser.csproj`, `CloakLauncher.cs`, `Config.cs`, `ProxyResolver.cs`, `Handles.cs`, `Download.cs`, `License.cs`, `GeoIp.cs`, `Widevine.cs`, `Human/HumanEngine.cs`, `Human/HumanScroll.cs`, `Human/WorldHelpers.cs`; `dotnet/src/CloakBrowser.Generators/InterfaceDelegationGenerator.cs`; `dotnet/src/CloakBrowser.Cli/Program.cs`; `dotnet/tests/CloakBrowser.Tests/IsolatedCacheDir.cs`, `LicenseTests.cs`
