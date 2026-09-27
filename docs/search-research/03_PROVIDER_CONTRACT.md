# Provider contract

## Purpose

The provider contract isolates volatile external-search protocol details from the coordinator while preserving the exact behavior required for compatibility.

The coordinator should understand capabilities, shared deadlines, provider outcomes, and diagnostics. It should not understand XPath selectors, provider cookies, wrapper encodings, User-Agent lists, or challenge signatures.

## Public search input

A normalized internal request should contain:

    query: non-empty string
    page: one-based integer
    locale: provider-independent locale string
    safe_search: 0 | 1 | 2
    time_range: None | day | week | month | year
    timeout_limit: optional duration
    result_limit: positive bounded integer

The model/agent does not supply provider-specific raw fields.

## Capability contract

| Capability | Google | Bing | Primary DuckDuckGo |
| --- | --- | --- | --- |
| General web | yes | yes | yes |
| Page 1 | yes | yes | yes |
| Later pages | yes, max 50 | no | yes, stateful |
| Time range | yes | no | yes |
| Locale | language + region traits | region traits | region traits + Accept-Language |
| Safe-search capability | yes | yes | advertised; no explicit HTML request field in audited builder |
| JavaScript runtime | no | no | no |

The common gate skips unsupported page/time requests rather than letting each provider invent fallback parameters.

## Prepared request model

A provider mutates/returns a request equivalent to:

    method
    url
    headers
    cookies
    form/data
    json/content
    allow_redirects
    max_redirects
    auth
    verify override
    browser impersonation profile
    default-header policy
    provider network flags

Common defaults are defined in `07_HTTP_TRANSPORT.md`.

## Provider lifecycle

Conceptually:

    eligibility(request)
      -> prepare(request, common_context)
      -> if no URL: no network request
      -> transport(request, remaining_deadline)
      -> generic HTTP classification
      -> provider response parse
      -> provider result list / side channels

The actual Python API may combine preparation and parsing methods. Tests must still be able to assert them separately.

## Provider result contract

Provider parsers emit main-result objects/dictionaries containing enough information for common normalization:

    url
    title
    content/snippet
    thumbnail? / img_src?
    provider identity
    template/priority defaults

Provider position is assigned later by the common aggregation layer as accepted main-result ordinal.

## Exact error granularity is provider-specific

The contract must not impose a uniform “malformed item always skips” rule because source behavior differs:

- Google wraps each result block in an exception boundary and can skip a malformed block while continuing.
- Bing skips missing link/title, but malformed recognized base64 wrapper decoding can escape and fail the provider.
- primary DuckDuckGo indexes some href values without a per-item catch, so structural failures can escape and fail the provider.

A clean abstraction must preserve these different parser error boundaries.

## No-request outcome

Some provider request builders intentionally produce no URL:

- query too long;
- unsupported provider-specific continuation state;
- locale-specific continuation restriction;
- secondary adapter missing cached/discovered page URL.

This is distinct from an HTTP failure. The coordinator should represent it without fabricating a network exception.

## Generic HTTP layer contract

Before provider parsing, common transport may raise/classify:

- TLS/SSL errors;
- timeout;
- request/connection errors;
- recognized CAPTCHA/challenge responses;
- access denied (including 402/403 mapping);
- too many requests (429);
- other HTTP errors.

Providers then add their own response-level signatures.

## State contract

### Primary DuckDuckGo

State identity:

    transformed query + "//" + stable User-Agent

Stored value:

    vqd

TTL:

    3600 seconds

A continuation request without matching state must not be sent and becomes the explicit zero-suspension CAPTCHA/access-denied path.

### Secondary DuckDuckGo

Optional adapter state maps exact `(query, page)` to provider-generated request URLs with page-specific TTL behavior documented in `06_DUCKDUCKGO_WEB_PROTOCOL.md`.

### Traits

Provider trait mappings are release/runtime data, not per-query arbitrary state. Search should consume a prepared snapshot.

## Transport-profile contract

Profiles are part of provider compatibility:

- Google: fixed Nokia UA set + `chrome99_android`;
- Bing: common browser profile + provider HTTP/3 enablement;
- primary DDG: stable generated UA + common browser profile + explicit navigation headers;
- secondary DDG: Firefox impersonation + default browser headers disabled.

Do not move these values into a generic “random browser” helper that changes their relationships.

## Locale contract

Locale mapping is provider-specific and trait-driven. A common resolver can own best-fit logic, but providers receive their own resulting code strings.

Important special values include:

- Google all-region trait sentinel `ZZ`, all-language request behavior through empty `lr`;
- Bing all-region sentinel `clear`;
- DuckDuckGo all-region `wt-wt`.

Aliases must come from the trait snapshot, not ISO-string guessing.

## Aggregation boundary

Providers stop at result production. They do not decide:

- global duplicate identity;
- longer-title/content merge;
- engine-weight score;
- final group ordering;
- agent result limit.

Those belong to common compatibility aggregation.

## Host security boundary

Providers also do not fetch their output URLs. Agent-facing URL validation may run after compatibility aggregation. This keeps provider parity separate from host security policy.

## Provider diagnostics

A useful host-facing outcome can carry:

    provider
    started
    completed
    elapsed_ms
    result_count
    failure_class
    suspended
    timeout

Do not expose:

- cookies;
- `vqd`;
- complete provider response bodies;
- raw challenge pages;
- proxy credentials.

## Required provider tests

Every adapter must test:

- capability gating;
- exact endpoint/method;
- exact parameter conditions;
- exact transport profile;
- locale mapping;
- parser selectors;
- provider-specific wrapper handling;
- provider-specific malformed-item/error boundaries;
- challenge/block signatures;
- state rules where applicable;
- healthy zero-result behavior.

A provider adapter is not complete until its request fixture and response fixture both pass independently.