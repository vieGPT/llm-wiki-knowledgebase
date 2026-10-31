---
title: >-
    Proxy handling
category: subsystems
tags: [proxy, networking, playwright, puppeteer, chromium]
aliases: [proxy.ts, ProxySettings, resolveProxyConfig]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/launcher-and-stealth-args]]"
    type: related_to
  - target: "[[projects/cloakbrowser/techniques/proxy-signal-removal]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/geoip-resolver]]"
    type: related_to
sources:
  - repo:github.com/CloakHQ/CloakBrowser/js/src/proxy.ts
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py
  - repo:github.com/CloakHQ/CloakBrowser/bin/cloakserve
  - repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/src/types.ts
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/proxy.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/puppeteer.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/launch.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_proxy.py
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/ProxyResolver.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/ProxySettings.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/ProxyResolverTests.cs
summary: >-
    How the wrappers turn a proxy string or dict into either a Playwright proxy option or a Chrome --proxy-server flag, and why credentialed proxies bypass Playwright's CDP auth path.
provenance:
  extracted: 0.84
  inferred: 0.14
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:22+01:00
updated: 2026-10-31T06:58:25+01:00
---

# Proxy handling

The shared proxy module (`js/src/proxy.ts`, mirrored in Python) decides, for every launch, whether a proxy reaches Chromium as Playwright's `proxy` launch option or as a raw `--proxy-server=` flag on the binary's command line. The result of `resolveProxyConfig` is one or the other, never both. The binary-flag route is preferred whenever credentials are involved and the binary can parse them, because Playwright's CDP-based proxy auth interceptor breaks on some proxies (issue #182). The wrapper does not rotate proxies or ship any; the user brings their own (README).

## Accepted input

- `proxy` is either a URL string (`http://user:pass@host:port`, `socks5://...`, bare `user:pass@host:port`, bare `host:port`) or a dict `{ server, bypass?, username?, password? }` (`ProxyDict`, typed in `types.ts`).
- Bare strings without a scheme get `http://` prepended before parsing (`ensureProxyScheme`); GeoIP resolution only needs the hostname from this.
- `parseProxyUrl` splits inline credentials into separate `username`/`password` fields and rebuilds a credential-free `server`, URL-decoding special characters. Strings that are not parseable URLs pass through unchanged.

## Routing decision (`resolveProxyConfig`)

| Proxy kind | Binary supports inline HTTP auth? | Route |
|---|---|---|
| SOCKS5 / SOCKS5h (any credentials) | n/a | `--proxy-server=socks5://user:pass@host` (+ `--proxy-bypass-list` from dict `bypass`) |
| HTTP/HTTPS with credentials | yes | `--proxy-server=http://user:pass@host` (+ bypass list) |
| HTTP/HTTPS with credentials | no (older free macOS / linux-arm64 builds) | Playwright `proxy` dict with split credentials |
| HTTP/HTTPS without credentials | either | Playwright `proxy` dict |

- Whether the binary supports inline HTTP credentials is a version gate, `binarySupportsHttpProxyInlineAuth(licenseKey, browserVersion, releaseChannel)`, keyed per platform with a minimum version table. The floors are linux-x64 and windows-x64 `146.0.7680.177.5`, linux-arm64 and macOS `148.0.7778.215.3` (as of 26/10/2026, commit b1a00ed; see [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]). Below the floor, `user:pass@` in `--proxy-server` is treated as an invalid host. It fails safe (returns false) for a local binary override with no declared version. Python, JS and .NET mirror this gate.
- "HTTP proxy inline credentials" ships as a network-layer feature of the v0.6.4 binary line (README "Latest", as of 26/10/2026, commit b1a00ed).
- SOCKS5 always bypasses Playwright: Chrome handles RFC 1929 username/password auth natively. The README adds that QUIC/HTTP3 tunnels through SOCKS5 via UDP ASSOCIATE, and recommends SOCKS5 where available because it tunnels raw TCP and avoids HTTP CONNECT problems some proxies have with HTTP/2.

## Credential re-encoding

Chromium's `--proxy-server` parser truncates passwords at `=` and other special characters (issue #157). `normalizeSocksStringUrl` and `normalizeHttpStringUrl` therefore re-encode the userinfo part:

- They split userinfo from host at the **last** `@` (RFC 3986), so a raw `@` inside a password parses correctly, matching Python's `rpartition('@')`.
- Decoding is lenient (a bare `%` stays literal, like Python's `unquote`), then each part is percent-encoded again. The operation is idempotent: pre-encoded input is unchanged.
- Parsing is manual rather than via WHATWG `URL` setters, because those setters re-encode `%` and would double-encode on a round trip.
- An empty password (`user:@host`) is kept distinct from no password (`user@host`).
- A malformed port or any unexpected failure logs a warning and passes the original string through, so Chromium surfaces the real error.
- When credentials actually changed, the wrapper logs an info notice suggesting the user pre-encode the URL.

## Puppeteer differences

The Puppeteer wrapper has its own `resolveProxy` because Puppeteer has no `proxy` launch option:

- SOCKS5 and inline-auth-capable binaries: identical `--proxy-server` flags.
- Older binaries: credentials are stripped from `--proxy-server` and the wrapper patches `browser.newPage()` to call `page.authenticate({ username, password })` on every new page. Dict credentials take precedence over credentials embedded in `server`.

See [[projects/cloakbrowser/architecture/playwright-vs-puppeteer-backends|Playwright vs Puppeteer backends]].

## Request-client leak warning (#579)

When the proxy goes to Chrome as `--proxy-server`, Playwright does not know about it, so Playwright's own HTTP client (`context.request`, `page.request`, `route.fetch()`) sends from the machine's real IP. `warnOnRequestClientUse` wraps the private `request._innerFetch` once per context and prints a one-time warning on first use, recommending `page.evaluate(() => fetch(...))` for proxied requests instead.

- The wrap self-removes after the first call so Playwright's API naming in traces stays intact.
- It is skipped when the context has its own per-context proxy (that one does reach the request client), and silently skipped if the private API is renamed.
- `launch()` installs it by wrapping `browser.newContext`; `launchPersistentContext()` installs it directly on the returned context.

This is a real IP leak path that the wrapper only warns about, not closes. ^[inferred] See [[projects/cloakbrowser/detection-vectors/proxy-and-network-signals|Proxy and network signals]].

## Interaction with other launch inputs

- Proxy args are appended after WebRTC/GeoIP-derived args and before `buildArgs` dedup in [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]].
- The same proxy URL feeds exit-IP resolution in [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]] and `--fingerprint-webrtc-ip=auto` ([[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]]).
- Binary-side proxy hardening (timing zeroing, header stripping, transparent proxy mode) is separate from this wrapper logic: see [[projects/cloakbrowser/techniques/proxy-signal-removal|Proxy signal removal]].

## Python implementation (`_resolve_proxy_config`, P2 pass)

`cloakbrowser/browser.py` implements the same routing table. Its docstring gives the rule in one line: credentialed proxies (SOCKS5 always, HTTP/HTTPS only on binaries with inline proxy auth) go through `--proxy-server` with inline credentials, "bypassing Playwright's CDP auth interceptor which breaks on some proxies and Google domains (#182)". All four launchers (`launch`, `launch_async`, `launch_persistent_context` and its async twin) call it. Details that differ from, or add to, the JS description:

- **Credential test is coarse for strings.** A string counts as credentialed whenever it contains `@`; a dict only when `username` is non-empty.
- **Two empty-password conventions.** Rebuilding a URL from a dict treats an empty or missing password as "no password" (no colon), while re-encoding a string keeps `user:@host` distinct from `user@host`, mirroring how `urlparse` distinguishes them.
- **Scheme handling.** The HTTP re-encoder prepends `http://` to a scheme-less string before it goes into `--proxy-server`. The Playwright-dict parser adds the scheme only when the bare string has credentials, and returns a credential-free string unchanged.
- **Dict parsing for Playwright.** Credentials are URL-decoded into `username`/`password`; the rebuilt `server` keeps scheme, host, port and path but drops params, query and fragment. A password is added only when non-empty.
- **SOCKS detection** is a case-insensitive prefix check for `socks5://` or `socks5h://`.
- **#579 warning.** The Python hook wraps the private `_inner_fetch` of Playwright's implementation-level request context, which the sync and async APIs share, so one wrap catches `context.request`, `page.request` and `route.fetch()`. It removes itself on first call (tolerating two concurrent first calls), writes straight to stderr so an application's logging setup cannot silence it, and is skipped for a context that has its own proxy or if the private attribute is missing.

### cloakserve does not use this table

The Docker CDP server imports only the SOCKS string normalizer. It passes every `?proxy=` value, HTTP or SOCKS, as `--proxy-server` after credential re-encoding, with no inline-auth version check and no fallback, because there is no Playwright launch to hand a proxy dict to. See [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]].

## .NET implementation (`ProxyResolver.cs`, batch D)

`ProxyResolver` is a "Direct port of the proxy helpers in Python `cloakbrowser/browser.py`" (commit b1a00ed). It has the same routing table: SOCKS5 always inline, credentialed HTTP inline only above `BinarySupportsHttpProxyInlineAuth`, everything else through Playwright's `Proxy`. A `ProxySettings` bypass list becomes `--proxy-bypass-list`.

- **Own URL parser.** Instead of `System.Uri`, it hand-parses scheme, userinfo, bracketed IPv6, port, path, query and fragment so the pieces match Python's `urlparse`, down to lowercasing the host, "so the assembled proxy URL/server string is byte-for-byte identical". Username/password distinguish absent (null) from present-but-empty. A port outside 0-65535 is a `FormatException`; the string normalizers catch it, warn, and pass the URL through unchanged.
- **Re-encoding** uses `Uri.EscapeDataString(Uri.UnescapeDataString(...))` as the counterpart of `quote(unquote(...), safe="")`, and logs the same "Auto URL-encoded … credentials" notice when anything changed.
- `ExtractProxyUrl` rebuilds a credentialed URL from a `ProxySettings` object for the GeoIP and WebRTC lookups (see [[projects/cloakbrowser/subsystems/geoip-resolver|GeoIP resolver]]).
- `ProxyResolverTests` pins the table: SOCKS5 string and dict with credentials and bypass, special-character encoding, and HTTP with credentials going inline on a patched binary but falling back to Playwright's proxy below the floor. See [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

## Pinned by tests (B6, JS suite)

`js/tests/proxy.test.ts`, `puppeteer.test.ts` and `launch.test.ts` (commit b1a00ed) confirm the routing table and re-encoding rules above. Points they fix:

- **Floors are exact.** On linux-x64, a declared `146.0.7680.177.5` takes the inline `--proxy-server=http://user:pass@...` route and `.177.3` falls back to the Playwright dict. A pinned darwin-arm64 `148.0.7778.215.3` already gets inline auth. Free macOS and free linux-arm64 with no pin fall back. The gate reads `process.platform`/`process.arch` at call time.
- **Channel-aware gate.** With a cached stable Pro build below the floor and a cached preview build above it, the same credentialed proxy goes to the Playwright dict on `stable` and inline on `preview`.
- **Dict credentials are rebuilt into the URL.** A SOCKS5 dict with `password: "p@ss"` becomes `socks5://user:p%40ss@host`. A dict `bypass` becomes `--proxy-bypass-list` on both SOCKS5 and inline-HTTP routes. A credential-free HTTP dict with `bypass` is passed through to Playwright unchanged.
- **Re-encoding edge cases**: password-only userinfo (`:pass=123@`), a literal `%` that is not a valid escape (`100%sure` becomes `100%25sure`), IPv6 hosts, several raw `@` in a password (split at the last), and malformed URLs (broken IPv6 bracket, non-numeric port) passed through without throwing.
- **Log hygiene.** The info notice ("Auto URL-encoded SOCKS5 ...") appears only when credentials really changed. It is not logged for pre-encoded input, credential-free URLs or a host that differs only in letter case, and it never contains the password in raw or encoded form.
- **Bare formats**: `user:pass@host:port` gets `http://` and split credentials, but a bare `host:port` without credentials passes through unchanged, scheme-less.
- **Puppeteer**: a credential-free HTTP *string* proxy is a `--proxy-server` flag (Puppeteer has no proxy option); SOCKS5 credentials stay inline and `page.authenticate` is never called for them.
- **#579 warning**: emitted once, on the first request-client call rather than at launch, for both `launchContext` and `launchPersistentContext`. It is suppressed when Playwright owns the proxy (a credential-free HTTP proxy) or when the context has its own proxy.

## Pinned by Python tests (B8 batch)

`tests/test_proxy.py` pins the Python side of `_resolve_proxy_config` and confirms that it matches the JS routing table above:

- **SOCKS5/SOCKS5h** (strings or dicts, with or without auth, in any letter case) always become `--proxy-server`. Dicts are rebuilt into a URL, with IPv6 brackets kept and the `bypass` value emitted as `--proxy-bypass-list`.
- **HTTP with credentials** becomes `--proxy-server` on linux-x64 and windows-x64 at `146.0.7680.177.5`. On darwin-arm64 and linux-arm64 it falls back to the Playwright dict. The test comment gives the reason: the free macOS build (145.x) and linux-arm64 build (`146.0.7680.177.3`) predate the inline proxy-auth patch. A macOS caller who pins `148.0.7778.215.3` gets `--proxy-server`. A pinned older version also turns inline auth off on supported platforms.
- **HTTP without credentials** (string or dict, with or without bypass) always becomes the Playwright dict.
- **Re-encoding.** `=` and `@` in passwords are percent-encoded. The operation is idempotent, and a literal `%` survives. An empty password keeps its colon. Path and query, and port `0`, are preserved. A malformed port or IPv6 host passes through with a warning. The info log fires only when the credentials really changed, not for cosmetic differences. Bare `user:pass@host:port` strings parse, and credentials never leak into `server`.
- **GeoIP input.** Dict proxies have their credentials rebuilt for exit-IP lookup (SOCKS5 and HTTP). Bare and scheme-less strings are normalized. An explicit timezone is preserved, `geoip=False` skips resolution, and no proxy means the machine's own IP is used.
- **Request-client warning (#579).** Python hooks the browser's `context` event, or the persistent context directly, and **only** when the proxy took the `--proxy-server` route (SOCKS5 in the tests, not a plain HTTP dict). Nothing prints until the request client is first used, then exactly once. A context with its own proxy never warns.
- **#27 regression** (`tests/test_stealth.py`, live): `add_init_script` combined with a context-level proxy must load pages without `ERR_TUNNEL_CONNECTION_FAILED`.

## Open Questions

- Whether `--proxy-bypass-list` is honoured when a string (not dict) proxy is used: the string path never emits it, so bypass requires the dict form. ^[inferred]

## Sources

- repo:github.com/CloakHQ/CloakBrowser/js/src/proxy.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py (commit b1a00ed, P2 pass)
- repo:github.com/CloakHQ/CloakBrowser/bin/cloakserve (commit b1a00ed, P2 pass)
- repo:github.com/CloakHQ/CloakBrowser/js/src/playwright.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/puppeteer.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/src/types.ts (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/README.md (commit b1a00ed)
- repo:github.com/CloakHQ/CloakBrowser/js/tests/proxy.test.ts, js/tests/puppeteer.test.ts, js/tests/launch.test.ts (commit b1a00ed, B6 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_proxy.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/tests/test_stealth.py (commit b1a00ed, B8 batch)
- repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/ProxyResolver.cs, dotnet/src/CloakBrowser/ProxySettings.cs, dotnet/tests/CloakBrowser.Tests/ProxyResolverTests.cs (commit b1a00ed, batch D)
