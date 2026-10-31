---
title: >-
    API server
category: subsystems
tags: [fastapi, rest-api, websocket, profile-manager, cdp]
aliases: [backend/main.py, FastAPI app, AuthMiddleware]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/browser-manager]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/profile-database]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/api-models]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/diagnostics]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/architecture/manager-architecture]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/diagnostics.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/README.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_auth.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_rfb.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/conftest.py
summary: >-
    backend/main.py, the Manager's hub (degree 57): FastAPI routes for profiles, files, launch/stop, CDP and VNC proxies, settings, status and auth, plus the SPA fallback.
provenance:
  extracted: 0.86
  inferred: 0.12
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T06:20:23+01:00
updated: 2026-10-31T06:53:05+01:00
---

# API server

`backend/main.py` is the Manager's single FastAPI application and its most connected non-test file (degree 57 in ast-extract, 31/10/2026). It wires a raw ASGI auth middleware, a lifespan that boots the database and browser manager, about 40 routes, two WebSocket proxies (VNC and CDP) and a catch-all that serves the React build. Business logic is delegated to the [[projects/cloakbrowser-manager/subsystems/browser-manager|browser manager]], [[projects/cloakbrowser-manager/subsystems/profile-database|profile database]] and [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts|artifact store]]; schemas live in [[projects/cloakbrowser-manager/subsystems/api-models|API models]].

## Route groups (as of commit 11e0043)

| Group | Endpoints | Notes |
|---|---|---|
| Auth | `GET /api/auth/status`, `POST /api/auth/login`, `POST /api/auth/logout` | See [[projects/cloakbrowser-manager/architecture/authentication-model]] |
| Profile CRUD | `GET/POST /api/profiles`, `GET/PUT/DELETE /api/profiles/{id}`, `POST /api/profiles/reorder` | `PUT` passes only fields explicitly set (`exclude_unset`) |
| Lifecycle | `POST .../launch`, `POST .../stop`, `GET .../status`, `POST .../reset`, `POST .../duplicate` | Reset wipes browser state and re-rolls the seed; duplicate described in [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]] |
| Proxy test | `POST /api/profiles/test-proxy` | Exit IP, [[projects/cloakbrowser/subsystems/geoip-resolver\|GeoIP]] country/city/timezone, latency ([[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]) |
| Files | `GET/POST .../files`, `GET/DELETE .../files/{artifact_id}` | Raw-body streaming upload, see [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]] |
| Automation | `GET .../cdp`, `.../cdp/json`, `.../cdp/json/version`, `.../cdp/json/list`, WS `.../cdp`, WS `.../cdp/devtools/{path}` | See [[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]] |
| Viewer | WS `.../vnc`, `GET/POST .../clipboard`, `GET .../screenshot` | VNC relay rewrites RFB messages; see [[projects/cloakbrowser-manager/subsystems/vnc-viewer]] |
| System | `GET /api/health`, `GET /api/status`, `GET /api/update-check`, `GET/PUT /api/settings`, `POST /api/shutdown`, `POST /api/open-external` | |

## Error mapping on launch

The launch route turns failures into distinct HTTP codes so the UI can explain them:

- `CloakBrowserLicenseError` → **402** for a concurrency-seat denial (with `upgrade_url`), **403** for other license problems (bad, expired or revoked key, license server unreachable). The structured detail is `{message, reason, upgrade_url?}`.
- `ProfileBusyError` → **409** (state being copied/reset, or previous browser still closing); already running → 409.
- `ValueError` → **400** (bad proxy format, [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|transparent proxy]] without credentials, user-supplied `--remote-debugging-*` flags).
- Anything else → 500 "Failed to launch browser", with the traceback logged.

Seat denials that arrive after the HTTP 200 (the binary boots, then self-exits about 1 s later) are surfaced on the next status poll as `last_error`; see [[projects/cloakbrowser-manager/subsystems/browser-manager]].

## Notable mechanisms

- **Raw ASGI middleware** instead of `BaseHTTPMiddleware`, because the latter breaks WebSocket upgrades.
- **Same-origin guards**: WebSocket upgrades with a browser `Origin` must match `Host` (CSWSH protection, close code 4403); `POST /api/shutdown` applies the same check because a bodyless POST is a CORS simple request any website could fire at localhost. Clients without `Origin` (Playwright, curl) pass.
- **Update check**: polls GitHub's latest-release API for the upstream `CloakHQ/CloakBrowser-Manager` repository, cached 6 h, fail-soft; only results that actually reached GitHub are cached. ^[inferred] This fork therefore compares against upstream releases, not its own.
- **`/api/open-external`** opens http(s) URLs in the host browser, because `window.open` is a no-op inside the pywebview window.
- **Settings hot-apply**: `PUT /api/settings` persists the key/channel, validates channel ∈ {stable, preview}, re-resolves tier and binary off the event loop, and returns the fresh system status. Keys are masked as first 5 + last 4 characters.
- **Status**: `/api/status` reports running count, binary version, license tier, host/runtime/viewer mode and, only under Linux Docker (where a Windows persona is emulated), Windows font coverage via the wrapper's font-tell check ([[projects/cloakbrowser/detection-vectors/font-fingerprinting]]).
- **Downloads served as attachments**: artifact bytes always go out as `application/octet-stream` with `nosniff` and `no-store`, so a document from a merchant portal can never render inline on the Manager's origin.
- **SPA fallback** is mounted last so `/api/*` is never swallowed by `index.html`; an explicit `GET .../cdp` route exists for the same reason.

## Diagnostics

At import the server installs `backend/diagnostics.py` hooks: uncaught exceptions from any thread or the asyncio loop go to `manager.log` with tracebacks, and direct writes to stderr (wrapper banners) are mirrored into the log. Each launch logs the seed, a credential-redacted proxy, tier, plan and runtime. A one-line startup fingerprint records versions, OS, frozen state, stream encodings, tier and data dir (CHANGELOG 0.1.2). See [[projects/cloakbrowser-manager/subsystems/diagnostics]].

## Pinned by tests

`backend/tests/test_api.py`, `test_auth.py` and `test_rfb.py` act as the server's specification (as of commit `11e0043`). The suite mocks the whole `cloakbrowser` package in `conftest.py` before any import. It runs every API test against the Linux Docker contract (`RuntimeConfig("linux","docker","vnc")`) on every host OS, and it points `$HOME` at a temp directory so the GTK bookmarks file is never written to a real home.

- **CRUD codes.** Create returns **201** with a 36-character UUID and `status: "stopped"`. Unknown ids return **404** on get, update, delete, status, launch, stop, reset, duplicate, clipboard and CDP `json/*`. Delete and reset of a running profile call `browser_mgr.stop()` first. `POST /reorder` returns `{"ok": true}` and the list then follows the given order.
- **Strict schemas.** An unknown or removed field (`headless`, `user_agent`) returns **422**, and so does `gpu_family: null` on update. The duplicate body rejects unknown keys too (`copy_state` → 422).
- **Launch mapping confirmed.** Already running → 409. `ValueError` → 400 with the message (e.g. "Invalid proxy scheme 'ftp'"). Any other exception → 500 with the fixed detail "Failed to launch browser"; the inner message (e.g. "Xvnc failed") is not leaked.
- **Viewer fields.** Every profile response carries `runtime_mode` and `viewer_mode`. A native launch returns `vnc_ws_port: null` and `display: null`. `cdp_url` is `null` when stopped and the relative `/api/profiles/<id>/cdp` when running.
- **CDP discovery rewrite.** `json/version` and `json/list` rewrite Chrome's loopback `webSocketDebuggerUrl` to `ws://<request host>/api/profiles/<id>/cdp[/devtools/page/<id>]`, or `wss://` when `X-Forwarded-Proto: https` is set. Entries without a WS URL are left alone. An unreachable Chrome gives **502**.
- **Download-behaviour sniffing on the CDP proxy.** `_sets_download_behaviour` counts only `Browser.setDownloadBehavior` or `Page.setDownloadBehavior` aimed at the profile's own context (including through a flattened `sessionId`). It ignores calls with a client-created `browserContextId`, non-JSON frames and string mentions. Only a client that actually set the behaviour re-arms capture on detach, and only after the upstream socket has closed ([[projects/cloakbrowser-manager/techniques/download-capture]]).
- **Origin guards.** VNC and CDP WebSockets from `http://evil.com` are refused, while same-origin connections and connections with no Origin pass. `POST /api/shutdown` returns **403** for a cross-origin request and for the same host on a different port. `_same_origin_request` treats `:443` (https) and `:80` (http) as equal to a bare host.
- **Settings.** `GET /api/settings` masks a key as `cb_ba…63aa`. `PUT` persists to `settings.json` and hot-applies, calling `resolve_binary_status()` once. An empty `license_key` clears it from the file. A bad channel (`nightly`) returns **400**, and nothing is persisted or applied. A corrupt `settings.json` loads as `{}` ([[projects/cloakbrowser-manager/subsystems/settings-and-license]]).
- **Duplicate and reset concurrency.** See [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]. Duplicating with browser state is refused (409) while the source is launching, running or still closing. During the copy the source is held: launch, reset, delete and state-duplicate all return 409, and only a config-only duplicate passes. The clone stays unlisted until the copy finishes. A failed copy returns 500 with the OS error and leaves no row or directory behind. A cancelled request waits for the copy thread, then cleans up.
- **RFB translator.** `_filter_rfb_client_messages`, `_rewrite_*` and the clipboard converters are tested byte for byte in `test_rfb.py`; the rules are on [[projects/cloakbrowser-manager/subsystems/vnc-viewer]].
- **On the open question below.** `BrowserManager.get_status()` returns a `last_error` key (`test_vnc_manager.py`), but no test exercises the route's model. The suspicion that `ProfileStatusResponse` drops `last_error` stands: the model has no such field.

## Open Questions

- `GET /api/profiles/{id}/status` validates through `ProfileStatusResponse`, which has no `last_error` field, so the post-handshake denial is likely dropped on that route and only visible via the profile list/detail responses. ^[ambiguous]

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/diagnostics.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/README.md, CHANGELOG.md (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py, test_auth.py, test_rfb.py, conftest.py (commit 11e0043)
