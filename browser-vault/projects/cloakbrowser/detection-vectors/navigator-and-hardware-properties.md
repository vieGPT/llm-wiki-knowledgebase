---
title: >-
    Navigator and hardware properties
category: detection-vectors
tags: [navigator, user-agent, client-hints, bot-detection, fingerprinting]
aliases: [navigator.webdriver, window.chrome check, plugins check, UA and Client Hints]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/headless-detection]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_launch.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_launch_context.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_persistent_context.py
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/puppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/humanizeEngine.test.ts
summary: >-
    The first-pass automation checks on navigator and window (webdriver, UA, window.chrome, plugins, languages, CPU/RAM, Client Hints) and how the binary and wrapper make them read like stock Chrome.
provenance:
  extracted: 0.74
  inferred: 0.20
  ambiguous: 0.06
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:54:19+01:00
updated: 2026-10-31T06:00:39+01:00
---

# Navigator and hardware properties

The cheapest bot checks read a handful of JavaScript properties and compare them with what stock desktop Chrome reports. Automation frameworks leave obvious tells here: `navigator.webdriver` set to true, a `HeadlessChrome` user agent, no `window.chrome` object, and an empty plugin list. Better detectors also check that the user agent, the User-Agent Client Hints, `navigator.platform`, CPU count and memory describe one plausible machine. CloakBrowser handles these surfaces in the compiled binary ([[projects/cloakbrowser/techniques/source-level-chromium-patches|Source-level Chromium patches]]) and has the wrapper pick the persona and seed ([[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]). The wrapper injects no JavaScript to patch these properties.

## What detectors read

| Property | Automation tell | Stock Playwright | CloakBrowser (README, as of 26/10/2026, commit b1a00ed) |
|---|---|---|---|
| `navigator.webdriver` | `true` under WebDriver/CDP automation | `true` | `false` (source-level patch) |
| `navigator.userAgent` | contains `HeadlessChrome` in headless mode | `HeadlessChrome` | `Chrome/151.0.0.0` |
| `window.chrome` | `undefined` in automated or headless builds | `undefined` | `object` |
| `navigator.plugins.length` | 0 | 0 | 5 (the real Chrome PDF plugin list) |
| `navigator.languages` | empty, or contradicting the IP's country | n/a | filled from `--lang` / `--fingerprint-locale` |
| `window` keys | ChromeDriver's `cdc_*` and `__webdriver*` globals | n/a | absent |
| `navigator.hardwareConcurrency`, `navigator.deviceMemory` | values that do not fit the claimed device | host values | generated from the seed (default 8 and 8 GB) |
| `navigator.platform`, UA OS, Client Hints platform | an OS that contradicts fonts, GPU or screen | host OS | the persona from `--fingerprint-platform` |
| Client Hints brand and version | a brand list or version that disagrees with the UA | n/a | `--fingerprint-brand` (Chrome, Edge, Opera, Vivaldi) and `--fingerprint-brand-version` update the UA and Client Hints together |

deviceandbrowserinfo.com rolls several of these into named flags: `hasBotUserAgent`, `hasWebdriverTrue`, `isHeadlessChrome` and `hasInconsistentChromeObject`. The project's suites read those flags (see [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]]).

## How CloakBrowser covers them

- **Binary patches.** `navigator.webdriver`, the plugin list, `window.chrome` and the absence of a headless UA marker are compiled in. The README lists "hardware reporting" and "automation signal removal" among the patch areas.
- **Removing leaky launch defaults.** The wrapper strips Playwright's `--enable-automation`, the default switch that sets `navigator.webdriver = true` ([[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]).
- **One seed, one device.** The binary derives hardware concurrency, device memory, GPU and screen from the `--fingerprint` seed as one package, so the values fit together. The README warns that "changing single values by hand can break that coherence". See [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].
- **Persona selection.** `--fingerprint-platform` controls `navigator.platform`, the UA OS and GPU pool selection. The wrapper sets `windows` on Linux and Windows hosts and `macos` on macOS. See [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]].
- **Language through flags, not CDP.** `locale` becomes the binary flags `--lang` and `--fingerprint-locale`. It is not passed as a context option, because Playwright would implement that with detectable CDP emulation. See [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]].

Patching in C++ instead of overriding getters in JavaScript means there is no wrapper function for a detector to unmask with `Function.prototype.toString` or property-descriptor checks. ^[inferred] See [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]].

## Pinned by tests

- The live suite `tests/test_stealth.py` launches **headless** and asserts: `navigator.webdriver` is `false`; the UA contains `Chrome/` and not `HeadlessChrome`; `typeof window.chrome` is `"object"`; there are at least 5 plugins; `navigator.languages` is not empty; and no `window` key starts with `cdc_` or `__webdriver`. `tests/test_launch.py` repeats the webdriver, `window.chrome` and plugin checks. Its docstrings name the stock Playwright values these replace ("Playwright leaks undefined", "Playwright has 0").
- The headless run matters because headless Chromium is where the `HeadlessChrome` UA and the missing `window.chrome` normally appear. ^[inferred] See [[projects/cloakbrowser/detection-vectors/headless-detection|Headless detection]].
- `tests/test_launch_context.py` and `tests/test_persistent_context.py` require that `locale` and `timezone` go to the launch call as binary flags and never to `new_context()`, while `user_agent` and `color_scheme` are passed through as context options.

### JS suite (B6)

- `js/tests/launch.test.ts` has a smaller live smoke test, run only with `CLOAKBROWSER_BINARY_PATH` set: a headless JS launch reports a falsy `navigator.webdriver` and a non-empty `navigator.plugins`. Its mocked tests confirm the launch-side half: `ignoreDefaultArgs` contains `--enable-automation`, and `contextOptions.locale` and `contextOptions.timezoneId` are stripped with a warning, so language and timezone reach the page only through `--lang` and `--fingerprint-timezone`. `puppeteer.test.ts` pins the same flags for Puppeteer.
- `js/tests/humanizeEngine.test.ts` shows the persona flag at work: with `--fingerprint-platform=macos`, `navigator.platform` reads `MacIntel` whatever the host OS, and the humanize engine picks its select-all modifier from that value. This checks the platform-coherence row of the table above from JS.

## Residual signals and inconsistencies

- **incolumitas `WEBDRIVER`.** bot.incolumitas.com still reports one failure, `WEBDRIVER`. The Python suite allow-lists it as a "spec-level false positive across all builds", and the README says "WEBDRIVER spec only". The check is not explained, and it fails even though `navigator.webdriver` reads `false`. ^[ambiguous]
- **UA version in the README.** The results table shows `Chrome/151.0.0.0`, but the same section says it was last tested in October 2026 on Chromium 155. The row may not have been updated since the 151 build. ^[ambiguous] The `.0.0.0` tail matches Chrome's reduced user-agent format, in which only the major version is real. ^[inferred]
- **A user-supplied `user_agent`.** It is passed straight to Playwright as a context option. A UA that disagrees with the binary's Client Hints or persona would bring back the inconsistency the binary avoids. ^[inferred]

## Open Questions

- Whether the binary also covers `navigator.userAgentData.getHighEntropyValues()` fields beyond brand, brand version and platform version (architecture, bitness, model) is not stated in the README.
- What incolumitas's `WEBDRIVER` test measures, given that `navigator.webdriver` is false.

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
- repo:github.com/CloakHQ/CloakBrowser/tests/test_launch.py
- repo:github.com/CloakHQ/CloakBrowser/tests/test_launch_context.py
- repo:github.com/CloakHQ/CloakBrowser/tests/test_persistent_context.py
- repo:github.com/CloakHQ/CloakBrowser/README.md (Test Results, Default Fingerprint, Additional Flags)
- repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts, js/tests/puppeteer.test.ts, js/tests/humanizeEngine.test.ts (B6 batch)
