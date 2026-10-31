---
title: >-
    CDP traffic minimization
category: techniques
tags: [cdp, recaptcha, playwright, bot-detection, browser-automation, visibility/internal]
aliases: [native sleep instead of waitForTimeout, fewer evaluate calls, quiet CDP]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends]]"
    type: related_to
  - target: "[[projects/cloakbrowser/architecture/framework-integration-modes]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/examples/stealth_test.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/recaptcha_score.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/langchain_loader.py
summary: >-
    Usage discipline that keeps Chrome DevTools Protocol chatter low before scoring scripts run: native sleep over waitForTimeout, few evaluate calls, and Playwright rather than Puppeteer.
provenance:
  extracted: 0.62
  inferred: 0.32
  ambiguous: 0.06
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:52:24+01:00
updated: 2026-10-31T06:37:29+01:00
---

# CDP traffic minimization

CDP traffic minimization is the only CloakBrowser technique that lives in the user's script rather than in the binary or the wrapper. The patched binary removes static automation markers, but every Chrome DevTools Protocol command a script sends still runs inside the page's browser process, and the README says reCAPTCHA detects some of that activity. The guidance is to send as few CDP commands as possible, especially before a scoring script such as `grecaptcha.execute()` fires. It complements the binary patches described in [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP and automation detection]] and the wrapper's [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]].

## Rules (README troubleshooting, as of 26/10/2026, commit b1a00ed)

| Rule | README reasoning |
|---|---|
| Use native sleep (`time.sleep`, a `setTimeout` promise) instead of `page.wait_for_timeout()` / `page.waitForTimeout()` | the framework wait "sends CDP protocol commands that reCAPTCHA detects"; a native sleep is "invisible to the browser" |
| Minimize `page.evaluate()` calls before the reCAPTCHA check | "each one sends CDP traffic" |
| Use Playwright, not Puppeteer | Puppeteer "sends more CDP protocol traffic" and "leaks automation signals that reCAPTCHA Enterprise can detect, causing intermittent 403 errors"; see [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends\|Playwright vs Puppeteer backends]] |
| Set timezone and locale with binary flags, not CDP emulation | the `launch()` docs call the binary-flag path "no detectable CDP emulation"; see [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching\|GeoIP timezone/locale matching]] |
| Prefer `type()` over `fill()` | behavioral, not CDP volume, but listed in the same reCAPTCHA tips; see [[projects/cloakbrowser/detection-vectors/behavioral-analysis\|Behavioral analysis]] |

## Why it plausibly works

- A script-side wait in Playwright or Puppeteer is a protocol round trip (polling or a timer evaluated in the page), while a host-language sleep sends nothing to the browser at all. ^[inferred]
- `Runtime.evaluate` creates script executions in the page's main world. Even without leaked globals, detectors can observe side effects such as timing, stack frames or extra microtasks. The humanize engine moves its own reads into an isolated world for the same reason. ^[inferred]
- How exactly reCAPTCHA measures CDP activity is not documented in the repository; the claim rests on the vendor's testing. ^[ambiguous]

## How the repository's own scripts follow it

- `examples/stealth_test.py` waits with `time.sleep()` between navigations (3 to 15 s) and never calls `wait_for_timeout`. It still uses `page.evaluate()` to scrape each detector's verdict, but only after the detector has finished running. ^[inferred] The JS mirror `js/examples/stealth-test.ts` uses `page.waitForTimeout()` instead; see [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]].
- `examples/recaptcha_score.py` waits for the result with `page.wait_for_function(...)` on the response text. Its comment explains that the score renders after an asynchronous token and backend-verify round trip that can finish after `networkidle`, so it waits for the actual text. `wait_for_function` polls through CDP, but only after the score request has already been sent. ^[inferred]
- `examples/integrations/langchain_loader.py` reads page text with `page.evaluate("document.body.innerText")` once per URL. That is fine for unprotected pages, but on a reCAPTCHA-protected page it is exactly the pattern the README advises keeping to a minimum. ^[inferred]

## Limits

- The technique only controls the user's script. A framework connected over CDP in [[projects/cloakbrowser/architecture/framework-integration-modes|Mode 2]] issues its own protocol commands (Crawl4AI, browser-use, Scrapling), which CloakBrowser cannot throttle. ^[inferred]
- The README's guidance is reCAPTCHA-specific. Whether other vendors (Turnstile, DataDome, Kasada) score CDP volume is not stated. ^[ambiguous]

## Open Questions

- Which CDP commands are actually observable from page JavaScript, and whether the "CDP input behavior" binary patches reduce that surface, is not visible in the wrapper code.

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/README.md (Troubleshooting: reCAPTCHA v3 scores are low; Puppeteer note; `launch()` timezone/locale)
- repo:github.com/CloakHQ/CloakBrowser/examples/stealth_test.py
- repo:github.com/CloakHQ/CloakBrowser/examples/recaptcha_score.py
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/langchain_loader.py
