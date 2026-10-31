---
title: >-
    Playwright vs Puppeteer backends
category: architecture
tags: [playwright, puppeteer, architecture, cdp]
aliases: [cloakbrowser/puppeteer, puppeteer.ts, playwright.ts]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: uses
  - target: "[[projects/cloakbrowser/architecture/python-js-wrapper-parity]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/js/src/index.ts
  - ~/code/CloakBrowser/js/src/playwright.ts
  - ~/code/CloakBrowser/js/src/puppeteer.ts
  - ~/code/CloakBrowser/js/src/types.ts
  - ~/code/CloakBrowser/js/src/human-puppeteer/index.ts
  - ~/code/CloakBrowser/js/src/human-puppeteer/keyboard.ts
  - ~/code/CloakBrowser/js/src/human-puppeteer/scroll.ts
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/js/package.json
  - ~/code/CloakBrowser/js/examples/basic-playwright.ts
  - ~/code/CloakBrowser/js/examples/basic-puppeteer.ts
  - ~/code/CloakBrowser/js/README.md
  - ~/code/CloakBrowser/js/tests/puppeteer.test.ts
  - ~/code/CloakBrowser/js/tests/stealth.puppeteer.test.ts
  - ~/code/CloakBrowser/js/tests/humanizePuppeteer.test.ts
  - ~/code/CloakBrowser/js/tests/launch.test.ts
  - ~/code/CloakBrowser/js/tests/humanizeEngine.test.ts
summary: >-
    The JS package drives the same stealth binary through Playwright (default export) or Puppeteer (cloakbrowser/puppeteer); they differ in proxy auth, viewport, humanize depth and CDP noise.
provenance:
  extracted: 0.78
  inferred: 0.17
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Playwright vs Puppeteer backends

The JavaScript/TypeScript package exposes two launch backends over one binary. The default export (`cloakbrowser`, `index.ts`) re-exports the Playwright wrapper (`playwright.ts`); `cloakbrowser/puppeteer` exposes a Puppeteer wrapper (`puppeteer.ts`). Both share the same pipeline for binary download, GeoIP, WebRTC, profile seed, stealth args, license env and Widevine, then hand off to `playwright-core` or `puppeteer-core`. The Python and .NET wrappers are Playwright-only. ^[inferred] The README recommends Playwright: Puppeteer "sends more CDP protocol traffic", which reCAPTCHA Enterprise can detect, causing intermittent 403s.

## Shared pipeline

Both backends run the same steps before launching (see [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]):

1. `ensureBinary` (or `CLOAKBROWSER_BINARY_PATH`).
2. `maybeResolveGeoip` → timezone/locale plus exit IP.
3. `resolveWebrtcArgs` + `appendWebrtcExitIp` ([[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]]).
4. `persistentSeedArgs` for persistent profiles ([[projects/cloakbrowser/subsystems/profile-seed|Profile seed persistence]]).
5. `buildArgs` with dedup; `maybeWarnWindowsFonts`.
6. `buildLaunchEnv` + `mintDenialFile` for Pro license handling, and `installLicenseGuard` after launch.
7. `IGNORE_DEFAULT_ARGS` to drop the framework's automation-revealing default flags. ^[inferred] (the list is in `config.ts`).

## Where they differ

| Concern | Playwright (`cloakbrowser`) | Puppeteer (`cloakbrowser/puppeteer`) |
|---|---|---|
| Entry points | `launch`, `launchContext`, `launchPersistentContext`, plus composable `buildLaunchOptions`, `buildContextOptions`, `humanizeBrowser` | `launch`, `launchPersistentContext` (returns a `Browser`) |
| Proxy without inline-auth binary | Playwright `proxy` launch option | Credentials stripped; `page.authenticate()` patched into `newPage()` |
| Request-client proxy leak warning (#579) | Yes | Not applicable |
| Viewport default | `viewport: null` when headed, or headless on binaries passing `binarySupportsHeadlessNoViewport`; else `DEFAULT_VIEWPORT` | `defaultViewport` resolved the same way; user `launchOptions.defaultViewport` wins |
| Headless precedence | raw `launchOptions.headless` overrides top-level (spread last) | top-level `headless` wins (set after the spread) |
| Locale/timezone | Binary flags only; `contextOptions.locale`/`timezoneId` stripped with a warning | Binary flags only |
| Humanize engine | Full engine: Playwright's own InjectedScript in CDP isolated worlds, strict selectors, actionability checks | Separate, lighter port (`human-puppeteer/`) sharing mouse/keyboard primitives |
| CDP traffic | Lower | Higher; README warns about reCAPTCHA Enterprise |

### Why the viewport logic matters

A headed browser with an emulated fixed viewport reports `outerWidth < innerWidth`, a physically impossible window and a bot tell. Playwright's default 1280×720 and Puppeteer's default 800×600 both cause this, so both wrappers default to no emulation when headed. Headless has no window chrome (outer equals inner), so a fixed viewport stays coherent on older binaries. Newer binaries report coherent dimensions in headless without emulation. See [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen and viewport consistency]].

The two wrappers resolve `headless` from opposite precedence because Playwright's options spread puts `launchOptions` last, while the Puppeteer wrapper sets `headless` after the spread. Each viewport decision reads the same effective value as the real launch call. Otherwise a headed browser could get a fixed viewport by accident.

## Humanize on Puppeteer

`human-puppeteer/` mirrors the Playwright humanize architecture with Puppeteer adaptations: `page.createCDPSession()`, `page.viewport()`, `page.$()` instead of locators, `keyboard.sendCharacter()` for non-ASCII text, `page.select()`, `click({ clickCount: 2 })` instead of `dblclick`.

- It patches page, mouse, keyboard, frame and `ElementHandle` methods (click, hover, type, press, tap, select, focus, drop, dragAndDrop, nested `$`/`$$`/`waitForSelector`). The Playwright side humanizes `ElementHandle` too (`cloakbrowser/human/patch.py`, `js/src/human/patch.ts`, pinned by both test suites); the README note that Playwright handles from `query_selector()` "bypass all patches" is stale. Browser-level patching covers `newPage`, `createBrowserContext` (v22+) or `createIncognitoBrowserContext` (v21), and `targetcreated`.
- Stealth DOM queries use a CDP isolated world; the ElementHandle input check uses `DOM.describeNode` so no JavaScript runs in the main world. `page.evaluate` is a fallback only.
- Shift symbols are typed with CDP `Input.dispatchKeyEvent` (trusted events, no evaluate stack frame), driven by the shared `Typist` timeline.
- Scrolling wheels in whole notches sized by the persona's `navigator.platform`: 100 px on Windows, 40 on macOS, 120 on Linux. CDP reports one notch per wheel event, so only whole notches keep `deltaY` and `wheelDeltaY` consistent. `el.scrollIntoView()` (Puppeteer v22+) is replaced by the same wheel sequence, because a native snap-scroll is a strong bot signal.
- Frame actions resolve the selector inside the frame (stock Puppeteer semantics, #184) and run the humanized ElementHandle action. The isolated world is invalidated on any main-frame navigation (#507).

Engine details: [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]].

## Pinned by tests (B6, JS suite)

`js/tests/puppeteer.test.ts` (mocked `puppeteer-core`), `stealth.puppeteer.test.ts` (mock pages) and `humanizePuppeteer.test.ts` (real binary, skipped without `CLOAKBROWSER_BINARY_PATH`) fix the Puppeteer side at commit b1a00ed:

- **Launch.** The binary comes from `ensureBinary` with `releaseChannel` forwarded. Stealth args, `--fingerprint-timezone`/`--lang`, user args and `launchOptions` fields such as `slowMo` all reach `puppeteer.launch`. `launchPersistentContext` passes `userDataDir` through.
- **Proxy.** A credential-free HTTP string becomes `--proxy-server`, and a dict `bypass` becomes `--proxy-bypass-list`. On free macOS a credentialed HTTP proxy is stripped to `--proxy-server=http://proxy:8080` and every `newPage()` calls `page.authenticate`. On a supported binary, or for SOCKS5, credentials stay inline and `authenticate` is never called.
- **Viewport.** Headless gets `DEFAULT_VIEWPORT`, headed gets `null` (otherwise Puppeteer forces 800×600), and an explicit `launchOptions.defaultViewport`, including `null`, always wins. With `headless: true` plus `launchOptions.headless: false`, the top-level value wins and the viewport follows it.
- **Humanize wiring (mock tests).** The isolated world uses `page.createCDPSession()`, not `context.newCDPSession`. `StealthEval.evaluate` retries once after `exceptionDetails` and returns `undefined` after a second failure. The Playwright engine raises `StealthWorldError` in that case instead. A main-frame `goto` invalidates the world. Page, `$`, `$$`, `waitForSelector`, nested `el.$()` and frame handles are patched once, guarded by `_humanPatched`. Dynamically attached frames are patched once, and browser patching covers `newPage`, `createIncognitoBrowserContext` and `targetcreated`. `focus()` clicks instead of focusing programmatically. `keyboard.sendCharacter` serves as `insertText`.
- **Two deliberate fallbacks to stock behaviour** that the Playwright engine does not have:
  - an `ElementHandle` with a `null` bounding box runs the original `click`;
  - `select()` hovers and then delegates to Puppeteer's original `page.select` (per the test name), which sets the value without key presses. ^[inferred]
- **Real-browser behaviour.** Right click via `page.click` and `mouse.click` gives button 2 plus `contextmenu`. `click({ count: 2 })` is a real double click (`mousedown:1 click:1 mousedown:2 click:2 dblclick:2`). Frame `click`/`type`/`hover` resolve inside the frame (#184). The test notes a known gap: Puppeteer's scroll may wheel the iframe instead of the page. There is no repeat `keydown` for `xX`/`1!`, and uppercase typos use Shift. These tests answer part of the parity question below.
- **Select-all modifier.** The Puppeteer mock test expects `Meta` on a macOS host and `Control` elsewhere, keyed on the *host* `process.platform`, while the Playwright engine test requires the *persona's* modifier (`--fingerprint-platform=macos` on a Linux host). Whether the Puppeteer port follows the persona is not settled by these tests. ^[ambiguous]

## Packaging and choosing a backend

These details come from `js/package.json` and the JS examples (as of 26/10/2026, commit b1a00ed):

- **Subpath exports**: `cloakbrowser` 0.6.4 exposes three: `.` for Playwright, `./puppeteer`, and `./human` for the humanize API.
- **Optional peers**: both automation libraries are optional peer dependencies, `playwright-core` >=1.53 and `puppeteer-core` >=21. A user installs only the backend they import (`npm install cloakbrowser playwright-core` or `... puppeteer-core`). `tar` is the only hard dependency.
- **Same calling code**: `examples/basic-playwright.ts` and `examples/basic-puppeteer.ts` differ only in the import path (`src/index.js` vs `src/puppeteer.js`). The surface a user sees is the same. The differences in the table above are internal.
- **Dev pins**: development tracks recent majors, `playwright-core` 1.64 and `puppeteer-core` ^25. The peer floors stay far lower, a wide compatibility promise that the `_check_playwright`-style internals checks have to defend. ^[inferred]

## Open Questions

- The Puppeteer humanize port does not use Playwright's InjectedScript, so selector semantics and actionability checks probably differ from the Playwright engine. ^[inferred] `humanizePuppeteer.test.ts` (B6) covers clicks, double clicks, frames and typing on the shared fixture page, but not actionability, strict mode or stalled pages.

## Sources

- ~/code/CloakBrowser/js/src/index.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/playwright.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/puppeteer.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/types.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/human-puppeteer/index.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/human-puppeteer/keyboard.ts (commit b1a00ed)
- ~/code/CloakBrowser/js/src/human-puppeteer/scroll.ts (commit b1a00ed)
- ~/code/CloakBrowser/README.md (commit b1a00ed)
- ~/code/CloakBrowser/js/package.json, js/examples/basic-playwright.ts, js/examples/basic-puppeteer.ts, js/README.md (commit b1a00ed, B4 batch)
- ~/code/CloakBrowser/js/tests/puppeteer.test.ts, js/tests/stealth.puppeteer.test.ts, js/tests/humanizePuppeteer.test.ts, js/tests/launch.test.ts, js/tests/humanizeEngine.test.ts (commit b1a00ed, B6 batch)
