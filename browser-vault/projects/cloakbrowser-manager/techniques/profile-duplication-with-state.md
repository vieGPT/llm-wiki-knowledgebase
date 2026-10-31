---
title: >-
    Profile duplication with state
category: techniques
tags: [persistent-profile, identity, multi-account, concurrency, profile-manager]
aliases: [duplicate profile, include_browser_state, clone profile]
relationships:
  - target: "[[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/persistent-profile-warmup]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/portable-cookies]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/database.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/models.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_artifacts.py
summary: >-
    Cloning a profile copies settings and the same seed; with include_browser_state it also snapshots cookies, sessions and storage from the stopped source, publishing the clone only when complete.
provenance:
  extracted: 0.80
  inferred: 0.17
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:27:05+01:00
updated: 2026-10-31T06:46:59+01:00
---

# Profile duplication with state

`POST /api/profiles/{id}/duplicate` clones a profile in two modes (config-only since v0.1.1, with browser state since v0.1.6, 08/10/2026). Both copy every setting, tags, notes and the **same [[projects/cloakbrowser/techniques/fingerprint-seed-identity|fingerprint seed]]** into a new profile named "<name> (copy)" with its own id, user-data dir and sort order. With `{"include_browser_state": true}` the source's user-data dir (cookies, logged-in sessions, history, local storage) is copied too, so the clone launches as the same identity *and* the same session. This trades isolation for convenience: it deliberately creates two profiles that look like one device, the opposite of [[projects/cloakbrowser-manager/techniques/per-profile-identity-isolation|per-profile isolation]]. ^[inferred]

## Consistency mechanism

The hard part is copying a Chromium profile without catching it mid-write or exposing a half-built clone.

1. **Source must be fully stopped.** If a browser is launching, running or still closing for the source, the request is refused with 409 ("Stop the profile before duplicating its browser state").
2. **Hold the source.** The copy runs inside `hold_stopped`, so launch, reset and delete of the source are refused (409) until it finishes ([[projects/cloakbrowser-manager/subsystems/browser-manager]]).
3. **Mint the id first, write the row last.** The clone's id and directory are created before its database row exists. Nothing can list, launch, reset or delete a profile the database has not heard of, so an unfinished clone is unreachable ([[projects/cloakbrowser-manager/subsystems/profile-database]]).
4. **Copy off the event loop, uninterruptibly.** `copytree` runs in a worker thread (a profile with a large cache takes seconds). A thread cannot be cancelled, so request cancellation is absorbed until the worker finishes and then re-raised; otherwise the hold would be released and the clone dir deleted while the worker still writes into it.
5. **Skip what must not travel.** Chromium's `SingletonLock` (a dangling symlink), `SingletonCookie` and `SingletonSocket` would make the clone think another Chrome owns its directory; the source's preview screenshot shows the source, not the clone. Symlinks are copied as links and never followed out of the directory.
6. **Clean failure.** On any error or cancellation the clone directory is removed; no row exists to roll back.

Per-profile files ([[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]]) are stored outside the user-data dir and are never copied.

## Use and risk

- **Use**: spin up a second working copy of an already-warmed, logged-in identity, or keep a backup of a session before a risky action. ^[inferred]
- **Risk**: running source and clone at the same time, or using them for different accounts, presents the same device and the same cookies twice, which is a direct [[projects/cloakbrowser-manager/detection-vectors/cross-profile-linkage|cross-profile linkage]] signal. Re-rolling the clone's seed (reset wipes its state as well; editing the seed keeps it) separates the device but not the copied cookies. ^[inferred]
- With [[projects/cloakbrowser-manager/techniques/portable-cookies|`portable_cookies`]] on (v0.1.8), the copied cookies are the kind meant to move between machines; without it, whether copied cookies stay valid in a clone is up to the engine. ^[ambiguous]

## Pinned by tests

`backend/tests/test_api.py` (duplicate and lifecycle-safety sections) and `test_artifacts.py` (as of commit `11e0043`):

- **Config-only (default).** It returns 201 with "Src (copy)", the same seed and proxy, and a different `user_data_dir` that does not exist yet. It is allowed **while the source runs**. Unknown body keys (`copy_state`) return 422.
- **With state.** The clone gets `Local State`, `Default/Cookies`, `Preferences` and `Local Storage`. It does not get `SingletonLock`, `SingletonCookie` or `last_screenshot.jpg`. The source is left byte-for-byte as it was, lock included. A never-launched source yields a clone with no directory. Artifacts never travel.
- **Refusals.** The copy is refused (409, and nothing is created) while the source is running, mid-launch (`launch()` registers in `running` only at the end, so the gap counts as active) or still closing (`stop()` pops `running` before the context finishes flushing).
- **During the copy.** The clone is absent from `list_profiles()`. Reset, delete, launch and a second state-duplicate of the source all return 409, while a config-only duplicate still passes. An interleaved reset cannot wipe the snapshot being read.
- **Failure and cancellation.** A failing `copytree` returns 500 with the OS message (e.g. "disk full"), and leaves no row, no clone directory and no hold. A cancelled request keeps the source held until the worker thread ends, then deletes the clone directory. A worker error that arrives after cancellation does not turn the cancel into a 500.

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py (`duplicate_profile`, `_copy_browser_state_to_completion`), backend/database.py, backend/browser_manager.py, backend/models.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md (0.1.1, 0.1.6, 0.1.8) (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py, test_artifacts.py (commit 11e0043)
