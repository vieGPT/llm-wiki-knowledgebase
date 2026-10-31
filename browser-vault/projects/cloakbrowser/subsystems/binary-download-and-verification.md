---
title: >-
    Binary download and verification
category: subsystems
tags: [chromium, supply-chain, caching, auto-update, licensing]
aliases: [ensure_binary, ensureBinary, download.py]
relationships:
  - target: "[[projects/cloakbrowser/architecture/supply-chain-verification]]"
    type: implements
  - target: "[[projects/cloakbrowser/architecture/release-channels-and-versioning]]"
    type: implements
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/runtime-and-binary]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/cloakbrowser/download.py
  - ~/code/CloakBrowser/cloakbrowser/config.py
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/js/src/download.ts
  - ~/code/CloakBrowser/js/src/config.ts
  - ~/code/CloakBrowser/js/README.md
  - ~/code/CloakBrowser/js/examples/stagehand.ts
  - ~/code/CloakBrowser/js/tests/update.test.ts
  - ~/code/CloakBrowser/js/tests/signature.test.ts
  - ~/code/CloakBrowser/js/tests/welcome.test.ts
  - ~/code/CloakBrowser/js/tests/launch.test.ts
  - ~/code/CloakBrowser/tests/test_extract.py
  - ~/code/CloakBrowser/tests/test_free_version_pin.py
  - ~/code/CloakBrowser/tests/test_config.py
  - ~/code/CloakBrowser/tests/test_update.py
  - ~/code/CloakBrowser/tests/test_welcome_banner.py
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Download.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/SignatureTests.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/DownloadConfigTests.cs
summary: >-
    ensure_binary resolves which Chromium build to run, downloads it from cloakbrowser.dev or GitHub, verifies a pinned Ed25519 signature, extracts atomically, and auto-updates.
provenance:
  extracted: 0.89
  inferred: 0.08
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:44:40+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Binary download and verification

`download.py` (`download.ts` in JS, the highest-degree non-test hub at 42 as of 31/10/2026) owns everything between "the user called `launch()`" and "a verified Chromium executable path exists on disk". Its public surface is `ensure_binary()`, `binary_info()`, `check_for_update()`, `clear_cache()`, plus `check_for_pro_update()` used by the CLI. It is called first in every launch path (see [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]) and by `cloakbrowser install` (see [[projects/cloakbrowser/subsystems/cli|CLI]]).

## Resolution order in `ensure_binary()`

1. **Local override**: `CLOAKBROWSER_BINARY_PATH` is returned as-is (error if missing). No download, no update checks.
2. **Version pin** normalized from `browser_version` or `CLOAKBROWSER_VERSION`.
3. **License routing**: resolve the key (argument, env, or `~/.cloakbrowser/license.key`), unless `CLOAKBROWSER_DOWNLOAD_URL` is set, which disables the Pro path. Then validate it:
   - valid → Pro path (`_ensure_pro_binary`); a `free`-plan key has its pin dropped because the server force-serves latest to free keys;
   - rejected (invalid/expired) or unvalidatable (server down, no cache) → raise `CloakBrowserLicenseError`. Since 0.5.10 a supplied key never silently downgrades to the free binary.
4. **Free path**: fail fast if the platform has no prebuilt binary; serve a pinned version from cache or download it; otherwise use the newest auto-updated version (marker file), falling back to the wrapper's hardcoded per-platform version.

Tiers and validation caching are covered in [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]]; channel and pin semantics in [[projects/cloakbrowser/architecture/release-channels-and-versioning|Release channels and versioning]].

## Pro path

- **Pinned**: use the cached `chromium-<v>-pro` build or download exactly that version; never write the "latest" marker, so a rollback pin cannot stick.
- **Unpinned**: read the channel's cached marker, ask the server for the channel's latest (rate-limited to one call per hour), and pick the newer of the two. With `CLOAKBROWSER_AUTO_UPDATE=false` and a cached build, skip the server entirely.
- Download from `cloakbrowser.dev/api/download/<version>` with `Authorization: Bearer <key>` and an `X-Platform` header, requesting the explicit version so the bytes match the signed manifest.
- If downloading a newer build fails transiently, launch the cached Pro build; if nothing is cached, raise. **Never fall back to the free binary.** A verification failure always propagates.
- A Preview request with no Preview build for the platform warns once on stderr and uses Stable.

## Download, verify, extract

- **Sources**: primary `cloakbrowser.dev/chromium-v<v>/cloakbrowser-<platform>.<tar.gz|zip>`, fallback GitHub Releases, unless a custom URL is set, in which case there is no fallback.
- Download to a temp file first so the cache never holds a partial archive; httpx streaming with a 10 s connect / 60 s read timeout.
- **Verify** before extraction: fetch `SHA256SUMS` + `SHA256SUMS.sig` from the same origin, verify Ed25519 against pinned keys, check the manifest's `version=` line equals the requested version, then compare the archive SHA-256. Mandatory on official and Pro paths; the legacy unsigned checksum (skippable with `CLOAKBROWSER_SKIP_CHECKSUM`) only applies to self-hosted mirrors. See [[projects/cloakbrowser/architecture/supply-chain-verification|Supply-chain verification]].
- **Extract** into a private `<dir>.partial-<uuid>` sibling, flatten a single wrapping directory (never a macOS `.app` bundle), `chmod +x`, strip macOS quarantine xattrs, then rename into place. If the destination exists, keep it when it already holds a complete executable install, otherwise move the leftover aside to `.stale-<uuid>` and retry (up to 5 times). On Windows, a rename denied by antivirus is retried for about 2.5 s.
- This staging design (0.6.0, #555) fixed concurrent first launches deleting each other's half-written installs, which surfaced as `ETXTBSY` on Linux and `dlopen` failures on macOS. ^[inferred]

## Cache layout

Under `CLOAKBROWSER_CACHE_DIR` (default `~/.cloakbrowser/`):

- `chromium-<version>/` (free) and `chromium-<version>-pro/` (Pro) binary dirs; the executable is `chrome`, `chrome.exe`, or `Chromium.app/Contents/MacOS/Chromium`;
- markers `latest_version_<platform>` (free), `latest_pro_version_<platform>`, `latest_pro_version_preview_<platform>`, written atomically via temp-file rename;
- `.last_update_check`, `.welcome_shown`, `.font_warning_shown`, `license.key`, plus `geoip/` and `WidevineCdm/` used by other subsystems.

## Auto-update

- **Free binary**: after a cache hit, a daemon thread checks GitHub Releases (at most hourly), picks the newest non-draft `chromium-v*` release that has an asset for this platform, downloads it in the background and writes the marker; the next launch uses it.
- **Wrapper**: once per process, checks PyPI and logs an upgrade hint if a newer `cloakbrowser` exists.
- Disabled by `CLOAKBROWSER_AUTO_UPDATE=false`, a local override, or a custom download URL.
- The Pro check is a rate-limited foreground check rather than a background thread (0.4.9), so `info` and the actual launch cannot disagree about which build runs. ^[inferred]

## Welcome banner

Printed to stderr (never stdout, so it cannot corrupt JSON or MCP stdio). It is tier-aware: keyless invites the free login, free shows "1 concurrent session", Pro shows the support address. Paid keys see it once; free users again after 1 day (`WELCOME_FREE_INTERVAL = 24 * 3600` in Python, `WELCOME_FREE_INTERVAL_SEC = 1 * 24 * 60 * 60` in JS, `24L * 3600` in .NET). ASCII-only with every write guarded, since a cp1252 Windows console once aborted launches on a non-ASCII glyph (0.5.10). The CHANGELOG entry for 0.4.5 still says free users see it every 3 days; all three wrappers use 1 day, so that CHANGELOG line is stale documentation rather than an open contradiction.

## JS port (`download.ts`)

`download.ts` "Mirrors Python cloakbrowser/download.py" function for function (1,328 vs 1,330 lines at commit b1a00ed). The JS-specific details:

- **Timeouts**: one overall 10-minute `AbortController` timeout per download, instead of httpx's separate connect and read timeouts.
- **Background work**: the binary and wrapper update checks are fire-and-forget promises with errors swallowed, not daemon threads. The wrapper check queries the npm registry (`registry.npmjs.org/cloakbrowser/latest`) instead of PyPI.
- **Shared rate limits**: `.last_update_check` and the welcome-banner interval are stored in **seconds**, so the Python, JS and .NET wrappers can share one cache dir without breaking each other's rate limits. `WELCOME_FREE_INTERVAL_SEC` is 1 day here too, matching Python (see Welcome banner above).
- **Banner constant**: `PRO_MAJOR = "155"` is hardcoded for the banner (as of 26/10/2026, commit b1a00ed) because the live Pro version would need a network call.
- **Error wording**: when a valid Pro license cannot download and nothing is cached, the error tells the user to retry or to unset `CLOAKBROWSER_LICENSE_KEY` to switch to the free binary on purpose.
- **Pre-fetching**: the JS README suggests calling `ensureBinary()` during a Docker build so the binary is baked into the image. `examples/stagehand.ts` uses `ensureBinary()` directly to hand the path to another framework (see [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]]).

Cross-language details are on [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].

## .NET port (`Download.cs`, batch D)

`Download.cs` (1,453 lines at commit b1a00ed) is a "Direct port of Python `cloakbrowser/download.py`". It keeps the resolution order, the never-downgrade Pro routing, explicit-version Pro URLs, version binding, the free updater and the staged extraction. The .NET-specific details:

- **Ed25519 via BouncyCastle.** .NET 8 has no built-in Ed25519, so `VerifySignature` uses the `BouncyCastle.Cryptography` package. It tries every pinned key, skips an unparseable one, and fails closed. A Pro manifest fetch failure is a plain `InvalidOperationException` (transient). Signature, checksum and version mismatches are converted to `BinaryVerificationError`, which `EnsureBinaryAsync` re-throws verbatim.
- **API shape.** `EnsureBinaryAsync` (plus a sync `EnsureBinary`) takes a `CancellationToken`. License validation, which is blocking HTTP, runs on `Task.Run`. A single `HttpClient` has a 10-minute timeout and a `cloakbrowser-dotnet/<version>` User-Agent.
- **Extraction.** It uses the BCL `TarFile.ExtractToDirectory` (over `GZipStream`) and `ZipFile.ExtractToDirectory` unchanged, with the comment "Signature-verified before extraction, so unpack as-is". Like Python, it does no per-member sanitization, and it preserves symlinks for the macOS `.app` layout. The `dotnet/README.md` claim of an extra "zip-slip guard" in this file is not borne out by the code. ^[ambiguous]
- **Windows rename retries** match the HRESULTs for access denied, sharing violation and lock violation: 10 attempts × 250 ms.
- **Wrapper update check** queries the NuGet flat-container index (`api.nuget.org/v3-flatcontainer/cloakbrowser/index.json`). It skips prerelease versions and suggests `dotnet add package CloakBrowser`.
- **Background work** is fire-and-forget `Task.Run`, gated per process for the wrapper check and hourly (seconds-based `.last_update_check`, shared with the other wrappers) for the binary.
- **Tests.** `SignatureTests.cs` (22 cases) ports the Python `TestSignatureVerification`/`TestVersionBinding` classes and `js/tests/signature.test.ts`. `DownloadConfigTests.cs` covers version comparison, checksum parsing, the NuGet version compare and concurrent first runs against a local `HttpListener` mirror. See [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

## Pinned by tests (B6, JS suite)

`js/tests/update.test.ts`, `signature.test.ts`, `welcome.test.ts` and `launch.test.ts` (commit b1a00ed) pin the following:

- **Concurrent first runs.** Four first-run callers in one process, with three downloads held back until the first caller has returned, never expose an incomplete install: every inspection reads the full binary plus all library files, executable bit set. A partial install left by an interrupted extraction is replaced, both in-process and when four separate processes race against a local mirror. Afterwards the cache directory contains only the version directory. These tests are skipped on Windows.
- **stdout stays clean.** A first-run download writes nothing to stdout, because MCP stdio servers use stdout as their JSON-RPC channel. The GeoIP database download follows the same rule (see [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]]).
- **Pro never degrades to free.** `getEffectiveVersion(pro=true)` returns null when nothing is cached or when the marker names a binary that is missing, while the free tier still resolves to the hardcoded version. `ensureBinary` with a rejected key throws "invalid or expired", and with an unreachable server and no cache it throws "could not be validated".
- **Version ordering.** Four- and five-part versions compare numerically, and a build suffix ranks above the same four-part version without one (`145.0.7632.109.2` is newer than `145.0.7632.109`).
- **Free URL and updater.** Free URLs point at cloakbrowser.dev under `chromium-v<version>/...tar.gz`, never the old `chromium-stealth-builds` repository. The free updater skips draft releases and releases without an asset for this platform, and returns null on network errors. The checksum fetch falls back to GitHub when the primary answers 429 and returns null when both fail.
- **Wrapper update check** runs once per process. It warns "Update available" only for a newer version, stays silent on network errors, and is disabled by `CLOAKBROWSER_AUTO_UPDATE=false` or a custom `CLOAKBROWSER_DOWNLOAD_URL`.
- **Overrides.** A missing `CLOAKBROWSER_BINARY_PATH` target fails with "does not exist", and `clearCache` tolerates a missing cache directory.
- **Pro download URL** names the explicit version (`/api/download/<version>`), never `/latest`.
- **Welcome banner.** Pro sees it once, ever. Free sees it again after `WELCOME_FREE_INTERVAL_SEC`. A legacy empty marker re-shows it for free users and keeps Pro silent. Keyless, free and Pro banners are all pure ASCII.

### Contradiction: transient Pro verification failure

A comment in `signature.test.ts` says a failed manifest *fetch* must stay a plain `Error` so that "router falls back to free", as opposed to `BinaryVerificationError`, which is re-thrown. That conflicts with the Pro path described above and with `update.test.ts` ("A valid Pro license must NEVER fall back to the free binary"). The likelier reading is that the transient error lets `ensureBinary` fall back to a cached Pro build or raise "Pro binary unavailable ... retry", and that the comment predates the 0.5.10 no-downgrade rule. The test asserts only the error type, not the fallback target. ^[ambiguous]

## Pinned by tests (B7, Python suite)

Commit b1a00ed:

- **Free key drops the pin** (`tests/test_free_version_pin.py`). With a validated `free`-plan key, `ensure_binary(browser_version=…)` enters the Pro path with the requested version set to `None`. A `business` key passes the pin through unchanged. The test docstring gives the reason: the server force-serves latest to free keys, so "a pinned manifest would fail checksum".
- **Extraction** (`tests/test_extract.py`):
  - tar.gz and zip archives extract with nested directories.
  - **Symlinks are extracted as-is, with no per-member sanitization.** The archive is signature-verified before extraction, and the macOS `.app` Framework layout depends on symlinks.
  - A single wrapping directory is flattened, but a `Chromium.app` bundle is never flattened, and a top level with several entries is left alone.
  - The binary is made executable.
- **Concurrent first runs** (Python mirror of the JS test above): four simultaneous cold `ensure_binary()` callers against a local mirror all return the same path. An observer reading the install while late callers finish only ever sees a complete install. A partial install left by an interrupted extraction is replaced, whether one caller or four race against it. No staging or stale directories remain afterwards. Skipped on Windows.
- **Platform layout** (`tests/test_config.py`):
  - The executable is `chromium-<v>/chrome` on Linux, `chrome.exe` on Windows, and `Chromium.app/Contents/MacOS/Chromium` on macOS.
  - Archives are `.zip` on Windows and `.tar.gz` elsewhere, named `cloakbrowser-<platform-tag><ext>`.
  - The fallback URL is `github.com/CloakHQ/cloakbrowser/releases/download/chromium-v<v>/…`.
  - An unsupported OS/arch (e.g. FreeBSD) raises "Unsupported platform".
  - `CLOAKBROWSER_CACHE_DIR` overrides the `~/.cloakbrowser` default.
  - An explicit pin beats `CLOAKBROWSER_VERSION`. Path traversal and non-ASCII digits are rejected ("Invalid browser version pin"), for parity with the JS and .NET wrappers.
- **Welcome banner interval**: not covered by the Python tests in this batch (see the B8 banner-cadence tests below).

Trade-off: unsanitized symlink extraction is safe only as long as the archive really was verified. On a self-hosted mirror with `CLOAKBROWSER_SKIP_CHECKSUM`, an unverified archive's symlinks would land unfiltered. ^[inferred]

## Pinned by Python tests (B8 batch)

`tests/test_update.py` (the largest file in the batch, 1,512 lines at commit b1a00ed) and `tests/test_welcome_banner.py` confirm the flow above and fix these details:

- **Version ordering.** Versions compare as integer tuples with an optional fifth "build" segment. `145.0.7632.109.2` is newer than `145.0.7632.109`, and any newer Chromium beats an older build suffix.
- **Update gating.** Checks are skipped when `CLOAKBROWSER_AUTO_UPDATE` is `false` (case-insensitive), when a local binary override or custom download URL is set, and within the hourly rate limit. A stale timestamp allows a check. A cached marker for an older version than the wrapper's is ignored. The free release scan skips drafts, non-`chromium-v*` tags and releases without an asset for this platform. The PyPI wrapper check runs once per process and stays silent on network errors.
- **Fallback origin.** A binary or checksum fetch that fails on cloakbrowser.dev retries on GitHub Releases, except with a custom URL. `SHA256SUMS` and its `.sig` must come from the same origin.
- **Signatures.** Valid signatures pass. A tampered manifest, a wrong key or a malformed signature fails. An unparseable placeholder entry in the pinned key list is skipped rather than crashing. A manifest signed with a second pinned key is accepted, which allows key rotation. The `version=` line must match the requested version, so a downgrade fails even with a valid signature. Old checksum parsers ignore the `version=` line.
- **Pro parity.** Pro downloads get the same signature check, and `CLOAKBROWSER_SKIP_CHECKSUM` does not weaken it. The Pro download URL names the exact version (`/api/download/<v>`, never `/latest`), so the served bytes cannot advance past the signed manifest between fetch and verify.
- **Unpinned Pro.** The wrapper upgrades to the server's latest. A newer cached build advances the marker without a download. A server outage uses the cached Pro build, a failed download falls back to the cached build, and a verification error is surfaced instead of being masked by the cache. No cache and no server raises an error, never the free binary. `get_effective_version(pro=True)` returns `None` rather than the free base version.
- **Banner cadence.** Pro sees the banner once, ever. Free sees it again after `WELCOME_FREE_INTERVAL`. A pre-cadence empty marker re-shows for free but counts as "already shown" for Pro. The banner is pure ASCII and never raises, even on a strict cp1252 stderr. The tests use the constant, not a literal; the 1-day value comes from the constant itself, so the CHANGELOG's 3 days is stale (see Welcome banner above).

One test docstring contradicts the routing tests. `test_missing_manifest_is_transient_not_tampering` says a failed Pro manifest fetch is transient because "router falls back to free". The routing tests in `tests/test_license.py` require transient Pro failures to hard-error and never reach the free binary. The docstring looks stale from before 0.5.10; the behavior it actually asserts (a plain `RuntimeError`, not `BinaryVerificationError`) matches the "transient, retry" path. ^[ambiguous]

## Open Questions

- The server-side contract for `/api/download/<version>` (entitlement checks, plan-specific builds) is not visible from the wrapper.

## Related

- [[projects/cloakbrowser-manager/subsystems/runtime-and-binary|Runtime and binary (Manager)]] — CloakBrowser-Manager calls `ensure_binary` once at startup and after a settings change.

## Class stubs (ast-extract)

- [[projects/cloakbrowser/subsystems/binary-verification-error|BinaryVerificationError]]

## Sources

- `~/code/CloakBrowser/cloakbrowser/download.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/README.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/js/src/download.ts`, `js/src/config.ts`, `js/README.md`, `js/examples/stagehand.ts` (commit `b1a00ed`, B4 batch)
- `~/code/CloakBrowser/js/tests/update.test.ts`, `js/tests/signature.test.ts`, `js/tests/welcome.test.ts`, `js/tests/launch.test.ts` (commit `b1a00ed`, B6 batch)
- `~/code/CloakBrowser/tests/test_extract.py`, `tests/test_free_version_pin.py`, `tests/test_config.py` (commit `b1a00ed`, B7 batch)
- ~/code/CloakBrowser/tests/test_update.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/tests/test_welcome_banner.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/dotnet/src/CloakBrowser/Download.cs, dotnet/tests/CloakBrowser.Tests/SignatureTests.cs, dotnet/tests/CloakBrowser.Tests/DownloadConfigTests.cs (commit b1a00ed, batch D)
