# Independent implementation blueprint

## Proposed package shape

The following shape is a recommendation for a small self-contained package. It is intentionally functional and does not mirror any other application’s layout.

    web_search/
        __init__.py
        models.py
        coordinator.py
        transport.py
        locales.py
        urls.py
        ranking.py
        errors.py
        state.py
        telemetry.py
        providers/
            __init__.py
            base.py
            google.py
            bing.py
            duckduckgo.py

The package can be renamed to match the host project. The responsibilities and boundaries are the important part.

## Public API

Conceptual entry point:

    async search(
        query: str,
        *,
        limit: int = 10,
        language: str | None = None,
        region: str | None = None,
        safe_search: SafeSearch = SafeSearch.MODERATE,
        time_range: TimeRange | None = None,
        page: int = 1,
        providers: Sequence[str] | None = None,
        timeout: float = 4.0,
    ) -> SearchResponse

The exact enum syntax may change with the host project. The API must make filters explicit and must not expose raw provider parameter dictionaries.

## Type-level pseudocode

    enum SafeSearch:
        OFF
        MODERATE
        STRICT

    enum TimeRange:
        DAY
        WEEK
        MONTH
        YEAR

    SearchRequest:
        query
        limit
        language
        region
        safe_search
        time_range
        page
        providers
        timeout

    ProviderCapabilities:
        supports_paging
        max_page
        supports_time_range
        supports_language
        supports_region
        supports_safe_search

    ProviderRequest:
        provider
        method
        url
        query
        form
        headers
        cookies
        follow_redirects
        browser_profile
        deadline

    ProviderResult:
        provider
        title
        url
        snippet
        position
        published_at
        thumbnail
        metadata

    ProviderOutcome:
        provider
        results
        failure
        diagnostics

    ResultObservation:
        provider
        position
        url
        title
        snippet

    SearchResult:
        title
        url
        snippet
        providers
        observations
        score
        published_at
        thumbnail
        metadata

    SearchResponse:
        query
        results
        providers
        degraded
        elapsed_ms

## Module responsibilities

### models.py

Responsibility:

- public request/response types;
- provider result and observation types;
- safe-search and time-range enums;
- serialization-safe field bounds.

Public API:

- SearchRequest;
- SearchResponse;
- SearchResult;
- ProviderDiagnostic;
- enum values.

Dependencies:

- standard typing/dataclasses or the host’s existing model system.

Must not know:

- HTML selectors;
- transport client classes;
- provider cookies;
- ranking internals.

Tests:

- validation;
- serialization;
- bounds;
- enum mapping.

### errors.py

Responsibility:

- typed provider failures;
- retryability and phase;
- safe diagnostic codes.

Public API:

- ProviderFailure;
- FailureKind;
- SearchInputError;
- UnsupportedCapability.

Dependencies:

- standard exceptions and enums.

Must not know:

- raw response bodies;
- provider parser structure.

Tests:

- classification;
- redaction;
- stable diagnostic serialization.

### transport.py

Responsibility:

- one async client/pool;
- GET/POST with query/form fields;
- bounded body reading;
- TLS, compression, redirects, retries;
- deadline propagation.

Public API:

- SearchTransport;
- TransportResponse;
- request method accepting ProviderRequest.

Dependencies:

- one selected async HTTP library.

Must not know:

- result classes;
- provider challenge signatures;
- URL identity or ranking.

Tests:

- fake server request shape;
- body limits;
- retry/deadline;
- cancellation;
- connection reuse;
- close behavior.

### locales.py

Responsibility:

- parse caller locale;
- map it separately for Google, Bing, and DuckDuckGo;
- provide all-locale fallback;
- expose unsupported/fallback diagnostics.

Public API:

- ParsedLocale;
- map_google_locale;
- map_bing_locale;
- map_duckduckgo_locale.

Dependencies:

- standard parser or optional locale library;
- static provider trait data.

Must not know:

- HTTP client;
- HTML parser.

Tests:

- common locales;
- script tags;
- country aliases;
- fallback behavior.

### urls.py

Responsibility:

- provider wrapper decoding;
- absolute URL validation;
- display URL normalization;
- canonical identity construction.

Public API:

- unwrap_provider_url;
- validate_result_url;
- canonical_identity;
- choose_display_url.

Dependencies:

- urllib.parse;
- optional IDNA support from the standard library.

Must not know:

- ranking;
- provider HTML selectors;
- network access.

Tests:

- every wrapper example;
- conservative identity table;
- malformed/hostile URLs;
- collision-safe lookup.

### state.py

Responsibility:

- provider-scoped TTL state;
- DuckDuckGo token keying by query/User-Agent;
- cooldown/circuit state;
- concurrency protection.

Public API:

- TokenStore;
- ProviderCircuit;
- StateSnapshot.

Dependencies:

- standard dict, hashlib, time, asyncio lock.

Must not know:

- HTML structure;
- how a token is extracted;
- result ranking.

Tests:

- TTL;
- key isolation;
- concurrent reads/writes;
- invalidation;
- cooldown transitions.

### providers/base.py

Responsibility:

- common adapter protocol;
- capability description;
- provider outcome interface.

Public API:

- SearchProvider protocol;
- ProviderCapabilities;
- ProviderContext.

Dependencies:

- models, errors, transport abstractions.

Must not know:

- any provider selector or cookie name.

Tests:

- fake adapter contract;
- unsupported capability handling.

### providers/google.py

Responsibility:

- Google endpoint and query construction;
- mobile User-Agent/profile policy;
- locale, safe-search, time, and page mapping;
- sorry/CAPTCHA detection;
- HTML/XML-like parser;
- /url?q= decoding.

Dependencies:

- base contract;
- locale mapper;
- HTML parser;
- URL decoder;
- transport.

Must not know:

- Bing or DuckDuckGo state;
- global ranking;
- coordinator task management.

Tests:

- all Google cases in 14_TEST_STRATEGY.md.

### providers/bing.py

Responsibility:

- Bing endpoint and q/adlt construction;
- setlang/cc market mapping;
- HTML parser;
- ck/a base64url decoding;
- block/invalid-wrapper classification.

Dependencies:

- base contract;
- locale mapper;
- HTML parser;
- URL decoder;
- transport.

Must not know:

- other provider cookies or tokens;
- global ranking.

Tests:

- request mappings, fixtures, wrapper decoding, malformed inputs.

### providers/duckduckgo.py

Responsibility:

- HTML POST first-page request;
- kl/df cookie fields;
- stable User-Agent;
- token extraction and later-page request;
- TTL state lookup/invalidation;
- challenge-form detection;
- web-result parsing.

Dependencies:

- base contract;
- locale mapper;
- state store;
- HTML parser;
- transport.

Must not know:

- global merge/rank policy;
- CAPTCHA-solving behavior;
- arbitrary page fetching.

Tests:

- first page, later pages, cache isolation, challenge, query limit, fixtures.

### coordinator.py

Responsibility:

- validate request;
- establish total deadline;
- prepare provider requests;
- run provider tasks;
- collect partial outcomes;
- normalize and merge;
- invoke ranking;
- enforce final limit.

Public API:

- SearchCoordinator.search;
- top-level search function.

Dependencies:

- models, errors, provider registry, transport, URL policy, ranker, telemetry.

Must not know:

- provider selectors;
- provider-specific form fields;
- how redirects are decoded.

Tests:

- all concurrency and partial-failure cases;
- deterministic completion-order independence.

### ranking.py

Responsibility:

- provider-local duplicate collapse;
- provenance-aware merge;
- score calculation;
- deterministic sort and final cap.

Public API:

- merge_results;
- rank_results;
- score_result.

Dependencies:

- models;
- URL identity policy.

Must not know:

- network or HTML.

Tests:

- observed compatibility score;
- recommended V1 score;
- tie-breaking and consensus cap.

### telemetry.py

Responsibility:

- structured provider metrics;
- redaction and low-cardinality labels.

Public API:

- ProviderMetrics;
- record_provider_outcome.

Dependencies:

- host logger/metrics interface.

Must not know:

- raw body details;
- ranking internals except counts.

Tests:

- redaction;
- field bounds;
- no token/cookie leakage.

## Full call sequence

    caller
      |
      v
    web_search.search(...)
      |
      v
    validate SearchRequest
      |
      v
    start = monotonic()
    deadline = start + timeout
      |
      v
    for each enabled provider:
        capabilities()
        locale mapping
        prepare ProviderRequest
      |
      v
    create one async task per eligible provider
      |
      +--> Google:
      |      GET mobile endpoint
      |      inspect block status
      |      parse blocks
      |      unwrap destination
      |
      +--> Bing:
      |      GET standard endpoint
      |      inspect block status
      |      parse b_algo blocks
      |      decode ck/a destination
      |
      +--> DuckDuckGo:
             POST HTML form
             inspect challenge-form
             extract/cache vqd
             parse web-result blocks
      |
      v
    collect ProviderOutcome values until deadline
      |
      v
    normalize text and URLs
      |
      v
    collapse provider-local duplicates
      |
      v
    merge cross-provider identities
      |
      v
    calculate recommended score
      |
      v
    deterministic sort and final limit
      |
      v
    SearchResponse

## V1 construction order

1. Implement models and errors.
2. Implement URL validation and identity with tests.
3. Implement fake transport and coordinator deadline tests.
4. Implement Google request/parser with fixtures.
5. Implement Bing request/parser with fixtures.
6. Implement DuckDuckGo first-page HTML path.
7. Add token state and later pages behind a capability flag.
8. Add merge/rank and provenance.
9. Add telemetry and redaction tests.
10. Run optional live smoke tests from a controlled environment.

## Non-goals for the implementation agent

- Do not create a server.
- Do not create a web interface.
- Do not implement a browser.
- Do not implement a CAPTCHA solver.
- Do not fetch result pages.
- Do not add unrelated providers before the three fixed adapters are tested.
- Do not copy source structure or source code from any analyzed project.
