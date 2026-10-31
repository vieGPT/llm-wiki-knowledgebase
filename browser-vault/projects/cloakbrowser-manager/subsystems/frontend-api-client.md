---
title: >-
    Frontend API client
category: subsystems
tags: [react, typescript, api-client, error-handling, fastapi]
aliases: [api.ts, ApiError, frontend/src/lib/api.ts]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/api-server]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/api-models]]"
    type: derived_from
  - target: "[[projects/cloakbrowser-manager/subsystems/frontend-ui]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/frontend/src/lib/api.ts
  - ~/code/CloakBrowser-Manager/frontend/src/main.tsx
  - ~/code/CloakBrowser-Manager/frontend/src/novnc.d.ts
  - ~/code/CloakBrowser-Manager/frontend/src/styles/globals.css
  - ~/code/CloakBrowser-Manager/frontend/tailwind.config.ts
  - ~/code/CloakBrowser-Manager/frontend/tsconfig.json
  - ~/code/CloakBrowser-Manager/frontend/vite.config.ts
  - ~/code/CloakBrowser-Manager/frontend/vitest.config.ts
  - ~/code/CloakBrowser-Manager/frontend/src/lib/api.test.ts
  - ~/code/CloakBrowser-Manager/frontend/src/App.tsx
summary: >-
    frontend/src/lib/api.ts: one typed fetch wrapper over the same-origin /api, cookie auth, a global 401 hook, and ApiError carrying structured license denials (reason, upgrade URL).
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

# Frontend API client

`frontend/src/lib/api.ts` is the only place the React UI talks to the backend. It exports TypeScript interfaces that mirror the backend's response models ([[projects/cloakbrowser-manager/subsystems/api-models|API models]]), an `ApiError` class, a global unauthorized hook, and an `api` object with one method per [[projects/cloakbrowser-manager/subsystems/api-server|API server]] endpoint. Every component and the `useProfiles` hook in [[projects/cloakbrowser-manager/subsystems/frontend-ui|the frontend UI]] import from it (as of 14/10/2026, commit `11e0043`).

## Request model

- **One wrapper.** A private generic `request<T>(path, options)` calls `fetch` on a relative `/api/...` path with a JSON `Content-Type`, and returns the parsed JSON on success.
- **Same origin, cookie auth.** Paths are relative and no `Authorization` header is added. Authentication rides on the `auth_token` cookie that `/api/auth/login` sets (HttpOnly, `SameSite=Strict`, `Secure` over HTTPS), which `fetch` sends by default on same-origin requests (see [[projects/cloakbrowser-manager/architecture/authentication-model|authentication model]]). ^[inferred] In development, Vite proxies `/api` to `localhost:8080`, so the dev server stays same-origin as well.
- **Global 401 hook.** `setOnUnauthorized(cb)` registers one module-level callback. `App` sets it to switch to the login page and clears it on unmount. Any 401 calls the callback and throws `ApiError(401, "Unauthorized")`.
- **No retry, timeout or cancellation.** No `AbortController`; polling and refresh cadence live in the UI hooks. ^[inferred]
- `options` is spread after the default headers, so a caller that passes its own `headers` replaces the JSON `Content-Type` instead of merging with it. No current call does this. ^[inferred]

## `ApiError` and license denials

`ApiError` extends `Error` with `status`, and optionally `reason` and `upgradeUrl`. FastAPI's `detail` is normally a string, which becomes the message. A launch blocked by licensing returns an object instead: `{message, reason, upgrade_url}`, with HTTP 402 for a seat-limit denial and 403 for other license problems (bad, expired or unreachable key). The client maps that object onto `reason` and `upgradeUrl`. `App` turns a 402/403 `ApiError` from launch into a `LaunchDenial` (`reason: "seat_limit" | "license"`) for the dismissible upgrade banner. The same shape is persisted on the profile as `last_error` and cleared on the next launch. This is the UI end of CloakBrowser's seat counting and exit-code mapping ([[projects/cloakbrowser/subsystems/license-and-tiers|license and tiers]]).

When the body is not JSON, the error falls back to the HTTP status text (covered by `api.test.ts`).

## Endpoint surface

| Area | Methods (HTTP) |
|---|---|
| Auth | `authStatus` (GET), `login` (POST token), `logout` |
| Profiles | `listProfiles`, `getProfile`, `createProfile`, `updateProfile` (PUT, partial), `deleteProfile`, `reorderProfiles` (`ordered_ids`), `resetProfile`, `duplicateProfile` |
| Proxy | `testProxy` returns exit IP, country, city, timezone, latency |
| Lifecycle | `launchProfile` returns `LaunchResult` (runtime mode, viewer mode, VNC WebSocket port, display, `cdp_url`, see [[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying\|CDP endpoint proxying]]); `stopProfile` |
| Files | `listProfileFiles`, `uploadProfileFile`, `deleteProfileFile`, `profileFileUrl` |
| Clipboard | `setClipboard`, `getClipboard` (VNC mode only) |
| System | `getStatus`, `checkUpdate`, `getSettings`, `updateSettings`, `openExternal`, `shutdown` |

Details worth knowing:

- **Duplicate** defaults to a config-only clone; `includeBrowserState=true` sends `include_browser_state` (see [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state|profile duplication with state]]).
- **Upload** bypasses `request`. The file itself is the streamed request body, its name goes URL-encoded in `X-File-Name` (header values cannot carry arbitrary Unicode), and `Content-Type` falls back to `application/octet-stream`. Its error path keeps only string details. `profileFileUrl` just builds the GET URL for a direct download, which works because the auth cookie travels with plain navigations too. ^[inferred] Files and limits: [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts|profile files and artifacts]].
- **Settings.** In `SettingsUpdate`, an omitted `license_key` means unchanged and `""` clears it. `updateSettings` returns a fresh `SystemStatus`, which fits the hot-apply-without-restart design. ^[inferred] `ManagerSettings` exposes only `license_key_set` and a masked key, never the key itself.
- **`openExternal`** exists because `window.open` does nothing inside the native pywebview window; the backend opens http(s) URLs in the host browser.

## Types that encode the domain

- `Profile` is the per-profile identity record: fingerprint seed, proxy, timezone, locale, screen size, `gpu_family` (`auto|nvidia|intel`), humanize and preset, GeoIP, launch args, extensions, compatibility flags (`allow_3p_cookies`, [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|`transparent_proxy`]], [[projects/cloakbrowser-manager/techniques/portable-cookies|`portable_cookies`]], `restore_session`, `set_google_default`), tags, plus server-side runtime state (`status: running|stopped|initializing`, `runtime_mode`, `viewer_mode`, `vnc_ws_port`, `cdp_url`). See [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|per-profile identity isolation]].
- Platform split: `HostOS` (`windows|macos|linux`), `RuntimeMode` (`native|docker`) and `ViewerMode` (`native-window|vnc`) let the UI choose between a native-window status card and the embedded VNC viewer ([[projects/cloakbrowser-manager/architecture/platform-modes|platform modes]]).
- `SystemStatus` also reports `license_tier` (`pro|free|keyless`), the binary version, and the Windows-font health counters from [[projects/cloakbrowser-manager/subsystems/diagnostics|diagnostics]].
- `ProfileFile.container_path` is where the browser process sees the file, for CDP calls that take a path.

## Frontend scaffolding (same batch)

- `main.tsx` mounts `App` under `React.StrictMode` and loads `globals.css`.
- `novnc.d.ts` is a hand-written type shim for `@novnc/novnc`'s `RFB` class (scaling, quality and compression, `sendKey`, `clipboardPasteFrom`, power actions), which the [[projects/cloakbrowser-manager/subsystems/vnc-viewer|VNC viewer]] uses.
- Styling: Tailwind with `darkMode: "class"`, a near-black `surface` 0 to 4 scale, a `border` colour and an indigo `accent`; `globals.css` defines `.btn*`, `.input` and `.label` component classes, thin scrollbars and a `no-spin` number input.
- Tooling: TypeScript `strict` with `noUncheckedIndexedAccess` and no-unused checks, bundler module resolution, `noEmit` (Vite emits); test files are excluded from `tsc -b`. Vitest runs in `jsdom` with globals.

## Open Questions

- The client sends no CSRF token; state-changing calls rely on `SameSite=Strict` plus the backend's Origin/Host check on sensitive endpoints. Is that check applied to all mutating routes, or only to shutdown and WebSockets? ^[ambiguous]

## Class stubs (ast-extract)

- [[projects/cloakbrowser-manager/subsystems/api-error|ApiError]]

## Sources

Commit `11e0043` of `~/code/CloakBrowser-Manager`:

- frontend/src/lib/api.ts, frontend/src/main.tsx, frontend/src/novnc.d.ts, frontend/src/styles/globals.css
- frontend/tailwind.config.ts, frontend/tsconfig.json, frontend/vite.config.ts, frontend/vitest.config.ts
- frontend/src/lib/api.test.ts, frontend/src/App.tsx (consulted for consumers and test coverage)
