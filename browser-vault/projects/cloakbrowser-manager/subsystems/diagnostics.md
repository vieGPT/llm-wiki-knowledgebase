---
title: >-
    Diagnostics
category: subsystems
tags: [diagnostics, logging, fonts, system-status, profile-manager]
aliases: [manager.log, startup fingerprint, font health, system status badge]
relationships:
  - target: "[[projects/cloakbrowser/detection-vectors/font-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/api-server]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/diagnostics.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/models.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/SystemStatusBadge.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
summary: >-
    Crash-diagnosable manager.log (startup fingerprint, crash hooks, stderr tee, redacted launch lines) plus /api/status with tier, binary and Windows font health behind the top-bar badge.
provenance:
  extracted: 0.8
  inferred: 0.18
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:32:16+01:00
updated: 2026-10-31T06:37:29+01:00
---

# Diagnostics

Diagnostics in the Manager has two audiences. For support, `backend/diagnostics.py` aims to make "an arbitrary future crash diagnosable from the rotating `manager.log` alone, without a repro" (module docstring, commit `11e0043`). For the user, `GET /api/status` feeds the always-visible `SystemStatusBadge` with [[projects/cloakbrowser/subsystems/license-and-tiers|license tier]], binary version and, on Linux Docker, the health of the Windows persona's fonts. The font check exists because a Linux container presenting a Windows persona leaks its real font set unless the Windows fonts are installed ([[projects/cloakbrowser/detection-vectors/font-fingerprinting|font fingerprinting]]). ^[inferred]

## The log (`manager.log`)

- **Location and rotation**: `<data dir>/logs/manager.log`, 2 MB per file with 3 backups. If the data dir is read-only, logging falls back to the console only ([[projects/cloakbrowser-manager/subsystems/api-server|API server]]).
- **Startup fingerprint** (`startup_line`): one `diag:` line with Manager version, Python version, OS and architecture, frozen vs source, the raw stdout/stderr encodings, UI mode, runtime mode, host OS, data dir, license tier and plan, wrapper version and binary version.
  - The stream encodings are captured at process entry, *before* the native entry point reconfigures streams to `errors="replace"`. Otherwise the line would always read "replace" and hide the strict cp1252 encoding that crashed frozen Windows apps (fixed in v0.1.2 and v0.1.5).
  - The Manager version comes from `CB_VERSION`, then a bundled `backend/version.txt`, then `git describe`, else `unknown`. The update check reuses it, and an `unknown` version disables the comparison.
- **Crash hooks**: `sys.excepthook` and `threading.excepthook` log uncaught exceptions as critical with full tracebacks (KeyboardInterrupt and SystemExit excepted); an asyncio exception handler, installed in the lifespan, logs loop and task errors. uvicorn runs in a thread and auto-launch spawns tasks, so without these a crash off the request path would vanish into a frozen app's dead stderr.
- **Stderr tee**: wraps `sys.stderr`, passes writes through and mirrors complete lines to a separate `cloakbrowser.console` logger. This catches what the CloakBrowser wrapper writes straight to stderr (welcome banner, preview-fallback notice, the font warning). Log records do not loop, because the logging handler captured the original stderr before the tee was installed.
- **Launch lines**: each launch logs profile id, seed, the proxy with `user:pass@` stripped (`redact_proxy`), tier, plan and runtime. The license key is never logged. That makes the log itself a file that links every profile's seed and proxy for whoever holds it ([[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage|cross-profile linkage]], operator side). ^[inferred]

All installs are idempotent and run at import of `main.py`, so they cover both the native entry point and Docker's uvicorn.

## System status and the badge

`GET /api/status` (authenticated since v0.1.0; `/api/health` is the open probe, see [[projects/cloakbrowser-manager/architecture/authentication-model|authentication model]]) returns running and total profile counts, binary version, `license_tier`, host OS, runtime and viewer mode, and three font-health fields.

- **Font health** (`_windows_font_health` in `main.py`): runs only when the runtime is Docker on Linux, the one setup where the Windows persona is emulated on a non-Windows host. It imports the wrapper's private Windows font-tell list and counter from `cloakbrowser.browser`, counts the tells present via `fc-list`, and reports present, required and complete. Elsewhere, or on any error (logged as a warning), all three are `null`. The probe runs in a worker thread on every status poll.
- **Badge** (`SystemStatusBadge.tsx`): tier label (`Pro`, `Free key`, `Free (keyless)`) with colour, the binary version, and an amber "Fonts incomplete" warning with a present/required tooltip only when `windows_fonts_complete` is `false`. A keyless tooltip points the user to Settings ([[projects/cloakbrowser-manager/subsystems/settings-and-license|settings and license]]).
- The Docker image installs Microsoft core fonts for this reason, and the counter checks it at runtime ([[projects/cloakbrowser-manager/architecture/platform-modes|platform modes]]).

Font coverage is a per-host property: every profile on one Docker host shares the same font set, so the badge is a host check, not a per-profile one. ^[inferred]

## History (CHANGELOG, dated)

- v0.1.0 (19/08/2026): tier and binary-version badge; `/api/status` requires auth.
- v0.1.2 (19/08/2026): startup fingerprint, crash hooks, redacted launch context, engine output mirrored into `manager.log`.
- v0.1.5 (30/08/2026): engine 0.5.10 makes the legacy-codepage status output safe on Windows.

## Open Questions

- The font check uses private wrapper names (`_WINDOWS_FONT_TELLS`, `_count_fonts_present`); a wrapper rename would turn the badge silently to "unknown" (`null`), with only a log warning. ^[inferred]
- The macOS persona's font tells are not checked by the Manager. ^[inferred]

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager (commit `11e0043`): `backend/diagnostics.py`, `backend/main.py` (logging setup, hook installation, `_windows_font_health`, `/api/status`, update check), `backend/browser_manager.py` (launch log line), `backend/models.py` (`StatusResponse`), `frontend/src/components/SystemStatusBadge.tsx`, `CHANGELOG.md` (0.1.0, 0.1.2, 0.1.5)
