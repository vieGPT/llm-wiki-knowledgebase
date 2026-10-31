---
title: >-
    Font fingerprinting
category: detection-vectors
tags: [fonts, fingerprinting, persona, linux, detection]
aliases: [font enumeration, emoji canvas hash, Windows font tells]
relationships:
  - target: "[[projects/cloakbrowser/techniques/platform-persona-spoofing]]"
    type: related_to
  - target: "[[projects/cloakbrowser/detection-vectors/canvas-fingerprinting]]"
    type: related_to
  - target: "[[projects/cloakbrowser-manager/subsystems/diagnostics]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/js/src/fonts.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/args.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/cli.ts
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__main__.py
  - repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_font_warning.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_cli.py
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/fonts.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/Dockerfile
summary: >-
    Detectors enumerate installed fonts and hash emoji/text renders; a Linux host spoofing Windows or macOS leaks its real font set unless the persona's fonts are installed.
provenance:
  extracted: 0.77
  inferred: 0.19
  ambiguous: 0.04
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:46:33+01:00
updated: 2026-10-31T06:53:05+01:00
---

# Font fingerprinting

Fonts are one of the few fingerprint surfaces that the patched binary cannot fully fake, because glyphs are drawn from font files actually installed on the host. Detectors exploit this in two ways:

- **Enumeration and metrics**: probe whether named fonts exist and measure text widths. FingerprintJS and CreepJS are named in the README.
- **Rendered-pixel hashes**: draw emoji or CJK text on a hidden canvas and hash the pixels. Kasada and Akamai are named in the README.

A Linux host that claims to be Windows (the default persona, see [[projects/cloakbrowser/techniques/platform-persona-spoofing|Platform persona spoofing]]) but has no Windows fonts contradicts its own claim. Font fingerprinting tools flag that mismatch.

## What the detector measures

- **Font presence**: whether OS-specific families such as Segoe UI, Calibri or Marlett resolve, and what the generic `monospace` family maps to.
- **Font metrics**: glyph widths and heights for the platform's fonts. The binary's `--fingerprint-windows-font-metrics` flag aligns metrics with Windows when spoofing Windows on Linux. It needs a Chromium 148+ binary and installed Windows fonts. On `154.0.8037.57.2` and newer it is on by default once those fonts exist (README, as of 26/10/2026).
- **Emoji/CJK canvas hash**: minimal Linux images (Docker, cloud VMs) lack color-emoji and CJK fonts, so their hashes "don't match any real browser". This overlaps with [[projects/cloakbrowser/detection-vectors/canvas-fingerprinting|Canvas fingerprinting]].

## Tell-tale font sets (wrapper constants, `fonts.ts`)

| Set | Fonts | Status |
|---|---|---|
| Windows OS (8) | Segoe UI, Segoe UI Light, Calibri, Marlett, MS UI Gothic, Franklin Gothic, Consolas, Courier New | Required for the Windows persona. Consolas and Courier New are included so `monospace` resolves to a Windows font (issue #395) |
| MS Office (10) | MT Extra, Century, Century Gothic, MS Reference Specialty, Wingdings 2/3, Book Antiqua, Bookshelf Symbol 7, Monotype Corsiva, Bookman Old Style | Informational only. About half of real Windows machines have Office fonts, so missing them is a normal Windows setup |
| macOS core (20) | e.g. Apple Color Emoji, Helvetica, Helvetica Neue, Menlo, Monaco, Gill Sans | Required for the macOS persona. Several of these also ship on Windows, so a partial count off-Mac is normal |

## How CloakBrowser responds

- **Runtime warning**: on Linux, when the final argv carries `--fingerprint-platform=windows` and the full Windows set is not installed (checked with `fc-list`), the wrapper warns once per environment. A `.font_warning_shown` marker in the cache dir keeps it quiet afterwards, and `CLOAKBROWSER_SUPPRESS_FONT_WARNING` silences it entirely. The check is strict, all eight fonts or warn, because "real font installs are atomic". An undeterminable result (`fc-list` missing) counts as unknown, never as zero.
- **Diagnostics**: `cloakbrowser info` reports the raw N/total count for each of the three sets on Linux hosts. `info` has no persona flag, so users read the count for whichever persona they spoof (see [[projects/cloakbrowser/subsystems/cli|CLI]]).
- **Setup guidance** (README): install baseline emoji and CJK packages, then copy real Windows fonts from `C:\Windows\Fonts\`. The proprietary Microsoft fonts are not in package repos, and `ttf-mscorefonts-installer` only has XP-era fonts. Install them system-wide in Docker images. A persona without its fonts is called "a common cause of blocks".
- **Docker image baseline** (root `Dockerfile`, commit b1a00ed): the official image installs Liberation, Noto Color Emoji, Unifont, FreeFont, IPA Gothic, WenQuanYi Zen Hei, TLWG Loma and URW base35. That covers emoji, CJK, Thai and metric-compatible Latin fallbacks, but none of the eight Windows tell fonts, which are proprietary. A stock container running the default Windows persona therefore fails the strict check and should print the one-time warning unless Windows fonts are added. ^[inferred] See [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]] for the image layers.
- **Feature parity flag**: when the font-metrics flag is present, the launcher also re-enables `MediaRouter`. Playwright disables it and stock Chrome ships it on (see [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]).

## Python wrapper parity and detector mapping

- The Python wrapper (`browser.py`) carries the same three tell sets and the same strict all-eight rule; `cloakbrowser info` (`__main__.py`) counts all three unconditionally on Linux. The warning goes straight to stderr in ASCII, so an app's logging config cannot silence it and a cp1252 Windows console cannot crash on it.
- The README's FingerprintJS troubleshooting table maps a **"Browser tampering (fonts)"** verdict to font metrics that do not match the spoofed Windows platform, fixed by `--fingerprint-windows-font-metrics` plus installed Windows fonts. This is separate from the noise-driven tampering verdict (see [[projects/cloakbrowser/detection-vectors/fingerprint-tampering-detection|Fingerprint tampering detection]]).
- For Kasada/Akamai, the README calls missing emoji/CJK fonts "the most common cause of blocks on aggressive sites" once proxy, geoip and headed mode are already correct.

### Pinned by tests (B7)

`tests/test_font_warning.py` mocks the platform, `fc-list` and the cache dir (commit b1a00ed). It pins the warning rules described above:

- the warning fires only on Linux, and only when the argv says `--fingerprint-platform=windows`;
- one of eight tell fonts present still warns (the strict rule), while the full set stays silent;
- a missing `fc-list` gives no warning, no crash and no marker;
- `CLOAKBROWSER_SUPPRESS_FONT_WARNING` silences the warning **without** writing the marker;
- an in-process flag runs the probe once per process, and a marker left by an earlier process suppresses the warning across processes;
- the message ("Incomplete Windows font set") goes to stderr.

`tests/test_cli.py` pins the 20-font macOS set and `info`'s conditional nudge on a partial count. See [[projects/cloakbrowser/subsystems/cli|CLI]].

### Pinned by tests (B6, JS suite)

`js/tests/fonts.test.ts` (commit b1a00ed) confirms the JS warning rules described above:

- `fc-list` runs at most once per process, and a marker that already exists suppresses the warning.
- `CLOAKBROWSER_SUPPRESS_FONT_WARNING` suppresses it without writing a marker.
- There is no warning on non-Linux hosts or when the final argv overrides the persona to a non-Windows platform.
- A missing `fc-list` neither warns, crashes nor writes a marker, so the check can run again later.
- One Windows tell out of eight still warns; only the complete set is silent.
- `MACOS_FONT_TELLS` holds 20 fonts, including Helvetica Neue, and `countFontsPresent` counts them from `fc-list` output.

## History (CHANGELOG, dated)

- 2026-03-02 (0.3.0): binary flag `--fingerprint-fonts-dir` added.
- 2026-03-05 (0.3.9): binary "font auto-hide for cross-platform fingerprints".
- 2026-04-28 (0.3.26): Docker image gains emoji and extended font packages to resolve Kasada/Akamai canvas blocks (#179); README "Font Setup on Linux" added.
- 2026-06-29 (0.4.5): one-time Linux warning when spoofing Windows without Windows fonts.
- 2026-07-09 (0.4.9): strict 8-font Windows set (monospace fonts added) and informational 10-font Office group.
- 2026-08-25 (0.5.9): Docker image declares the URW base35 fonts explicitly so the Latin fallback for Verdana, Georgia, Trebuchet MS and Tahoma cannot be silently dropped.
- 2026-09-24 (0.5.11): `info` reports macOS persona font completeness.
- 2026-10-08 (0.6.0): Linux Preview `154.0.8037.57.2` matches Windows system and code fonts by default.

## Why the binary cannot just fake it

The seed can perturb how fonts are measured or reported, but a canvas hash of rendered glyphs reflects the real font files. Fonts that are absent cannot be drawn convincingly, so the fix has to happen at the host level. ^[inferred]

## Open Questions

- How the binary's font patches (listed among the 87 patch areas) interact with the host font set beyond the metrics flag is not documented in the wrapper.
- `docs/chrome40-fpjs-font-minimum-set-investigation.md` is referenced by `fonts.ts` but was not ingested in this batch.

## Related

- [[projects/cloakbrowser-manager/subsystems/diagnostics|Diagnostics (Manager)]] — CloakBrowser-Manager reports Windows persona font coverage in its status badge on Linux Docker.

## Sources

- `repo:github.com/CloakHQ/CloakBrowser` at commit `b1a00ed`: `js/src/fonts.ts`, `js/src/args.ts`, `js/src/cli.ts`, `README.md` (section "Font Setup on Linux", flag table)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/Dockerfile` (commit `b1a00ed`, P2 pass)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__main__.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/tests/test_font_warning.py`, `tests/test_cli.py` (commit `b1a00ed`, B7 batch)
- `repo:github.com/CloakHQ/CloakBrowser/js/tests/fonts.test.ts` (commit `b1a00ed`, B6 batch)
