---
title: >-
    Manager vs CloakBrowser responsibility split
category: synthesis
tags: [architecture, orchestration, profile-manager, anti-detect, boundaries]
aliases: [what the Manager adds, Manager-engine boundary]
relationships:
  - target: "[[projects/cloakbrowser/architecture/wrapper-binary-split]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/proxy-signal-removal]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser/architecture/release-channels-and-versioning]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/persistent-profile-warmup]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/cloakserve-cdp-server]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/diagnostics]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/README.md
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/database.py
  - ~/code/CloakBrowser-Manager/frontend/src/App.tsx
  - ~/code/CloakBrowser-Manager/frontend/src/components/ProfileForm.tsx
  - ~/code/CloakBrowser-Manager/frontend/src/components/ProfileViewer.tsx
  - ~/code/CloakBrowser-Manager/frontend/src/components/SystemStatusBadge.tsx
  - ~/code/CloakBrowser-Manager/frontend/src/components/SettingsPanel.tsx
summary: >-
    Page-by-page map of which CloakBrowser subsystems and techniques the Manager relies on, and what it adds: profile storage, lifecycle, viewers, files, auth and license UI. No fingerprint logic.
provenance:
  extracted: 0.66
  inferred: 0.3
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:22:37+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Manager vs CloakBrowser responsibility split

[[projects/cloakbrowser-manager/cloakbrowser-manager|CloakBrowser-Manager]] is a profile manager over [[projects/cloakbrowser/cloakbrowser|CloakBrowser]]. The split is sharp: **every fingerprint, network-stealth and input-humanization mechanism lives in CloakBrowser** (binary plus Python wrapper), and the Manager imports the wrapper (`launch_persistent_context_async`, `cloakbrowser.license`, `cloakbrowser.download`, `cloakbrowser.geoip`) and drives it with per-profile parameters. What the Manager adds is persistence, multi-profile lifecycle, viewing, file transfer, access control and a GUI (as of 31/10/2026, commit `11e0043`). This mirrors CloakBrowser's own [[projects/cloakbrowser/architecture/wrapper-binary-split|wrapper/binary split]], one layer higher: the binary holds the evasion, the wrapper flags and launches it, and the Manager decides which flags each profile gets. ^[inferred]

## What the Manager uses, page by page

| CloakBrowser page | How the Manager relies on it | What the Manager adds on top |
|---|---|---|
| [[projects/cloakbrowser/subsystems/launcher-and-stealth-args\|Launcher and stealth args]] | Every launch goes through the wrapper's persistent-context launcher with `user_data_dir`, `headless=False`, proxy, timezone, locale, humanize, GeoIP, colour scheme, extensions, license key and release channel; extra `--fingerprint-*` flags travel in `args` | Builds the per-profile argument list from stored settings; reserves a `--remote-debugging-port` per launch and retries on CDP start failure |
| [[projects/cloakbrowser/techniques/fingerprint-seed-identity\|Fingerprint seed identity]] | Passes `--fingerprint=<seed>`; the binary derives the whole device | Stores the seed in SQLite (random 10000-99999 when blank), re-rolls it on Reset, copies it on [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state\|Duplicate]] |
| [[projects/cloakbrowser/subsystems/profile-seed\|Profile seed]] | Not used in practice: an explicit seed opts out of the `.cloakbrowser-seed` file ^[inferred] | The database row replaces the seed file as the identity store ^[inferred] |
| [[projects/cloakbrowser/techniques/platform-persona-spoofing\|Platform persona spoofing]] | Passes `--fingerprint-platform` | Chooses it from the host runtime (`macos` on Mac, else `windows`), never from profile data |
| [[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting\|WebGL/GPU fingerprinting]] | Passes `--fingerprint-gpu-vendor=NVIDIA/Intel` | GPU-family picker, disabled on macOS; adds `--use-angle=swiftshader` in VNC mode |
| [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency\|Screen/viewport consistency]] | Passes `--fingerprint-screen-width/height` | Resolution presets; in VNC mode sets the viewport to the screen height minus 133 px of browser chrome |
| [[projects/cloakbrowser/subsystems/geoip-resolver\|GeoIP resolver]] / [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching\|GeoIP matching]] | Forwards `geoip`, timezone, locale; the proxy test calls the wrapper's exit-IP resolver and GeoLite2 database directly | Per-profile switch (on by default) and a "Test" button showing exit IP, place and latency |
| [[projects/cloakbrowser/subsystems/proxy-handling\|Proxy handling]] / [[projects/cloakbrowser/techniques/proxy-signal-removal\|Proxy signal removal]] | Forwards the proxy; `--fingerprint-transparent-proxy` per profile | Normalizes and validates the proxy string; enforces "http:// with username and password" for [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy\|transparent mode]] |
| [[projects/cloakbrowser/techniques/persistent-profile-warmup\|Persistent profile warmup]] | Persistent user-data directory per profile; `--fingerprint-portable-cookies`, `--fingerprint-allow-3p-cookies` | Per-profile toggles ([[projects/cloakbrowser-manager/techniques/portable-cookies\|portable cookies]]), `--restore-last-session`, first-launch bookmarks and default search setup |
| [[projects/cloakbrowser/subsystems/humanize-engine\|Humanize engine]] | Forwards `humanize` and `human_preset` (`default`/`careful`) | Checkbox and preset picker only |
| [[projects/cloakbrowser/subsystems/license-and-tiers\|License and tiers]] | Validates the key, resolves the Pro version, reads the wrapper's per-launch denial file and maps codes to `CloakBrowserLicenseError` | One app-wide key shared by all profiles; [[projects/cloakbrowser-manager/subsystems/settings-and-license\|settings panel]]; tier badge (`pro`/`free`/`keyless`); HTTP 402/403 denial banner with upgrade link |
| [[projects/cloakbrowser/subsystems/binary-download-and-verification\|Binary download]] / [[projects/cloakbrowser/architecture/release-channels-and-versioning\|Release channels]] | Calls `ensure_binary` once at startup (off the launch path); Stable/Preview channel | Channel picker in Settings; keeps the baked-in keyless binary as fallback if the Pro download fails |
| [[projects/cloakbrowser/detection-vectors/font-fingerprinting\|Font fingerprinting]] | Relies on host fonts for the Windows persona | Surfaces Windows persona font completeness in the [[projects/cloakbrowser-manager/subsystems/diagnostics\|status badge]] |
| [[projects/cloakbrowser/subsystems/cloakserve-cdp-server\|cloakserve CDP server]] | **Not used.** The Manager runs its own per-profile CDP endpoint (`/api/profiles/<id>/cdp`) over the browser it launched ^[inferred] | Stable CDP URL per profile ([[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]), so humans and automation share one live session |

## What the Manager adds that CloakBrowser does not have

- **Profile store**: SQLite rows with identity, network, behaviour and organisation fields (tags, notes, manual order). See [[projects/cloakbrowser-manager/subsystems/profile-database|Profile database]].
- **Lifecycle across many profiles**: launch, stop, reset, duplicate (with or without browser state), auto-launch on start, status polling, concurrency bounded by license seats. See [[projects/cloakbrowser-manager/subsystems/browser-manager|Browser manager]] and [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|Per-profile identity isolation]].
- **Viewing**: native windows on Windows/macOS, KasmVNC streamed into the web UI on Linux, with clipboard sync and periodic preview screenshots. See [[projects/cloakbrowser-manager/subsystems/vnc-viewer|VNC viewer]] and [[projects/cloakbrowser-manager/subsystems/frontend-ui|Frontend UI]].
- **Files**: uploads into a profile, capture of browser downloads. See [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts|Profile files and artifacts]].
- **Access**: optional single-token auth for UI and API, update check, shutdown. See [[projects/cloakbrowser-manager/architecture/authentication-model|Authentication model]].

## Boundary observations

- The Manager owns *policy* (which flags, from which data), CloakBrowser owns *mechanism* (what a flag does). A detection-side failure is therefore usually an engine issue; a coherence failure (wrong persona for the host, a GPU family that contradicts the platform, two profiles with one seed) is a Manager issue. ^[inferred]
- The Manager narrows CloakBrowser's surface deliberately: platform persona and GPU on macOS are not user-editable, so users cannot build personas the host cannot render coherently. ^[inferred]
- The "Advanced launch arguments" box reopens the full surface: its own help text says these flags "can override Manager-controlled behavior". ^[inferred]
- CDP use by the Manager itself (clipboard polling every 2 s via Playwright CDP in VNC mode, periodic screenshots) adds protocol traffic that CloakBrowser's [[projects/cloakbrowser/techniques/cdp-traffic-minimization|CDP traffic minimization]] advice tries to keep low. Whether this is measurable by pages is not tested in the sources. ^[ambiguous]
- In VNC mode the Manager also injects an init script that keeps the copied text in a page-visible global for the clipboard relay ([[projects/cloakbrowser-manager/subsystems/vnc-viewer|VNC viewer]]). That is a main-world artefact the engine itself does not add, and native-window profiles do not carry it. ^[inferred]

## Open Questions

- Which CloakBrowser wrapper version the Manager pins is not visible from the files read here (see `backend/requirements.txt`).
- Whether the Manager ever uses cloakserve (for example in Docker) was not confirmed beyond the browser-manager code path. ^[ambiguous]

## Sources

- ~/code/CloakBrowser-Manager (commit `11e0043`): `README.md` (Features, license key), `backend/browser_manager.py` (imports from `cloakbrowser`, launch options, `_build_fingerprint_args`, `resolve_binary_status`, `test_proxy`), `backend/database.py` (seed default and re-roll), `frontend/src/App.tsx`, `frontend/src/components/ProfileForm.tsx`, `frontend/src/components/ProfileViewer.tsx`, `frontend/src/components/SystemStatusBadge.tsx`, `frontend/src/components/SettingsPanel.tsx`
