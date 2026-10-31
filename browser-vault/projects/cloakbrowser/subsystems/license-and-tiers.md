---
title: >-
    License and tiers
category: subsystems
tags: [licensing, pro-tier, session-seats, chromium, browser-automation]
aliases: [license.py, CloakBrowser Pro, CloakBrowserLicenseError]
relationships:
  - target: "[[projects/cloakbrowser/architecture/release-channels-and-versioning]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/cloakbrowser/license.py
  - ~/code/CloakBrowser/cloakbrowser/download.py
  - ~/code/CloakBrowser/dotnet/README.md
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/js/src/license.ts
  - ~/code/CloakBrowser/js/src/playwright.ts
  - ~/code/CloakBrowser/js/src/puppeteer.ts
  - ~/code/CloakBrowser/js/tests/cli.test.ts
  - ~/code/CloakBrowser/js/tests/license.test.ts
  - ~/code/CloakBrowser/js/tests/launch.test.ts
  - ~/code/CloakBrowser/js/tests/puppeteer.test.ts
  - ~/code/CloakBrowser/js/tests/setup.ts
  - ~/code/CloakBrowser/js/tests/humanizeEngine.test.ts
  - ~/code/CloakBrowser/tests/test_license.py
  - ~/code/CloakBrowser/tests/test_license_error.py
  - ~/code/CloakBrowser/tests/test_login.py
  - ~/code/CloakBrowser/tests/test_welcome_banner.py
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/License.cs
  - ~/code/CloakBrowser/dotnet/src/CloakBrowser/Handles.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/LicenseTests.cs
  - ~/code/CloakBrowser/dotnet/tests/CloakBrowser.Tests/Wrappers/LicenseGuardCompositionTests.cs
summary: >-
    How the wrapper resolves and validates a Pro license key, injects it into the binary's env, checks the latest Pro build, counts seats, and maps binary exit codes 76-79 to clear errors.
provenance:
  extracted: 0.87
  inferred: 0.11
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T10:47:25+01:00
---

# License and tiers

`cloakbrowser/license.py` is the wrapper's licensing layer: it finds a license key, validates it against cloakbrowser.dev with local caching, decides whether the key must be injected into the browser process environment, asks the server which Pro build is current for this platform and channel, reports live seat usage, and turns the Pro binary's numeric license exit codes into readable errors. The license decision itself is enforced by the Pro binary; the wrapper only resolves inputs and explains denials. ^[inferred] It is mirrored in `js/src/license.ts` and the .NET `License` class (see [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]]).

## Tiers

As described in the READMEs (as of 26/10/2026, commit b1a00ed):

| Tier | Binary | Concurrency | How obtained |
|---|---|---|---|
| Free latest | newest build (Chromium 155) | one concurrent session | key via GitHub sign-in (`cloakbrowser login`) |
| Pro | newest build first, Linux x64/ARM64, Windows x64, macOS | plans for 5, 20, 200, 2,000 or more sessions | paid key |
| Old free | v146 on GitHub Releases, no key | n/a | default download without a key |

The wrappers themselves are MIT. A valid key downloads the latest Pro binary from cloakbrowser.dev; without one, the free binary comes from GitHub Releases. Both are authenticated with the same pinned Ed25519 signature, and a key whose Pro download or signature check fails raises an error rather than silently downgrading (see [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]] and [[projects/cloakbrowser/architecture/supply-chain-verification|Supply-chain verification]]). Pinning an exact `browser_version` is a paid-Pro feature; a free GitHub key ignores the pin. The reason is in `download.py`: the server force-serves the latest build to free keys, so a pinned version's signed manifest would not match the served bytes and checksum verification would fail.

## How a key routes the binary (`download.py`, P2 pass)

`ensure_binary()` is where the license decides which binary runs. The full download logic is on [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]]; the license-relevant rules are:

- **Keys can be switched off by configuration.** A local `CLOAKBROWSER_BINARY_PATH` override returns before any key is looked at, and a custom `CLOAKBROWSER_DOWNLOAD_URL` discards the resolved key, so self-hosted downloads never take the Pro path.
- **Free and paid keys share one path.** Any key that validates goes to `_ensure_pro_binary`, which fetches the same latest build from cloakbrowser.dev with the key as a Bearer token. The concurrency difference is enforced by the server and the binary, not by the wrapper ("the server enforces the concurrency cap"). Only the welcome banner differs: `free` gets the free banner, every other plan the Pro one.
- **Three ways a key can fail, none of them a downgrade.** A key the server rejects raises `CloakBrowserLicenseError` naming the plan; a key that cannot be validated at all (server down, no cached validation) raises `CloakBrowserLicenseError` asking to retry; a valid key whose Pro download fails with nothing cached raises "Pro binary unavailable". Each message tells the user to unset `CLOAKBROWSER_LICENSE_KEY` if they want the free binary, so the downgrade is always explicit. A signature failure is re-raised verbatim as a tampering signal.
- **Cached Pro builds absorb outages.** An update that fails falls back to the cached Pro build, never the free one, and `CLOAKBROWSER_AUTO_UPDATE=false` keeps a cached Pro build without asking the server. With no cached Pro build, a valid key must download one.
- **Welcome banner cadence**: a Pro key sees it once ever; free keys and keyless users see it again every 24 hours (`WELCOME_FREE_INTERVAL`, 1 day in all three wrappers). The CHANGELOG entry for 0.4.5 says every 3 days; that line is stale, the code is authoritative. The free banner advertises the latest major as a hardcoded `PRO_MAJOR` (`155` as of 26/10/2026, commit b1a00ed), bumped by hand at each Pro major because the wrapper does not call the network just to print a banner.

`login` and `logout` live in the CLI: `login` saves a server-accepted key as `license.key`, which the resolution order above then picks up, and `logout` removes it so launches revert to the keyless free binary (see [[projects/cloakbrowser/subsystems/cli|CLI]] and "Pinned by Python tests" below).

## Key resolution and env injection

Resolution order: explicit `license_key` parameter, then `CLOAKBROWSER_LICENSE_KEY`, then `license.key` in the wrapper cache directory. The resolver also records *where* the key came from, because that decides whether the key must be injected into the child process environment:

| Source | Injected into browser env? | Reason given in code |
|---|---|---|
| parameter or custom cache-dir file | yes | the binary cannot see a wrapper-only key or a non-default file path |
| environment variable | no, unless a custom `env=` is passed | the child inherits the parent environment |
| default `~/.cloakbrowser/license.key` | no, unless a custom `env=` is passed | the binary reads that file itself, which keeps the key out of the process environment |
| none | no | |

Whenever the caller passes their own `env=`, the key is injected anyway, because Playwright *replaces* rather than merges the child environment and a replaced env can drop `HOME` and hide the default file.

## Validation and caching

- `validate_license` posts the key to the validate endpoint and caches a positive result for 24 hours in the cache directory, keyed by the key's SHA-256 rather than the key itself.
- If the server is unreachable, a stale cache entry is accepted with a warning; total failure returns `None`.
- A cached entry whose `expires` date has passed is reported as invalid regardless of TTL.
- Unknown plan values default to `solo`.

## Pro release resolution

`get_pro_latest_release` asks the version endpoint for the current Pro build, sending the platform tag in an `X-Platform` header and `channel=preview` when requested. The answer (version, requested channel, resolved channel, fallback flag) is cached for one hour per platform and channel in two marker files written atomically. Offline, the last answer is reused; if only an old version marker exists, it is labelled a conservative stable fallback. Preview resolves to the newest build for the platform, falling back to stable when no newer preview exists. Channel semantics belong to [[projects/cloakbrowser/architecture/release-channels-and-versioning|Release channels and versioning]].

## Seats

`get_session_seats` returns the live seat count, the plan cap, and a `state`: `ok`, `unreachable` (never reached the server), `denied` (server refused, with its error code such as invalid key, inactive license or rate limited), or `unknown` (server up and key valid, but it cannot count right now, for example in leaseless mode). The docstring explains the redesign: these cases used to collapse into one bare `None`, so `info` printed the same "unavailable" for a dead key and a degraded backend. Seat counts are deliberately never cached ("a cached seat count is a wrong seat count"), and a missing `limit` means callers show the bare count rather than invent a denominator.

## Denial reporting

The Pro binary signals license denials only by exit code, with no diagnostic strings by design; the messages live in the wrapper:

| Exit code | Meaning |
|---|---|
| 76 | session limit reached for the plan; a crashed or killed session frees its seat within 4 min (free, trial, solo), 15 min (team) or 30 min (business, scale) |
| 77 | key invalid, expired or missing |
| 78 | license server unreachable or connection problem |
| 79 | local configuration problem: `~/.cloakbrowser` not writable, or an invalid `--license-route` value |

Two paths deliver the code:

1. **Launch failure**: Playwright's error text includes "process did exit: exitCode=N"; the wrapper matches that exact record (so an unrelated `exitCode=` cannot false-match) and raises `CloakBrowserLicenseError`.
2. **Post-handshake denial**: a denial that resolves after the CDP handshake, such as an over-cap seat, kills the browser after Playwright already holds a connection, so no launch failure occurs. The wrapper mints a unique per-launch file path under `~/.cloakbrowser/denials/` and passes it as `CLOAKBROWSER_LICENSE_STATUS_FILE`; the binary writes the code there before exiting. A guard reads the file after browser calls (a single existence check on the fast path), consumes it, and caches the observed code in-process so concurrent guarded calls on the same denied browser all surface it. Orphaned files older than one hour are swept when new paths are minted. Old binaries ignore the variable, so a missing file means "behave as before".

The README adds `--license-route=auto|direct|proxy` (Chromium 154.0.8037.57.2+ on Linux, 155+ elsewhere): by default license and session calls go direct to cloakbrowser.dev and retry once through the proxy, so they never spend proxy bandwidth or touch the scraping session.

## License guard (TypeScript, B5 batch)

`js/src/license.ts` mirrors the Python module and contains the guard that consumes the denial file. `installLicenseGuard(target, denialPath)` wraps every public, non-getter method on a browser, context or page and on its prototype chain:

- **Checked on success too.** Each wrapped call reads the denial file both when it throws and when it succeeds. The binary writes the file the moment the session goes over cap but keeps serving blank responses for about one second before exiting, so a fast flow that never throws must still surface the denial.
- **Deep guarding.** Factory methods (`newPage`, `newContext`, Puppeteer's `createBrowserContext`) also guard the object they return, so a page created after launch is covered.
- **Never wrapped.** `close` already carries teardown logic. The EventEmitter surface is called internally by Playwright/Puppeteer during teardown, where throwing would crash the driver instead of reaching the user.
- **Sync-ness preserved.** Async methods get their promise chained; sync methods (`url()`, `isClosed()`, `on()`) return directly.
- **Two driver error formats.** Playwright embeds `process did exit: exitCode=N`, Puppeteer 25.x reports `Failed to launch the browser process: Code: N`; both are matched with anchored patterns. Where the driver reports neither (the .NET Playwright driver says only "Target ... has been closed"), the launch's denial file is the fallback.
- **Install order.** `launch()` installs the guard right after the browser starts, before the other wraps, so it sits closest to the real call. On the persistent path humanize is patched first, because the guard stores own-property wrappers that would shadow the humanized prototype methods. A persistent context or browser arrives with a page already open, so the guard is also installed on the existing `pages()`.
- A denial path is minted only when a license key resolves, so keyless launches skip the whole mechanism.

### Seat display (CLI tests)

`js/tests/cli.test.ts` pins how `formatSeats` renders the `state` values: "used / limit" when the server sends a cap, the bare count (with a singular form) when it does not, zero kept as a real answer, an explicit message when the server is unreachable, unrecognised denial reasons passed through, and a degraded backend that must not read like a licence problem. `collectDiagnostics` reports a free license, a Preview-to-Stable fallback and the version the next launch will use. See [[projects/cloakbrowser/subsystems/cli|CLI]].

## .NET port (`License.cs`, `Handles.cs`, batch D)

The C# `License` static class is a "Direct port of Python `cloakbrowser/license.py` and JS `js/src/license.ts`" (commit b1a00ed). It keeps the same endpoints, the 24 h validation cache keyed by SHA-256, the 1 h Pro version check with its snake_case/camelCase resolution sidecar, exit codes 76-79 with identical messages, and the `SessionSeats` states. Differences and .NET-specific details:

- **Key source tracking.** It is an explicit enum (`Param`, `Env`, `DefaultFile`, `CustomFile`, `None`). The default file is recognised by comparing full paths, case-insensitively, against `<home>/.cloakbrowser`. Env-injection rules match the table above.
- **Blocking HTTP.** Validation, the Pro version lookup and seat counts are synchronous calls over a shared `HttpClient` (10 s timeout, User-Agent `cloakbrowser-dotnet/<version>`). `EnsureBinaryAsync` runs validation on `Task.Run` so a UI thread is not blocked.
- **Launch-failure mapping.** .NET Playwright's launch error does not carry the child's exit code ("Target ... has been closed"), so `LicenseErrorFrom` tries the `process did exit: exitCode=N` regex and then falls back to the launch's denial file. Exit codes are parsed with `TryParse` so a Windows SEH code cannot overflow and mask the real error.
- **Denial file.** It is read as tolerantly as Python's `int(json.load)` and JS's `Number(JSON.parse)`: a bare number, quoted, or whitespace-padded, "so all three wrappers read an identical denial file the same way". Observed codes are remembered in a `ConcurrentDictionary`, so concurrent guarded calls all see the denial.
- **Guard mechanism.** The guard is a `DispatchProxy` (`LicenseGuardProxy<T>`) over `IBrowser`, `IBrowserContext` and `IPage`, not per-method wrapping. Property accessors are forwarded unguarded, except `Pages` (existing persistent pages get guarded) and `Context` (memoized). Page and frame arguments are unwrapped before the call, because Playwright down-casts them. Humanize wraps first and the guard sits outside. `LicenseGuardCompositionTests` pins this order, and its comment says the Python and JS wrappers once had it backwards ("clicks would teleport").
- **Login.** The .NET CLI saves `license.key` with `File.WriteAllText` and sets no `0600` mode, unlike the Python and JS CLIs. ^[inferred]
- **Tests.** `LicenseTests.cs` (83 cases) is declared a port of `tests/test_license.py` and `js/tests/license.test.ts`. Its seams (`ValidateLicenseOverride`, `SessionSeatsOverride`, `HomeDirOverride`, a swappable `HttpClient`) stand in for the Python/JS monkey-patching. See [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

## Pinned by tests (B6, JS suite)

`js/tests/license.test.ts` plus the launch tests (commit b1a00ed) confirm the design above and add these specifics:

- **Validation cache.** A fresh entry (younger than 24 h) skips the server, and an entry 25 h old triggers a call. An entry for a different key hash is ignored. A non-numeric `validated_at` is treated as no cache rather than trusted. Negative answers (`valid: false`) are never cached. The cache file holds the SHA-256, never the raw key.
- **Release markers.** The preview lookup calls `/api/download/version?channel=preview` and writes only the `_preview_` marker and resolution sidecar, leaving the stable marker alone. An explicit `stable` beats `CLOAKBROWSER_RELEASE_CHANNEL=preview`. An old server that omits the channel fields is reported as a Stable fallback. A legacy camelCase sidecar written by an older JS wrapper is still read; new sidecars use snake_case. The rate-limit window is one hour. Offline, a preview sidecar keeps its real channel, while a marker without a sidecar is labelled a Stable fallback.
- **Env injection.** `buildLaunchEnv` treats an empty or whitespace key as missing and filters out `undefined` values on every path. With an explicit key and a user `env`, it merges only those two, without copying the parent environment. A status-file path is carried even when the key comes from the parent environment. Both backends inject `CLOAKBROWSER_LICENSE_KEY` only when a key resolves, merged with `launchOptions.env`.
- **Exit-code mapping.** Codes 76-79 map to messages that start with "CloakBrowser Pro:". Other codes pass through unchanged, including Windows SEH codes such as 3221225477 (0xC0000005) that must not false-match or overflow. A launch rejected with exit code 77 becomes `CloakBrowserLicenseError`, and any other launch error is re-thrown as the identical object.
- **Denial file.** `readDenialFile` deletes the file, still returns the code on a second read from its in-process cache, and deletes a garbage file while returning null. `mintDenialFile` sweeps a two-hour-old orphan but keeps a fresh, in-flight denial file, consistent with the one-hour sweep described above.
- **Guard reach.** The navigation methods of a page that was already open (`goto`, `reload`, `waitForLoadState`, `waitForURL`, `waitForSelector`) surface the denial too. So does a page or context returned by a factory before the denial landed, and a denial thrown during `createBrowserContext`. A guard installed on an already-humanized page forwards to the human engine, so a humanized click still travels in a curve (`humanizeEngine.test.ts`).
- **Seats.** The count is fetched with a POST to `/api/license/session/count` and the key in the body ("a query string would land in the server's access log"). HTTP 429 counts as a denial. A denial without a JSON body reports `HTTP <status>`, and a 200 response with an unparseable body is `unknown`. `getActiveSessionCount` stays public API returning the bare count.
- **Tier reporting.** `binaryInfo().tier` follows the installed binary, not the cached license: a fresh valid license cache without a Pro binary on disk reports `free`. The suite's `setup.ts` points `CLOAKBROWSER_CACHE_DIR` at a temp directory for this reason, because a developer's real `license.key` would flip version-gated defaults.

## Pinned by Python tests (B8 batch)

`tests/test_license.py`, `tests/test_license_error.py`, `tests/test_login.py` and `tests/test_welcome_banner.py` confirm the behavior above and add these details:

- **Resolution.** An empty-string `license_key` counts as absent and falls through to the environment variable. The environment variable beats the file.
- **Validation cache.** A fresh cache skips the server, a stale one calls it, and an unreachable server falls back to a stale entry. The cache stores the key's SHA-256, never the raw key. A cache written for a different key is ignored. An expired license is rejected from cache, including with a timezone-naive `expires` date. A corrupted `validated_at` field must not crash.
- **Never a silent downgrade.** With a supplied key, a verification failure is re-raised verbatim (a tampering signal, no "retry" wording). A transient network or server failure becomes an actionable "Pro binary unavailable, retry" error that carries the cause. A rejected or unvalidatable key aborts. A macOS Pro 404 is a hard error: the v0.4.2 stopgap that fell back to the free binary on macOS was reverted in v0.4.3.
- **Tier reporting.** `binary_info()` reports `pro` only when a Pro binary is actually on disk, not because a valid license is cached.
- **Env injection.** `build_launch_env` returns no env when there is nothing to inject, and treats an empty key as missing. When a status-file path exists, it is carried even on the inherit path, and on the default-file path the key still stays out of the environment.
- **Exit codes.** 76 to 79 map to messages that start with "CloakBrowser Pro:". Ordinary crash codes (1, 139) and a large Windows SEH code (`3221225477`, which overflows a 32-bit parse) pass through unchanged. `CloakBrowserLicenseError` subclasses `RuntimeError`.
- **Denial files.** Reading a denial file consumes it, but a second guarded call still sees the code. Garbage or empty files are ignored. Minting a new path sweeps stale files and keeps fresh ones, and an unwritable denials directory yields no path instead of an error.
- **Guards.** A guard survives humanize copying page methods into a class attribute. Pages returned by guarded factories are guarded too. On an already-open persistent page, `goto`, `reload`, `wait_for_load_state`, `wait_for_url` and `wait_for_selector` each surface the denial; this fixed a bare `TargetClosedError` from `pages[0].goto()`. Without a denial file, the original exception object is re-raised untouched.
- **Login.** `cloakbrowser login` saves only keys the server accepts, as `license.key` with mode `600`. An invalid key, or an unreachable server, exits 1 and never overwrites an existing good key. With no key and no TTY it exits 2. `logout` is idempotent.
- **Welcome banner.** A free-plan key gets the free banner and any other plan, including unknown, gets the Pro one. See [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]] for the banner cadence.

## Open Questions

- The free tier's session cap and plan names come from README marketing text; the server-side plan model is not in the repository. ^[ambiguous]
- Resolved for JS by the B5 batch: the guard is `installLicenseGuard` in `license.ts` (see above). The Python equivalent `_install_license_guard` was not compared in detail.

## Related

- [[projects/cloakbrowser-manager/subsystems/settings-and-license|Settings and license (Manager)]] — how CloakBrowser-Manager stores one app-wide key and maps license denials to HTTP 402/403.

## Class stubs (ast-extract)

- [[projects/cloakbrowser/subsystems/cloak-browser-license-error|CloakBrowserLicenseError]]

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/cloakbrowser/license.py (re-read in the P2 pass)
- ~/code/CloakBrowser/cloakbrowser/download.py (commit b1a00ed, P2 pass)
- ~/code/CloakBrowser/dotnet/README.md
- ~/code/CloakBrowser/README.md
- ~/code/CloakBrowser/js/src/license.ts, js/src/playwright.ts, js/src/puppeteer.ts, js/tests/cli.test.ts (commit b1a00ed, B5 batch)
- ~/code/CloakBrowser/js/tests/license.test.ts, js/tests/launch.test.ts, js/tests/puppeteer.test.ts, js/tests/setup.ts, js/tests/humanizeEngine.test.ts (commit b1a00ed, B6 batch)
- ~/code/CloakBrowser/tests/test_license.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/tests/test_license_error.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/tests/test_login.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/tests/test_welcome_banner.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/CHANGELOG.md (0.4.5 entry, stale banner interval)
- ~/code/CloakBrowser/dotnet/src/CloakBrowser/License.cs, dotnet/src/CloakBrowser/Handles.cs, dotnet/tests/CloakBrowser.Tests/LicenseTests.cs, dotnet/tests/CloakBrowser.Tests/Wrappers/LicenseGuardCompositionTests.cs (commit b1a00ed, batch D)
