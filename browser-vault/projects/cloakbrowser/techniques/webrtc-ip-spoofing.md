---
title: >-
    WebRTC IP spoofing
category: techniques
tags: [webrtc, proxy, networking, stealth, visibility/internal]
aliases: [--fingerprint-webrtc-ip]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/webrtc-ip-leak]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: mitigates
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/js/src/playwright.ts
  - ~/code/CloakBrowser/js/src/puppeteer.ts
  - ~/code/CloakBrowser/js/src/geoip.ts
  - ~/code/CloakBrowser/js/tests/config.test.ts
summary: >-
    The binary rewrites WebRTC ICE candidate IPs to the proxy exit IP via --fingerprint-webrtc-ip; the wrapper resolves the IP from the proxy or reuses the GeoIP lookup.
provenance:
  extracted: 0.75
  inferred: 0.2
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T10:47:25+01:00
---

# WebRTC IP spoofing

WebRTC gathers ICE candidates from the machine's network interfaces and STUN servers, which can reveal an IP other than the proxy's (see [[projects/cloakbrowser/detection-vectors/webrtc-ip-leak|WebRTC IP leak]]). CloakBrowser's binary accepts `--fingerprint-webrtc-ip=<ip>` and replaces the IP in ICE candidates with that value. The wrapper's part is choosing the IP: it makes the WebRTC-visible address equal the proxy exit IP, so WebRTC, HTTP and GeoIP-derived timezone/locale all tell one story. ^[inferred]

## Three ways to set it

| Usage | What happens | Network cost |
|---|---|---|
| `geoip: true` with a proxy | The exit IP found during GeoIP resolution is appended as `--fingerprint-webrtc-ip=<exitIp>` | None extra (reuses the GeoIP lookup) |
| `args: ['--fingerprint-webrtc-ip=auto']` with a proxy | Wrapper resolves the exit IP via HTTP through the proxy and substitutes it | One exit-IP request (README names ipify.org, checkip.amazonaws.com) |
| `args: ['--fingerprint-webrtc-ip=1.2.3.4']` | Passed through unchanged | None |

- An explicit user value always wins: `appendWebrtcExitIp` only appends when no `--fingerprint-webrtc-ip` arg is present.
- `auto` without a proxy is removed with a warning. If exit-IP resolution fails, the flag is also removed with a warning, so the launch continues with no WebRTC override. ^[ambiguous] (the repository does not say what the binary then reports).
- `buildArgs` passes the flag through from `args` and never injects it on its own (`config.test.ts`, "buildArgs webrtc IP").

## Wiring in the launch functions

- Playwright `launch()`, `buildLaunchOptions()`, `launchPersistentContext()` and the Puppeteer launchers all call `resolveWebrtcArgs` then `appendWebrtcExitIp` before `buildArgs`.
- `launchContext()` resolves GeoIP and WebRTC once and then calls the inner `launch()` with `geoip: false`, so the exit IP is not fetched twice.
- The flag is process-wide in the binary, so it applies to every context of the browser. ^[inferred]

## Why it works

Detection pages compare the IP seen in ICE candidates (via `RTCPeerConnection`) against the IP the HTTP request came from. Rewriting candidates inside Chromium means no JavaScript shim exists for a page to detect, unlike extension-based WebRTC blockers that disable or wrap `RTCPeerConnection`. ^[inferred] See [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]].

## Related

- [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]] shares the exit-IP lookup.
- [[projects/cloakbrowser/techniques/proxy-signal-removal|Proxy signal removal]] covers the HTTP-side proxy traces.
- [[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency|Timezone/locale/IP consistency]] compares the WebRTC IP with the exit IP's geolocation, timezone and language.
- [[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|Proxy and network signals]] lists WebRTC among the side channels that bypass the proxy.

## Open Questions

- Does the binary also suppress local/private (host) candidates, or only rewrite the public IP? Not stated in the repository.
- What WebRTC reports when `auto` resolution fails and the flag is dropped (real IP vs binary default)?

## Sources

- ~/code/CloakBrowser/README.md (commit b1a00ed)
- ~/code/CloakBrowser/js/src/playwright.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/puppeteer.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/geoip.ts (commit b1a00ed, `resolveWebrtcArgs` / `appendWebrtcExitIp` only)
- ~/code/CloakBrowser/js/tests/config.test.ts (commit b1a00ed)
