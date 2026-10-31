---
title: >-
    Cross-profile linkage
category: detection-vectors
tags: [multi-account, identity, bot-detection, fingerprinting, network-identity]
aliases: [account linking, multi-accounting detection]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/font-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/audio-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/behavioral-analysis]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/humanized-input]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/diagnostics]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/README.md
  - ~/code/CloakBrowser-Manager/backend/database.py
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/backend/artifacts.py
  - ~/code/CloakBrowser-Manager/CHANGELOG.md
  - ~/code/CloakBrowser-Manager/BINARY-LICENSE.md
summary: >-
    Sites link accounts run from one operator by shared signals across profiles: same IP, matching fingerprint parts, shared cookies or files, identical seeds after duplication.
provenance:
  extracted: 0.45
  inferred: 0.50
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T06:20:23+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Cross-profile linkage

Cross-profile linkage is the detection that matters most to a profile manager: not "is this a bot" but "are these accounts the same person". The README frames the product against it: "nothing bleeds between profiles, so your accounts never link back to each other, or to you." A site does not need to identify the browser as automated; it only needs one signal that two accounts share. ^[inferred] Per-browser surfaces are covered by the CloakBrowser detection-vector pages; this page lists what links profiles *to each other* when they run from one Manager.

## Linking signals and the Manager's answer

| Shared signal | How detectors use it | Manager countermeasure | Residual risk |
|---|---|---|---|
| Device fingerprint ([[projects/cloakbrowser/detection-vectors/canvas-fingerprinting\|canvas]], WebGL, [[projects/cloakbrowser/detection-vectors/audio-fingerprinting\|audio]], hardware) | Equal hashes across accounts | Distinct [[projects/cloakbrowser/techniques/fingerprint-seed-identity\|seed]] per profile; seed re-roll on reset ([[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]) | Seeds come from 10000 to 99999 (90,000 values). Many profiles across many operators make collisions plausible; within one Manager the chance of a pair sharing a seed passes about 1% at roughly 40 profiles (birthday bound). ^[inferred] |
| Duplicated seed | Clone and source present the same device | None by design: duplicate keeps the seed so the clone is the "same identity" | Running both, or using them for different accounts, presents one device twice. Re-rolling the clone's seed (reset, or editing it) breaks this. ^[inferred] |
| Cookies, storage, history | Shared first-party or third-party IDs | One Chromium user-data dir per profile | Duplicating *with browser state* copies every cookie and session on purpose ([[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]). [[projects/cloakbrowser-manager/techniques/portable-cookies\|Portable cookies]] (v0.1.8) move logins between machines. |
| Exit IP and geography | Same IP or subnet, IP reputation | Per-profile proxy; timezone/locale follow the proxy exit via GeoIP ([[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]) | Profiles without a proxy all exit on the host IP ([[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]). ^[inferred] |
| Host fonts and GPU | Font lists and render quirks shared by every profile on one host | Engine persona handling; Windows font health shown in the top bar for Linux Docker | Font coverage is per host, not per profile ([[projects/cloakbrowser/detection-vectors/font-fingerprinting]]). Docker profiles all render through SwiftShader. ^[inferred] |
| Uploaded files and downloads | Same document uploaded from several accounts | Artifacts are stored per profile, outside the user-data dir, and never copied by duplication | Operator behaviour, not the Manager, decides whether the same file goes to several accounts. ^[inferred] |
| Timing and behaviour ([[projects/cloakbrowser/detection-vectors/behavioral-analysis]]) | Accounts active at the same moments with the same input rhythm | Optional per-profile [[projects/cloakbrowser/techniques/humanized-input\|humanize]] | Auto-launch starts flagged profiles in sequence at startup, which can produce correlated session-start times. ^[inferred] |
| Default bookmarks and search engine | Rarely read by pages | Every profile gets the same detection-test bookmark set and Google default | Bookmarks are not web-exposed, so the uniform set is not a page-visible signal. ^[inferred] |

## Operator-side leakage (not page-visible)

`manager.log` ([[projects/cloakbrowser-manager/subsystems/diagnostics]]) records each launch's seed and a credential-redacted proxy, and the database holds every profile's seed and proxy in one file. These link profiles for anyone holding the log or data dir, not for websites. ^[inferred]

## Open Questions

- Does the CloakBrowser engine derive anything from the [[projects/cloakbrowser-manager/subsystems/settings-and-license|license key]] or binary build that is identical across profiles and visible to pages? The binary license (v1.3, July 2026) says CloakHQ telemetry is limited to license validation and session counting, which is not page-visible. ^[ambiguous]
- How fine-grained is the seed's effect? If several distinct seeds map to the same GPU model or screen class, collisions on individual surfaces are more frequent than on the full seed. ^[ambiguous]

## Sources

- ~/code/CloakBrowser-Manager/README.md (commit 11e0043)
- ~/code/CloakBrowser-Manager/backend/database.py, backend/browser_manager.py, backend/main.py, backend/artifacts.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/CHANGELOG.md (commit 11e0043, v0.1.6 and v0.1.8)
- ~/code/CloakBrowser-Manager/BINARY-LICENSE.md (commit 11e0043, version 1.3 of July 2026, Data Collection section)
