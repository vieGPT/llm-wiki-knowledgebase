---
title: >-
    Platform modes
category: architecture
tags: [docker, kasmvnc, native-app, deployment, runtime]
aliases: [runtime mode, viewer mode, native vs Docker]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/vnc-viewer]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/architecture/release-and-packaging]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/font-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/screen-viewport-consistency]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/download-capture]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/architecture/manager-architecture]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/Dockerfile
  - ~/code/CloakBrowser-Manager/run.py
  - ~/code/CloakBrowser-Manager/app_entry.py
  - ~/code/CloakBrowser-Manager/backend/runtime.py
  - ~/code/CloakBrowser-Manager/packaging/manager.spec
  - ~/code/CloakBrowser-Manager/frontend/src/lib/api.ts
  - ~/code/CloakBrowser-Manager/CHANGELOG.md
  - ~/code/CloakBrowser-Manager/.env.example
  - ~/code/CloakBrowser-Manager/entrypoint.sh
  - ~/code/CloakBrowser-Manager/docker-compose.yml
  - ~/code/CloakBrowser-Manager/backend/tests/test_runtime.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_launcher.py
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/backend/artifacts.py
summary: >-
    The host decides the mode: Windows/macOS run natively with real browser windows and a pywebview shell; Linux runs only in Docker, streaming each profile over KasmVNC.
provenance:
  extracted: 0.82
  inferred: 0.15
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:19:30+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Platform modes

The Manager has two runtime modes, and the host OS picks one; the user does not (as of 14/10/2026, commit `11e0043`). `backend/runtime.py` defaults Linux to `docker` and Windows/macOS to `native`, and rejects any other pairing set through `CLOAKBROWSER_MANAGER_RUNTIME`. The viewer mode follows from the runtime mode: `docker` means `vnc`, `native` means `native-window`. The frontend receives `host_os`, `runtime_mode` and `viewer_mode` in `SystemStatus` and on every profile ([[projects/cloakbrowser-manager/subsystems/frontend-api-client|frontend API client]]), and uses them to choose between a native-window status card and the embedded [[projects/cloakbrowser-manager/subsystems/vnc-viewer|VNC viewer]].

| | Native (Windows, macOS) | Docker (Linux) |
|---|---|---|
| How it ships | PyInstaller app in an Inno Setup installer or a universal `.dmg`, or `run.py` from source | Multi-arch image (amd64, arm64) |
| Manager UI | pywebview window (WKWebView / WebView2) or a browser tab at `127.0.0.1:8080` | Web UI on port 8080, reachable remotely |
| Profile browser | A real desktop window on the host | Headful Chromium on a per-profile virtual X display, streamed through KasmVNC and noVNC |
| Clipboard sync | Not applicable (hidden since v0.1.0) | Relayed into and out of the guest |
| [[projects/cloakbrowser-manager/techniques/download-capture\|Download capture]], upload sidebar | Not applicable: the browser shares the host filesystem | Captured into the profile's files |
| Chromium binary | Downloaded on first launch | Pre-downloaded at image build |

## Native mode

- **One code path for dev and frozen builds.** `run.py` sets `CLOAKBROWSER_MANAGER_RUNTIME=native` and execs `app_entry.py`, which is also the PyInstaller entry point. It runs uvicorn in-process on 127.0.0.1:8080 and shows the UI in a pywebview window, or in the default browser if pywebview is missing or `CLOAKBROWSER_MANAGER_UI=browser` is set.
- **Local only.** Binding to 127.0.0.1 and refusing to start when 8080 is taken (plus `LSMultipleInstancesProhibited` on macOS) keeps one Manager per machine. ^[inferred]
- `run.py` refuses Linux outright ("use Docker on Linux").
- Because JS `window.open` does nothing inside the webview, external links go through `/api/open-external`, which opens the host's default browser.

## Docker mode

From the `Dockerfile`:

- Two stages: a Node 20 frontend build, then a `python:3.12-slim` runtime with the Chromium shared libraries, Playwright's Chromium system deps, `xclip` for the clipboard relay, and Mesa GL/EGL drivers.
- **Microsoft core fonts** (Arial, Times New Roman, Verdana and others) are installed from Debian contrib with the EULA preseeded. A Linux container that presents a Windows-like persona would otherwise expose a Linux font set ([[projects/cloakbrowser/detection-vectors/font-fingerprinting|font fingerprinting]]). ^[inferred] The font-health counters in `SystemStatus` (`windows_fonts_present/required/complete`) check this at runtime ([[projects/cloakbrowser-manager/subsystems/diagnostics|diagnostics]]).
- **KasmVNC 1.3.3**, with the `.deb` chosen by `TARGETARCH`, so one Dockerfile serves amd64 and arm64.
- The CloakBrowser binary is fetched at build time (`ensure_binary`), which keeps the first launch off the network. ^[inferred]
- `EXPOSE 8080`, `VOLUME /data` for persistent profiles, a health check every 30 s against the unauthenticated `/api/health`, and `entrypoint.sh` as entry point.

Because the Docker UI can be reached over the network, `AUTH_TOKEN` matters mainly here (see [[projects/cloakbrowser-manager/architecture/authentication-model|authentication model]]). ^[inferred]

## Configuration surface (`.env`, both modes)

`.env.example` lists the settings a `.env` beside the Manager can hold. The file is loaded automatically at startup in both Docker and native mode and must never be committed. On native builds the license key and channel can also be set in the in-app Settings panel ([[projects/cloakbrowser-manager/subsystems/settings-and-license|settings and license]]).

| Variable | Meaning |
|---|---|
| `CLOAKBROWSER_LICENSE_KEY` | One CloakBrowser Pro key per Manager instance; all launched profiles share its concurrency-seat pool. Blank runs the free keyless build ([[projects/cloakbrowser/subsystems/license-and-tiers\|license and tiers]]) |
| `CLOAKBROWSER_RELEASE_CHANNEL` | `stable` (default) or `preview`; preview needs a valid key and is ignored on the keyless build |
| `AUTH_TOKEN` | Optional bearer token or cookie gate for the whole Manager; blank leaves it open |
| `ARTIFACT_MAX_BYTES`, `ARTIFACT_MAX_COUNT`, `ARTIFACT_MAX_TOTAL_BYTES` | Per-profile file limits, read once at startup; defaults 256 MiB per file, 200 files, 2 GiB total (as of commit `11e0043`). Nothing is evicted: a file over the per-file limit gets 413, one that would breach the others gets 507, and a non-numeric value stops the Manager from starting ([[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts\|profile files and artifacts]]) |

## Pinned by tests

`backend/tests/test_runtime.py` and `test_launcher.py` (as of commit `11e0043`) confirm the following:

- **Host detection.** `win32`, `darwin` and `linux` map to `windows`, `macos` and `linux`. Any other platform (e.g. `freebsd`) raises "Unsupported operating system".
- **Mode pairing.** Windows and macOS resolve to `native` + `native-window`. Linux resolves to `docker` + `vnc`. Forcing `CLOAKBROWSER_MANAGER_RUNTIME=native` on Linux raises "Native Linux desktop mode is not supported". Forcing `docker` on macOS raises "Docker runtime is supported only on Linux".
- **Data-dir defaults.** Windows uses `%LOCALAPPDATA%\CloakBrowser Manager`. macOS uses `~/Library/Application Support/CloakBrowser Manager`. Linux/Docker uses `/data`.
- **Data-dir overrides.** `CLOAKBROWSER_MANAGER_DATA_DIR` wins, and the legacy `CLOAKBROWSER_DATA_DIR` is still honoured. A bare `DATA_DIR`, which appears in the override test, is not among the variables `default_data_dir` reads. The test only proves the canonical variable beats it.
- **Native launcher.** `run.py` `_ensure_environment()` raises "use Docker on Linux". `run-windows.bat` and `run-macos.sh` (`python3 run.py`) are thin wrappers over the same runner.

## Docker entrypoint

`entrypoint.sh` starts **no** display server itself. Displays are started per profile, on demand, by `VNCManager` ([[projects/cloakbrowser-manager/subsystems/vnc-viewer]]). On every container start the entrypoint:

1. creates `/data/profiles`;
2. kills stale `Xvnc`, CloakBrowser/Chromium and `xclip` processes;
3. deletes Chrome `Singleton*` locks left on the persistent volume, along with `/tmp/.X1*-lock`;
4. `exec`s uvicorn on `0.0.0.0:8080`.

`docker-compose.yml` publishes that port on host loopback only and bind-mounts `~/.cloakbrowser-manager` as `/data` (the README's `docker run` uses a named `cloakprofiles` volume instead). Compose renders an unset artifact limit as an empty string, which means "use the default" (pinned by `test_blank_limit_settings_mean_the_defaults`).

## Mode-dependent backend behaviour (commit 11e0043)

The single `RuntimeConfig` drives several branches outside the viewer:

| Behaviour | Native | Docker |
|---|---|---|
| Stale `Singleton*` locks in a profile dir | Left to Chromium, which arbitrates ownership itself | Deleted before each launch (an unclean container exit leaves them) |
| Rendering | Host GPU | `--use-angle=swiftshader` (software GL) |
| Viewport | Window-managed | Screen height minus 133 px inside the VNC display ([[projects/cloakbrowser/detection-vectors/screen-viewport-consistency]]) |
| [[projects/cloakbrowser/techniques/platform-persona-spoofing\|Persona flag]] | `macos` on macOS hosts, `windows` on Windows | `windows` |
| Windows font health in `/api/status` | Not reported | Reported (Linux host emulating a Windows persona) |
| File-chooser view and GTK bookmarks | Off (`$HOME` is a real user's; Windows restricts symlinks) | On |
| Shutdown from UI | Flips the in-process uvicorn server's exit flag | Sends SIGINT to itself (dev `uvicorn` path) |

Native-only shell details from `app_entry.py`: stdout/stderr are switched to `errors="replace"` before anything prints, because a frozen Windows app has a cp1252 strict console and a non-ASCII glyph in the wrapper's download banner used to abort launches (CHANGELOG 0.1.2, 0.1.5); window size and position persist in a separate `window.json` so geometry writes can never clobber `settings.json`; on Windows a second start focuses the existing window by title.

## Open Questions

- The offline-safe startup mentioned for v0.1.7 shows up in `VNCManager` as `-publicIP 127.0.0.1`, which skips KasmVNC's STUN lookup; the entrypoint has nothing to do with it. Whether v0.1.7 changed anything else for offline startup is not covered here. ^[inferred]

## Sources

Commit `11e0043` of `~/code/CloakBrowser-Manager`:

- Dockerfile, run.py, .env.example, packaging/manager.spec, frontend/src/lib/api.ts
- app_entry.py, backend/runtime.py, CHANGELOG.md (consulted)
- backend/browser_manager.py, backend/main.py, backend/artifacts.py (M0 batch: mode-dependent behaviour)
- entrypoint.sh, docker-compose.yml, backend/tests/test_runtime.py, backend/tests/test_launcher.py
