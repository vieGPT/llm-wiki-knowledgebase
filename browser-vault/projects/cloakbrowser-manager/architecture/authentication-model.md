---
title: >-
    Authentication model
category: architecture
tags: [authentication, security, csrf, websocket, profile-manager]
aliases: [AUTH_TOKEN, auth_token cookie, /api/health vs /api/status]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/api-server]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/app_entry.py
  - ~/code/CloakBrowser-Manager/README.md
  - ~/code/CloakBrowser-Manager/CHANGELOG.md
  - ~/code/CloakBrowser-Manager/backend/tests/test_auth.py
  - ~/code/CloakBrowser-Manager/backend/tests/test_api.py
  - ~/code/CloakBrowser-Manager/docker-compose.yml
summary: >-
    Optional single shared AUTH_TOKEN gates /api/* via Bearer header or an HttpOnly cookie (which also covers VNC WebSockets); /api/health stays open, origin checks guard WS and shutdown.
provenance:
  extracted: 0.84
  inferred: 0.14
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:27:05+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Authentication model

The Manager has no user accounts. Authentication is one optional shared secret, `AUTH_TOKEN`: unset (the default, meant for local use) means every route is open; set means every `/api/*` path needs the token. The static UI itself is never gated, so the login page can load. Because a running profile's CDP endpoint is an `/api/` path, the same token protects full remote control of every browser ([[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]). ^[inferred]

## How a request is authenticated

- A raw ASGI middleware ([[projects/cloakbrowser-manager/subsystems/api-server]]) checks HTTP and WebSocket scopes. It accepts `Authorization: Bearer <token>` or an `auth_token` cookie, both compared in constant time (`hmac.compare_digest`).
- Rejections: HTTP 401 JSON; for WebSockets it first receives the connect message (ASGI requires that), then closes with code 4401.
- **Login** (`POST /api/auth/login`) checks the token and sets the `auth_token` cookie to the token itself: HttpOnly, SameSite=Strict, path `/`, and `Secure` only when `X-Forwarded-Proto` says https. Logout deletes it. The cookie is how browser WebSockets (VNC viewer) authenticate, since a browser cannot add an `Authorization` header to a WebSocket upgrade. ^[inferred]
- **Exempt paths**: `/api/auth/status` (tells the UI whether auth is required and whether this request is authenticated), `/api/auth/login`, and `/api/health`.

## Health vs status (v0.1.0)

`/api/health` returns only `{"status": "ok"}` and stays unauthenticated for the Docker health check and the native launcher's readiness poll. `/api/status` (running count, binary version, tier, profile total) used to be open and leaked those to scanners; it now requires auth. The native launcher previously polled `/api/status` and hung when a token was set; it now polls `/api/health` (CHANGELOG 0.1.0, contributed per README by shellus).

## Cross-origin defences (independent of the token)

- **WebSocket origin check** on VNC and CDP upgrades: if a browser sends `Origin`, its host:port must equal `Host` (default ports normalised), otherwise close 4403. Non-browser clients send no `Origin` and pass. This blocks cross-site WebSocket hijacking of a localhost Manager.
- **CSRF guard on `POST /api/shutdown`**: a bodyless POST is a CORS simple request, so any website could fire it at `127.0.0.1:8080`; the same Origin-vs-Host rule rejects it with 403.
- SameSite=Strict keeps the auth cookie off cross-site requests. ^[inferred]

## Deployment guidance (README)

- The Docker image binds to host localhost by default; remote use goes through an SSH tunnel.
- The token travels in cleartext over HTTP; anything exposed to the internet should sit behind an HTTPS reverse proxy (Caddy, nginx, Traefik). `X-Forwarded-Proto` handling makes the cookie `Secure` and CDP discovery URLs `wss` there.

## Pinned by tests

`backend/tests/test_auth.py` and the origin cases in `test_api.py` (as of commit `11e0043`):

**Auth off (`AUTH_TOKEN` unset)**

- `/api/profiles` answers 200.
- `/api/auth/status` reports `auth_required: false, authenticated: false`.
- `POST /api/auth/login` with any token is a no-op that returns `{"ok": true}`.
- `docker-compose.yml` passes `AUTH_TOKEN=${AUTH_TOKEN:-}`, so an unset variable arrives as an empty string. `main.py` reads it with `os.environ.get(...) or None`, so empty also means auth off ([[projects/cloakbrowser-manager/subsystems/vnc-viewer|container lifecycle]]).

**Auth on (`AUTH_TOKEN` set)**

- **Rejected.** A missing or wrong Bearer token gets 401. A wrong login token gets 401 and sets no cookie.
- **Accepted.** The correct Bearer token, or the `auth_token` cookie alone, gets 200. A correct login returns `{"ok": true}` and sets `auth_token`. Logout returns `{"ok": true}`.
- **`/api/auth/status`.** It is reachable without credentials and reports `authenticated: true` only when the cookie is present.
- **Health vs status.** `/api/health` stays 200 without credentials. `/api/status` returns 401 without the token and 200 with it, which pins the v0.1.0 split.

**Origin rules**

- VNC and CDP WebSockets from a foreign `Origin` are refused. Same-origin connections and connections with no `Origin` pass the check.
- `POST /api/shutdown` returns 403 for a foreign origin and for the same host on another port. Default ports are normalised: `https://h:443` equals `h`, and `http://h` equals `h:80`.

## Gaps

- With no token, a local Manager trusts every non-browser process on the host and, for routes other than shutdown and WebSockets, relies on browsers' CORS rules: a cross-site `fetch` can still send bodyless simple requests such as `POST /api/profiles/{id}/launch` or `/stop`, only the response is unreadable. Only shutdown carries the Origin check. ^[inferred] Routes that need a JSON body are harder to hit, since a cross-site JSON content type triggers a preflight. ^[inferred]
- One shared token, no rotation or per-client scoping; the cookie value equals the token. ^[inferred]

## Sources

- ~/code/CloakBrowser-Manager/backend/main.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/app_entry.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/README.md (Authentication, Remote Access), CHANGELOG.md (0.1.0) (commit 11e0043)
- ~/code/CloakBrowser-Manager/backend/tests/test_auth.py, backend/tests/test_api.py, docker-compose.yml (commit 11e0043)
