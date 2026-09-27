# Dependencies

## Selection principles

The new subsystem should have the smallest dependency set that still provides:

- reliable asynchronous HTTP;
- robust HTML parsing;
- URL parsing and encoding;
- locale mapping;
- bounded state and hashing;
- deterministic concurrency.

Do not add a dependency merely because a larger search application uses it. Every dependency must have a provider or runtime reason and an explicit fallback.

## Recommended minimum

| Capability | Recommendation | Status |
| --- | --- | --- |
| Async scheduling | Python asyncio | REQUIRED; standard library |
| Deadline and monotonic clock | time.monotonic | REQUIRED; standard library |
| URL parsing/encoding | urllib.parse plus a small tested policy layer | REQUIRED; standard library |
| Hashing token cache keys | hashlib | REQUIRED; standard library |
| HTTP | Existing async client in the host runtime; otherwise httpx or aiohttp | REQUIRED, choose one |
| HTML parsing | lxml.html or selectolax | REQUIRED for robust provider selectors |
| Locale parsing | Existing runtime locale/Babel support or a bounded mapping table | REQUIRED as a policy, library optional |
| In-memory TTL state | dict plus monotonic expiry and asyncio lock | REQUIRED; standard library |
| Structured telemetry | Host runtime logger/metrics interface | REQUIRED as an integration, no new framework |

## HTTP choice

### httpx

Why it may be needed:

- clean async client;
- connection pooling;
- familiar timeout and response APIs;
- optional HTTP/2.

Provider fit:

- Google, Bing, and DuckDuckGo HTML can use ordinary async HTTP if browser-profile fidelity is not required by live tests.

Standard-library alternative:

- urllib is available but would require a thread or custom async integration and weaker pooling.

Classification: RECOMMENDED default when the host already uses httpx or does not need TLS/browser impersonation matching.

### aiohttp

Why it may be needed:

- mature async streaming and pooling;
- good control over response body limits and cookie jars.

Provider fit:

- all three HTML paths.

Standard-library alternative:

- urllib plus asyncio.to_thread, with worse cancellation and pooling behavior.

Classification: VALID alternative.

### curl-cffi

Why it may be needed:

- browser impersonation;
- HTTP/2 and HTTP/3 controls;
- behavior close to the observed transport profile.

Provider fit:

- Google’s mobile profile;
- Bing’s optional HTTP/3 profile;
- DuckDuckGo’s coherent browser identity.

Standard-library alternative:

- none with equivalent browser-fingerprint control.

Classification: OPTIONAL until live smoke tests demonstrate that ordinary async HTTP is blocked or produces incompatible markup. If selected, pin versions and test platform wheels.

## HTML parsing choice

### lxml

Why it may be needed:

- tolerant HTML parsing;
- XPath and class-structure selection;
- efficient parsing of small result pages;
- handles the XML-like Google response after declaration removal.

Alternative:

- selectolax provides fast CSS-oriented parsing;
- BeautifulSoup is convenient but adds a parser dependency and can be slower or less precise for structural tests;
- standard-library html.parser requires more custom tree handling.

Classification: RECOMMENDED if the host already includes it; otherwise selectolax is a reasonable V1 alternative. Use one parser consistently across providers.

### BeautifulSoup

Why it may be useful:

- simple parser API for small fixtures;
- readable selectors.

Why it is not the default:

- usually requires an additional parser backend for robust behavior;
- encourages loose selection that can silently return navigation text.

Classification: OPTIONAL, not required when lxml or selectolax is available.

## Locale support

A full internationalization library is not required if V1 supports a bounded set of common language-country tags and ships provider trait tables.

### Babel

Use when:

- the host already depends on it;
- language/script/territory parsing and best-fit selection are needed.

Alternative:

- parse a strict subset of BCP-47-like tags with the standard library and map through explicit provider tables.

Classification: OPTIONAL. Avoid making a network search depend on a runtime trait refresh or a large locale database.

## State and persistence

No database is required for V1:

- DuckDuckGo token state can be in-memory with TTL.
- Locale traits can be static configuration plus optional refresh.
- Circuit-breaker state can be in-memory.

SQLite is only justified later if multiple processes must share token/trait state and the security model for that state is documented. Do not persist raw queries or validation tokens by default.

## What is explicitly not needed

- Flask or another web framework;
- an HTTP server;
- Docker;
- a database;
- a configuration framework;
- browser automation;
- a JavaScript runtime;
- a CAPTCHA-solving service;
- a search-index library;
- a general plug-in framework;
- a message queue.

## Version and packaging guidance

**Recommendation:** Target a currently supported Python version already used by the host application, preferably one with modern asyncio cancellation behavior. Pin lower and upper dependency bounds, run parser fixtures across supported platforms, and verify that any browser-impersonation wheel is available for the deployment OS.

Do not make the public search API expose library-specific response objects. Return project-owned dataclasses or typed dictionaries so the HTTP/parser dependency can change later.

## Dependency acceptance tests

Before choosing a library, verify:

- async cancellation actually closes the response;
- connection pools are reused;
- compressed responses are bounded after decompression;
- HTML parsing does not execute external entities or resources;
- Unicode and malformed input are handled;
- the library does not silently follow result URLs;
- Windows and deployment-platform installation works.
