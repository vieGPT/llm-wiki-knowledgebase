---
title: >-
    GeoIP resolver
category: subsystems
tags: [geoip, timezone, locale, proxy, network-identity]
aliases: [geoip.py, geoip.ts, maybe_resolve_geoip]
relationships:
  - target: "[[projects/cloakbrowser/techniques/geoip-timezone-locale-matching]]"
    type: implements
  - target: "[[projects/cloakbrowser/techniques/webrtc-ip-spoofing]]"
    type: related_to
  - target: "[[projects/cloakbrowser/subsystems/proxy-handling]]"
    type: uses
sources:
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/geoip.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py
  - repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__main__.py
  - repo:github.com/CloakHQ/CloakBrowser/README.md
  - repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md
  - repo:github.com/CloakHQ/CloakBrowser/js/src/geoip.ts
  - repo:github.com/CloakHQ/CloakBrowser/js/README.md
  - repo:github.com/CloakHQ/CloakBrowser/js/tests/geoip.test.ts
  - repo:github.com/CloakHQ/CloakBrowser/tests/test_geoip.py
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/GeoIp.cs
  - repo:github.com/CloakHQ/CloakBrowser/dotnet/tests/CloakBrowser.Tests/MiscTests.cs
summary: >-
    Optional geoip=True path: discovers the egress IP via echo services through the proxy, maps it with a local GeoLite2 DB to timezone and locale, and reuses the IP for WebRTC.
provenance:
  extracted: 0.88
  inferred: 0.1
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: core
created: 2026-10-31T05:45:14+01:00
updated: 2026-10-31T06:58:25+01:00
---

# GeoIP resolver

`geoip.py` (mirrored by `geoip.ts`) turns a proxy into the network identity the browser should claim: an egress IP, an IANA timezone and a BCP 47 locale. It runs only with `geoip=True`, needs the optional extra `pip install 'cloakbrowser[geoip]'` (`geoip2`, plus `socksio` for SOCKS proxies), and feeds the binary flags set by [[projects/cloakbrowser/subsystems/launcher-and-stealth-args|Launcher and stealth args]]. The rationale is in [[projects/cloakbrowser/techniques/geoip-timezone-locale-matching|GeoIP timezone/locale matching]].

## Flow

1. **Explicit values win.** `maybe_resolve_geoip()` first promotes raw flags in `args` (`--fingerprint-timezone`, `--lang`, `--fingerprint-locale`) to explicit values (fix in 0.5.0). If both timezone and locale are explicit, it only resolves the exit IP for WebRTC, and only when a proxy is set: without a proxy that IP equals what the site already sees, so the third-party echo call is skipped.
2. **Ensure the database** `GeoLite2-City.mmdb` (~70 MB) under `~/.cloakbrowser/geoip/`, downloaded from the P3TERX GitHub mirror of MaxMind GeoLite2 (no MaxMind key needed). The download is deliberately outside the resolution timeout because a first-use fetch legitimately outlasts it.
3. **Discover the egress IP** by querying echo services in order (`api.ipify.org`, `checkip.amazonaws.com`, `ifconfig.me/ip`) **through the proxy**; with no proxy, the machine's own public IP (proxy-free geoip since 0.4.8). The response must parse as an IP. Querying through the proxy gives the real exit IP, which can differ from the gateway hostname's DNS. ^[inferred]
4. **Fallback**: resolve the proxy hostname via DNS (only when a proxy is set and time remains).
5. **Lookup**: city record → `location.time_zone`; country ISO code → locale via `COUNTRY_LOCALE_MAP` (132 countries as of 26/10/2026, commit b1a00ed, e.g. `AT → de-AT`, `SC → en-SC`). An unmapped country logs a warning and uses `en-US` (0.6.3, #591) rather than failing.
6. **Return** `(timezone, locale, exit_ip)`. The exit IP is a free by-product reused for `--fingerprint-webrtc-ip`. See [[projects/cloakbrowser/techniques/webrtc-ip-spoofing|WebRTC IP spoofing]].

## Failure policy

Since 0.5.10 a requested resolution that fails **aborts the launch** with `RuntimeError`: egress IP undiscoverable, deadline expired, database unavailable, lookup error, or timezone/locale still unresolved. Before that, failures silently continued on the container clock, i.e. UTC + `en-US`, which is exactly the mismatch geoip exists to prevent. ^[inferred] The exit IP is resolved even when the database is missing, because dropping it would let WebRTC expose the real IP behind a proxy while the connection shows the proxy IP, "a real deanonymization" (code comment).

## Timeout

The resolution deadline comes from `CLOAKBROWSER_GEOIP_TIMEOUT_SECONDS`; non-finite values fall back to the default, `0` or negative disables the deadline, and each echo request is capped at 10 s. The code default is **20 s** (raised from 5 s in 0.5.10), while the README configuration table still lists `5`. Both test suites pin 20 s (Python `DEFAULT_GEOIP_TIMEOUT_SECONDS == 20.0` in `tests/test_geoip.py`, JS 20,000 ms), so the README value is stale documentation rather than an open contradiction.

## Concurrency and freshness

- A per-process lock serializes first-use downloads, so N concurrent launches fetch the DB once (#458).
- A DB older than 30 days is refreshed in a background thread, skipped when another download holds the lock. Replacement uses `os.replace`, because a plain rename cannot overwrite on Windows; that bug made every launch re-download the DB in a loop until 0.4.13.

## Proxy credential handling

Credentials from a proxy *dict* are folded back into a URL before resolution. Until 0.5.3 (#469) dict-form HTTP credentials were dropped, so the echo call failed and resolution silently fell back to the gateway hostname, reporting the gateway's location instead of the exit. JS-specific fixes covered the CONNECT `Host` header (0.5.1) and per-proxy connection pooling (0.5.3). See [[projects/cloakbrowser/subsystems/proxy-handling|Proxy handling]].

## Diagnostics

`cloakbrowser info --proxy <url>` runs the same resolution and prints exit IP, timezone and locale without launching a browser; plain `info` only reports whether the DB file exists (see [[projects/cloakbrowser/subsystems/cli|CLI]]). `cloakserve` accepts `proxy` and `geoip=true` query params per seed (see [[projects/cloakbrowser/subsystems/cloakserve-cdp-server|cloakserve CDP server]]).

## Privacy note

Using geoip sends HTTP requests through the proxy to third-party echo services (README). With no proxy, the machine's own public IP is sent to those services. ^[inferred]

## JS port (`geoip.ts`)

These details come from the B4 batch reading `js/src/geoip.ts` and `js/README.md` (commit b1a00ed):

- **Dependencies**: both are optional npm peers, loaded with dynamic `import()`. Without `mmdb-lib`, `geoip: true` throws an install hint. Without `socks-proxy-agent`, a SOCKS5 proxy cannot be tunnelled for the echo call: the wrapper warns and the exit IP comes back empty, so the DNS fallback or an abort follows.
- **Timeout default**: `DEFAULT_GEOIP_TIMEOUT_MS` is 20,000. It is exported "for parity testing", which confirms the code-side 20 s against the README's 5.
- **WebRTC flag rule**: lives in `geoip.ts` in JS (in `browser.py` in Python).
  - `appendWebrtcExitIp` adds `--fingerprint-webrtc-ip=<exitIp>` unless the user already set the flag. It lives in one place because "this rule must stay identical across every launch path".
  - `resolveWebrtcArgs` swaps a literal `--fingerprint-webrtc-ip=auto` for the resolved exit IP. Without a proxy, or when resolution fails, it removes the flag with a warning instead of passing `auto` through.
- **Rotating proxies**: the JS README warns that with rotating residential proxies the DNS-resolved IP may differ from the exit IP. In that case users should pass explicit `timezone` and `locale`.
- **No private-IP helper**: there is no counterpart to Python's `_is_private_ip`, which is itself never called. See [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].

## .NET port (`GeoIp.cs`, batch D)

`GeoIp.cs` is a "Direct port of Python `cloakbrowser/geoip.py`" (commit b1a00ed). The flag promotion, the both-explicit shortcut and the WebRTC helpers live in `CloakLauncher.cs`, as they do in Python's `browser.py`. The C# port keeps the echo-service order, the P3TERX mirror, the 30-day background refresh, the 20 s `CLOAKBROWSER_GEOIP_TIMEOUT_SECONDS` default (`MiscTests` pins it), the abort-on-failure policy, the `en-US` fallback for unmapped countries, and a country→locale map with the same entry count as Python's. .NET-specific details:

- **No optional extra.** `MaxMind.GeoIP2` (`DatabaseReader`) is a hard NuGet dependency, so `GeoIp = true` cannot fail for a missing package.
- **Proxying the echo call.** The echo call goes through `HttpClientHandler` with a `WebProxy`. Credentials are stripped from the proxy URL and passed separately as `NetworkCredential`, which covers dict-form credentials (#469) by construction. ^[inferred] There is no `socksio`/`socks-proxy-agent` equivalent. If building the handler throws, the code warns "SOCKS5 proxy requires a SOCKS-capable transport" and returns no exit IP.
- **Concurrency.** A `SemaphoreSlim(1, 1)` serializes first-use downloads (#458). The background refresh tries it without waiting and skips if a download is running. Replacement is a temp file plus `File.Move(overwrite: true)`, the .NET counterpart of `os.replace`.
- **Deadlines** use the monotonic `Stopwatch` clock, and each echo request is capped at min(10 s, remaining).
- **Dead helper carried over.** `IsPrivateIp` is public and unit-tested in `MiscTests`, but nothing calls it, just like Python's `_is_private_ip` (see [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]]).
- `MiscTests` also pins flag promotion: raw `--lang` promotes to locale, an explicit parameter beats a raw flag, and with no proxy and both values explicit there is no echo call.

## Pinned by tests (B6, JS suite)

`js/tests/geoip.test.ts` (commit b1a00ed) fixes these behaviours of `geoip.ts`:

- **Timeout default** is 20,000 ms, which settles the code side of the README's "5" discrepancy above.
- **Explicit values.** With no proxy and both timezone and locale explicit, the echo services are never contacted. Raw `--fingerprint-timezone` and `--fingerprint-locale` flags count as explicit, and so does `--lang`, which promotes to locale. A dedicated `timezone` option beats a different raw flag.
- **Exit-IP-only path degrades instead of aborting.** With a proxy and both values explicit, an expired deadline returns quickly (under 500 ms in the test) with the explicit values and an undefined exit IP; it does not throw. The abort policy above therefore applies when timezone or locale still need resolving. In this case `resolveWebrtcArgs` would drop `--fingerprint-webrtc-ip=auto` with a warning instead of passing it on. ^[inferred]
- **Hard failures** surface as errors: a failed database download ("database is unavailable") and an unreadable database ("GeoIP lookup failed").
- **Database download.** It is not subject to the resolution timeout. It follows redirects, writes nothing to stdout (status goes to stderr, so MCP stdio servers stay intact), and is fetched only once when five first-use launches race in one process (#458).
- **Proxy credentials.** Dict-form `username`/`password` reach the HTTP proxy as a `Proxy-Authorization: Basic` header on every CONNECT request, which pins the #469 fix in JS.
- **Proxy URL parsing.** `resolveProxyIp` handles credentials and returns null for an empty, invalid or scheme-less string. That is why `maybeResolveGeoip` prepends `http://` first.
- **Locale map.** Every value is two-part BCP 47 `language-REGION`. An unmapped country (stubbed as `XX` with timezone `Indian/Mahe`) yields `en-US` plus the real timezone and exit IP (#591).

## Pinned by tests (B7, Python suite)

`tests/test_geoip.py` (commit b1a00ed) confirms the flow and failure policy above for `geoip.py` and `browser.py::maybe_resolve_geoip`, mirroring the JS suite:

- **Abort, never degrade silently**: an undiscoverable egress IP ("could not discover the egress IP"), a missing database ("database is unavailable"), a lookup exception (e.g. a corrupt DB) and a partial result ("could not determine locale") each raise `RuntimeError`. Without `geoip2` the call raises `ImportError` with the `pip install 'cloakbrowser[geoip]'` hint. With `CLOAKBROWSER_GEOIP_TIMEOUT_SECONDS=0.05` and a locale still to resolve, a stalled lookup aborts the launch in under 0.5 s.
- **Per-field precedence**: dedicated `timezone=`/`locale=` parameter > raw flag in `args` (`--fingerprint-timezone`, `--lang`, `--fingerprint-locale`) > GeoIP value, decided field by field, so GeoIP fills only the missing one. When both fields are explicit, the GeoIP lookup is skipped. With a proxy the exit IP is still resolved for WebRTC. Without a proxy no echo call is made and the exit IP stays empty.
- **No proxy, nothing explicit**: the machine's own public IP is resolved, with the echo request sent directly (`proxy=None`); the IP still drives `--fingerprint-webrtc-ip`.
- **Locale map**: every value is `language-REGION` (lowercase/uppercase); common countries (US, GB, DE, FR, JP, BR, IL, RU) are present; a Seychelles datacenter exit maps to `en-SC` and an unmapped country to `en-US` (#591).
- **Proxy host parsing**: literal IPv4 and bracketed IPv6, with or without credentials, and DNS for hostnames; an empty or scheme-less string yields `None`.
- **Database download**: `os.replace` overwrites an existing file (the Windows rename bug), and five threads racing on first use trigger exactly one download (#458).
- `_is_private_ip` is unit-tested (loopback and RFC 1918 true, public false) although nothing in the package calls it; see [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].

## Open Questions

- Locale mapping is one locale per country (e.g. `CH → de-CH`, `BE → fr-BE`, `CA → en-CA`), so multilingual countries always get one language; whether the binary derives `Accept-Language` variants from this is not documented here.

## Sources

- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/geoip.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/browser.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/cloakbrowser/__main__.py` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/README.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/CHANGELOG.md` (commit `b1a00ed`)
- `repo:github.com/CloakHQ/CloakBrowser/js/src/geoip.ts`, `js/README.md` (commit `b1a00ed`, B4 batch)
- `repo:github.com/CloakHQ/CloakBrowser/js/tests/geoip.test.ts` (commit `b1a00ed`, B6 batch)
- `repo:github.com/CloakHQ/CloakBrowser/tests/test_geoip.py` (commit `b1a00ed`, B7 batch)
- `repo:github.com/CloakHQ/CloakBrowser/dotnet/src/CloakBrowser/GeoIp.cs`, `dotnet/tests/CloakBrowser.Tests/MiscTests.cs` (commit `b1a00ed`, batch D)
