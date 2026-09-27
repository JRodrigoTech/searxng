# Dependencies

## Compatibility principle

Dependency choice is not purely ergonomic for this tool. The audited provider paths depend on HTTP/browser-fingerprint behavior and tolerant HTML parsing. A library substitution is acceptable only when compatibility tests prove equivalent provider requests and responses.

## Required baseline for parity

| Capability | Baseline |
| --- | --- |
| Async scheduling | Python `asyncio` or host equivalent |
| Monotonic deadlines | standard `time` clock |
| HTTP/browser fingerprint | `curl_cffi` |
| HTML parsing/XPath | `lxml.html` / lxml XPath |
| URL parsing/encoding | `urllib.parse` |
| Base64url | standard `base64` |
| Locale parsing/best-fit | Babel plus generated provider trait data, or a verified equivalent port |
| Provider TTL state | provider-scoped cache with expiry and secret/hash keying where required |
| Concurrency synchronization | asyncio coordination or lock-equivalent semantics |

## `curl_cffi`

For compatibility this is not merely an optimization. Relevant source behavior includes:

- browser impersonation profiles;
- default Chrome-family profile;
- Google `chrome99_android` override;
- secondary DuckDuckGo Firefox override with browser default headers disabled;
- HTTP/2 selection;
- conditional HTTP/3 for Bing;
- curl protocol restrictions;
- pooled async clients;
- explicit cookie-discard behavior.

A stock `httpx`/`aiohttp` implementation cannot be assumed equivalent at TLS/browser fingerprint level.

**PARITY BASELINE:** use `curl_cffi` for the initial implementation.

`httpx` or `aiohttp` can be evaluated later behind a transport experiment, but passing functional GET/POST tests alone is insufficient; live provider acceptance and request-profile tests are required.

## `lxml`

The audited parsers rely on tolerant HTML parsing and XPath behavior. Google additionally removes an optional XML declaration and then parses using HTML semantics.

`lxml` therefore provides the lowest-risk compatibility path. Rewriting selectors into CSS/selectolax/BeautifulSoup can work, but it changes parser semantics and should not happen in the first parity implementation.

**PARITY BASELINE:** use `lxml`.

## Babel and locale traits

Provider locale behavior is more than splitting `en-US`:

- provider-specific language/region tables;
- all-locale sentinels;
- aliases;
- language/script handling;
- territory-first and language fallback matching;
- official-language and population-based best-fit logic in the broader locale resolver.

Two viable compatibility approaches exist:

1. depend on Babel and port the generic best-fit rules plus generated provider trait snapshots;
2. bundle a complete generated lookup table for every locale Overmind will support, with golden parity tests.

For broad locale compatibility, option 1 is safer. A tiny hand-written `en/es/zh` map is not feature parity.

## Trait data

The larger source system persists generated engine trait data and can refresh it from provider pages/resources. Search calls consume the persisted mapping rather than scraping locale metadata every time.

The embedded tool should similarly ship a generated snapshot. Optional refresh tooling can live outside the search critical path.

Do not make each agent search depend on live requests to provider preference/region discovery pages.

## Provider state/cache

Primary DuckDuckGo requires a TTL cache for `vqd` keyed by transformed query and User-Agent. The audited source uses a provider-scoped persistent SQLite-backed engine cache and a secret hash of the key material.

Secondary DuckDuckGo also stores page-specific provider URLs with two TTLs:

- discovered page-1 URL: 7200 s;
- learned continuation URL: 3600 s.

For a simple single-process Overmind runtime, an in-memory TTL cache is technically sufficient for requests within one process. It is not persistence parity across restarts. If choosing in-memory state, document that deliberate deviation and ensure token/page semantics within a process remain exact.

A general database framework is not otherwise needed.

## Standard-library dependencies

Use standard library for:

- `urllib.parse` query/url operations;
- `base64.urlsafe_b64decode`;
- hashing for private cache keys if reimplemented;
- dataclasses/types where useful;
- monotonic clocks;
- asyncio task coordination.

Provider-specific behavior should not be hidden behind large generic crawling/search packages.

## What the embedded tool does not need

The search library does not need:

- Flask;
- a public HTTP server;
- templates/UI;
- Docker;
- browser automation;
- Selenium/Playwright;
- a JavaScript runtime;
- a search index;
- a queue system;
- the larger application's plugin framework.

The secondary DuckDuckGo challenge logic does not justify a JS runtime; its audited behavior is a narrow arithmetic parser.

## Suggested package dependency set

For the first high-fidelity implementation:

    curl-cffi
    lxml
    Babel

plus Python standard library and Overmind's existing runtime/tool interfaces.

If any of these already exist in the host dependency graph, reuse compatible pinned versions.

## Version pinning

Browser impersonation profiles are library/version-sensitive. Pin a tested `curl_cffi` range and run provider smoke tests when upgrading.

Parser behavior and Babel locale data can also shift across major versions. Golden fixtures and locale mapping tests should be part of dependency upgrade CI.

## Acceptance gate for replacing a dependency

A replacement is acceptable only if it passes:

- exact request parameter/header/cookie tests;
- provider browser-profile requirements or proven equivalent live acceptance;
- HTML fixture extraction tests;
- timeout/deadline tests;
- URL/encoding tests;
- locale trait tests;
- live smoke tests from intended egress.

The implementation agent should prioritize compatibility over minimizing three well-justified dependencies.