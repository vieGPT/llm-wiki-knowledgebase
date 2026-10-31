---
title: >-
    TLS and network fingerprinting
category: detection-vectors
tags: [tls, ja3, http2, network-fingerprinting, bot-detection]
aliases: [JA3, JA4, ja3n, Akamai HTTP/2 fingerprint, TLS fingerprint]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
summary: >-
    Servers fingerprint the TLS ClientHello (JA3/JA4) and HTTP/2 frames (Akamai) before any JS runs; CloakBrowser matches Chrome by running Chromium's own network stack, not by spoofing.
provenance:
  extracted: 0.58
  inferred: 0.32
  ambiguous: 0.10
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:54:19+01:00
updated: 2026-10-31T05:54:19+01:00
---

# TLS and network fingerprinting

Network fingerprinting classifies a client from its connection before any page script runs. The TLS ClientHello (cipher suites, extensions and their order, supported groups) hashes to a **JA3** or **JA4** fingerprint; **ja3n** is a normalized JA3 that ignores the extension order, which Chrome randomizes. The HTTP/2 connection preface (SETTINGS values, WINDOW_UPDATE, priority and pseudo-header order) gives the **Akamai** HTTP/2 fingerprint. ^[inferred] A server compares these with the browser the user agent claims to be. An HTTP library, or a browser whose TLS stack disagrees with its UA, stands out at once. ^[inferred] The JavaScript-level counterpart is [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]]; proxy-path signals are on [[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|Proxy and network signals]].

## CloakBrowser's position

- The README results table (as of 26/10/2026, commit b1a00ed) lists the TLS fingerprint as "Identical to Chrome" with "ja3n/ja4/akamai match", against "Mismatch" for stock Playwright.
- The reason is structural, not a spoof: CloakBrowser is Chromium with patches to fingerprint surfaces, so the TLS and HTTP/2 handshakes are Chromium's own. ^[inferred] The FAQ makes the same point against Firefox-based Camoufox: patching Chromium brings "TLS fingerprints that match real Chrome". See [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]].
- Brand spoofing (`--fingerprint-brand=Edge|Opera|Vivaldi`) stays coherent at this layer, because those browsers are Chromium-based and share its handshake. ^[inferred]
- The README does not explain why stock Playwright counts as a "Mismatch" when it also drives Chromium. One possibility is a `HeadlessChrome` UA or a Chromium-for-testing build that does not match the version the detector expects. ^[ambiguous]

## Network-layer side effects CloakBrowser does change

- **HTTP/2 first-visit challenges.** Some sites challenge cookie-less first visits over HTTP/2 ("affects all Chromium browsers"). The README workaround is one warm-up run with `--disable-http2`, which forces HTTP/1.1 and so presents a completely different protocol fingerprint, then later runs on HTTP/2 with the saved cookies. See [[projects/cloakbrowser/techniques/persistent-profile-warmup|Persistent profile warm-up]].
- **Proxies and HTTP/2.** SOCKS5 is recommended because it tunnels raw TCP and avoids HTTP CONNECT problems that some proxies have with HTTP/2. QUIC/HTTP3 is tunnelled over SOCKS5 with UDP ASSOCIATE. See [[projects/cloakbrowser/subsystems/proxy-handling|Proxy handling]].
- **Timing.** The binary zeroes the DNS, connect and SSL timing phases, which would otherwise show that the handshake happened at a proxy. See [[projects/cloakbrowser/techniques/proxy-signal-removal|Proxy signal removal]].
- **Round-trip time.** The Python stealth suite allow-lists incolumitas's `connectionRTT` failure as detecting "datacenter/proxy network latency, not browser". Latency is a network property that no browser patch can fix; only proxy choice can. ^[inferred]

## Not pinned by tests

No test in the suite measures JA3, JA4 or the Akamai fingerprint directly. The "Identical to Chrome" claim rests on the README alone. The live services the suite does check (FingerprintJS, BrowserScan, reCAPTCHA) may use network fingerprints internally, but the tests only read their verdicts. ^[inferred]

## Open Questions

- Which tool and reference Chrome version produced the "ja3n/ja4/akamai match" result is not stated.
- Whether `--fingerprint-transparent-proxy` mode changes anything visible at the TLS layer to the destination is not documented.

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/README.md (Test Results, FAQ, Troubleshooting: HTTP/2 and SOCKS5, Latest)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
