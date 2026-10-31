---
title: >-
    Profile files and artifacts
category: subsystems
tags: [file-transfer, profile-manager, docker, storage-quota, cdp]
aliases: [backend/artifacts.py, backend/downloads.py, artifact store, profile files, ARTIFACT_MAX_BYTES]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/profile-database]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/download-capture]]"
    type: implements
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/artifacts.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/downloads.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/database.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/README.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_artifacts.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_downloads.py
summary: >-
    Per-profile file store (v0.1.6): streamed uploads and captured downloads live beside, not inside, the profile dir, with per-profile caps, crash reclaim and a GTK file-chooser view under Docker.
provenance:
  extracted: 0.86
  inferred: 0.12
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:27:05+01:00
updated: 2026-10-31T06:46:59+01:00
---

# Profile files and artifacts

Since v0.1.6 (08/10/2026) a profile has files of its own: documents pushed in through the API, and files its browser downloaded. `backend/artifacts.py` (degree 27, the third non-test hub) owns the on-disk store and its safety rules; `backend/downloads.py` turns Chromium download events into artifacts ([[projects/cloakbrowser-manager/techniques/download-capture|download capture]]); metadata rows live in the `artifacts` table of the [[projects/cloakbrowser-manager/subsystems/profile-database|profile database]]. The feature matters mostly under Docker, where the browser cannot see the operator's filesystem ([[projects/cloakbrowser-manager/architecture/platform-modes]]).

## Layout and why

- Bytes live at `<data_dir>/artifacts/<profile_id>/<artifact_id>/<sanitized name>`, outside the profile's Chromium user-data dir. Duplicating a profile's browser state therefore never carries merchant documents, and an upload arriving mid-copy cannot mutate a directory a snapshot is reading. Deleting a profile deletes its files.
- The artifact id directory, not the file name, is the trust boundary, so a crafted name cannot escape it. The file keeps its real name so a page that receives it via CDP sees the expected name and extension.
- The upload response's `container_path` is what a CDP call that takes a path (`DOM.setFileInputFiles`, `Input.dispatchDragEvent`) hands to the page; uploading alone selects nothing.

## Upload path

`POST /api/profiles/{id}/files` takes the raw body (not multipart) and the name in a URL-encoded `X-File-Name` header (a raw UTF-8 name works too).

1. Refuse early from `Content-Length`: over the per-file cap → 413; would breach count or total → 507. Both answers carry `Connection: close` so the client stops sending.
2. Stream chunks to an exclusive `.incoming-<id>` temp file, writes off the event loop, enforcing the cap as bytes arrive (Content-Length "proves nothing"); fsync, then atomic rename.
3. Re-check the quota against the real size; an over-quota upload is deleted, not published.
4. Write the row; if that fails, delete the bytes, since an unreferenced file could never be listed, deleted or counted.
5. Content type: the request's `Content-Type` unless it is a generic default (`application/octet-stream`, curl's form-urlencoded), else a guess from the name.

Downloads (`GET .../files/{artifact_id}`) are always served as an opaque attachment with `nosniff` and `no-store`, and 409 while a captured download is still pending. Deletion removes the row first, then the bytes, and is refused (409) for a pending download while the browser runs.

## Limits (as of commit 11e0043)

| Variable | Default | Meaning |
|---|---|---|
| `ARTIFACT_MAX_BYTES` | 256 MiB | Largest single file (413) |
| `ARTIFACT_MAX_COUNT` | 200 | Files per profile (507) |
| `ARTIFACT_MAX_TOTAL_BYTES` | 2 GiB | Total per profile (507) |

Nothing expires and nothing is evicted to make room. Only `ready` rows count toward the byte total; a pending download holds a count slot from the moment it starts.

## Cross-platform file names

`safe_filename` applies the strictest rules of every supported host, so a profile directory written under Docker still opens on Windows: NFC normalization, strip path parts and control characters, replace Windows-illegal characters, drop trailing dots and spaces, prefix reserved device names (CON, NUL, COM1..9, LPT1..9 including superscript forms), cap at 255 UTF-8 bytes while keeping the extension, and on Windows trim to fit a 250-character path budget.

## File-chooser view (Docker only)

A website's "Choose file" opens the container's own GTK dialog, which cannot reach an id-keyed path. The store keeps a flat `files/` folder of symlinks per profile, named by a `picker_name` fixed once at upload (" (1)", " (2)" suffixes on clashes, so an existing path never starts pointing at a different document), and rewrites the GTK bookmarks file with one sidebar entry per profile: "Uploads — <profile name>". It is cosmetic and never fails an upload. It is disabled on native hosts, where `$HOME` belongs to a real user and Windows restricts symlink creation.

## Crash repair

At startup `reclaim_all()` deletes half-written `.incoming-*` files and artifact directories that no row references (a container kill skips the endpoint's cleanup), then the picker views and bookmarks are rebuilt once.

## Pinned by tests

`backend/tests/test_artifacts.py` and `test_downloads.py` (as of commit `11e0043`) confirm the design above and fix these edges.

**Uploads and names**

- **`safe_filename`.** `../../etc/passwd` becomes `passwd`, and a Windows path such as `..\..\windows\system32\cfg` becomes `cfg`. `''`, `.`, `..` and `None` all become `file`. NUL bytes are removed, and a 500-character name is cut to 255 bytes. An upload named `../../../../etc/passwd` is stored as `passwd` under `artifacts_root()`.
- **Upload result.** A successful upload returns 201 with `kind: upload` and `state: ready`. Its `container_path` keeps the real name (e.g. `weekly report.csv`) and is never under the profile's `user_data_dir`. An unknown profile gives 404. A missing `X-File-Name` gives 400 and stores nothing.
- **Encoded and raw names.** URL-encoded names round-trip exactly, e.g. `résumé (final) 100%.pdf`. A raw UTF-8 name, which Starlette hands over decoded as Latin-1, is accepted too.
- **Content type.** curl's default `application/x-www-form-urlencoded` is replaced by a guess from the name (`report.csv` → `text/csv`). `application/octet-stream` is replaced when the name gives a guess and kept for unknown extensions. With no `Content-Type` at all, the stored type is `null`.

**Limits and isolation**

- **Refusal before the body is read.** An oversized declared length gets 413, and a declared length that would breach the total gets 507, both before `save_stream` is ever called and both with `Connection: close`. A chunked body with no length is cut off at the cap with 413, and the partial file is removed. A 413 or 507 leaves no row and no bytes.
- **Blank limits.** A blank `ARTIFACT_MAX_*` value, as `docker compose` renders unset variables, means "use the default" ([[projects/cloakbrowser-manager/subsystems/vnc-viewer|container lifecycle]]).
- **Isolation.** One profile cannot fetch another profile's artifact id (404). Downloads are served as `application/octet-stream`, `attachment`, `nosniff` and `no-store`. Duplicating a profile *with browser state* does not carry its artifacts ([[projects/cloakbrowser-manager/techniques/profile-duplication-with-state]]).

**Picker view (Docker only, `docker_runtime` fixture)**

- **Stable names.** Each upload appears as a symlink under its real name, and a clash gets `report (1).csv`. A `picker_name` keeps pointing at the same bytes after later uploads, and deleting one file never renames its neighbours.
- **Bookmarks.** The GTK bookmark line is `file://<picker_dir> <label>`, ending with the profile name, and renaming the profile relabels it. Bookmarks are rebuilt without loading every profile's tags.
- **Native hosts.** `picker_view_enabled()` is false and `_gtk_bookmarks_path()` is `None`, while storage and serving behave exactly as on Linux.
- **Startup repair.** Only profiles that hold files are reconciled. Bytes with no row are deleted, a dangling picker link is dropped, and the picker view and the staging directory are left alone.

**Downloads (`DownloadTracker`)**

- **Recording.** `Browser.downloadWillBegin` records a `pending` row of `kind: download` with size 0 and a sanitised name; a nameless download is called `download`. On `completed`, the file moves from the GUID-named staging file to the canonical path and becomes `ready`. The **size comes from the bytes on disk, not the browser's claim**.
- **Foreign downloads.** A download that landed elsewhere, because another CDP client re-pointed the path, leaves **no row at all**, and its reported size is never charged to the profile's quota. A cancelled download leaves nothing behind.
- **Cap enforcement.** At the count cap, `begin()` returns `None`. The refused transfer is cancelled only once a staged GUID file proves it belongs to this profile, so another client's download is never killed. A download that grows past the byte cap mid-flight is cancelled and marked `failed`, and so is one found over the cap at publication. Publishing does not re-charge the count slot it already holds.
- **Restarts.** After a restart, `reconcile_interrupted()` marks leftover `pending` rows `failed` and empties the staging directory; `ready` rows are untouched. A pending download can be deleted only once its browser is gone; while the browser runs, deletion and download both return 409 "still in progress".
- **CDP session.** Replies are matched to calls by id, not arrival order, and a refused call raises. A closed socket fails pending calls at once, and malformed messages do not end capture. A refused re-arm drops the session so it reconnects. A session that armed successfully earns a fresh reconnect budget of two attempts. Tracking survives a reconnect: a completion that arrives on the new socket still publishes the file.

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager/backend/artifacts.py, backend/downloads.py, backend/main.py, backend/database.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_artifacts.py, backend/tests/test_downloads.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/README.md (Profile files), CHANGELOG.md (0.1.6, 08/10/2026) (commit 11e0043)
