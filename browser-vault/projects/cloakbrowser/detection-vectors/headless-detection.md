---
title: >-
    Headless detection
category: detection-vectors
tags: [headless, bot-detection, creepjs, fingerprinting, browser-automation]
aliases: [headless Chrome detection, like-headless signals]
sources:
  - ~/code/CloakBrowser/examples/fingerprint_scan_test.py
  - ~/code/CloakBrowser/examples/integrations/aws_lambda/INSTRUCTIONS.md
  - ~/code/CloakBrowser/examples/integrations/aws_lambda/lambda-entrypoint.sh
  - ~/code/CloakBrowser/dotnet/README.md
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/tests/test_stealth.py
  - ~/code/CloakBrowser/tests/test_stealth_reproduction_110.py
  - ~/code/CloakBrowser/tests/test_launch_context.py
summary: >-
    Signals that betray headless or automated Chromium (missing taskbar, absent ContentIndex/Contacts/downlinkMax APIs, Playwright globals) and how the project's scan script scores them.
provenance:
  extracted: 0.70
  inferred: 0.25
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:47:12+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Headless detection

Headless detection looks for the gaps a headless or containerized Chromium leaves compared with a desktop browser: APIs that only exist with a real display or OS integration, a screen with no taskbar, and automation-framework globals. `examples/fingerprint_scan_test.py` is the project's probe for these signals, written against "the specific headless detection signals flagged by the community" and run against fingerprint-scan.com and CreepJS. Related vectors: [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]] and [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]].

## Signals probed by the scan script

| Signal | Headless tell | Notes |
|---|---|---|
| `noTaskbar` | `screen.height == screen.availHeight` | a desktop OS reserves space for a taskbar or dock; the binary can override it with `--fingerprint-taskbar-height` |
| `noContentIndex` | `window.ContentIndex` undefined | |
| `noContactsManager` | `navigator.contacts` absent | |
| `noDownlinkMax` | `navigator.connection.downlinkMax` absent | |
| `webdriver` | `navigator.webdriver` true | |
| `isPlaywright` | `__pwInitScripts` or `__playwright__binding__` on `window` | globals that Playwright init scripts and bindings leave behind |
| WebGPU | `navigator.gpu` missing | reported, not scored |
| scrollbar width | measured from an overflow box | overlay or zero-width scrollbars differ by platform ^[inferred] |

Any `no*` signal that is true counts as a headless fail. The script also scrapes fingerprint-scan.com's bot risk score and its WebDriver, Selenium, CDP and Playwright checks.

## CreepJS scoring

The script waits 30 s for CreepJS, then reads three percentages from the page and the `window.Fingerprint.headless` object: like-headless, headless and stealth. It prints targets of like-headless ≤ 30%, headless 0% and stealth 0%, but the pass verdict it computes is looser: headless ≤ 30% and stealth ≤ 30%. The stealth score measures detectable anti-fingerprinting tampering, so a stealth browser has to keep it low as well. ^[inferred] See [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]].

The script launches with explicit screen dimensions (1920x1080) and a fixed timezone through binary flags, headed by default with `--headless` as an option (as of 26/10/2026, commit b1a00ed).

## Avoiding headless mode altogether

The AWS Lambda image always runs headed under an Xvfb virtual display at 1920x1080x24, rather than using headless mode. ^[inferred] Running a real (virtual) display sidesteps headless-only differences at the cost of an X server per container. See [[projects/cloakbrowser/architecture/aws-lambda-deployment|AWS Lambda deployment]]. The .NET README's options table also notes that detectors often flag headless.

## Project claims vs advice

The README makes two claims that pull in different directions (as of 26/10/2026, commit b1a00ed). The comparison table says CloakBrowser **passes** "Headless / Docker detection", where Playwright, playwright-stealth and undetected-chromedriver are flagged. The troubleshooting section and the recommended anti-bot config say "some sites detect headless even with our C++ patches" and advise `headless=False`, under Xvfb on servers, for DataDome and Turnstile. The best reading is that the patches beat the common headless checks, and headed mode covers the rest on the hardest targets. ^[ambiguous]

## Pinned by tests

- The live suite `tests/test_stealth.py` runs **headless** (`launch(headless=True)`) and requires a UA without `HeadlessChrome`, a `window.chrome` object, at least 5 plugins, zero failures on bot.sannysoft.com, and `isBot: false` on deviceandbrowserinfo.com, whose sub-flags include `isHeadlessChrome`. These checks pin that the classic headless tells are gone in headless mode. See [[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties|Navigator and hardware properties]].
- The issue #110 reproduction (`tests/test_stealth_reproduction_110.py`) runs headless with `humanize=True`, so the input-layer stealth is also verified without a display.
- The viewport defaults differ by mode: `tests/test_launch_context.py` requires `no_viewport=True` for headed contexts ("CDP viewport emulation would force outerWidth < innerWidth"). Headless contexts get `DEFAULT_VIEWPORT` unless the caller sets one, which matches the version gate for the free default binary. ^[inferred] See [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]].
- No test probes the scan script's `noTaskbar`, `noContentIndex`, `noContactsManager` or `noDownlinkMax` signals. They are covered only by the manual example script.

## Open Questions

- The script prints targets that are stricter than its own pass condition (like-headless is printed but not scored); which is the project's real bar is unclear. ^[ambiguous]
- Results of running the script are not in the repository; dated live results belong in [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]].

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/examples/fingerprint_scan_test.py
- ~/code/CloakBrowser/examples/integrations/aws_lambda/INSTRUCTIONS.md
- ~/code/CloakBrowser/examples/integrations/aws_lambda/lambda-entrypoint.sh
- ~/code/CloakBrowser/dotnet/README.md
- ~/code/CloakBrowser/README.md
- ~/code/CloakBrowser/tests/test_stealth.py
- ~/code/CloakBrowser/tests/test_stealth_reproduction_110.py
- ~/code/CloakBrowser/tests/test_launch_context.py
