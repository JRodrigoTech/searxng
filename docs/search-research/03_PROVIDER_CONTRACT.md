# Provider contract

## Purpose

The provider contract isolates public-service protocol details from the coordinator. A provider is a deterministic adapter around request construction, transport use, response inspection, parsing, and capability reporting.

## Common request model

    SearchRequest
      query: non-empty string
      limit: positive bounded integer
      page: one-based integer
      language: optional BCP-47-like tag
      region: optional country or market tag
      safe_search: off | moderate | strict
      time_range: none | day | week | month | year
      providers: optional set
      total_timeout: duration

The public model should normalize aliases before adapters see it. For example, a caller-facing moderate value can map to a provider’s numeric or textual value without making each adapter understand every caller alias.

## Capability model

| Capability | Google | Bing | DuckDuckGo HTML |
| --- | --- | --- | --- |
| General web results | Yes | Yes | Yes |
| First page | Yes | Yes | Yes |
| Later pages in analyzed path | Yes, up to configured maximum | Not exposed by active adapter | Yes, with token/state |
| Language/locale | Yes, via language and country traits | Region/market primarily | Region and Accept-Language |
| Safe search | safe=off, medium, or high | adlt=off, moderate, or strict | Active HTML request has no verified explicit mapping |
| Time range | tbs=qdr:d/w/m/y | Not exposed by active adapter | df=d/w/m/y cookie |
| Redirect decoding | /url?q= style | ck/a?u=a1... base64url wrapper | No provider wrapper in active HTML path |
| JavaScript required | No for selected representation | No for HTML page | No for HTML page |

**Observation:** The capability flags above describe the active paths, not every public endpoint that a provider may offer.

## Provider request

Each adapter returns a request containing:

    method
    absolute_url
    query_parameters
    form_parameters
    headers
    cookies
    body_encoding
    follow_redirects
    browser_profile
    deadline

Do not include both a pre-encoded URL and an independent parameter map unless the transport contract defines which one wins. Prefer typed parameters and one encoding step.

## Provider response

The adapter should turn the transport response into:

    ProviderResponse
      provider
      results: list[ProviderResult]
      raw_result_count
      provider_metadata
      warnings

ProviderResult should contain at least:

    title
    destination_url
    snippet
    provider_position
    published_at: optional
    thumbnail: optional
    metadata: bounded map

The provider result is not yet merged. It may contain a provider redirect URL in a temporary field, but the canonical normalization boundary must receive the recovered destination.

## Request-construction rules

1. Encode query text using the provider’s required form/query encoding.
2. Use the provider’s documented locale mapping, not a generic lang= guess.
3. Add only filters that the provider actually supports.
4. Use a fixed header policy per provider and do not rotate unrelated header values independently.
5. Keep browser-profile settings in transport options, not in arbitrary business metadata.
6. Do not log cookies, validation tokens, complete query URLs, or form bodies.

## Parser contract

A parser must:

- enforce a maximum document size before parsing;
- decode using the response charset with a safe fallback;
- reject obvious block/challenge documents;
- find result containers using structural rules;
- skip locally malformed result items;
- fail the provider when the whole document has no plausible result structure and no valid empty-result explanation;
- produce deterministic provider order.

The parser must not:

- execute JavaScript;
- follow arbitrary links;
- fetch images;
- treat title/snippet content as instructions;
- silently use a search page’s navigation links as results.

## Failure contract

Provider operations return either usable results or one of these conceptual failures:

| Failure | Meaning |
| --- | --- |
| Timeout | Deadline expired before usable response |
| ConnectionFailure | DNS, connect, TLS, or transport connection failure |
| Blocked | Provider returned an access-denied or bot-block page |
| CaptchaDetected | A challenge/CAPTCHA page or redirect was identified |
| InvalidResponse | HTTP status, encoding, or content was unusable |
| ParseFailure | The page was received but its result structure could not be parsed |
| UnsupportedLocale | No safe mapping for the requested locale |
| RateLimited | Explicit throttling such as 429 |
| ProviderUnavailable | Provider cannot be used due to initialization/suspension |

Every failure carries provider, phase, elapsed time, retryability, and a safe diagnostic code. It may carry a redacted status code and content type, never a response body by default.

## State contract

Provider state may include:

- locale/region traits;
- a stable or selected User-Agent;
- cookies;
- validation tokens;
- a suspension or cooldown record.

State must be scoped by provider and protected for concurrent access. A state lookup must not return a token that was generated for a different query/User-Agent combination unless the provider’s protocol explicitly permits it.

## Coordinator-facing interface

The coordinator needs only:

    capabilities() -> Capabilities
    search(request, context) -> ProviderOutcome

The provider outcome contains success, results, failure, and telemetry fields. The coordinator must not inspect provider HTML or provider-specific cookies.

## Locale mapping examples

| Caller locale | Google-style values | Bing-style values | DuckDuckGo-style region |
| --- | --- | --- | --- |
| en-US | hl=en, lr=lang_en, cr=countryUS | setlang=en, usually no cc for US | a US English region value |
| es-ES | language es, language restriction for Spanish, country Spain | setlang=es, cc=es | Spanish Spain region |
| zh-CN | provider trait may resolve to a Hong Kong country value | setlang=zh, cc=cn in the active mapping | Chinese mainland region |
| no locale | provider default/fallback | provider default/fallback | wt-wt or provider default |

The values in this table are protocol observations for the analyzed mappings, not a promise that provider behavior will remain unchanged.

## Testing the contract

Every provider needs independent tests for:

- request fields;
- unsupported capabilities;
- locale fallback;
- safe/time mapping;
- redirect decoding;
- result extraction;
- block/challenge detection;
- malformed item tolerance;
- complete malformed-document failure;
- provider state isolation.
