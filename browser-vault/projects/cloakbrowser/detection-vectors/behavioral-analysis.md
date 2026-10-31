---
title: >-
    Behavioral analysis
category: detection-vectors
tags: [behavioral-biometrics, bot-detection, mouse-dynamics, keystroke-dynamics, recaptcha]
aliases: [behavioral bot detection, interaction biometrics]
sources:
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/cloakbrowser/human/mouse.py
  - ~/code/CloakBrowser/cloakbrowser/human/keyboard.py
  - ~/code/CloakBrowser/cloakbrowser/human/engine.py
  - ~/code/CloakBrowser/js/tests/humanizeEngine.test.ts
summary: >-
    What behavioral bot detection measures (mouse paths, click timing, keystroke dynamics, scroll deltas, fill vs type) and which CloakBrowser input behaviors address each signal.
provenance:
  extracted: 0.52
  inferred: 0.44
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Behavioral analysis

Behavioral analysis scores *how* a session interacts with a page rather than what the browser reports about itself. A perfect device fingerprint still fails if the cursor teleports, clicks last zero milliseconds, or form values appear without key events. CloakBrowser's C++ patches do not address this vector; the wrapper's `humanize=True` layer does (see [[projects/cloakbrowser/techniques/humanized-input|Humanized input]], implemented by [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]]). The repository documents this vector only from the evading side, so the descriptions of what detectors measure are mostly inferred from what the countermeasures target. ^[inferred]

## Signals

| Signal family | What a detector can observe | Stock Playwright behavior | CloakBrowser countermeasure |
|---|---|---|---|
| Pointer trajectory | straight lines, constant velocity, no corrective sub-movements ^[inferred] | instant teleport to the element centre | Fitts-law Bézier strokes with early speed peak, wobble, 35% overshoot plus correction |
| Pointer event stream | event rate, duplicate-coordinate moves, first move from (0, 0) ^[inferred] | one or a few synthetic moves | ~60 Hz events, no zero-movement events, cursor starts near the address bar |
| Clicks | aim point distribution, mouse-down hold, aim pause before press ^[inferred] | centred point, immediate down/up | randomized aim (left-biased in inputs), 40–150 ms hold, aim delay |
| Keystroke dynamics | inter-key intervals, hold times, rollover, uniformity, typos ^[inferred] | uniform or zero delays | log-normal timing from the 136M Keystrokes study, letter-pair effects, rollover, thinking pauses, corrected typos |
| Value injection | field value changes with no keyboard events | `fill()` sets the value directly | fill clears and types key by key; date/range inputs operated with trusted events |
| Event trust | `isTrusted=false` events from script dispatch, script stack frames | n/a | CDP `Input.dispatchKeyEvent` for shift symbols; no `page.evaluate` key dispatch |
| Scrolling | jump scrolls, `deltaY` vs `wheelDeltaY` mismatches, wheel cadence ^[inferred] | programmatic scroll into view | whole-notch wheel events sized per platform, flick rhythm, occasional overshoot |
| Focus | focus changes without a pointer or Tab cause ^[inferred] | programmatic `focus()` | focus only by click |
| Dwell and pacing | very short visits, back-to-back challenge calls | n/a | README advice: stay 15+ s before reCAPTCHA fires, wait 30+ s between pages with reCAPTCHA |

## Evidence in the repository

- The README states that `fill()` "sets values directly without keyboard events, which reCAPTCHA's behavioral analysis flags", and recommends `page.type()` with a delay; with `humanize=True`, `fill()` itself types (as of 26/10/2026, commit b1a00ed).
- The README reports deviceandbrowserinfo.com behavioral bot detection returning "You are human!" with `humanize=True` (24/24 signals passed). This is a vendor claim with a screenshot, not a reproducible test in this batch. ^[ambiguous]
- The keyboard model cites Dhakal et al., CHI 2018 (136M Keystrokes); the mouse model cites Fitts' law, Meyer's optimised sub-movements and Plamondon's log-normal strokes (code comments in `keyboard.py` and `mouse.py`).

## What the end-to-end suite observes (P2 pass)

`js/tests/humanizeEngine.test.ts` drives the real binary with `humanize: true` against a local page that records every input event, so it checks several rows of the table above from the detector's side of the page (commit b1a00ed):

- **Trajectory before a click**: at least three `mousemove` events precede each click, and click options such as right button plus Shift arrive on the `mousedown`.
- **No untrusted value events**: filling date and range inputs and picking a `<select>` option produces zero `input` or `change` events with `isTrusted=false`. Inputs with no human route are refused rather than set by script: a colour input fails with "native colour picker", a checkbox with "cannot be filled".
- **No programmatic focus**: a field that cannot be scrolled into view times out with "outside of the viewport" and stays empty instead of being focused by script.
- **Realistic key stream**: typing `xX aA zZ 1! qQ` three times never produces a `repeat` keydown and never presses a physical key twice while it is held; uppercase letters always carry Shift; with a 100% typo rate the final value is still correct, and typos are skipped on masked fields (inputs whose page script strips invalid characters, like card or IBAN masks), where a corrected typo would leave visible damage. ^[inferred]
- **No page-side footprint**: humanized clicks, hovers, frame actions and a strict-mode failure add no listeners to `window` or `document` and cause no DOM mutations, so a page that instruments `addEventListener` or a `MutationObserver` sees nothing from the engine. This overlaps with the CDP vector below. ^[inferred]
- **Persona-consistent shortcuts**: under a macOS persona, `navigator.platform` reads `MacIntel` and fill's select-all uses the persona's modifier, not the host's (see [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]]).

These are properties the suite asserts, not measurements against a commercial detector. ^[inferred] The full test inventory is on [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]].

## Relationship to other vectors

Behavioral signals overlap with [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP and automation detection]]: synthetic events and evaluate calls are visible both as behavior and as protocol traffic. Session-level consistency, such as a returning visitor with a stable seed, is covered by [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]]. ^[inferred]

## Open Questions

- Which commercial detectors weight which of these signals is not documented in the repository.
- Whether humanized timing remains statistically distinguishable from real users at scale (for example, identical presets across many sessions) is not addressed. ^[inferred]

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/README.md
- ~/code/CloakBrowser/cloakbrowser/human/mouse.py
- ~/code/CloakBrowser/cloakbrowser/human/keyboard.py
- ~/code/CloakBrowser/cloakbrowser/human/engine.py
- ~/code/CloakBrowser/js/tests/humanizeEngine.test.ts (commit b1a00ed, P2 pass)
