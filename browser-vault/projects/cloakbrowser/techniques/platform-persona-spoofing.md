---
title: >-
    Platform persona spoofing
category: techniques
tags: [persona, fingerprinting, user-agent, fonts, stealth, visibility/internal]
aliases: [--fingerprint-platform, Windows persona, macOS persona]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/font-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/screen-viewport-consistency]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/behavioral-analysis]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/args.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/fonts.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/engine.ts
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py
  - repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md
summary: >-
    The wrapper makes Linux/Windows hosts present as a Windows desktop and macOS hosts as a native Mac via --fingerprint-platform, then keeps fonts, input and window geometry consistent with that persona.
provenance:
  extracted: 0.73
  inferred: 0.21
  ambiguous: 0.06
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:47:03+01:00
updated: 2026-10-31T06:37:29+01:00
---

# Platform persona spoofing

A persona is the operating system the browser claims to run on. The binary picks its default persona at compile time: a Linux build reports Linux with an NVIDIA GPU, a macOS build reports macOS with an Apple GPU. The wrapper overrides this with `--fingerprint-platform`, which controls `navigator.platform`, the OS in the User-Agent string and the GPU pool the seed draws from (README). Detectors check whether those signals agree with each other and with everything else the host leaks, mainly fonts, input behavior and window geometry. Covered detection surfaces: [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]] and [[projects/cloakbrowser/detection-vectors/font-fingerprinting|Font fingerprinting]].

## Defaults the wrapper applies

| Host | Flag set by `getDefaultStealthArgs` | Rationale |
|---|---|---|
| Linux, Windows | `--fingerprint-platform=windows` | Windows desktops are the most common fingerprint and "harder to cluster" (README) |
| macOS | `--fingerprint-platform=macos` | Run as a native Mac browser, where the GPU and UA already match |

A user-supplied `--fingerprint-platform` overrides the default, because user args beat stealth defaults in `buildArgs` (see [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]). On Linux the macOS persona is described as "a second identity" to rotate with Windows. It needs the Mac fonts installed and, ideally, a large display, so that it reports the device pixel ratio most real Macs use (README).

## Keeping the rest of the stack consistent

The persona flag alone is not enough. The wrapper aligns the other layers:

- **Fonts**: the host's real fonts are rendered. On Linux the wrapper warns once when a Windows persona lacks the 8 Windows OS fonts, and `info` counts the Windows, Office and macOS font sets. With Windows fonts installed, `--fingerprint-windows-font-metrics` aligns glyph metrics, and the launcher then re-enables `MediaRouter` so the feature set looks like stock Chrome.
- **Input**: the humanize engine reads `navigator.platform` inside its isolated world, not the host OS. That decides Meta vs Control for select-all, and the mouse-wheel notch size (100 px Windows, 40 px macOS, 120 px Linux) when `scroll_notch_px` is 0. A Windows persona therefore scrolls and edits like a Windows user even on a Linux host. See [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]] and [[projects/cloakbrowser/detection-vectors/behavioral-analysis|Behavioral analysis]].
- **Window geometry**: the headless default viewport of 1920x947 models a maximized Chrome on a 1080p Windows screen minus about 85 px of browser UI. Headed launches use the real window, and `--start-maximized` is version-gated so that `outerWidth >= innerWidth` always holds (see [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]]).
- **GPU blocklist**: on Windows hosts `--ignore-gpu-blocklist` keeps WebGPU available on the Microsoft Basic Render Driver. In headed Docker/Xvfb it lets SwiftShader serve WebGL, so the persona is not betrayed by missing GPU APIs (see [[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting|WebGL and GPU fingerprinting]]). ^[inferred]

## Why not spoof Windows on a Mac

The Python `get_default_stealth_args()` docstring gives the reason macOS hosts keep a native persona: "Spoofing Windows on Mac creates detectable mismatches (fonts, GPU, etc.)". The asymmetry follows from what can and cannot be faked: a Linux server has no strong native identity of its own and can be provisioned with Windows fonts, while a Mac already presents a coherent, common Apple identity. ^[inferred]

## Evolution (CHANGELOG, dated)

- 2026-03-03 (0.3.3): binary becomes platform-aware, matching GPU, screen and UA to the real OS by default; the wrapper briefly passed explicit Mac GPU flags (`Apple M3 Metal`), later dropped as the binary took over GPU selection (0.3.21).
- 2026-03-05 (0.3.9): binary "font auto-hide for cross-platform fingerprints".
- 2026-06-22 (0.4.0): opt-in `--fingerprint-windows-font-metrics` for Windows profiles on Linux.
- 2026-07-05 (148.0.7778.215.5): "deeper, more consistent Windows persona" with expanded, localized font populations and seed-coherent screen, color depth and audio.
- 2026-07-18 (150.0.7871.114.3): `--fingerprint-sapi-voices` so the Windows speech-voice list matches a real install.
- 2026-07-25 / 2026-08-18 (150.4 / 150.6): closer Windows text rendering, Greek locale, expanded Windows hardware pool, "host-font isolation for cross-platform personas".
- 2026-10-08 (0.6.0): Linux Preview `154.0.8037.57.2` makes the Windows persona's system and code fonts match real Windows by default.
- By 2026-10-26 the README documents the macOS persona on Linux hosts as a second rotatable identity.

The direction is from "claim an OS" toward "reproduce that OS's whole surface" (fonts, voices, text metrics, hardware pool). ^[inferred]

## Why it works

Detectors cluster sessions by OS and hardware. A Windows persona puts automation traffic in the largest population of real users. Coherence checks such as fonts present for the claimed OS, OS-specific scroll deltas and plausible window chrome remove the cheap ways to prove the claim false. ^[inferred]

## Open Questions

- The README says the binary derives "screen dimensions" from the seed, but a `config.ts` comment states that screen and window size "come from the real display, not this flag (verified: identical across seeds)". The two may describe different things (headless vs headed, or `screen.*` vs window size), or one may be outdated. ^[ambiguous] The Python `config.py` places the same comment specifically in the headed-mode context ("the wrapper must not emulate a viewport on top in headed mode"), and the README lists seed-derived `--fingerprint-screen-width/height` defaults of 1920×1080 (Win/Linux) and 1440×900 (macOS), which suggests the seed governs headless/`screen.*` values while headed window size follows the real display. ^[inferred]
- A Linux persona (`--fingerprint-platform=linux`) is not part of the wrapper defaults. Whether the binary supports it is not stated in this batch's sources.

## Related

- [[projects/cloakbrowser/detection-vectors/canvas-fingerprinting|Canvas fingerprinting]] — canvas text renders use the host's fonts, so a Windows or macOS persona needs that OS's fonts installed for its canvas hashes to fit the claim.
- [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|Per-profile identity isolation (Manager)]] — CloakBrowser-Manager always picks the persona from the host runtime, never from profile data.

## Sources

- `repo:github.com/CloakHQ/CloakBrowser` at commit `b1a00ed`: `js/src/config.ts`, `js/src/args.ts`, `js/src/fonts.ts`, `js/src/human/config.ts`, `js/src/human/engine.ts`, `README.md` (sections "Fingerprint Management", "Default Fingerprint")
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
