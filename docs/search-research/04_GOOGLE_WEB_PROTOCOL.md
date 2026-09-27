# Google Web Search protocol

## Status and compatibility target

This document is the implementation specification for the audited Google web-search path. Statements marked **PARITY MUST** describe behavior that must be reproduced by the new search tool unless a later change is explicitly documented as a deliberate deviation.

The target is behavioral compatibility, not a redesign. Do not silently add URL validation, different safe-search semantics, different browser fingerprints, different redirect handling, or different parser recovery rules and still call the implementation compatible.

## Endpoint and capabilities

| Item | Audited behavior |
| --- | --- |
| Endpoint | `https://www.google.com/wml/search` |
| Method | GET |
| Query encoding | `urllib.parse.urlencode` semantics |
| Body | none |
| Response | XML-like mobile markup parsed as tolerant HTML |
| JavaScript | not required |
| Paging | yes |
| Maximum page | 50 |
| Time range | yes: day/week/month/year |
| Language/region | yes, through trait mappings |
| Safe search | yes |
| Redirect following | false in the ordinary online-engine path |

The mobile/legacy representation is intentional. The desktop web representation is not the audited path.

## Request construction

The request begins with:

    q=<query>
    sca_esv=1

Provider locale logic adds:

    hl=<interface-language>
    lr=<language-restriction>
    cr=<country-restriction when applicable>
    ie=utf8
    oe=utf8

### Language and country rules

`hl` is derived from the provider language trait after removing the `lang_` prefix. `lr` receives the complete provider language trait, for example `lang_en`; when the selected locale is the all-locale value, `lr` is the empty string.

When a country trait exists, `cr` is initialized to an empty string. It becomes `country<COUNTRY>` only if the caller locale contains a region component. The web path does not actively send `gl`.

The trait data includes provider-specific aliases. In the audited Google trait builder, generic `zh` maps to `lang_zh-CN`, the all-region sentinel is `ZZ`, and `zh-CN` has a region alias to `HK`. A compatible implementation must use an equivalent trait snapshot/mapping rather than deriving all values mechanically from ISO codes.

### Pagination

Pages are one-based:

    start = (page - 1) * 10

`start` is omitted when the value is zero, so page one sends no `start` parameter.

### Time filter

| Input | Request parameter |
| --- | --- |
| day | `tbs=qdr:d` |
| week | `tbs=qdr:w` |
| month | `tbs=qdr:m` |
| year | `tbs=qdr:y` |

Unsupported or absent time ranges do not add `tbs`.

### Safe-search behavior

The provider defines the conceptual mapping:

    0 -> off
    1 -> medium
    2 -> high

However, request construction adds `safe` only when the numeric setting is truthy. Therefore the **actual** request behavior is:

| Setting | Exact request behavior |
| ---: | --- |
| 0 | omit `safe` |
| 1 | `safe=medium` |
| 2 | `safe=high` |

**PARITY MUST:** do not send `safe=off` for setting 0.

### Result-count parameter

`num` is not sent. The audited code explicitly leaves it disabled because it was observed not to affect this representation.

## HTTP fingerprint

### Explicit User-Agent

Each request chooses one value at random from this fixed set:

    Nokia7610/2.0 (5.0509.0) SymbianOS/7.0s Series60/2.1 Profile/MIDP-2.0 Configuration/CLDC-1.0
    Nokia7610/2.0 (7.0642.0) SymbianOS/7.0s Series60/2.1 Profile/MIDP-2.0 Configuration/CLDC-1.0
    Nokia6230/2.0 (05.50) Profile/MIDP-2.0 Configuration/CLDC-1.1
    Nokia6230i/2.0 (03.80) Profile/MIDP-2.0 Configuration/CLDC-1.1
    Nokia6280/2.0 (03.60) Profile/MIDP-2.0 Configuration/CLDC-1.1
    NokiaN72/2.0617.1.0.3 Series60/2.8 Profile/MIDP-2.0 Configuration/CLDC-1.1

### Browser/TLS impersonation

The request explicitly selects the transport profile:

    chrome99_android

This produces an intentionally unusual combination: a Nokia HTTP User-Agent together with an Android Chrome 99 curl/TLS impersonation profile. This is what the audited implementation does.

**PARITY MUST:** preserve this combination for compatibility testing. Replacing the transport with stock `httpx` or `aiohttp` without equivalent browser impersonation is a behavioral change, not a transparent refactor.

### Accept-Language

The generic online request layer adds `Accept-Language` before the engine-specific request builder when locale headers are enabled. The forms are:

    <lang>,<lang>-<territory>;q=0.7,en;q=0.3

or, when no parsed locale exists:

    en-US,en;q=0.9

### Important helper-only values

The locale helper computes these additional values:

    Accept: */*
    CONSENT=YES+

but the audited Google web request builder does **not** merge the helper's `headers` or `cookies` dictionaries into the outgoing request parameters. Only the helper's query parameters are merged, followed by the explicit Nokia `User-Agent` and `chrome99_android` impersonation setting.

**PARITY MUST:** do not claim `Accept: */*` or `CONSENT=YES+` are sent by this exact path unless the future implementation deliberately changes the behavior and records that deviation.

## Redirect policy and CAPTCHA detection

The generic online request defaults to:

    allow_redirects = false
    max_redirects = 0
    raise_for_httperror = true

Before result parsing, the provider performs these checks:

1. response host equals `sorry.google.com` -> CAPTCHA/access-denied failure;
2. response path starts with `/sorry` -> CAPTCHA/access-denied failure;
3. status code is `302` -> CAPTCHA/access-denied failure;
4. response text is shorter than 2,000 characters and contains `/sorry/` -> CAPTCHA/access-denied failure.

HTTP statuses `>= 400` are normally intercepted by the common HTTP error layer before provider parsing. A 302 is not a generic HTTP error in that layer, which is why the provider-specific 302 check matters.

## Response parsing

If the response, after left whitespace, begins with an XML declaration, remove everything through the first `?>`. Parse the remaining text with tolerant HTML parsing.

### Result blocks

Current selectors are:

| Purpose | Selector |
| --- | --- |
| Result block | `//div[contains(@class, "zMzFAb")]` |
| Title element | `.//a[contains(@class, "fuLhoc")]//span[contains(@class, "CVA68e")]` |
| Raw URL | `.//a[contains(@class, "fuLhoc")]/@href` |
| Snippet | `.//div[contains(@class, "taTFJ")]//span[contains(@class, "FrIlee")]` |
| Thumbnail | `.//img[contains(@src, "encrypted-tbn")]/@src` |
| Suggestions | table class `HExoMb`, link class `ZWRArf` |

Per block:

1. if the title element is missing, skip the block;
2. extract title text;
3. if the title href is missing, skip the block;
4. unwrap the Google URL when applicable;
5. extract snippet text, defaulting effectively to an empty string;
6. extract the first matching thumbnail, defaulting to an empty string;
7. emit a main result;
8. catch any exception from that individual block, log it, skip the block, and continue parsing later blocks.

That per-item exception isolation is provider-specific and must be preserved.

## Exact URL unwrapping

The current transformation is semantically:

    if raw_url starts with "/url?q=":
        encoded = raw_url after the first 7 characters
        encoded = encoded split on literal "&sa=U", first part only
        return URL-decode(encoded)
    return raw_url unchanged

The operation order is therefore:

1. remove `/url?q=`;
2. split on literal `&sa=U` **before decoding**;
3. keep the first part;
4. call URL percent-decoding;
5. return the result.

Example:

    /url?q=https%3A%2F%2Fexample.test%2Fdocs%3Fx%3D1%26y%3D2&sa=U&ved=x

becomes:

    https://example.test/docs?x=1&y=2

**PARITY MUST:** the provider does not add an absolute-URL or HTTP(S)-scheme validation after this unwrap. Such validation can be added at a host security boundary only as a clearly documented deliberate deviation.

## Trait acquisition

The provider can build traits from:

    https://www.google.com/preferences

with a short bounded request. It reads interface-language options and country options, applies aliases, and persists the resulting trait dataset outside the critical search path in the larger application.

For the embedded tool, equivalent behavior can be achieved by shipping a generated trait snapshot and optionally refreshing it out of band. Search execution must not depend on a live trait scrape for every query.

## Compatibility tests

At minimum, golden tests must assert:

- page 1 omits `start`; page 2 sends `start=10`;
- safe-search 0 omits `safe`; 1 sends `medium`; 2 sends `high`;
- all four `tbs` values;
- all-locale produces empty `lr`;
- Nokia User-Agent is selected only from the audited set;
- transport profile is `chrome99_android`;
- redirects are disabled;
- 302 and all three `/sorry` signatures are classified as CAPTCHA/access denial;
- XML declaration removal;
- each current XPath selector;
- missing title/href skips one block rather than failing the provider;
- redirect unwrapping splits before percent-decoding;
- `Accept: */*` and `CONSENT=YES+` are not asserted as outgoing values for this exact web path.

## Live validation boundary

External behavior is mutable. Live smoke tests should verify only whether the endpoint still accepts this protocol and returns the expected structural markers. If live behavior changes, update fixtures and this protocol document before changing implementation behavior.