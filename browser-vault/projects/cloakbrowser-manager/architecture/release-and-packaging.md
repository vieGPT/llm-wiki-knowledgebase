---
title: >-
    Release and packaging
category: architecture
tags: [release-management, packaging, macos, windows, docker]
aliases: [Manager installers, Manager release pipeline, build_macos, manager.spec]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: uses
  - target: "[[projects/cloakbrowser-manager/architecture/platform-modes]]"
    type: related_to
  - target: "[[projects/cloakbrowser/architecture/supply-chain-verification]]"
    type: related_to
sources:
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/manager.spec
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/build_macos.sh
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/make_universal_app.sh
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/sign_macos_app.sh
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/entitlements.plist
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/build_windows.ps1
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/installer.iss
  - repo:github.com/vieGPT/CloakBrowser-Manager/packaging/requirements-build.txt
  - repo:github.com/vieGPT/CloakBrowser-Manager/run.py
  - repo:github.com/vieGPT/CloakBrowser-Manager/run-macos.sh
  - repo:github.com/vieGPT/CloakBrowser-Manager/run-windows.bat
  - repo:github.com/vieGPT/CloakBrowser-Manager/Dockerfile
  - repo:github.com/vieGPT/CloakBrowser-Manager/LICENSE
  - repo:github.com/vieGPT/CloakBrowser-Manager/.github/workflows/release.yml
  - repo:github.com/vieGPT/CloakBrowser-Manager/CHANGELOG.md
summary: >-
    How the Manager ships: PyInstaller onedir wrapped as a Windows Inno Setup installer and a lipo-fused universal macOS .dmg, a multi-arch Docker image, and a run-from-source path.
provenance:
  extracted: 0.78
  inferred: 0.15
  ambiguous: 0.07
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T06:19:30+01:00
updated: 2026-10-31T06:19:30+01:00
---

# Release and packaging

CloakBrowser-Manager ships in three forms (as of 14/10/2026, commit `11e0043`, v0.1.8): a per-user Windows x64 installer (Inno Setup), one universal macOS `.dmg` (Apple Silicon + Intel), and one multi-architecture Linux Docker image (amd64 + arm64). The two native builds share one PyInstaller spec and differ only in how they are wrapped and signed; the Docker image is a separate path that serves the UI over KasmVNC (see [[projects/cloakbrowser-manager/architecture/platform-modes|Platform modes]]). A run-from-source launcher (`run.py`) gives developers the same code path as the frozen app. None of the installers carries the stealth Chromium itself: the packaged CloakBrowser wrapper fetches it on first launch through [[projects/cloakbrowser/subsystems/binary-download-and-verification|binary download and verification]].

## Shared native build pipeline

Both `packaging/build_macos.sh` and `packaging/build_windows.ps1` run the same four steps:

1. **Version baking.** The version comes from `git describe --tags --always` (Windows also accepts `CB_VERSION`, trimmed because `cmd`'s `set VAR=x & ...` leaves a trailing space). It is written to `backend/version.txt`, which the spec ships as data so the frozen app can log its own build without a `.git` directory. A failing `git` never aborts the Windows build; it falls back to `dev`.
2. **Frontend build.** `npm ci` then `npm run build` (`tsc -b && vite build`) in `frontend/`; the React SPA is embedded as data.
3. **Clean build venv.** A dedicated `.venv-build` (override with `PACKAGING_VENV`) gets `backend/requirements.txt` plus `packaging/requirements-build.txt`, which holds only `pyinstaller>=6.10` and is explicitly build-time, never shipped.
4. **Freeze** with `packaging/manager.spec` into `dist_native/` (onedir), then wrap.

### What `manager.spec` bundles

- Entry point `app_entry.py` (in-process uvicorn plus a pywebview window), not `backend.main:app`, because a frozen bundle cannot resolve that import string.
- `frontend/dist` as data (the spec refuses to run if `index.html` is missing), plus `backend/version.txt`.
- All `backend` submodules (main.py lazy-imports CloakBrowser modules), and `collect_all` for `playwright`, `cloakbrowser`, `uvicorn` and `webview`.
- Hidden imports for uvicorn's lazily loaded protocol/loop backends and for the CloakBrowser GeoIP extra (`geoip2`, `maxminddb`, `socksio`); `uvloop` is excluded on Windows (POSIX-only).
- Playwright's Node driver is shipped (CloakBrowser drives the browser over CDP), but Playwright's downloaded browsers are not.
- `console=False`: no terminal window; logs go to a file in the data directory.
- macOS `BUNDLE`: bundle id `dev.cloakbrowser.manager`, `LSMultipleInstancesProhibited` so relaunching focuses the running instance instead of hitting the busy port 8080 and opening a stray tab, high-resolution capable, category "utilities".

## macOS: build, sign, fuse

**Build** (`build_macos.sh`): freeze, then `.app`, then a drag-to-Applications `.dmg` (stale mounted volumes with the same name are detached first, since they make `hdiutil` fail). Signing, notarization and stapling run only when `CB_SIGN_IDENTITY` and `CB_NOTARY_PROFILE` are set; otherwise the build stays unsigned.

**The cryptography/OpenSSL pin.** The build force-installs the `universal2` `cryptography` wheel (binary only, pinned range `>=44,<49`). The newest cryptography has no single-arch macOS wheel, so on Intel pip source-built it against Homebrew's `openssl@3`, PyInstaller bundled that `libssl`, and the frozen app crashed at launch with a missing OpenSSL symbol. The universal2 wheel links OpenSSL statically, so both arches get an identical extension and the later lipo merge is a no-op for it. A guard step inspects the frozen `_rust.abi3.so` with `otool` and fails the build if it links an external `libssl`. This matches the v0.1.3 fix ("macOS app failed to launch profiles on a fresh machine", 19/08/2026).

**Inside-out signing.** Components are signed explicitly because `codesign --deep` with `--timestamp` fails on subcomponents:

- `.so`/`.dylib` files: hardened runtime, no entitlements (they load into a host process that carries them).
- Every other Mach-O executable: hardened runtime plus entitlements. Playwright's Node driver runs as its own process, so V8's JIT needs `allow-jit` on Node's own signature or it dies at V8 init with ENOMEM.
- Finally the bundle, with entitlements on the main executable, then `codesign --verify --strict`.

`packaging/entitlements.plist` grants `allow-jit`, `allow-unsigned-executable-memory` (CPython/PyInstaller need writable+executable memory) and `disable-library-validation` (third-party `.so`/`.dylib` are not individually signed by their vendors). Disabling library validation weakens what the hardened runtime protects against, the usual price of freezing Python apps. ^[inferred]

`packaging/sign_macos_app.sh` duplicates the signing block on purpose: `build_macos.sh` is the "proven manual-release path" and stays frozen, so CI changes cannot destabilise it. The two must be kept in sync by hand. It adds an optional `KEYCHAIN`.

**Universal app** (`make_universal_app.sh`). PyInstaller's `--target-arch universal2` is not used because it requires every embedded binary (Playwright's Node driver, single-arch wheels) to already be universal2. Instead each arch is built natively on its own runner and the two `.app` trees are fused:

- Base is a copy of the arm64 tree; every Intel-only path is unioned in.
- Each Mach-O present in both trees is `lipo`-joined; files with identical arch sets on both sides are kept as-is.
- Safety gate: a shared Mach-O that ends up thin is a hard error; a single-arch extra that exists on one side only (for example an x86_64-only OpenSSL legacy provider) is a warning, because the other arch's native build proved it is not needed.

## Windows: installer

`build_windows.ps1` needs Python 3.10+, Node 18+ and Inno Setup 6 (`ISCC.exe` on PATH or in the default install location) and passes `AppVersion` and `SourceDir` to `packaging/installer.iss`. The installer:

- installs per user (`PrivilegesRequired=lowest`, under `{localappdata}\Programs`), so there is no admin prompt and no code-signing certificate is needed;
- uses LZMA2 solid compression, a modern wizard, an optional desktop shortcut and a post-install "Launch" option;
- keeps a fixed `AppId`, which lets later installers upgrade in place. ^[inferred] The `AppId` value contains non-hex characters, so it is not a well-formed GUID; Inno Setup treats it as an opaque string. ^[ambiguous]

The build is unsigned, so Windows users click through SmartScreen.

## Release CI (`release.yml`)

A pushed `v*` tag cuts a release; `workflow_dispatch` builds everything without publishing (a check before tagging). Python 3.12 and Node 20; all actions are pinned to commit SHAs.

| Job | What it does |
|---|---|
| `version` | Validates the tag as canonical `vMAJOR.MINOR.PATCH`, else uses `git describe` |
| `build-windows` | Installs Inno Setup, runs `build_windows.ps1` |
| `build-mac` | Matrix `macos-15` (arm64) and `macos-15-intel` (x86_64); unsigned `.app`, arch sanity check, tarred |
| `universal` | `make_universal_app.sh`, ad-hoc re-sign, `-universal.dmg` |
| `build-docker` / `docker-manifest` | Native amd64 and arm64 runners push by digest; digests and provenance are validated before one multi-arch manifest is created and re-verified |
| `promote-docker-latest` | Moves `latest` only to the newest complete, verified stable release |
| `release` | `SHA256SUMS` over the `.dmg` and `-setup.exe`, notes sliced from this version's `CHANGELOG.md` section, `gh release create` |

**Why ad-hoc signing.** The `universal` job ships without a Developer ID and without notarization "to keep a personal certificate name off the public binary", consistent with the ad-hoc-signed Chromium engine. Lipo-merging invalidates the per-arch signatures and arm64 refuses to launch unsigned code, so the app is re-signed with identity `-`. Release notes therefore tell macOS users to clear the quarantine attribute or use "Open Anyway", and Windows users to bypass SmartScreen; they also say the engine downloads on first launch. A move to a CloakHQ organisation Developer ID is planned.

The release publishes a plain `SHA256SUMS` with no signature over it. CloakBrowser's own binaries carry a pinned Ed25519 signature over their checksum file ([[projects/cloakbrowser/architecture/supply-chain-verification|supply-chain verification]]), so the Manager installers are the weaker link in the chain: they guard against corruption, not against tampering. ^[inferred]

## Docker image

The `Dockerfile` has two stages. `node:20-slim` builds the frontend, using `npm install` rather than the `npm ci` of the native builds, so lockfile drift is not caught there. ^[inferred] The `python:3.12-slim` runtime adds Chromium system libraries, Playwright's Chromium system deps (the `playwright` pip package is installed only to run `install-deps`, with failures ignored, then uninstalled), Microsoft core fonts (EULA preseeded) and KasmVNC 1.3.3 for the build's `TARGETARCH`. It also pre-downloads the CloakBrowser binary at build time with `ensure_binary`. It exposes 8080, declares `VOLUME /data`, health-checks the unauthenticated `/api/health` every 30 s, and starts through `entrypoint.sh`. The Windows core fonts keep a Windows persona's font set plausible on a Linux host ([[projects/cloakbrowser/detection-vectors/font-fingerprinting|font fingerprinting]]). ^[inferred]

## Run from source

`run.py` (wrapped by `run-macos.sh` and `run-windows.bat`, which prefers the `py -3` launcher and pauses on error) supports only Windows and macOS and tells Linux users to use Docker. It requires Python 3.10+ and Node 18+. It:

- creates `.venv` and records SHA-256 hashes of `backend/requirements.txt` and `frontend/package-lock.json` in `.venv/.manager-setup.json`, reinstalling only when they change;
- rebuilds the frontend when any file under `frontend/src` is newer than `frontend/dist/index.html`;
- refuses to start if 127.0.0.1:8080 is already bound;
- sets `CLOAKBROWSER_MANAGER_RUNTIME=native` and replaces itself (`os.execve`) with `app_entry.py`, so a dev run is the same in-process server and native window as the frozen build. `CLOAKBROWSER_MANAGER_UI=browser` forces a browser tab instead.

## Licensing

The Manager code is MIT, copyright 2026 CloakHQ (`LICENSE`). The Chromium binary it downloads is licensed separately (`BINARY-LICENSE.md`), mirroring CloakBrowser's [[projects/cloakbrowser/architecture/wrapper-binary-split|wrapper/binary split]]. ^[inferred]

## Packaging history (from CHANGELOG)

| Version | Date | Packaging-relevant change |
|---|---|---|
| 0.1.0 | 19/08/2026 | First native macOS/Windows installers (unsigned), standalone webview window, single instance |
| 0.1.2 | 19/08/2026 | Console-less Windows app no longer aborts on startup output |
| 0.1.3 | 19/08/2026 | macOS bundles a self-contained TLS library (the cryptography pin) |
| 0.1.4 | 21/08/2026 | Automated multi-arch Docker releases; digest/provenance validation; canonical semver tags; pinned workflow deps |
| 0.1.5 | 30/08/2026 | Windows legacy-codepage launch failure fixed through engine 0.5.10 |
| 0.1.6 to 0.1.8 | 08/10 to 14/10/2026 | Bundled CloakBrowser wrapper 0.6.0, 0.6.1, 0.6.3 |

## Contradictions and stale text

- `release.yml`'s header and the `universal` job comment say the macOS `.dmg` is "signed + notarized"; the step itself ad-hoc signs without notarization, and the generated release notes say "not notarized yet". The step reflects actual behaviour. ^[ambiguous]
- `sign_macos_app.sh` says it is "used by the GitHub Actions release workflow", but `release.yml` (commit `11e0043`) never calls it. ^[ambiguous]
- CHANGELOG 0.1.0 says installers "bundle the Manager and stealth Chromium binary". The spec and the release notes say the Chromium binary downloads on first launch; "bundles CloakBrowser 0.6.x" in later entries refers to the wrapper package (commit message: "bundle wrapper 0.6.3"). ^[ambiguous]
- `app_entry.py`'s docstring says `run.py` shells out to `uvicorn backend.main:app`; `run.py` now execs `app_entry.py`. The docstring is stale. ^[inferred]

## Open Questions

- Has the planned CloakHQ Developer ID signing and notarization landed after v0.1.8?
- Does the Docker image get signed (Cosign) or attested like CloakBrowser's images? `release.yml` validates provenance, but no signing step was checked here.
- The icon sources (`packaging/icon.png`, `icon-macos.png`) and their derived `icon.icns`/`icon.ico`: is the conversion step documented anywhere?

## Sources

Commit `11e0043` of `repo:github.com/vieGPT/CloakBrowser-Manager`:

- packaging/manager.spec, packaging/build_macos.sh, packaging/make_universal_app.sh, packaging/sign_macos_app.sh, packaging/entitlements.plist
- packaging/build_windows.ps1, packaging/installer.iss, packaging/requirements-build.txt
- run.py, run-macos.sh, run-windows.bat, Dockerfile, LICENSE
- .github/workflows/release.yml, CHANGELOG.md (consulted)
