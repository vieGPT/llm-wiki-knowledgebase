---
title: >-
    CDP endpoint proxying
category: techniques
tags: [cdp, playwright, puppeteer, browser-automation, websocket]
aliases: ["/api/profiles/{id}/cdp", per-profile CDP endpoint, Automation API]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/cloakserve-cdp-server]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/cdp-traffic-minimization]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/api-server]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/README.md
  - ~/code/CloakBrowser-Manager/backend/tests/test_api.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_browser_manager.py
summary: >-
    Each running profile's Chrome debugger stays on a private loopback port; the Manager exposes it at /api/profiles/{id}/cdp, rewriting discovery URLs and relaying WebSocket frames.
provenance:
  extracted: 0.82
  inferred: 0.16
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:27:05+01:00
updated: 2026-10-31T10:47:25+01:00
---

# CDP endpoint proxying

Every running profile is scriptable: Playwright or Puppeteer connect to `http://<manager>/api/profiles/<id>/cdp` and drive the same session a human watches in the native window or VNC viewer (README "Automation API"). The browser's real DevTools port never leaves loopback; the [[projects/cloakbrowser-manager/subsystems/api-server|API server]] is the only door, so the Manager's auth and origin checks apply to automation too. It plays the role [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve]] plays in the CloakBrowser Docker image, but keyed by profile id instead of seed query parameters. ^[inferred]

## How it works

- **Port ownership.** The [[projects/cloakbrowser-manager/subsystems/browser-manager|browser manager]] reserves an OS-chosen free port on 127.0.0.1 per launch, adds `--remote-debugging-address=127.0.0.1` and `--remote-debugging-port`, and refuses user launch args that set either. It verifies `/json/version` reports a debugger URL on that port before declaring the launch good.
- **Discovery rewrite.** `GET .../cdp/json/version` and `.../cdp/json/list` (with and without trailing slash) fetch Chrome's JSON and rewrite every `webSocketDebuggerUrl` to the Manager's host: the browser target to `ws(s)://<host>/api/profiles/<id>/cdp`, page targets to `.../cdp/devtools/<path>`. `wss` is chosen when `X-Forwarded-Proto` says https, so it works behind a TLS reverse proxy.
- **Frame relay.** WebSocket `.../cdp` looks up Chrome's browser-level debugger URL and relays frames both ways without translation (unlike the VNC path, which filters RFB messages); `.../cdp/devtools/{path}` does the same for page targets. Message size is unlimited and keepalive pings are off.
- **Guards.** Browser-originated upgrades must have `Origin` equal to `Host` (cross-site WebSocket hijacking protection); Playwright and Puppeteer send no `Origin` and pass. With `AUTH_TOKEN` set, clients need the bearer header or login cookie ([[projects/cloakbrowser-manager/architecture/authentication-model]]).
- **Download hand-off.** The relay watches client frames for `setDownloadBehavior` so the Manager can re-arm its own capture after the client leaves ([[projects/cloakbrowser-manager/techniques/download-capture]]).

## Trade-offs

- **Human and automation on one identity.** Driving the profile a person also uses keeps cookies, history and fingerprint consistent between manual and scripted sessions, rather than spawning a separate automation browser. ^[inferred]
- **Extra hop, no filtering.** The proxy adds latency but does not reduce CDP chatter; the detection risk of heavy CDP use ([[projects/cloakbrowser/detection-vectors/cdp-automation-detection]]) stays the client's responsibility ([[projects/cloakbrowser/techniques/cdp-traffic-minimization]]). ^[inferred]
- **Last-writer-wins browser state.** Settings like download behaviour are browser-wide, so an automation client and the Manager can overwrite each other; the Manager handles downloads explicitly but not other browser-wide settings. ^[inferred]
- `GET .../cdp` (plain HTTP) returns connection info and usage text, partly so the SPA catch-all does not answer that path with `index.html`.

## Pinned by tests

`backend/tests/test_api.py` and `test_browser_manager.py` (as of commit `11e0043`):

- **`json/version`.** It rewrites `ws://127.0.0.1:5100/devtools/browser/<id>` to `ws://<request host>/api/profiles/<pid>/cdp` and passes other fields (`Browser`) through. `X-Forwarded-Proto: https` gives `wss://`.
- **`json/list`.** It rewrites page URLs to `.../cdp/devtools/page/<id>` and leaves entries without a `webSocketDebuggerUrl` untouched.
- **Error codes.** A stopped or unknown profile gives 404 on both discovery routes. An unreachable Chrome gives 502.
- **`cdp_url`.** It is the *relative* `/api/profiles/<id>/cdp` while running and `null` when stopped. Clients build the absolute URL themselves.
- **Origin check.** A cross-origin browser upgrade on `.../cdp` is refused; a client with no `Origin` passes.
- **Port ownership.** User `--remote-debugging-address=0.0.0.0` raises "Manager owns remote debugging". A debugger answering on a port other than the reserved one counts as not ready, and a failed first CDP check closes that context and relaunches once.

## Sources

- ~/code/CloakBrowser-Manager/backend/main.py, backend/browser_manager.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/README.md (Automation API) (commit 11e0043)
- ~/code/CloakBrowser-Manager/backend/tests/test_api.py, test_browser_manager.py (commit 11e0043)
