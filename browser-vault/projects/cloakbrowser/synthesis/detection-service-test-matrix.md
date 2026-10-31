---
title: >-
    Detection service test matrix
category: synthesis
tags: [detection-services, recaptcha, turnstile, testing, cloakbrowser]
aliases: [stealth test suite, test results]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/behavioral-analysis]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/images/recaptcha_v3_score_09.png
  - ~/code/CloakBrowser/images/turnstile_non_interactive.png
  - ~/code/CloakBrowser/js/examples/stealth-test.ts
  - ~/code/CloakBrowser/js/README.md
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/examples/stealth_test.py
  - ~/code/CloakBrowser/examples/recaptcha_score.py
  - ~/code/CloakBrowser/images/browserscan_normal.png
  - ~/code/CloakBrowser/images/fingerprintjs_pass.png
  - ~/code/CloakBrowser/tests/test_stealth.py
  - ~/code/CloakBrowser/tests/test_stealth_unit.py
summary: >-
    Dated, vendor-reported results of CloakBrowser against live detection services (reCAPTCHA v3, Turnstile, FingerprintJS, BrowserScan and others), with the pass criteria the test suite uses.
provenance:
  extracted: 0.64
  inferred: 0.27
  ambiguous: 0.09
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:47:25+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Detection service test matrix

CloakBrowser backs its stealth claims with two kinds of evidence: a results table in its READMEs and a runnable suite (`js/examples/stealth-test.ts`, which "Mirrors Python examples/stealth_test.py") that visits live detection services and scores each one. Every result here is **vendor-reported**: the project ran the tests itself, against services that change weekly. Treat the numbers as dated snapshots, not guarantees. ^[inferred] Results depend on the Playwright backend, on proxy quality and on behavior. See [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP and automation detection]] and [[projects/cloakbrowser/detection-vectors/behavioral-analysis|Behavioral analysis]].

## Claimed results (JS README, as of 26/10/2026, commit b1a00ed)

| Service | Stock browser | CloakBrowser |
|---|---|---|
| reCAPTCHA v3 | 0.1 (bot) | 0.9 (human) |
| Cloudflare Turnstile | FAIL | PASS |
| FingerprintJS | DETECTED | PASS |
| BrowserScan | DETECTED | NORMAL (4/4) |
| bot.incolumitas.com | 13 fails | 1 fail |
| `navigator.webdriver` | `true` | `false` |
| CDP detection | Detected | Not detected |
| TLS fingerprint | Mismatch | Identical to Chrome |

The README adds "tested against 50+ detection sites". The table gives no binary version or test date. ^[ambiguous]

## Claimed results (main README, "Last tested: Oct 2026 (Chromium 155)", commit b1a00ed)

The main README table is newer and longer than the JS README's. Results are for "the latest Pro/current build unless noted":

| Service | Stock Playwright | CloakBrowser | README note |
|---|---|---|---|
| reCAPTCHA v3 | 0.1 (bot) | 0.9 (human) | Pro/current build, server-side verified |
| Cloudflare Turnstile (non-interactive) | FAIL | PASS | auto-resolve |
| Cloudflare Turnstile (managed) | FAIL | PASS | single click |
| ShieldSquare | BLOCKED | PASS | production site |
| FingerprintJS bot detection | DETECTED | PASS | Pro/current build, demo.fingerprint.com |
| BrowserScan bot detection | DETECTED | NORMAL (4/4) | |
| bot.incolumitas.com | 13 fails | 1 fail | "WEBDRIVER spec only" |
| deviceandbrowserinfo.com | 6 true flags | 0 true flags | `isBot: false` |
| `navigator.plugins.length` | 0 | 5 | real plugin list |
| `window.chrome` | `undefined` | `object` | |
| UA string | `HeadlessChrome` | `Chrome/151.0.0.0` | no headless leak |
| CDP detection | Detected | Not detected | `isAutomatedWithCDP: false` |
| TLS fingerprint | Mismatch | Identical to Chrome | ja3n/ja4/akamai match |

The UA row shows `Chrome/151.0.0.0` although the table is labelled Chromium 155, so at least that row predates the latest re-test. ^[ambiguous] A further README proof caption claims deviceandbrowserinfo's behavioral check reports "You are human!" with `humanize=True` (24/24 signals). See [[projects/cloakbrowser/detection-vectors/behavioral-analysis|Behavioral analysis]].

**Comparison table (README)**, reCAPTCHA v3 score: Playwright 0.1, playwright-stealth 0.3 to 0.5, undetected-chromedriver 0.3 to 0.7, CloakBrowser 0.9. Turnstile: "Sometimes" for both competitors. These competitor figures are vendor claims with no method given. ^[ambiguous]

## Screenshot evidence (`images/`)

- **`recaptcha_v3_score_09.png`**: the Google `recaptcha-demo.appspot.com` "Request scores" page. The backend response shows `"success": true`, `"score": 0.9`, `"action": "examples/v3scores"` and `"challenge_ts": "2026-02-22T19:17:18Z"`. The timestamp falls on the date of wrapper 0.1.0 (Chromium 142, 16 patches), so the screenshot most likely shows the first release rather than the current 155 build. ^[inferred] The demo page itself warns that its score "is not a reflection" of production traffic and that the threshold must be tuned per site. The 0.9 is therefore a smoke test, not a production guarantee. ^[inferred]
- **`turnstile_non_interactive.png`**: a "Cloudflare turnstile demo", "Non-interactive challenge" page with the widget showing a green check and "Success!". There is no date or version on the image. ^[inferred]
- **`browserscan_normal.png`** (B2): the browserscan.net "Bot Detection" page with "Test Results: Normal" and four category tiles: Webdriver, User-Agent, CDP, Navigator. This is the "4/4" in the README. No date or build is visible. ^[inferred]
- **`fingerprintjs_pass.png`** (B2): Fingerprint's web-scraping demo ("Search for today's flights", SFO to JFK) displaying flight results with prices, i.e. data served rather than a block message. The flight dates read February 21 and 25, 2026, which suggests a capture around the same time as the reCAPTCHA screenshot (22/02/2026), long before the Chromium 155 build. ^[inferred] The README's "Pro/latest build" caption for the equivalent hosted image is therefore probably out of date for the repository copy. ^[ambiguous]

## Python suite (`examples/stealth_test.py`) differs from the JS one

The JS suite says it "Mirrors Python examples/stealth_test.py", but the Python file at commit b1a00ed runs a different set of checks. It launches with `geoip=True` (plus optional `--proxy` and `--headed`), opens a fresh 1920×1080 context per test, prints the presented device first (UA, full version from client hints, platform, cores, screen, WebGL vendor/renderer, languages, exit IP), and exits non-zero unless every test passes:

| Check | Pass rule |
|---|---|
| bot.sannysoft.com | no table row marked failed |
| bot.incolumitas.com | failures limited to `WEBDRIVER` and `connectionRTT` ("known false positives"); polls until 30+ results stabilize |
| Rebrowser Bot Detector | zero tests rated as triggered in `detections-json` |
| deviceandbrowserinfo.com | `isBot` false **and** no sub-flag true |
| BrowserScan | zero "Abnormal" |
| CreepJS (`--fingerprint-noise=false`, separate launch) | `totalLies == 0` |

Differences from the JS suite: Python adds Rebrowser and CreepJS and drops FingerprintJS and reCAPTCHA; its incolumitas rule allows `connectionRTT` as well as `WEBDRIVER`; and it waits with `time.sleep()` rather than `page.waitForTimeout()`, which follows the README's own advice. ^[inferred] The CreepJS test is the only place in the repository where noise is shown to matter for a specific detector. See [[projects/cloakbrowser/detection-vectors/canvas-fingerprinting|Canvas fingerprinting]] and [[projects/cloakbrowser/techniques/noise-injection|Noise injection]].

**`examples/recaptcha_score.py`** is the standalone reCAPTCHA check: headless `launch()`, the Google v3 demo page, a wait for the backend-response text (because scoring can finish after `networkidle`), then a regex for `"score"`. Its docstring expects 0.9 with CloakBrowser and 0.1 to 0.3 for default Playwright, and it has no pass threshold. Note that it runs headless with no proxy, settings the README elsewhere says lower scores on hard targets. ^[inferred]

## Pass criteria in the JS stealth suite

The JS suite (`js/examples/stealth-test.ts`) launches headless with an optional `--proxy` argument, runs six checks and exits non-zero unless all pass:

| Check | Pass rule |
|---|---|
| bot.sannysoft.com | No table row marked `failed` |
| bot.incolumitas.com | At most **1** failed test. One `WEBDRIVER` false positive "is expected" |
| BrowserScan bot-detection | Zero "Abnormal" verdicts |
| deviceandbrowserinfo.com `are_you_a_bot` | `isBot` false. Sub-flags reported: `hasBotUserAgent`, `hasWebdriverTrue`, `isHeadlessChrome`, `isAutomatedWithCDP`, `hasSuspiciousWeakSignals`, `isPlaywright`, `hasInconsistentChromeObject` |
| FingerprintJS web-scraping demo | Flight results are shown and no "request was blocked" / "bot visit detected" text appears |
| reCAPTCHA v3 demo | Score >= 0.7 (the README claims 0.9) |

The deviceandbrowserinfo flags list the detection vectors the project cares about most: user agent, webdriver, headless, CDP automation, Playwright artifacts and `window.chrome` consistency. See [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]] and [[projects/cloakbrowser/detection-vectors/headless-detection|Headless detection]]. ^[inferred]

## Conditions behind good scores (README guidance)

- Use Playwright, not Puppeteer, which sends more CDP traffic (see [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]]).
- Use residential proxies, because datacenter IPs fail on IP reputation, not fingerprint.
- Stay on the page for 15+ seconds and leave 30+ seconds between pages that call reCAPTCHA.
- Use a fixed `--fingerprint` seed for a returning-visitor identity.
- Use `type()` instead of `fill()`.
- Minimize `page.evaluate()` calls and avoid `page.waitForTimeout()`, which sends CDP commands. See [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]].
- Some sites detect headless even with the C++ patches, so the README suggests `headless: false` plus `geoip` and `humanize` for hard targets.

## Pytest suite (`tests/test_stealth.py`, B8 batch)

A third suite lives in the pytest tests, separate from `examples/stealth_test.py`. Its live tests are marked `slow` and share one **headless** browser, with an optional proxy from `CLOAKBROWSER_TEST_PROXY`. Its service set matches the **JS** suite, not the Python example: sannysoft, incolumitas, BrowserScan, deviceandbrowserinfo (`isBot` false only), the FingerprintJS web-scraping demo and reCAPTCHA v3 at 0.7 or higher. So the JS suite effectively mirrors this file, despite its comment naming the example. ^[inferred] Differences from the JS suite:

- **incolumitas** uses the same allow-list as the Python example (`WEBDRIVER`: "spec-level false positive across all builds"; `connectionRTT`: "detects datacenter/proxy network latency, not browser"), not the JS rule of at most one failure.
- **Waiting** uses native `time.sleep` (3 to 12 s), so the README's `waitForTimeout` warning does not apply here either.
- **Property checks** run first: `navigator.webdriver` false, no `HeadlessChrome` in the UA, `window.chrome` an object, at least 5 plugins, non-empty `navigator.languages`, and no `cdc_` or `__webdriver` keys on `window`. See [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]].
- **Regression tests**: `page.goto()` immediately after launch must work (#9, reCAPTCHA reportedly failed when navigating too soon), and `add_init_script` must work with and without a context-level proxy, with no `ERR_TUNNEL_CONNECTION_FAILED` (#27).
- **Behavioral check** (`tests/test_stealth_unit.py`, headed, `humanize=True`, `geoip=True`): a full login-form fill on deviceandbrowserinfo.com's `are_you_a_bot_interactions` page must not raise the `superHumanSpeed` or `suspiciousClientSideBehavior` flags, with no untrusted key events. See [[projects/cloakbrowser/detection-vectors/behavioral-analysis|Behavioral analysis]].

The README's TLS row is not tested by any suite. See [[projects/cloakbrowser/detection-vectors/tls-network-fingerprinting|TLS and network fingerprinting]].

## Inconsistencies noticed

- The JS suite itself calls `page.waitForTimeout()` between steps (3 to 12 s), the very call the README says "sends CDP protocol commands that reCAPTCHA detects", and it runs headless. Its reCAPTCHA result is therefore a pessimistic or noisy measure, or the waitForTimeout warning is overstated. ^[ambiguous]
- The JS suite's pass threshold for reCAPTCHA (0.7) is lower than the headline claim (0.9).
- The JS suite claims to mirror `examples/stealth_test.py`, but the two test different services with different tolerances (see above). The Python suite uses native sleep, so the `waitForTimeout` issue is specific to the JS port. ^[inferred]
- The FingerprintJS and reCAPTCHA checks do exist in Python, in the pytest suite `tests/test_stealth.py` (B8 correction; the B2 note above considered only `examples/`). Turnstile and ShieldSquare are not checked automatically anywhere; those rows rest on screenshots and README text. ^[inferred]

## Sources

- `~/code/CloakBrowser` at commit `b1a00ed`: `images/recaptcha_v3_score_09.png` (vision-interpreted), `images/turnstile_non_interactive.png` (vision-interpreted), `js/examples/stealth-test.ts`, `js/README.md`
- `~/code/CloakBrowser` at commit `b1a00ed` (B2): `README.md` (Test Results, Comparison), `examples/stealth_test.py`, `examples/recaptcha_score.py`, `images/browserscan_normal.png` (vision-interpreted), `images/fingerprintjs_pass.png` (vision-interpreted)
- ~/code/CloakBrowser/tests/test_stealth.py (commit b1a00ed, B8 batch)
- ~/code/CloakBrowser/tests/test_stealth_unit.py (commit b1a00ed, B8 batch)
