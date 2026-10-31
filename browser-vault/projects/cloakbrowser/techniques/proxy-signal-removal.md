---
title: >-
    Proxy signal removal
category: techniques
tags: [proxy, networking, stealth, chromium, visibility/internal]
aliases: [transparent proxy, proxy timing zeroing]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/source-level-chromium-patches]]"
    type: derived_from
  - target: "[[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/src/proxy.ts
summary: >-
    Binary-level patches that hide that traffic goes through a proxy: zeroed DNS/connect/SSL timing, stripped proxy headers, transparent proxy mode, and SOCKS5 routing.
provenance:
  extracted: 0.68
  inferred: 0.27
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T06:46:59+01:00
---

# Proxy signal removal

A proxy changes the exit IP but leaves traces a page can measure: connection-phase timings, proxy-added headers, and the shape of the tunnel itself. CloakBrowser removes these inside the compiled binary rather than in the wrapper, as part of its [[projects/cloakbrowser/techniques/source-level-chromium-patches|source-level Chromium patches]]. The wrapper's job is only to hand the proxy to Chromium natively (see [[projects/cloakbrowser/subsystems/proxy-handling|Proxy handling]]). This page collects what the README says the binary does; the C++ patches themselves are not in the repository.

## Mechanisms (binary)

- **Timing zeroing**: DNS, connect and SSL timing are zeroed (README "Latest", as of 26/10/2026, commit b1a00ed). Through a proxy, these phases happen at the proxy, so a real browser's Resource/Navigation Timing values would look anomalous (for example, a connect phase that reflects the proxy hop). Reporting zero matches a reused or pre-warmed connection. ^[inferred]
- **Header stripping**: proxy cache headers are stripped and the `Proxy-Connection` header leak is removed. Both are classic markers that a request passed through an HTTP proxy. ^[inferred]
- **Transparent proxy mode** (`--fingerprint-transparent-proxy`, Chromium 151+ binary): described only as "a stronger way of connecting through your `--proxy-server`, for sites and providers that block every proxy you try". Off by default; enabled alongside `--proxy-server`. ^[ambiguous]
- **Transparent-proxy DNS** (`--fingerprint-transparent-proxy-dns=<url>|local`, Chromium 155.0.8059.39.1+, Linux and Windows): in transparent mode, hostnames resolve by default through Cloudflare DNS-over-HTTPS sent through the proxy, so DNS queries do not leave from the local network. A custom DoH URL or `local` (old behaviour) can be passed.
- **License traffic routing**: `--license-route=auto|direct|proxy` keeps Pro license/session calls off the proxy by default (direct first, one retry through the proxy). This avoids spending proxy bandwidth and touching the scraping session. ^[inferred] See [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].

## Mechanisms (wrapper)

- **Native proxy path instead of CDP auth**: credentialed proxies go to Chrome as `--proxy-server` with inline credentials, not through Playwright's CDP `Fetch.authRequired` interceptor. This was introduced for reliability (#182). It also reduces CDP traffic during navigation. ^[inferred] See [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]].
- **SOCKS5 recommendation**: SOCKS5 tunnels raw TCP and bypasses HTTP CONNECT entirely, avoiding CONNECT problems some proxies have with HTTP/2. QUIC/HTTP3 tunnels through SOCKS5 via UDP ASSOCIATE.

## Limits

- None of this fixes IP reputation. The README repeatedly says datacenter IPs are blocked "by reputation alone" and recommends residential proxies.
- Playwright's request client (`context.request`, `route.fetch()`) does not use a `--proxy-server` proxy and sends from the real IP; the wrapper warns but does not reroute.
- WebRTC can still expose the real IP independently of the proxy: see [[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]].

## Open Questions

- What exactly transparent proxy mode changes on the wire is not documented in the repository. ^[ambiguous]
- Whether zeroed timings are themselves detectable (all-zero DNS/connect/SSL on a first visit to a new origin) is not addressed. ^[inferred]

## Related

- [[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting|TLS and network fingerprinting]] — lists the zeroed DNS/connect/SSL timing as a network-layer change; the TLS handshake itself is Chromium's own.
- [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|Per-profile transparent proxy (Manager)]] — CloakBrowser-Manager exposes transparent mode as a per-profile switch and refuses the launch unless the proxy is one `http://` URL with username and password.

## Sources

- repo:github.com/CloakHQ/CloakBrowser/README.md (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/proxy.ts (commit b1a00ed)
