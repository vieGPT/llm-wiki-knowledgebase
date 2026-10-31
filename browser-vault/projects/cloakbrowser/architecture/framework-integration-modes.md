---
title: >-
    Framework integration modes
category: architecture
tags: [browser-automation, cdp, ai-agents, integrations, playwright]
aliases: [integration modes, connect over CDP, direct binary launch]
relationships:
  - target: "[[projects/cloakbrowser/architecture/wrapper-binary-split]]"
    type: derived_from
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/agent_browser.sh
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/browser_use_example.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/crawl4ai_example.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/INSTRUCTIONS.md
  - repo:github.com/CloakHQ/CloakBrowser/examples/basic.py
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/examples/stagehand.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/config.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/args.ts
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/crawlee_example.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/langchain_loader.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/scrapling_example.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/selenium_example.py
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/undetected_chromedriver.py
summary: >-
    How third-party tools adopt CloakBrowser: launch the binary with the wrapper's stealth args, connect over CDP to a wrapper-launched browser, or wrap the wrapper itself; humanize stays wrapper-only.
provenance:
  extracted: 0.76
  inferred: 0.19
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:46:02+01:00
updated: 2026-10-31T05:52:24+01:00
---

# Framework integration modes

Because fingerprint spoofing lives in the patched binary and the wrapper only picks the binary and its flags (see [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]]), other automation frameworks can adopt CloakBrowser without using its Playwright API. The README and the `examples/integrations/` scripts show two modes, plus the plain wrapper launch (`examples/basic.py`: `launch()`, `new_page()`, `goto()`). The choice decides which CloakBrowser features the framework gets: binary-level stealth comes along in both modes, while wrapper-level features such as `humanize=True` and GeoIP resolution only apply when the wrapper does the launching or patches the connected client. ^[inferred]

## Mode 1: framework launches the binary

The framework starts Chromium itself and is pointed at the CloakBrowser executable and the wrapper's stealth flags. The wrapper is used only as a helper library: `ensure_binary()` returns the cached binary path (downloading if needed) and `get_default_stealth_args()` returns the flag list.

- README examples for this mode: Selenium, Stagehand, undetected-chromedriver.
- `agent_browser.sh` (agent-browser, a Node.js CLI) uses it because, per its comment, agent-browser launches Chrome itself through environment variables and cannot attach to an existing browser over CDP. The script exports the binary path and the comma-joined stealth args into agent-browser's executable-path and args variables, then checks `navigator.webdriver`, plugin count and platform.
- What is lost: anything the wrapper does at launch time beyond the default arg list, such as GeoIP timezone/locale resolution, proxy credential handling, persistent-profile seeding and humanize patching. ^[inferred] See [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]].

## JS evidence for Mode 1: Stagehand (`js/examples/stagehand.ts`)

These notes come from the B4 batch (commit b1a00ed):

- The Stagehand example is the TypeScript form of Mode 1. It calls `ensureBinary()` for the executable path and `getDefaultStealthArgs()` for the flags, then passes both to Stagehand's `localBrowserLaunchOptions` (`env: "LOCAL"`, headless). Stagehand handles AI-driven navigation, and its model needs `OPENAI_API_KEY`. CloakBrowser handles detection.
- `getDefaultStealthArgs()` returns only the base set: `--no-sandbox`, a random `--fingerprint` seed and `--fingerprint-platform`. Everything that `buildArgs` adds on top is missing in Mode 1:
  - `--ignore-gpu-blocklist` for headed mode and Windows
  - the `MediaRouter` re-enable
  - the version-gated `--start-maximized`
  - timezone and locale flags
  - the `IGNORE_DEFAULT_ARGS` removal of `--enable-automation` and `--enable-unsafe-swiftshader`

  Whether the host framework's own launcher re-adds those automation defaults depends on that framework. ^[ambiguous]
- Each `getDefaultStealthArgs()` call draws a fresh random seed. A Mode 1 integration therefore presents a new device on every launch unless it replaces the `--fingerprint` value with a fixed seed. ^[inferred] See [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].

## Mode 2: wrapper launches, framework connects over CDP

The wrapper launches the browser with all stealth flags and a remote-debugging port, and the framework connects to that endpoint.

- `browser_use_example.py`: `launch_async(headless=True)` with `--remote-debugging-port=9242` bound to 127.0.0.1, then browser-use's `BrowserSession(cdp_url=…)` drives an LLM agent (OpenAI model in the example). The docstring pitches this as browsing "behind Cloudflare, reCAPTCHA, DataDome".
- `crawl4ai_example.py`: same pattern on port 9243, with Crawl4AI's `BrowserConfig(browser_mode="cdp")` doing extraction to markdown.
- README examples for this mode: browser-use, Crawl4AI, Scrapling.
- The examples bind the debugging port to loopback, which keeps the unauthenticated CDP endpoint off the network. ^[inferred]
- The wrapper-owned browser must be closed by the wrapper (`await cb_browser.close()`) after the framework finishes.

**Humanize over CDP**: fingerprint patches work automatically over CDP, but `humanize=True` is wrapper-level. A separate script connected over CDP has to import the patch functions (`patchBrowser` + `resolveConfig` in JS, the `patch_*` functions in Python) to get humanized input. A framework that drives the page through its own CDP calls bypasses the [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]] entirely. ^[inferred]

CDP traffic from the framework is itself a detection surface: frameworks that send more protocol traffic are easier to flag. See [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]] and [[projects/cloakbrowser/detection-vectors/cdp-automation-detection|CDP and automation detection]]. ^[inferred]

## Deployment surfaces

The same choice recurs at deployment level. The CloakServe container exposes a CDP endpoint with per-seed browser processes ([[projects/cloakbrowser/subsystems/cloakserve-cdp-server|CloakServe CDP server]]), while the AWS Lambda image runs the wrapper in-process per invocation ([[projects/cloakbrowser/architecture/aws-lambda-deployment|AWS Lambda deployment]]).

## Per-integration evidence (batch B2, commit b1a00ed)

| Framework | Mode | How the example wires it | Notable details |
|---|---|---|---|
| Crawlee (Python) | 1, inside Playwright | subclasses `PlaywrightBrowserPlugin`; `new_browser()` calls Playwright's `chromium.launch()` with `ensure_binary()`, the stealth args appended to any user args, and `ignore_default_args=IGNORE_DEFAULT_ARGS` | drops user-supplied `executable_path` and `chromium_sandbox`; sets `header_generator=None` because "CloakBrowser handles fingerprints at the binary level"; one page per browser |
| Scrapling | 2 | `launch_async(headless=True)` with `--remote-debugging-port=9245` and `--remote-debugging-address=127.0.0.1`, then reads `/json/version` for `webSocketDebuggerUrl` and passes it to `StealthyFetcher.async_fetch(cdp_url=…)` | Scrapling needs a `ws://` URL, not the HTTP endpoint |
| LangChain | neither: wrapper used directly | a `load_urls_stealth()` helper calls `launch_async()` and turns each page's `innerText` and title into LangChain `Document` objects | LangChain's `PlaywrightURLLoader` "hardcodes chromium.launch() with no way to pass a custom binary", so the example replaces the loader instead of configuring it |
| Selenium | 1, via ChromeDriver | `Options.binary_location` = binary path, `--headless`, each stealth arg added | docstring: "ChromeDriver version must match Chromium 145" |
| undetected-chromedriver | 1, via ChromeDriver | same as Selenium with `uc.ChromeOptions`, plus `version_main` taken from the major of `get_chromium_version()` | pitched as a "double stealth layer": uc patches ChromeDriver signals, CloakBrowser patches fingerprints |

Design observations:

- **Crawlee is the most complete Mode 1 example.** It is the only one that also passes `IGNORE_DEFAULT_ARGS`, so Playwright's `--enable-automation` and `--enable-unsafe-swiftshader` defaults are removed. It also switches off Crawlee's own header/fingerprint generator, which would otherwise layer JS-level spoofing over the binary's coherent identity. ^[inferred]
- **WebDriver integrations skip `IGNORE_DEFAULT_ARGS`.** ChromeDriver adds its own default switches, and the examples do not exclude them. Whether ChromeDriver's defaults (for example `--enable-automation`) undo part of the stealth is not addressed; the examples only spot-check `navigator.webdriver`, plugin count, platform and (uc) `hardwareConcurrency`. ^[ambiguous]
- **Version coupling.** Only the uc example derives the driver version from the binary. The Selenium docstring pins Chromium 145, while the README's platform table (as of 26/10/2026) lists free builds at 145/146 and Pro at 155. A manually installed ChromeDriver therefore has to track whichever binary `ensure_binary()` resolved. ^[inferred]
- **Fresh seed per launch** applies to every Mode 1 example here: each calls `get_default_stealth_args()` once per launch and does not pin `--fingerprint`. ^[inferred]
- **LangChain helper and CDP volume**: one `page.evaluate()` per URL is harmless on plain pages, but see [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]] for reCAPTCHA-protected targets. ^[inferred]
- **Mode assignments confirmed from code** for every README integration: Stagehand, Selenium, undetected-chromedriver and Crawlee are Mode 1; browser-use, Crawl4AI and Scrapling are Mode 2; LangChain bypasses both by calling the wrapper directly. The README's two-option snippet omits Crawlee and LangChain.

## Open Questions

- Whether Mode 1 frameworks that inject their own automation flags (for example Selenium's WebDriver) undo binary-level stealth is not discussed. ^[ambiguous]
- Whether Crawlee's other per-browser features (session pool, fingerprint injection when `header_generator` is left on) conflict with the binary's seed is untested in the repository.

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/agent_browser.sh
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/browser_use_example.py
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/crawl4ai_example.py
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/INSTRUCTIONS.md
- repo:github.com/CloakHQ/CloakBrowser/examples/basic.py
- repo:github.com/CloakHQ/CloakBrowser/README.md
- repo:github.com/CloakHQ/CloakBrowser/js/examples/stagehand.ts, js/src/config.ts, js/src/args.ts (commit b1a00ed, B4 batch)
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/crawlee_example.py, langchain_loader.py, scrapling_example.py, selenium_example.py, undetected_chromedriver.py (commit b1a00ed, B2 batch)
