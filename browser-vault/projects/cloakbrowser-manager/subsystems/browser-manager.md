---
title: >-
    Browser manager
category: subsystems
tags: [profile-manager, browser-lifecycle, concurrency, cdp, licensing]
aliases: [backend/browser_manager.py, BrowserManager, ProfileBusyError, RunningProfile]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]"
    type: implements
  - target: "[[projects/cloakbrowser-manager/subsystems/vnc-viewer]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: implements
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/screen-viewport-consistency]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/architecture/manager-architecture]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/CHANGELOG.md
  - ~/code/CloakBrowser-Manager/backend/tests/test_browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_vnc_manager.py
summary: >-
    BrowserManager launches, tracks and stops one CloakBrowser persistent context per profile, maps profile rows to --fingerprint flags, guards dirs with hold_stopped and surfaces license denials.
provenance:
  extracted: 0.84
  inferred: 0.14
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T06:20:23+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Browser manager

`BrowserManager` (`backend/browser_manager.py`, degree 19) is the singleton that turns a profile row into a running CloakBrowser instance and owns every in-memory fact about running browsers: the `running` map of `RunningProfile` records (Playwright context, CDP port, X display and VNC port, preview and download tasks, the wrapper's denial-file path), plus launching, stopping, held and initializing sets guarded by one asyncio lock. It is the only component that calls the [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|CloakBrowser launcher]].

## Launch pipeline

1. **Admission** under the lock: refuse if the profile is held by a filesystem operation or still closing (`ProfileBusyError`), or already running/launching; clear the previous `last_error`.
2. **Display** (Linux Docker only): allocate an X display + websocket port and start KasmVNC at the profile's screen size ([[projects/cloakbrowser-manager/subsystems/vnc-viewer]]).
3. **Profile dir prep**: under Docker, delete stale Chromium `Singleton*` lock files left by an unclean container exit (native mode lets Chromium arbitrate). Write default bookmarks on first launch (folders of detection-test, fingerprint, headers/TLS and reCAPTCHA check sites).
4. **One-time default search engine**: the binary is de-Googled and the authoritative default lives in MAC-protected Secure Preferences, and a hand-written Web Data row has an unforgeable `url_hash`, so the Manager does one short headless launch that drives `chrome://settings/searchEngines` to add Google and click "Make default". A marker file records success or `failed:N`; it gives up after 3 attempts. Status reads `initializing` meanwhile.
5. **Arguments**: `_build_fingerprint_args` (see below), then user `launch_args` (any `--remote-debugging-port/address` is rejected, since the Manager owns debugging), `--remote-debugging-address=127.0.0.1`, and `--restore-last-session` when `restore_session` is on.
6. **Proxy**: normalize `host:port:user:pass` and `host:port` to `http://user:pass@host:port`, validate scheme (http/https/socks5), host and port; with [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|`transparent_proxy`]], also require one `http://` proxy with username and password ([[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale]]).
7. **Launch options** passed to `launch_persistent_context_async`: user-data dir, `headless=False`, proxy, timezone, locale, [[projects/cloakbrowser/subsystems/humanize-engine|humanize]] + preset, geoip, color scheme, extension paths, [[projects/cloakbrowser-manager/subsystems/settings-and-license|license key and release channel]]; under VNC also a [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|viewport]] of screen height minus 133 px and `DISPLAY`.
8. **CDP verification**: reserve an OS-chosen loopback port (unique within the process), launch, then poll `/json/version` until the debugger URL names that port (10 s, 3 attempts). The denial file is checked before and on every poll, so a seat or license denial fails fast instead of waiting out the timeout and is never retried.
9. **Post-launch**: under VNC inject a copy listener for clipboard sync; register the close handler; start the 30 s preview-screenshot loop (JPEG q55 in the profile dir, opt-out `capture_preview`); under Docker start [[projects/cloakbrowser-manager/techniques/download-capture|download capture]].

Any failure swaps the profile from launching to stopping under the lock and tears down context, port and display before re-raising, so `is_active()` never shows a gap while a half-launched Chromium might still hold the directory.

## Profile → engine flag mapping

| Profile field | Engine flag | Notes |
|---|---|---|
| `fingerprint_seed` | `--fingerprint=<seed>` | The whole device derives from it ([[projects/cloakbrowser/techniques/fingerprint-seed-identity]]) |
| (host OS) | `--fingerprint-platform=macos/windows` | Persona is runtime-determined, not editable: macOS hosts present as Mac, Windows and Linux as Windows ([[projects/cloakbrowser/techniques/platform-persona-spoofing]]) |
| `gpu_family` nvidia/intel | `--fingerprint-gpu-vendor=NVIDIA/Intel` | Ignored on macOS, where the seed picks an Apple GPU ([[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]) |
| `screen_width/height` | `--fingerprint-screen-width/height` | |
| `allow_3p_cookies` | `--fingerprint-allow-3p-cookies` | |
| `transparent_proxy` | `--fingerprint-transparent-proxy` | v0.1.8 |
| `portable_cookies` | `--fingerprint-portable-cookies` | v0.1.8 ([[projects/cloakbrowser-manager/techniques/portable-cookies\|portable cookies]]) |
| (VNC mode) | `--use-angle=swiftshader` | Software GL in the container |

## Stop, close and reservations

- `stop()` pops the profile from `running` first, takes a counted stopping reservation, captures a final preview, cancels tasks, awaits the download watcher, fails any still-pending downloads, closes the context, stops VNC and releases the CDP port.
- `_on_browser_closed` handles crashes and user-closed windows; it acts only if the closing `RunningProfile` is still the registered one, so a late close event from a superseded launch cannot tear down a healthy relaunch. It reads the denial file to record a post-handshake license denial as `last_error` (CHANGELOG 0.1.4 also fixed a ~130 MB helper-process leak on this path).
- Stopping reservations are counted, so one operation's cleanup never releases another's.
- `hold_stopped(profile_id)` is the exclusive lock for [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state|duplicate]], reset and delete: it refuses while a browser is launching, running or closing, and launch refuses while it is held. This is `ProfileBusyError`'s main source.

## Binary and license status

`resolve_binary_status()` runs once at startup and after a settings change: it validates the key via the wrapper, sets tier `keyless`/`free`/`pro`, looks up the latest Pro version for the channel, and pre-downloads the binary via `ensure_binary`. It never raises; on failure the keyless binary stays usable ([[projects/cloakbrowser/subsystems/license-and-tiers]], [[projects/cloakbrowser/subsystems/binary-download-and-verification]]).

## Proxy test

`test_proxy` reuses the launch-time normalize/validate, resolves the exit IP through the wrapper's GeoIP echo services, then best-effort maps it to country, city and timezone with the local GeoLite2 database ([[projects/cloakbrowser/subsystems/geoip-resolver]]).

## Pinned by tests

`backend/tests/test_browser_manager.py` and the status tests in `test_vnc_manager.py` (as of commit `11e0043`) confirm the description above and add these details:

- **Proxy normalisation edge cases.** URLs that already have a scheme (`http`, `https`, `socks5`) pass through unchanged. `host:port:user:pass` becomes `http://user:pass@host:port` even with an empty host (`:8080:user:pass` → `http://user:pass@:8080`). Strings with three or five colon-separated parts are returned unchanged, and the validator then rejects them. Validation errors are "Invalid proxy scheme 'ftp'", "missing hostname" and "missing port".
- **Transparent-proxy gate.** Only an `http://` proxy with both username and password is accepted. `None`, `socks5://` and `https://` proxies are rejected with "Transparent proxy needs…", as are a proxy with no credentials, a username without a password, and a username containing a percent-encoded colon (`us%3Aer`).
- **Flag set is minimal.** An empty Docker profile yields exactly two args, `--use-angle=swiftshader` and `--fingerprint-platform=windows`. The legacy `--disable-infobars` and `--test-type` are gone. `gpu_family: auto` adds no vendor flag. User `launch_args` always come after the managed flags, and any user `--remote-debugging-*` raises "Manager owns remote debugging" with no port or launching state left behind.
- **Runtime split.** A native launch never touches `VNCManager`, injects no init script, passes no `env` or `viewport`, uses `headless=False` and `--remote-debugging-address=127.0.0.1`, and gets no software GL. A Docker launch gets `DISPLAY=:100`, a viewport of 1920×947 for a 1080-px screen, and exactly one clipboard init script ([[projects/cloakbrowser-manager/subsystems/vnc-viewer]]).
- **CDP verification.** A failed first CDP check closes the first context and relaunches (two launch calls). A debugger answering on the wrong port is treated as not ready (`TimeoutError … was not ready`). Ports are unique within the process, and releasing one is idempotent.
- **Lifecycle windows.** A launch that fails *after* the browser is up stays `is_active()` until its context has closed, so `hold_stopped()` raises `ProfileBusyError` in that window. A relaunch while the previous browser is still closing is refused ("still closing") without allocating a display. The native close event removes the profile from `running` and frees its CDP port.
- **Status shape.** `get_status()` returns `status`, `runtime_mode`, `viewer_mode`, `vnc_ws_port`, `display` (as `":100"`), `cdp_url` (`/api/profiles/<id>/cdp`) and `last_error`. A stopped profile reports `null` for all the per-run fields.
- **Profile defaults on disk.** First launch seeds a `Bookmarks` file with exactly four folders: "Detection Tests", "Fingerprint", "Headers & TLS" and "reCAPTCHA". It never writes `Preferences`, because the search engine cannot be set there. The seeding is idempotent and never overwrites an existing file. Reset keeps the search-engine marker, so no Google rebuild runs on the next launch.
- **Downloads.** Capture starts only under Docker; native browsers keep writing to the user's own Downloads folder. Stopping a browser fails its still-pending download rows immediately, rather than at the next launch. A late detach notification from a superseded browser never re-arms the current one ([[projects/cloakbrowser-manager/techniques/download-capture]]).

## Open Questions

- Resolved: the wrapper's `.cloakbrowser-seed` mechanism ([[projects/cloakbrowser/subsystems/profile-seed]]) treats an explicit `--fingerprint=<seed>` as an opt-out and neither reads nor writes the file, so the Manager's database seed always wins and reset needs no seed-file cleanup. ^[inferred]

## Class stubs (ast-extract)

- [[projects/cloakbrowser-manager/subsystems/profile-busy-error|ProfileBusyError]]

## Sources

- ~/code/CloakBrowser-Manager/backend/browser_manager.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/backend/main.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/CHANGELOG.md (commit 11e0043, entries 0.1.0 to 0.1.8)
- ~/code/CloakBrowser-Manager/backend/tests/test_browser_manager.py, test_vnc_manager.py (commit 11e0043)
