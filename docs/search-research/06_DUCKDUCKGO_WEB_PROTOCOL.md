# DuckDuckGo Web Search protocol

## Active path and alternatives

The ordinary active path is the no-JavaScript HTML interface:

| Item | Active behavior |
| --- | --- |
| Endpoint | https://html.duckduckgo.com/html/ |
| Method | POST |
| Body | application/x-www-form-urlencoded |
| Response | HTML document |
| JavaScript | Not required |
| First page | Does not require a validation token |
| Later pages | Require a query/User-Agent-bound validation token |
| Redirect wrapper | None observed in the active HTML result links |

Other public paths are present in the analyzed code but are not the active V1 path:

- https://links.duckduckgo.com/d.js for a JavaScript/JSON-style response.
- https://noai.duckduckgo.com/ as another public endpoint.
- https://lite.duckduckgo.com/lite as a lightweight HTML alternative.

**Recommendation:** Implement the HTML POST path first. Treat the JSON-style path as a separate experiment, not as a silent fallback, because it has a different state machine and an anti-automation challenge path.

## State machine

    NEW QUERY
       |
       v
    choose stable process-level User-Agent
       |
       v
    build first-page browser-like POST
       |
       v
    HTML response
       |
       +--> challenge form
       |       |
       |       +--> CaptchaDetected; no solver
       |
       +--> result blocks
       |       |
       |       +--> return first-page results and cache vqd if present
       |
       +--> empty/redirect status
               |
               +--> return an explicit empty or degraded outcome

    LATER PAGE
       |
       v
    look up token by query + User-Agent
       |
       +--> absent
       |       |
       |       +--> do not send a tokenless page request
       |             return a typed validation/CAPTCHA failure
       |
       +--> present
               |
               v
          build tokenized POST
               |
               v
          parse result blocks or classify challenge

**Observation:** The token is cached after parsing a hidden input named vqd from the form containing the results. It is not a general session token.

## Query constraints and first-page request

Queries with length 500 characters or more are rejected by the active adapter before a network request. The V1 adapter should return UnsupportedQuery or InvalidRequest rather than truncating the user query.

The first-page form contains:

| Field | Value |
| --- | --- |
| q | Query text, with recognized external bang tokens quoted in the observed path |
| b | Empty string |

The first request has no vqd, nextParams, api, o, v, dc, or s fields.

The query is form-encoded. Do not build the body by string concatenation.

## Later-page request

For page values greater than one, the observed form contains:

| Field | Value |
| --- | --- |
| q | The same query text, after the provider’s bang-quoting step |
| vqd | Cached validation token |
| nextParams | Empty string |
| api | d.js |
| o | json |
| v | l |
| s | offset |
| dc | offset plus one |
| offset | 10 + (page - 2) * 15 |

Examples:

| Page | offset | dc | s |
| --- | ---: | ---: | ---: |
| 2 | 10 | 11 | 10 |
| 3 | 25 | 26 | 25 |
| 4 | 40 | 41 | 40 |

**Observation:** The adapter uses fifteen-result increments after the initial offset of ten. This is provider behavior, not a generic page-size assumption.

**Observation:** For locales beginning with zh, the active path suppresses pages beyond one because the observed endpoint returned HTTP/2 403 or omitted the next-page control. Keep this as a locale-specific capability restriction until live behavior changes.

## Token state

### Cache key and lifetime

The cache key is derived from:

    query + "//" + stable_user_agent

The stored key is a secret hash rather than the raw concatenated value. The token lifetime is approximately 3,600 seconds. A future implementation should also store the creation time and discard a token on a validation failure.

### Required invariants

- A token must not be reused for another query.
- A token must not be reused with another User-Agent.
- A token must not be sent after its TTL.
- A missing token for a later page is a provider-state failure, not a reason to send a tokenless request.
- The token must never appear in normal logs, telemetry labels, or the returned tool response.

### Process User-Agent

The active adapter creates a User-Agent at import/startup and reuses it. This is materially different from selecting a fresh User-Agent for every request: the token relationship depends on stable identity.

**Recommendation:** Choose one coherent browser identity per provider instance or process. If rotation is needed later, rotate the User-Agent together with token state and cooldown state.

## Headers and cookies

The active HTML request uses the following provider-specific fields:

| Field | Value/policy | Classification |
| --- | --- | --- |
| User-Agent | Stable generated browser-like value | REQUIRED for token consistency |
| Sec-Fetch-Dest | document | RECOMMENDED |
| Sec-Fetch-Mode | navigate | RECOMMENDED |
| Sec-Fetch-Site | same-origin | RECOMMENDED |
| Sec-Fetch-User | ?1 | RECOMMENDED |
| Referer | Initial origin https://html.duckduckgo.com/, then the HTML endpoint | RECOMMENDED |
| Content-Type | application/x-www-form-urlencoded | REQUIRED |
| Accept-Language | Locale-derived value if not already set by common transport | RECOMMENDED |
| Cookies | kl for region; df for time filter | REQUIRED when those filters are selected |

The active request sets the Referer to the endpoint after the initial setup. A future implementation should use the exact endpoint with its trailing slash consistently.

The common request layer may add an Accept-Language value before the provider adds its fallback. Observed fallback construction is:

    language, language-UPPERCASE; q=0.7

For an all-locale request, a generic layer may still add an English fallback. The code comments and the executable header path are not fully aligned here; classify this as a live-validation item rather than an invariant.

## Locale and traits

DuckDuckGo region traits are derived from a provider JavaScript resource:

    https://duckduckgo.com/dist/util/u.7669f071a13a7daa57cb.js

The startup fetch has a short timeout and parses region and language sections to create:

- language-to-region mappings;
- region-to-language mappings;
- a custom language-region mapping for provider-specific combinations;
- all-locale sentinel wt-wt.

The active HTML request uses the resolved region in the kl form value. For the all-locale value, the form contains kl=wt-wt and no kl cookie is added. For a non-default region it also sends kl as a cookie. Language behavior is primarily carried by Accept-Language and the trait selection rather than by a verified independent HTML form language field.

The broader provider code contains related language-region cookie values named ad, ah, and l for other flows and special region/bang handling. They are not populated by the ordinary active HTML search path described here. Do not add them to V1 without a separate request/response fixture proving that they affect ordinary web results.

Common observed examples include:

| Caller locale | Provider-style region |
| --- | --- |
| en-US | us-en |
| es-ES | es-es |
| zh-CN | cn-zh |
| zh-TW | tw-tzh |
| no locale | wt-wt |

These values are observations from a trait snapshot. Use a provider mapping table, not string concatenation, for production.

## Time filtering

When a time range is present, set the df cookie:

| Generic option | df value |
| --- | --- |
| day | d |
| week | w |
| month | m |
| year | y |

No time cookie is sent for an unset filter. The V1 request should reject unsupported time values before building the form.

## Safe search

**Observation:** The active HTML request advertises safe-search support at the adapter capability level, but the observed request construction does not add a verified safe-search form parameter or cookie.

**Recommendation:** Do not claim strict safe-search semantics for DuckDuckGo V1 until a live request and fixture establish the correct field. Options:

1. expose safe search as provider-agnostic intent and mark DuckDuckGo as best effort;
2. omit DuckDuckGo when strict safe search is required;
3. add a separately tested provider field once verified.

Never silently claim that the provider applied a filter it did not receive.

### Current protocol observations: brittle selectors

The active parser currently relies on:

    form#challenge-form
    input[name="vqd"]
    div#links > div whose class contains web-result
    h2 > a
    a.result__snippet

Ad-style blocks are excluded by selecting web-result rather than the ad classes.

## Response handling

1. A 303 response is treated as an empty outcome by the observed path.
2. Parse the HTML document.
3. If a form with id challenge-form exists, return CaptchaDetected and do not retry immediately.
4. Find the form containing an input named vqd and extract the token.
5. Store the token with its query/User-Agent key and TTL.
6. Select result blocks under the container with id links and the web-result class.
7. Read title from h2/a.
8. Read snippet from the result__snippet link.
9. Read destination from the direct href.
10. Exclude ad-style result classes by selecting only web-result blocks.

An optional zero-click abstract may be available. The observed path adds it as an answer only when it is non-empty and does not contain diagnostic text such as “Your IP address is”, “Your user agent:”, or “URL Decoded:”. V1 should omit answer objects from the main SearchResult list or return them through a separate optional field.

## Anti-automation behavior

Known indicators:

- challenge-form in the returned HTML;
- missing vqd for a requested later page;
- HTTP 403 or similar access denial on a continuation request;
- a tiny/non-result response that contains provider challenge text;
- a token rejected after previously working.

The active documentation suggests that blocking may be related to the source IP as well as request/session characteristics. This is an inference, not a guarantee.

Correct behavior:

- classify the outcome as CaptchaDetected, Blocked, or RateLimited;
- record a cooldown;
- return other providers’ results;
- expire the suspect token;
- do not solve a challenge;
- do not rotate headers repeatedly in a tight loop;
- do not disclose challenge HTML to the AI caller.

## Optional JSON-style path

The inactive alternative path has a different state machine:

1. GET https://duckduckgo.com/?q=...&t=h_&ia=web.
2. Use a Firefox-like browser profile and default header generation disabled.
3. Extract a preload script link with id deep_preload_link.
4. Cache the continuation URL for the exact query and page.
5. Convert the d.js query to an o=json request for later pages.
6. Parse result fields u, t, and a; cache a next path from n.

Its request fingerprint includes Accept */*, script/no-cors/same-site fetch fields, and a DuckDuckGo Referer. The path also contains code that evaluates a small arithmetic challenge embedded in a response. That challenge-solving behavior is explicitly NOT_NEEDED for V1 and must not be copied into the new subsystem. A challenge should instead become a typed provider failure.

## Minimal V1 and later work

### ESSENTIAL_V1

- HTML POST first page.
- Query length validation.
- Stable User-Agent.
- Required form fields and browser-like headers.
- kl region and df time cookie.
- challenge-form detection.
- vqd extraction, bounded cache, and tokenized later-page request if pagination is enabled.
- web-result extraction.

### USEFUL_LATER

- More complete trait refresh.
- Verified safe-search field.
- Locale-specific pagination policy.
- Optional zero-click answer object.
- Separate, tested JSON-style adapter.

### NOT_NEEDED

- JavaScript execution.
- Arithmetic/challenge solving.
- External bang redirect behavior.
- Arbitrary result-page fetching.

## Unknowns requiring live validation

- Whether the current token TTL remains one hour.
- Whether the token is accepted across processes or only within the originating session/IP.
- Whether a browser profile must match the stable User-Agent byte-for-byte.
- Whether kl and df remain the correct cookie names and values.
- Whether safe-search is configurable on the current HTML endpoint.
- Whether the Chinese pagination restriction is still necessary.
