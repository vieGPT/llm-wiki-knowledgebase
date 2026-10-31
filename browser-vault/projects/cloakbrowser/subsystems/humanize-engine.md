---
title: >-
    Humanize engine
category: subsystems
tags: [humanize, playwright, cdp, behavioral-biometrics, stealth, visibility/internal]
aliases: [cloakbrowser.human, humanize=True, HumanEngine]
relationships:
  - target: "[[projects/cloakbrowser/techniques/humanized-input]]"
    type: implements
  - target: "[[projects/cloakbrowser/subsystems/humanize-errors]]"
    type: related_to
  - target: "[[projects/cloakbrowser/architecture/python-js-wrapper-parity]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: uses
sources:
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/engine.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/patch.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/world.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/injected.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/mouse.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/keyboard.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/fields.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/config.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/errors.py
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/engine.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/fields.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/index.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/injected.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/keyboard.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/mouse.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/patch.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/world.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human/worldHelpers.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/human-puppeteer/index.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/humanizeEngine.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/humanizeHarness.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/humanize.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/stealth.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/stealth.puppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/humanizePuppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_humanize_engine.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_humanize_unit.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_humanize_unit.mjs
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_human_visual.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_human_visual.mjs
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_resolver_sources_match.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth_dom_browser.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth_unit.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth_reproduction_110.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Human/HumanEngine.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Human/InjectedSource.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Wrappers/HumanCursor.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/HumanEngine/HumanEngineBrowserTests.cs
summary: >-
    The cloakbrowser.human package (mirrored in TS and C#): one pipeline (resolve, wait, wheel-scroll, curved move, hit-check, press/type) replacing Playwright actions when humanize=True.
provenance:
  extracted: 0.84
  inferred: 0.14
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T06:54:01+01:00
---

# Humanize engine

The humanize engine is the wrapper-side behavior layer that `humanize=True` turns on in `launch()` / `launch_async()`. It patches Playwright's sync and async APIs at class level so every page, frame, locator, element-handle, mouse and keyboard action on a humanized page runs through one async implementation (`engine.py`). Elements are resolved by Playwright's own selector engine inside a CDP isolated world, and input is produced as wheel bursts, curved pointer strokes and timed key presses. The engine never falls back to Playwright's stock action; failures raise Playwright `Error` / `TimeoutError` with a call log. The behavioral models it drives are described in [[projects/cloakbrowser/techniques/humanized-input|Humanized input]]; the detection side is [[projects/cloakbrowser/detection-vectors/behavioral-analysis|Behavioral analysis]].

The engine lives entirely in the wrapper, not in the patched Chromium binary, so it only applies when the wrapper drives the browser (see [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]] and [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]]).

## Module map

| Module | Role |
|---|---|
| `config.py` | `HumanConfig` dataclass with every timing parameter; presets `default` and `careful`; `resolve_config` (preset + overrides) and `merge_config` (per-call overrides, never mutates the base); log-normal samplers |
| `engine.py` | `Human` class: target resolution, state waits, retry/deadline loop, scroll, pointer, keyboard, field-specific actions; `_Typist` key timeline |
| `world.py` | `Worlds`: one CDP isolated world per frame and document, frame mapping, cross-frame geometry, helper library (`__cloakH`), call bounding (`bounded`, `PageStalled`) |
| `injected.py` | Finds Playwright's InjectedScript source in the installed driver and builds the install expression (global `__cloakInjected`) |
| `mouse.py` | Stroke planning (Fitts' law timing, Bézier path, overshoot plus corrective stroke), click aim points, idle drift |
| `keyboard.py` | Key timing (`key_gap`, `key_hold`), QWERTY neighbour table for typos, shift-symbol CDP codes, legacy `human_type` |
| `fields.py` | Fill-target classification mirroring Playwright's `InjectedScript.fill`; native date/time editor segmentation |
| `patch.py` | Class-level wiring into Playwright, signature validation, `human_config` kwarg, escape hatches, browser/context hooks |
| `errors.py` | Legacy error classes kept importable; see [[projects/cloakbrowser/subsystems/humanize-errors\|Humanize errors]] |

## Action pipeline

Every pointer action follows the same sequence (`pointer_action`):

1. **Resolve** the target in the frame's isolated world with Playwright selector semantics and strict mode. `internal:control=enter-frame` segments are split so frame locators descend through `<iframe>` owners.
2. **Wait for states.** Click waits for visible, enabled, stable; hover for visible, stable; fill/type for visible, enabled, editable; focus for visible, enabled.
3. **Scroll into view** with whole-notch mouse-wheel bursts (page, scrollable containers and iframes), moving the cursor into the scrolled area first.
4. **Aim** at a point inside the visible part of the element (input fields get a left-biased point).
5. **Hit-target check** at the aim point, for the element and every ancestor `<iframe>` owner. An interceptor produces a retry with "`<desc>` intercepts pointer events".
6. **Move** the cursor along a planned stroke, holding any requested modifiers during travel.
7. **Re-check** that the element is still connected and still hit under the cursor. A target replaced while the cursor travelled is never pressed, even with `force`; locators re-resolve and run the whole move again.
8. **Press** (click hold, double-click sequence) or **type**.

Retries back off 0, 20, 100, 100, 500 ms. All sub-steps of an action, including nested sub-actions such as the click inside a fill, share one deadline taken from the call's `timeout`, else the page default; `timeout=0` means no limit. On expiry the engine raises `TimeoutError` with a call log of the distinct retry reasons. `force`, `trial`, `position`, `button`, `click_count`, `modifiers`, `delay` and `strict` keep their Playwright meaning.

Focusing is done by clicking. If an element cannot be brought on screen at all, the click times out with "element is outside of the viewport" rather than silently calling programmatic focus. `type` continues at the end of existing text (End or Ctrl+End after the click). `fill` clears by select-all plus Backspace, and falls back to repeated Backspace for widgets that ignore select-all.

## Isolated worlds and the injected selector engine

- `Worlds` creates one isolated world per frame per document with `Page.createIsolatedWorld`. Out-of-process iframes get their own CDP session; same-process frames are mapped to CDP frame ids through `Page.getFrameTree` only.
- Iframe owner elements are reached with `DOM.getFrameOwner` plus `DOM.resolveNode` directly in the parent frame's isolated world, which is how cross-frame offsets, clips and owner hit checks are computed.
- Elements live in a world-local registry and are addressed by integer ids. A missing id surfaces as a stale-element signal that the retry loop treats as "detached, retrying".
- Worlds are dropped on `framenavigated` / `framedetached`; a stale context ("Execution context was destroyed" and similar) recreates the world once. An isolated-world failure raises `StealthWorldError` and never falls back to evaluating in the page's main world.
- Into each world the engine evaluates Playwright's **InjectedScript**, extracted as a string literal from the installed driver (`coreBundle.js` on Playwright ≥ 1.55, `generated/injectedScriptSource.js` on older layouts) and cached per driver path. Selector syntax (role, label, text, test-id, `>>` chains, `has` / `has-text`, `nth`, `visible`) therefore matches the client exactly, which fixed selector mismatches tracked as #522.
- The install step disables InjectedScript's global-listener-removal detection and hit-target interceptor hooks because both register listeners or dispatch events on the shared `window`, which page scripts could observe. ^[inferred]
- The world name is empty and globals are invisible to page scripts, so `__cloakInjected` and `__cloakH` only need to avoid self-collisions.

`ElementHandle` targets carry no identity the isolated world can read, so the engine locates the element whose border box exactly matches the handle's box (`matchRect`). If two elements share the box, actions raise an error that recommends a `Locator`; reads and `dispatch_event` instead run Playwright's own call (`RawFallback`).

## Wiring into Playwright (`patch.py`)

- Patching is class-level and idempotent across `Page`, `Frame`, `Locator`, `ElementHandle`, `Mouse` and `Keyboard`, for the sync and async generated APIs. Each wrapper checks whether its page carries a `Human` instance (`_cloak_human`) and otherwise calls the original method, so un-humanized pages in the same process are unaffected.
- Arguments are bound against the original method's signature, so typos still raise `TypeError`; `timedelta` values become milliseconds; a `human_config` keyword is accepted for per-call overrides.
- Humanized methods include click, dblclick, hover, tap, fill, clear, type, press, press_sequentially, check/uncheck/set_checked, select_option, focus, drag_and_drop / drag_to, scroll_into_view_if_needed, the reads (`input_value`, `text_content`, `inner_text`, `inner_html`, `get_attribute`) and `dispatch_event`. Raw `Mouse.move/click/dblclick` and `Keyboard.type` are humanized; mouse down/up/wheel and keyboard down/up/press/insert_text pass through. Reads and dispatches run in the same isolated world as the rest of the engine.
- `_check_playwright` fails loudly below Playwright 1.53 or when the internals the engine relies on (`Frame._timeout`, `ElementHandle._frame`) are missing, instead of letting a humanized page silently fall back to stock actions.
- `patch_context` / `patch_browser` (and async variants) install class patches before any page exists, then wrap `new_page` / `new_context` and the context `page` event. The code comment gives the reason: wrappers that capture bound methods at page creation, such as the license guard, must see the humanized methods.
- Escape hatches: `page._original` exposes raw Playwright calls (actions, reads, mouse and keyboard primitives), and `page._stealth_world.evaluate(expr)` evaluates in the main frame's isolated world.

## Timeouts and stalled pages

Each patched action sets an `Action` context variable holding its timeout. Every browser call made during the action goes through `bounded()`, which counts only the time a single call waits for an answer, not the pauses between calls. If the page stops answering (for example an `onchange` running a synchronous XHR, issue #619), the call raises `PageStalled`, which the patch layer turns into Playwright's `TimeoutError` ("waiting for the page to respond"). Cleanup calls made while unwinding, such as key and mouse releases, are still sent but not awaited, so a key or button is never left stuck down.

## Field handling

`fields.py` mirrors Playwright's `InjectedScript.fill` rules (playwright-core 1.63), so the humanized layer accepts and rejects the same targets:

- Text-like inputs (`text`, `search`, `email`, `number`, `password`, `tel`, `url`, untyped), `<textarea>` and contenteditable are typed key by key.
- Value inputs (`date`, `time`, `datetime-local`, `month`, `week`, `range`, `color`) are operated like a person: date/time editors are clicked on the first segment and typed segment by segment in whatever order the locale shows; sliders are clicked near the target value and nudged with arrow keys; `<input type=color>` is refused because it opens a native OS colour dialog that page input cannot drive.
- Everything else (checkbox, file, hidden, radio) raises Playwright's "cannot be filled" message. Non-numeric text for `type=number` is rejected as in Playwright.
- `select_option` opens a dropdown with a click and reaches the option with type-ahead or arrow keys plus Enter; list boxes get option clicks, adding with Ctrl/Cmd held. A pick that navigates the page reports what was chosen, as Playwright does.

## Persona awareness

The engine reads `navigator.platform` in the isolated world, so the browser persona rather than the host OS decides Meta vs Control for select-all and `ControlOrMeta`, and the wheel notch size (macOS 40 px, Windows 100 px, Linux 120 px). The dropdown path is one exception: it checks the host's `sys.platform`, because a macOS host's native popup ignores arrow keys and the whole label is typed instead. Persona selection itself is covered in [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]].

## Configuration

- Two presets: `default` and `careful`, which is slower everywhere and enables idle micro-movements between actions.
- `human_config` overrides at launch, and per call through the `human_config` keyword on any humanized method.
- Settings removed when moves switched to Fitts-law timing and scrolling to whole notches (`mouse_steps_divisor`, `mouse_burst_size`, `mouse_burst_pause`, `scroll_delta_base`, `scroll_delta_variance`) are dropped with a `FutureWarning` that points at the caller's frame, so old configs keep loading.

## TypeScript port (`js/src/human/`)

These notes come from the B4 batch, which read `engine.ts`, `config.ts`, `fields.ts` and `errors.ts` (commit b1a00ed).

- `engine.ts` calls itself a "Port of cloakbrowser/human/engine.py" and implements the same pipeline: resolve in the isolated world, wait for states, wheel-scroll, Bézier move, hit-target check for the element and ancestor iframes, then press or type. It uses the same state sets (click: visible, enabled, stable; hover: visible, stable; input: visible, enabled, editable; focus: visible, enabled) and the same retry backoff of 0, 20, 100, 100, 500 ms. One `Deadline` object is shared by nested sub-actions (`_deadline`, `_nested`). A `RawFallback` error routes reads and `dispatchEvent` calls on ambiguous handles to Playwright's own call.
- The host-platform exception in the dropdown path is mirrored as well: `engine.ts` checks `process.platform === 'darwin'` because "a macOS host's native popup ignores arrow keys". Every other decision follows the persona's `navigator.platform`.
- `config.ts` keeps the **snake_case keys** of the Python dataclass and identical preset values, so one `human_config` object works in both languages. The values follow published research:
  - **Typing** (136M Keystrokes study, Dhakal et al., CHI 2018): a key is held about 100 ms and the median press-to-press gap is about 150 ms. Key holds and gaps are sampled log-normally with configured ~5th/95th percentiles, and Shift is usually released before the letter.
  - **Mouse**: move duration follows Fitts' law, `160 + 75·log2(1 + d/8)` ms with a 0.3 log-normal spread. Pointer events go out at about 60 Hz. There is a 35% chance of an overshoot followed by a corrective stroke.
  - **Scroll**: whole wheel notches, with `scroll_notch_px: 0` meaning the persona platform's native value.
  - **Initial cursor**: x 400 to 700, y 45 to 60, "as if coming from the address bar area".
  - **Careful preset**: typing 210 ms, `mistype_chance` 0.05 and idle-between-actions switched on.
- `fields.ts` is a "Port of cloakbrowser/human/fields.py" with the same `TYPE_INTO`, `SET_VALUE` and `NO_MISTYPE` sets. Typos are never simulated in number, email, password, tel or url fields, where a correction could be unsafe.
- No counterpart to the Python `FutureWarning` shim for removed config keys was found in `config.ts` (as of commit b1a00ed). ^[ambiguous]
- JS-only: the `human-puppeteer/` tree (`index.ts`, `keyboard.ts`, `scroll.ts`) provides humanize for the Puppeteer backend, and `worldHelpers.ts` holds the helpers injected into the isolated world. See [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].

### Wiring, worlds and input primitives in TypeScript (B5 batch)

These notes compare `patch.ts`, `world.ts`, `worldHelpers.ts`, `injected.ts`, `mouse.ts` and `keyboard.ts` with the Python design above (commit b1a00ed).

- **Prototype patching through live instances.** playwright-core does not export its client classes, so `patch.ts` reaches the prototypes through a real page: the page itself, `page.mainFrame()`, `page.locator('html')` (no protocol call), `page.mouse` and `page.keyboard`. `ElementHandle` is only reachable from an instance, so the wrapper hooks `Connection._createRemoteObject` and patches the prototype of the first handle created. A missing hook is an error, because otherwise `handle.click()` would silently run Playwright's stock action.
- **Same version floor and fail-loud checks** as Python: `MIN_PLAYWRIGHT = 1.53.0`, plus checks for `Frame._timeout` and `Connection._createRemoteObject` (Python checks `Frame._timeout` and `ElementHandle._frame`).
- **Per-call action context.** `world.ts` uses Node's `AsyncLocalStorage` (`ACTION`) where Python uses a context variable. `bounded()` and `PageStalled` work the same way (#619). Timeouts beyond `setTimeout`'s 2^31−1 ms range are treated as no limit.
- **Frame mapping.** `matchChild` pairs Playwright child frames with CDP child frames by URL and position within same-URL groups, never by name, because page JavaScript can change `window.name` and Playwright never refreshes it.
- **Date/time editor segments** are read through CDP's DOM domain with `pierce` from the input's closed user-agent shadow root, so no script runs in the page.
- **One helper library for both languages.** `worldHelpers.ts` is a byte-identical copy of Python's `_HELPERS_JS`, enforced by `tests/test_resolver_sources_match.py`. Notable helper rules:
  - boxes for `display: contents` elements are the union of what they render;
  - an `ElementHandle` whose box is shared by a nested wrapper chain resolves to the deepest focusable element (where a real click would land), else the outermost;
  - overflow clipping follows the containing-block chain rather than DOM ancestors, so fixed and absolute boxes escape scrollers correctly (#593);
  - dropdown type-ahead planning mirrors Blink's TypeAhead: shortest unambiguous prefix, then arrow presses, with Home plus arrows when no prefix is unique.
- **`injected.ts`** matches `injected.py`: it extracts the InjectedScript literal from `coreBundle.js` (≥ 1.55) or `injectedScriptSource.js`, installs it under `__cloakInjected` with `isUtilityWorld: true`, and stubs the two listener hooks.
- **Mouse model (`mouse.ts`).** Each move is a fast main stroke whose speed peaks at 22-38% of the stroke time, followed by a slow approach. The profile is a time-warped minimum-jerk curve, citing Meyer's optimised sub-movements and Plamondon's log-normal strokes. Both Bézier control points usually bend the same way ("a hand sweeps an arc, it rarely draws an S"). A low-frequency sideways drift is zero at both ends, and no event is emitted without at least a pixel of movement. Overshoot happens only on moves over 40 px, with 30% of misses landing short. The corrective stroke follows a 40-140 ms pause. Click points fall in the middle 35-65% of buttons; inputs use a configured horizontal range and 30-70% vertically.
- **Keyboard model (`keyboard.ts`).** `gapFactor` encodes the Dhakal et al. findings: a letter typed with the other hand follows faster (×0.85), same hand slower (×1.1), a repeated letter ×1.05, and the first letter after a space slowest (×1.3). Thinking pauses are ten times likelier at word starts. Gaps never go below 15 ms. The `Typist` timeline allows rollover (next key down before the previous key is up), never presses a physical key twice while held ("x"/"X", "1"/"!"), and releases Shift around the letter's release. Non-ASCII text arrives as one `insertText`, like an IME. Digit typos map only to other digits so a correcting Backspace cannot eat a real digit in `type=number` (#573).
- **Shift symbols** go through CDP `Input.dispatchKeyEvent` with Windows virtual key codes, producing trusted events with no evaluate stack frame. The legacy fallback without a CDP session dispatches a synthetic `KeyboardEvent` through `page.evaluate`, which the source itself labels detectable (`isTrusted=false`).
- **Puppeteer port.** `human-puppeteer/` reuses `mouse.ts` and the `Typist` but has its own simpler page/ElementHandle patching, scroll and isolated-world code. It does not load Playwright's InjectedScript. See [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].

## .NET port (`dotnet/src/CloakBrowser/Human/`, batch D)

The C# engine is a file-by-file port of the same pipeline (commit b1a00ed). Full context is on [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

- **Same pipeline, same models.** `HumanEngine` (+ `HumanEngine.Actions.cs`) ports `engine.py`, `HumanWorld` ports `world.py`, `HumanMouse` ports the stroke planner (60 Hz frames, Fitts' law, corrective stroke), and `HumanKeyboard` ports the timing model and the `Typist` rollover timeline. `Fields.cs` ports the fill classification and date/time segmentation. `HumanConfig` keeps every default (typing 150 ms, mistype 0.04, `MouseTimeBase` 160, `MouseTimePerBit` 75, `ScrollNotchPx` 0 = persona default). Failures are `PlaywrightException` / `System.TimeoutException`, and a stall is an internal `PageStalledException` turned into a timeout.
- **InjectedScript from the NuGet driver.** `InjectedSource` extracts the literal from the `.playwright/package/lib` driver copy that Microsoft.Playwright places next to the app (or from `PLAYWRIGHT_DRIVER_SEARCH_PATH` or the NuGet cache). It reports `sdkLanguage: "csharp"`, stubs the same two listener hooks, and supports both the positional (< 1.47) and options-object constructors.
- **Internals by reflection.** A locator's frame and selector, the page's default timeout, and an element handle's frame are read from private Microsoft.Playwright fields (`_frame`, `_selector`, `_timeoutSettings`, `Parent`). There is no version gate like `_check_playwright`; a missing field throws "Microsoft.Playwright layout changed". Action options are also read by reflection, because the Playwright option classes share property names but no base type.
- **Frame locator quirk.** .NET joins frame-locator selector parts with an extra space (`enter-frame  >> `), so the split tolerates surrounding whitespace.
- **Wiring.** Instead of `patch.py`, Roslyn-generated decorators (`Wrappers/Humanized*.cs`) implement the Playwright interfaces and forward every member that is not humanized. `HumanCursor` holds one engine per page (virtual cursor, worlds, raw input), shared by the page, its frames, locators, mouse and keyboard. `HumanizedMouse` humanizes `MoveAsync`/`ClickAsync`/`DblClickAsync`, and `HumanizedKeyboard` humanizes `TypeAsync`; down/up/wheel/press/insert-text pass through, as in Python.
- **Compatibility leftovers.** `Actionability` (public typed checks, now running on the engine), `HumanScroll` (standalone wheel helper modelled on the JS Puppeteer scroller, with a live `innerWidth`/`innerHeight` fallback when a headed `NoViewport` page has no `ViewportSize`) and `IsolatedWorld` are kept for source compatibility. Humanized actions do not use them.
- **Tests.** `HumanEngineBrowserTests` (38 real-browser cases) is a declared port of `test_humanize_engine.py` / `humanizeEngine.test.ts` on the same fixture page. It covers curve-then-press, side-effect-free reads, stalls (#619), type-ahead on long and emoji dropdowns, the stale handle after navigation, and persona-following select-all. `TimeoutBudgetTests` pins the single-deadline arithmetic (#307), `SelectorParityBrowserTests` mirrors the selector-parity cases of `test_stealth_dom_browser.py`, and `NonAsciiKeyboardTests` ports Python's `TestNonAsciiKeyboard`. ^[inferred]

## Pinned by tests (B6, JS suite)

`js/tests/humanizeEngine.test.ts` replaced the earlier mock-based tests. It drives the real binary headless, with `humanize: true` and a near-zero-delay config (`FAST`), against a local fixture page served from 127.0.0.1 (`humanizeHarness.ts`). The fixture logs every input event and every DOM read that only a main-world evaluate would make. It is the same page the Python and .NET suites use, and the suite runs only when `CLOAKBROWSER_BINARY_PATH` is set. It pins the following:

- **One helper library.** The test reads `_HELPERS_JS` out of `cloakbrowser/human/world.py` and requires `HELPERS_JS` to be byte-identical, so the parity rule is enforced from the JS side as well.
- **InjectedScript is found** in the installed playwright-core (a literal of more than 10,000 characters that contains `InjectedScript`).
- **No page-visible footprint.** Humanized reads (`inputValue`, `textContent`, `innerText`, `innerHTML`, `getAttribute`) return correct values without the page's probe recording anything. A run of role, chained, ElementHandle, frame-locator and strict-mode actions adds zero `window`/`document` listeners and zero DOM mutations.
- **Raw path where needed.** Reads on hidden or same-box handles take Playwright's own call at once, under 5 s, and land on the handle's own element rather than its same-size wrapper. `dispatchEvent` reaches page listeners with the same event shape, and a `JSHandle` (such as a `DataTransfer`) inside `eventInit` uses the raw path.
- **Trusted input only.** Filling date and range inputs and selecting an option produces zero untrusted `input`/`change` events. `<input type=color>` is refused ("native colour picker") and a checkbox "cannot be filled". An element that cannot be scrolled into view times out with "outside of the viewport" and is never focused programmatically.
- **Keyboard.** `press` focuses its target first and forwards `delay`. `press` and `pressSequentially` reach non-focusable targets such as a canvas or a document key handler. Typing `xX aA zZ 1! qQ` never produces a repeat `keydown` and never presses a physical key that is already held. Typos are corrected, skipped in masked fields, and uppercase letters, typos included, are typed with Shift (also on Puppeteer).
- **Scrolling.** A forced overshoot wheels past the target and back, while no overshoot wheels one way only. Five containing-block layouts check which scroller the engine moves (#593).
- **Dropdowns.** A 205-option `<select>` is set by type-ahead in at most 13 keys without ArrowDown, committing with one `input` plus `change` outside macOS (#581). Emoji labels are reached with arrows, and on a macOS host fail with "cannot be picked with human input". A selection that navigates the page, even from an iframe, still returns (#597, #619).
- **Stalled pages (#619).** A synchronous XHR in `onchange`, `onclick` or `oninput` turns the call into a `TimeoutError` within the budget ("waiting for the page to respond"), with one wait rather than one per held modifier, and leaves no modifier key down afterwards. Deliberate pauses, such as a 2.5 s field-switch delay against a 1 s timeout, and long fills do not count against the timeout.
- **Persona, not host.** Under `--fingerprint-platform=macos`, `navigator.platform` is `MacIntel` and fill's select-all still clears the field, using the persona's modifier.
- **Coexistence.** A non-humanized browser in the same process keeps Playwright's teleporting click. The humanized page exposes `_humanCfg`, `_stealth.evaluate()` (the JS name for the isolated-world escape hatch) and `_original`.

The mock-level files `humanize.test.ts` and `stealth.test.ts` add these points:

- `resolveConfig` throws on an unknown preset, and `careful` types no faster than `default`. `mergeConfig` returns a new object and never mutates the base.
- `matchChild` survives a frame renamed by page JavaScript and keeps attach order for same-URL siblings.
- `bounded()` treats a timeout beyond `setTimeout`'s range as no limit.
- `patchBrowser` on a CDP-connected browser patches pages in contexts that already exist as well as later ones (#126).

The Puppeteer port's tests are summarised on [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].

## Pinned by tests (B7 batch)

`tests/test_humanize_engine.py` replaced the older mock-based tests. It drives the real binary headless against a local recorder page that logs every input event, with a near-zero-delay `FAST` config, so it pins behaviour, not internals. `tests/test_humanize_unit.py` adds config, math and world-level unit tests (commit b1a00ed). The rules above are confirmed: curved move then press, Playwright option semantics, the `detail` 1-then-2 double-click sequence, modifiers held during a hover, "intercepts pointer events", strict mode, `timeout=0`, "outside of the viewport" with no programmatic focus, trusted `input`/`change` events for date, range and select, a refused colour input, and select-all following the persona (a `macos` persona reports `MacIntel`). Details the prose above leaves open:

- **ElementHandles are humanized.** Handles from `query_selector` and `query_selector_all` click along curves and type key by key. A handle from a previous document fails fast with "not attached". A same-document `pushState` keeps it valid.
- **Same-box handles.** A focusable wrapper sharing its box with a child receives the keys. A click on a block link wrapping a same-size image lands on the image, which is what a real click hits. Two unrelated elements stacked on one box raise "shares its box".
- **Raw fallbacks.** Reads on hidden (box-less) or same-box handles run Playwright's own read at once. `dispatch_event` with a JSHandle in `event_init` (Playwright's `DataTransfer` drag pattern) also runs Playwright's call, because that handle lives in the main world.
- **Non-focusable targets**: `press`/`type` on a `<canvas>` click it and send the keys anyway.
- **Select type-ahead**:
  - the plan does not depend on which row the open popup highlights (#580), uses at most 3 arrow presses and folds accents;
  - a 205-option list is reached in at most 13 keys with no ArrowDown walk (#581);
  - emoji labels are reached with arrows, and on a macOS host the pick is refused ("cannot be picked with human input");
  - a pick that submits the form returns normally (#597).
- **Stalls (#619)**: click, type and select time out with "waiting for the page to respond" after one timeout, not one more per held modifier. Every modifier is released afterwards. Human pauses and slow typing never count against the timeout. `bounded` has no limit outside an action or with `timeout=0`, and bounding does not suppress the idle between top-level actions.
- **Scrolling**: containing-block-aware clipping (#593). A fixed modal inside a scroller is clicked without scrolling, while clipped children are wheeled in. Horizontal overflow scrolls on the x axis (#521).
- **Frames**: iframe actions route into the sub-frame, including iframes attached after load (#428). Python `_match_child` survives page JavaScript renaming `window.name` and keeps attach order for same-URL siblings, mirroring `matchChild` in TS.
- **Isolation of un-humanized pages**: a `humanize=False` browser in the same process has no `_original` and behaves like stock Playwright.
- **`_check_playwright`** accepts a Nuitka-compiled Playwright (placeholder bytecode) but still flags missing internals ("missing internals: ElementHandle._frame").

The `.mjs` and `test_human_visual.*` scripts are manual: they run against `js/dist` and live sites (wikipedia.org, deviceandbrowserinfo.com) and draw a red cursor trail for visual inspection. They are not part of the pytest or vitest CI. `test_humanize_unit.mjs` asserts that frame methods carry a `Symbol.for('cloakbrowser.humanized')` marker and that `press` on an already focused field makes no mouse path. Its idle-config check passes `humanize_config`, while the JS option is `humanConfig` (`types.ts`), so that check only proves `_humanCfg` exists. ^[inferred]

## Pinned by tests (B8 batch)

- **Three hand-copied ports.** `tests/test_resolver_sources_match.py` checks that the `__cloakH` helper library is byte-identical in **three** places: Python `world.py`, `js/src/human/worldHelpers.ts` and the .NET `dotnet/src/CloakBrowser/Human/WorldHelpers.cs`. Nothing generates these copies. The test also bans characters that would end one of the host string literals (a backtick or `${` for the TypeScript `String.raw`, a triple quote for the Python and C# raw strings), as well as carriage returns and trailing whitespace, which would cause invisible drift. See [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].
- **Selector parity with Playwright** (`tests/test_stealth_dom_browser.py`, real browser, `slow`). Humanized clicks must land on the element Playwright's own locator resolves:
  - text selectors skip a hidden `<head>` script that contains the same text (#512);
  - a box-less `display: contents` target uses the geometry of what it renders;
  - text regex, exact fragments, input values and whitespace normalization match Playwright;
  - `:has-text` keeps its compound position and pierces open shadow roots;
  - first/nth order across light DOM and nested shadow roots matches.
- **Reimplemented `internal:*` engines.** The docstring states the stakes: under-matching gives a typed error, but over-matching "silently clicks the wrong coordinates". The test pins these semantics:
  - `get_by_test_id` uses strict equality;
  - `get_by_placeholder` with the `i` flag is a case-insensitive substring match;
  - attribute values are compared raw, without trimming;
  - `internal:text` exact compares the full normalized subtree text;
  - `<br>` adds nothing to element text, and shadow-root text joins the host's;
  - labels resolve in the order `aria-labelledby`, `aria-label`, then `.labels`.
- **Actionability parity.** Visibility, native `disabled` and ARIA `readonly` match Playwright. `force` clicks through a covering element without changing which element is the target.
- **Target swapped mid-move.** The press never lands on whatever is under the cursor after the move. A removed target times out. A target replaced in the DOM is re-resolved and reached with a fresh move, as Playwright re-resolves locators. An async `ElementHandle` whose element is replaced fails, because handles are not re-resolved, and nothing is clicked.
- **Stealth infrastructure** (`tests/test_stealth_unit.py`, `tests/test_stealth_reproduction_110.py`): evaluate-free DOM queries, trusted shift symbols over CDP and re-creating the world after navigation are covered on [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP and automation detection]].

## Open Questions

- Resolved by tests: the README warning that `ElementHandle` objects from `page.query_selector()` "bypass all patches" is stale. Both test suites pin humanized handle clicks, typing, fills and hovers, matching `patch.py`.
- The TypeScript mirror matches the pipeline, presets, field rules and the host-platform dropdown check (see "TypeScript port" above). `patch.ts`, `world.ts`, `worldHelpers.ts`, `mouse.ts` and `keyboard.ts` were compared in the B5 batch (see above); the remaining Python-only difference noted is which internals `install` checks. See [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/engine.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/patch.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/world.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/injected.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/mouse.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/keyboard.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/fields.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/config.py
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/human/errors.py
- repo:github.com/CloakHQ/CloakBrowser/README.md
- repo:github.com/CloakHQ/CloakBrowser/js/src/human/engine.ts, js/src/human/config.ts, js/src/human/fields.ts (commit b1a00ed, B4 batch)
- repo:github.com/CloakHQ/CloakBrowser/js/src/human/index.ts, injected.ts, keyboard.ts, mouse.ts, patch.ts, world.ts, worldHelpers.ts and js/src/human-puppeteer/index.ts (commit b1a00ed, B5 batch)
- repo:github.com/CloakHQ/CloakBrowser/js/tests/humanizeEngine.test.ts, humanizeHarness.ts, humanize.test.ts, stealth.test.ts, stealth.puppeteer.test.ts, humanizePuppeteer.test.ts (commit b1a00ed, B6 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_humanize_engine.py, tests/test_humanize_unit.py, tests/test_humanize_unit.mjs, tests/test_human_visual.py, tests/test_human_visual.mjs (commit b1a00ed, B7 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_resolver_sources_match.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth_dom_browser.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth_unit.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Human/ (HumanEngine.cs, HumanEngine.Actions.cs, HumanWorld.cs, InjectedSource.cs, HumanMouse.cs, HumanKeyboard.cs, HumanConfig.cs, HumanScroll.cs, Actionability.cs, Fields.cs), dotnet/src/CloakBrowser/Wrappers/ (HumanCursor.cs, HumanizedMouse.cs, HumanizedKeyboard.cs), dotnet/tests/CloakBrowser.Tests/HumanEngine/HumanEngineBrowserTests.cs, Human/TimeoutBudgetTests.cs, Human/SelectorParityBrowserTests.cs, Human/NonAsciiKeyboardTests.cs (commit b1a00ed, batch D)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth_reproduction_110.py (commit b1a00ed, B8 batch)
