# Independent implementation blueprint

## Objective

Build a small self-contained Python search capability for an existing AI runtime. The package must reproduce the audited behavior of three web providers—Google, Bing, and the primary DuckDuckGo HTML path—then expose merged/ranked results through one Tool call.

This blueprint distinguishes two layers:

- **V0 implementation path** — first-page, all-locale, no-time, no-safe-control, bounded `web.search(query, limit)` integrated into the current synchronous Overmind Tool contract.
- **Full compatibility reference** — the broader documented behavior covering locale traits, paging, DDG continuation, safe/time capability handling, and related state.

For the first coding pass, `18_SIMPLIFIED_V0_CONTRACT.md` is normative for feature scope and this document supplies package/build guidance.

## Recommended Overmind package shape

    overmind/tools/web_search/
        __init__.py
        tool.py
        models.py
        coordinator.py
        transport.py
        normalize.py
        aggregate.py
        errors.py
        providers/
            __init__.py
            google.py
            bing.py
            duckduckgo.py

Full-reference-only additions may later include:

    traits.py
    state.py
    providers/duckduckgo_json.py

Do not create a server, daemon, Docker service, browser process, generic crawler, or separate application framework.

## V0 public Tool API

The current Overmind Tool contract is synchronous. V0 therefore exposes conceptually:

    WebSearchTool.execute(
        {
            "query": str,
            "limit": int = 10,
        },
        cancellation=...
    ) -> dict

Tool identity:

    component_identity = "tools/web_search"
    name = "web.search"

Public V0 arguments are only:

    query
    limit

Fixed internal behavior:

    page = 1
    locale = "all"
    safe_search = 0
    time_range = None
    shared_timeout = 3.0
    provider weights = 1.0

The model must not be allowed to send raw provider parameters, cookies, provider names, browser profiles, page numbers, locale values, or time/safe fields in V0.

## Full compatibility API is later work

A later internal/full API may conceptually support:

    search(
        query,
        limit,
        locale,
        safe_search,
        time_range,
        page,
        timeout_limit,
    )

but do not implement/expose that surface merely because the compatibility documentation describes it. V0 deliberately avoids traits, Babel, DDG continuation requirements, and extra caller-controlled capability gates.

## Internal models

### SearchRequest

V0 fields:

    query
    limit

Internal constants supply the fixed V0 profile.

### PreparedRequest

Recommended fields:

    provider
    method
    url
    headers
    cookies
    query_params
    form_data
    allow_redirects
    browser_profile
    default_headers
    prefer_http3

### ProviderMainResult

Fields sufficient for parity behavior:

    url
    parsed_url
    title
    content
    thumbnail
    img_src
    provider
    template
    priority
    category

Defaults for ordinary V0 results:

    template = "default.html"
    priority = ""
    category = "general"
    img_src = ""
    thumbnail = ""

Keep `thumbnail` and `img_src` separate. Ordinary identity uses `img_src`; final grouping considers image presence.

### AggregatedResult

    url
    parsed_url
    title
    content
    thumbnail
    img_src
    primary_provider
    providers
    positions
    template
    category
    priority
    score

### ProviderDiagnostic

At minimum:

    provider
    status
    result_count
    elapsed_ms
    reason_code

Do not place raw bodies, cookies, tokens, full request URLs, or arbitrary exception reprs here.

### SearchResponse

Agent-facing projection:

    query
    results
    provider_diagnostics
    elapsed_ms

## Module responsibilities

### `tool.py`

Owns only:

- canonical Tool metadata/schema;
- public argument validation;
- cancellation check before execution;
- delegation to the coordinator;
- bounded JSON-serializable observation projection;
- optional low-cardinality event metadata.

Must not know provider selectors, cookies, ranking internals, or HTTP endpoints.

### `providers/google.py`

V0 owns:

- WML endpoint;
- fixed all-locale request values from `18`;
- exact safe=0 omission;
- fixed Nokia User-Agent set;
- per-request random Nokia UA selection;
- `chrome99_android` profile;
- Google-sorry detection;
- XML-declaration removal;
- exact XPath extraction;
- exact `/url?q=` split-before-unquote decoder;
- per-result exception isolation.

Full locale/time/page support belongs to later/full compatibility work.

### `providers/bing.py`

V0 owns:

- `/search` endpoint;
- `q` and `adlt=off` request shape;
- omission of `setlang`, `cc`, and `mkt` under fixed all-locale V0;
- exact result selectors;
- exact decorative icon removal;
- exact `ck/a?u=a1...` decoder;
- provider-level failure when a recognized malformed wrapper raises;
- provider transport preference for HTTP/3 when supported.

Do not invent paging/time support.

### `providers/duckduckgo.py`

V0 owns:

- query max-length guard;
- provider-specific whitespace collapse;
- public bang syntax rejection occurs in Tool validation, so no external bang table is required;
- stable Firefox-formatted generated User-Agent from exact OS/version corpus;
- HTML POST endpoint;
- exact Sec-Fetch/Referer/content-type headers;
- fixed `kl=wt-wt` all-locale form value;
- first-page empty `b` field;
- 303 healthy-empty behavior;
- challenge-form failure with zero explicit suspension;
- exact result selectors;
- hidden `vqd` capture/cache may be retained even though V0 does not consume it.

Later/full compatibility may add exact continuation fields and persistent TTL state.

### `transport.py`

V0 compatibility baseline:

    curl_cffi

Must support:

- provider-specific impersonation;
- explicit headers/cookies;
- TLS verification;
- redirects disabled for ordinary provider requests;
- common HTTP error classification;
- timeout based on remaining shared deadline;
- optional HTTP/3 preference for Bing;
- no accidental cross-provider cookie/session state.

A narrow synchronous wrapper is preferred for V0 because Overmind's Tool contract is synchronous and the compatibility semantics are naturally expressed with provider worker threads.

Do not make `httpx` the provider transport if doing so loses Google browser impersonation parity.

### `normalize.py`

Must reproduce exactly:

- whitespace collapse;
- title/content limits 200/1200;
- word-boundary ellipsis behavior;
- normalized content==title clearing;
- parsed URL construction;
- missing-scheme default to `http`;
- observed IDNA conversion behavior.

### `aggregate.py`

May contain both identity and aggregation logic for V0. Must reproduce:

- provider-local accepted position assignment;
- ordinary identity fields:

      template
      netloc
      path
      params
      query
      fragment
      img_src

- scheme exclusion from identity;
- global duplicate merge including same-provider duplicates;
- longer title/content selection;
- provenance union;
- secure-suffixed scheme preference;
- appended positions;
- exact score formula;
- stable score-descending sort;
- exact second grouping pass with `max_count=8`, `max_distance=20`.

Do not pre-collapse same-provider duplicates.

For collision safety, a structured identity tuple may replace raw Python integer-hash storage only as the documented internal deviation. The identity fields themselves must remain exact.

### `coordinator.py`

V0 must:

1. validate cancellation before dispatch;
2. record `start = monotonic()`;
3. establish one shared deadline `start + 3.0`;
4. filter temporarily suspended providers;
5. construct all provider workers;
6. start all eligible workers before waiting for any one of them;
7. poll/join with short bounded waits so Overmind cancellation remains responsive;
8. isolate provider failure;
9. reject all provider completion after deadline/cancellation admission closes;
10. normalize and merge only accepted outputs;
11. score and group;
12. apply caller `limit` only after ordering/grouping;
13. return bounded provider diagnostics.

Recommended V0 execution model: one synchronous coordinator plus worker threads around synchronous provider HTTP calls.

Do not create a new asyncio event loop per Tool call. A future async Tool runtime can replace the coordinator internals without changing provider/aggregate contracts.

## Healthy-empty versus failure

This distinction is mandatory.

Examples of healthy-empty:

- provider returns a valid page with zero ordinary results;
- primary DuckDuckGo returns HTTP 303 according to its documented adapter behavior.

Healthy-empty still counts as a completed provider, not a failed provider.

`SEARCH_UNAVAILABLE` should describe the case in which every eligible provider is failed, timed out, or suspended. If every provider completes successfully but returns zero results, return:

    ok = true
    results = []

## Provider capability matrix

### V0

| Capability | Google | Bing | Primary DuckDuckGo |
| --- | --- | --- | --- |
| Page 1 | yes | yes | yes |
| All-locale fixed mode | yes | yes | yes |
| Caller paging | no | no | no |
| Caller time filter | no | no | no |
| Caller safe setting | no | no | no |
| Arbitrary fetch | no | no | no |

### Full-reference behavior documented for later

| Capability | Google | Bing | Primary DuckDuckGo |
| --- | --- | --- | --- |
| Later pages | yes, max 50 | no | yes with cached state |
| Time filter | yes | no | yes |
| Safe capability | yes | yes | advertised but no explicit request mapping in audited HTML builder |
| Locale | language + region traits | region traits | region traits + Accept-Language |

## Exact V0 call sequence

    authorized Tool call web.search
        |
        v
    validate query + limit + unsupported bang syntax
        |
        v
    cancellation check
        |
        v
    search_start + 3.0 s shared deadline
        |
        v
    provider suspension filter
        |
        v
    start all eligible worker threads
        |
        +--> Google request -> parse
        +--> Bing request -> parse
        +--> DDG HTML request -> parse/capture optional vqd
        |
        v
    accept only in-deadline, non-cancelled provider outputs
        |
        v
    exact common normalization
        |
        v
    global identity merge in accepted insertion order
        |
        v
    exact score calculation
        |
        v
    stable score sort + exact grouping pass
        |
        v
    apply final caller limit
        |
        v
    bounded Tool observation

## Overmind integration points

The coding agent must update existing ownership locations rather than invent search-specific infrastructure:

- `overmind/extensions/builtin.py` — built-in Tool descriptor/factory;
- `overmind/runtime_stargate/production_policy.py` — exact first-party grant `("tools/web_search", "web.search")`;
- runtime dependency input/manifest/lock surfaces;
- CI dependency installation where explicitly enumerated;
- Tool/RuntimeStargate tests.

The Tool must not receive Runtime, Agent, Session, ContextCompiler, RuntimeStargate, registries, or service locators.

## Dependencies

V0 direct additions:

    curl_cffi
    lxml

Do not add Babel for fixed all-locale V0.

Use the target repository's established reproducible dependency workflow to update all packaging/lock surfaces. Do not hand-edit generated lock material when a regeneration command exists.

## Recommended construction order

1. read `18`, `19`, `20`, `21`;
2. write deterministic tests first;
3. models;
4. normalization;
5. identity/merge/rank/grouping;
6. common HTTP error classification;
7. narrow curl transport;
8. Google adapter;
9. Bing adapter;
10. primary DDG page-one adapter;
11. coordinator/deadline/cancellation;
12. Tool wrapper;
13. built-in registration + RuntimeStargate policy;
14. dependency closure/packaging updates;
15. optional live smoke tests;
16. final compatibility audit from `21_IMPLEMENTATION_DRY_RUN.md`.

## Definition of V0 implementation parity

V0 is not ready merely because it returns links. It must pass tests for:

- exact V0 provider request parameters and fingerprints;
- exact parser selectors/error granularity;
- common normalization;
- URL identity;
- duplicate merge;
- same-provider duplicate behavior;
- exact score formula;
- exact final grouping pass;
- one shared deadline;
- late-result rejection;
- cancellation;
- provider failure isolation and healthy-empty semantics;
- exact Overmind Tool registration/authorization;
- dependency closure in developer/CI/portable runtime.

## Non-goals for V0

Do not add:

- a server/UI;
- arbitrary page fetching;
- locale selector;
- paging selector;
- time filter;
- safe-search selector;
- external bang dataset;
- a search index;
- generic crawling;
- browser automation;
- JavaScript runtime;
- generic provider plugin framework;
- alternative ranking;
- secondary DDG adapter.

The V0 implementation should remain small, embedded, bounded, and testable while preserving the search mechanics that matter.