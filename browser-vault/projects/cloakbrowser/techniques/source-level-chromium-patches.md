---
title: >-
    Source-level Chromium patches
category: techniques
tags: [chromium, stealth, fingerprinting, anti-detection, evasion, visibility/internal]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/architecture/wrapper-binary-split]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/headless-detection]]"
    type: mitigates
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md
  - repo:github.com/CloakHQ/CloakBrowser/BINARY-LICENSE.md
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py
summary: >-
    Why CloakBrowser modifies fingerprint surfaces in Chromium's C++ and compiles them in, instead of JS injection or flag tweaks, and what that buys and costs.
provenance:
  extracted: 0.68
  inferred: 0.27
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:46:41+01:00
updated: 2026-10-31T06:16:30+01:00
---

# Source-level Chromium patches

CloakBrowser's core technique is to change what Chromium reports at the C++ source level and ship the result as a compiled binary, rather than patching a stock browser from the outside. The README's framing: "Not a patched config. Not a JS injection. A real Chromium binary with fingerprints modified at the C++ source level." The patches cover canvas, WebGL, WebGPU, audio, fonts, GPU, screen, WebRTC, network timing, hardware reporting, automation signals and CDP input behavior; there are 87 in the Chromium 155 Pro build (as of 26/10/2026, commit b1a00ed). The patch sources are proprietary and not in the repository (see [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]]).

## Three ways to spoof, compared

| Approach | Examples | How it changes values | Weak points |
|---|---|---|---|
| JS injection | `playwright-stealth`, `puppeteer-extra` | Overrides properties via init scripts | Overrides are visible (non-native function `toString`, prototype descriptors, iframe/worker gaps, timing); break with Chrome updates |
| Config/flag patching | `undetected-chromedriver` | Launch flags and driver patching | Limited surface; flags themselves can be tells |
| Source-level patches | CloakBrowser | Changes the C++ that computes the value | Expensive to maintain; must be rebased on every Chromium major; closed binary |

Comparison table values in the README (as of Oct 2026): reCAPTCHA v3 0.1 stock Playwright, 0.3–0.5 playwright-stealth, 0.3–0.7 undetected-chromedriver, 0.9 CloakBrowser Pro. These are vendor-reported. ^[ambiguous]

## Why it works

- A value computed natively has no JavaScript wrapper to detect: property descriptors, `Function.prototype.toString`, and cross-realm checks (iframes, workers) all see the same native code path, so consistency checks between contexts pass. ^[inferred]
- Values can be coordinated across APIs inside the engine, for example GPU strings, [[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting|WebGL parameters]] and WebGPU adapter limits drawn from one profile, which per-API JS overrides tend to get wrong. CHANGELOG entries repeatedly cite "cross-API consistency" (0.3.0, 0.3.12, 0.4.0). ^[inferred] See [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].
- Automation tells are removed where they originate: `navigator.webdriver`, one of the signals probed in [[projects/cloakbrowser/detection-vectors/headless-detection|Headless detection]], is handled by a binary patch (patch 009, per CHANGELOG 0.3.21), CDP input synthesis is patched to look like real input (4 patches, 0.3.11), and locale is spoofed natively instead of through detectable CDP emulation (0.3.12). See [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP automation detection]].
- Because the network stack is real Chromium, TLS (JA3/JA4) and HTTP/2 fingerprints match Chrome automatically; the README cites this as the advantage over Firefox-based Camoufox. ^[inferred] See [[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting|TLS/network fingerprinting]].

## Design principles visible in the CHANGELOG

- **Flag discipline** (2026-02-28): patches activate only when configured via command-line flags; later (binary v14, 0.3.4) a random seed is auto-injected so the binary is stealthy with zero flags.
- **Coherence over randomness**: later releases emphasize "most common real-world hardware values chosen together" (148.0.7778.215.5) and stability of values across repeat reads and launch modes (150 line). Random noise is offered but can be disabled with `--fingerprint-noise=false`, because ML detectors flag it as tampering (see [[projects/cloakbrowser/techniques/noise-injection|Noise injection]] and [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]]).
- **Pass-through where real is better**: on Windows, native GPU values pass through instead of being spoofed (0.3.30), and `--fingerprint=off` (148+) presents the machine's real fingerprint for debugging.
- **Removing harmful patches**: 0.3.19 removed experimental patches that caused anti-bot compatibility issues; 0.3.10 fixed a regression caused by an unnecessary browser flag.
- **The reported Chrome version tracks real Chrome**, independent of the Chromium version built from (0.4.12). This explains why the README's test table shows UA `Chrome/151.0.0.0` while the Pro build is 155. ^[inferred]

## Costs

- **Rebase burden**: every Chromium major requires re-applying and adapting the whole patch set (146 → 147 → 148 in 0.4.0; 155 by October 2026), and per-platform builds lag each other (macOS stayed on 145 for free users). See [[projects/cloakbrowser/architecture/release-channels-and-versioning|Release channels and versioning]].
- **Trust**: users run a closed binary; mitigated by signed downloads (see [[projects/cloakbrowser/architecture/supply-chain-verification|Supply-chain verification]]) but not auditable.
- **Not sufficient alone**: the README's troubleshooting stresses that most blocks come from IP reputation, timezone/locale mismatch, headless mode, missing fonts or behavior, not the fingerprint itself. Source patches have to be combined with [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]], [[projects/cloakbrowser/techniques/humanized-input|Humanized input]] and a matching font set.
- The FAQ concedes detection is an arms race: source-level patches are "harder to detect than config-level patches, but not impossible".

## Open Questions

- Per-patch mapping to detection vectors is not published; counts are the only public metric.

## Sources

- `repo:github.com/CloakHQ/CloakBrowser/README.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/BINARY-LICENSE.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
