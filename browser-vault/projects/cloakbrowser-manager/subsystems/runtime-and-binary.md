---
title: >-
    Runtime and binary
category: subsystems
tags: [runtime, deployment, licensing, versioning, profile-manager]
aliases: [backend/runtime.py, RuntimeConfig, resolve_runtime, bundle_dir, resolve_binary_status]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/architecture/platform-modes]]"
    type: implements
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/runtime.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_runtime.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_launcher.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/conftest.py
summary: >-
    backend/runtime.py fixes the host runtime (native vs Docker), data dir and bundle root; BrowserManager.resolve_binary_status picks tier and engine version and pre-downloads the binary.
provenance:
  extracted: 0.86
  inferred: 0.12
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:27:10+01:00
updated: 2026-10-31T06:37:29+01:00
---

# Runtime and binary

Two small pieces decide *where* the Manager runs and *which* CloakBrowser engine it runs (as of commit `11e0043`). `backend/runtime.py` turns the host OS into a frozen `RuntimeConfig(host_os, runtime_mode, viewer_mode, data_dir)` and nothing else; every other module reads that one object. `BrowserManager.resolve_binary_status()` turns the [[projects/cloakbrowser-manager/subsystems/settings-and-license|license key and release channel]] into a tier, an engine version and a pre-downloaded binary, by calling the CloakBrowser wrapper ([[projects/cloakbrowser/subsystems/binary-download-and-verification]], [[projects/cloakbrowser/subsystems/license-and-tiers]]). How the two modes differ in practice is covered on [[projects/cloakbrowser-manager/architecture/platform-modes]].

## `backend/runtime.py`

- **`detect_host_os`.** It maps `win32`, `darwin` and `linux*` to `windows`, `macos` and `linux`, and raises "Unsupported operating system" for anything else.
- **`resolve_runtime`.** The default mode is `docker` on Linux and `native` elsewhere. `CLOAKBROWSER_MANAGER_RUNTIME` may only restate that default. Any other value raises: "must be 'native' or 'docker'", "Native Linux desktop mode is not supported", or "Docker runtime is supported only on Linux". The viewer mode follows the runtime: `docker` → `vnc`, `native` → `native-window`.
- **`default_data_dir`.** It returns the state directory without creating it:
  1. `CLOAKBROWSER_MANAGER_DATA_DIR` if set;
  2. otherwise the legacy `CLOAKBROWSER_DATA_DIR`;
  3. otherwise the platform default: `%LOCALAPPDATA%\CloakBrowser Manager` (or `~/AppData/Local/...`), `~/Library/Application Support/CloakBrowser Manager`, or `/data` in Docker.
- **`bundle_dir`.** It returns the resource root. In a PyInstaller/Nuitka freeze that is `sys._MEIPASS`, or the executable's directory. From source it is the repo root, which holds `backend/` and `app_entry.py`. `FRONTEND_DIR` is `bundle_dir()/frontend/dist`.
- **Why a policy object rather than flags.** Code paths test `runtime_mode`/`viewer_mode` rather than `sys.platform`, so the test suite can impersonate either build on any CI host by swapping one `RuntimeConfig` (`native_runtime` and `docker_runtime` fixtures). ^[inferred]

## Engine binary and tier

`resolve_binary_status()` runs in a thread at startup and again after every settings change. It never raises.

1. **Base version.** The keyless baseline version comes from `cloakbrowser.config.get_chromium_version()`, falling back to the constant `CHROMIUM_VERSION`.
2. **Tier.** A key resolved via `resolve_license_key` is checked with `validate_license`. A valid key sets the tier to `free` or `pro` (otherwise `keyless`) and asks `get_pro_latest_version(channel)` for the version to report.
3. **Pre-download.** It calls `ensure_binary(license_key, release_channel)`, so a multi-hundred-MB Pro download happens off the launch path and outside auto-launch's 60 s timeout. On failure, the keyless binary baked into the image or install remains usable.

The **release channel** is `stable` or `preview`. The default comes from `CLOAKBROWSER_RELEASE_CHANNEL` (`stable` in `docker-compose.yml`) or `settings.json`, and an environment value beats the stored setting. Channel semantics belong to CloakBrowser ([[projects/cloakbrowser/architecture/release-channels-and-versioning]]).

## Pinned by tests

- **`test_runtime.py`.** It covers every row above: host mapping, the three defaults, both rejected pairings, the override precedence and the Windows `LOCALAPPDATA` path. The override test also sets a bare `DATA_DIR`. That variable is not among those `default_data_dir` reads, so it is simply ignored.
- **`test_launcher.py`.** The native `run.py` refuses Linux ("use Docker on Linux"), and both platform scripts call the shared `run.py`.
- **`test_api.py`.**
  - `bundle_dir()` resolves to the repo root when unfrozen.
  - `/api/status` reports `binary_version` from the wrapper (`0.0.0-test` under the mock).
  - `PUT /api/settings` re-runs `resolve_binary_status()` exactly once.
  - An unknown channel such as `nightly` returns 400 and changes nothing ([[projects/cloakbrowser-manager/subsystems/settings-and-license]]).
- **`conftest.py`.** It registers a fake `cloakbrowser` package (`launch_persistent_context_async`, `config`, `download.ensure_binary`, and `license.*` including `CloakBrowserLicenseError`, `license_error_for_code` and `read_denial_file`) before any backend import. These are the wrapper modules the backend needs at import and startup. Other wrapper helpers (GeoIP for the proxy test, the font check) are imported lazily and are not mocked here. ^[inferred]

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager/backend/runtime.py, backend/browser_manager.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_runtime.py, test_launcher.py, test_api.py, conftest.py (commit 11e0043)
