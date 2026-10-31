---
title: >-
    Profile database
category: subsystems
tags: [sqlite, profile-manager, schema-migration, persistent-profile]
aliases: [backend/database.py, profiles.db, profile_tags, artifacts table]
relationships:
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
sources:
  - ~/code/CloakBrowser-Manager/backend/database.py
  - ~/code/CloakBrowser-Manager/backend/runtime.py
  - ~/code/CloakBrowser-Manager/CHANGELOG.md
  - ~/code/CloakBrowser-Manager/backend/tests/test_database.py
summary: >-
    backend/database.py keeps profiles, tags and artifact metadata in one SQLite file in the data dir, rebuilds the profiles table on schema drift, and mints seeds and per-profile dirs.
provenance:
  extracted: 0.88
  inferred: 0.10
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T06:20:23+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Profile database

`backend/database.py` (degree 29) is a thin, synchronous SQLite layer: `profiles.db` in the runtime data dir ([[projects/cloakbrowser-manager/architecture/platform-modes]]), opened per call with WAL journaling and foreign keys on. It holds three tables and is the single source of the per-profile configuration that the [[projects/cloakbrowser-manager/subsystems/browser-manager|browser manager]] turns into engine flags. Browser state itself (cookies, history, storage) is not in the database; it lives in each profile's Chromium user-data dir.

## Tables (as of commit 11e0043)

- **`profiles`**: id (UUID4), name, `fingerprint_seed`, proxy, timezone, locale, screen size (1920×1080 default), `gpu_family` (auto/nvidia/intel), humanize + `human_preset`, `geoip` (default on), `clipboard_sync`, `auto_launch`, `color_scheme`, `launch_args` and `extension_paths` (JSON lists), `allow_3p_cookies` (default on), `set_google_default`, `capture_preview`, `restore_session` (all default on), [[projects/cloakbrowser-manager/techniques/per-profile-transparent-proxy|`transparent_proxy`]] and [[projects/cloakbrowser-manager/techniques/portable-cookies|`portable_cookies`]] (default off), notes, `user_data_dir`, timestamps, `sort_order`. 28 columns.
- **`profile_tags`**: (profile_id, tag) primary key, optional color, `ON DELETE CASCADE`.
- **`artifacts`**: file metadata per profile (name, size, kind upload/download, state pending/ready/failed, content type, `picker_name`), cascade-deleted with the profile; bytes live on disk ([[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]]).

## Identity-related behaviour

- **Seed minting**: a new profile without an explicit seed gets `random.randint(10000, 99999)`, a space of 90,000 values. Reset re-rolls it from the same range. See [[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage]] for what a small seed space means across many profiles. ^[inferred]
- **Directory layout in one place**: `user_data_dir_for(id)` returns `<data_dir>/profiles/<id>`, so every profile has its own Chromium profile directory.
- **Duplicate** copies every setting, tags, notes and the same seed into a new row with a fresh id, directory and sort order, named "<name> (copy)". A caller may mint the id first (`new_profile_id`) so the clone's directory can be filled before the row exists.
- **Reset** changes only the seed and `updated_at`; name, proxy, locale and tags stay.

## Migration strategy

`init_db()` compares the live column set with the expected one. On any difference it rebuilds `profiles` in one transaction: copy the surviving columns into a new table, back tags up into a temp table (so no foreign key points at the table being replaced), swap, restore tags. Older free-text `gpu_vendor`/`gpu_renderer` values are folded into `gpu_family` (nvidia/intel/auto) when the old schema lacks it. This is the "existing profiles are migrated automatically" path in CHANGELOG 0.1.0, and how the v0.1.8 columns arrive switched off. ^[inferred]

## Ordering

New profiles take `min(sort_order) − 1`, so they appear at the top; drag-and-drop reordering writes `sort_order` = list position. Lists sort by `sort_order` then newest first.

## Pinned by tests

`backend/tests/test_database.py` (as of commit `11e0043`) runs every case against a temp `profiles.db`.

- **Migration confirmed, not just inferred.** The test builds a pre-0.1 schema with `platform`, `user_agent`, `gpu_vendor`, `gpu_renderer`, `hardware_concurrency` and `headless`, then runs `init_db()`. The profile and its tags survive, and `gpu_vendor = "NVIDIA"` becomes `gpu_family = "nvidia"`. All six legacy columns are gone. `launch_args`, `extension_paths`, `allow_3p_cookies`, `transparent_proxy` and `portable_cookies` are added, `geoip` starts at 1, and the two v0.1.8 flags start at 0. The removal of user-agent, platform and hardware-concurrency fields is deliberate: they are now derived by the engine from the seed ([[projects/cloakbrowser/techniques/fingerprint-seed-identity]]). ^[inferred]
- **`init_db()` is idempotent and keeps each row's stored `user_data_dir`.** It does not recompute the path from the current data dir.
- **Seed range.** A default seed always falls within 10000–99999, and reset re-rolls inside the same range, changing the seed and `updated_at` but never `created_at`.
- **Booleans are stored as SQLite integers** (`humanize` 0/1, `geoip` 1, `allow_3p_cookies` 1). `launch_args` and `extension_paths` round-trip as JSON lists, and `launch_args = None` on update is stored as `[]`.
- **Ordering.** A new profile lands on top (`sort_order` smaller than all others). Reorder writes 0…n−1, and the next profile created after a reorder gets −1.
- **Updates.** Partial updates leave other fields untouched, and updating `tags` replaces the whole set. An update with no fields returns the row unchanged; one on an unknown id returns `None`. Delete returns a boolean and cascades to `profile_tags`.
- **Caller-minted ids.** Explicit ids are honoured by both create and duplicate, and `user_data_dir` then equals `user_data_dir_for(id)`. This is the path the state-copying duplicate uses ([[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]).

## Sources

- ~/code/CloakBrowser-Manager/backend/database.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/backend/runtime.py (commit 11e0043)
- ~/code/CloakBrowser-Manager/CHANGELOG.md (commit 11e0043)
- ~/code/CloakBrowser-Manager/backend/tests/test_database.py (commit 11e0043)
