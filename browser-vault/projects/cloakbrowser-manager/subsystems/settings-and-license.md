---
title: >-
    Settings and license
category: subsystems
tags: [configuration, licensing, release-channel, profile-manager, fastapi]
aliases: [settings store, in-app settings, license key handling, settings.json]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/license-and-tiers]]"
    type: uses
  - target: "[[projects/cloakbrowser/architecture/release-channels-and-versioning]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/subsystems/api-server]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/browser-manager]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/backend/settings_store.py
  - ~/code/CloakBrowser-Manager/backend/env_file.py
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/models.py
  - ~/code/CloakBrowser-Manager/.env.example
  - ~/code/CloakBrowser-Manager/README.md
  - ~/code/CloakBrowser-Manager/CHANGELOG.md
  - ~/code/CloakBrowser-Manager/backend/tests/test_api.py
summary: >-
    One app-wide CloakBrowser license key and release channel, read from env (.env) first, then settings.json; PUT /api/settings saves and hot-applies them, but env wins again at restart.
provenance:
  extracted: 0.74
  inferred: 0.22
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:32:16+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Settings and license

The Manager has exactly two app-wide settings (as of 14/10/2026, commit `11e0043`): the CloakBrowser **license key** and the **release channel** (`stable` or `preview`). Both are passed to every profile launch, so all profiles share one key and therefore one concurrency-seat pool. Two small modules hold them: `backend/env_file.py` loads a `.env` into the process environment, and `backend/settings_store.py` persists what the in-app Settings panel saves. `backend/main.py` resolves the two values at import, exposes `GET`/`PUT /api/settings`, and hands them to the [[projects/cloakbrowser-manager/subsystems/browser-manager|browser manager]], which forwards them to the wrapper's [[projects/cloakbrowser/subsystems/license-and-tiers|license and tiers]] layer. The Manager validates nothing about the key itself; tier, seats and denials are the wrapper's and the Pro binary's job. ^[inferred]

## Two stores

- **`.env` loader** (`env_file.py`): reads `KEY=VALUE` lines from a `.env` in the Manager root (from source) or next to the executable (frozen native app). It skips blank lines and comments, strips quotes, and **never overrides** a variable already in the environment, so Docker's `-e` values or a shell export win over the file. It runs once, first thing in `main.py`, before anything reads the environment (`AUTH_TOKEN`, runtime, license).
- **`settings.json`** (`settings_store.py`): a flat JSON dict in the same data directory as `profiles.db` (see [[projects/cloakbrowser-manager/subsystems/runtime-and-binary|runtime and binary]]). It replaces hand-editing `.env` for the native app, "where there is no terminal and no reachable file next to a `.app` bundle". New settings are new keys, with no schema migration. Writes are atomic (temp file in the same directory, then `os.replace`). A missing, unreadable, corrupt or non-dict file loads as `{}`.

## Precedence

At import, `main.py` resolves each value as **environment first, then `settings.json`, else unset**: `CLOAKBROWSER_LICENSE_KEY` vs `license_key`, `CLOAKBROWSER_RELEASE_CHANNEL` vs `release_channel`. The environment includes whatever the `.env` loader pulled in. The README states the rule for users: "An environment variable overrides the in-app setting."

- An empty variable (the `.env.example` default `CLOAKBROWSER_LICENSE_KEY=`) counts as unset, because the resolver uses a truthiness test, so the stored setting applies. ^[inferred]
- **Hot-apply lasts until restart.** `PUT /api/settings` writes `settings.json` and sets the values on the running browser manager directly. On the next start, env is read first again, so if `CLOAKBROWSER_LICENSE_KEY` or `CLOAKBROWSER_RELEASE_CHANNEL` is set (Docker `-e`, `.env`), the in-app change silently reverts. The README's Docker text ("stored in the mounted `/data` volume, so it persists across restarts") holds only when no env value is set. ^[inferred]
- **Clearing the key may not reach keyless.** An empty `license_key` in the PUT removes the key from the file and sets the manager's key to `None`. The wrapper, given no explicit key, still falls back to `CLOAKBROWSER_LICENSE_KEY` and then to `~/.cloakbrowser/license.key` ([[projects/cloakbrowser/subsystems/license-and-tiers|resolution order]]). With either present, the Manager keeps launching keyed even after "clear". ^[inferred]
- The Settings response does not say where a value came from, so the UI cannot show that an env value overrides it. ^[inferred]

## API

| Route | Behaviour (commit `11e0043`) |
|---|---|
| `GET /api/settings` | `SettingsResponse`: `license_key_set`, `license_key_masked` (first 5 and last 4 characters, e.g. `cb_ba…63aa`; keys of 9 characters or fewer show only `…`), `release_channel` (reported as `stable` when unset). The full key is never returned. |
| `PUT /api/settings` | `SettingsUpdate`: each field `None` = leave unchanged. A non-empty key is trimmed and stored; `""` clears it (back to keyless). The channel is trimmed and lower-cased and must be `stable` or `preview`, else **400** and nothing is persisted or applied. Then it saves, re-runs `resolve_binary_status()` in a worker thread and returns the refreshed `/api/status` payload, so the top-bar badge updates at once. |

`resolve_binary_status()` may download the multi-hundred-MB Pro build, so the PUT returns only when the binary is ready. It never raises: on any failure the baked-in keyless binary stays usable ([[projects/cloakbrowser/subsystems/binary-download-and-verification|binary download]]). Running profiles keep the binary they started with; the new key and channel apply from the next launch. ^[inferred]

## License handling around launches

- **Tier**: `resolve_binary_status()` resolves the key through the wrapper, validates it, and sets `license_tier` to `pro`, `free` (plan `free`) or `keyless`, plus the binary version (the latest Pro build for the channel, else the keyless version). Startup runs it once, off the launch path. The badge is described on [[projects/cloakbrowser-manager/subsystems/diagnostics|diagnostics]].
- **Denials**: a launch that raises `CloakBrowserLicenseError` maps to HTTP **402** when the message contains "session limit reached" (seat exit code 76, with an upgrade URL to cloakbrowser.dev pricing) and **403** for other license problems (invalid, expired or revoked key, server unreachable). Post-handshake denials are read from the wrapper's per-launch denial file and surfaced on the next status poll ([[projects/cloakbrowser-manager/subsystems/browser-manager|browser manager]]). The banner UI dates from v0.1.4 (21/08/2026).
- **Logging**: launch logs carry tier and plan, never the key.

## Release channel

`stable` is the default; `preview` "needs a valid key; ignored on the keyless build" (`.env.example`). The Manager only validates the word and forwards it. What Preview resolves to, including its fallback to Stable when no newer preview exists, is defined in [[projects/cloakbrowser/architecture/release-channels-and-versioning|release channels and versioning]]. A channel switch can change version-gated launch flags as well as the binary. ^[inferred]

## History (CHANGELOG, dated)

- v0.1.0 (19/08/2026): in-app Settings panel with hot-apply, app-wide licensing, tier and version badge, keyless empty-state prompt.
- v0.1.4 (21/08/2026): license and seat-limit banner.

## Open Questions

- Whether an env override should be shown in Settings (or block the PUT) is not addressed in code or docs. ^[ambiguous]
- `settings.json` holds the key in plain text in the data dir. File permissions are not set by `settings_store.py`; the wrapper's own `license.key` is written with mode 600. ^[inferred]

## Sources

- ~/code/CloakBrowser-Manager (commit `11e0043`): `backend/settings_store.py`, `backend/env_file.py`, `backend/main.py` (license config, `_resolve_setting`, `_mask_key`, settings routes, launch error mapping), `backend/browser_manager.py` (`resolve_binary_status`, `is_seat_limit_error`, `license_error_detail`), `backend/models.py` (`SettingsResponse`, `SettingsUpdate`), `.env.example`, `README.md` (CloakBrowser license key), `CHANGELOG.md` (0.1.0, 0.1.4), `backend/tests/test_api.py` (settings tests)
