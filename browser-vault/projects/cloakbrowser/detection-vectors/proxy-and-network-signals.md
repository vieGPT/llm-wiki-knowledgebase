---
title: >-
    Proxy and network signals
category: detection-vectors
tags: [proxy, networking, ip-reputation, detection]
aliases: [IP reputation, proxy detection]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/webrtc-ip-leak]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/js/src/proxy.ts
summary: >-
    What detectors learn from the network path: exit-IP reputation, proxy timing and header traces, HTTP/2 first-visit challenges, and side channels that bypass the proxy.
provenance:
  extracted: 0.7
  inferred: 0.25
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Proxy and network signals

Anti-bot systems look at how a request arrived, not only at what the browser reports. The CloakBrowser README states plainly that "most blocks come from missing one of these three things, not from browser fingerprint detection": a residential proxy, GeoIP-matched timezone/locale, and headed mode. This page covers the network half of that: IP reputation, traces a proxy leaves, and paths that bypass the proxy.

## Signals

- **IP reputation**: datacenter IP ranges are blocked "by reputation alone", and FingerprintJS's "bad bot" verdict can come from poor proxy IP reputation. reCAPTCHA flags datacenter IPs by reputation, not fingerprint. No browser patch can change this.
- **Connection timing**: through a proxy, DNS/connect/TLS phases happen at the proxy, which can show up as anomalous Resource/Navigation Timing values. ^[inferred] (the README only states the countermeasure).
- **Proxy-added headers**: proxy cache headers and `Proxy-Connection` reveal an HTTP proxy hop. ^[inferred]
- **HTTP/2 first-visit challenge**: some sites challenge cookieless first visits over HTTP/2; the README says this affects all Chromium browsers. Some proxies also misbehave with HTTP CONNECT plus HTTP/2. Exactly what the site measures is not stated. ^[ambiguous]
- **Proxy-blocking providers**: some sites and providers "block every proxy you try", which motivates transparent proxy mode. How they detect the proxy is not documented. ^[ambiguous]
- **Request rhythm**: back-to-back `grecaptcha.execute()` calls and short visits score lower; the README advises 15+ seconds on a page and 30+ seconds between reCAPTCHA pages. This is behavioral rather than network-level. ^[inferred]

## Side channels that bypass the proxy

- **WebRTC** ICE candidates can reveal the real IP: see [[projects/cloakbrowser/detection-vectors/webrtc-ip-leak|WebRTC IP leak]].
- **Playwright request client**: with the proxy passed as `--proxy-server`, `context.request` / `page.request` / `route.fetch()` send from the real IP (#579). The wrapper warns on first use. This leak is to the target server when a script uses those APIs, not something a page can trigger. ^[inferred]
- **DNS**: outside transparent-proxy mode, DNS may be resolved locally. In transparent mode the default is Cloudflare DoH through the proxy.
- **GeoIP/IP mismatch**: an exit IP in one country with a UTC clock and `en-US` locale is a bot signal; see [[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency|Timezone/locale/IP consistency]].

## Countermeasures in CloakBrowser

| Signal | Technique |
|---|---|
| Timing, proxy headers, proxy blocking | [[projects/cloakbrowser/techniques/proxy-signal-removal\|Proxy signal removal]] |
| HTTP/2 first-visit challenge | [[projects/cloakbrowser/techniques/persistent-profile-warmup\|Persistent profile warm-up]] (`--disable-http2` once), or SOCKS5 |
| WebRTC IP | [[projects/cloakbrowser/techniques/webrtc-ip-spoofing\|WebRTC IP spoofing]] |
| Timezone/locale vs IP | [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching\|GeoIP timezone/locale matching]] |
| IP reputation | none in the product: bring a residential proxy |

## Related

- [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|Per-profile transparent proxy (Manager)]] — transparent mode as a per-profile option in CloakBrowser-Manager.
- [[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage|Cross-profile linkage (Manager)]] — a shared exit IP is one of the signals that links accounts across profiles.

## Sources

- ~/code/CloakBrowser/README.md (commit b1a00ed)
- ~/code/CloakBrowser/js/src/proxy.ts (commit b1a00ed)
