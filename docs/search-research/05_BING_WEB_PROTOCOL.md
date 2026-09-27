# Bing Web Search protocol

## Status and compatibility target

This document specifies the behavior that the new search tool must reproduce for the Bing web path. The goal is request/parser compatibility, including its current limitations and failure behavior.

## Endpoint and capabilities

| Item | Audited behavior |
| --- | --- |
| Endpoint | `https://www.bing.com/search` |
| Method | GET |
| Query encoding | `urllib.parse.urlencode` semantics |
| Body | none |
| Response | HTML |
| JavaScript | not required for the extracted first-page results |
| Paging | not exposed by this web adapter |
| Time range | not exposed by this web adapter |
| Safe search | yes |
| Region | yes |
| HTTP/3 | enabled for this provider when transport conditions allow it |

The generic capability gate skips this provider when a caller asks for page > 1 or a time range, because this adapter does not advertise those capabilities.

## Request construction

The request always contains:

    q=<query>
    adlt=<safe-search-value>

Safe-search mapping is exact:

    0 -> off
    1 -> moderate
    2 -> strict

Unknown numeric values fall back to `off`.

### Region behavior

The provider trait resolves a region string such as `en-us` or `es-es`. If the resolved value is empty or equals the all-region sentinel `clear`, no locale parameters are added.

Otherwise:

1. split the region on the first `-`;
2. send the language part as `setlang`;
3. send the country part as `cc` unless the country is `us`, `cn`, or `ru`.

The code intentionally omits `cc` for those three country values because they were observed to produce poor/junk behavior.

A separate helper can construct `mkt=<language-country>` for other Bing engines, but the ordinary web request does **not** call that helper. **PARITY MUST:** do not add `mkt` to this path.

## Transport fingerprint

The engine itself does not add a custom search-header set. It receives common online-request headers, including locale-derived `Accept-Language` when enabled.

The transport baseline uses browser impersonation with the default browser profile and supports HTTP/2. This provider additionally sets `enable_http3 = true`; when there is no proxy and the transport supports it, the provider-specific network selects HTTP/3. With a proxy, it falls back according to the common transport rules.

Redirect following remains disabled by the generic online request defaults unless explicitly changed elsewhere.

### Trait-discovery request

Region traits can be refreshed from:

    GET https://www.bing.com/account/general

with a five-second timeout and this explicit header shape:

    User-Agent: generated browser-like value
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Accept-Language: en-US;q=0.5,en;q=0.3
    DNT: 1
    Connection: keep-alive
    Upgrade-Insecure-Requests: 1
    Sec-GPC: 1
    Cache-Control: max-age=0

The trait parser extracts region links, records `clear` as the all-region sentinel, derives official-language market strings, and contains a provider-specific `zh-hk -> en-hk` alias.

For the embedded tool, use an equivalent generated snapshot on the critical path and refresh it out of band if desired.

## Response parser

Parse the HTML document and iterate exactly these standard result blocks:

    //ol[@id="b_results"]/li[contains(@class, "b_algo")]

For each block:

1. select the first `.//h2/a`;
2. if no link exists, skip the block;
3. read `href` and the extracted text title;
4. if either href or title is empty, skip the block;
5. if the href is the known Bing wrapper, attempt the exact decoding rule below;
6. collect all `.//p` elements;
7. remove descendants exactly matching `span[@class="algoSlug_icon"]` from those paragraph trees;
8. extract the remaining paragraph text as content;
9. append the result.

The parser returns legacy dictionary-shaped results containing `url`, `title`, and `content`; the common result layer subsequently normalizes them.

## Exact wrapper decoding

Only hrefs beginning exactly with:

    https://www.bing.com/ck/a?

enter the wrapper path.

The algorithm is:

1. parse the wrapper query;
2. read the first `u` value;
3. if there is no `u`, leave the original href unchanged;
4. if `u` does not begin with `a1`, leave the original href unchanged;
5. otherwise remove the first two characters (`a1`);
6. append `=` padding until the payload length is divisible by four;
7. URL-safe-base64 decode;
8. decode the bytes as UTF-8 using `errors="replace"`;
9. replace `href` with that decoded string.

### Important failure semantics

There is no local `try/except` around base64 decoding in this parser. A malformed `a1` payload can therefore raise an exception that escapes the provider parser. The common online processor catches the exception and records the provider as failed; it is **not** an item-level skip in the current behavior.

There is also no post-decode requirement that the recovered string be absolute or use HTTP(S). Likewise, a direct non-wrapper href is not subjected to an absolute-URL check in this adapter.

**PARITY MUST:** if exact compatibility is the target, do not silently convert malformed-wrapper failure into a per-item skip and do not add absolute-URL rejection inside the provider parser. A host-level security validation may be layered later only as a deliberate, tested deviation.

## Empty and malformed pages

The provider-specific parser has no explicit CAPTCHA or zero-result structural detector. It simply emits whatever matching `b_algo` blocks it can parse. Generic HTTP handling catches HTTP-level access denial, rate limiting, CAPTCHA signatures recognized globally, and other HTTP errors before this parser runs.

Therefore a successful HTTP response with no `b_algo` blocks returns an empty result list rather than a provider-specific `ParseFailure` from this adapter.

## HTTP error interaction

The common transport/error layer handles status errors before parsing. In particular:

- 402/403 -> access denied;
- 429 -> too-many-requests/rate-limit failure;
- recognized Cloudflare/reCAPTCHA patterns -> CAPTCHA/access-denied failure;
- other status >= 400 -> underlying HTTP error.

The online processor catches these failures and isolates them from sibling providers.

## Compatibility tests

Golden tests should assert:

- exact `q` and `adlt` mapping;
- `setlang` behavior;
- `cc` omission for `us`, `cn`, `ru`;
- no `mkt` in this web request;
- page > 1 and time range are capability-rejected before request execution;
- exact `b_results/b_algo` selection;
- missing link/href/title skips the item;
- exact `algoSlug_icon` removal;
- valid `ck/a?u=a1...` decoding including missing base64 padding;
- no absolute-URL validation after decoding;
- malformed `a1` base64 escapes the parser and becomes provider-level failure;
- an HTTP-200 page with zero matching result blocks yields an empty list;
- provider network configuration requests HTTP/3 when allowed.

## Live validation boundary

Periodically verify the HTML selectors, `ck/a` wrapper format, region exclusions, and transport acceptance. Changes in public markup should update the compatibility fixtures and this document before changing implementation behavior.