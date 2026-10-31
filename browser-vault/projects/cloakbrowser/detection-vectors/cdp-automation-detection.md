---
title: >-
    CDP and automation detection
category: detection-vectors
tags: [cdp, bot-detection, playwright, browser-automation, isolated-world]
aliases: [isAutomatedWithCDP, CDP detection, automation signals]
sources:
  - ~/code/CloakBrowser/cloakbrowser/human/keyboard.py
  - ~/code/CloakBrowser/cloakbrowser/human/world.py
  - ~/code/CloakBrowser/cloakbrowser/human/injected.py
  - ~/code/CloakBrowser/cloakbrowser/human/patch.py
  - ~/code/CloakBrowser/examples/fingerprint_scan_test.py
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/examples/stealth_test.py
  - ~/code/CloakBrowser/examples/integrations/selenium_example.py
  - ~/code/CloakBrowser/examples/integrations/undetected_chromedriver.py
  - ~/code/CloakBrowser/images/browserscan_normal.png
  - ~/code/CloakBrowser/tests/test_stealth_reproduction_110.py
  - ~/code/CloakBrowser/tests/test_stealth_unit.py
  - ~/code/CloakBrowser/tests/test_stealth.py
  - ~/code/CloakBrowser/js/tests/stealth.test.ts
  - ~/code/CloakBrowser/js/tests/stealth.puppeteer.test.ts
  - ~/code/CloakBrowser/js/tests/humanizeEngine.test.ts
  - ~/code/CloakBrowser/js/tests/humanizeHarness.ts
summary: >-
    How pages detect a CDP-driven browser (untrusted events, evaluate stack frames, framework globals, protocol traffic) and what the humanize layer does to stay out of the page's realm.
provenance:
  extracted: 0.68
  inferred: 0.28
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:47:42+01:00
updated: 2026-10-31T10:47:25+01:00
---

# CDP and automation detection

Automation frameworks drive Chromium through the Chrome DevTools Protocol (CDP). Pages can detect this indirectly: script-dispatched events are untrusted, code injected with `Runtime.evaluate` shows up in stack traces or leaves globals, and heavy protocol use has side effects some detectors measure. CloakBrowser addresses part of this in the binary (README: patches for "automation signals" and "CDP input behavior"; see [[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]]) and part in the wrapper's [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]]. This page records the wrapper-side, input-related findings from batch B1; the protocol-traffic side is covered in [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]].

## Signals

| Signal | What the page sees | Wrapper-side mitigation |
|---|---|---|
| Untrusted events | `event.isTrusted === false` for events created with `dispatchEvent` from script | all key and pointer input goes through CDP `Input.*` commands, which the browser treats as trusted; shift symbols use `Input.dispatchKeyEvent` with the Shift modifier and physical key code |
| Evaluate stack frames | a script frame from `page.evaluate` in the call stack of a hooked DOM method (pinned by the #110 tests below) | no `page.evaluate` on the action path; the legacy `human_type` fallback that used `page.evaluate` is documented in its own docstring as detectable |
| Main-world footprint | helper globals, patched prototypes, or listeners added to `window` | element resolution and reads run in a CDP isolated world (`Page.createIsolatedWorld`) whose globals page scripts cannot see; InjectedScript hooks that would register listeners or dispatch events on the shared `window` are disabled |
| Framework globals | `__pwInitScripts`, `__playwright__binding__` on `window` | probed by `examples/fingerprint_scan_test.py`; not addressed by the humanize files in this batch ^[ambiguous] |
| `navigator.webdriver` | true under automation | binary-level; see [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties\|Navigator and hardware properties]] |
| Protocol traffic volume | side effects of many CDP round trips before a challenge fires ^[inferred] | README: prefer Playwright over Puppeteer, minimize `page.evaluate()` calls before reCAPTCHA runs, and use native sleep instead of `wait_for_timeout()` ([[projects/cloakbrowser/techniques/cdp-traffic-minimization\|CDP traffic minimization]]) |
| Programmatic DOM actions | `focus()`, value setting, scroll jumps without input | humanized actions focus by click, type values, wheel-scroll ([[projects/cloakbrowser/techniques/humanized-input\|Humanized input]]) |

## Why the isolated world matters

An isolated world shares the DOM with the page but has its own JavaScript global object and prototypes. Queries the engine runs there cannot be observed through hooked `Element.prototype` methods, `MutationObserver` tricks on injected globals, or `Error().stack` inspection in the page's realm. ^[inferred] The engine treats a failed isolated-world evaluation as an error (`StealthWorldError`) and never falls back to evaluating in the main world, so a failure cannot silently leak a main-world footprint. The escape hatch `page._stealth_world.evaluate()` lets user code use the same isolated world instead of `page.evaluate`.

## Pinned by tests (issue #110, B8 batch)

Issue #110 reported three leaks in the humanize layer (`isInputElement`, `isSelectorFocused`, `typeShiftSymbol`). `tests/test_stealth_reproduction_110.py` replays the issue's detection script exactly, and `tests/test_stealth_unit.py` adds mock-based unit tests and more real-browser runs. The detection script shows how a page catches automation without any CDP-specific API:

- **Evaluate-origin probe.** The page replaces `document.querySelector` with a wrapper that throws and catches an `Error` and checks whether the stack contains `:302:`. That marker identifies calls coming from Playwright's evaluate wrapper rather than from page code. ^[inferred] Each hit is logged as a leak.
- **Trust probe.** A capturing `keydown` listener records every event with `isTrusted === false`.

The tests require **zero** detections from both probes when clicking, `fill()`, `clear()` and `keyboard.type()` with text such as `Hello!@#$%^&*()`. All 21 shift symbols must arrive as trusted key events, in both headless and headed mode and in the sync and async APIs. The unit tests fix the mechanism:

- A shift symbol is sent as exactly two `Input.dispatchKeyEvent` calls with the Shift modifier flag `8`. The `keyDown` carries `text`, `unmodifiedText`, the physical `code` and `windowsVirtualKeyCode` (digit-row symbols map to key codes 49 to 57 and 48). The `keyUp` carries no text. `page.evaluate` and `insert_text` are never called.
- Without a CDP session, the code falls back to `page.evaluate` plus `insert_text`, which the tests themselves label the "detectable path".
- Plain letters never touch CDP or `evaluate`. Uppercase letters use Shift plus the raw keyboard.
- Every humanized page carries `_stealth_world`. After `page.goto()` the isolated world is invalidated and re-created transparently, so clicks and typing keep working after navigation.

`tests/test_stealth.py` also asserts that no `window` key starts with `cdc_` or `__webdriver`, the globals ChromeDriver-style drivers leave behind. This confirms the "Evaluate stack frames" row above, which earlier ingests had inferred.

### JS mirror (B6 batch)

`js/tests/stealth.test.ts` and `stealth.puppeteer.test.ts` (commit b1a00ed) repeat the issue #110 probes for both JS backends:

- The same `:302:` evaluate-origin probe and capturing `isTrusted` listener are used. The live runs are gated behind `SLOW=1` and use wikipedia.org, example.com and deviceandbrowserinfo.com.
- In the mock tests the shift-symbol CDP sequence (modifier `8`, text-free `keyUp`, no `page.evaluate`) holds for the Playwright and the Puppeteer keyboard alike.
- On Puppeteer, the input-element and focus checks behind `click()` and `focus()` go through the isolated world (`stealth.evaluate`), not `page.evaluate`.

The real-browser engine suite goes further than a leak probe. Its fixture page (`humanizeHarness.ts`) hooks `Element.prototype.getAttribute` and `document.activeElement` to record any read only a main-world evaluate would make. A sequence of humanized actions must add zero `window`/`document` listeners and zero DOM mutations, and value inputs must produce zero untrusted `input`/`change` events (see [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]]).

One divergence: when the isolated world fails twice, the Puppeteer `StealthEval.evaluate` returns `undefined`, whereas the Playwright engine raises `StealthWorldError`. Neither backend falls back to the main world on that path. ^[inferred]

## Limits

- `dispatch_event` with a JSHandle in its init dict runs Playwright's own main-world call, because that handle lives in the main world (`RawFallback`).
- Frameworks connected over CDP that issue their own `Runtime.evaluate` or synthetic events bypass these protections. See [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]]. ^[inferred]

## How the detectors test it (batch B2 evidence)

- **deviceandbrowserinfo.com** reports a dedicated `isAutomatedWithCDP` flag next to `isPlaywright`, `hasWebdriverTrue` and `hasInconsistentChromeObject`. `examples/stealth_test.py` passes only when `isBot` is false **and** every sub-flag is false. The README test table (as of Oct 2026, Chromium 155) cites `isAutomatedWithCDP: false` as its evidence for "CDP detection: Not detected", and reports 0 true deviceandbrowserinfo flags for CloakBrowser against 6 for stock Playwright.
- **BrowserScan** bot detection groups its checks into four categories: Webdriver, User-Agent, CDP and Navigator. `images/browserscan_normal.png` shows the overall verdict "Test Results: Normal" with those four tiles, which matches the README's "NORMAL (4/4)". The screenshot carries no date or build. ^[inferred]
- **Rebrowser Bot Detector** (`bot-detector.rebrowser.net`), described in the suite as an "automation signal detector", publishes per-test ratings in a `detections-json` element. The suite passes only with zero tests rated as triggered. The individual Rebrowser tests are not named in the repository. ^[ambiguous]
- **bot.incolumitas.com**: the Python suite tolerates `WEBDRIVER` and `connectionRTT` failures as "known false positives"; the JS suite allows one failure. See [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]].

## WebDriver-based integrations

Selenium and undetected-chromedriver drive the CloakBrowser binary through ChromeDriver (W3C WebDriver), not through Playwright's CDP session (`examples/integrations/selenium_example.py`, `undetected_chromedriver.py`). Both examples check `navigator.webdriver`, plugin count and platform after launch to confirm the binary patches hold under ChromeDriver. undetected-chromedriver is pitched as a "double stealth layer": it patches ChromeDriver's own detection signals while CloakBrowser patches fingerprints in C++. The README's comparison table rates undetected-chromedriver alone as config-level patching that "breaks often". ChromeDriver still uses CDP internally, so the CDP-side protections of the humanize engine do not apply on this path. ^[inferred] See [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]].

## Open Questions

- Which binary patches implement "CDP input behavior mimicking" (README) and how they interact with the wrapper's CDP key path is not visible in the repository. ^[ambiguous]

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/cloakbrowser/human/keyboard.py
- ~/code/CloakBrowser/cloakbrowser/human/world.py
- ~/code/CloakBrowser/cloakbrowser/human/injected.py
- ~/code/CloakBrowser/cloakbrowser/human/patch.py
- ~/code/CloakBrowser/examples/fingerprint_scan_test.py
- ~/code/CloakBrowser/README.md
- ~/code/CloakBrowser/examples/stealth_test.py (B2)
- ~/code/CloakBrowser/examples/integrations/selenium_example.py (B2)
- ~/code/CloakBrowser/examples/integrations/undetected_chromedriver.py (B2)
- ~/code/CloakBrowser/images/browserscan_normal.png (B2, vision-interpreted)
- ~/code/CloakBrowser/tests/test_stealth_reproduction_110.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/tests/test_stealth_unit.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/tests/test_stealth.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/js/tests/stealth.test.ts, js/tests/stealth.puppeteer.test.ts, js/tests/humanizeEngine.test.ts, js/tests/humanizeHarness.ts (commit b1a00ed, B6 batch)
