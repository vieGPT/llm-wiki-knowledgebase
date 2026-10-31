---
title: >-
    Frontend UI
category: subsystems
tags: [react, frontend, profile-manager, novnc, ui]
aliases: [React app, Manager web UI, useProfiles]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/frontend-api-client]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/api-server]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/vnc-viewer]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/architecture/authentication-model]]"
    type: implements
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/App.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/hooks/useProfiles.ts
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/hooks/useProfiles.test.ts
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileForm.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileForm.test.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileList.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileViewer.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileFiles.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/ProfileFiles.test.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/NativeWindowStatus.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/NativeWindowStatus.test.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/CdpEndpointButton.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/LaunchButton.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/LaunchErrorBanner.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/LoginPage.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/SettingsPanel.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/StatusIndicator.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/SystemStatusBadge.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/components/UpdateBanner.tsx
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/src/lib/api.test.ts
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/package.json
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/index.html
  - repo:github.com/vieGPT/CloakBrowser-Manager/frontend/postcss.config.js
summary: >-
    The Manager's React single-page app: auth gate, profile list and form, a 3 s status poll, license and update banners, and two running views (noVNC canvas on Linux, native-window card elsewhere).
provenance:
  extracted: 0.84
  inferred: 0.13
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:20:42+01:00
updated: 2026-10-31T06:37:29+01:00
---

# Frontend UI

The frontend is a React + Tailwind single-page app (icons from `lucide-react`, drag-and-drop from `@dnd-kit`, noVNC loaded on demand) served by the [[projects/cloakbrowser-manager/subsystems/api-server|API server]]. It talks to the backend only through the typed client in [[projects/cloakbrowser-manager/subsystems/frontend-api-client|frontend API client]] (`lib/api.ts`). It holds no fingerprint logic of its own: it edits the per-profile configuration that the backend later turns into CloakBrowser launch options, and it shows the state of running profiles (as of 31/10/2026, commit `11e0043`). ^[inferred]

## App shell and auth gate (`App.tsx`, `LoginPage`)

- `App` is a four-state auth machine: `checking` → `ok` / `required` / `error`. It asks `/api/auth/status` for `auth_required` and `authenticated`; with no token configured, the app opens directly.
- The API client exposes `setOnUnauthorized`; `App` registers a callback so any 401 anywhere drops the user back to `LoginPage`. Login posts the access token; the logout button appears only when auth is required. See [[projects/cloakbrowser-manager/architecture/authentication-model|Authentication model]].
- `error` (server unreachable) shows a Retry button that re-runs the status check.
- `AppContent` keeps one of four views: `empty`, `create`, `edit`, `view`. Selecting a profile opens `view` if it is running, otherwise `edit`. Creating or duplicating a profile selects the new profile in `edit`. Stopping returns to `edit`; Reset stays in `edit` (the profile is wiped but not launched).
- **Quit** (power icon) confirms, then calls `POST /api/shutdown`, which stops the server and every running profile. The network error from the server exiting mid-request is expected and swallowed; the page then shows a "has stopped" screen.

## State: the `useProfiles` hook

- Loads `/api/profiles` on mount and **polls every 3 s**; status changes (running, stopped, `initializing`, `last_error`) reach the UI only through this poll. ^[inferred]
- `create` and `duplicate` prepend the returned profile; `update` replaces it in place; `remove` filters it out. Failures set a hook-level error shown as a red bar under the top bar.
- `reorder` is **optimistic**: the list is reordered locally, then persisted with `/api/profiles/reorder`; on failure it sets the error and refreshes from the server (pinned by `useProfiles.test.ts`).
- `launch` deliberately **re-throws** instead of setting the hook error, so a structured license denial reaches the caller. `reset` wipes browser state and re-rolls the fingerprint while keeping configuration; `duplicate(id, includeBrowserState)` clones settings and fingerprint, optionally with cookies, sessions and history.

## Sidebar (`ProfileList`)

- Search filters by name; a "N running" counter shows when profiles run.
- Rows show a `StatusIndicator` (green pulse running, amber pulse `initializing` = "First-time setup...", grey stopped), a "Proxy" hint when a proxy is set, and coloured tags.
- Drag-to-reorder uses a 5 px activation distance so a click still selects. Reordering is disabled while a search filter is active, because a drag inside a filtered subset is ambiguous.

## Profile form (`ProfileForm`)

One form serves create and edit. Sections and defaults for a new profile:

| Section | Fields (default) |
|---|---|
| Basic | name (required), fingerprint seed (blank = server picks a random one; dice button rolls 10000-99999) |
| Network | proxy with a **Test** button (exit IP, city/country, latency), timezone, locale, GeoIP auto-detect (on) |
| Hardware | screen resolution (six presets or custom; 1920×1080), [[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting\|GPU family]] (`auto`, NVIDIA, Intel; forced to Apple Silicon automatic on macOS hosts, disabled while the host OS is unknown) |
| Behavior | [[projects/cloakbrowser/subsystems/humanize-engine\|humanize]] (off) with preset `default`/`careful`, clipboard sync (on, VNC mode only), auto-launch (off), preview screenshots (on), restore previous tabs (on) |
| Compatibility | third-party cookies (on), [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy\|transparent proxy]] (off), [[projects/cloakbrowser-manager/techniques/portable-cookies\|portable cookies]] (off), Google as default search (on) |
| Other | tags with colours, unpacked extension directories, advanced Chromium launch args, notes |

- What these fields mean for identity and network is covered in [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|Per-profile identity isolation]] and [[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale|Per-profile network and locale]].
- Edit mode adds a "Last preview" image (`/api/profiles/<id>/screenshot`, cache-busted per profile), and Duplicate, Reset and Delete actions, each behind a `confirm()` that states what is lost or copied.
- **Duplicate split button** (pinned by `ProfileForm.test.tsx`): the main half is a config-only copy; the menu's "With browser state" item is disabled with "Stop the profile first" unless the profile is stopped. Escape and outside clicks close the menu, and Escape returns focus to the trigger. See [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state|Profile duplication with state]].
- The advanced-arguments box warns that these unrestricted Chromium flags "can override Manager-controlled behavior".

## Running views

`view` picks a component by the profile's `viewer_mode` (see [[projects/cloakbrowser-manager/architecture/platform-modes|Platform modes]]):

- **VNC (`ProfileViewer`, Linux/Docker)**: dynamically imports noVNC's RFB client and connects to `ws(s)://<host>/api/profiles/<id>/vnc` with viewport scaling and no session resize. Disconnect returns to `edit`. The stage (canvas plus overlays) is what goes fullscreen, so upload feedback stays visible. Details of the server side are on [[projects/cloakbrowser-manager/subsystems/vnc-viewer|VNC viewer]].
- **Clipboard sync**, three paths, all active only while connected and enabled:
  1. Host → browser: a capture-phase keydown listener intercepts Ctrl/Cmd+V before noVNC sees it, reads the host clipboard, posts it to `/api/profiles/<id>/clipboard`, then replays a full Ctrl+V key sequence (the user may have released Ctrl during the async call).
  2. Browser → host: noVNC `clipboard` events, produced when the backend proxy converts KasmVNC's binary clipboard message into a standard cut-text message.
  3. Browser → host fallback: polling `GET /clipboard` every 2 s, because Chrome under KasmVNC does not write the X11 clipboard. A code comment says the backend reads it via Playwright CDP.
  A host clipboard read failure turns sync off; the toggle persists through `update(id, {clipboard_sync})` and rolls back on failure.
- **File drop**: files dropped on the canvas are caught by the Manager page (the VNC protocol carries no files) and uploaded over HTTP to the profile's file store; then the user picks them from "Uploads — <profile>" in the page's file dialog.
- **Native (`NativeWindowStatus`, Windows/macOS)**: a card saying the profile runs in a native window, with a near-live preview: the backend captures a screenshot about every 30 s and the card re-fetches every 15 s (unless `capture_preview` is off).
- Both views carry `CdpEndpointButton`, which copies an **absolute** CDP URL built from the page's own protocol and host plus the profile's relative `cdp_url` (`/api/profiles/<id>/cdp`), so automation goes through the Manager (pinned by `NativeWindowStatus.test.tsx`). See [[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying|CDP endpoint proxying]].

## Profile files panel (`ProfileFiles`)

`useProfileFiles(profileId)` is shared by the toolbar panel and the drop target ([[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts|Profile files and artifacts]]):

- Uploads run **sequentially**, in drop order, because each upload is bounded server-side. The client sends the raw file as the request body with its URL-encoded name in `X-File-Name` and falls back to `application/octet-stream` (pinned by `api.test.ts`); a refusal (for example 413) surfaces the server's `detail`.
- A **mutation generation counter** discards list responses that started before an upload or delete, and listing is single-flight, so a slow poll cannot erase a row that was just added.
- Transfers survive the panel being closed. Finished ones disappear after 6 s, failed ones stay with Retry.
- While open, the panel polls every 2 s so browser downloads appear. Files are marked as `upload` or `download` and as `pending`, `ready` or `failed`; only `ready` files get a download link. The tests also pin focus return on Escape and Close and a screen-reader label that states the file count.

## Banners, status and settings

- `SystemStatusBadge`: license tier (`pro`, `free`, `keyless`), binary version, and a "Fonts incomplete" warning with present/required counts for the Windows persona fonts. See [[projects/cloakbrowser-manager/subsystems/settings-and-license|Settings and license]] and [[projects/cloakbrowser-manager/subsystems/diagnostics|Diagnostics]].
- With no key (`keyless`), the empty view offers to enter a key, get a free key, or continue keyless.
- `SettingsPanel` edits the license key (masked, password field) and the release channel (`stable`/`preview`). An empty key field leaves the stored key untouched; "Remove key" sends an empty string. Saving returns a fresh system status, so the badge updates without a reload.
- `LaunchErrorBanner`: a launch refused with HTTP 402/403 (out of seats, invalid or expired key) shows the message and an "Upgrade your plan" link. A denial that arrives after the handshake, through `profile.last_error` in the poll, shows only while that profile stays selected and is cleared on switch.
- `UpdateBanner`: shown when `/api/update-check` reports a newer Manager release on GitHub. Both banners open links through `POST /api/open-external`, because `window.open` does nothing inside the native app's webview; dismissal is in memory only.

## Build toolchain

As of commit `11e0043`, `frontend/package.json` (package `cloakbrowser-manager`, private, version `0.1.0`, which does not track the Manager release number) declares:

- **Runtime dependencies:** React and React DOM `^19.0.0`, `lucide-react ^0.468.0`, `@dnd-kit/core`, `@dnd-kit/sortable` and `@dnd-kit/utilities`, and `@novnc/novnc ^1.4.0`. The noVNC 1.4 line is the one whose RFB extension messages the backend filters out for KasmVNC ([[projects/cloakbrowser-manager/subsystems/vnc-viewer]]).
- **Build:** Vite `^6` with `@vitejs/plugin-react` and TypeScript `~5.7`. `npm run build` runs `tsc -b` before `vite build`, so a type error fails the build.
- **Styling:** Tailwind CSS `^3.4` through PostCSS with Autoprefixer (`postcss.config.js` holds only those two plugins).
- **Tests:** Vitest `^4.1` (`npm test` runs `vitest run`) with jsdom and Testing Library (`@testing-library/react`, `@testing-library/jest-dom`).
- **Entry:** `frontend/index.html` is the Vite entry. It is titled "CloakBrowser Manager", sets `class="dark"` on `<html>` (dark theme by default), styles the body with the custom Tailwind token `bg-surface-0`, and mounts the app from `/src/main.tsx` into `#root`.
- `package-lock.json` (lockfile v3) pins the resolved tree. It is recorded as a source, with no distilled content.

## Open Questions

- `useProfiles.launch` says the denial is rendered by `LaunchButton`, but `App.handleLaunch` catches it first and `LaunchButton` only logs. The comment is stale; the behaviour (top banner) is what `App.tsx` does. ^[ambiguous]
- `ProfileViewer` logs the first 50 characters of clipboard text to the browser console on every sync path. Whether this debug logging is intended in release builds is not stated. ^[inferred]
- The `Profile` type carries `color_scheme` (passed to the launcher by the backend), but the form has no field for it.

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager (commit `11e0043`): `frontend/src/App.tsx`, `frontend/src/hooks/useProfiles.ts`, `frontend/src/hooks/useProfiles.test.ts`, `frontend/src/components/ProfileForm.tsx`, `frontend/src/components/ProfileForm.test.tsx`, `frontend/src/components/ProfileList.tsx`, `frontend/src/components/ProfileViewer.tsx`, `frontend/src/components/ProfileFiles.tsx`, `frontend/src/components/ProfileFiles.test.tsx`, `frontend/src/components/NativeWindowStatus.tsx`, `frontend/src/components/NativeWindowStatus.test.tsx`, `frontend/src/components/CdpEndpointButton.tsx`, `frontend/src/components/LaunchButton.tsx`, `frontend/src/components/LaunchErrorBanner.tsx`, `frontend/src/components/LoginPage.tsx`, `frontend/src/components/SettingsPanel.tsx`, `frontend/src/components/StatusIndicator.tsx`, `frontend/src/components/SystemStatusBadge.tsx`, `frontend/src/components/UpdateBanner.tsx`, `frontend/src/lib/api.test.ts`, `frontend/package.json`, `frontend/index.html`, `frontend/postcss.config.js`, `frontend/package-lock.json` (lockfile only)
