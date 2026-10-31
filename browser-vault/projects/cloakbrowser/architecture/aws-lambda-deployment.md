---
title: >-
    AWS Lambda deployment
category: architecture
tags: [aws-lambda, docker, deployment, scraping, ssrf]
aliases: [CloakBrowser on Lambda, lambda_handler.py]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/cloakserve-cdp-server]]"
    type: related_to
  - target: "[[projects/cloakbrowser/architecture/framework-integration-modes]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/Dockerfile
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/INSTRUCTIONS.md
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/lambda-entrypoint.sh
  - repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/lambda_handler.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_lambda_security.py
summary: >-
    A container image layered on cloakhq/cloakbrowser that runs one-shot headed scrapes in AWS Lambda via Xvfb, with a dual-mode entrypoint, retry strategies and SSRF-guarded URLs.
provenance:
  extracted: 0.82
  inferred: 0.15
  ambiguous: 0.03
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:46:02+01:00
updated: 2026-10-31T06:10:53+01:00
---

# AWS Lambda deployment

`examples/integrations/aws_lambda/` packages CloakBrowser as an AWS Lambda container image for one-shot scrapes. It derives from the official `cloakhq/cloakbrowser` Docker Hub image and adds a thin Lambda layer, so Lambda is an extra invocation surface: the image still runs `python`, `cloakserve`, `cloaktest`, `node` and `bash` like the canonical image. The directory is self-contained and does not prescribe a deployment tool. Compared with the long-running CDP server ([[projects/cloakbrowser/subsystems/cloakserve-cdp-server|CloakServe CDP server]]), each invocation launches, scrapes and closes its own browser in-process. See also [[projects/cloakbrowser/architecture/framework-integration-modes|Framework integration modes]].

## Image layers (Dockerfile, P2 pass)

`examples/integrations/aws_lambda/Dockerfile` (commit b1a00ed) adds a thin layer on top of the canonical image described in [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]]:

- **Base**: `FROM cloakhq/cloakbrowser:latest`. A header comment recommends pinning a tag (it cites `0.3.25` as an example) for reproducible builds, since `latest` floats with CloakHQ's releases. It also argues that pulling the official image is an official distribution channel under the Binary License, not redistribution.
- **Runtime glue**: `pip install awslambdaric`; the AWS Runtime Interface Emulator added from the emulator's GitHub `releases/latest` URL for the buildx `TARGETARCH`, so local `docker run` invocations hit the standard endpoint without mounting anything. That download is unpinned and has no checksum, unlike the binary itself. ^[inferred]
- **Entrypoint and handler**: `lambda-entrypoint.sh` replaces `/entrypoint.sh`; `lambda_handler.py` lands in `/app`, already the working directory and on Python's import path. Default command: `lambda_handler.handler`.
- **Non-root readability fix**: the canonical image bakes the binary under `/root/.cloakbrowser`, but Lambda runs the container as a non-root user that cannot read `/root` (mode 750). The layer makes `/root` and the binary tree world-readable and traversable, and recreates the welcome-banner marker that the canonical image deletes, because Lambda's read-only filesystem could not write it and the banner would print to CloudWatch on every cold start.
- **Environment**: `HOME=/tmp` (Lambda allows writes only under `/tmp`), `CLOAKBROWSER_CACHE_DIR=/root/.cloakbrowser` so the cache still resolves to the baked binary instead of an empty `/tmp/.cloakbrowser`, and `CLOAKBROWSER_AUTO_UPDATE=false` because the runtime filesystem is read-only.

Consequence: the cache directory is read-only at runtime, so only the baked free binary can run. A license key would send `ensure_binary()` down the Pro path, which must download a Pro build when none is cached and would fail on the read-only cache. A Pro deployment would have to bake its Pro build in at image build time. ^[inferred] See [[projects/cloakbrowser/subsystems/binary-download-and-verification|Binary download and verification]].

## Dual-mode entrypoint

`lambda-entrypoint.sh`:

1. Always starts Xvfb on display `:99` (1920x1080x24), after deleting stale lock and socket files that a dead Xvfb left behind during cold-start storms. It polls for the X11 socket, then waits a further 0.2 s to cover the gap between `bind()` and `listen()` under CPU contention.
2. If the command is exactly one dotted identifier (`module.func`), routes it through `awslambdaric`: via the bundled Runtime Interface Emulator (`aws-lambda-rie`) when `AWS_LAMBDA_RUNTIME_API` is unset (local testing), or directly against the real Runtime API in production.
3. Otherwise execs the command unchanged, preserving the canonical image's surfaces.

## Handler contract

`lambda_handler.py` takes an event with a required `url` and returns `title`, final `url`, `html` and an optional base64 screenshot.

- Launch options forwarded to `launch_context_async`: `proxy`, `humanize`, `human_preset`, `geoip`, `timezone`, `locale`, `viewport` (default 1920x947), `user_agent`.
- The browser always runs **headed** under Xvfb, never headless. ^[inferred] This avoids headless-specific signals; see [[projects/cloakbrowser/detection-vectors/headless-detection|Headless detection]].
- **Smart wait** is the default post-navigation wait: poll the document's HTML length and return once it has not changed for `dom_stable_ms` (1500 ms), capped at `max_settle_ms` (15000 ms). The rationale given is that `networkidle` is unreliable on modern single-page apps because analytics beacons, long-polling and websockets keep the network busy without changing the DOM. Explicit load-state, selector and fixed waits are also available.
- Two Chromium flags are always forced: `--disable-dev-shm-usage` (Lambda's `/dev/shm` is about 64 MB and renderers crash mid-paint without it) and `--no-zygote` (Lambda's process model cannot fork from the zygote, so renderers fail to spawn and the first new page raises `TargetClosedError`).

## Retry orchestration

- **Launch retries**: 3 attempts with 0.3 s and 0.6 s backoff, for Xvfb/Chromium spawn races at cold start; not configurable.
- **Strategy retries** (default 1, set by `retries`): classify the error and relaunch with adjusted settings. `ERR_CERT_*` adds `--ignore-certificate-errors` and a 60 s navigation budget; `Timeout … exceeded` and `ERR_CONNECTION_TIMED_OUT` raise the navigation budget to 90 s and settle cap to 25 s. DNS failures, SSL protocol errors, refused connections and HTTP response-code failures are not retried.
- A final failure raises with a `retry_history` block listing each attempt; successful responses carry no extra fields.

## Security posture

- Only `http`/`https` URLs. Hostnames are resolved before navigation and rejected if any resolved address is not globally routable (private, loopback, link-local, reserved or multicast), which blocks cloud metadata endpoints such as 169.254.169.254.
- The final URL is re-validated after load and after waits to catch redirects. The documented limits: re-validation prevents exfiltrating the response but not the request itself (side-effecting GETs on internal endpoints still fire), and DNS rebinding can in theory bypass the pre-check.
- Callers cannot pass Chromium flags: the handler strips any `extra_args` or internal `_strategy_args` keys from the incoming event, so only its own retry strategies add flags. `wait_for_function` is not exposed, so events cannot inject arbitrary JavaScript.
- An authentication layer is required if the function is reachable by untrusted callers; URL validation is defense in depth.

## Operational guidance (as of 26/10/2026, commit b1a00ed)

| Setting | Recommendation | Reason given |
|---|---|---|
| Architecture | arm64 | about 20% cheaper; native build on Apple Silicon |
| Memory | 3008 MB | Lambda ties vCPU to memory; Chromium is noticeably slower below about 1769 MB |
| Timeout | 120–180 s | a timed-out first attempt (30 s) plus a 90 s retry plus cleanup reaches about 120–130 s |
| Ephemeral storage | 1024 MB | profiles and screenshots can fill the 512 MB default |
| Networking | no VPC | the binary is baked into the image; add VPC/NAT only for proxy egress |

Cold starts take about 80–90 s; warm invocations 3–15 s. Raising `max_settle_ms` helps when dynamic content is missing on cold runs.

## Licensing constraint

The binary inside the image is governed by the CloakBrowser Binary License: internal use is free, but exposing the function as a paid browser-as-a-service API needs an OEM/SaaS license, and pushing the image to a public registry counts as prohibited redistribution. See [[projects/cloakbrowser/subsystems/license-and-tiers|License and tiers]].

## Pinned by tests (B8 batch)

`tests/test_lambda_security.py` numbers its classes after the handler's security fixes and confirms the posture above:

- **Fix 1, schemes.** Only `http://` and `https://` are accepted. A URL without a hostname is rejected.
- **Fix 2, SSRF.** Blocked: the AWS metadata address with and without a path; loopback (`127.0.0.1`, `127.0.0.2`, `localhost`, `[::1]`); RFC 1918 ranges; `0.0.0.0`; carrier-grade NAT (`100.64.0.0/10`); and an IPv4-mapped IPv6 loopback (`[::ffff:127.0.0.1]`). Unresolvable hostnames are rejected with "Cannot resolve".
- **Fix 3, flags.** A caller's `extra_args` (the test tries `--remote-debugging-port=9222`) never reaches the launch. Internal `_strategy_args` from the retry classifier do (for example `--ignore-certificate-errors` after `ERR_CERT_AUTHORITY_INVALID`). The Lambda hardening flags `--disable-dev-shm-usage` and `--no-zygote` are always present.
- **Fix 5, redirects.** `_validate_url` rejects redirect targets such as the metadata address or localhost, and the scrape path validates `page.url` before reading content.

Two of these pins are weaker than the rest. That callers cannot inject `_strategy_args`, and that validation runs after navigation, are checked by inspecting the source text of `_run` and `_attempt_scrape` (string search and AST walk), not by running them. Fix 4 has no test in this file. ^[ambiguous] See [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]] for how such flags would otherwise reach Chromium.

## Open Questions

- Both the Dockerfile header and INSTRUCTIONS.md cite `0.3.25` as the example tag to pin while the wrappers are at 0.6.4, and the Dockerfile itself uses `latest`. Which tag is actually tested is not stated. ^[ambiguous]
- How Pro license seats behave across many concurrent Lambda containers is not discussed. ^[inferred]

## Sources

Commit `b1a00ed` (26/10/2026):

- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/Dockerfile (commit b1a00ed, P2 pass)
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/INSTRUCTIONS.md
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/lambda-entrypoint.sh
- repo:github.com/CloakHQ/CloakBrowser/examples/integrations/aws_lambda/lambda_handler.py
- repo:github.com/CloakHQ/CloakBrowser/tests/test_lambda_security.py (commit b1a00ed, B8 batch)
