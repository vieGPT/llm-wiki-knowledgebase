---
title: >-
    Noise injection
category: techniques
tags: [fingerprinting, canvas, webgl, audio, stealth, visibility/internal]
aliases: [fingerprint noise, --fingerprint-noise]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/audio-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/examples/stealth_test.py
summary: >-
    Seed-derived noise the binary adds to canvas, WebGL and audio readouts to vary fingerprints per identity; can be turned off with --fingerprint-noise=false when ML flags it as tampering.
provenance:
  extracted: 0.53
  inferred: 0.42
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T05:52:24+01:00
---

# Noise injection

Noise injection is the part of CloakBrowser's C++ fingerprint patching that perturbs canvas, WebGL and audio readouts so that each identity produces its own hash for these surfaces. The perturbation is driven by the fingerprint seed: "each seed gets unique canvas noise and other browser signals", and with no flags a random seed is generated at startup and "all noise patches are spoofed automatically" (README, as of 26/10/2026, commit b1a00ed). The patches live in the closed-source binary, not in this repository, so the mechanism details below are inferred from flags and documentation. ^[inferred] See [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]] and [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].

## How it is controlled

| Setting | Effect |
|---|---|
| no flags | random seed per launch, noise on: a fresh canvas/WebGL/audio identity each launch |
| `--fingerprint=<seed>` | deterministic noise: same seed, same hashes across launches (returning visitor) |
| `--fingerprint-noise=false` | disable noise on canvas, WebGL and audio while keeping the deterministic seed active for everything else |
| `--fingerprint=off` (Chromium 148+) | all spoofing off, real native fingerprint (debug pass-through) |

In CloakServe, a `fingerprint` query parameter on the CDP URL gives each connection its own seed and therefore its own noise pattern (see [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|CloakServe CDP server]]).

## Why it works

- Canvas, WebGL and audio fingerprints hash rendering output that differs subtly between GPU, driver and OS stacks. Adding small seed-stable perturbations makes the hash unique per seed and stable within a seed, so a fleet of sessions does not share one cross-site identifier. ^[inferred]
- Seed stability matters: noise that changed on every read would itself be a giveaway, since a real device returns the same canvas hash on repeated draws. ^[inferred]

## Trade-off: tampering detection

The README's FingerprintJS troubleshooting table lists "Browser tampering — noise injection detected by ML" and gives `--fingerprint-noise=false` as the fix; the recommended FingerprintJS-passing configuration disables noise (with a residential proxy, `geoip=True`, and Windows font metrics on Chromium 148+). Noise trades cross-session unlinkability against looking like a modified browser:

- Noise on: each seed has a distinct canvas/WebGL/audio hash, but ML-based detectors may classify the perturbation as tampering. See [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]].
- Noise off: readouts are the unperturbed output of the spoofed device profile, which looks untampered, but sessions on the same hardware may share these hashes. ^[inferred]

## Test evidence: CreepJS (B2)

`examples/stealth_test.py` runs most detectors with noise on (the default), but launches a **separate** browser with `--fingerprint-noise=false` for CreepJS and requires `totalLies == 0`. This is the clearest in-repository sign that the project expects lie detectors such as CreepJS to count seed noise as lies, while classic bot checks (sannysoft, incolumitas, Rebrowser, BrowserScan, deviceandbrowserinfo) pass with noise on. ^[inferred] Per-surface detail: [[projects/cloakbrowser/detection-vectors/canvas-fingerprinting|Canvas fingerprinting]], [[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting|WebGL and GPU fingerprinting]], [[projects/cloakbrowser/detection-vectors/audio-fingerprinting|Audio fingerprinting]].

## Open Questions

- What the noise actually perturbs (pixel values, WebGL parameters, audio samples) and its magnitude are not documented in the repository. ^[ambiguous]
- Whether `--fingerprint-noise=false` also affects other "noise patches" beyond canvas, WebGL and audio is not stated.

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/README.md
- repo:github.com/CloakHQ/CloakBrowser/examples/stealth_test.py (B2)
