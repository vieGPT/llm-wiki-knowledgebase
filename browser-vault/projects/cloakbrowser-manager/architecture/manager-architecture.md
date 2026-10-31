---
title: >-
    Manager architecture
category: architecture
tags: [profile-manager, fastapi, sqlite, orchestration, browser-automation]
aliases: [CloakBrowser Manager architecture, backend.main lifespan]
relationships:
  - target: "[[projects/cloakbrowser/cloakbrowser]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/diagnostics]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/download-capture]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/synthesis/manager-cloakbrowser-responsibility-split]]"
    type: related_to
  - target: "[[projects/cloakbrowser/architecture/wrapper-binary-split]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/database.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/runtime.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/settings_store.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/env_file.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/requirements.txt
  - repo:github.com/vieGPT/CloakBrowser-Manager/app_entry.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/.github/workflows/ci.yml
  - repo:github.com/vieGPT/CloakBrowser-Manager/README.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
summary: >-
    One FastAPI process serves the React SPA and a REST/WebSocket API, stores profiles in SQLite and drives one CloakBrowser persistent context per running profile.
provenance:
  extracted: 0.82
  inferred: 0.16
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T06:20:23+01:00
updated: 2026-10-31T06:37:29+01:00
---

# Manager architecture

CloakBrowser-Manager is a single Python process: a FastAPI app (`backend/main.py`) that serves the built React UI as static files and exposes a REST + WebSocket API under `/api/`. Profile records live in SQLite; each running profile is one CloakBrowser persistent browser context launched through the `cloakbrowser` Python wrapper and tracked in memory by the [[projects/cloakbrowser-manager/subsystems/browser-manager|BrowserManager]]. The Manager adds configuration, isolation, lifecycle and access paths; all fingerprint evasion stays inside the [[projects/cloakbrowser/cloakbrowser|CloakBrowser]] engine. ^[inferred]

## Components

| Layer | Module | Page |
|---|---|---|
| HTTP/WS API, auth middleware, static SPA | `backend/main.py` | [[projects/cloakbrowser-manager/subsystems/api-server]] |
| Request/response schemas | `backend/models.py` | [[projects/cloakbrowser-manager/subsystems/api-models]] |
| Browser lifecycle, launch args, locks | `backend/browser_manager.py` | [[projects/cloakbrowser-manager/subsystems/browser-manager]] |
| Profile, tag and artifact rows | `backend/database.py` | [[projects/cloakbrowser-manager/subsystems/profile-database]] |
| Per-profile files, download capture | `backend/artifacts.py`, `backend/downloads.py` | [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]], [[projects/cloakbrowser-manager/techniques/download-capture]] |
| Host/runtime policy, data dir | `backend/runtime.py` | [[projects/cloakbrowser-manager/architecture/platform-modes]] |
| License key + channel persistence | `backend/settings_store.py`, `backend/env_file.py` | [[projects/cloakbrowser-manager/subsystems/settings-and-license]] |
| Live viewer (Linux) | `backend/vnc_manager.py` | [[projects/cloakbrowser-manager/subsystems/vnc-viewer]] |
| UI | `frontend/` | [[projects/cloakbrowser-manager/subsystems/frontend-ui]] |

## Dependencies (confirmed)

`backend/requirements.txt` (commit `11e0043`) pins: `fastapi>=0.115.0`, `uvicorn[standard]>=0.34.0`, `pydantic>=2.0`, `cloakbrowser[geoip]>=0.6.3`, `websockets>=14.0`, `httpx>=0.27.0`, and `pywebview>=5.0` (native window shell; `app_entry.py` falls back to the default browser without it). FastAPI is therefore a direct dependency (`backend/main.py` imports `FastAPI`, `HTTPException`, `WebSocket` and the response classes from it); why ast-extract did not surface it among the import edges is unclear. ^[ambiguous]

## Startup sequence (lifespan)

1. `load_env_file()` runs at import, before anything reads the environment, so a manager-root `.env` feeds `AUTH_TOKEN`, the license key and the data-dir resolution. Values already in the environment win.
2. Logging goes to the console and a rotating `logs/manager.log` (2 MB × 3) in the data dir; [[projects/cloakbrowser-manager/subsystems/diagnostics|diagnostics]] hooks (crash hooks, stderr tee) are installed at import.
3. Lifespan: validate VNC availability, `init_db()` (create or migrate schema), reclaim unpublished artifact bytes and rebuild the file-chooser views, kill stale display processes, then resolve the license tier and pre-download the binary off the event loop, so a multi-hundred-MB Pro download never sits on a launch or the 60 s auto-launch timeout.
4. Auto-launch every profile flagged `auto_launch`, sequentially, each with a 60 s timeout, as a background task.
5. Shutdown cancels auto-launch and stops every running browser (`cleanup_all`).

## Request flow: launching a profile

UI or API client → `POST /api/profiles/{id}/launch` → profile row read from SQLite → `BrowserManager.launch()` builds `--fingerprint*` arguments and launch options from the row → `cloakbrowser.launch_persistent_context_async` starts the binary with a Manager-reserved loopback CDP port → the Manager verifies CDP is up, registers the running profile, and returns `cdp_url=/api/profiles/{id}/cdp` plus viewer details. The UI then shows a native window (Windows/macOS) or connects to `/api/profiles/{id}/vnc` (Linux Docker). External automation connects through the [[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying|CDP proxy]].

## Configuration precedence

License key and release channel: environment (including `.env`) beats `settings.json` at startup. The in-app Settings `PUT /api/settings` then overwrites the in-memory value directly and re-resolves the binary, so a runtime change takes effect even when an environment variable was set; the env value would win again after a restart. ^[inferred] The README states "an environment variable overrides the in-app setting", which matches startup but not the hot-apply path. ^[ambiguous]

## Design decisions

- **One license, shared seats**: one key per Manager instance; every launch passes it, so the concurrency-seat pool is per license and profile count is unlimited ([[projects/cloakbrowser/subsystems/license-and-tiers]]).
- **Runtime chosen by host, not user**: Linux is Docker + KasmVNC only; Windows/macOS are native only ([[projects/cloakbrowser-manager/architecture/platform-modes]]).
- **Filesystem operations serialize against launches** through `hold_stopped`, so duplicate, reset and delete never touch a directory Chromium owns.
- **CI** runs the backend test suite on Ubuntu, Windows and macOS (Python 3.12) and the frontend tests + build on Node 20, matching the three supported hosts. ^[inferred]

## Open Questions

- The README stack names "noVNC/KasmVNC"; the code talks to KasmVNC's websockify endpoint. Whether noVNC is still bundled in the frontend is for the VNC/frontend pages.

## Related

- [[projects/cloakbrowser-manager/synthesis/manager-cloakbrowser-responsibility-split|Manager vs CloakBrowser responsibility split]] — which CloakBrowser subsystem each part of this architecture relies on.
- [[projects/cloakbrowser/architecture/wrapper-binary-split|Wrapper/binary split]] — the engine-side layering the Manager sits on top of.

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py, backend/browser_manager.py, backend/database.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/runtime.py, backend/settings_store.py, backend/env_file.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/requirements.txt (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/app_entry.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/.github/workflows/ci.yml (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/README.md, CHANGELOG.md (commit 11e0043, v0.1.8 of 14/10/2026)
