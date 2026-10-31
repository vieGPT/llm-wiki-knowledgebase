---
title: >-
    Per-profile transparent proxy
category: techniques
tags: [proxy, networking, profile-manager, compatibility, launch-flags, visibility/internal]
aliases: [transparent proxy connection, transparent_proxy]
relationships:
  - target: "[[projects/cloakbrowser/techniques/proxy-signal-removal]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: mitigates
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]"
    type: extends
  - target: "[[projects/cloakbrowser/techniques/geoip-timezone-locale-matching]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/webrtc-ip-spoofing]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/database.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/models.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileForm.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_database.py
summary: >-
    v0.1.8 per-profile switch for CloakBrowser's --fingerprint-transparent-proxy; the Manager refuses launch (400) unless the proxy is one http:// URL with username and password.
provenance:
  extracted: 0.72
  inferred: 0.2
  ambiguous: 0.08
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:32:16+01:00
updated: 2026-10-31T06:53:05+01:00
---

# Per-profile transparent proxy

Since v0.1.8 (14/10/2026) each Manager profile can turn on CloakBrowser's **transparent proxy mode**, "the browser's stronger proxy connection mode" (CHANGELOG). The mode itself is a binary feature, `--fingerprint-transparent-proxy`, documented on the CloakBrowser side as "a stronger way of connecting through your `--proxy-server`, for sites and providers that block every proxy you try" ([[projects/cloakbrowser/techniques/proxy-signal-removal|proxy signal removal]]). The Manager adds a per-profile switch, persistence, and a fail-fast guard so a profile never starts with a dead network. It is an option on top of the profile's normal proxy settings ([[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale|per-profile network and locale]]).

## How it works (commit `11e0043`)

1. **Setting**: a `transparent_proxy` boolean column (default `0`) in the profile table, mirrored in the API models; existing profiles were migrated with it off. In the form it is the "Transparent proxy connection" checkbox in the Compatibility section, with the hint "Requires an http:// proxy with a username and password".
2. **Guard at launch**: after the proxy string is normalized and validated, `_validate_transparent_proxy` accepts only scheme `http` with both a username and a password, and no `:` inside the decoded username. Anything else (no proxy, `https://`, `socks5://`, missing password, a username with an encoded colon) raises `ValueError`, which the API turns into **HTTP 400** with "Transparent proxy needs an http:// proxy with a username and password". The reason in code: the binary "fails closed (dead network, or exit on no proxy)" unless it gets exactly such a proxy, so refusing the launch is clearer than a browser with no network.
3. **Flag**: `_build_fingerprint_args` appends `--fingerprint-transparent-proxy` when the setting is on; the proxy itself still travels through the wrapper's normal proxy path ([[projects/cloakbrowser/subsystems/proxy-handling|proxy handling]]) and the flag rides in `args` of the [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|launcher]].
4. **Duplication** copies the setting with every other field ([[projects/cloakbrowser-manager/techniques/profile-duplication-with-state|profile duplication]]).

Tests pin the guard (accepts `http://user:pass@host:port`; rejects `None`, `socks5://`, `https://`, a password-less user, no credentials, and `us%3Aer`), the flag mapping, the default-off migration and the API round trip.

## What it counters

Transparent mode targets the network-path half of detection: sites and providers that detect and block proxied traffic ([[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|proxy and network signals]], "proxy-blocking providers"). In transparent mode the binary also resolves hostnames by default through DNS-over-HTTPS sent via the proxy, so DNS no longer leaves from the local network (binary 155.0.8059.39.1+, CloakBrowser docs). ^[inferred] What the mode changes on the wire is not documented in either repository, so how well it works against a given provider is unknown. ^[ambiguous]

It does not change IP reputation, and it does not affect WebRTC or timezone alignment, which stay with [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP]] and [[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC spoofing]]. ^[inferred]

## Why the credential rule

- The binary needs credentials it can use itself. Inline `user:pass` in an `http://` URL is the form the wrapper already passes to Chrome as `--proxy-server` for credentialed proxies. ^[inferred]
- A `:` in the username would make `user:pass` ambiguous once the credentials are split again downstream, hence the explicit check on the decoded username. ^[inferred]
- Rejecting SOCKS5 is a binary constraint, not a Manager choice. It means a profile cannot combine transparent mode with the SOCKS5 route that CloakBrowser recommends for HTTP/2 CONNECT problems. ^[inferred]

## Open Questions

- The bundled engine version that first supports the flag is not named in the Manager (CloakBrowser docs say Chromium 151+); v0.1.8 bundles CloakBrowser 0.6.3. ^[ambiguous]
- The Manager does not expose `--fingerprint-transparent-proxy-dns`; it can only be set through the Advanced launch arguments box. ^[inferred]

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager (commit `11e0043`): `CHANGELOG.md` (0.1.8), `backend/browser_manager.py` (`_validate_transparent_proxy`, launch path, `_build_fingerprint_args`), `backend/database.py` (column, default), `backend/models.py`, `frontend/src/components/ProfileForm.tsx` (Compatibility section); tests `backend/tests/test_browser_manager.py`, `test_api.py`, `test_database.py`
