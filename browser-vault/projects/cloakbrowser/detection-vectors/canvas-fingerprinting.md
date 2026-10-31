---
title: >-
    Canvas fingerprinting
category: detection-vectors
tags: [canvas, fingerprinting, bot-detection, fonts, anti-bot]
aliases: [canvas hash, emoji canvas hash, 2D canvas fingerprint]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/font-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/examples/stealth_test.py
  - ~/code/CloakBrowser/images/fingerprintjs_pass.png
summary: >-
    Detectors draw text, emoji and shapes on a hidden canvas and hash the pixels; CloakBrowser varies the hash per seed with noise, but missing host fonts and detectable noise are its weak spots.
provenance:
  extracted: 0.55
  inferred: 0.38
  ambiguous: 0.07
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:52:24+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Canvas fingerprinting

Canvas fingerprinting asks the browser to render text, emoji and shapes into an off-screen `<canvas>` and hashes the resulting pixels (via `toDataURL` / `getImageData`). Small differences in fonts, anti-aliasing, GPU rasterization and OS text stacks make the hash a stable device identifier, and the hash can also be compared against a database of hashes that real browsers produce. ^[inferred] CloakBrowser names canvas first in its list of source-level patch areas (README, 87 patches in the v0.6.4 / Chromium 155 build, as of 26/10/2026, commit b1a00ed). Canvas is where three of its mechanisms meet: [[projects/cloakbrowser/techniques/noise-injection|Noise injection]], [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]] and the host-font requirement from [[projects/cloakbrowser/detection-vectors/font-fingerprinting|Font fingerprinting]].

## What detectors use the hash for

| Use | What fails it | Where the README mentions it |
|---|---|---|
| Cross-site tracking / linking sessions | Many sessions sharing one hash | noise per seed: "each seed gets unique canvas noise" (CloakServe section) |
| "Is this a real browser?" lookup | A hash that matches no known real device | Kasada and Akamai "render emoji on a hidden canvas and hash the pixel output"; minimal Linux images produce "hashes that don't match any real browser" |
| Tampering / lie detection | Pixel readouts that look perturbed or inconsistent across reads | FingerprintJS "Browser tampering: noise injection detected by ML"; CreepJS lie count (see below) |
| Persona consistency | Glyph shapes from Linux fonts under a Windows or macOS persona | Font metrics and font setup sections ^[inferred] |

## How CloakBrowser responds

- **Seed-driven noise.** `--fingerprint=<seed>` is the "master seed for canvas, WebGL, audio, fonts"; with no flags a random seed (10000 to 99999) is drawn at startup. The same seed reproduces the same canvas hash, so a pinned seed looks like a returning visitor. ^[inferred]
- **Noise off switch.** `--fingerprint-noise=false` disables the canvas, WebGL and audio noise while keeping the rest of the seeded identity. The README's FingerprintJS-passing config uses it.
- **Host fonts.** Noise cannot add glyphs the host lacks. The README calls missing emoji and CJK fonts on minimal Linux "the most common cause of blocks on aggressive sites after proxy, geoip, and headed mode are already set up correctly", and tells users to install the font packages from "Font Setup on Linux". Real Windows (or Mac) fonts are also needed when the persona claims that OS. See [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]].

## The core trade-off

Noise makes two sessions unlinkable, but a perturbed canvas is itself a signal: ML-based detectors (FingerprintJS) and lie detectors (CreepJS) look for exactly that. Leaving noise off gives a clean, plausible hash that is shared by every session on the same spoofed device profile and host. ^[inferred] CloakBrowser picks per target:

- `examples/stealth_test.py` runs its CreepJS check in a separate browser launched with `--fingerprint-noise=false` and passes only with **zero** CreepJS lies (`window.Fingerprint.lies.totalLies == 0`). This implies the project expects CreepJS to count noise as lies. ^[inferred]
- The README's recommended FingerprintJS config also turns noise off. The `fingerprintjs_pass.png` screenshot shows the FingerprintJS web-scraping demo serving flight results (SFO to JFK, February 2026 dates) instead of a block page. ^[inferred] Which config produced the screenshot is not recorded. ^[ambiguous]

See [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]] and [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]].

## Open Questions

- What the canvas noise changes (pixel LSBs, sub-pixel offsets, specific APIs) and how it stays stable across repeated reads is not visible in the repository; the patches are in the closed binary. ^[ambiguous]
- Whether the Windows persona reproduces Windows text rasterization (ClearType, DirectWrite) or only font metrics is not documented.

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/README.md (Fingerprint Management, Additional Flags, Font Setup on Linux, Troubleshooting: FingerprintJS and Kasada/Akamai, CDP server mode)
- ~/code/CloakBrowser/examples/stealth_test.py (CreepJS noise-off test)
- ~/code/CloakBrowser/images/fingerprintjs_pass.png (vision-interpreted)
