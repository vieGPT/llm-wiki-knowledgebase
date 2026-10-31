---
title: >-
    Profile seed persistence
category: subsystems
tags: [fingerprinting, persistent-profile, identity, concurrency]
aliases: [profile-seed.ts, .cloakbrowser-seed, persistentSeedArgs]
relationships:
  - target: "[[projects/cloakbrowser/techniques/fingerprint-seed-identity]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/persistent-profile-warmup]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/profile-database]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/js/src/profile-seed.ts
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py
  - repo:github.com/CloakHQ/CloakBrowser/bin/cloakserve
  - repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/profile-seed.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/puppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_profile_seed.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Config.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/ProfileSeedTests.cs
summary: >-
    A persistent profile stores its fingerprint seed in .cloakbrowser-seed inside userDataDir, so the same profile presents the same device identity on every launch.
provenance:
  extracted: 0.85
  inferred: 0.15
  ambiguous: 0.0
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T06:58:25+01:00
---

# Profile seed persistence

Without flags the wrapper passes a random `--fingerprint=<seed>` on every launch, so each session looks like a new device. For persistent profiles that is incoherent: cookies and history say "returning visitor" while the fingerprint says "new machine". ^[inferred] `persistentSeedArgs` (`js/src/profile-seed.ts`, mirroring Python `cloakbrowser/config.py::persistent_seed_args`) fixes this by storing the seed in `<userDataDir>/.cloakbrowser-seed` on first launch and reusing it afterwards. The seed is the single source of the coherent device package described in [[projects/cloakbrowser/techniques/fingerprint-seed-identity|Fingerprint seed identity]].

## Behaviour

- **File format**: decimal seed plus newline, the same file in every wrapper (Python, JS, .NET), so a profile can move between wrappers. A valid value must be all digits and inside `[SEED_MIN, SEED_MAX]` (the README's default range is 10000-99999).
- **Precedence**: the stored seed is prepended as a *user* arg, so it overrides the random stealth default through `buildArgs` dedup (see [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]).
- **Opt-outs**: an explicit `--fingerprint=...` in `args` (including `--fingerprint=off`) or `stealthArgs: false` leaves the file untouched.
- **Scope**: only `launchPersistentContext` (Playwright and Puppeteer) passes `userDataDir`; plain `launch()` and `launchContext()` stay random.
- **Rotation**: delete `.cloakbrowser-seed`; the next launch picks a new seed (README).
- **Corruption**: a corrupt file is removed with a loud warning and replaced, which means a new identity. The warning exists because silently changing identity on a logged-in profile is costly. ^[inferred]

## Concurrency design

Two launches of the same fresh profile must not end up with different seeds:

1. Write the new seed to a unique temp file (`.cloakbrowser-seed.<uuid>.tmp`), exclusive create, `fsync`.
2. Publish with a hard link (`link(tmp, file)`), which never replaces an existing file. Concurrent first launches all converge on whichever seed was linked first.
3. On filesystems without hard links (FAT/exFAT, some network mounts), fall back to an exclusive create of the final file, which also never overwrites.
4. Re-read the file after publishing, so every racer uses the winner's value.

- The fallback path opens a short window where a reader sees an empty file. Readers therefore treat an empty file as "being written" and poll it for about one second (20 × 50 ms synchronous sleeps) before calling it corrupt.
- The temp file is always unlinked; failure to remove it only warns.

## Python implementation (`persistent_seed_args`, P2 pass)

`cloakbrowser/config.py` holds the Python original that the JS module mirrors. Both persistent launchers in `browser.py` (sync and async) call it after GeoIP and WebRTC arguments are resolved and before `build_args`. Details visible only in the Python code:

- **Explicit-seed test is exact.** An argument counts as an explicit seed only if the part before `=` is exactly `--fingerprint`, so `--fingerprint-platform=…` and other `--fingerprint-*` flags do not disable the stored seed.
- **Byte-identical writes.** The seed is written as ASCII with `\n` line endings forced, so a Windows host produces the same bytes as the JS and .NET wrappers, and every write is flushed and `fsync`ed.
- **Validation guards against Unicode digits.** A stored value must be digits, ASCII and inside 10000 to 99999. The ASCII check matters because Python's digit test also accepts non-Latin digits. ^[inferred]
- **Temp file.** The temp file has a UUID name and is written in plain write mode; the no-overwrite guarantee comes from the publish step (`os.link`, or exclusive create in the fallback), not from the temp write.
- **Hard-link publishing.** `os.link(tmp, path)` raising `FileExistsError` means another launch won and is ignored. Any other `OSError` (no hard-link support) logs at debug level and falls back to exclusive create of the final file. A code note admits the fallback's weakness: a concurrent reader can catch the file empty and regenerate, a window that `link()` avoids, hence the one-second empty-file poll.
- **Fail loudly instead of looping.** If the re-read after publishing still finds no valid seed (only possible through a concurrent corrupt-file race), the launch raises `RuntimeError("Could not create fingerprint seed file …")`. Removing a corrupt file tolerates another launch having removed it first.

### cloakserve uses a different identity store

The Docker CDP server does not call `persistent_seed_args`. Its seed comes from the `?fingerprint=` query parameter, its profile directory is `<data_dir>/<seed>`, and that directory is deleted on idle reaping and, at this commit, on close. There is no `.cloakbrowser-seed` file: the seed is the directory name. ^[inferred] See [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]].

## Pinned by tests (B6, JS suite)

`js/tests/profile-seed.test.ts` (commit b1a00ed) confirms the behaviour above and narrows a few points:

- **Five-digit seeds.** A freshly written file always matches exactly five digits plus a newline. A file written by the Python wrapper (`12345\n`) is read as is, which pins the cross-wrapper format.
- **What counts as corrupt**: an empty file (after the read poll), non-digits, too short (`5`), too long (`123456`) or negative (`-12345`). Each is replaced, and the warning names the file path.
- **Opt-outs leave no trace.** An explicit `--fingerprint=4242` or `--fingerprint=off` wins and neither creates nor rewrites the file. `stealthArgs: false` writes nothing, and an empty `userDataDir` is ignored. A missing profile directory is created on demand.
- **Hard-link fallback is real.** When `link()` fails with `ENOTSUP` the seed is still published once and reused, and the directory holds only the seed file, with no leftover temp file.
- **Cross-process race.** Eight separate Node processes launching the same fresh profile all print the same seed and leave a single file. The race test runs the built `dist/` because in-process calls cannot interleave.
- **Scope.** Two Playwright `launchPersistentContext` calls reuse one seed, while five plain `launch()` or `launchContext()` calls produce more than one seed and write nothing to the working directory. Puppeteer's `launchPersistentContext` reads the stored seed, and its `launch()` shares the argument resolver but stays random.

## Pinned by Python tests (B8 batch)

`tests/test_profile_seed.py` confirms that the Python `persistent_seed_args` behaves like the JS module described above:

- The first persistent launch writes the seed and the second reuses it, in both the sync and async launchers.
- An explicit `--fingerprint=4242` or `--fingerprint=off` wins, and the seed file is neither created nor changed. `stealth_args=False` writes nothing at all. Plain `launch()` stays random across launches and writes no file.
- Corrupt content (empty, non-numeric, `5`, `123456`, negative, non-UTF-8 bytes) logs a warning naming the file path and is replaced with a fresh seed in 10000 to 99999.
- The file format (ASCII digits plus newline) is pinned "cross-wrapper": the docstring says the JS and .NET tests read this exact content. The stored seed is placed before the caller's other args.
- An empty `user_data_dir` is ignored.
- Sixteen threads racing on a fresh profile all get the same seed and leave only the seed file behind. When `link()` fails with `ENOTSUP`, the exclusive-create fallback gives the same stable result.

## .NET implementation (`Config.PersistentSeedArgs`, batch D)

The C# port in `dotnet/src/CloakBrowser/Config.cs` follows the same algorithm: exact `--fingerprint` key test, empty-file poll (20 × 50 ms), loud corrupt-file warning, and an `IOException` when the re-read still finds no valid seed (commit b1a00ed). Its publish step differs by platform:

- **Unix: P/Invoked libc `link()`.** The code comment explains why: `File.Move(overwrite: false)` checks and then renames, "so concurrent callers can replace each other (seen on macOS)". `link()` fails atomically with `EEXIST` (17 on both Linux and macOS). Any other errno falls back to an exclusive `FileMode.CreateNew` write, flushed to disk.
- **Windows: no-replace `File.Move`.** It relies on `MoveFileEx` without `REPLACE_EXISTING`, which is atomic.
- **Bytes.** The seed is written as ASCII and read as Latin-1, and only ASCII digits count, so the stored file is byte-compatible with the Python and JS wrappers.
- `ProfileSeedTests` mirrors `test_profile_seed.py` and `profile-seed.test.ts`. It includes `Reads_PythonWritten_File`, concurrent first launches agreeing on one seed, and a call-site check through the real `LaunchPersistentContextAsync` with a fake `chrome` that records its argv. See [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

How the seed affects incognito-style checks is on [[projects/cloakbrowser/detection-vectors/incognito-storage-quota-detection|Incognito and storage quota detection]].

## Related

- Persistent profiles as an evasion: [[projects/cloakbrowser/techniques/persistent-profile-warmup|Persistent profile warm-up]].
- The README tip "use a fixed seed when revisiting the same site" (reCAPTCHA v3 Enterprise and similar scoring) is the manual version of what this file automates. ^[inferred]
- [[projects/cloakbrowser-manager/subsystems/profile-database|Profile database (Manager)]] — CloakBrowser-Manager keeps the seed in its SQLite row and always passes it explicitly.

## Sources

- repo:github.com/CloakHQ/CloakBrowser/js/src/profile-seed.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/config.py, cloakbrowser/browser.py (commit b1a00ed, P2 pass)
- repo:github.com/CloakHQ/CloakBrowser/bin/cloakserve (commit b1a00ed, P2 pass)
- repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/README.md (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/tests/profile-seed.test.ts, js/tests/launch.test.ts, js/tests/puppeteer.test.ts (commit b1a00ed, B6 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_profile_seed.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/Config.cs, dotnet/tests/CloakBrowser.Tests/ProfileSeedTests.cs (commit b1a00ed, batch D)
