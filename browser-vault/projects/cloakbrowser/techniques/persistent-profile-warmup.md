---
title: >-
    Persistent profile warm-up
category: techniques
tags: [persistent-profile, cookies, http2, incognito, visibility/internal]
aliases: [launchPersistentContext, cookie warmup]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/incognito-storage-quota-detection]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/proxy-and-network-signals]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/profile-seed]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/widevine-cdm-seeding]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/profile-seed.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/widevine.ts
  - repo:github.com/CloakHQ/CloakBrowser/examples/persistent_context.py
summary: >-
    Using a real user-data directory instead of an incognito context: escapes incognito checks, keeps a stable seed, and lets a one-time HTTP/1.1 visit warm cookies for later runs.
provenance:
  extracted: 0.76
  inferred: 0.22
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T06:46:59+01:00
---

# Persistent profile warm-up

By default `launch()` opens an incognito-style context with an empty, throwaway profile. `launchPersistentContext({ userDataDir })` (Python `launch_persistent_context`) uses a real on-disk profile instead. That defeats detectors that score ephemeral or private sessions, keeps cookies and storage between runs, and lets a profile be "warmed up" once and reused. The persistent path also pins the fingerprint seed ([[projects/cloakbrowser/subsystems/profile-seed|Profile seed persistence]]), so the profile is one consistent device over time.

## What a persistent profile changes

- **Not incognito**: services such as BrowserScan penalize incognito (−10% in its score, README). A real profile removes that signal. If the site still flags incognito, raise `--fingerprint-storage-quota` (for example `5000` MB), because the binary's default normalized quota reads as incognito to quota-based detectors. See [[projects/cloakbrowser/detection-vectors/incognito-storage-quota-detection|Incognito and storage-quota detection]].
- **Accumulated history**: cached fonts, service workers and IndexedDB build up over time, making the profile look lived-in (README). ^[inferred] that this improves scoring beyond the incognito check; the README asserts it without measurements.
- **Extensions** only load from a real user-data dir (`extensionPaths`).
- **Widevine**: with a sideloaded CDM, the wrapper seeds the CDM hint file into the profile before launch on Linux, so DRM works on the first run ([[projects/cloakbrowser/subsystems/widevine-cdm-seeding|Widevine CDM seeding]]).
- **Portable cookies**: `--fingerprint-portable-cookies` (Chromium 151+) encrypts cookies with a machine-independent key so a copied profile keeps logins; it must be on from the first launch.

## The HTTP/2 cookie warm-up recipe

Some sites challenge first-time visitors without cookies over HTTP/2; the README says this affects all Chromium browsers. The documented workaround:

1. First run: `launchPersistentContext({ userDataDir, args: ['--disable-http2'] })`, visit the site once over HTTP/1.1 to obtain cookies, close.
2. Later runs: same `userDataDir` without `--disable-http2`. Saved cookies let the site pass on HTTP/2.

- For stateless use, `--disable-http2` on every launch forces HTTP/1.1; the README says to use it only for sites that need it.
- A SOCKS5 proxy is the alternative: it bypasses HTTP CONNECT entirely.

## Wrapper mechanics

- Playwright: a direct `chromium.launchPersistentContext(userDataDir, ...)` call with the same stealth args, proxy routing and context options as `launchContext()`; `locale`/`timezone` go through binary flags, never Playwright context options.
- Puppeteer: `launch({ userDataDir })`; returns a `Browser`, not a context.
- A persistent context arrives with a page already open. The license guard is installed on those existing pages too, so a post-handshake license denial surfaces clearly on the first navigation of `pages()[0]`. See [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].
- Humanize is patched before the license guard on this path; the reverse order would make the guard's own-property wrappers shadow the humanized prototype methods and silently disable humanize.

## Python example (`examples/persistent_context.py`, B2)

The Python example shows the round trip at its simplest. Session 1 opens `./my-profile` headed, sets a cookie and a `localStorage` key on example.com, and closes. Session 2 reopens the same directory and reads both values back. It demonstrates persistence only: there is no seed pinning, warm-up or detector check. Writing the state through `page.evaluate()` is fine for a demo, but on a protected site the cookies would come from real navigation instead. ^[inferred] See [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]].

## Alternative without a profile folder

`launchContext({ contextOptions: { storageState } })` restores cookies/localStorage from a JSON file and saves them back later. It does not escape incognito detection. ^[inferred]

## Related

- [[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting|TLS and network fingerprinting]] — the `--disable-http2` warm-up run presents an HTTP/1.1 protocol fingerprint instead of Chrome's HTTP/2 one.
- [[projects/cloakbrowser-manager/techniques/portable-cookies|Portable cookies (Manager)]] — CloakBrowser-Manager turns `--fingerprint-portable-cookies` into a per-profile setting, off by default.
- [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|Per-profile identity isolation (Manager)]] — the Manager gives every profile its own persistent user-data directory.

## Sources

- repo:github.com/CloakHQ/CloakBrowser/README.md (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/profile-seed.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/widevine.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/examples/persistent_context.py (commit b1a00ed, B2)
