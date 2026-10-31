---
title: >-
    Fingerprint tampering detection
category: detection-vectors
tags: [tampering, fingerprinting, fingerprintjs, creepjs, bot-detection]
aliases: [browser tampering, lie detection, anti-fingerprinting detection]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/font-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
summary: >-
    Detectors flag a fingerprint as tampered when readouts look perturbed or contradict the claimed platform; CloakBrowser's noise and persona spoofing can trigger it, so some configs turn noise off.
provenance:
  extracted: 0.55
  inferred: 0.37
  ambiguous: 0.08
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:54:19+01:00
updated: 2026-10-31T06:37:29+01:00
---

# Fingerprint tampering detection

Tampering detection asks not "what device is this?" but "is this device lying?". Instead of matching a fingerprint against known bots, it looks for evidence that fingerprint surfaces were modified: noise in canvas, WebGL or audio output, values that contradict each other, or JavaScript overrides that show through. A stealth browser has to beat this as well as the plain fingerprint checks, because a convincing fake that looks modified is still flagged. ^[inferred] It is the counterweight to [[projects/cloakbrowser/techniques/noise-injection|Noise injection]] and [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]].

## How tampering shows up

| Tell | What the detector sees | CloakBrowser source |
|---|---|---|
| Perturbed readouts | canvas, WebGL or audio output that a classifier recognizes as noised (FingerprintJS: "noise injection detected by ML") | README FingerprintJS troubleshooting table |
| Platform contradiction | font metrics that do not fit the spoofed Windows platform (FingerprintJS "Browser tampering (fonts)") | README; see [[projects/cloakbrowser/detection-vectors/font-fingerprinting\|Font fingerprinting]] |
| Geometry contradiction | screen dimensions that do not fit the viewport (FingerprintJS "Virtual machine") | README; see [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency\|Screen and viewport consistency]] |
| JavaScript overrides | patched getters and prototypes that betray themselves | README "Why CloakBrowser": antibot systems "detect the patches themselves" in JS-injection tools |
| Aggregate lie score | CreepJS's "stealth" percentage | `examples/fingerprint_scan_test.py`, see [[projects/cloakbrowser/detection-vectors/headless-detection\|Headless detection]] |

The detection techniques themselves (repeat reads, comparison with known per-platform values, `toString` checks on native functions) are general knowledge, not described in the repository. ^[inferred]

## CloakBrowser's levers

- **Source-level patches** leave no JavaScript override to unmask, so the "JS overrides" row does not apply to the binary's spoofing. ^[inferred] See [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]].
- **`--fingerprint-noise=false`** turns off canvas, WebGL and audio noise but keeps the seed's deterministic identity. The README's FingerprintJS-passing config uses it ("prevents tampering detection"). The cost: without noise, sessions on the same hardware may produce the same render hashes. ^[inferred]
- **`--fingerprint-windows-font-metrics`** (Chromium 148+, needs Windows fonts installed; on by default from `154.0.8037.57.2` when the fonts are present) aligns font metrics with the Windows persona.
- **Coherent generation.** The seed derives GPU, screen, CPU and memory as one package, and the README warns that hand-editing single values can break that coherence. It also notes that all stealth tests use the default fingerprint config. See [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].

## Default config vs FingerprintJS

The defaults keep noise on. The live suite in `tests/test_stealth.py` still expects the FingerprintJS **web-scraping demo** to pass with defaults: flight data shown, no "bot visit detected". The tampering verdicts in the README come from FingerprintJS's **playground**. The two do not conflict: the scraping demo gates only on the bot verdict, while the playground reports tampering as a separate signal that a site may or may not act on. ^[inferred] Whether to disable noise therefore depends on the target. ^[ambiguous]

## Open Questions

- What share of real targets act on a tampering verdict, as opposed to just recording it, is not known from these sources.
- Whether the binary's noise is stable across repeated reads within a session (which defeats double-render checks) is claimed for the 150 line in other pages but not tested here. ^[ambiguous]

## Related

- [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|Per-profile identity isolation (Manager)]] — the Manager locks persona and GPU family to what the host can render coherently.

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/README.md (Why CloakBrowser, Default Fingerprint, Additional Flags, Troubleshooting: Detected by FingerprintJS)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
