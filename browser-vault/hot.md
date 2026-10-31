---
title: Hot Cache
updated: 2026-10-31T10:48:34+01:00
pages: 119
generated_by: obsidian-wiki memory hot
---

# Hot Cache

*Generated snapshot of recent vault activity. Rebuilt by `obsidian-wiki memory hot` — edits to the generated sections are overwritten, so put durable prose in `## Key Takeaways`, which survives every rebuild.*

## Recent Activity

- [2026-10-31] INGEST source=ast-extract class stubs (CloakBrowser b1a00ed, CloakBrowser... pages_created=48 pages_updated=10 ...
- [2026-10-31] WIKI_UPDATE project=cloakbrowser pages_created=0 pages_updated=0 ...
- [2026-10-31] WIKI_UPDATE project=cloakbrowser-manager pages_created=0 pages_updated=0 ...

## Active Threads

*No open threads.*

## Key Takeaways

CloakBrowser is a thin Python/JS/.NET wrapper around a closed C++-patched Chromium: one fingerprint seed derives a coherent device, the wrapper only picks the binary, builds flags (persona, GeoIP timezone/locale, WebRTC IP) and optionally humanizes input. Every detection vector has at least one mitigating technique; TLS, headless and screen coverage rest mostly on the binary and README claims.
The Manager is policy over that mechanism: one SQLite seed and one user-data dir per profile, its own per-profile CDP endpoint (not cloakserve). Duplicating a profile copies its seed, and the 90,000-value seed range collides around 40 profiles — the main cross-profile linkage risk.
Code beats docs where they disagree (GeoIP timeout 20 s, welcome banner 1 day, humanized ElementHandles). Both projects are tracked by source_repo and last_commit_synced; run wiki-update per repo after each git pull.

## Flagged Contradictions

*None flagged.*
