---
title: >-
    GeoIP timezone/locale matching
category: techniques
tags: [geoip, timezone, locale, network-identity, evasion, visibility/internal]
aliases: [geoip=True, timezone matching]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/webrtc-ip-leak]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/webrtc-ip-spoofing]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: mitigates
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/cloakbrowser/geoip.py
  - ~/code/CloakBrowser/cloakbrowser/browser.py
summary: >-
    Make the browser's timezone, language and WebRTC IP agree with the proxy exit IP, set through native binary flags rather than detectable CDP emulation.
provenance:
  extracted: 0.72
  inferred: 0.25
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:48:42+01:00
updated: 2026-10-31T10:47:25+01:00
---

# GeoIP timezone/locale matching

When a browser runs behind a proxy, the site sees the proxy's exit IP but the browser reports the host's clock and language. The technique closes that gap: discover the exit IP, map it to a timezone and locale, and make the browser claim exactly those, with the WebRTC-visible IP set to the same exit IP. In CloakBrowser this is one flag, `geoip=True`, implemented by the [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]]. It counters [[projects/cloakbrowser/detection-vectors/timezone-locale-ip-consistency|Timezone/locale/IP consistency]] checks.

## Mechanism

1. Resolve the egress IP by asking echo services **through the proxy** (the real exit, not the gateway's DNS), or the machine's public IP when no proxy is set.
2. Map IP → IANA timezone and country → BCP 47 locale with a local GeoLite2 database (132 country mappings as of 26/10/2026, commit b1a00ed; unmapped countries fall back to `en-US`).
3. Apply via **binary flags**: `--fingerprint-timezone=<tz>`, `--lang=<locale>` and `--fingerprint-locale=<locale>`, plus `--fingerprint-webrtc-ip=<exit_ip>` as a free by-product.
4. Explicit `timezone=` / `locale=` parameters, or raw flags in `args`, always win; geoip only fills gaps.

## Why native flags instead of CDP emulation

Playwright's `timezone_id` / `locale` context options use CDP `Emulation.*` commands. That traffic is itself an automation signal, and per-context emulation leaves workers and other contexts on the real values. CloakBrowser moved timezone and locale to binary flags in 0.3.12 ("eliminates a detection vector"), backed by a native locale patch. `--fingerprint-timezone` is read from the command line in the renderer, so it applies process-wide to every context. ^[inferred] See [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP automation detection]].

There was one reversal: 0.3.4 briefly used Playwright's per-context timezone in `launch_context()` to fix a multi-context mismatch, before a binary fix ("timezone multi-context fix", 0.3.9) made the flag path reliable and the wrapper returned to flags only.

## Why it works

- A residential proxy in Vienna with a browser reporting `UTC` and `en-US` is the cheapest inconsistency a detector can find; README troubleshooting labels "UTC + en-US" a bot signal and lists geoip among the three settings that fix most blocks. ^[inferred]
- Reusing the same exit IP for WebRTC ensures STUN/ICE candidates do not reveal a different public IP from the one the HTTP connection shows, so all three signals (IP, clock, language) agree. See [[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]]. ^[inferred]

## Design choices and trade-offs

- **Fail closed** (0.5.10): if a requested resolution fails, the launch aborts instead of silently running on the container clock, trading availability for never launching an inconsistent identity. ^[inferred]
- **Proxy-free mode** (0.4.8): even without a proxy, bare Docker/cloud hosts default to UTC; resolving the host's own public IP fixes the clock/IP mismatch of datacenter machines.
- **Coarse locale**: one locale per country, so a Swiss exit always gets `de-CH`, a Belgian one `fr-BE`. Real users in multilingual countries vary; detectors that model per-region language distributions might notice. ^[inferred]
- **Third-party lookups**: the echo services (ipify, Amazon checkip, ifconfig.me) see every launch's exit IP.
- **Proxy-dict credential bug** (fixed 0.5.3, #469): dict-form HTTP credentials were dropped, so resolution silently used the proxy gateway's location. Silent fallbacks in this path produce a confident but wrong identity, which is worse than an error. ^[inferred]

## Related practice

The README's recommended anti-bot config combines a residential proxy, `geoip=True`, headed mode and `humanize=True`. Datacenter IPs fail on reputation alone regardless of timezone matching (see [[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|Proxy and network signals]]).

## Related

- [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]] — the `--lang` / `--fingerprint-locale` flags set the browser's language natively instead of through detectable CDP emulation.
- [[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale|Per-profile network and locale (Manager)]] — per-profile proxy, GeoIP switch and proxy test in CloakBrowser-Manager.

## Sources

- `~/code/CloakBrowser/README.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/geoip.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
