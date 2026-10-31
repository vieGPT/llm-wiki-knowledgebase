---
title: >-
    cloakserve CDP server
category: subsystems
tags: [cdp, docker, multiplexing, fingerprinting, deployment]
aliases: [cloakserve, bin/cloakserve, docker-entrypoint.sh, cloaktest, ChromePool]
relationships:
  - target: "[[projects/cloakbrowser/architecture/framework-integration-modes]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/widevine-cdm-seeding]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/bin/cloakserve
  - ~/code/CloakBrowser/bin/cloaktest
  - ~/code/CloakBrowser/Dockerfile
  - ~/code/CloakBrowser/bin/docker-entrypoint.sh
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/.github/workflows/publish.yml
  - ~/code/CloakBrowser/tests/test_cloakserve.py
summary: >-
    cloakserve, the Docker image's CDP multiplexer: one directly launched Chrome per fingerprint seed behind one port, query-param identities, origin guard, idle reaping, Xvfb/openbox display.
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:46:10+01:00
updated: 2026-10-31T10:47:25+01:00
---

# cloakserve CDP server

`cloakserve` is the server mode of the `cloakhq/cloakbrowser` Docker image: a long-running Python process (`bin/cloakserve`, an aiohttp app) that exposes the Chrome DevTools Protocol on port 9222 so external tools can `connect_over_cdp()` to a stealth browser instead of launching one. Since 0.3.20 it is a **CDP multiplexer**: each distinct fingerprint seed gets its own Chrome process and identity behind the same port. It is the server side of the "connect over CDP" mode in [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]]. The page was first built from the README, CHANGELOG, the container entrypoint and `tests/test_cloakserve.py`; the P2 pass merged the script itself (`bin/cloakserve`, commit b1a00ed), which confirmed nearly all of it and contradicted one point (see "Close endpoint: profile preservation").

## Identity per connection

- Connect to `http://host:9222?fingerprint=<seed>`; each unique seed spawns a separate Chrome with its own coherent identity (see [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]]).
- Supported query params: `fingerprint`, `timezone`, `locale`, `platform`, `platform-version`, `brand`, `brand-version`, `gpu-vendor`, `hardware-concurrency`, `device-memory`, `screen-width`, `screen-height`, `proxy`, `geoip`. Only five are special-cased in code (`fingerprint`, `timezone`, `locale`, `proxy`, `geoip`); every other name is mapped generically to `--fingerprint-<name>=<value>`, so the list is open-ended rather than a whitelist. Blank values are dropped. Since 0.3.23 they route through the wrapper's `build_args()` so `--lang` accompanies a locale (#130). See [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]].
- The same seed reuses the same process and the **first connection's params win**: a later request with different params is served by the running process and only logs a warning.
- **No seed** means the shared `__default__` process. It runs with a random seed from 10000 to 99999 picked when that process launches, so the default identity changes every time the default process is restarted or reaped. ^[inferred]
- **Proxy**: the `proxy` value is re-encoded with the wrapper's SOCKS credential normalizer and passed straight to Chrome as `--proxy-server`, whatever its scheme. There is no Playwright proxy dict to fall back on and no inline-auth version gate here, unlike the wrapper's routing table in [[projects/cloakbrowser/subsystems/proxy-handling|Proxy handling]]. A credentialed HTTP proxy on a binary below the inline-auth floor would therefore not authenticate. ^[inferred]
- **GeoIP** runs only when `geoip` is true **and** a proxy is given; it fills timezone and locale through [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]] and injects the exit IP as `--fingerprint-webrtc-ip` unless the request already set one. WebRTC `auto` is resolved with the wrapper's own helper.
- Server-wide defaults: `--fingerprint=<seed>`, `--fingerprint-locale=` and `--fingerprint-timezone=` on the `cloakserve` command line become defaults that query params override per connection; they are lifted out of the passthrough flags so they too go through `build_args()`. An invalid default seed makes the server exit with status 1 at startup.
- Other extra Chrome flags (for example `--headless=false`) are passed after `cloakserve` on the container command line and appended to every Chrome it launches.

## Process model (`bin/cloakserve`)

- `main()` first calls the wrapper's `ensure_binary()`, so the server resolves, downloads and verifies the binary exactly like a library launch, including the license-key path (see [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]]).
- `ChromePool` keeps one `ChromeProcess` per seed key, guarded by a per-seed asyncio lock so concurrent first connections launch only one browser.
- Chrome is spawned **directly, not through Playwright**. The script therefore adds a base flag set that Playwright would normally contribute (no first run, no default-browser check, `--disable-dev-shm-usage`, extensions and popup blocking off, background networking off, `--metrics-recording-only`, `--ignore-gpu-blocklist`), followed by the stealth args from `build_args(stealth_args=True, …)`, the passthrough flags, and finally its own `--remote-debugging-port`, `--remote-debugging-address=127.0.0.1` and `--user-data-dir=<data_dir>/<seed>`.
- Internal CDP ports are allocated upward from 5100, each probed by binding on 127.0.0.1, so per-seed debug ports are never exposed directly.
- Readiness (as of commit b1a00ed, 26/10/2026): the pool polls the child's `/json/version` with exponential backoff (0.1 s doubling to 1 s) for up to 10 s. On failure it kills the child, deletes its profile and answers HTTP 502 "Chrome failed to start". This is the 502 users saw when Xvfb could not start (#283).
- A process found dead on the next request is cleaned up and relaunched.
- Listen address: `0.0.0.0` inside a container (detected via `/.dockerenv` or Podman's `/run/.containerenv`), otherwise `127.0.0.1`.
- The WebSocket bridge uses aiohttp on the client side and the `websockets` library toward Chrome, with no message-size limit and keep-alive pings disabled; when either direction ends, the other is cancelled.
- The WebSocket routes launch a missing seed with **no** connection params. A client that reuses a rewritten WebSocket URL after its seed was reaped gets a fresh Chrome with server defaults only (no timezone, locale or proxy from the original query), and the browser id in the URL will not match the new process. ^[inferred]

### Routes

| Route | Purpose |
|---|---|
| `GET /` | status JSON: `status`, `active`, `idle_timeout`, and per process its PID, port, seed, connection count, pending cleanup, timezone, locale and proxy |
| `GET /json/version`, `/json/list`, `/json` (each with or without trailing slash) | launch or reuse the seed named in the query, proxy the child's answer, rewrite WebSocket URLs |
| `POST /fingerprint/{seed}/close` | tear one seed down now |
| `GET /fingerprint/{seed}/devtools/{path}` | per-seed WebSocket bridge (registered before the default route) |
| `GET /devtools/{path}` | WebSocket bridge to the default process |

## Lifecycle

- Per-seed processes live until `cloakserve` exits by default. `--idle-timeout=SECONDS` / `CLOAKSERVE_IDLE_TIMEOUT` (`0`, `off`, `false`, `none`, `disabled` turn it off) terminates a seed's Chrome after its last CDP WebSocket disconnects and deletes its temporary profile under `--data-dir` (0.3.32, #352). Since 0.6.0 a browser started via `/json/version` or `/json/list` with no WebSocket client also gets idle cleanup (#558).
- Teardown sends SIGTERM, waits 5 s, then kills. Shutdown of the server cancels pending idle timers and tears down every process.
- `POST /fingerprint/<seed>/close` tears a seed's browser down immediately and frees its slot (Pro session seat, per the CHANGELOG); it answers `{"seed", "terminated"}` with HTTP 200 and is idempotent, returning `terminated: false` for a seed that is not running or not valid (0.5.2). This matters because Playwright's `browser.close()` on a CDP connection only disconnects and leaves Chrome holding the seat.
- The status JSON includes each process's `proxy` string verbatim, so proxy credentials are readable by anyone who can reach `GET /`. ^[inferred]

### Close endpoint: profile preservation

The CHANGELOG (0.5.2) says the close call keeps the profile, and the docstrings of `handle_close` and `terminate_seed` say the same: teardown goes through `_cleanup_process`, "whose `_safe_rmtree` the persistent pool overrides to a no-op". At commit b1a00ed, `bin/cloakserve` contains no persistent pool subclass. The only `ChromePool._safe_rmtree` deletes the seed's directory under `data_dir`, so in this file a close removes the profile just like idle reaping does. ^[ambiguous] The persistent pool may live outside this repository or may have been removed; the code as shipped does not preserve profiles.

## Endpoint rewriting

`/json/version` (optionally `?fingerprint=<seed>`) returns a `webSocketDebuggerUrl` rewritten to point back through cloakserve, e.g. `ws://host:9222/fingerprint/<seed>/devtools/browser/<id>`, so per-seed routing survives frameworks that need a raw WebSocket URL (0.3.32, #234). `/json/list` rewrites every entry the same way. Behind a reverse proxy it honours `Host`, `X-Forwarded-Host` and `X-Forwarded-Proto`, emitting public `wss://` URLs.

## Security posture

- CDP grants full browser control (run JS, read pages, access files). Examples bind to `127.0.0.1:9222`, and on bare metal cloakserve itself binds to `127.0.0.1` (0.3.28).
- Hardening history: seed sanitization against path traversal and Podman detection (0.3.28, #217); WebSocket origin guard against browser-origin CSRF through the CDP proxy (0.3.29, #239/#240).
- The origin guard runs only on the two WebSocket routes. The `POST …/close` and `/json/*` HTTP routes have no origin check, so a web page that can reach the server could still trigger launches or teardowns, though it cannot read CDP. ^[inferred]
- Since 0.3.11 the binary supports `--remote-debugging-address`, so the image no longer needs a socat relay to expose CDP on `0.0.0.0`.

## Container image and entrypoint

The root `Dockerfile` (commit b1a00ed) builds the image in these layers:

1. Base `python:3.12-slim` plus Chromium's runtime libraries, an extended font set (Liberation, Noto Color Emoji, Unifont, FreeFont, IPA Gothic, WenQuanYi Zen Hei, TLWG Loma, URW base35; see [[projects/cloakbrowser/detection-vectors/font-fingerprinting|Font fingerprinting]]), `xvfb`, `xdotool`, `openbox`, and Node.js 20 from the NodeSource setup script.
2. The Python wrapper installed with the `serve` and `geoip` extras, then the JS wrapper built with npm, then `examples/`.
3. The stealth binary **pre-downloaded at build time** with `ensure_binary()`, not at runtime. The welcome-banner marker is deleted afterwards so users see the banner on their first container run.
4. `cloakserve`, `cloaktest` and `fetch-widevine.py` copied to `/usr/local/bin`. `cloaktest` is a three-line shell shortcut that runs `examples/stealth_test.py --no-screenshots` with any extra arguments.
5. `EXPOSE 9222`, `DISPLAY=:99`, entrypoint `/entrypoint.sh`, default command `python`. No `USER` is set, so everything runs as root and the binary lives under `/root/.cloakbrowser` (which the Lambda variant has to make world-readable; see [[projects/cloakbrowser/architecture/aws-lambda-deployment|AWS Lambda deployment]]).

`docker-entrypoint.sh` prepares a headed display before running the user command (`cloakserve`, `cloaktest`, or a script):

1. Deletes a stale `/tmp/.X99-lock` left by a previous container instance; otherwise Xvfb refuses to start after `docker restart` and cloakserve returns 502 forever (#283).
2. Starts `Xvfb :99` at 1920×1080×24 without TCP listening.
3. Polls up to about 10 s with `xdotool` until X accepts connections, instead of a blind sleep that raced under CPU starvation.
4. Starts `openbox`, because bare Xvfb has no window manager and `--start-maximized` would be a silent no-op. Maximized windows are part of the window-geometry coherence story. ^[inferred] See [[projects/cloakbrowser/detection-vectors/screen-viewport-consistency|Screen/viewport consistency]].
5. If `CLOAKBROWSER_FETCH_WIDEVINE` is truthy and the user has neither pointed at a CDM nor disabled seeding, runs `fetch-widevine.py --quiet` best-effort; failure never blocks launch (see [[projects/cloakbrowser/subsystems/widevine-cdm-seeding|Widevine CDM seeding]]).
6. `exec`s the command.

The image ships the free binary; a `CLOAKBROWSER_LICENSE_KEY` env var makes the latest build download at runtime (through the `ensure_binary()` call at cloakserve start). It is built for linux/amd64 and linux/arm64 and Cosign-signed (see [[projects/cloakbrowser/architecture/supply-chain-verification|Supply-chain verification]]). The Node.js layer, by contrast, comes from a `curl | bash` of the NodeSource setup script at build time, which the signing covers only after the fact. ^[inferred] Measured resource use: about 190 MB RAM idle, 280 MB with 3 tabs, about 30 MB per extra tab (README).

## Pinned by tests (`tests/test_cloakserve.py`)

Commit b1a00ed; the suite needs the `serve` extra (`aiohttp`) and is skipped without it. The P2 read of `bin/cloakserve` matches every rule below.

### Connection parameters

- `fingerprint` becomes the seed. `timezone`, `locale`, `proxy` and `geoip` are consumed as dedicated fields. Every other parameter becomes `--fingerprint-<name>=<value>` (e.g. `platform`, `hardware-concurrency`, `gpu-vendor`).
- A repeated parameter keeps its first value.
- `geoip` is true only for `true`, `1` or `yes` (any case). Anything else, including typos, is false.

### Server command line

- Defaults: port 9222, headless, idle timeout 0 (off), and a data dir that is always set. The data dir is `/tmp/cloakserve` when a container marker file exists, otherwise `~/.cloakbrowser/cloakserve`.
- `--port`, `--data-dir` and `--idle-timeout` are consumed and never passed to Chrome. A `--fingerprint=<seed>` becomes the default seed. `--remote-debugging-port` and `--remote-debugging-address` are stripped silently, because cloakserve assigns each seed's debug port and address itself. `--headless=false` is both read and passed through to Chrome.
- Idle timeout: the CLI flag beats `CLOAKSERVE_IDLE_TIMEOUT`, fractional seconds are accepted, `0`/`off`/`false`/`none`/`disabled` turn it off, and negative values raise.

### Public URL rewriting

- The public host is the first `X-Forwarded-Host` value (a blank header is ignored), else `Host`, else `localhost:<port>`.
- The first `X-Forwarded-Proto` value of `https` selects `wss`.
- Browser and page WebSocket URLs become `/fingerprint/<seed>/devtools/<kind>/<id>` for a seeded process and `/devtools/<kind>/<id>` for the default one.

### WebSocket origin guard

- A request with no `Origin` header is allowed, because non-browser CDP clients (Playwright, Puppeteer) send none (stated in a code comment).
- `devtools://` and `chrome-devtools://` origins are allowed.
- Any other origin must be a bare `scheme://host[:port]` that equals the request's `Host`, and only for a loopback host. A public origin that matches its own `Host` is still rejected, which defends against DNS rebinding. ^[inferred]
- Rejected origins include those with a path, query, fragment, userinfo, trailing slash or empty port, plus `null` and `file://`.
- A malformed `Host` header (userinfo, path, comma list, empty port) is rejected even when it names loopback. The default port follows the request scheme.
- Rejection is an HTTP 403 ("untrusted") on both the default and the per-seed WebSocket route, sent before any Chrome process is launched.

### Connection refcounting and idle reaping

- `ChromePool` counts connections per seed. A disconnect never goes below zero, and seeds are counted independently.
- When a seed's count reaches zero, cleanup is scheduled only if an idle timeout is set. A reconnect cancels the pending cleanup.
- A `/json` discovery call against a live process reschedules the timer. A freshly launched process with no WebSocket client also gets one (the 0.6.0 fix, #558).

### Seed validation and deletion containment (path-traversal fix)

- `SAFE_SEED_RE` accepts 1 to 128 letters, digits, `-` and `_`. It rejects `.`, `..`, slashes, backslashes, NUL, an empty seed and 129 or more characters. Invalid seeds get HTTP 400.
- `__default__` is reserved for the shared no-seed process.
- `_safe_rmtree` resolves the path and deletes only strict subdirectories of `data_dir`. It refuses `data_dir` itself, sibling directories and `..` traversal.

### Test gap

The file defines `TestHandlerURLRewriting` twice. The second class, which only re-implements the rewrite as local string helpers, replaces the first in the module namespace. As a result, the two handler-level tests of `/json/version` and `/json/list` with forwarded headers are never collected. ^[inferred] The forwarded-header behaviour is therefore pinned only at the `_external_host`/`_ws_scheme` helper level, and the rewrite format only against a copy of the logic. The script itself implements the rewrite as the tests' copy describes.

## Trade-offs

- Humanized input is a wrapper feature and does not travel over raw CDP; clients must import the wrapper's humanize patching themselves (README). ^[inferred]
- "First connection's params win" means two clients using the same seed with different timezones silently share the first one's identity; the server only logs a warning. ^[inferred]
- Launching Chrome directly keeps cloakserve independent of Playwright, but it has to replicate the flags Playwright normally adds and loses Playwright's proxy-auth fallback. ^[inferred]

## Open Questions

- How cloakserve handles license seats per seed is implied by the close endpoint but not stated. Each seed is a separate binary process, so presumably one seat each. ^[ambiguous]
- Where the "persistent pool" named in the close-endpoint docstrings lives, and whether profile preservation on close ever shipped in this repository. ^[ambiguous]

## Related

- [[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying|CDP endpoint proxying (Manager)]] — CloakBrowser-Manager runs its own per-profile CDP endpoint instead of cloakserve.

## Sources

- `~/code/CloakBrowser/bin/cloakserve` (commit `b1a00ed`, P2 pass)
- `~/code/CloakBrowser/bin/cloaktest` (commit `b1a00ed`, P2 pass)
- `~/code/CloakBrowser/Dockerfile` (commit `b1a00ed`, P2 pass)
- `~/code/CloakBrowser/bin/docker-entrypoint.sh` (commit `b1a00ed`)
- `~/code/CloakBrowser/README.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/.github/workflows/publish.yml` (commit `b1a00ed`)
- `~/code/CloakBrowser/tests/test_cloakserve.py` (commit `b1a00ed`, B7 batch)
