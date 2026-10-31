---
title: >-
    CLI
category: subsystems
tags: [cli, diagnostics, licensing, chromium, tooling]
aliases: [cloakbrowser CLI, __main__.py, cli.ts, cloakbrowser info]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
sources:
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__main__.py
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md
  - repo:github.com/CloakHQ/CloakBrowser/.github/ISSUE_TEMPLATE/bug_report.md
  - repo:github.com/CloakHQ/CloakBrowser/js/src/cli.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/download.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/README.md
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_cli.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser.Cli/Program.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Diagnostics.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/DiagnosticsTests.cs
summary: >-
    The cloakbrowser CLI (python -m cloakbrowser, npx cloakbrowser, .NET) installs, updates and clears binaries, manages license keys, and runs info diagnostics that mirror launch.
provenance:
  extracted: 0.88
  inferred: 0.1
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:45:43+01:00
updated: 2026-10-31T06:58:25+01:00
---

# CLI

The CLI (`cloakbrowser/__main__.py`, mirrored by `js/src/cli.ts` and the .NET `CloakBrowser.Cli` project, whose assembly is named `cloakbrowser-cli`) is a management front end over the same functions the launcher uses. It exists so users can pre-download the binary during image builds, manage license keys, and above all produce a diagnostic that matches what a launch would actually do. First added in 0.3.14 (#43); `info` became a full diagnostic in 0.4.5. See [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]].

## Commands (as of 26/10/2026, commit b1a00ed)

| Command | Effect |
|---|---|
| `install` | `ensure_binary()` with visible progress; prints the binary path. Recommended in Dockerfiles. |
| `info` / `doctor` | Environment and binary diagnostics; flags `--quick` (alias `--no-launch`), `--json`, `--proxy <url>` |
| `update` | License-aware: a valid Pro key runs `check_for_pro_update`, everyone else `check_for_update` (free). Ignores `CLOAKBROWSER_AUTO_UPDATE`. |
| `clear-cache` | Deletes the whole cache dir (binaries, markers, GeoIP DB, saved key); confirmed for JS `clearCache()`, see below |
| `login [key]` | Validate and save a key to `~/.cloakbrowser/license.key` (mode 0600); bare `login` prompts to paste a key or press Enter for the free GitHub flow |
| `logout` | Remove the saved key; launches revert to the free keyless binary |

All commands honour `CLOAKBROWSER_RELEASE_CHANNEL=preview` (README).

## `info`: mirror the launch, don't guess

The design goal is that `info` never disagrees with the binary a launch would run (CHANGELOG 0.4.9, after a customer saw `info` show the latest build while launch ran a stale cache). It therefore:

- **resolves the license the same way** `ensure_binary` does, including disabling Pro when `CLOAKBROWSER_DOWNLOAD_URL` is set, and reports the real tier (`free`, `solo`, `team`, `business`, `invalid`, `unknown`);
- **describes the effective binary** in the same order: override > pin > Pro (channel marker vs server latest, honouring `AUTO_UPDATE=false`) > free marker. Unlike `binary_info()`, a cached Pro binary is reported only if the current license entitles Pro. It shows requested vs resolved channel and whether Preview fell back to Stable;
- **runs a launch test**: `<binary> --version` on POSIX plus an `ldd` probe for missing shared libraries on Linux. On Windows Chromium ignores `--version` and starts a real browser that would hold a license seat, so it reads the version manifest file instead (0.5.3);
- **counts fonts** on Linux: Windows set (8), Office set (10, informational) and macOS set, each as N/total, for the persona checks in [[projects/cloakbrowser/detection-vectors/font-fingerprinting|Font fingerprinting]];
- **shows live seat usage** for Pro licenses (`used/limit` with a reason when unavailable: invalid key, inactive license, rate limited). The count is never cached;
- **GeoIP**: reports DB presence only; with `--proxy`, runs the full `geoip=True` resolution and prints exit IP, timezone and locale (see [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]]);
- lists optional modules (`playwright`, `geoip2`, `aiohttp`, `websockets`).

`--quick` keeps `info` network-free except license validation: no launch test, no server latest lookup, no seat count. Output marks (check, cross, arrow) degrade to ASCII per character on consoles that cannot encode them, such as `cmd.exe` code pages (0.5.3).

### Pinned by tests (`tests/test_cli.py`, B7)

The Python suite confirms the rules above and adds a few specifics (commit b1a00ed):

- **No downloads, ever**: `info` never calls the free or Pro downloader, with or without `--quick`. The full run executes the resolved binary as-is, and a failing `--version` reports its stderr (e.g. a missing shared library).
- **Proxy diagnostics are non-fatal**: without `--proxy`, `info` never resolves and prints a "pass --proxy" hint. With `--proxy`, a resolution error is printed ("could not resolve …") rather than raised, unlike `geoip=True` at launch, which aborts. The JSON form carries `geoip.resolved` = exit IP, timezone, locale.
- **Tier mapping**: keyless gives a free binary and free license. A server-rejected key gives license `invalid` but still a free binary. A valid key gives a Pro binary, with `latest_version` filled only outside `--quick`.
- **Seats**: looked up only for a valid paid key outside `--quick`; free and invalid keys never ask. The rendering matches the JS `formatSeats` rules on [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]]. The `unknown` state reads "server cannot report seats right now" so it does not look like a licence problem.
- **Free upgrade hint**: text output says `Free (no key)`, invites `cloakbrowser login` and mentions plans for more than one concurrent session. JSON omits the hint and exposes the tier as a field instead.
- **Fonts**: the macOS tell set has 20 fonts. A partial count prints a conditional nudge ("if spoofing macOS on this host"), and JSON reports `fonts.macos` as `[present, total]`.
- **Windows**: the binary is never executed, the version comes from the `.manifest` file name, and `launch.tested` is false.
- **Console glyphs**: the fallback is decided per glyph. A cp1252 console keeps the em dash but gets `OK` for the check mark.

## Login flow

The free tier (0.5.0) uses GitHub sign-in: `login` opens `cloakbrowser.dev/api/license/free/github/start`, the server emails a free key to the GitHub email, and the user pastes it back. Non-interactive stdin without a key exits with usage. A key is saved only after the server validates it; an unreachable server saves nothing. See [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].

## Support workflow

The bug-report issue template asks for wrapper version, language, environment (Docker, base image, arch), launch options, and whether a different IP/proxy or running outside Docker changes the result. `info` printing the wrapper version next to binary diagnostics (0.5.2) lets one command cover most of this. ^[inferred]

## JS port (`cli.ts`, `npx cloakbrowser`)

`cli.ts` ships as the `cloakbrowser` bin of the npm package and offers the same commands and `info` flags. These are the JS-specific details at commit b1a00ed:

- **`clear-cache` wipes the whole cache dir**: `clearCache()` deletes `getCacheDir()` recursively, which confirms the row above for JS. That directory also holds `license.key`, the GeoIP database and the version markers, so in practice `clear-cache` also logs the user out. The command describes itself as "Remove cached binaries", which hides that. ^[inferred]
- **Optional modules listed by `info`**: the npm peers `playwright-core`, `puppeteer-core` and `mmdb-lib`, plus Node's version in place of Python's. Environment details are on [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].
- **Shared seat wording**: `formatSeats` is "kept identical in the Python and .NET wrappers". Server codes become readable reasons (`invalid_key` → "invalid key", `license_inactive`, `rate_limited`). An unknown code is printed verbatim rather than dropped. A missing limit prints "N seats in use", never "N/unknown".
- **Opening the login page**: `login` opens it with `open` on macOS, `start` on Windows (through the shell) or `xdg-open` elsewhere. Failure is silent, and the user pastes the emailed key afterwards. `--proxy` accepts both the `--proxy=<url>` and `--proxy <url>` forms.
- **Launch test on Windows**: the binary is not launched. When the binary directory contains a `.manifest` file, its name gives the version.

## .NET port (`dotnet/src/CloakBrowser.Cli/Program.cs`, batch D)

A "Direct port of Python `cloakbrowser/__main__.py`" written as top-level statements (commit b1a00ed). The rendering logic lives in `Diagnostics.cs` in the library: top-level local functions cannot be reached from the test assembly, so `FormatSeats` sits in the library to stay testable.

- **Commands**: `login [key]`, `logout`, `install`, `info`/`doctor` (`--quick`/`--no-launch`, `--json`, `--proxy <url>` or `--proxy=<url>`), `update`, `clear-cache`. All of them are implemented, although `dotnet/README.md`'s CLI section lists only four. Exit codes: 2 for usage, 1 for errors, 130 for cancellation.
- **`info` mirrors the launch** the same way. It resolves the license as `EnsureBinary` does (a custom download URL disables Pro), reports override, pin, Pro or free, separates "next launch" from the server latest, skips the launch test on Windows (using the version manifest name instead), probes the Windows/Office/macOS font sets, and shows seats via `FormatSeats`, which is "byte-identical" to the Python and JS renderers. The modules section lists `Microsoft.Playwright` and `MaxMind.GeoIP2`, which are hard NuGet references, so "missing" means a broken deployment.
- **`update`** is license-aware like Python's: a valid key runs `CheckForProUpdateAsync`, everyone else the free check.
- **`login`** validates before saving, never saves when the server is unreachable, and with redirected stdin and no key exits 2. The free flow opens `cloakbrowser.dev/api/license/free/github/start` through the shell. Unlike Python and JS, the key file is written without setting mode `0600`. ^[inferred]
- **`clear-cache`** deletes the whole cache directory, as in JS, which also removes the saved key.
- **Console glyphs**: `info` prints ✓, ✗ and → directly, without the per-glyph ASCII fallback Python added for legacy Windows consoles (0.5.3). ^[inferred]

## Sources

- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__main__.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser.Cli/Program.cs`, `dotnet/src/CloakBrowser.Cli/CloakBrowser.Cli.csproj`, `dotnet/src/CloakBrowser/Diagnostics.cs`, `dotnet/tests/CloakBrowser.Tests/DiagnosticsTests.cs` (commit `b1a00ed`, batch D)
- `repo:github.com/CloakHQ/CloakBrowser/README.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/.github/ISSUE_TEMPLATE/bug_report.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/js/src/cli.ts`, `js/src/download.ts`, `js/README.md` (commit `b1a00ed`, B4 batch)
- `repo:github.com/CloakHQ/CloakBrowser/tests/test_cli.py` (commit `b1a00ed`, B7 batch)
