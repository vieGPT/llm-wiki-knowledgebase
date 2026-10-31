---
title: >-
    WebRTC IP leak
category: detection-vectors
tags: [webrtc, proxy, networking, detection]
aliases: [ICE candidate leak, STUN leak]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
summary: >-
    A page can open an RTCPeerConnection and read ICE candidates, exposing an IP that differs from the proxy exit IP the HTTP request came from.
provenance:
  extracted: 0.45
  inferred: 0.5
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T10:47:25+01:00
---

# WebRTC IP leak

WebRTC lets a page gather network candidates for peer-to-peer connections without any permission prompt. Those ICE candidates contain IP addresses discovered from local interfaces and from STUN servers. ^[inferred] When traffic goes through an HTTP or SOCKS proxy, the candidates can carry an address that is not the proxy's exit IP. A detector compares the two: a mismatch means "proxied browser", and the leaked address may be the operator's real IP. ^[inferred] The README lists "WebRTC IP leaks" as the problem `--fingerprint-webrtc-ip` prevents.

## What the detecting side measures

- **Public IP from STUN** (server-reflexive candidates) vs the request IP seen by the server. ^[inferred]
- **Local addresses** in host candidates, which modern Chromium usually masks with mDNS names. Whether CloakBrowser touches these is not documented. ^[ambiguous]
- **Consistency with other signals**: an IP from WebRTC can also be geolocated and compared against timezone and locale, so it feeds the same check as [[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency|Timezone/locale/IP consistency]]. ^[inferred]
- **Disabled WebRTC**: blocking WebRTC outright (as some privacy extensions do) is itself an unusual fingerprint for a desktop Chrome. ^[inferred] This explains why CloakBrowser rewrites candidates rather than disabling WebRTC.

## Countermeasure

[[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]]: the binary replaces the IP in ICE candidates with the value of `--fingerprint-webrtc-ip`, which the wrapper sets to the proxy exit IP (automatically with `geoip: true`, or via `auto`).

## Related

- [[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|Proxy and network signals]] covers the HTTP-side traces of a proxy.
- [[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale|Per-profile network and locale (Manager)]] — GeoIP is on by default for new CloakBrowser-Manager profiles.

## Sources

- ~/code/CloakBrowser/README.md (commit b1a00ed)
