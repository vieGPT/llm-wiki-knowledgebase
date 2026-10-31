---
title: >-
    Timezone/locale/IP consistency
category: detection-vectors
tags: [timezone, locale, geoip, network-identity, detection]
aliases: [timezone mismatch, UTC en-US bot signal]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/webrtc-ip-leak]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/cloakbrowser/geoip.py
  - ~/code/CloakBrowser/cloakbrowser/browser.py
summary: >-
    Detectors compare the IP's geolocation with the browser's timezone, language and WebRTC IP; a proxied browser on the host's UTC clock and en-US locale contradicts itself.
provenance:
  extracted: 0.62
  inferred: 0.34
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:49:08+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Timezone/locale/IP consistency

This detection vector does not look at any single browser property. It cross-checks what the network says about the visitor against what the browser says about itself. The server knows the connecting IP and can geolocate it; the page can read the browser's timezone (`Intl.DateTimeFormat().resolvedOptions().timeZone`, `Date` offsets), its languages (`navigator.language(s)`, `Accept-Language`), and, through WebRTC ICE candidates, any other public IP the machine has. When these disagree, the visitor is probably behind a proxy and probably automated. ^[inferred]

## What gets compared

| Network side | Browser side | Typical inconsistency |
|---|---|---|
| IP geolocation (country, timezone) | Timezone reported by JS and date formatting | Exit in Germany, clock on `UTC` or the host's zone |
| IP country | `navigator.language(s)`, `Accept-Language`, `Intl` locale | Exit in Japan, `en-US` only |
| Connecting IP | WebRTC STUN/ICE candidate IPs | WebRTC shows the host's real IP or a different exit (see [[projects/cloakbrowser/detection-vectors/webrtc-ip-leak\|WebRTC IP leak]]) |
| How values were set | Presence of CDP emulation, per-worker or per-context disagreement | Timezone overridden in one context but not in workers ^[inferred] |

## Why automation trips it

- Containers and cloud VMs default to `UTC` and `en-US`. Behind a residential proxy, that pairing is the cheapest mismatch to find; the README calls "UTC + en-US" a bot signal and says most blocks come from missing a residential IP, geoip matching or headed mode rather than from the browser fingerprint.
- Emulating timezone through CDP (`Emulation.setTimezoneOverride`, Playwright's `timezone_id`) generates protocol traffic and can leave other realms on the real value, turning the fix into another tell. ^[inferred] See [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP automation detection]].
- A proxy gateway's hostname can geolocate differently from its actual exit; resolving the wrong one yields a confidently wrong timezone (CHANGELOG 0.5.1, 0.5.3).

## How CloakBrowser counters it

- [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]]: `geoip=True` resolves the exit IP through the proxy and sets timezone, language and WebRTC IP from it, via native binary flags (`--fingerprint-timezone`, `--lang`, `--fingerprint-locale`, `--fingerprint-webrtc-ip`), not CDP emulation.
- [[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]] replaces ICE candidate IPs with the exit IP.
- The resolver fails closed (0.5.10) rather than launching on the container clock.
- `cloakbrowser info --proxy <url>` lets users verify the resolved triple before launching (see [[projects/cloakbrowser/subsystems/cli|CLI]]).
- Binary-side network identity work: 150.0.7871.114.6 cites consistency for IPv4, IPv6, dual-stack, direct, proxied and automatically located sessions.

Implementation: [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]].

## Residual risks

- One locale per country does not model multilingual regions. ^[inferred]
- GeoLite2 accuracy limits: a city-level mismatch between CloakBrowser's database and a detector's commercial database can produce a different timezone near zone borders. ^[inferred]
- Explicit user values override geoip; a user who hardcodes `timezone=` and rotates proxies across regions recreates the mismatch. ^[inferred]
- Consistency does not fix reputation: datacenter IP ranges are blocked regardless (see [[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|Proxy and network signals]]).

## Open Questions

- Which detection services in the README's test matrix actually score timezone/IP mismatch is not broken out; see [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]].

## Related

- [[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale|Per-profile network and locale (Manager)]] — per-profile proxy, timezone, locale and GeoIP in CloakBrowser-Manager.

## Sources

- `~/code/CloakBrowser/README.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/geoip.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
