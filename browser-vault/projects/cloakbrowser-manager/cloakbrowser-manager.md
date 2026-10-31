---
title: >-
    CloakBrowser-Manager
category: project
tags: [anti-detect, profile-manager, fastapi, react, orchestration]
source_repo: github.com/vieGPT/CloakBrowser-Manager
source_cwd_hint: ~/code/CloakBrowser-Manager
sources: [repo:github.com/vieGPT/CloakBrowser-Manager]
summary: >-
    Self-hosted GUI profile manager over the CloakBrowser engine: FastAPI backend, React UI, SQLite; each profile is a persistent separate identity with its own seed, proxy and locale.
relationships:
  - target: "[[projects/cloakbrowser/cloakbrowser]]"
    type: uses
provenance:
  extracted: 0.85
  inferred: 0.15
  ambiguous: 0.0
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:37:09+01:00
updated: 2026-10-31T06:37:29+01:00
---

# CloakBrowser-Manager

CloakBrowser-Manager is the orchestration layer over [[projects/cloakbrowser/cloakbrowser|CloakBrowser]]: a self-hosted, open-source (MIT) GUI that creates, stores and launches browser profiles, each one a persistent, separate identity (fingerprint seed, GPU family, screen, timezone, proxy, cookies, history). It does not implement fingerprint evasion itself; every profile runs the patched CloakBrowser binary, and the Manager's job is per-profile configuration, isolation, lifecycle and access (native window, VNC or CDP). ^[inferred]

The repository given for this vault is `github.com/vieGPT/CloakBrowser-Manager`; its README links the upstream `github.com/CloakHQ/CloakBrowser-Manager`, so this checkout is a fork of it (latest release in the checkout: v0.1.8, 14/10/2026). ^[inferred]

## How it fits together

- **Stack**: FastAPI backend (Python), React + Tailwind frontend, SQLite database, CloakBrowser binary as engine.
- **Platform modes**: Windows and macOS open profiles in native desktop windows; Linux runs in Docker and streams the browser to the web UI through KasmVNC/noVNC.
- **Automation**: every running profile exposes a CDP endpoint through the Manager API (`/api/profiles/<id>/cdp`), so Playwright/Puppeteer can drive the same session a human watches.
- **Licensing**: one CloakBrowser license key for all profiles; concurrency (not profile count) is the limit.

## Structural hubs (ast-extract, 31/10/2026)

`ast-extract` processed 59 files (Python 27, TypeScript 26, shell 5, JavaScript 1) with 761 edges, all `EXTRACTED`. As in CloakBrowser, 7 of the tool's 10 `god_nodes` are test files (`backend/tests/test_api.py` leads with degree 106); the non-test god nodes are `backend/main.py` (57), `backend/database.py` (29) and `backend/artifacts.py` (27).

| Hub | Degree | Subsystem page |
|---|---|---|
| `backend/main.py` | 57 | [[projects/cloakbrowser-manager/subsystems/api-server\|API server]] |
| `backend/database.py` | 29 | [[projects/cloakbrowser-manager/subsystems/profile-database\|Profile database]] |
| `backend/artifacts.py` | 27 | [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts\|Profile files and artifacts]] |
| `backend/models.py` | 21 | [[projects/cloakbrowser-manager/subsystems/api-models\|API models]] (18 classes share the `BaseModel` parent) |
| `backend/browser_manager.py` | 19 | [[projects/cloakbrowser-manager/subsystems/browser-manager\|Browser manager]] |

## Dependencies

Top external imports in non-test code (ast-extract): `lucide-react` (13), `react` (12), `httpx` (2), `tailwindcss` (1), `vite` (1). `starlette` also appears (5 edges, mostly in tests). FastAPI is confirmed as a direct dependency: `backend/requirements.txt` (commit `11e0043`) pins `fastapi>=0.115.0`, alongside `uvicorn[standard]`, `pydantic>=2.0`, `cloakbrowser[geoip]>=0.6.3`, `websockets`, `httpx` and `pywebview`; ast-extract simply did not surface it among the import edges (see [[projects/cloakbrowser-manager/architecture/manager-architecture|Manager architecture]]).

## Key pages

- [[projects/cloakbrowser-manager/architecture/manager-architecture|Manager architecture]]
- [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|Per-profile identity isolation]]
- [[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage|Cross-profile linkage]]
- [[projects/cloakbrowser-manager/subsystems/frontend-ui|Frontend UI]]
- [[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale|Per-profile network and locale]]
- [[projects/cloakbrowser-manager/synthesis/manager-cloakbrowser-responsibility-split|Manager vs CloakBrowser responsibility split]]
- [[projects/cloakbrowser-manager/architecture/release-and-packaging|Release and packaging]]
- [[projects/cloakbrowser-manager/subsystems/frontend-api-client|Frontend API client]]
- [[projects/cloakbrowser-manager/subsystems/api-server|API server]]
- [[projects/cloakbrowser-manager/subsystems/browser-manager|Browser manager]]
- [[projects/cloakbrowser-manager/subsystems/profile-database|Profile database]]
- [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts|Profile files and artifacts]]
- [[projects/cloakbrowser-manager/subsystems/api-models|API models]]
- [[projects/cloakbrowser-manager/architecture/authentication-model|Authentication model]]
- [[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying|CDP endpoint proxying]]
- [[projects/cloakbrowser-manager/techniques/download-capture|Download capture]]
- [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state|Profile duplication with state]]
- [[projects/cloakbrowser-manager/subsystems/vnc-viewer|VNC viewer]]
- [[projects/cloakbrowser-manager/subsystems/runtime-and-binary|Runtime and binary]]
- [[projects/cloakbrowser-manager/subsystems/settings-and-license|Settings and license]]
- [[projects/cloakbrowser-manager/subsystems/diagnostics|Diagnostics]]
- [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|Per-profile transparent proxy]]
- [[projects/cloakbrowser-manager/techniques/portable-cookies|Portable cookies]]

## Sources

- `repo:github.com/vieGPT/CloakBrowser-Manager`: `README.md` (commit `11e0043`, 14/10/2026) and `ast-extract` output
