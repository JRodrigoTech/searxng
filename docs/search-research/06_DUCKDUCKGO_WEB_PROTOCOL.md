# DuckDuckGo Web Search protocol

## Status and compatibility target

There are two distinct general-web implementations in the audited codebase. They must not be conflated:

1. **Primary HTML adapter** — enabled in the audited default configuration. It POSTs to the no-JavaScript HTML endpoint, extracts result blocks, and uses a query/User-Agent-bound `vqd` value for continuation pages.
2. **Secondary JSON/script adapter** — disabled in the audited default configuration. It first discovers a provider-generated `d.js` URL from the normal web page and then consumes JSON-like API responses. It has its own pagination cache and arithmetic challenge path.

For the three-provider tool, **PARITY V1 uses the primary HTML adapter**. The secondary adapter is documented so no useful behavior is lost and can later be implemented as an explicit fallback/alternate provider mode.

---

# A. Primary HTML adapter

## Endpoint and capabilities

| Item | Audited behavior |
| --- | --- |
| Endpoint | `https://html.duckduckgo.com/html/` |
| Method | POST |
| Body | `application/x-www-form-urlencoded` |
| Response | HTML |
| JavaScript | not required |
| Paging | yes, stateful after page 1 |
| Time range | yes |
| Locale/region | yes |
| Safe-search capability flag | true, although this request path does not add an explicit safe-search field |
| Maximum query length | 499 characters |

If `len(query) >= 500`, no network URL is produced and the provider is effectively skipped.

## Query preprocessing

Recognized external `!bang` tokens are quoted before submission to prevent an external redirect. The algorithm splits around whitespace, drops pure-whitespace pieces, wraps recognized bang tokens in single quotes, then rejoins tokens with one space.

This means whitespace in the submitted query can differ from the caller's original formatting. The transformed query is also the value used when storing/retrieving continuation state.

## Stable User-Agent and `vqd` identity

A generated browser User-Agent is created once at module/process initialization and reused by this adapter.

The continuation token cache key is a secret hash of:

    <transformed-query> + "//" + <User-Agent>

The token value expires after 3,600 seconds.

**PARITY MUST:** the User-Agent must remain stable while a token may be reused. Rotating the User-Agent independently between page 1 and later pages breaks the observed token identity.

The audited implementation stores this state in a provider-scoped persistent engine cache. An in-memory TTL cache can reproduce per-process behavior but is a deliberate persistence deviation and must be documented/tested if chosen for the embedded tool.

## Request headers

The adapter explicitly sets:

    User-Agent: <stable generated browser UA>
    Sec-Fetch-Dest: document
    Sec-Fetch-Mode: navigate
    Sec-Fetch-Site: same-origin
    Sec-Fetch-User: ?1
    Referer: https://html.duckduckgo.com/
    Content-Type: application/x-www-form-urlencoded

The common online layer normally supplies `Accept-Language` first. If it did not, the adapter adds its own fallback based on the selected locale.

## First-page form

The first-page form contains at least:

    q=<transformed query>
    b=
    kl=<region>

`vqd`, `nextParams`, `api`, `o`, `v`, `dc`, and `s` are not added on page 1.

### Region behavior

The all-region provider value is:

    wt-wt

For all-region:

    form kl=wt-wt
    no kl cookie

For a specific region:

    form kl=<provider-region>
    cookie kl=<provider-region>

Trait mapping must be used rather than constructing all region strings naively. The provider has explicit aliases and custom language-region mappings.

### Time range

| Input | `df` value |
| --- | --- |
| day | `d` |
| week | `w` |
| month | `m` |
| year | `y` |

When a supported time range exists, the adapter sets **both**:

    form df=<value>
    cookie df=<value>

With no supported time range, neither is added.

### Safe search

The adapter advertises safe-search support at the capability level, but the audited HTML request builder does not add a safe-search form field or cookie based on the numeric setting.

**PARITY MUST:** do not invent a safe-search request parameter for this path. If the host requires guaranteed strict filtering, that product policy must be layered separately and must not be described as source parity.

## Continuation pages

For page > 1, first retrieve the cached `vqd` for the transformed query and stable User-Agent.

If no token exists, raise the provider CAPTCHA/access-denied class with `suspended_time=0`; do **not** send a tokenless continuation request.

For locales whose internal locale string starts with `zh`, page > 1 produces no request URL and returns early. This encodes the observed provider limitation where continuation was unavailable/forbidden for those locales.

Continuation form fields are:

    q=<same transformed query>
    vqd=<cached token>
    nextParams=
    api=d.js
    o=json
    v=l
    s=<offset>
    dc=<offset + 1>
    kl=<region>

where:

    offset = 10 + (page - 2) * 15

Thus:

| Page | `s` | `dc` |
| ---: | ---: | ---: |
| 2 | 10 | 11 |
| 3 | 25 | 26 |
| 4 | 40 | 41 |

The optional `df` field/cookie is added exactly as on page 1 when a time range is selected.

## Response behavior

### Status 303

A response with status `303` returns an empty result collection immediately. It is not converted into an exception by this provider parser.

### CAPTCHA detection

Parse the response HTML. If:

    //form[@id="challenge-form"]

exists, raise the provider CAPTCHA/access-denied class with `suspended_time=0`.

This path detects but does not solve the HTML challenge.

### Token extraction

Find the parent element of an input named `vqd`. If present, take the first matching form, extract the first `vqd` value, and store it using:

    submitted q + "//" + submitted User-Agent

with a 3,600-second expiry.

Absence of a `vqd` input on a normal first-page response does not by itself fail parsing; it merely means no continuation token was cached.

### Main result extraction

Select only:

    //div[@id="links"]/div[contains(@class, "web-result")]

This intentionally excludes ad-style result blocks.

For each selected block, the current parser uses:

| Field | Selector |
| --- | --- |
| Title | `.//h2/a` text |
| URL | `.//h2/a/@href`, first value |
| Snippet | first `.//a[contains(@class, "result__snippet")]` |

The URL is used directly; this active HTML path does not apply a provider redirect-wrapper decoder.

Important failure detail: there is no per-result broad exception guard around every field extraction. A structurally malformed selected result that lacks the indexed URL can escape the response parser and become a provider-level failure through the common processor.

### Zero-click answer side channel

The parser also inspects:

    //div[@id="zero_click_abstract"]

When non-empty, it may emit an answer object unless the text contains diagnostic phrases including:

    Your IP address is
    Your user agent:
    URL Decoded:

This is a side channel, not a normal web result. The V1 Overmind-style tool may omit answer objects from its public result projection, but doing so is an output-surface decision; it should not alter normal result extraction.

## Trait acquisition

The provider trait builder sets all-region to `wt-wt` and fetches a provider JavaScript resource at a versioned URL to derive region/language mappings. The audited URL is:

    https://duckduckgo.com/dist/util/u.7669f071a13a7daa57cb.js

The module also contains explicit mapping exceptions such as traditional-Chinese, Catalan, Indonesian, Norwegian, Japanese, Korean, Arabic, Slovenian, Thai, and Vietnamese aliases.

For an embedded implementation, ship an equivalent generated trait snapshot and keep refresh out of the per-query critical path. Exact broad locale parity requires equivalent best-fit trait mapping, not simple string concatenation.

---

# B. Secondary JSON/script adapter

## Status

This is a separate general-web adapter and is **disabled in the audited default configuration**. It is nevertheless real, active code and must not be described as nonexistent.

Its purpose is to discover provider-generated JSON/script URLs that contain opaque parameters which cannot be reconstructed solely from `vqd`.

## First-page discovery

For queries shorter than 500 characters, first request:

    GET https://duckduckgo.com/?q=<query>&t=h_&ia=web

with:

    impersonate=firefox
    default_headers=false
    timeout=2

Parse the returned HTML and extract the href of:

    link#deep_preload_link

That URL points to the first `links.duckduckgo.com/d.js?...` page and includes provider-generated opaque state such as `dp`.

Cache the discovered page-1 URL under a query/page key for 7,200 seconds.

If no preload URL is found, the adapter produces no search URL for that operation.

## JSON/script request

Before requesting the discovered URL:

- replace `/d.js?` with `/d.js?o=json&`;
- set `impersonate=firefox`;
- set `default_headers=false`;
- add:

    Accept: */*
    Sec-Fetch-Dest: script
    Sec-Fetch-Mode: no-cors
    Sec-Fetch-Site: same-site
    Referer: https://duckduckgo.com/

This adapter does not currently implement safe-search, time-range, or locale traits in its request builder.

## Secondary pagination

For page > 1, the adapter does not reconstruct a URL. It loads a cached URL for exactly that query and requested page.

The JSON results contain a continuation path in field `n` on the last result. When present, the adapter caches:

    https://duckduckgo.com + <n>

for the next page with a one-hour expiry.

Consequently pages are sequentially stateful: requesting page 3 requires the page-3 URL to have been learned from page 2.

## JSON result parsing

Read:

    response.json()["results"]

For each item containing `u`:

    url     = item["u"]
    title   = HTML-to-text(item["t"])
    content = HTML-to-text(item["a"])

Items without `u` are skipped.

## Arithmetic challenge path

If the response text contains `let jsa =`, this adapter attempts a narrow provider-specific arithmetic challenge reconstruction. It parses a small set of generated arithmetic functions, calculates the expected numeric value using multiplication or fixed browser-HTML-length constants, constructs the provider follow-up URL, and performs another GET with the same Firefox-oriented request identity.

This is not arbitrary JavaScript execution; it is a hard-coded parser/evaluator for a small observed challenge grammar. Nevertheless it is an anti-automation challenge-response mechanism.

For **exact secondary-adapter parity**, this behavior belongs to the compatibility specification. For the primary three-provider V1, the secondary adapter is not required and this challenge path should not be implemented accidentally as part of the HTML adapter.

---

# C. Compatibility tests

## Primary adapter golden tests

- query length 499 produces a request; 500 produces none;
- recognized external bangs are quoted and whitespace is normalized;
- one stable generated User-Agent is reused;
- first page has `q`, `b`, `kl`, optional `df`, but no continuation fields;
- region-specific `kl` appears in form and cookie; `wt-wt` has no `kl` cookie;
- time range writes `df` to both form and cookie;
- page 2/3/4 offsets are 10/25/40 and `dc=s+1`;
- missing continuation `vqd` raises CAPTCHA/access-denied with zero suspension;
- `zh*` continuation produces no request;
- 303 returns an empty result list;
- `challenge-form` raises CAPTCHA/access-denied with zero suspension;
- hidden `vqd` caches for transformed-query + User-Agent with 3,600-second TTL;
- only `web-result` blocks are parsed;
- malformed indexed result fields can escalate to provider-level failure;
- zero-click diagnostic strings are excluded from answer output.

## Secondary adapter golden tests

- first page discovers `deep_preload_link`;
- first-page discovered URL caches for 7,200 seconds;
- `/d.js?` becomes `/d.js?o=json&`;
- Firefox impersonation and `default_headers=false` are set;
- script/no-cors/same-site headers are exact;
- result fields `u/t/a` map correctly;
- final result `n` seeds the next page for 3,600 seconds;
- arbitrary page skipping fails because the cached URL does not exist;
- arithmetic challenge fixtures reproduce the narrow audited computation.

## Primary V1 decision

The implementation agent should build the **HTML adapter** for the initial Google+Bing+DuckDuckGo tool. The JSON/script adapter is retained in this specification as a separately testable optional adapter, not as a hidden fallback.