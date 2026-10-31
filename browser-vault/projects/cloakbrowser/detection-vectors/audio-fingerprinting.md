---
title: >-
    Audio fingerprinting
category: detection-vectors
tags: [audio, fingerprinting, bot-detection, web-audio]
aliases: [AudioContext fingerprint, OfflineAudioContext hash]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
summary: >-
    Web Audio output varies subtly by platform and is hashed as a device ID; CloakBrowser derives audio traits from the seed and adds noise, which --fingerprint-noise=false turns off.
provenance:
  extracted: 0.45
  inferred: 0.45
  ambiguous: 0.10
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:52:24+01:00
updated: 2026-10-31T05:52:24+01:00
---

# Audio fingerprinting

Audio fingerprinting renders a short signal through the Web Audio API (typically an oscillator through a compressor in an `OfflineAudioContext`) and hashes the output samples. Floating-point and DSP implementation differences between CPUs, OSes and browser builds make the result a device identifier that needs no permission and plays no sound. ^[inferred] CloakBrowser lists audio among its source-level patch areas, uses the master `--fingerprint` seed for audio alongside canvas, WebGL and fonts, and includes audio in the noise that `--fingerprint-noise=false` disables (README, as of 26/10/2026, commit b1a00ed). The repository says little more, so most of this page is context and inference. Related: [[projects/cloakbrowser/techniques/noise-injection|Noise injection]], [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].

## What CloakBrowser states

- `--fingerprint` is the "master seed for canvas, WebGL, audio, fonts"; with no flags a random seed is generated and "all noise patches are spoofed automatically".
- `--fingerprint-noise=false` disables noise on canvas, WebGL **and audio** while keeping the deterministic seed active.
- Since build 148.0.7778.215.5, the seed draws from "the most common real-world hardware values (… fonts, audio)", per [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]]. The audio profile is therefore picked for commonness, not only randomized.

## Detection logic and trade-offs

- **Uniqueness vs crowd.** A seeded audio profile chosen from common values helps a session blend in, and seed noise separates sessions from each other. ^[inferred]
- **Stability.** A real device returns the same audio hash on repeated renders. Noise that changed between renders would flag the browser, so the noise is presumably seed-stable. ^[inferred]
- **Tampering detection.** Detectors can render the same buffer twice or compare against known per-platform values to spot perturbation. This is the same risk that leads the README to disable noise for FingerprintJS. See [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]]. ^[inferred]
- **Persona consistency.** An audio hash typical of Linux under a Windows persona would contradict the platform claim; whether the persona changes audio output is not stated. ^[ambiguous]

## Open Questions

- Which Web Audio surfaces are patched (`OfflineAudioContext` rendering, `AnalyserNode` data, `baseLatency`, `sampleRate`, channel counts) is not documented. ^[ambiguous]
- No test in `examples/stealth_test.py` targets audio specifically.

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/README.md (feature list, Fingerprint Management, Default Fingerprint, Additional Flags)
