---
title: >-
    Screen and viewport consistency
category: detection-vectors
tags: [viewport, screen, headless, detection]
aliases: [impossible window, outerWidth innerWidth, VM detection]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/headless-detection]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/navigator-and-hardware-properties]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/config.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_config.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_build_args.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/conftest.py
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/puppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/update.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/setup.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts
summary: >-
    Detectors compare window, viewport and screen dimensions; an emulated viewport inside a real window gives outerWidth < innerWidth, a physically impossible window.
provenance:
  extracted: 0.72
  inferred: 0.22
  ambiguous: 0.06
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:46:34+01:00
updated: 2026-10-31T06:53:05+01:00
---

# Screen and viewport consistency

A real browser's geometry is internally consistent: the outer window contains the viewport (`outerWidth >= innerWidth`), the screen contains the window, and headless browsers without window chrome report outer equal to inner. Automation frameworks break this by emulating a fixed viewport through CDP. Playwright's default is 1280×720, Puppeteer's 800×600. In a headed browser whose real window is larger or smaller, this yields an outer window narrower than its own viewport, which the CloakBrowser code calls "a physically impossible window" and a bot tell. FingerprintJS also flags "Virtual machine" when screen dimensions do not match the viewport (README).

## What the detecting side measures

- `window.outerWidth/outerHeight` vs `innerWidth/innerHeight`: outer smaller than inner is impossible.
- `screen.width/height` and `availHeight` vs viewport: FingerprintJS's "Virtual machine" verdict. The README's fix is `--fingerprint-screen-width/height` matching the viewport.
- Taskbar height (`screen.height - availHeight`): the binary defaults to Win 48, Mac 95, Linux 0, overridable with `--fingerprint-taskbar-height`. A Windows persona with zero taskbar would be inconsistent. ^[inferred]
- Maximized-window plausibility: the wrapper's comments say "real Chrome overwhelmingly runs maximized", so it adds `--start-maximized` when the user sets no geometry. ^[inferred] that unmaximized odd sizes are used as a weak signal.

## How CloakBrowser keeps it coherent

- **Headed** (Playwright and Puppeteer): no viewport emulation (`viewport: null` / `defaultViewport: null`), so the page tracks the real OS window. Playwright's `browser.newPage()`/`newContext()` are wrapped to default to `viewport: null` when the caller gives none. An explicit viewport, including null, is always honoured.
- **Headless on newer binaries** (`binarySupportsHeadlessNoViewport`, on at `148.0.7778.215.4`, as of 26/10/2026, commit b1a00ed): also no emulation, since the binary reports coherent headless dimensions natively. Whether any shipped binary is past this gate today is disputed; see "Pinned by tests" below. ^[ambiguous]
- **Headless on older binaries**: a fixed `DEFAULT_VIEWPORT`, coherent because headless has no window chrome (outer equals inner) and deterministic across runs.
- **Effective headless value**: each wrapper reads `headless` with the same precedence its launch call uses, so a headed browser cannot get a fixed viewport by accident.
- The gate fails safe (off) for a local binary override with no declared version and for malformed versions (`config.test.ts`).

- **Docker/Xvfb**: the container entrypoint starts `openbox` on its 1920×1080 Xvfb display so `--start-maximized` actually maximizes; bare Xvfb has no window manager and the flag would be a silent no-op. See [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]].

## Pinned by tests (Python suite, B7)

- **Parity-critical gates.** `tests/test_config.py` labels `binary_supports_headless_no_viewport` and `binary_supports_maximized_window` "parity-critical: JS and .NET mirror this". Both gates are off at a declared `148.0.7778.215.3` and on at `.4` and above.
  - A declared version (argument or `CLOAKBROWSER_VERSION`) wins even under a `CLOAKBROWSER_BINARY_PATH` override.
  - An override with no declared version stays off.
  - A keyless default (the free per-platform version) stays off.
- **Why the maximize gate exists.** The test says that below the threshold an auto `--start-maximized` "would make headless report outerWidth < innerWidth (a bot tell)". That is the impossible-window signal at the top of this page.
- **When `--start-maximized` is added** (`tests/test_build_args.py`, mirroring `js/tests/config.test.ts`): only when the gate is passed in as on, never when a user `--window-size` chose a geometry, and never twice.
- **Test isolation shows the gates are live for Pro users.** `tests/conftest.py` points every unit test at a temporary cache directory. Its reason: a developer's real `license.key` or cached Pro marker "flips version-gated defaults such as the headless `no_viewport` shim and inline proxy auth". A resolved Pro build can therefore pass the gate.
- **Contradiction.** The test comments call `148.0.7778.215.3` the "current live Pro version" and the threshold "an unshipped version", so "the resolved-version path is a no-op today". The README lists Pro Stable `155.0.8059.39.1` (as of 26/10/2026), which compares above the threshold, and the conftest reasoning above agrees. The comments probably predate the 15x builds. If so, Pro users already get native headless geometry and auto-maximize, while keyless free builds (145/146) keep the fixed `DEFAULT_VIEWPORT`. ^[ambiguous]

### JS suite (B6)

`js/tests/launch.test.ts`, `puppeteer.test.ts` and `update.test.ts` (commit b1a00ed) pin the same rules for both JS backends:

- **Playwright.** `buildContextOptions` returns `viewport: null` when headed ("CDP emulation would force outerWidth < innerWidth") and `DEFAULT_VIEWPORT` when headless on a binary below the gate. An explicit viewport wins even when headed. The headless value is read from `launchOptions.headless` when present, because that is what Playwright actually launches with.
- **Puppeteer.** `defaultViewport` is `null` when headed, otherwise Puppeteer forces 800×600, and `DEFAULT_VIEWPORT` when headless. An explicit `launchOptions.defaultViewport`, including `null`, is honoured. The top-level `headless` decides.
- **Gate per channel.** A cached preview build `148.0.7778.215.4` passes the headless-no-viewport and maximized-window gates, while a stable build `145.0.1000.1` does not. The fixed headless viewport therefore depends on which channel's build launches.
- `js/tests/setup.ts` gives the same isolation reason as the Python conftest: a cached Pro marker flips "the headless no-viewport shim".

Details: [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]], [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].

## Open Questions

- Which shipped binaries pass `HEADLESS_NO_VIEWPORT_MIN_VERSION` today: the tests' "unshipped" comment or the README's 155 Pro builds (see above). ^[ambiguous]

- The README says the binary derives screen dimensions from the seed (auto-generated 1920×1080 on Windows/Linux personas, 1440×900 on macOS), while the real window size comes from the OS or Xvfb display. A `js/src/config.ts` comment (mirrored in `cloakbrowser/config.py`) states the opposite for screen size: screen and window size "come from the real display, not this flag (verified: identical across seeds)", in the headed-mode context. How the binary reconciles a seed-derived screen with a real window of a different size is not documented. ^[ambiguous]

## Sources

- repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/tests/config.test.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/README.md (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_config.py, tests/test_build_args.py, tests/conftest.py (commit b1a00ed, B7 batch)
- repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts, js/tests/puppeteer.test.ts, js/tests/update.test.ts, js/tests/setup.ts (commit b1a00ed, B6 batch)
- repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts (commit b1a00ed, "identical across seeds" comment)
