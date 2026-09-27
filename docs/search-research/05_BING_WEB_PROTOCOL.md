# Bing Web Search protocol

## Endpoint strategy

| Item | Behavior |
| --- | --- |
| Endpoint | https://www.bing.com/search |
| Method | GET |
| Query encoding | Standard URL query encoding |
| Body | None |
| Expected response | Standard HTML result page |
| JavaScript | Not required for ordinary result extraction |
| Active pagination | Not advertised by the active adapter |
| Active time filter | Not advertised by the active adapter |

**Observation:** The active provider exposes general web search and safe search, but its capability flags do not expose pagination or time-range filtering. The future V1 should not claim those filters for Bing until a separate request/fixture investigation verifies them.

## Request parameters

The active search request contains:

| Parameter | Value |
| --- | --- |
| q | User query |
| adlt | off, moderate, or strict |
| setlang | Language portion of the selected market when a region is available |
| cc | Country portion for most countries; omitted for US, CN, and RU in the observed policy |

The active request does not use the helper-style market parameter mkt, although a separate locale helper can produce it. Do not add mkt merely because it exists in a utility function; executable request construction is authoritative.

### Safe-search mapping

| Generic option | Bing parameter |
| --- | --- |
| off | adlt=off |
| moderate | adlt=moderate |
| strict | adlt=strict |

### Region mapping

1. Resolve a provider region such as en-us or es-es.
2. Split the value into language and country.
3. Set setlang to the language.
4. Set cc to the country unless it is us, cn, or ru.
5. Omit cc for those excluded codes because the observed behavior treats them as undesirable Bing parameters.

The no-region fallback is the provider’s default; do not fabricate a country.

## Request fingerprint

### Active search request

The provider does not add a custom header set to its ordinary search request. The common transport/provider layer can add an Accept-Language value and a browser-profile header set.

| Field | Classification | Guidance |
| --- | --- | --- |
| User-Agent | RECOMMENDED | Use the transport’s coherent browser profile |
| Accept | RECOMMENDED | Browser-like HTML accept value |
| Accept-Language | RECOMMENDED | Derive from the caller locale |
| Referer | OPTIONAL | Not explicitly required by the active search path |
| Sec-Fetch-* | OPTIONAL | Let a coherent browser profile supply them if supported |
| DNT / Sec-GPC | OPTIONAL | Not required by the search request |
| Cookies | OPTIONAL | No search-specific cookie is required by the active path |
| Content-Type | NOT_NEEDED | GET with no body |
| Redirects | RECOMMENDATION: do not follow external redirects automatically | Decode result wrappers locally |

### Region discovery request

The optional startup discovery request is:

    GET https://www.bing.com/account/general

The observed header set is:

    User-Agent: generated browser-like value
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Accept-Language: en-US;q=0.5,en;q=0.3
    DNT: 1
    Connection: keep-alive
    Upgrade-Insecure-Requests: 1
    Sec-GPC: 1
    Cache-Control: max-age=0

This discovery request has a short timeout and is not part of every search if traits are already cached.

## Locale traits

The region map is built from links in a region-selection area of the account page. The parser reads the country query value from those links, records an all-locale sentinel clear, and maps supported languages to market strings. There are provider-specific aliases such as a Hong Kong Chinese mapping.

**Observation:** The stored Bing trait table contains region mappings but no general language mapping table comparable to Google’s language restriction map.

**Recommendation:** Keep a static common-market map and use the discovery page only as a bounded refresh. If discovery fails, use the locale’s language-country form when syntactically valid and record a warning.

## Result structure

The parser:

1. Locates the ordered result list with id b_results.
2. Selects list items carrying the b_algo result class.
3. Reads the title and destination from the h2 link.
4. Removes decorative span elements carrying the algoSlug_icon class from the paragraph.
5. Uses the remaining paragraph text as the snippet.

Items without a title or href are skipped. The result position is the ordinal of accepted result items.

### Current protocol observations: brittle selectors

The current structural selection is equivalent to:

    ol#b_results
      -> li whose class contains b_algo
         -> h2 -> a
         -> p, after removing span.algoSlug_icon

Keep these markers in sanitized HTML fixtures and add a live smoke test that detects a sudden zero-result parse.

## Redirect/wrapper decoding

Bing may return:

    https://www.bing.com/ck/a?u=a1<base64url-payload>

The observed recovery algorithm is:

1. Parse the wrapper URL query.
2. Read the u parameter.
3. Require the value to begin with a1.
4. Remove the a1 prefix.
5. Add one or more equals signs until the length is divisible by four.
6. Decode with URL-safe base64.
7. Decode bytes as UTF-8 using replacement for malformed byte sequences.
8. Require the result to be an absolute URL.

If the href is not the known wrapper, preserve it as-is after absolute-URL validation.

Synthetic examples:

    wrapper value: a1aHR0cHM6Ly9leGFtcGxlLnRlc3QvZG9jcz9wYWdlPTE
    payload:       aHR0cHM6Ly9leGFtcGxlLnRlc3QvZG9jcz9wYWdlPTE
    decoded:       https://example.test/docs?page=1

The payload above is synthetic and exists only to illustrate padding and decoding. A malformed wrapper must become a typed parse failure or item-level skip according to the failure policy; never issue a second network request to decode it.

## Anti-automation and failure behavior

The generic transport can identify status 403, 429, 503, explicit challenge HTML, and connection failure. Bing-specific parser behavior should additionally treat the following as suspicious:

- a response that is successful HTML but contains no result list and has a challenge/login marker;
- a result wrapper that decodes to an invalid destination;
- a sudden change from a normal result count to a tiny document with no result structure.

Do not solve challenges or retry rapidly. Return Blocked, CaptchaDetected, RateLimited, or ParseFailure with a provider cooldown recommendation.

## Pagination and time filters

**Observation:** The active adapter does not advertise page support or time-range support. The generic request preparation layer therefore rejects page values above one and time filters for Bing.

**Recommendation:** Treat later pages and time filters as USEFUL_LATER. Add them only after recording exact live requests and parser fixtures. Do not emulate them by appending undocumented parameters without tests.

## Minimal V1

### ESSENTIAL_V1

- One GET to the standard search endpoint.
- q and adlt construction.
- setlang and conservative cc region handling.
- coherent browser-like transport headers.
- b_results/b_algo extraction.
- title, snippet, and href extraction.
- ck/a wrapper decoding.
- typed block and parse failure.

### USEFUL_LATER

- Trait refresh from the account page.
- Verified Bing pagination.
- Verified Bing date filtering.
- Published-date extraction.

### NOT_NEEDED

- Bing account/session automation.
- JavaScript execution.
- Arbitrary URL fetching.

## Unknowns requiring live validation

- Whether the current HTML class names remain stable.
- Whether the excluded country list still has the same behavior.
- Which status/body combinations indicate a Bing CAPTCHA rather than a generic block.
- Whether the ck/a payload may contain a second encoding in future responses.
- Whether an endpoint-specific cookie improves reliability.
