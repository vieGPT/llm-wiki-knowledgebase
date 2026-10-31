---
title: >-
    VNC viewer (KasmVNC displays, RFB proxy, clipboard relay)
category: subsystems
tags: [vnc, docker, remote-desktop, clipboard, websocket]
aliases: [VNCManager, vnc_manager, KasmVNC viewer, RFB proxy]
sources:
  - ~/code/CloakBrowser-Manager/backend/vnc_manager.py
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_rfb.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_vnc_manager.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_api.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_models.py
  - ~/code/CloakBrowser-Manager/entrypoint.sh
  - ~/code/CloakBrowser-Manager/docker-compose.yml
  - ~/code/CloakBrowser-Manager/Dockerfile
  - ~/code/CloakBrowser-Manager/frontend/package.json
  - ~/code/CloakBrowser-Manager/README.md
  - ~/code/CloakBrowser-Manager/frontend/src/components/ProfileViewer.tsx
summary: >-
    Linux/Docker-only live viewer: one KasmVNC Xvnc display per running profile, a FastAPI WebSocket proxy that rewrites noVNC's RFB stream for KasmVNC 1.3.3, and a clipboard relay.
relationships:
  - target: "[[projects/cloakbrowser-manager/architecture/platform-modes]]"
    type: implements
  - target: "[[projects/cloakbrowser/subsystems/cloakserve-cdp-server]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/screen-viewport-consistency]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
provenance:
  extracted: 0.82
  inferred: 0.15
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:21:06+01:00
updated: 2026-10-31T10:47:25+01:00
---

# VNC viewer (KasmVNC displays, RFB proxy, clipboard relay)

On Linux the Manager runs in Docker and has no desktop to open browser windows on, so each running profile gets its own virtual X display served by KasmVNC's `Xvnc`. The React UI views it through noVNC, and every byte goes through a FastAPI WebSocket proxy at `/api/profiles/<id>/vnc` (as of commit `11e0043`). The subsystem has three parts: `VNCManager` in `backend/vnc_manager.py` allocates displays and runs the process; an RFB translator in `backend/main.py` fixes protocol mismatches between noVNC 1.4 and KasmVNC 1.3.3; and a clipboard relay moves text in and out of the guest. On Windows and macOS (native-window mode) none of this exists: see [[projects/cloakbrowser-manager/architecture/platform-modes|platform modes]].

## Where it is active

- `BrowserManager` builds `VNCManager(enabled = viewer_mode == "vnc")`. A disabled manager refuses `allocate()` and `start_vnc()` with "KasmVNC is available only in the Linux Docker runtime".
- At startup the FastAPI lifespan calls `validate_available()`. In VNC mode this fails fast with "Linux Manager requires the Docker image with KasmVNC installed" when no `Xvnc` binary is on `PATH`. In native mode it is a no-op.
- When a native profile launches, it gets no display, no WebSocket port, no `DISPLAY` env and no clipboard init script. The API reports `vnc_ws_port: null` and `display: null` for it (`test_native_launch_has_no_vnc_display`).

## Display allocation

- Displays start at `:100` and WebSocket ports at `6100`. The port is always `6100 + (display − 100)`, so one integer identifies the pair.
- `allocate()` takes the lowest free display number under an `asyncio.Lock`, so a display freed by a stopped profile is reused first (the gap-fill test).
- Each slot is a `VNCInstance(display, ws_port, process)`. `process` stays `None` until `start_vnc()` attaches the `Popen`.
- `get_ws_port()` returns `None` for an unknown display. `active_displays` lists the allocated numbers.

## The Xvnc process

`start_vnc()` launches one `Xvnc :<display>`. The geometry comes from the profile's `screen_width` × `screen_height` (default 1920×1080) at depth 24. The flags show the security model:

| Flag | Purpose (per code comments) |
|---|---|
| `-websocketPort <ws_port>`, `-rfbport -1` | WebSocket only; the raw VNC TCP port is disabled |
| `-interface 127.0.0.1` | Listens on loopback only; reachable only through the FastAPI proxy |
| `-SecurityTypes None`, `-DisableBasicAuth` | No VNC-level auth; access control is delegated to the Manager (Origin check plus optional `AUTH_TOKEN`) ^[inferred] |
| `-publicIP 127.0.0.1` | Skips KasmVNC's STUN public-IP discovery (pinned by `test_start_vnc_uses_loopback_public_ip`; the README credits a contributor for "offline VNC startup") |
| `-AlwaysShared` | Several viewers can attach to the same display |
| `-httpd /usr/share/kasmvnc/www` | Required: without it KasmVNC accepts TCP on the WebSocket port but never completes the WebSocket upgrade |

- Output goes to `/tmp/xvnc-<display>.log`. After a 0.5 s grace period, if the process has already exited, `start_vnc()` raises with the log contents. The API turns that into a generic 500 "Failed to launch browser" (`test_launch_failure_500`).
- `stop_vnc()` frees the slot, sends SIGTERM, waits up to 5 s, then kills. `cleanup_all()` runs on shutdown. `cleanup_stale()` runs `pkill -f "Xvnc :[0-9]"` to remove orphans from earlier runs.
- The KasmVNC version is 1.3.3: the Dockerfile installs `kasmvncserver_bookworm_1.3.3` for the build architecture, and the main.py comments name the same version.

## The browser on the display

The Docker launch path (`test_docker_launch_keeps_vnc_display`) passes `DISPLAY=:100` to the browser, a Playwright viewport of 1920×947 for a 1920×1080 screen, and `--use-angle=swiftshader`. Native launches get none of these.

- The 133 px gap between screen height and viewport leaves room for real browser chrome, so window, viewport and screen sizes stay physically plausible. This is the signal described in [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency]]. ^[inferred]
- SwiftShader means WebGL is rendered in software, since a container has no GPU. GPU *identity* still comes from the CloakBrowser `--fingerprint-gpu-vendor` flags ([[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]). ^[inferred]

## WebSocket proxy (`/api/profiles/<id>/vnc`)

1. **Origin guard.** A cross-origin browser connection is closed with code 4403, which protects against cross-site WebSocket hijacking. A same-origin connection, or one with no `Origin` header at all, is accepted (`test_vnc_ws_rejects_cross_origin`, `test_ws_allows_same_origin`, `test_ws_allows_no_origin`).
2. **Auth.** When `AUTH_TOKEN` is set, the ASGI middleware closes unauthenticated WebSockets with 4401. The browser authenticates with the login cookie ([[projects/cloakbrowser-manager/architecture/authentication-model|authentication model]]).
3. **State checks.** Close code 4004 means the profile is not running. Close code 4005 means native-window mode, or no display or port.
4. **Upstream.** The proxy accepts the `binary` subprotocol only if the client asked for it (RFC 6455). It then dials `ws://127.0.0.1:<ws_port>/websockify` with compression off (KasmVNC cannot handle permessage-deflate), pings off (KasmVNC ignores WS pings) and no frame-size limit (full 1080p framebuffers).
5. **Pumps.** Two tasks run, client→VNC and VNC→client. When either one ends, the other is cancelled. The proxy logs whether Xvnc is still alive and copies the last 20 lines of its log into the Manager log, which makes disconnects diagnosable.

### Client → server RFB filter

The problem: noVNC v1.4 sends RFB extension messages that KasmVNC 1.3.3 treats as "unknown message type" and disconnects on. noVNC also packs several RFB messages into one WebSocket frame, so the proxy has to parse message boundaries rather than drop whole frames. The first three binary messages (the RFB handshake) pass through untouched. After that, every frame goes through `_filter_rfb_client_messages`. `test_rfb.py` pins its behaviour:

| Type | Message | Size | Treatment |
|---|---|---|---|
| 0 | SetPixelFormat | 20 | kept |
| 2 | SetEncodings | 4 + 4n | **rewritten** to an allowlist |
| 3 | FramebufferUpdateRequest | 10 | kept |
| 4 | KeyEvent | 8 | kept |
| 5 | PointerEvent | 6 → 11 | **expanded** to KasmVNC's format |
| 6 | ClientCutText | 8 + len | kept |
| 150, 248, 252, 255 | EnableContinuousUpdates, QEMU-like key, xvp, QEMU audio | 10, 10, 4, 4 | skipped, parsing continues |
| other | unknown | — | rest of the frame dropped |

- **Incomplete messages are dropped, never forwarded.** A partial message would desynchronise the stream, because KasmVNC buffers partial reads across frames.
- **Encodings are allowlisted, not blocklisted.** The code says an earlier blocklist had the wrong numbers. The allowlist keeps Raw, CopyRect, RRE, Hextile, Tight and ZRLE, the Cursor (−239) and LastRect (−224) pseudo-encodings, and the Tight quality (−32…−23) and compress (−256…−247) levels. Everything else, such as −260 and −307 in the tests, is stripped and the count is rewritten.
- **PointerEvent** becomes `type, u16 mask, x, y, s16 sx, s16 sy`. The u8 mask is widened, and the scroll deltas are zero because noVNC encodes the wheel as mask bits 3–6, which KasmVNC also accepts.
- **Safety net.** Filtered output whose first byte is not a standard type is refused. Text WebSocket frames are dropped, because they would bypass the filter.

### Server → client: clipboard translation

KasmVNC sends its clipboard as a proprietary BinaryClipboard message (type 180) with MIME-tagged entries. noVNC only understands standard ServerCutText (type 3). The proxy takes the first `text/plain` entry, even if an image entry comes first, and re-emits it as ServerCutText. Messages with no text are dropped. RFB requires Latin-1 for ServerCutText, so characters outside Latin-1 (CJK, emoji) arrive as `?` (`test_build_cut_text_latin1_fallback`).

## Clipboard relay (REST)

`POST` and `GET /api/profiles/<id>/clipboard` give the UI a second clipboard path that avoids the Latin-1 limit. Both return 404 when the profile is not running and 409 outside Docker/VNC mode.

- **Push (host → guest).** Text goes to `xclip -selection clipboard` on the profile's display. xclip stays alive to serve paste requests, so the previous xclip for that display is killed first.
- **Read (guest → host).** Chrome under KasmVNC does not write the X11 clipboard, so the Manager injects an init script into every page in VNC mode. The script records the selection into a page global `window.__clipboardText` on `copy` events and on Ctrl/Cmd+C. The GET handler evaluates that global in each open tab and returns the first non-empty value. If none is set, it falls back to `xclip -o` with a 5 s timeout.
- **Limits.** Reads and writes are both capped at 1 MiB: `ClipboardRequest.text` accepts exactly 1,048,576 characters and rejects one more (`test_models.py`).
- **Per-profile switch.** Profiles carry `clipboard_sync`, which defaults to `true` and can be toggled with `PUT` (`test_profile_clipboard_sync_*`). The UI decides whether to sync; the backend endpoints do not check the flag (confirmed by the client-side bullet below).
- **Client side confirms the UI gate.** `ProfileViewer` runs all three clipboard paths only while connected and `clipboard_sync` is on: it intercepts Ctrl/Cmd+V in the capture phase, pushes the host clipboard with `POST`, then replays a full Ctrl+V key sequence; it listens for noVNC `clipboard` events (the translated type-3 messages); and it polls `GET` every 2 s for Chrome copies. A failed host clipboard read switches sync off. Details on [[projects/cloakbrowser-manager/subsystems/frontend-ui|Frontend UI]] (`frontend/src/components/ProfileViewer.tsx`, commit `11e0043`).
- The init script is a page-visible global plus listeners in the main world. A page script could see it, so Docker profiles carry a small surface that native profiles do not. ^[inferred]

## Container lifecycle (`docker-compose.yml`, `entrypoint.sh`)

- **Compose** builds the local Dockerfile. It publishes port 8080 **on host loopback only** (`127.0.0.1:8080:8080`) and bind-mounts `~/.cloakbrowser-manager` to `/data`. It passes `AUTH_TOKEN`, `CLOAKBROWSER_LICENSE_KEY` and `CLOAKBROWSER_RELEASE_CHANNEL` ([[projects/cloakbrowser-manager/subsystems/settings-and-license]]) (default `stable`) from the manager-root `.env`, plus the per-profile file limits `ARTIFACT_MAX_BYTES`, `ARTIFACT_MAX_COUNT` and `ARTIFACT_MAX_TOTAL_BYTES`. Compose renders an unset limit as an empty string, and `artifacts._env_int` treats a blank value as "use the default" (`test_blank_limit_settings_mean_the_defaults`; see [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]]).
- **Volume difference.** The README's `docker run` uses a named volume, `cloakprofiles`, while compose uses a host bind mount. The two set-ups keep profiles in different places. ^[inferred]
- **Entrypoint.** The entrypoint runs on every container start:
  1. Creates `/data/profiles`.
  2. Kills stale `Xvnc :N`, CloakBrowser/Chromium and `xclip` processes.
  3. Deletes Chrome's `SingletonLock`, `SingletonCookie` and `SingletonSocket` (depth ≤ 2 under `/data/profiles`). These locks survive on the persistent volume after a hard kill and would otherwise block the next launch of that profile. ^[inferred]
  4. Removes `/tmp/.X1*-lock`, the X display locks for `:1xx`.
  5. `exec`s uvicorn on `0.0.0.0:8080` at log level `warning`. Inside the container it binds all interfaces; only the host-side publish restricts it to loopback.
- **Frontend dependency.** The noVNC client is `@novnc/novnc ^1.4.0` in `frontend/package.json` (as of commit `11e0043`). The KasmVNC workarounds above assume that major version.

## Related

- [[projects/cloakbrowser-manager/subsystems/browser-manager]]: owns `VNCManager`, allocates the display at launch and stops it when the profile stops.
- [[projects/cloakbrowser-manager/subsystems/api-server]]: hosts the proxy, the RFB translator and the clipboard routes.
- [[projects/cloakbrowser-manager/subsystems/frontend-ui]]: the noVNC viewer component.
- [[projects/cloakbrowser/subsystems/cloakserve-cdp-server]]: CloakBrowser's own Docker image uses an Xvfb/openbox display behind a CDP multiplexer instead of a VNC viewer.

## Open Questions

- The UI uses both guest→host paths: noVNC `clipboard` events, which carry Latin-1 text with `?` substitutions, and the 2 s `GET` poll, which returns full UTF-8. Which value ends up on the host clipboard when the two disagree (e.g. CJK text) is not settled by the sources. ^[ambiguous]
- Neither the Docker image nor the backend gates `window.__clipboardText` on `clipboard_sync`. Is the global injected even when sync is off? The code injects it whenever `viewer_mode == "vnc"`.
- `-SecurityTypes None` relies on loopback binding. Any process inside the container can reach every profile's display directly.

## Sources

- `~/code/CloakBrowser-Manager` at commit `11e0043`: `backend/vnc_manager.py`, `backend/main.py` (RFB translator, VNC proxy, clipboard routes), `backend/browser_manager.py` (display launch, clipboard init script), `backend/tests/test_rfb.py`, `backend/tests/test_vnc_manager.py`, `backend/tests/test_api.py`, `backend/tests/test_models.py`, `entrypoint.sh`, `docker-compose.yml`, `Dockerfile`, `frontend/package.json`, `README.md`, `frontend/src/components/ProfileViewer.tsx`
