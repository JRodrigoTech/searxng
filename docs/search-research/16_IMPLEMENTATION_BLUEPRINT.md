# Independent implementation blueprint

## Objective

Build a small self-contained Python search package for an existing AI runtime. The package must reproduce the audited behavior of three web providers—Google, Bing, and the primary DuckDuckGo HTML path—then expose the merged/ranked results through one tool call.

The package is independent in naming and structure, but its provider request, parsing, normalization, deduplication, merge, ranking, deadline, and failure semantics are compatibility requirements.

## Proposed package shape

    search/
        __init__.py
        models.py
        coordinator.py
        transport.py
        traits.py
        state.py
        normalize.py
        identity.py
        aggregate.py
        errors.py
        telemetry.py
        providers/
            __init__.py
            base.py
            google.py
            bing.py
            duckduckgo.py
            # optional later:
            duckduckgo_json.py

Names are generic and project-owned. Do not mirror a larger application's internal module/class hierarchy.

## Public tool API

Conceptual API:

    async search(
        query: str,
        *,
        limit: int = 10,
        locale: str = "all",
        safe_search: int = 0,
        time_range: str | None = None,
        page: int = 1,
        timeout: float | None = None,
    ) -> SearchResponse

Suggested accepted values:

    safe_search: 0 | 1 | 2
    time_range: None | "day" | "week" | "month" | "year"
    page: one-based

The public API should not expose raw provider query parameters or cookies.

## Compatibility mode is the default implementation target

The first implementation should have one behavior mode only: compatibility. Avoid shipping a parallel “improved” ranking/URL-normalization path until the parity suite passes.

Host hardening can exist at explicit boundaries, for example:

    provider parity pipeline
       -> agent-output URL policy
       -> tool serialization

Do not mix those concerns inside provider parsers or ranking.

## Internal models

### SearchRequest

    query
    limit
    locale
    safe_search
    time_range
    page
    timeout_limit

### PreparedRequest

    provider
    method
    url
    headers
    cookies
    form/data
    json/content
    allow_redirects
    browser_profile
    default_headers
    network flags

### ProviderMainResult

Use fields sufficient to reproduce common normalization/identity:

    url
    parsed_url
    title
    content
    thumbnail
    img_src
    provider/engine
    template
    priority

### AggregatedResult

    url
    parsed_url
    title
    content
    thumbnail
    img_src
    primary_provider
    providers: set
    positions: list[int]
    template
    category
    priority
    score

Optionally retain an auxiliary labelled observation list for telemetry/explainability, but do not replace the compatibility `positions` list used by scoring.

### SearchResponse

Agent-facing projection:

    query
    results
    provider_diagnostics
    elapsed_ms

Each result may expose:

    title
    url
    snippet
    score
    providers
    positions
    thumbnail

All provider text remains untrusted data.

## Module responsibilities

### `providers/google.py`

Must own:

- WML endpoint;
- locale query mappings;
- exact safe/time/page parameter conditions;
- fixed Nokia User-Agent set;
- `chrome99_android` profile;
- Google-sorry detection;
- XML-declaration removal;
- exact XPath extraction;
- exact `/url?q=` split-before-unquote decoder;
- per-result exception isolation.

Must not own global dedupe/ranking.

### `providers/bing.py`

Must own:

- `/search` endpoint;
- `q/adlt/setlang/cc` mapping;
- no `mkt` for the ordinary web path;
- exact result selectors;
- decorative icon removal;
- exact `ck/a?u=a1...` decoder;
- the fact that malformed recognized base64 can fail the provider;
- HTTP/3 provider transport flag.

Must not invent pagination/time support.

### `providers/duckduckgo.py`

Must own:

- query-length guard;
- external-bang quoting/whitespace behavior;
- stable generated User-Agent;
- HTML POST endpoint;
- exact Sec-Fetch/Referer/content-type headers;
- `kl` region form/cookie behavior;
- `df` time form/cookie behavior;
- first-page `b` field;
- continuation `vqd/nextParams/api/o/v/s/dc` fields;
- `vqd` state keyed by transformed query + UA, TTL 3600;
- Chinese continuation suppression;
- 303 empty behavior;
- `challenge-form` CAPTCHA behavior with zero suspension;
- exact result and zero-click selectors.

Do not add an unverified safe-search field.

### optional `providers/duckduckgo_json.py`

Only after primary V1 parity passes. It may implement the separately documented preload-link/JSON path, including Firefox profile, page URL cache, sequential pagination, and narrow arithmetic challenge behavior.

### `transport.py`

Compatibility baseline: `curl_cffi` async clients.

Must support:

- provider-specific impersonation;
- explicit default-header enable/disable;
- HTTP/2 and conditional HTTP/3;
- TLS verification;
- pooled clients keyed by material transport settings;
- explicit cookies with no accidental jar carryover;
- redirects disabled for ordinary provider calls;
- common HTTP error classification;
- one shared remaining-time budget.

### `traits.py`

Must provide equivalent Google/Bing/DDG provider mappings from a generated snapshot and best-fit locale logic. Broad locale support requires more than splitting BCP-47 strings.

Trait refresh is optional tooling, not a per-query dependency.

### `state.py`

Must provide provider-scoped TTL state with safe concurrent access.

For primary DDG:

    secret/query-UA key -> vqd, 3600 s

If persistence is omitted in favor of process memory, record that as an explicit implementation deviation.

### `normalize.py`

Must reproduce:

- exact whitespace collapse;
- title/content limits 200/1200;
- word-boundary ellipsis;
- content==title clearing;
- parsed URL construction;
- missing-scheme `http` behavior;
- observed IDNA conversion.

### `identity.py`

Must reproduce ordinary identity fields:

    template
    netloc
    path
    params
    query
    fragment
    img_src

Scheme is excluded.

A structured tuple/string can replace a raw Python integer hash for collision safety only if documented as an internal deviation that produces identical ordinary-case equivalence.

### `aggregate.py`

Must reproduce:

- provider-local accepted position assignment;
- global duplicate merge (including same-provider duplicates);
- longer title/content selection;
- provenance union;
- secure-suffixed scheme preference;
- exact score formula;
- score-descending stable sort;
- exact second grouping pass with `max_count=8`, `max_distance=20`.

Do not pre-collapse same-provider duplicates.

### `coordinator.py`

Must:

1. record search start;
2. resolve eligible providers;
3. skip suspended/unsupported providers;
4. derive one shared timeout;
5. start all eligible providers concurrently;
6. accept only results completed before the deadline;
7. isolate provider failures;
8. normalize/merge in accepted provider completion order for parity-sensitive ties;
9. close/score/order;
10. apply the final caller result limit;
11. project results through any explicit host output policy.

A native asyncio coordinator is preferred; it need not reproduce source worker threads internally.

## Provider capability matrix

| Capability | Google | Bing | Primary DuckDuckGo |
| --- | --- | --- | --- |
| First page | yes | yes | yes |
| Later pages | yes, max 50 | no | yes with cached state |
| Time filter | yes | no | yes |
| Safe capability | yes | yes | advertised, no explicit request mapping in audited HTML builder |
| Locale | language + region traits | region traits | region traits + Accept-Language |
| JS runtime | no | no | no |

The coordinator should mimic the common capability gate: when a global requested option is unsupported, that provider is skipped rather than receiving invented parameters.

## End-to-end call sequence

    ToolRegistry / authorized search call
        |
        v
    SearchRequest validation
        |
        v
    search_start + actual_timeout
        |
        v
    capability / suspension filter
        |
        v
    launch eligible provider tasks concurrently
        |
        +--> Google request -> parse
        +--> Bing request -> parse
        +--> DDG HTML request -> parse/state update
        |
        v
    accept only in-deadline provider outputs
        |
        v
    common normalization
        |
        v
    global identity merge in insertion order
        |
        v
    score on close
        |
        v
    score sort + category/template/image grouping
        |
        v
    final result limit
        |
        v
    host output URL/security policy
        |
        v
    bounded SearchResponse -> agent

## Recommended construction order

1. models + parity fixtures;
2. curl transport and deadline tests;
3. common text/URL normalization;
4. identity + merge + exact rank/grouping;
5. trait snapshot/best-fit mapping;
6. Google adapter;
7. Bing adapter;
8. primary DDG first-page adapter;
9. DDG token state + continuation;
10. coordinator/partial failures/suspension;
11. agent-facing serialization and host policy;
12. optional live smoke tests;
13. optional secondary DDG adapter.

This order lets the core aggregation semantics be validated independently from volatile live provider HTML.

## Definition of implementation parity

The implementation is not considered ready merely because all three providers return links. It must pass golden tests for:

- exact provider request parameters and fingerprints;
- exact parser selectors/error granularity;
- common normalization;
- URL identity;
- duplicate merge;
- same-provider duplicate behavior;
- exact score formula;
- final grouping pass;
- shared deadline/late-result rejection;
- suspension behavior;
- trait mapping for tested locales.

## Non-goals

Do not add in V1:

- a server/UI;
- arbitrary page fetching;
- a search index;
- generic crawling;
- browser automation;
- a general JavaScript runtime;
- provider auto-discovery;
- alternative ranking hidden behind the default path.

The implementation should be small, embedded, and testable while preserving the mechanics that make the audited providers work.