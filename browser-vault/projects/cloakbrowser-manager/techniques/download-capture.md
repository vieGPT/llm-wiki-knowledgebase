---
title: >-
    Download capture
category: techniques
tags: [cdp, file-transfer, docker, browser-automation, playwright]
aliases: [Browser.setDownloadBehavior capture, last-writer-wins downloads, DownloadTracker]
relationships:
  - target: "[[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/downloads.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/main.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/browser_manager.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/README.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_downloads.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_api.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_browser_manager.py
summary: >-
    Under Docker the Manager holds its own CDP socket per browser, stages GUID-named downloads, publishes on completion and re-arms when a client that changed download behaviour detaches.
provenance:
  extracted: 0.84
  inferred: 0.14
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:27:05+01:00
updated: 2026-10-31T06:53:05+01:00
---

# Download capture

In the Docker build, anything a profile's browser downloads would otherwise be trapped in the container. `backend/downloads.py` (v0.1.6) captures those files into the profile's [[projects/cloakbrowser-manager/subsystems/profile-files-and-artifacts|artifact store]]. It is a mechanism over the Chrome DevTools Protocol, not an evasion technique, and it has to coexist with external automation attached through the [[projects/cloakbrowser-manager/techniques/cdp-endpoint-proxying|CDP proxy]]. On native Windows/macOS it is off: the browser writes to the user's own Downloads folder, and diverting that would take their files away.

## Mechanism

1. When a Docker profile launches, the [[projects/cloakbrowser-manager/subsystems/browser-manager|browser manager]] starts `downloads.watch()`. It first fails any downloads left `pending` by a previous browser and clears the staging dir (done once only: on a reconnect it would delete a live transfer's bytes).
2. The watcher opens its own browser-level CDP WebSocket and sends `Browser.setDownloadBehavior` with `allowAndName`, a staging path (`artifacts/<id>/.downloads`) and `eventsEnabled`. Files land named by GUID, so a filename appearing proves nothing about completion.
3. `Browser.downloadWillBegin` → a `pending` artifact row (zero bytes, slot claimed against the count cap). `Browser.downloadProgress` with `completed` → re-check size and quota on the real bytes, atomically move the staged file into place, mark `ready`. Canceled → row and bytes discarded.
4. A transfer that grows past the per-file cap mid-flight is cancelled with `Browser.cancelDownload` and marked `failed`; one refused at start for quota is cancelled once its staged bytes show up.
5. One `DownloadTracker` lives for the browser's lifetime, not per socket: a socket can drop mid-transfer and the completion event arrives on the next one.
6. A dropped socket is retried with backoff 1, 2, 5, 10, 30 s while the browser runs; a session that armed successfully earns a fresh budget, so drops spread over days do not exhaust it. Capture never raises into the launch path.
7. On stop, the watcher is awaited (it owns staged files) and pending rows are failed at once rather than at the next launch.

## Last-writer-wins with automation clients

Chromium's download behaviour is browser-wide and last-writer-wins. Playwright's `connect_over_cdp` sets its own download directory (or disables downloads) during initialisation, so while such a client is attached its downloads go where it asked, not into the profile's files (README). The design rules that follow:

- **Record only what it can see.** Events still arrive on the Manager's socket for the other client's downloads. A completion with nothing in staging is "not ours": the row is deleted rather than kept as a phantom failure.
- **Never cancel a transfer it does not own.** Size or quota cancellations happen only when the staged file exists, so an automation session's downloads are never broken.
- **Re-arm on detach.** When a client detaches, Chromium falls back to its *default* behaviour, not to the Manager's. The CDP proxy inspects client→browser text frames for `Browser.setDownloadBehavior` / `Page.setDownloadBehavior` without a `browserContextId` (a call naming a context targets one the client created, which Chromium will not undo for the Manager). If one was seen, after the upstream socket closes it sets the running profile's re-arm event and the watcher re-sends its configuration. A late notification for a superseded browser is ignored.

The README says capture is "re-armed as soon as a proxied CDP connection closes". The code, as pinned by tests, re-arms only when that connection changed download behaviour for the default context. A client that connects to Chrome's debugging port directly, bypassing the proxy, would leave capture off until the next launch, but the port is loopback-only inside the container. ^[inferred]

## Pinned by tests

`backend/tests/test_downloads.py`, `test_api.py` and `test_browser_manager.py` (as of commit `11e0043`) pin the rules above:

- **Which frames count.** `_sets_download_behaviour` returns true for `Browser.setDownloadBehavior` (with or without a flattened `sessionId`) and for `Page.setDownloadBehavior`. It returns false for a call naming a `browserContextId`, for `Runtime.evaluate` whose expression merely contains the string, for non-JSON text and for JSON arrays.
- **Re-arm conditions.** A client that set the behaviour re-arms capture when it leaves, and the hook fires only once the upstream socket is closed; its frames still reach the browser. A client that left the behaviour alone does not re-arm. Neither does a proxy session that never reached the browser (upstream connect refused).
- **Watcher session.** On re-arm, the watcher's second message is `Browser.setDownloadBehavior` again. A refused re-arm drops the session so it reconnects. A reconnect after a successful arm gets a fresh two-attempt budget (five attempts succeed in sequence in the test). A completion that arrives on a reconnected socket still publishes.
- **Ownership.** A completion whose bytes landed elsewhere leaves no row and charges no quota. A refused transfer is cancelled only after its GUID file appears in staging.
- **Lifecycle.** Native launches start no watcher (`download_task is None`). Docker launches do. Stopping fails pending rows immediately, and a late detach from a stale browser never touches the relaunched one.

## Sources

- repo:github.com/vieGPT/CloakBrowser-Manager/backend/downloads.py, backend/main.py, backend/browser_manager.py (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/README.md (Profile files), CHANGELOG.md (0.1.6) (commit 11e0043)
- repo:github.com/vieGPT/CloakBrowser-Manager/backend/tests/test_downloads.py, test_api.py, test_browser_manager.py (commit 11e0043)
