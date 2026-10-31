---
title: >-
    Fingerprint seed identity
category: techniques
tags: [fingerprinting, identity, stealth, chromium, evasion, visibility/internal]
aliases: [--fingerprint seed, seed identity]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/profile-seed]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/audio-fingerprinting]]"
    type: mitigates
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/cloakbrowser/config.py
  - ~/code/CloakBrowser/cloakbrowser/browser.py
summary: >-
    One integer seed (--fingerprint) deterministically derives a whole coherent device (GPU, screen, CPU, RAM, fonts, noise), so identities are reproducible and internally consistent.
provenance:
  extracted: 0.76
  inferred: 0.2
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:47:15+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Fingerprint seed identity

The binary models a browser identity as a function of one master seed passed as `--fingerprint=<seed>`. From that seed and the persona platform it derives the GPU (vendor/renderer, WebGL and WebGPU parameters), hardware concurrency, device memory, screen size and color depth, font population, [[projects/cloakbrowser/detection-vectors/audio-fingerprinting|audio characteristics]], and the per-API noise patterns for canvas, WebGL and audio. The same seed reproduces the same device; a different seed is a different device. This is the identity layer that the [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]] implement.

## Modes

| Input | Result |
|---|---|
| No flags | Binary auto-generates a random seed at startup (binary v14, 0.3.4): a fresh, coherent identity each launch |
| `--fingerprint=<seed>` | Deterministic identity; same seed = same fingerprint across launches |
| Seed + explicit flags (`--fingerprint-gpu-vendor`, `--fingerprint-hardware-concurrency`, `--fingerprint-device-memory`, `--fingerprint-screen-width/height`, `--fingerprint-brand`, …) | Explicit flags override single values; the seed fills in the rest |
| `--fingerprint=off` (148+) | Pass-through: real native fingerprint, injected seed and platform stripped |
| `--fingerprint-noise=false` | Keeps the deterministic seed but disables canvas/WebGL/audio noise (see [[projects/cloakbrowser/techniques/noise-injection\|Noise injection]]) |

The wrapper always passes a seed: a random integer in 10000–99999 per launch, or for persistent profiles a seed stored in `<user_data_dir>/.cloakbrowser-seed` (0.6.0; see [[projects/cloakbrowser/subsystems/profile-seed|Profile seed]]). cloakserve takes it per connection as `?fingerprint=<seed>` (see [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]]).

## Why one seed instead of many knobs

- **Coherence.** Detectors cross-check values: a high-end GPU with 2 cores, a 4K screen with a laptop-class GPU, or a Windows UA with an Apple GPU are implausible combinations. Drawing all values together from a pool of real hardware profiles avoids contradictions. The README warns that "changing single values by hand can break that coherence". ^[inferred]
- **Popularity.** Since 148.0.7778.215.5 each seed is built from "the most common real-world hardware values (screen, GPU, RAM, CPU cores, color depth, fonts, audio), chosen together", so the device blends into the largest crowd instead of being a unique outlier. Expanded NVIDIA profiles are each matched to a plausible CPU, RAM and screen. ^[inferred]
- **Diversity across users.** In 0.3.21 the wrapper stopped passing hardcoded GPU vendor/renderer flags so that each seed gets its own GPU instead of every user sharing one, which would itself be a cluster signal.
- **Determinism for returning visitors.** Scoring systems such as reCAPTCHA v3 Enterprise reward a consistent device on repeat visits from the same IP; a random seed per session "makes every session look like a different device" (README tip). Persistent profiles therefore keep their seed. ^[inferred]

## Trade-offs and caveats

- Seed → identity mapping is versioned with the binary: 0.5.8 notes "fixed seeds may resolve to a different complete hardware identity than on the previous Stable build". Pinning a seed does not pin a device across binary upgrades. ^[inferred] See [[projects/cloakbrowser/architecture/release-channels-and-versioning|Release channels and versioning]].
- The seed range used by the wrapper (90,000 values) is small; whether the binary's identity pool is smaller still is not documented. ^[ambiguous]
- Screen and window size in headed mode come from the real display, not the seed (wrapper comment, "verified: identical across seeds"), so the host display still matters; see [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen/viewport consistency]].
- `--fingerprint-storage-quota` is auto-normalized when a seed is set, which hides disk size but reads as incognito to some detectors (see [[projects/cloakbrowser/detection-vectors/incognito-storage-quota-detection|Incognito storage-quota detection]]).
- Persona platform is a separate input (`--fingerprint-platform`); the seed picks values from that platform's pool. See [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]].

## Related

- [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|Per-profile identity isolation (Manager)]] — CloakBrowser-Manager stores one seed per profile and passes it as `--fingerprint`.

## Sources

- `~/code/CloakBrowser/README.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
