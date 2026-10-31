---
title: >-
    Per-profile identity isolation
category: techniques
tags: [identity, fingerprinting, profile-manager, isolation, evasion, visibility/internal]
aliases: [profile isolation, one profile one machine]
relationships:
  - target: "[[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: uses
  - target: "[[projects/cloakbrowser/techniques/persistent-profile-warmup]]"
    type: uses
  - target: "[[projects/cloakbrowser/subsystems/profile-seed]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection]]"
    type: mitigates
  - target: "[[projects/cloakbrowser/techniques/humanized-input]]"
    type: uses
  - target: "[[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/README.md
  - ~/code/CloakBrowser-Manager/frontend/src/components/ProfileForm.tsx
  - ~/code/CloakBrowser-Manager/frontend/src/components/ProfileForm.test.tsx
  - ~/code/CloakBrowser-Manager/frontend/src/hooks/useProfiles.ts
  - ~/code/CloakBrowser-Manager/backend/browser_manager.py
  - ~/code/CloakBrowser-Manager/backend/database.py
  - ~/code/CloakBrowser-Manager/backend/main.py
  - ~/code/CloakBrowser-Manager/backend/artifacts.py
  - ~/code/CloakBrowser-Manager/assets/manager-macos.png
summary: >-
    Each Manager profile is a separate device: its own stored fingerprint seed, GPU family, screen and user-data directory, turned into CloakBrowser flags at launch so accounts do not share signals.
provenance:
  extracted: 0.70
  inferred: 0.27
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T06:22:07+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Per-profile identity isolation

The Manager's central promise is "each profile is a different machine": its own fingerprint seed, GPU family, screen, cookies, localStorage, cache and history, persistent across restarts (README). The Manager does not generate fingerprints itself. It stores a small set of identity parameters per profile and turns them into CloakBrowser flags at every launch, so the engine's [[projects/cloakbrowser/techniques/fingerprint-seed-identity|seed identity]] derives a coherent device per profile. The goal is to stop a site from linking accounts through shared device signals ([[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage|cross-profile linkage]]). ^[inferred]

## What the user can set per profile (ProfileForm, commit 11e0043)

| Setting | UI | Becomes at launch |
|---|---|---|
| Fingerprint seed | number field; blank means "Auto (random)"; dice button rolls 10000-99999 | `--fingerprint=<seed>` |
| GPU family | `auto` ("from seed"), NVIDIA, Intel; on macOS hosts locked to "Apple Silicon (automatic)" | `--fingerprint-gpu-vendor=NVIDIA` or `=Intel`; nothing for `auto` or on macOS ([[projects/cloakbrowser/detection-vectors/webgl-gpu-fingerprinting]]) |
| Screen | six presets (1920×1080 default, 2560×1440, 1366×768, 1440×900, 1536×864, 1280×720) or custom width/height | `--fingerprint-screen-width/height` |
| Humanize | off by default; preset `default` or `careful` | wrapper `humanize` / `human_preset` ([[projects/cloakbrowser/techniques/humanized-input\|humanized input]]) |
| Extensions | unpacked extension directories | wrapper `extension_paths` |
| Advanced args | unrestricted Chromium flags | appended to the launch args |
| Cookies | third-party cookies (on), [[projects/cloakbrowser-manager/techniques/portable-cookies\|portable cookies]] (off) | `--fingerprint-allow-3p-cookies`, `--fingerprint-portable-cookies` |

Network and locale settings (proxy, GeoIP, timezone, locale, [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|transparent proxy]]) are on [[projects/cloakbrowser-manager/techniques/per-profile-network-and-locale|Per-profile network and locale]].

- The form hint states the coherence model: "The seed selects a coherent GPU model, CPU, memory, and screen profile within the family" (Windows persona), or "a coherent Apple Silicon model and matching hardware profile" on macOS. The family is a constraint on the seed's choice, not a separate random value. ^[inferred]
- **Not user-settable: the platform persona.** The backend always passes `--fingerprint-platform`, `macos` on a macOS host and `windows` otherwise; a code comment says the persona "is always determined by the runtime, not editable profile data". This keeps the persona consistent with what the host can render, see [[projects/cloakbrowser/techniques/platform-persona-spoofing|platform persona spoofing]]. ^[inferred]
- GPU vendor overrides are skipped on macOS because Windows vendor families are "incoherent" there (code comment); the form mirrors this by forcing `auto`.

## Where the identity lives

- **Seed in the Manager database.** A new profile without a seed gets `random.randint(10000, 99999)` server-side; Reset re-rolls it. Because the Manager always passes an explicit `--fingerprint=<seed>`, the wrapper's own `.cloakbrowser-seed` file mechanism is bypassed ([[projects/cloakbrowser/subsystems/profile-seed|profile seed]] treats an explicit seed as an opt-out). The SQLite row, not the profile directory, is the identity's source of truth. ^[inferred]
- **Browser state in a per-profile user-data directory** (`user_data_dir`, e.g. `/data/profiles/<id>` in the test fixture), launched with CloakBrowser's persistent context, so cookies, storage and history never mix between profiles. See [[projects/cloakbrowser/techniques/persistent-profile-warmup|persistent profile warmup]].

## Other per-profile dimensions (backend, commit 11e0043)

Beyond the form fields above, the backend gives each running profile its own resources:

| Dimension | Mechanism |
|---|---|
| Platform persona | Host OS → `--fingerprint-platform` (see above) |
| Debug port | One OS-chosen loopback CDP port per running browser, reachable only through the Manager ([[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]) |
| Live display (Linux) | One X display + KasmVNC instance per running profile, started at the profile's screen size ([[projects/cloakbrowser-manager/subsystems/vnc-viewer]]) |
| Files | Artifact store under `<data_dir>/artifacts/<id>`, outside the user-data dir ([[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]]) |
| Preview | `last_screenshot.jpg` inside the profile's own user-data dir |

The macOS screenshot (`assets/manager-macos.png`) shows the editor's Basic (name, seed 57855 with a dice re-roll button), Network (proxy, timezone, locale, GeoIP checkbox "Auto-detect timezone/locale from the proxy exit, or host public IP without a proxy") and Hardware sections ("GPU Family: Apple Silicon (automatic)"), with a "Pro · 151.0.7922.108.3" tier/version badge in the top bar.

## Why the isolation holds up

- **Persistence makes each identity a returning visitor.** A stable seed plus a stable user-data dir keeps fingerprint and cookie history in agreement across sessions; a new fingerprint on old cookies, or an old fingerprint on an empty profile, would contradict itself. ^[inferred]
- **Seed-derived coherence beats per-field randomisation.** One integer drives GPU, screen, CPU, fonts and noise together inside the binary, so two profiles differ as wholes rather than as mismatched field mixes, which keeps each internally consistent ([[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection]]). ^[inferred]

## Lifecycle operations and their effect on identity

- **Reset**: wipes cookies, history and site data and generates a new fingerprint; bookmarks, settings and default search stay. The profile stays stopped. Net effect: a fresh device on the same configuration. ^[inferred] Backend detail: `POST /api/profiles/{id}/reset` takes the `hold_stopped` lock, deletes a fixed list of files and directories in `Default/` (Cookies, History, Login Data, Favicons, Shortcuts, Top Sites, Visited Links, TransportSecurity, session and tab files, Cache, Code Cache, GPUCache, Service Worker, Local/Session Storage, IndexedDB, File System, GCM Store, extension state and more), and keeps Bookmarks, Preferences, Secure Preferences and Web Data so the Google default survives. It does not touch the artifact store or the preview screenshot.
- **Duplicate (config only)**: copies settings **and the fingerprint seed**, with empty browser state.
- **Duplicate with browser state**: also copies cookies, logged-in sessions and history; only offered while the source is stopped (pinned by `ProfileForm.test.tsx`). See [[projects/cloakbrowser-manager/techniques/profile-duplication-with-state|Profile duplication with state]].
- Both duplicate modes produce two profiles with the same seed, hence the same derived device fingerprint. Running both against the same site would present one device under two accounts, which is exactly the linkage this technique exists to prevent; the UI does not warn about it. ^[inferred]

## Limits of the isolation

- Advanced launch args "can override Manager-controlled behavior" (form text); an arg such as an explicit `--fingerprint` could break the per-profile mapping. ^[inferred]
- Under Linux Docker every profile renders through SwiftShader (`--use-angle=swiftshader`), so the real rendering backend is identical across profiles and the engine's GPU spoofing has to hide it. ^[inferred]
- All profiles on one Manager share the host, binary version and license seat pool; isolation is at the fingerprint and storage level, not the network level unless each profile has its own proxy. ^[inferred]

## Open Questions

- ~~Whether `Reset` also clears the profile's file store and preview screenshot.~~ Answered from `backend/main.py`: it does not; artifacts live outside the user-data dir and `last_screenshot.jpg` is not on the reset list.
- Whether duplicating should re-roll the seed by default (to avoid two identical devices) is not discussed in the sources. ^[ambiguous]

## Sources

- ~/code/CloakBrowser-Manager (commit `11e0043`): `README.md` (Features), `frontend/src/components/ProfileForm.tsx`, `frontend/src/components/ProfileForm.test.tsx`, `frontend/src/hooks/useProfiles.ts`, `backend/browser_manager.py` (`_build_fingerprint_args`, launch options), `backend/database.py` (seed default and re-roll), `backend/main.py` (reset file list), `backend/artifacts.py` (store location), `assets/manager-macos.png`
