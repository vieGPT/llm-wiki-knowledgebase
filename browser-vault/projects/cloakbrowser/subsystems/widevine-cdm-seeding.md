---
title: >-
    Widevine CDM seeding
category: subsystems
tags: [widevine, drm, chromium, persistent-profile]
aliases: [widevine.py, Widevine hint file]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/cloakbrowser/widevine.py
  - ~/code/CloakBrowser/bin/fetch-widevine.py
  - ~/code/CloakBrowser/bin/docker-entrypoint.sh
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/dotnet/README.md
  - ~/code/CloakBrowser/js/src/widevine.ts
  - ~/code/CloakBrowser/js/src/playwright.ts
  - ~/code/CloakBrowser/js/src/puppeteer.ts
  - ~/code/CloakBrowser/tests/test_fetch_widevine.py
  - ~/code/CloakBrowser/js/tests/widevine.test.ts
summary: >-
    Pre-writes Chromium's Widevine hint file so a sideloaded CDM works on first launch (Linux only); seeding never downloads, an opt-in fetch-widevine.py pulls and verifies the CDM.
provenance:
  extracted: 0.87
  inferred: 0.11
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Widevine CDM seeding

The CloakBrowser binary is built with Widevine support but ships no Content Decryption Module, because the CDM is a proprietary Google binary that cannot be redistributed. Users sideload a `WidevineCdm/` directory. `cloakbrowser/widevine.py` closes the gap that remains: it pre-seeds Chromium's CDM hint file into a persistent profile so a sideloaded CDM works on the very first launch. It is called on the persistent-context launch path (see [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]) and mirrored in the JS wrapper and the .NET `Widevine.cs` ([[projects/cloakbrowser/subsystems/dotnet-client|.NET client]]).

## Why seeding is needed

Chromium finds a sideloaded CDM in two phases: an early-startup pass reads a hint file from the user-data-dir, and a later asynchronous component-updater pass writes that file. On a fresh profile the hint does not exist on first launch, and Playwright passes `--disable-component-update`, so the updater never writes it. Without seeding, Widevine works only after a manual two-launch dance (issue #96).

## Behavior

- Writes `<user-data-dir>/WidevineCdm/latest-component-updated-widevine-cdm` containing a compact JSON object with the absolute, symlink-resolved CDM path. The JSON formatting is chosen to byte-match the JS wrapper's output.
- Skips the write when the file already holds the same content.
- CDM lookup order: `CLOAKBROWSER_WIDEVINE_CDM` if set (used exclusively; an empty or invalid value disables seeding rather than matching a stray `manifest.json` in the working directory), then `WidevineCdm/` next to the Chrome binary, then `~/.cloakbrowser/WidevineCdm`. A directory counts only if it contains `manifest.json`.
- The cache-dir location is version-independent: the CDM `.so` is architecture-specific but not version-specific, so one fetched CDM serves any free or Pro binary.
- No-op on non-Linux platforms (the hint mechanism is Linux/ChromeOS-specific; on Windows the CDM cannot initialise due to DRM host verification, and macOS uses a different layout), when `CLOAKBROWSER_WIDEVINE` is falsey (`0`, `false`, `off`, `no`), or for an ephemeral profile (empty `user_data_dir`).
- Never raises: every failure is logged, because seeding must not break a launch.

### JavaScript mirror (B5 batch)

`js/src/widevine.ts` ("Mirrors Python cloakbrowser/widevine.py") has the same lookup order, `manifest.json` test, falsey `CLOAKBROWSER_WIDEVINE` values, Linux-only guard (`process.platform`) and never-throw rule. `seedWidevineHint(userDataDir, binaryPath)` runs in both `launchPersistentContext` implementations, Playwright and Puppeteer, after the binary path is resolved and before the browser starts. Plain `launch()` and `launchContext()` never seed, because they use Playwright's ephemeral profile. An existing hint that cannot be read is rewritten with a warning. A set but invalid `CLOAKBROWSER_WIDEVINE_CDM` (no `manifest.json`) logs a warning and skips seeding (commit b1a00ed).

## Obtaining the CDM

The README describes two routes (as of 26/10/2026, commit b1a00ed): `bin/fetch-widevine.py` pulls the CDM from Google's component server (Linux x86-64 only, SHA-256 and CRX3-signature verified) into `~/.cloakbrowser/WidevineCdm`; or copy `WidevineCdm/` from a local Chrome install. In Docker, `CLOAKBROWSER_FETCH_WIDEVINE=1` runs the fetch on container start; it is off by default, so no network call happens unless opted in. The P2 pass read both scripts directly; the test-derived section further down still holds.

### Docker opt-in (`bin/docker-entrypoint.sh`)

- **Stated reason**: the entrypoint comment says the fetch exists "so persistent contexts present as a real Chrome", because "a DRM/EME probe is used by some bot detectors".
- **Trigger**: `CLOAKBROWSER_FETCH_WIDEVINE` set to `1`, `true`, `yes` or `on` (any case). The fetch is skipped when `CLOAKBROWSER_WIDEVINE_CDM` already points at a CDM or `CLOAKBROWSER_WIDEVINE` disables seeding (`0`, `false`, `off`, `no`). The value is tested with `printf` rather than `echo`, so a value such as `-n` cannot be swallowed as a flag.
- **Target**: the version-independent cache root `~/.cloakbrowser/WidevineCdm`, which the seeder's lookup reaches after the per-binary directory. Any process in the container (the main command or a `docker exec`) and any binary, free or Pro, finds the CDM with no env var. It is fetched per container from Google's component server and persists in the `~/.cloakbrowser` volume.
- **Failure**: best-effort. A non-zero exit from the fetcher prints "Widevine fetch failed; continuing without it" and the container command still starts. The entrypoint runs after Xvfb and openbox are up and just before `exec` (see [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]]).

### Fetcher mechanics (`bin/fetch-widevine.py`)

Facts from the script that its tests do not cover:

- **Dependency-free by design**: standard library only (HTTP, zip, hashing), "no curl/jq/unzip needed"; `cryptography` is optional and only enables the signature check. Options: `--out` (default `$CLOAKBROWSER_CACHE_DIR/WidevineCdm`, falling back to `~/.cloakbrowser/WidevineCdm`), `--force` to re-download over a cache hit, `--quiet` to print only the final path and errors. It prints the CDM directory on success.
- **Update protocol**: it posts an on-demand update check (protocol 4.0) for the Widevine app id to Google's component-update JSON endpoint, strips the XSSI prefix from the reply, and claims a deliberately low installed version (`1.4.9.1088`) so the server always offers the current CDM. It downloads the first HTTPS URL in the reply's pipelines together with that operation's SHA-256, if present.
- **Which proof is trusted**: only the RSA PKCS#1 v1.5 / SHA-256 proofs of the CRX3 header are checked, because Google signs the Widevine component with RSA; ECDSA proofs are ignored. Proofs from any key that does not derive the Widevine app id are skipped, and if none matches the script refuses to install.
- **Replacement is not concurrency-safe**: the new CDM is staged in a temp directory and renamed into place atomically, but the removal of the old directory that precedes the rename is not atomic. The docstring states that two processes writing the same output directory at once are not safe. The test suite's "replaced atomically" therefore holds for one writer only.
- **Exit status**: any unexpected exception prints `[fetch-widevine] error: …` and exits 1, which the entrypoint treats as a soft failure.

## The fetcher, pinned by tests (`tests/test_fetch_widevine.py`, B7)

The suite runs offline. It synthesizes a CRX3 package signed with a throwaway RSA key and mocks the component-server calls, so the real verification path is exercised (commit b1a00ed).

- **Self-contained CRX3 parsing**: a minimal protobuf reader handles varints, with explicit "truncated" and "too long" errors, and keeps only length-delimited fields. No protobuf dependency is needed. ^[inferred]
- **Trust anchor**: `APP_ID` is pinned to `oimompecagnajdejgnnjijobebaeigek`, the Widevine component id. The test comment warns that a typo here "would silently accept the wrong publisher". A CRX id is derived from the first 16 bytes of the publisher key's SHA-256, written in the letters a to p.
- **Verification**: the file must start with the `Cr24` magic and be long enough. It must carry a proof whose public key derives the pinned id, the signed header's `crx_id` must match, and the RSA PKCS#1 v1.5 / SHA-256 signature over the CRX3 signed-data preamble, header and archive must verify. A tampered archive, a different publisher key, a `crx_id` mismatch, bad magic and a too-short file each stop the script with a specific error.
- **Integrity policy**: the CDM is installed when the signature verified **or** the server supplied a SHA-256. It is refused only when neither can be checked: no `cryptography` module and no hash. The hash-only path trusts the update server's response over TLS, so it is weaker than the signature check. ^[inferred]
- **Scope**: Linux only ("Linux-only"), x86-64 only. On arm64 the script stops with "not published for linux arm64" rather than fetching nothing useful.
- **Cache hit**: a target directory that already holds `manifest.json` short-circuits before any network call.
- **Extraction**: only `manifest.json` and `_platform_specific/linux_x64/libwidevinecdm.so` are written. Extra archive members, including path-traversal names, are ignored. A missing expected member fails, and an existing CDM directory is replaced atomically, with no stale files left behind.

`js/tests/widevine.test.ts` (vitest) pins the JS seeding rules described above:

- lookup order: an exclusive `CLOAKBROWSER_WIDEVINE_CDM`, then `WidevineCdm/` next to the binary, then the cache-root fallback for Pro binaries kept in their own directory;
- a set but invalid or empty env var resolves to nothing, without falling back and without scanning the working directory;
- the hint holds the symlink-resolved path;
- seeding is idempotent and rewrites a mismatched hint;
- `CLOAKBROWSER_WIDEVINE=0` disables it, it is a no-op off Linux, and an empty `userDataDir` is skipped;
- a write failure never throws.

## Trade-offs

- Seeding only writes a pointer; responsibility for obtaining and licensing the CDM stays with the user. ^[inferred]
- DRM playback (Netflix, Spotify Web) is a realism feature: a browser that reports Widevine but cannot play protected media may look inconsistent. ^[inferred] See [[projects/cloakbrowser/techniques/persistent-profile-warmup|Persistent profile warmup]] for the persistent-profile context.

## Open Questions

- The entrypoint says some bot detectors use a DRM/EME probe, but which services do so, and whether any service in the project's test matrix does, is not stated. ^[ambiguous]

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/cloakbrowser/widevine.py
- ~/code/CloakBrowser/bin/fetch-widevine.py, bin/docker-entrypoint.sh (commit b1a00ed, P2 pass)
- ~/code/CloakBrowser/README.md
- ~/code/CloakBrowser/dotnet/README.md
- ~/code/CloakBrowser/js/src/widevine.ts, js/src/playwright.ts, js/src/puppeteer.ts (commit b1a00ed, B5 batch)
- ~/code/CloakBrowser/tests/test_fetch_widevine.py, js/tests/widevine.test.ts (commit b1a00ed, B7 batch)
