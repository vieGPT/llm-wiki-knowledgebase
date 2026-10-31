---
title: >-
    CloakBrowser
category: project
tags: [browser-automation, chromium, fingerprinting, playwright, stealth]
source_repo: github.com/CloakHQ/CloakBrowser
source_cwd_hint: ~/code/CloakBrowser
sources: [repo:github.com/CloakHQ/CloakBrowser]
summary: >-
    Stealth Chromium: a custom C++-patched binary plus thin Python/JS/.NET wrappers that launch it through Playwright or Puppeteer with stealth args, GeoIP and humanized input.
provenance:
  extracted: 0.85
  inferred: 0.15
  ambiguous: 0.0
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:37:09+01:00
updated: 2026-10-31T06:37:29+01:00
---

# CloakBrowser

CloakBrowser is a Chromium build with fingerprints modified at the C++ source level (87 source-level patches in the v0.6.4 / Chromium 155.0.8059.39.1 Pro build), shipped with thin wrappers for Python, JavaScript/TypeScript and .NET. The wrapper downloads and verifies the binary, builds the stealth launch arguments, and hands control to the standard Playwright or Puppeteer API. The repository holds the wrappers, not the Chromium patches themselves. ^[inferred]

## How it fits together

- **Binary vs wrapper split**: fingerprints live in the compiled binary; the wrapper only selects the binary, sets flags (`--fingerprint`, `--fingerprint-platform`, timezone/locale, WebRTC IP) and optionally patches Playwright/Puppeteer for humanized input. See [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]].
- **Identity model**: one fingerprint seed derives a coherent device package (GPU, screen, hardware, noise patterns). See [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].
- **Behavior layer**: `humanize=True` replaces input calls with human-like mouse, keyboard and scroll behavior. See [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]].
- **Network consistency**: GeoIP matches timezone/locale and WebRTC IP to the proxy exit IP. See [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]].

## Structural hubs (ast-extract, 31/10/2026)

`ast-extract` processed 127 files (Python 65, TypeScript 57, shell 3, JavaScript 2) with 1,415 edges, all `EXTRACTED`. The tool's 10 `god_nodes` are dominated by test files (7 of 10: `tests/test_humanize_engine.py`, `tests/test_humanize_unit.py`, `tests/test_cli.py`, `tests/test_geoip.py`, `tests/test_update.py`, `tests/test_license_error.py`, `tests/test_launch_context.py`); the non-test god nodes are `js/src/download.ts`, `cloakbrowser/config.py` and the `cloakbrowser` package import.

Highest-degree non-test files, which anchor the subsystem pages:

| Hub | Degree | Subsystem page |
|---|---|---|
| `js/src/download.ts`, `cloakbrowser/download.py` | 42, 22 | [[projects/cloakbrowser/subsystems/binary-download-and-verification\|Binary download and verification]] |
| `cloakbrowser/config.py`, `js/src/config.ts`, `js/src/args.ts` | 28, 25 | [[projects/cloakbrowser/subsystems/launcher-and-stealth-args\|Launcher and stealth args]] |
| `cloakbrowser/license.py`, `js/src/license.ts` | 27, 25 | [[projects/cloakbrowser/subsystems/license-and-tiers\|License and tiers]] |
| `js/src/cli.ts`, `cloakbrowser/__main__.py` | 27, 15 | [[projects/cloakbrowser/subsystems/cli\|CLI]] |
| `js/src/human/patch.ts`, `cloakbrowser/human/engine.py`, `cloakbrowser/human/world.py` | 23, 18, 16 | [[projects/cloakbrowser/subsystems/humanize-engine\|Humanize engine]] |
| `js/src/geoip.ts`, `cloakbrowser/geoip.py` | 21, 14 | [[projects/cloakbrowser/subsystems/geoip-resolver\|GeoIP resolver]] |

The Python and TypeScript wrappers mirror each other file for file (for example `ActionabilityError` exists in both `cloakbrowser/human/errors.py` and `js/src/human/errors.ts`). ^[inferred]

**Gap**: the .NET client (76 `.cs` files under `dotnet/`) is not covered by `ast-extract` or `batch-plan`; only `dotnet/README.md` is ingested.

## Dependencies

Top external imports in non-test code (ast-extract):

1. `playwright-core` (6 import edges)
2. `puppeteer-core` (4)
3. `playwright` (3)
4. `httpx` (2)
5. `tar` (1)

## Key pages

- Architecture: [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]]
- Techniques: [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]], [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]], [[projects/cloakbrowser/techniques/humanized-input|Humanized input]]
- Detection vectors: [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP and automation detection]], [[projects/cloakbrowser/detection-vectors/canvas-fingerprinting|Canvas fingerprinting]], [[projects/cloakbrowser/detection-vectors/behavioral-analysis|Behavioral analysis]]
- Network and profiles (B5): [[projects/cloakbrowser/subsystems/proxy-handling|Proxy handling]], [[projects/cloakbrowser/subsystems/profile-seed|Profile seed persistence]], [[projects/cloakbrowser/techniques/proxy-signal-removal|Proxy signal removal]], [[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]], [[projects/cloakbrowser/techniques/persistent-profile-warmup|Persistent profile warm-up]], [[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|Proxy and network signals]], [[projects/cloakbrowser/detection-vectors/webrtc-ip-leak|WebRTC IP leak]], [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]], [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]]
- Humanize, licensing and integrations (B1): [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]], [[projects/cloakbrowser/subsystems/humanize-errors|Humanize errors]], [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]], [[projects/cloakbrowser/subsystems/widevine-cdm-seeding|Widevine CDM seeding]], [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]], [[projects/cloakbrowser/techniques/noise-injection|Noise injection]], [[projects/cloakbrowser/detection-vectors/headless-detection|Headless detection]], [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]], [[projects/cloakbrowser/architecture/aws-lambda-deployment|AWS Lambda deployment]]
- Wrappers, fonts and test evidence (B4): [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]], [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]], [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]], [[projects/cloakbrowser/detection-vectors/font-fingerprinting|Font fingerprinting]], [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]]
- Binary lifecycle, launch core and network identity (B0): [[projects/cloakbrowser/architecture/release-channels-and-versioning|Release channels and versioning]], [[projects/cloakbrowser/architecture/supply-chain-verification|Supply-chain verification]], [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]], [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]], [[projects/cloakbrowser/subsystems/cli|CLI]], [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]], [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]], [[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency|Timezone/locale/IP consistency]]
- Rendering fingerprints, CDP hygiene and test evidence (B2): [[projects/cloakbrowser/detection-vectors/canvas-fingerprinting|Canvas fingerprinting]], [[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting|WebGL and GPU fingerprinting]], [[projects/cloakbrowser/detection-vectors/audio-fingerprinting|Audio fingerprinting]], [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]], [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]], [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]]
- Browser-property, network and profile-state vectors, test-suite pins (B8): [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]], [[projects/cloakbrowser/detection-vectors/headless-detection|Headless detection]], [[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting|TLS and network fingerprinting]], [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]], [[projects/cloakbrowser/detection-vectors/incognito-storage-quota-detection|Incognito and storage quota detection]]
- Related project: [[projects/cloakbrowser-manager/cloakbrowser-manager|CloakBrowser-Manager]], a GUI profile manager running this engine

## Sources

- `repo:github.com/CloakHQ/CloakBrowser`: `README.md` (commit `b1a00ed`, 26/10/2026) and `ast-extract` output
