---
title: >-
    Per-profile network and locale
category: techniques
tags: [proxy, geoip, timezone, locale, profile-manager, visibility/internal]
aliases: [per-profile proxy, proxy test]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency]]"
    type: mitigates
  - target: "[[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/geoip-timezone-locale-matching]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/proxy-signal-removal]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/webrtc-ip-leak]]"
    type: mitigates
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/README.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileForm.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileList.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/models.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
summary: >-
    Each profile carries its own proxy, timezone, locale and GeoIP switch; the Manager validates and tests the proxy, then lets CloakBrowser align timezone and language with the exit IP.
provenance:
  extracted: 0.74
  inferred: 0.23
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:22:07+01:00
updated: 2026-10-31T06:53:05+01:00
---

# Per-profile network and locale

Every Manager profile has its own network identity: proxy, GeoIP, timezone and locale, with "timezone and language follow the proxy exit IP automatically" (README). The Manager stores and validates these fields and offers a proxy test; the actual alignment of timezone, language and [[projects/cloakbrowser/detection-vectors/webrtc-ip-leak|WebRTC IP]] with the exit IP is done by CloakBrowser's [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]] and binary flags. Distinct proxies per profile also keep accounts from sharing an IP, one of the signals behind [[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage|cross-profile linkage]]. ^[inferred]

## What the user can set (ProfileForm "Network" section, commit 11e0043)

- **Proxy**: one URL per profile (placeholder `http://user:pass@host:port`). The sidebar marks profiles that have one with a "Proxy" hint.
- **Test**: calls `/api/profiles/test-proxy` and shows the exit IP, city and country, and latency, or the error. Editing the proxy clears the old result.
- **Timezone** and **Locale**: free-text fields (placeholders `America/New_York`, `en-US`); empty means unset.
- **GeoIP** (on by default): "Auto-detect timezone/locale from the proxy exit, or host public IP without a proxy".
- **Transparent proxy** (Compatibility section, off by default): "Stronger proxy connection mode. Requires an http:// proxy with a username and password." See [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|Per-profile transparent proxy]].

## How the fields reach CloakBrowser

- At launch the backend normalizes and validates the proxy, checks the transparent-proxy requirement, and passes `proxy`, `timezone`, `locale` and `geoip` to the Python wrapper's persistent-context launcher; transparent mode becomes `--fingerprint-transparent-proxy` ([[projects/cloakbrowser/techniques/proxy-signal-removal|proxy signal removal]]).
- Timezone and locale go in as dedicated launch parameters. In the wrapper, dedicated values win field by field over GeoIP, which fills only what is missing ([[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]]). So with GeoIP on, a filled-in timezone stays fixed while an empty locale follows the exit IP. ^[inferred]
- Proxy string routing (Playwright option vs `--proxy-server`) is the wrapper's job, see [[projects/cloakbrowser/subsystems/proxy-handling|proxy handling]].
- The proxy test runs the same normalize/validate step as launch, then reuses CloakBrowser's GeoIP module: exit-IP discovery through the proxy and a lookup in the GeoLite2 database. The geo lookup is best-effort and never fails the test. Because it uses the same exit-IP path as GeoIP at launch, a passing test predicts what GeoIP will see, except with rotating proxies. ^[inferred]

## Backend validation details (commit 11e0043)

- **Proxy normalization**: `host:port:user:pass` becomes `http://user:pass@host:port`, bare `host:port` becomes `http://host:port`; strings already starting with `http://`, `https://` or `socks5://` pass through. The result must have scheme http/https/socks5, a host and a port, else the launch or test returns 400.
- **Transparent proxy gate** (v0.1.8): only a single `http://` proxy with both username and password (and no `:` inside the decoded username) is accepted. The binary's transparent mode fails closed (dead network, or exit without a proxy), so the Manager refuses the launch with HTTP 400 instead of starting a browser with no network (CHANGELOG 0.1.8).
- **Timezone and locale are not validated by the Manager**: the API models accept any string, and empty values become "unset" before they reach the wrapper. ^[inferred]
- **Proxy test errors** are coarse: "Could not connect through proxy" for connection failures, "Proxy did not return an exit IP" for timeouts or blocks; latency covers the exit-IP lookup only.
- The bundled engine 0.6.3 (v0.1.8) no longer fails a launch when the proxy exits in a country with no mapped language.
- Launch logs record the proxy with credentials stripped.

## Why it matters

Detectors compare the IP's location with the browser's timezone, language and WebRTC IP ([[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency|timezone/locale/IP consistency]]). Making these per-profile settings, with GeoIP on by default, means each identity is internally consistent with its own exit, not with the host. ^[inferred]

## Open Questions

- The form accepts any timezone or locale text, and the Manager backend does not validate it either (see above); any validation is the wrapper's or the binary's.
- GeoIP without a proxy sends the host's public IP to third-party echo services (documented on the CloakBrowser side); the Manager UI text says "host public IP" but does not mention the external lookup. ^[inferred]

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager (commit `11e0043`): `README.md` (Features), `frontend/src/components/ProfileForm.tsx` (Network and Compatibility sections), `frontend/src/components/ProfileList.tsx`, `backend/browser_manager.py` (launch options, `test_proxy`, `_normalize_proxy`, `_validate_transparent_proxy`), `backend/models.py`, `CHANGELOG.md` (0.1.8)
