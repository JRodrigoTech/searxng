# Simplified V0 web-search contract

## Purpose

This document defines the first implementation profile for the target AI runtime. It deliberately implements a smaller surface than the full compatibility reference while preserving the audited provider mechanics that matter for first-page general web search, aggregation, deduplication, ranking, deadlines, and failure isolation.

The goal is that a coding agent can implement V0 without reopening the analyzed source repository.

This profile is intentionally strict: every omitted feature is named explicitly so that a simplification cannot silently mutate core search behavior.

## Public tool surface

Expose exactly one model-callable operation:

    web.search

with arguments conceptually equivalent to:

    query: string
    limit: integer = 10

Validation:

- `query` must be a string;
- `1 <= len(query) <= 499`;
- query must contain at least one non-whitespace character;
- `1 <= limit <= 10`;
- no additional arguments;
- tokens beginning with `!` are rejected in V0 because exact external-bang recognition depends on a separate provider-maintained bang table that is outside the V0 dependency set.

The query string is otherwise preserved for Google and Bing. DuckDuckGo applies its provider-specific whitespace preprocessing described below.

## Fixed internal search profile

V0 fixes these values internally:

    page = 1
    locale = all
    safe_search = 0
    time_range = None
    shared_timeout = 3.0 seconds
    provider weights = 1.0 for google, bing, duckduckgo

These values are not exposed to the model in V0.

Consequences:

- no locale trait snapshot is required for the first implementation;
- Babel is not required for V0;
- no DuckDuckGo continuation token is required because only page 1 is searched;
- no paging state is required;
- no time-filter state is required;
- no public safe-search control is claimed;
- all three providers can be dispatched for every valid ordinary V0 query unless suspended/failed.

## Provider set

V0 deliberately enables three adapters even though the audited application's default engine configuration does not enable all three simultaneously:

    google
    bing
    duckduckgo HTML

This is a target-product choice. Provider request/parse mechanics still follow the audited adapters.

## Common request language header

With the fixed `locale=all`, the generic locale layer has no parsed locale and uses the fallback:

    Accept-Language: en-US,en;q=0.9

Set this before provider-specific request mutation.

# Google V0

## Request

Method:

    GET

Endpoint:

    https://www.google.com/wml/search

Query parameters for V0:

    q=<original query>
    sca_esv=1
    hl=en
    lr=
    cr=
    ie=utf8
    oe=utf8

Absent in V0:

    start
    tbs
    safe
    num

The empty `lr` is the audited all-locale behavior. The Google language fallback is `lang_en`, which produces `hl=en`. The all-region trait is present, so `cr` is initialized to an empty value because `all` contains no explicit region.

## Fingerprint

Choose one Nokia User-Agent independently for each Google request from the exact six-value set in `04_GOOGLE_WEB_PROTOCOL.md`.

Transport profile:

    chrome99_android

Redirect following:

    false

Do not inject the helper-only `CONSENT=YES+` cookie or `Accept: */*` as if the audited web request used them; the audited request builder does not merge those helper dictionaries into this path.

## Parsing and failure

Use exactly the selectors, block isolation, Google-sorry detection, XML-declaration handling, and `/url?q=` split-before-unquote behavior defined in `04_GOOGLE_WEB_PROTOCOL.md`.

# Bing V0

## Request

Method:

    GET

Endpoint:

    https://www.bing.com/search

Query parameters:

    q=<original query>
    adlt=off

Because V0 locale is `all`, the provider region resolves to its all-region sentinel and therefore V0 sends neither:

    setlang
    cc

It also never sends:

    mkt

No page or time parameters are invented.

## Fingerprint

Use the common Chrome-family browser impersonation profile. Preserve the provider network preference for HTTP/3 when the selected transport implementation supports the same direct/no-proxy conditions; otherwise record HTTP/2 as an explicit transport deviation rather than changing request parameters.

Redirect following:

    false

## Parsing and failure

Use the exact `b_results/b_algo` selectors, paragraph/icon handling, wrapper decoding, and malformed-wrapper provider-failure boundary defined in `05_BING_WEB_PROTOCOL.md`.

# DuckDuckGo V0

## Query preprocessing

The primary HTML adapter always passes the query through its bang-preprocessing function. Even when V0 rejects bang syntax, that function's whitespace behavior still matters: it splits on whitespace and rejoins non-empty tokens with one ordinary space.

Therefore V0 DuckDuckGo sends:

    "  python   asyncio\ttaskgroup  "

as:

    "python asyncio taskgroup"

Google and Bing still receive the original validated query string.

## Stable User-Agent generation

The audited generator is:

    Mozilla/5.0 ({os}; rv:{version}) Gecko/20100101 Firefox/{version}

where `os` is chosen from:

    Windows NT 10.0; Win64; x64
    X11; Linux x86_64

and `version` is chosen independently from:

    154.0
    153.0

The primary DuckDuckGo adapter generates this value once at provider/module initialization and reuses it.

**V0 MUST:** generate one value once when the DuckDuckGo provider object/service is constructed and reuse it for every request during that runtime lifetime. Do not rotate it per query.

## Request

Method:

    POST

Endpoint:

    https://html.duckduckgo.com/html/

Form fields:

    q=<DuckDuckGo-preprocessed query>
    b=
    kl=wt-wt

Absent in V0:

    df
    vqd
    nextParams
    api
    o
    v
    s
    dc

Cookies:

    none from kl/df in the all-region/no-time V0 profile

Headers:

    User-Agent: <stable generated value>
    Accept-Language: en-US,en;q=0.9
    Sec-Fetch-Dest: document
    Sec-Fetch-Mode: navigate
    Sec-Fetch-Site: same-origin
    Sec-Fetch-User: ?1
    Referer: https://html.duckduckgo.com/
    Content-Type: application/x-www-form-urlencoded

Transport browser profile remains the common Chrome-family profile; the explicit HTTP User-Agent is the stable Firefox-formatted value above. Do not “fix” this mismatch.

Redirect following:

    false

## Response

Use the exact primary HTML behavior in `06_DUCKDUCKGO_WEB_PROTOCOL.md`:

- 303 -> healthy empty result collection;
- `form#challenge-form` -> zero-duration CAPTCHA/access-denied failure;
- cache hidden `vqd` if present even though V0 does not consume it yet; retaining it costs little and makes a later paging extension straightforward;
- parse only `div#links > div.web-result`;
- preserve provider-level structural failure behavior;
- zero-click answer is not exposed in the V0 public result projection.

The V0 implementation may keep the captured `vqd` only in process memory. It is not required for page-one behavior.

# Concurrency and deadline

## Host-compatible execution model

The target runtime's Tool interface is synchronous. V0 SHOULD therefore use a synchronous coordinator with concurrent worker threads instead of adding a private event-loop lifecycle merely to make the internal API async.

This is compatible with the audited search semantics because those semantics are themselves worker-thread based.

Recommended structure:

    start = monotonic()
    deadline = start + 3.0

    construct three provider workers
    start all three workers before waiting for any

    each worker:
        prepare exact provider request
        perform provider HTTP call
        parse result
        if not timed out/cancelled:
            insert into shared aggregator under lock

    coordinator:
        wait using remaining = max(0, deadline - monotonic())
        mark still-running workers timed out
        ignore all late insertion attempts

A worker's network call may still finish after the caller stops waiting. Its output must not enter the aggregate after the timeout marker is set.

## Cancellation

The Tool receives the runtime `CancellationToken`.

V0 MUST:

- check cancellation before launching workers;
- poll cancellation while waiting rather than blocking for the whole three-second timeout in one uninterruptible join;
- on cancellation, mark all unfinished workers rejected/late and propagate the runtime cancellation exception;
- never let a worker that finishes after cancellation mutate the accepted result set.

Threads do not need to be forcibly killed; result acceptance is the authority boundary.

# Normalization, identity, merge, score, grouping

V0 does **not** simplify these stages.

Apply the exact behavior from `09`, `10`, and `11`:

1. normalize result fields before identity;
2. provider-local accepted main-result positions start at 1;
3. use the exact ordinary identity string components:

       template|netloc|path|params|query|fragment|img_src

   and Python hash semantics for exact parity if desired;
4. do not pre-deduplicate within a provider;
5. same-provider and cross-provider duplicates use one global map;
6. merge longer title/content, provenance, default fields, secure-suffixed scheme, and append every position;
7. with V0 weights all equal to 1.0, ordinary score is:

       len(positions) * sum(1 / position for position in positions)

8. stable sort descending score with no explicit tie-break;
9. run the exact category/template/image grouping pass with `max_count=8` and `max_distance=20`;
10. apply caller `limit` only after grouping.

For the three V0 providers, template is ordinary/default and category is the general/web family. Google thumbnails can still make image-marker grouping observable.

# Failure isolation and suspension

V0 keeps provider failure isolation: one failed provider must not erase successful siblings.

Implement provider health in process memory using the exact active durations from `12_FAILURE_AND_RESILIENCE.md`:

    generic failure            5 s
    access denied            180 s
    CAPTCHA                 3600 s
    too many requests        180 s
    Cloudflare CAPTCHA   1296000 s
    Cloudflare firewall    86400 s
    reCAPTCHA              604800 s

Primary DuckDuckGo missing-vqd/challenge paths use explicit zero suspension. Missing-vqd cannot occur in page-one V0, but challenge-form can.

Persistence across process restart is not required for V0. That is an operational deviation only; within one runtime, suspension behavior must be exact.

# Public result

A successful tool observation should be bounded and structurally simple:

    {
      "ok": true,
      "query": "...",
      "results": [
        {
          "title": "...",
          "url": "...",
          "snippet": "...",
          "score": 5.5,
          "providers": ["google", "bing"],
          "positions": [1, 3],
          "thumbnail": "..." | null
        }
      ],
      "providers": {
        "google": {"status": "ok", "result_count": 8},
        "bing": {"status": "failed", "reason": "..."},
        "duckduckgo": {"status": "ok", "result_count": 7}
      },
      "elapsed_ms": 412
    }

Provider-derived fields are untrusted data. Do not expose cookies, raw HTML, validation tokens, proxy information, or challenge bodies.

If all providers fail, return a bounded failed/empty observation rather than inventing results.

# Explicit V0 deviations from the full compatibility reference

V0 deliberately does **not** implement:

- page > 1;
- caller-selectable locale;
- caller-selectable safe search;
- caller-selectable time range;
- complete locale trait tables;
- Babel-based best-fit locale resolution;
- external bang lookup tables;
- secondary DuckDuckGo JSON/script adapter;
- persistent provider cache across runtime restart;
- a public suggestions/answers side channel;
- arbitrary result-page fetching.

None of those omissions changes the page-one all-locale aggregation/ranking logic implemented by V0.

# V0 definition of done

V0 is implementation-ready when a coding agent can build it using only this documentation set and pass:

- exact provider request-vector tests;
- provider parser synthetic fixtures;
- exact normalization/dedupe/ranking/grouping tests;
- shared-deadline tests;
- cancellation/late-result tests;
- failure-isolation/suspension tests;
- Overmind registration/authorization tests;
- optional live smoke tests from the intended deployment network.
