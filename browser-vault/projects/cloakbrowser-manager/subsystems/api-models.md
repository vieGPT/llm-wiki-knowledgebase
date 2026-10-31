---
title: >-
    API models
category: subsystems
tags: [pydantic, rest-api, schema, profile-manager]
aliases: [backend/models.py, ProfileCreate, ProfileUpdate, ProfileResponse, BaseModel classes]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/api-server]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/profile-database]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/settings-and-license]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/models.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/runtime.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_models.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py
summary: >-
    The 18 Pydantic v2 BaseModel classes in backend/models.py that define the Manager API's request and response shapes; inputs forbid unknown fields and constrain enums.
provenance:
  extracted: 0.90
  inferred: 0.07
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:27:05+01:00
updated: 2026-10-31T06:46:59+01:00
---

# API models

`backend/models.py` (degree 21) holds the 18 Pydantic v2 classes that share the `BaseModel` parent and define every request and response body of the [[projects/cloakbrowser-manager/subsystems/api-server|API server]]. They are grouped here on one page per the inheritance rule. The frontend mirrors them as TypeScript types ([[projects/cloakbrowser-manager/subsystems/frontend-api-client]]).

## The 18 classes (as of commit 11e0043)

| Group | Classes |
|---|---|
| Profile input | `ProfileCreate`, `ProfileUpdate`, `ProfileDuplicateRequest` ([[projects/cloakbrowser-manager/techniques/profile-duplication-with-state\|duplication]]), `ReorderRequest`, `TagCreate` |
| Profile output | `ProfileResponse`, `TagResponse`, `ProfileStatusResponse`, `LaunchResponse` |
| Files | `ArtifactResponse` |
| System | `StatusResponse`, `UpdateCheckResponse`, `SettingsResponse`, `SettingsUpdate` |
| Tools | `ProxyTestRequest`, `ProxyTestResponse`, `ClipboardRequest`, `LoginRequest` |

## Design choices

- **Strict inputs.** `ProfileCreate`, `ProfileUpdate`, `ProfileDuplicateRequest` and `ProxyTestRequest` set `extra="forbid"`, so a misspelled field is a 422 instead of being silently ignored.
- **Enums as `Literal`.** `gpu_family` ∈ {auto, nvidia, intel}; `human_preset` ∈ {default, careful}; `color_scheme` ∈ {light, dark, no-preference}. Runtime fields reuse `HostOS`, `RuntimeMode` (native/docker) and `ViewerMode` (native-window/vnc) from `backend/runtime.py`.
- **Partial updates.** Every `ProfileUpdate` field is optional and the route dumps with `exclude_unset`, so only fields the client sent change. Explicit `null` is allowed for clearable fields (proxy, timezone, locale, color scheme, notes) but rejected for `gpu_family` by a validator.
- **Defaults mirror the database.** `ProfileCreate` defaults (geoip, 3rd-party cookies, Google default, preview capture, session restore on; humanize, auto-launch, [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|transparent proxy]], [[projects/cloakbrowser-manager/techniques/portable-cookies|portable cookies]] off; 1920×1080) match the column defaults in the [[projects/cloakbrowser-manager/subsystems/profile-database|profile database]].
- **Response merges state.** `ProfileResponse` is a profile row plus live status from the browser manager (`status`, `runtime_mode`, `viewer_mode`, `vnc_ws_port`, `cdp_url` ([[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying|CDP endpoint proxying]]), `last_error`). `clipboard_sync` coerces a NULL from old rows to true.
- **Size caps.** `ClipboardRequest.text` ≤ 1 MiB; `ProxyTestRequest.proxy` 1 to 512 characters.
- **Secrets never echoed.** `SettingsResponse` ([[projects/cloakbrowser-manager/subsystems/settings-and-license]]) has only `license_key_set`, a masked key and the channel. `SettingsUpdate` uses `None` for "leave unchanged" and `""` for "clear the key".
- `ArtifactResponse` adds `container_path` (where the browser sees the file) and `picker_name` (its stable name in the file-chooser view) to the stored row ([[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]]).

## Pinned by tests

`backend/tests/test_models.py`, plus the schema cases in `test_api.py` (as of commit `11e0043`):

- **Defaults.** A minimal `ProfileCreate(name=…)` has no seed (the database mints one), 1920×1080, `gpu_family="auto"`, `humanize=False`, `geoip=True`, `human_preset="default"`, `extension_paths=[]`, `allow_3p_cookies=True` and `launch_args=[]`.
- **Rejected values.** `gpu_family="amd"`, `human_preset="fast"`, `color_scheme="auto"` and `ProfileUpdate(gpu_family=None)` all raise `ValidationError`.
- **Removed fields.** `ProfileCreate(user_agent=…)` is rejected. Over HTTP, `{"headless": true}` and the duplicate body `{"copy_state": true}` return 422. The removed per-device fields (user agent, platform, GPU strings, hardware concurrency) are now derived by the engine from the seed ([[projects/cloakbrowser/techniques/fingerprint-seed-identity]]). Rejecting them stops an old client from believing it set them. ^[inferred]
- **Partial updates.** `ProfileUpdate(name=…).model_dump(exclude_unset=True)` is exactly `{"name": …}`, and likewise for `launch_args` alone.
- **Clipboard cap.** `ClipboardRequest` accepts exactly 1,048,576 characters and rejects 1,048,577 ([[projects/cloakbrowser-manager/subsystems/vnc-viewer]]).
- **`cdp_url`.** It defaults to `None` on `LaunchResponse`, `ProfileStatusResponse` and `ProfileResponse`, and accepts the relative `/api/profiles/<id>/cdp`. A `LaunchResponse` requires `runtime_mode` and `viewer_mode`; there is no default.
- **Tags.** `TagCreate` takes a tag with an optional colour, which defaults to `None`.

## Open Questions

- `ProfileStatusResponse` lacks the `last_error` field that `BrowserManager.get_status()` returns; with Pydantic's default `extra="ignore"`, the per-profile status route probably drops post-handshake license denials. ^[ambiguous]

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager/backend/models.py, backend/main.py, backend/runtime.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_models.py, backend/tests/test_api.py (commit 11e0043)
