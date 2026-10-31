---
title: >-
    Release channels and versioning
category: architecture
tags: [chromium, release-management, versioning, licensing, stealth]
relationships:
  - target: "[[projects/cloakbrowser/architecture/wrapper-binary-split]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/BINARY-LICENSE.md
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/download.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/_version.py
  - repo:github.com/CloakHQ/CloakBrowser/.github/workflows/publish.yml
  - repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/download.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/update.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/license.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/proxy.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/puppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_free_version_pin.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_config.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_cli.py
summary: >-
    Two version lines (wrapper semver, binary Chromium.build), Stable/Preview channels, free vs Pro builds per platform, exact pins, and a dated history of patch counts.
provenance:
  extracted: 0.85
  inferred: 0.1
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:25+01:00
updated: 2026-10-31T06:37:29+01:00
---

# Release channels and versioning

CloakBrowser ships on two independent version lines: the **wrapper** (semver, e.g. `0.6.4`, identical across PyPI, npm, NuGet and the Docker tag) and the **binary** (a five-part Chromium version such as `155.0.8059.39.1`, where the last component is CloakHQ's build number). Which binary a launch runs depends on license tier, platform, release channel, an optional pin, and cached auto-updates. This page distills the CHANGELOG; mechanics of fetching live in [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]] and tiers in [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].

## Current state (as of 26/10/2026, commit b1a00ed)

| Platform | Free keyless binary (hardcoded in wrapper) | Latest binary (free GitHub key or Pro) |
|---|---|---|
| linux-x64 | `146.0.7680.177.5` (58 patches) | `155.0.8059.39.1` (87 patches) |
| linux-arm64 | `146.0.7680.177.3` | `155.0.8059.39.1` |
| windows-x64 | `146.0.7680.177.5` (58 patches) | `155.0.8059.39.1` |
| darwin-arm64 / darwin-x64 | `145.0.7632.109.2` (26 patches) | `155.0.8059.39.1` |

- Wrapper `0.6.4` (2026-10-23). The wrapper's `CHROMIUM_VERSION` constant (`146.0.7680.177.5`) is the newest free build; `PLATFORM_CHROMIUM_VERSIONS` holds per-platform free defaults because platforms move on different schedules.
- Three ways to get a binary: keyless (old free build from GitHub Releases), free GitHub-login key (latest build, one concurrent session), paid Pro key (latest build, more sessions, pinning).
- License text: v146 and earlier is free for personal and commercial use without redistribution; v148+ requires a subscription to download (README). BINARY-LICENSE v1.3 (July 2026) phrases it as "the latest major" needing a paid tier, with 30 days' notice for changes. ^[ambiguous]

## Selection rules

1. `CLOAKBROWSER_BINARY_PATH` overrides everything (no download, unknown version).
2. A valid key routes to the Pro path. A free-plan key always gets latest: the wrapper drops any pin because the server force-serves latest to free keys, so a pinned signed manifest would not match the bytes. Pinned by `tests/test_free_version_pin.py` (B7): a `free` plan reaches the Pro path with the pin set to `None`, a `business` plan keeps it.
3. **Exact pin** (`browser_version=` / `CLOAKBROWSER_VERSION`): must match `^[0-9]+(\.[0-9]+){3,4}$` because it is interpolated into URLs and cache paths. A pin is never sticky: it does not write the "latest" marker. README calls pinning "paid Pro key only"; the keyless free path in code also honors a pin (downloads that version from GitHub). ^[ambiguous]
4. **Channel** (`release_channel=` / `CLOAKBROWSER_RELEASE_CHANNEL`, default `stable`). Preview means "newest build for this platform": a newer Preview if one exists, otherwise Stable, including when Stable has moved ahead. It is a standing setting, not a pin; an explicit pin overrides it. The wrapper warns once on stderr when Preview falls back to Stable, and `info` shows `Preview → Stable fallback`.
5. Unpinned Pro launches track the server's latest for the channel (looked up at most once per hour), preferring it over the cached build when newer; `CLOAKBROWSER_AUTO_UPDATE=false` freezes a cached Pro build.
6. Free keyless launches use a platform-scoped `latest_version_<tag>` marker written by a background updater that polls GitHub Releases at most hourly and only offers releases that carry an asset for the current platform.
7. Rollback by downgrading the wrapper also works, since each wrapper release hardcodes its free binary version.

Pro and free binaries cache side by side (`chromium-<v>-pro` vs `chromium-<v>`) with separate markers per channel (`latest_pro_version[_preview]_<platform>`). A valid Pro license never falls back to the free binary. ^[inferred]

## JS wrapper confirmation (B4 batch)

`js/src/config.ts` and `js/src/download.ts` carry the same constants and rules (commit b1a00ed):

- **Version constants**: `CHROMIUM_VERSION` 146.0.7680.177.5 and the same per-platform free map.
- **Channel and pin normalization**: anything other than `preview` normalizes to `stable`. Pins must match the same 4-to-5-part numeric pattern, otherwise the wrapper throws.
- **Pinning on the keyless path**: the JS keyless free path also honors a pin. `ensureBinary` downloads the pinned version from the free origins when no key is present. So both wrappers' code allows keyless pinning, while both READMEs say "paid Pro key only". The JS README is precise about only one case: a *free GitHub key* "ignores the pin and always gets the latest build". ^[ambiguous]
- **Wrapper update check**: the JS wrapper checks the npm registry, not PyPI, for its own updates, once per process. It is skipped when `CLOAKBROWSER_AUTO_UPDATE=false` or a custom download URL is set.
- **Shared markers**: the markers, rate-limit files and `-pro` directory suffix are byte-compatible with the Python wrapper (see [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]]).

## Pinned by tests (B6, JS suite)

The JS tests (commit b1a00ed) fix how channels interact with caches and capability gates:

- **Channels pick the binary that the gates judge.** With a stable Pro build `145.0.1000.1` and a preview build `148.0.7778.215.4` both cached, the headless-no-viewport and inline-proxy-auth gates are off for `stable` and on for `preview`, and the maximized-window gate is on for `preview` (`update.test.ts`, `proxy.test.ts`). So a channel switch can change launch flags, not only the binary.
- **Pins stay isolated.** A pinned preview launch (`ensureBinary(key, "151.0.1000.1", "preview")`) only validates the license and writes neither latest marker.
- **Markers are per channel.** An unpinned preview launch, or a manual `checkForProUpdate(key, "preview")`, advances only the `_preview_` marker, and the stable marker keeps its value. One cached build can satisfy both channels because the cache directory is keyed by version.
- **Fallback warning.** The "no preview build is available" warning fires when the server answers `resolved_channel: stable, fallback: true`, and stays silent for a genuine preview build.
- **Channel plumbing.** Both backends pass `releaseChannel` through to `ensureBinary(undefined, undefined, "preview")`. `binaryInfo(…, "preview")` reports a download URL ending in `/api/download/latest?channel=preview`, while stable has no channel parameter.
- **Version order**: a five-part build ranks above the same four-part Chromium version (`145.0.7632.109.2` newer than `145.0.7632.109`), so build bumps count as updates.

## Python test confirmation (B7 batch)

- **Free-key pin**: confirmed in Python by `tests/test_free_version_pin.py` (see rule 2). The keyless-pin question stays open: no Python test in this batch exercises a pin without a key. ^[ambiguous]
- **Pin validation**: `tests/test_config.py` pins that an explicit `browser_version` beats `CLOAKBROWSER_VERSION`. Path traversal and Unicode digits are rejected, with the comment "Parity with JS (ASCII-only \d)… the same input behaves identically across Python / JS / .NET".
- **Channel fallback display**: `tests/test_cli.py` pins `info` printing `Channel:   Preview → Stable fallback` when a Preview request resolves to a Stable build.
- **Version labels in test comments**: `tests/test_config.py` calls `148.0.7778.215.3` the "current live Pro version". That contradicts the Pro Stable `155.0.8059.39.1` listed above and is probably stale. ^[ambiguous] See [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]].

## Wrapper release process

Tags `v*` trigger `publish.yml`: run the Python, JS and .NET test suites, check that the tag equals `_version.py`, `js/package.json` and the `.csproj` version, then publish to PyPI, npm (with provenance) and NuGet via OIDC trusted publishing (no stored tokens), and build a multi-arch (amd64/arm64) Docker image tagged with the version and `latest`, Cosign-signed and attested. Binary releases are separate tags `chromium-v<version>`. See [[projects/cloakbrowser/architecture/supply-chain-verification|Supply-chain verification]].

## Dated history (distilled from CHANGELOG)

| Date | Wrapper | Binary milestone |
|---|---|---|
| 2026-02-22 | 0.1.0 | Chromium 142.0.7444.175, 16 patches, Linux x64 only |
| 2026-02-24…27 | 0.2.0 | JS/TS wrapper; macOS builds; cloakbrowser.dev self-hosted mirror; GPG-signed release workflow |
| 2026-03-02 | 0.3.0 | Chromium 145, 25 patches; per-platform versioning; SHA-256 verification; GitHub fallback; patchright made default (breaking) |
| 2026-03-03 | 0.3.3 | Windows x64 added; binary auto-spoofs with zero flags |
| 2026-03-04 | 0.3.4 | Binary v14: random seed auto-injected, full auto-spoof from seed; 26 patches |
| 2026-03-05 | 0.3.9 | Stock Playwright becomes default backend again |
| 2026-03-08 | 0.3.11 | `humanize=True`; 4 CDP-input-stealth patches |
| 2026-03-13 | 0.3.15 | 145.0.7632.159.7, 33 patches |
| 2026-03-14 | 0.3.16 | Linux arm64 builds |
| 2026-03-30 | 0.3.19 | 42 patches; `--fingerprint-noise` flag |
| 2026-04-06 | 0.3.20 | 48 patches; WebRTC IP spoofing, proxy signal removal; cloakserve rewritten as multiplexer |
| 2026-04-09 | 0.3.22 | Chromium 146.0.7680.177.1, 49 patches |
| 2026-04-16 | 0.3.25 | 146.0.7680.177.3, 57 patches (Linux) |
| 2026-04-28 | 0.3.26 | Windows on 146.0.7680.177.4, 57 patches |
| 2026-05-21 | 0.3.30 | 146.0.7680.177.5, 58 patches (Linux x64, Windows); inline HTTP proxy auth in binary |
| 2026-06-22 | 0.4.0 | First Pro build 148.0.7778.215.2 (59 patches); Ed25519-signed manifests; patchright removed |
| 2026-06-24 | 0.4.3 | macOS Pro; .NET client |
| 2026-06-27 | 0.4.4 | Exact version pinning; per-platform Pro version resolution |
| 2026-07-05 | 0.4.8 | 148.0.7778.215.5: "biggest fingerprint update"; `--fingerprint=off` |
| 2026-07-18 | 0.4.12 | Chromium 150.0.7871.114.3, 71 patches; distinct license exit codes |
| 2026-07-22 | 0.5.0 | Free tier via GitHub login (`cloakbrowser login`) |
| 2026-07-25 | 0.5.2 | Preview channel; 150.0.7871.114.4, 73 patches (Linux) |
| 2026-08-18 | 0.5.8 | 150.0.7871.114.6, 71 patches (Linux/Windows Stable) |
| 2026-10-08 | 0.6.0 | Pro Stable 154.0.8037.57.1 on all platforms; Linux Preview 154.0.8037.57.2; per-profile seeds |
| 2026-10-23 | 0.6.4 | Banner names v155; README lists Pro Stable 155.0.8059.39.1, 87 patches |

Patch counts are not monotonic (73 on 150.4 Linux Preview vs 71 on 150.6 Stable), and the CHANGELOG has no entry announcing the 155 build or the 87-patch count; those figures come from the README. ^[ambiguous]

The pattern is a fast rebase cadence (142 → 155 in eight months) with patches ported forward on every Chromium major, while the free keyless build was frozen at 146 once Pro launched. ^[inferred]

## Open Questions

- Which binary versions were Preview-only vs Stable on each platform before 0.6.0 is only partly recorded.
- The free GitHub key is described as "latest build" in README but the wrapper banner calls the free tier "v155"; how the free key relates to Preview is unstated.

## Related

- [[projects/cloakbrowser-manager/subsystems/settings-and-license|Settings and license (Manager)]] — CloakBrowser-Manager exposes the Stable/Preview channel as an app-wide setting.

## Sources

- `repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/README.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/BINARY-LICENSE.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/download.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/_version.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/.github/workflows/publish.yml` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts`, `js/src/download.ts`, `js/README.md` (commit `b1a00ed`, B4 batch)
- `repo:github.com/CloakHQ/CloakBrowser/js/tests/update.test.ts`, `js/tests/license.test.ts`, `js/tests/launch.test.ts`, `js/tests/proxy.test.ts`, `js/tests/puppeteer.test.ts` (commit `b1a00ed`, B6 batch)
- `repo:github.com/CloakHQ/CloakBrowser/tests/test_free_version_pin.py`, `tests/test_config.py`, `tests/test_cli.py` (commit `b1a00ed`, B7 batch)
