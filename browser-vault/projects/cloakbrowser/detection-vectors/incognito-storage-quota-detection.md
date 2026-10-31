---
title: >-
    Incognito and storage quota detection
category: detection-vectors
tags: [incognito, storage-quota, persistent-profile, browserscan, bot-detection]
aliases: [private mode detection, storage.estimate quota check, incognito detection]
relationships:
  - target: "[[projects/cloakbrowser/techniques/persistent-profile-warmup]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/profile-seed]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_persistent_context.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_profile_seed.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
summary: >-
    Sites infer private browsing from a small storage quota or an empty profile; CloakBrowser's quota normalization reads as incognito, fixed by a persistent profile and --fingerprint-storage-quota.
provenance:
  extracted: 0.66
  inferred: 0.28
  ambiguous: 0.06
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:54:19+01:00
updated: 2026-10-31T05:54:19+01:00
---

# Incognito and storage quota detection

Automation usually runs in a throwaway profile, and so does private browsing. Detectors that penalize incognito are therefore also catching most bots. The common probe is the storage quota from `navigator.storage.estimate()`. Chromium gives an incognito profile a small, memory-backed quota, while a normal profile's quota scales with free disk space. ^[inferred] An empty profile (no cookies, history or cached assets) is a softer version of the same signal. In CloakBrowser this interacts with a deliberate choice: the binary normalizes the quota. See [[projects/cloakbrowser/techniques/persistent-profile-warmup|Persistent profile warm-up]] for the countermeasure.

## The trade-off CloakBrowser makes

- **Quota normalization by default.** The binary normalizes the storage quota whenever `--fingerprint` is set, which is every wrapper launch. This hides the real disk size, which would otherwise be a high-entropy value and a tell for small VMs or containers. ^[inferred]
- **Side effect.** Detectors that infer private mode from the quota read the normalized default as incognito. The README names BrowserScan's incognito check, which costs −10% of its score (as of 26/10/2026, commit b1a00ed).
- **The override.** `--fingerprint-storage-quota=<MB>` sets the quota reported by `storage.estimate()`, `storageBuckets` and the legacy webkit storage APIs. The README example raises it to `5000` on a persistent context "to present as a regular profile".
- **Scope.** The README says quota tuning "is unrelated to FingerprintJS" and only affects detectors that infer incognito from the quota.

## Escaping incognito

1. **Use a real profile.** `launch()` opens an incognito context by default. `launch_persistent_context()` uses an on-disk user-data dir whose cookies, localStorage, cache, service workers and IndexedDB build up over time. The README lists "bypass incognito detection" as a reason to use it.
2. **Raise the quota** if the site still flags incognito.
3. `storage_state` restores cookies and localStorage into a fresh context, but it is still ephemeral and does not change the quota. ^[inferred]

A persistent profile also keeps its fingerprint seed, so a returning profile is the same device as well as a non-incognito one. See [[projects/cloakbrowser/subsystems/profile-seed|Profile seed persistence]].

## Pinned by tests

- No test measures the quota or an incognito verdict.
- `tests/test_persistent_context.py` requires that the stealth args and the caller's extra args are both passed to Playwright's `launch_persistent_context`, so a `--fingerprint-storage-quota` in `args` reaches the binary.
- `tests/test_profile_seed.py` requires that a persistent profile reuses its stored seed while non-persistent launches stay random.
- The live suite (`tests/test_stealth.py`) runs BrowserScan with plain `launch()`, which opens an incognito context with the normalized quota, and still requires zero "Abnormal" verdicts. So the incognito penalty lowers BrowserScan's score without producing an "Abnormal" bot-detection verdict. ^[inferred]

## Open Questions

- The normalized default quota value is not documented, so it is unclear how far below a typical desktop profile it sits.
- Whether the binary reports a quota consistent with the claimed persona's typical disk size is not stated. ^[ambiguous]

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/README.md (launch_persistent_context, Additional Flags, Troubleshooting: incognito and FingerprintJS)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_persistent_context.py
- repo:github.com/CloakHQ/CloakBrowser/tests/test_profile_seed.py
- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
