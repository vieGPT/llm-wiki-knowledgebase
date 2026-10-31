---
title: >-
    Supply-chain verification
category: architecture
tags: [supply-chain, security, signing, release-management, chromium]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/binary-download-and-verification]]"
    type: related_to
  - target: "[[projects/cloakbrowser/architecture/release-channels-and-versioning]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/cloakbrowser/config.py
  - ~/code/CloakBrowser/cloakbrowser/download.py
  - ~/code/CloakBrowser/.github/workflows/attest-release.yml
  - ~/code/CloakBrowser/.github/workflows/publish.yml
  - ~/code/CloakBrowser/.github/workflows/ci.yml
  - ~/code/CloakBrowser/.github/dependabot.yml
  - ~/code/CloakBrowser/bin/fetch-widevine.py
  - ~/code/CloakBrowser/README.md
  - ~/code/CloakBrowser/CHANGELOG.md
  - ~/code/CloakBrowser/js/src/download.ts
  - ~/code/CloakBrowser/js/src/config.ts
  - ~/code/CloakBrowser/js/tests/signature.test.ts
  - ~/code/CloakBrowser/js/tests/update.test.ts
summary: >-
    Layered trust: pinned Ed25519 signature over versioned SHA256SUMS at every download, plus GPG tags, Sigstore attestations, Cosign-signed images and OIDC publishing.
provenance:
  extracted: 0.84
  inferred: 0.13
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:25+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Supply-chain verification

Because users auto-download a ~200 MB closed-source browser on first launch (and auto-update it in the background), the authenticity of that download is the project's main supply-chain risk. CloakBrowser answers with one mandatory runtime check, an Ed25519 signature over the release checksums verified by the wrapper, plus several optional, manual verification layers on the release side. Runtime mechanics are in [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]].

## Runtime: signed, version-bound checksum manifest (wrapper ≥ 0.4.0)

- Each binary release publishes `SHA256SUMS` and a detached `SHA256SUMS.sig` (base64 of a 64-byte raw Ed25519 signature over the exact manifest bytes).
- The wrapper pins a list of base64 32-byte public keys (`BINARY_SIGNING_PUBKEYS`, one key as of commit b1a00ed). A list allows key rotation: any pinned key that validates is accepted; unparsable keys are skipped; no match fails closed.
- Order matters: verify the signature **first**, then trust hashes from the manifest. Before 0.4.0 the checksum came from the same origin as the binary, which proved integrity but not authenticity; a compromised mirror could serve a matching tampered pair (issue #308, "M1").
- **Version binding**: the signed manifest carries a `version=<v>` line, and the wrapper rejects a manifest whose declared version differs from the one requested. A signature only proves "CloakHQ made this", so without binding a mirror could replay an older, genuinely signed release (forced downgrade). The `version=` line has no internal whitespace so older wrappers' parsers, which accept only `<hash>  <file>` lines, ignore it. ^[inferred]
- The check is non-bypassable on the official path (cloakbrowser.dev primary, GitHub Releases fallback) and on the Pro path (`/releases/pro/chromium-v<v>/`). `CLOAKBROWSER_SKIP_CHECKSUM` only affects a custom `CLOAKBROWSER_DOWNLOAD_URL`, where the pinned keys do not apply and the legacy same-origin checksum is used.
- Pinning a constant public key rather than per-version hashes keeps silent auto-update working: new releases verify without a wrapper update (CHANGELOG 0.4.0).
- Failure taxonomy: a bad signature, version mismatch or hash mismatch raises `BinaryVerificationError`, treated as a tampering signal that is always surfaced and never masked by falling back to a cached or free binary. A failed manifest *fetch* is transient and raises a plain error.
- Archives are extracted only after verification, which is why extraction uses tar's `fully_trusted` filter (Python 3.14's default `data` filter would mangle macOS `.app` symlinks).

## TypeScript port specifics

`js/src/download.ts` mirrors the Python verification path. These are the Node-specific details (commit b1a00ed):

- **Key import**: the pinned raw 32-byte Ed25519 key is imported as a JWK (`kty: OKP`, `crv: Ed25519`) and checked with `crypto.verify(null, ...)`. A key that cannot be parsed is skipped, and a signature that throws counts as "this key did not match", so the check fails closed.
- **Strict base64**: the `.sig` text is decoded and then re-encoded, and must equal the original exactly. Node's lenient decoder would otherwise accept a corrupted signature file as long as the remaining bytes happened to verify. This matches Python's `validate=True`.
- **Same-origin pair**: `SHA256SUMS` and `SHA256SUMS.sig` are always fetched from the same base URL (cloakbrowser.dev first, then GitHub), so the signature covers exactly the bytes it was served with.
- **Concurrent first runs**: extraction goes into a private `<dir>.partial-<uuid>` staging directory, which is renamed into place with retries (5 attempts for install races, plus 10 × 250 ms for transient Windows rename denials). Concurrent first launches cannot see or delete each other's half-written installs. The archive is unpacked with the `tar` npm package without further filtering, because it was verified before extraction.
- **Pro routing**: `verifyProDownload` wraps signature, version and checksum failures in `BinaryVerificationError`, which `ensureBinary` re-throws verbatim. A failed manifest fetch becomes a plain `Error`, reported as "Pro binary unavailable ... retry".

## Pinned by tests (B6, JS suite)

`js/tests/signature.test.ts` (commit b1a00ed) generates a throwaway Ed25519 key pair, pins its public key through a mocked `BINARY_SIGNING_PUBKEYS`, and checks the runtime design end to end:

- **Signature.** A valid signature is accepted. A manifest edited after signing, malformed base64 in the `.sig`, or a signature from a key that is not pinned all fail. With two pinned keys (old and new), a signature from either passes, which is the rotation case.
- **Version binding is mandatory.** A genuinely signed manifest that declares another version fails with "Version mismatch", and so does a manifest without any `version=` line. Older `parseChecksums` callers simply skip the `version=` line, which pins the backward-compatibility claim above.
- **Fail closed** on the official path: no fetchable signed manifest, or no entry for this platform's archive, rejects the download. Only a custom `CLOAKBROWSER_DOWNLOAD_URL` with `CLOAKBROWSER_SKIP_CHECKSUM` skips the check, and it then makes no signature request at all.
- **Pro path.** `CLOAKBROWSER_SKIP_CHECKSUM` does not bypass Pro verification. Hash, version and signature failures are `BinaryVerificationError`, the type the `ensureBinary` router branches on. A failed manifest fetch is a plain `Error`. The test comment says that plain error lets the router "fall back to free", which conflicts with the no-downgrade rule; see the note on [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]]. ^[ambiguous]
- **Same-origin pair with fallback.** `fetchSignedManifest` takes manifest and signature from cloakbrowser.dev and, when the primary `.sig` is missing, fetches both again from GitHub rather than mixing origins. It returns null when every source fails.

## Release-side layers (manual verification)

| Layer | What it covers | Source |
|---|---|---|
| GPG-signed tags | Binary release tags like `chromium-v146.0.7680.177.5` (key `C60C0DDC9D0DE2DD`) | README; workflow added in 0.2.0 |
| GitHub build-provenance attestation (Sigstore) | Release archives `cloakbrowser-*.tar.gz` / `*.zip`, via a manually dispatched `attest-release` workflow | attest-release.yml |
| Cosign keyless signature + attestation, SBOM, provenance | Docker image `cloakhq/cloakbrowser` by digest | publish.yml |
| OIDC trusted publishing | PyPI, npm (`--provenance`), NuGet, so no long-lived registry tokens | publish.yml |
| Version-consistency gate | Tag must equal Python, npm and .NET package versions before publishing | publish.yml |

The attestation workflow downloads already-published release assets and attests them; it does not build them. The attestation therefore vouches for "these bytes were attested by this repo's workflow", not for a reproducible build from source. ^[inferred]

## Repository hygiene

- GitHub Actions are pinned to commit SHAs (since 0.3.19), with Dependabot weekly grouped updates for actions, pip and npm.
- The `attest-release` workflow passes the `workflow_dispatch` tag through an env var instead of interpolating it into the shell, closing a shell-injection shape (0.3.29, #223). The Windows extraction path in the JS/.NET wrappers likewise passes paths to PowerShell via env vars (0.3.32).
- CI runs a real-binary job that fails if the binary download fails, so browser suites cannot silently pass by skipping.

## Adjacent: Widevine CDM fetch

`bin/fetch-widevine.py` applies the same "verify before install" idea to Google's Widevine CDM: SHA-256 from the component server over TLS, plus CRX3 RSA publisher-signature verification bound to the Widevine app id when `cryptography` is available. See [[projects/cloakbrowser/subsystems/widevine-cdm-seeding|Widevine CDM seeding]].

## Open Questions

- Key rotation procedure and where the private signing key lives are undocumented.
- Whether binary builds are reproducible from any public source is not stated; the patches are proprietary.

## Sources

- `~/code/CloakBrowser/cloakbrowser/config.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/cloakbrowser/download.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/.github/workflows/attest-release.yml` (commit `b1a00ed`)
- `~/code/CloakBrowser/.github/workflows/publish.yml` (commit `b1a00ed`)
- `~/code/CloakBrowser/.github/workflows/ci.yml` (commit `b1a00ed`)
- `~/code/CloakBrowser/.github/dependabot.yml` (commit `b1a00ed`)
- `~/code/CloakBrowser/bin/fetch-widevine.py` (commit `b1a00ed`)
- `~/code/CloakBrowser/README.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `~/code/CloakBrowser/js/src/download.ts`, `js/src/config.ts` (commit `b1a00ed`, B4 batch)
- `~/code/CloakBrowser/js/tests/signature.test.ts`, `js/tests/update.test.ts` (commit `b1a00ed`, B6 batch)
