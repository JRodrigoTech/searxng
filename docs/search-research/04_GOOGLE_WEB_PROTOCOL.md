# Google Web Search protocol

## Scope

This specification covers the public, non-API representation used for ordinary web results. It intentionally does not depend on a JavaScript-rendered search page or a commercial search API.

**Observation:** The normal desktop representation is JavaScript-heavy. The selected mobile/legacy representation is preferred because it returns a parseable result document without running JavaScript and is therefore suitable for a bounded server-side request.

## Endpoint strategy

| Item | Behavior |
| --- | --- |
| Endpoint | https://www.google.com/wml/search |
| Method | GET |
| Query encoding | Standard URL query encoding |
| Body | None |
| Expected response | XML-like mobile HTML; parse as HTML after removing an optional XML declaration |
| JavaScript | Not required |
| Redirect following | Disabled in the active provider path |
| First-page size | Provider emits approximately ten standard results in the observed layout |
| Maximum page | Adapter advertises page values through 50 |

**Observation:** The endpoint is described as an XML result representation, but the parser uses an HTML tree and class-based structural selection. The implementation must therefore accept HTML-compatible markup rather than require a strict XML document.

**Recommendation:** Make the endpoint and parser selectors configurable constants inside the Google adapter, with fixture tests. Do not generalize them into a shared parser.

## Request construction

The active request has these query parameters:

| Parameter | Value |
| --- | --- |
| q | User query |
| sca_esv | 1 |
| hl | Provider language code |
| lr | Provider language restriction, or empty for the all-locale fallback |
| cr | country plus provider country code when a country is selected; empty otherwise |
| ie | utf8 |
| oe | utf8 |
| start | (page - 1) * 10, omitted for the first page |
| tbs | qdr:d, qdr:w, qdr:m, or qdr:y for day/week/month/year |
| safe | off, medium, or high when safe search is nonzero |

num is deliberately omitted. **Observation:** the analyzed request path contains a note that num has no effect for this representation.

### Time mapping

| Generic option | Google parameter |
| --- | --- |
| day | tbs=qdr:d |
| week | tbs=qdr:w |
| month | tbs=qdr:m |
| year | tbs=qdr:y |

### Safe-search mapping

| Generic option | Google parameter |
| --- | --- |
| off | omit safe or use safe=off according to the adapter’s normalized policy |
| moderate | safe=medium |
| strict | safe=high |

**Observation:** The numeric search setting maps 0 to off, 1 to medium, and 2 to high. The V1 public API should map named values to these strings before request construction.

## Request fingerprint

| Field | Observed value/policy | Classification |
| --- | --- | --- |
| User-Agent | Random choice from a fixed Nokia mobile User-Agent set | REQUIRED for the selected representation; exact availability is an external assumption |
| Accept | */* | RECOMMENDED |
| Cookie | CONSENT=YES+ | RECOMMENDED; helps avoid a consent interstitial |
| Accept-Language | Added by the generic provider layer when enabled; locale-specific form | RECOMMENDED |
| Browser profile | Android Chrome 99 impersonation | RECOMMENDED for matching the expected mobile request |
| Referer | Not explicitly set | OPTIONAL |
| Sec-Fetch-* | Not explicitly set by the provider | UNKNOWN; browser profile may supply defaults |
| DNT / Sec-GPC | Not explicitly set | OPTIONAL |
| Content-Type | No body; not applicable | NOT_NEEDED |
| Redirect policy | Do not follow redirects | REQUIRED for the observed CAPTCHA detection |

The fixed User-Agent values are legacy Nokia/Symbian profiles. The currently observed set is:

    Nokia7610/2.0 (5.0509.0) SymbianOS/7.0s Series60/2.1 Profile/MIDP-2.0 Configuration/CLDC-1.0
    Nokia7610/2.0 (7.0642.0) SymbianOS/7.0s Series60/2.1 Profile/MIDP-2.0 Configuration/CLDC-1.0
    Nokia6230/2.0 (05.50) Profile/MIDP-2.0 Configuration/CLDC-1.1
    Nokia6230i/2.0 (03.80) Profile/MIDP-2.0 Configuration/CLDC-1.1
    Nokia6280/2.0 (03.60) Profile/MIDP-2.0 Configuration/CLDC-1.1
    NokiaN72/2.0617.1.0.3 Series60/2.8 Profile/MIDP-2.0 Configuration/CLDC-1.1

A future implementation may use this exact verified list as configuration, but it must not generate an unrelated random browser identity for each header independently.

When the generic locale layer emits Accept-Language, the observed forms are:

    language,language-territory;q=0.7,en;q=0.3
    en-US,en;q=0.9

The first form is used when a parsed locale is available; the second is the fallback when it is not.

## Locale and traits

The provider uses a locale trait table built from Google’s preferences page:

1. Request https://www.google.com/preferences with a short startup timeout.
2. Read language options from the hl selector.
3. Read country options from the gl selector.
4. Map caller locales to the provider’s language and country values.
5. Use a special all-locale sentinel when no specific locale is selected.

**Observation:** The active request uses:

- hl for interface language;
- lr for language restriction;
- cr for country restriction;
- no active gl parameter, even though country traits are collected;
- ZZ as the all-locale trait sentinel.

The observed country transformation is cr=country followed by the provider country code for a language-country locale. The Chinese mapping contains a provider-specific alias in the trait table; do not derive it from the ISO country code without consulting the trait map.

**Recommendation:** Ship a small static map for common locales and refresh traits asynchronously as an optional enhancement. Never make a search wait indefinitely for a preferences scrape.

## URL examples

First page, English, United States, strict safe search:

    https://www.google.com/wml/search?q=example+query&sca_esv=1&hl=en&lr=lang_en&cr=countryUS&ie=utf8&oe=utf8&safe=high

Second page with a week filter:

    https://www.google.com/wml/search?q=example+query&sca_esv=1&hl=en&lr=lang_en&cr=countryUS&ie=utf8&oe=utf8&start=10&tbs=qdr:w

The exact query-string ordering is not semantically important; tests should compare decoded parameter maps rather than raw ordering.

## Response inspection and anti-automation detection

Inspect the response before parsing results:

1. If the final response host is sorry.google.com, classify it as CaptchaDetected.
2. If the final response path begins with /sorry, classify it as CaptchaDetected.
3. If redirects are disabled and the HTTP status is 302, classify it as a challenge/CAPTCHA response rather than treating the Location as a result.
4. If the body is smaller than roughly 2,000 bytes and contains /sorry/, classify it as a CAPTCHA response.
5. A normal HTTP error status should become an appropriate typed HTTP, blocked, or rate-limit failure.

**Observation:** These checks are intentionally conservative and do not solve a challenge.

**Recommendation:** Keep the size heuristic as a warning-level signal unless a fixture demonstrates that it is specific enough. Do not log the full challenge body.

## Result structure

The observed parser selects standard result blocks using a div class containing the provider’s current result-block marker. Within each block:

- The title is taken from the result link, using the current title-link and title-span classes.
- The destination is the link href.
- The snippet is taken from the result description container and its description span.
- The first image whose source contains the provider’s encrypted thumbnail marker is an optional thumbnail.

The parser skips a block when title or destination is missing. Item-level errors are isolated so one malformed block does not discard the whole response.

### Current protocol observations: brittle selectors

The currently observed structural markers are:

| Purpose | Current marker |
| --- | --- |
| Result block | div whose class contains zMzFAb |
| Title link | link carrying fuLhoc |
| Title text | span carrying CVA68e |
| Snippet container | div carrying taTFJ |
| Snippet text | span carrying FrIlee |
| Thumbnail | img source containing encrypted-tbn |
| Suggestions | table carrying HExoMb, links carrying ZWRArf |

These are public document markers, not stable API fields. Keep them in a dedicated adapter fixture and monitor them with a live smoke test.

## Redirect unwrapping

When the extracted link begins with /url?q=:

1. Remove the /url?q= prefix.
2. URL-decode the remainder.
3. Stop before the provider tracking suffix beginning with &sa=U.
4. Require the recovered value to be an absolute URL.

If the link is already absolute and is not a provider wrapper, preserve it. If unwrapping fails, skip the item or report a parse warning; never use the provider wrapper as the canonical identity.

Synthetic example:

    /url?q=https%3A%2F%2Fexample.test%2Fdocs%3Fx%3D1%26y%3D2&sa=U&ved=abc
    ->
    https://example.test/docs?x=1&y=2

## Pagination

The page number is one-based. The offset is:

    start = (page - 1) * 10

Omit start for page one. The adapter advertises a maximum page of 50, but the V1 coordinator should normally request only the first page unless the caller explicitly asks for more.

## Minimal V1 and later work

### ESSENTIAL_V1

- One GET to the mobile representation.
- Query, locale, safe-search, and optional time-range mapping.
- Fixed mobile User-Agent policy.
- Accept, consent cookie, and browser-profile transport setting.
- Redirect disabled.
- CAPTCHA/sorry detection.
- Current result-block/title/snippet/thumbnail extraction.
- /url?q= destination recovery.

### USEFUL_LATER

- Trait refresh and locale coverage beyond common locales.
- Later pages.
- Suggestions.
- Published dates if a stable source field is identified.

### NOT_NEEDED

- JavaScript browser automation.
- A commercial API key.
- Search suggestions as main results.
- Challenge solving.

## Unknowns requiring live validation

- Whether the mobile representation remains available from the deployment IP range.
- Whether the fixed Nokia User-Agent set continues to receive the same markup.
- Whether a 302 always indicates a challenge in every network edge.
- Whether Google introduces additional result classes or consent pages.
- Whether the provider’s country alias table changes.
