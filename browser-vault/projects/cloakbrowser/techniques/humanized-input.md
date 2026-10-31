---
title: >-
    Humanized input
category: techniques
tags: [humanize, behavioral-biometrics, mouse-dynamics, keystroke-dynamics, stealth, visibility/internal]
aliases: [human-like input, Bezier mouse, human typing]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/behavioral-analysis]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: uses
sources:
  - ~/code/CloakBrowser/cloakbrowser/human/mouse.py
  - ~/code/CloakBrowser/cloakbrowser/human/keyboard.py
  - ~/code/CloakBrowser/cloakbrowser/human/engine.py
  - ~/code/CloakBrowser/cloakbrowser/human/config.py
  - ~/code/CloakBrowser/cloakbrowser/human/fields.py
  - ~/code/CloakBrowser/cloakbrowser/human/world.py
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/js/src/human/mouse.ts
  - ~/code/CloakBrowser/js/src/human/keyboard.ts
  - ~/code/CloakBrowser/js/src/human-puppeteer/keyboard.ts
  - ~/code/CloakBrowser/js/src/human-puppeteer/scroll.ts
  - ~/code/CloakBrowser/js/tests/humanize.test.ts
  - ~/code/CloakBrowser/js/tests/stealth.test.ts
  - ~/code/CloakBrowser/js/tests/stealth.puppeteer.test.ts
  - ~/code/CloakBrowser/js/tests/humanizeEngine.test.ts
  - ~/code/CloakBrowser/js/tests/humanizePuppeteer.test.ts
  - ~/code/CloakBrowser/tests/test_humanize_unit.py
  - ~/code/CloakBrowser/tests/test_humanize_engine.py
summary: >-
    How CloakBrowser makes automated input look human: Fitts-law Bézier strokes at 60 Hz, log-normal key timing with rollover and typos, whole-notch wheel scrolling, focus by click.
provenance:
  extracted: 0.73
  inferred: 0.24
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Humanized input

Humanized input replaces Playwright's instant, synthetic-looking input (teleporting cursor, zero-duration clicks, value injection, jump scrolling) with input modelled on published human motor-control and typing data. It is implemented by the [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]] and targets what [[projects/cloakbrowser/detection-vectors/behavioral-analysis|Behavioral analysis]] measures. The numbers below are the `default` preset (as of 26/10/2026, commit b1a00ed); `careful` is slower across the board.

## Pointer movement

The model is stated in `mouse.py`: a pointing movement is a fast main stroke whose speed peaks early, then a slow approach, often finished by a short corrective stroke (Meyer's optimised sub-movements, Plamondon's log-normal strokes).

- **Duration** follows Fitts' law: median = `mouse_time_base` (160 ms) + `mouse_time_per_bit` (75 ms) × log2(1 + distance / 8), with log-normal spread (σ 0.3), so long moves take longer and no two moves of the same length take the same time.
- **Path** is a cubic Bézier curve with control points offset perpendicular to the straight line. 80% of the time both control points bend the same way, because "a hand sweeps an arc, it rarely draws an S". A low-frequency sideways wobble (≤ 1.5 px, zero at both ends) is added on top.
- **Velocity profile** is a time-warped minimum-jerk curve whose speed peaks at 22–38% of the stroke time: fast early, long slow tail.
- **Event rate**: one pointer event per display frame (~60 Hz), clamped to 2–200 events per stroke, with slight frame jitter. Consecutive events always differ, because a real mouse does not report a move without movement. The final event lands exactly on the target.
- **Overshoot and correction**: for moves over 40 px, 35% of strokes land slightly off target (a few % of the distance plus 2–6 px, 30% of those short of the target, slightly off-line), pause 40–140 ms, then a short Fitts-timed corrective stroke finishes the move.
- **Schedule keeping**: the engine subtracts each CDP round trip from the next wait, so the planned timing is not stretched by protocol latency.
- **Starting point**: the cursor starts near the address-bar area (x 400–700, y 45–60), as if coming from the browser chrome, rather than at (0, 0).
- **Idle drift** between actions (±3 px jitter returning to the rest point) is opt-in and enabled by the `careful` preset.

Why it works: detectors that look for straight lines, constant velocity, instantaneous jumps, perfectly centred clicks or duplicate-coordinate events see none of them; the Fitts-law timing keeps duration statistically consistent with distance. ^[inferred]

## Clicks

- Aim points are randomized inside the element's visible part. Buttons get a centred cluster (35–65% on both axes); text inputs get a left-biased x (5–30% of width), where people click to place a caret.
- Before pressing there is an aim delay (inputs 60–140 ms, buttons 80–200 ms), and mouse-down is held 40–100 ms (inputs) or 60–150 ms (buttons). Double-clicks emit a real `detail=1,2` sequence with a 60–140 ms gap.

## Keyboard

Defaults follow the 136M Keystrokes study (Dhakal et al., CHI 2018): keys are held about 100 ms, about 150 ms pass between key presses, and the next key is often pressed before the previous one is released.

- **Inter-key gap** is log-normal around `typing_delay` (150 ms, spread 70 ms), scaled by the letter pair: other hand ×0.85, same hand ×1.1, repeated letter ×1.05, after a space ×1.3, before a space ×0.9. A "thinking pause" (300–1000 ms) comes before 15% of words and at a tenth of that rate inside words. Minimum gap 15 ms.
- **Key hold** is log-normal with 60–175 ms as its ~5th/95th percentiles.
- **Rollover**: the `_Typist` timeline schedules presses by press-to-press gaps and queues releases by each key's own hold time, so the next key often goes down before the previous one comes up. A physical key is never pressed again while still held ("x" then "X", "1" then "!").
- **Shift**: goes down 60–220 ms before the letter and usually comes up just before the letter is released (−60 to +30 ms). Consecutive capitals keep Shift held; a lowercase letter releases it first.
- **Typos**: 4% of ASCII letters and digits are first typed as a QWERTY neighbour (digits only become neighbouring digits, issue #573), noticed after 100–300 ms, removed with Backspace, and corrected. No typos are simulated in number, email, password, tel or url fields, where a correction is unsafe or a person types carefully. If the page rejected the wrong key (input masks), nothing is "corrected", so a real character is never deleted.
- **Non-ASCII** (Cyrillic, CJK, emoji) has no physical key and is inserted with `insertText` one character at a time, like an IME.
- **Shift symbols** (`@ # ! $ …`) are sent through CDP `Input.dispatchKeyEvent` with the Shift modifier and the physical key code, producing `isTrusted=true` events and no script frame on the page. The legacy `human_type` helper still has a fallback that dispatches synthetic `KeyboardEvent`s through `page.evaluate`; its own docstring calls that path detectable (`isTrusted=false` plus an evaluate stack frame). The unified engine always uses the CDP path.
- `keyboard.press` gets a human key-down time; Playwright's own press releases in the same instant.

## Scrolling

- The wheel turns in **whole notches**: one wheel event per notch, sized to the persona's platform (Windows 100 px, macOS 40 px, Linux 120 px; `wheelDelta` 120 per notch). Code comments explain that CDP reports one notch per wheel event, so only whole-notch events keep `deltaY` and `wheelDeltaY` consistent the way a real wheel does.
- A logical scroll is a series of **flicks** of 2–5 notches with short pauses (30–80 ms) inside a flick, slower notches (80–200 ms) at its start and end, and a finger-lift pause between flicks.
- 10% of scrolls overshoot by 50–150 px, pause, and wheel back in one or two flicks.
- The cursor first moves into the area being scrolled, since a wheel scrolls what is under the pointer. Main-document targets settle in a band 20–80% down the viewport, but only if the page can still scroll that way.
- When whole notches would jump back and forth over a target with less slack than one notch, the engine finishes with one exact-remainder wheel event, as a free-spinning or high-resolution wheel would send.

## Focus, values and reads

- No programmatic focus: fields are focused by scrolling to them and clicking.
- Value inputs are operated like a person (date segments typed, sliders clicked then arrow-keyed, dropdowns opened and navigated), producing trusted events instead of a programmatic value set. `fill()` in plain Playwright sets values without keyboard events, which the README says reCAPTCHA's behavioral analysis flags.
- Element reads and `dispatch_event` run in a CDP isolated world, not through `page.evaluate` in the main world. ^[inferred] This keeps the engine's own DOM queries out of the page's JavaScript realm, where prototype hooks or stack inspection could observe them; relevant to [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP and automation detection]].

## JavaScript and Puppeteer

- The TypeScript files `js/src/human/mouse.ts` and `keyboard.ts` implement the same models with the same constants: speed peak at 22-38%, overshoot only above 40 px with 30% short, letter-pair gap factors ×0.85/×1.1/×1.05/×1.3/×0.9, 15 ms minimum gap, and digit-only digit typos (#573). They are shared by the Playwright and Puppeteer backends (as of commit b1a00ed).
- The Puppeteer edition (`human-puppeteer/`) reuses the stroke planner and the `Typist` timeline but has its own scroll code. `pageNotchPx` reads `navigator.platform` through the isolated world, falling back to `page.evaluate`, and caches it per page. `smoothWheel` turns the wheel in flicks of 2-5 whole notches. `page.mouse.wheel(distance)` is rounded to the nearest whole number of notches, at least one. Native `ElementHandle.scrollIntoView()` (Puppeteer v22+) is replaced by the same wheel sequence, because an instant snap-scroll is "a strong bot signal". See [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].
- With a null viewport (headed Puppeteer), scroll geometry falls back to the live window size, so humanize still works in the headed mode the README recommends for stealth.

## Pinned by tests (B6, JS suite)

The JS tests (commit b1a00ed) turn several of these models into checked properties:

- **Strokes.** A 500×300 px move at the default preset emits at least 10 intermediate points with one `move` call per step. It ends within 10 px of the target, and no single step covers 50% or more of the distance. A purely horizontal move bends off the line in at least one of ten trials. Very short and zero-length moves do not crash.
- **Aim points.** They always fall inside the bounding box, also for a 1×1 box. For inputs the average x sits clearly left of centre.
- **Keys.** Cyrillic, CJK and emoji go through `insertText`, and only ASCII produces key-down events. Typos are simulated for ASCII only. With a 100% typo rate, numeric text still yields only digit key-downs on both the Playwright and Puppeteer keyboards (#573).
- **Shift symbols.** Each one is a CDP `Input.dispatchKeyEvent` sequence with modifier flag 8 (Shift held), never `page.evaluate`. Key-ups carry no `text` fields, and the digit-row symbols use virtual key codes 48-57. Only without a CDP session does the legacy path run, sending `insertText` before its `page.evaluate` dispatch.
- **Live checks.** These run only with `SLOW=1` and use real sites. A humanized `!` arrives as a trusted `keydown`. A click causes no `querySelector` call from an evaluate stack. A login form on deviceandbrowserinfo.com filled with shift symbols shows zero evaluate leaks and zero untrusted keys, and its verdict contains neither `superHumanSpeed: true` nor `suspiciousClientSideBehavior: true` (results not dated in the test).
- **Puppeteer wheel.** `smoothWheel` sends exactly one platform notch per event (tested at 40, 100 and 120 px), never a zero step, and stops within half a notch of the requested distance. The Puppeteer path therefore rounds and does not end with the exact-remainder wheel event described above for the main engine. ^[inferred]

## Pinned by tests (B7 batch)

`tests/test_humanize_unit.py::TestHumanTiming` checks the sampled distributions "against human data (136M Keystrokes, Dhakal et al. CHI 2018; Fitts' law; SapiMouse)" (commit b1a00ed):

- Key hold: median between 85 and 120 ms, with under 1% of presses shorter than 40 ms.
- Inter-key gap: other hand < same hand < first letter after a space, by median.
- Mouse duration: a 1,200 px stroke takes longer than a 100 px one but less than twice as long (Fitts), and same-distance durations spread with a coefficient of variation above 0.2.
- Velocity: the speed peak falls before 40% of the stroke time by median ("people: ~0.3; a symmetric ease: 0.5").
- Every planned point moves at least one pixel, and the last point lands exactly on the target.

The real-browser suite (`tests/test_humanize_engine.py`) adds:

- key holds median 70 to 140 ms with rollover observed, and no physical key pressed twice while held ("xX", "1!" never produce `repeat=true`);
- every wheel event's `deltaY` is exactly one persona notch (40, 100 or 120 px), opposite in sign to `wheelDeltaY`, and a forced overshoot wheels back;
- typos are corrected, input-masked fields end with the intended value, and capitals keep Shift held;
- no `mousemove` reports zero movement.

The unit tests also pin non-ASCII input through `insertText` with typos limited to ASCII, digit neighbours that are digits only (#573), and `careful` being at least as slow and as error-prone as `default`.

Live behavioural checks are marked `slow` and run by hand. On deviceandbrowserinfo.com's interaction test, `superHumanSpeed` and `suspiciousClientSideBehavior` must not be true. Filling e-mail and password and submitting must take over 3 s, and a single `fill` over 1 s. See [[projects/cloakbrowser/synthesis/detection-service-test-matrix|Detection service test matrix]].

## Limits

- Humanization is wrapper-level. A browser driven over CDP by another tool gets the binary's fingerprint patches but not humanized input unless that tool's script imports the patch functions (README). See [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]].
- Human-like timing costs latency: each `fill` waits a field-switch delay of 800–1500 ms before acting.

## Open Questions

- Whether the 60 Hz pointer cadence matches high-refresh displays that a persona's screen might imply. ^[inferred]
- No measured detection results for the behavioral model in this batch beyond the README claim that deviceandbrowserinfo.com reports "You are human!" with 24/24 signals passed (as of 26/10/2026).

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/cloakbrowser/human/mouse.py
- ~/code/CloakBrowser/cloakbrowser/human/keyboard.py
- ~/code/CloakBrowser/cloakbrowser/human/engine.py
- ~/code/CloakBrowser/cloakbrowser/human/config.py
- ~/code/CloakBrowser/cloakbrowser/human/fields.py
- ~/code/CloakBrowser/cloakbrowser/human/world.py
- ~/code/CloakBrowser/README.md
- ~/code/CloakBrowser/js/src/human/mouse.ts, js/src/human/keyboard.ts, js/src/human-puppeteer/keyboard.ts, js/src/human-puppeteer/scroll.ts (commit b1a00ed, B5 batch)
- ~/code/CloakBrowser/js/tests/humanize.test.ts, stealth.test.ts, stealth.puppeteer.test.ts, humanizeEngine.test.ts, humanizePuppeteer.test.ts (commit b1a00ed, B6 batch)
- ~/code/CloakBrowser/tests/test_humanize_unit.py, tests/test_humanize_engine.py (commit b1a00ed, B7 batch)
