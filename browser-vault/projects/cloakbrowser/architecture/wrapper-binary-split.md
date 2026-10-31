---
title: >-
    Wrapper/binary split
category: architecture
tags: [chromium, playwright, stealth, architecture, licensing]
relationships:
  - target: "[[projects/cloakbrowser/techniques/source-level-chromium-patches]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/synthesis/manager-cloakbrowser-responsibility-split]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/BINARY-LICENSE.md
  - ~/code/CloakBrowser/cloakbrowser/__init__.py
  - ~/code/CloakBrowser/cloakbrowser/browser.py
  - ~/code/CloakBrowser/cloakbrowser/config.py
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/LICENSE
  - ~/code/CloakBrowser/Dockerfile
summary: >-
    CloakBrowser splits into a proprietary C++-patched Chromium binary that holds all fingerprint logic and MIT wrappers that only fetch, verify, flag and launch it.
provenance:
  extracted: 0.78
  inferred: 0.20
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:25+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Wrapper/binary split

CloakBrowser is two artifacts with different licenses, release cadences and responsibilities. The **binary** is a custom Chromium build (built on ungoogled-chromium) whose fingerprint behavior is compiled in at the C++ level; it is proprietary, closed and distributed only through GitHub Releases and cloakbrowser.dev. The **wrapper** (Python, JavaScript/TypeScript, .NET; MIT) is a thin layer that downloads and authenticates the right binary, builds its command-line flags, and hands control to stock Playwright or Puppeteer. The public repository contains only the wrapper side. See [[projects/cloakbrowser/cloakbrowser|CloakBrowser]] for the overview.

## Division of labor

| Concern | Binary | Wrapper |
|---|---|---|
| Canvas/WebGL/audio/font/GPU/screen/WebRTC spoofing | Yes, compiled in (87 patches in the 155 Pro build, as of 26/10/2026, commit b1a00ed) | No |
| Identity generation | Derives a coherent device from `--fingerprint=<seed>`; auto-generates a random seed when none is passed | Picks a random seed (10000–99999) or a per-profile persisted seed |
| Platform persona | Reports its compile-time OS by default | Passes `--fingerprint-platform=windows` on Linux/Windows, `macos` on macOS |
| `navigator.webdriver`, automation signals | Patched at source | Strips Playwright's `--enable-automation` and `--enable-unsafe-swiftshader` defaults |
| Timezone/locale | Implements `--fingerprint-timezone`, `--fingerprint-locale` natively | Resolves values (explicit or GeoIP) and sets the flags; never uses CDP emulation |
| License enforcement | Pro binary validates the key itself and exits with codes 76–79 on denial; writes a denial file | Resolves the key, injects it into the browser env, maps exit codes/denial file to `CloakBrowserLicenseError` |
| Human-like input | Has CDP-input-stealth patches only | `humanize=True` engine patches Playwright/Puppeteer objects |
| Download/verification | n/a | Fetches, Ed25519-verifies, extracts, caches, auto-updates |

Details of the wrapper launch pipeline live in [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]; download and signature checking in [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]].

## Why split this way

- The README's central claim is that config-level and JS-injection stealth tools (`playwright-stealth`, `undetected-chromedriver`, `puppeteer-extra`) break on every Chrome update and are themselves detectable, while source-level patches make the browser "a normal browser". See [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]].
- Keeping fingerprint logic out of the wrapper lets the same wrapper run any binary version: each wrapper release hardcodes a default free version per platform (`PLATFORM_CHROMIUM_VERSIONS`), but Pro builds are resolved from the server at launch. ^[inferred]
- The split also carries the business model: wrappers are free forever, while the latest major binary needs a license key (free GitHub key with one concurrent session, or paid Pro). See [[projects/cloakbrowser/architecture/release-channels-and-versioning|Release channels and versioning]] and [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].
- Because the binary is "stealthy with zero flags" (binary v14, CHANGELOG 0.3.4), the wrapper's default flag set shrank over time: hardcoded GPU vendor/renderer flags and `--disable-blink-features=AutomationControlled` were removed in 0.3.21 because the binary now owns them. The wrapper's job drifted from "configure stealth" toward "choose and verify the right binary, keep window geometry coherent, and orchestrate network identity". ^[inferred]

## Consequences and trade-offs

- **Version-gated wrapper behavior.** Some wrapper features depend on binary capabilities, so the wrapper carries per-version thresholds: headless no-viewport and auto `--start-maximized` start at `148.0.7778.215.4`; inline HTTP proxy credentials have a per-platform floor (`146.0.7680.177.5` on linux-x64/windows-x64, `148.0.7778.215.3` elsewhere). A local override binary (`CLOAKBROWSER_BINARY_PATH`) with no declared version stays on the safe fallback path.
- **Opaque core.** Users cannot audit what the 87 patches do; the wrapper repo documents flags and outcomes only. The binary license forbids reverse engineering and redistribution. ^[inferred]
- **Framework neutrality.** Since stealth lives in the binary, any tool that can launch a Chromium executable or connect over CDP gets the fingerprint layer; only `humanize` needs the wrapper. See [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]].
- **Three mirrored wrappers.** Python, JS and .NET implement the same gates and file formats (e.g. the `.cloakbrowser-seed` file is byte-identical across wrappers). See [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].
- Stock Playwright is the only backend since 0.4.0; the `patchright` fork was dropped because the binary already suppresses automation signals and patchright broke proxy auth and `add_init_script`.

## Licensing of the two halves

- The repository's `LICENSE` is the standard MIT text, "Copyright (c) 2026 CloakHQ". It covers the wrapper code in this repository; the binary is governed separately by `BINARY-LICENSE.md` (no redistribution, no reverse engineering, OEM/SaaS terms for resale; see [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]]).
- The official Docker image is where the two halves ship together. The root `Dockerfile` copies both license files into the image, installs the wrappers from source and **pre-downloads the binary at build time** through the wrapper's own `ensure_binary()`, so the image carries a verified binary rather than fetching one on first use (details in [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]]). Derived images such as the Lambda example rely on `FROM cloakhq/cloakbrowser` counting as an official distribution channel rather than redistribution. ^[inferred]

## Open Questions

- The patch list itself is not in this repository; which detection vectors each of the 87 patches targets is known only from README/CHANGELOG prose.
- How the binary validates licenses at runtime (protocol, cadence) is undocumented beyond `--license-route` and exit codes.

## Related

- [[projects/cloakbrowser-manager/synthesis/manager-cloakbrowser-responsibility-split|Manager vs CloakBrowser responsibility split]] — the same policy/mechanism split one layer higher, in CloakBrowser-Manager.

## Sources

- `~/code/CloakBrowser/README.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/BINARY-LICENSE.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/__init__.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/LICENSE` (commit `b1a00ed`, P2 pass)
- `~/code/CloakBrowser/Dockerfile` (commit `b1a00ed`, P2 pass)
