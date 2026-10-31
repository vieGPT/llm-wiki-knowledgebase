---
title: >-
    WebGL and GPU fingerprinting
category: detection-vectors
tags: [webgl, gpu, fingerprinting, bot-detection, headless]
aliases: [WebGL fingerprint, UNMASKED_RENDERER_WEBGL, GPU vendor spoofing]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/examples/stealth_test.py
summary: >-
    Detectors read the WebGL vendor/renderer strings, parameters and rendered output to identify the GPU; CloakBrowser derives a GPU from the seed and persona, with a Windows passthrough mode.
provenance:
  extracted: 0.60
  inferred: 0.33
  ambiguous: 0.07
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:52:24+01:00
updated: 2026-10-31T10:47:25+01:00
---

# WebGL and GPU fingerprinting

WebGL exposes the graphics stack to page JavaScript. A detector can read the unmasked vendor and renderer strings (`WEBGL_debug_renderer_info`), dozens of capability parameters and extension lists, and the pixels of a rendered 3D scene. Together these identify the GPU and driver, and they reveal servers and headless containers, which typically report software renderers such as SwiftShader. ^[inferred] CloakBrowser treats the GPU as part of the seeded device package: the README lists WebGL and GPU among its source-level patch areas and says the binary auto-generates the GPU from the seed. See [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]] and [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]].

## Signals and CloakBrowser's answer (README, as of 26/10/2026, commit b1a00ed)

| Signal | Risk | Control |
|---|---|---|
| Unmasked vendor/renderer string | software renderer or server GPU gives away a VM ^[inferred] | seed + platform choose a GPU from a pool; `--fingerprint-gpu-vendor=NVIDIA\|Intel` switches the family and reports `Google Inc. (NVIDIA)` or `Google Inc. (Intel)` |
| GPU vs OS persona | Apple GPU under a Windows UA, or the reverse | `--fingerprint-platform` also controls "GPU pool selection"; a Linux build natively reports Linux with NVIDIA, a macOS build macOS with Apple GPU |
| WebGL parameters and rendered output | parameters that do not match the claimed card; identical render hashes across a fleet ^[inferred] | v0.6.4 notes "corrected GPU/display/graphics parameters to match stock Chrome profiles"; seed-driven WebGL noise ([[projects/cloakbrowser/techniques/noise-injection\|Noise injection]]) |
| Real Windows hardware | spoofing a real GPU can create inconsistencies with what the hardware actually renders ^[inferred] | "Windows native GPU passthrough": on Windows real hardware values pass through instead of being spoofed |
| Missing GPU APIs in headless/Docker | no WebGL at all is a headless tell ^[inferred] | wrapper passes `--ignore-gpu-blocklist` in headed and Windows cases; see [[projects/cloakbrowser/techniques/platform-persona-spoofing\|Platform persona spoofing]] and [[projects/cloakbrowser/detection-vectors/headless-detection\|Headless detection]] |

The README warns that setting single values by hand (for example a GPU vendor that does not fit the screen or core count) can break the coherence the seed provides. ^[inferred]

## Evidence in the stealth suite

Before running its detector checks, `examples/stealth_test.py` prints the device the browser presents: the WebGL unmasked vendor and renderer, plus UA, platform, core count, screen size and languages. The project treats the GPU string as a headline identity value that the user should eyeball. ^[inferred] No detector in the suite scores WebGL by itself; it is covered indirectly by BrowserScan, CreepJS and deviceandbrowserinfo. ^[ambiguous]

## Open Questions

- Which GPU models the pools contain and how the seed selects among them is not visible in the repository.
- How far WebGL readback noise and the real rasterizer output can diverge before a renderer-vs-pixels consistency check notices is not documented. ^[ambiguous]
- WebGPU (`navigator.gpu` adapter info) is mentioned only indirectly, through `--ignore-gpu-blocklist`.

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/README.md (Latest v0.6.4 notes, Fingerprint Management, Default Fingerprint, Additional Flags)
- ~/code/CloakBrowser/examples/stealth_test.py (device info printout)
