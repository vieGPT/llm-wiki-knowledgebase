---
title: >-
    Portable cookies
category: techniques
tags: [cookies, session-persistence, profile-manager, migration, launch-flags]
aliases: [portable_cookies, move a profile to another machine]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/persistent-profile-warmup]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/CHANGELOG.md
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/database.py
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/frontend/src/components/ProfileForm.tsx
  - ~/code/CloakBrowser-Manager/backend/tests/test_browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_api.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_database.py
summary: >-
    v0.1.8 per-profile switch for --fingerprint-portable-cookies so a profile can move to another machine with its logins; only cookies saved while it is on are portable.
provenance:
  extracted: 0.7
  inferred: 0.25
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:32:16+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Portable cookies

Since v0.1.8 (14/10/2026) a Manager profile can turn on **portable cookies**, which "lets a profile move to another machine with its logins" (CHANGELOG). The mechanism is CloakBrowser's binary flag `--fingerprint-portable-cookies`, which encrypts cookies with a machine-independent key so a copied profile keeps its logins and must be on from the first launch ([[projects/cloakbrowser/techniques/persistent-profile-warmup|persistent profile warmup]]). The Manager contributes the per-profile switch, its default (off) and the warning to enable it before logging in. It is a state-portability feature, not an anti-detection one: it keeps a warmed, logged-in identity usable after a host move. ^[inferred]

## How it works (commit `11e0043`)

1. **Setting**: a `portable_cookies` boolean column (default `0`) in the profile table and API models. Existing profiles were migrated with it off. In the form it is a Compatibility checkbox: "Only cookies saved while this is on are portable, so turn it on before logging in."
2. **Flag**: `_build_fingerprint_args` appends `--fingerprint-portable-cookies` when it is on; the flag travels in `args` through the wrapper's [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|launcher]]. The Manager applies no other logic and no validation.
3. **Duplication** copies the setting along with the rest of the profile's configuration (CHANGELOG 0.1.8; `duplicate_profile` copies every column except identity fields).

Tests pin the flag mapping (on adds the flag, off or `0` omits it), the default-off migration, and that updating one Compatibility flag leaves the other unchanged.

## Relation to duplication and moving profiles

- The Manager has **no export or import** of profiles (no such route at commit `11e0043`). Moving a profile to another machine means copying its user-data directory and recreating the profile row by hand. ^[inferred]
- **Duplicate with browser state** copies the user-data directory on the same host, `Cookies` and `Local State` included ([[projects/cloakbrowser-manager/techniques/profile-duplication-with-state|profile duplication with state]]). Whether copied cookies stay readable without portable mode is up to the engine and the host's cookie-encryption key. ^[ambiguous]
- Switching the setting on later does not convert existing cookies; logins made before it was on stay machine-bound (UI text). ^[inferred]

## Detection relevance

Portable cookies does not counter a detection vector. It preserves the cookie history that makes a persistent profile look like a returning visitor, which [[projects/cloakbrowser/techniques/persistent-profile-warmup|persistent profile warmup]] relies on. ^[inferred] The same property is a linkage risk: logins that travel with a copied profile can end up in two places at once. The Manager's [[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage|cross-profile linkage]] page lists it under residual risk. ^[inferred]

## Open Questions

- Whether a moved profile should keep its seed when the new host has a different OS ([[projects/cloakbrowser/techniques/platform-persona-spoofing|persona]] is chosen from the host runtime, not the profile) is not addressed. A Windows-persona profile moved to a Mac would reappear as a macOS device with the same cookies. ^[inferred]

## Sources

- ~/code/CloakBrowser-Manager (commit `11e0043`): `CHANGELOG.md` (0.1.8), `backend/browser_manager.py` (`_build_fingerprint_args`), `backend/database.py` (column, default, `duplicate_profile`), `backend/main.py` (duplicate route, `_copy_browser_state`), `frontend/src/components/ProfileForm.tsx` (Compatibility section); tests `backend/tests/test_browser_manager.py`, `test_api.py`, `test_database.py`
