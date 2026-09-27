# Failure and resilience

## Failure design goals

The search tool is a best-effort aggregation operation. One provider’s failure must not erase successful results from another provider. Failures must be typed, bounded, observable, and safe to expose at the tool boundary.

## Typed failure model

    ProviderFailure
      provider
      kind
      phase
      retryable
      elapsed_ms
      status_code: optional
      content_type: optional
      diagnostic_code
      cooldown_until: optional

Kinds:

- Timeout
- ConnectionFailure
- Blocked
- CaptchaDetected
- InvalidResponse
- ParseFailure
- UnsupportedLocale
- RateLimited
- ProviderUnavailable

The diagnostic code should be stable and low-cardinality. For example, google_sorry_redirect is safer than embedding a URL or response excerpt.

## Failure handling matrix

| Failure | Detection | Retry | Continue search? | Observability | Expose to AI agent? |
| --- | --- | --- | --- | --- | --- |
| Timeout | Remaining deadline reaches zero, connect/read timeout | No after deadline; at most one early transient retry | Yes | timeout count, elapsed, phase | Safe summary only |
| ConnectionFailure | DNS, connect, TLS, socket, proxy failure | One retry only for clearly transient connect failure | Yes | error class, host, retry count | Usually summarize degraded provider |
| Blocked | 403/access-denied page, explicit provider block marker | No tight retry; cooldown | Yes | blocked flag, status | Optional safe provider status |
| CaptchaDetected | Google sorry path, challenge form, known challenge body | No solver; cooldown | Yes | captcha flag, detection code | Safe “provider unavailable” summary |
| InvalidResponse | Bad status, unsupported encoding, size limit | Usually no; status-specific policy | Yes | status/content type/bytes | Usually no raw detail |
| ParseFailure | Expected result structure missing or malformed | No same-response retry | Yes | fixture/regression counter | Safe degraded summary |
| UnsupportedLocale | No safe provider mapping | No | Yes, if fallback is safe | locale and fallback label | Only if user requested strict locale |
| RateLimited | 429 or provider throttle marker | Backoff later, not in same tight search | Yes | status, retry-after if safe | Safe degraded summary |
| ProviderUnavailable | Disabled, suspended, not initialized | Wait for cooldown or reinitialize | Yes | availability state | Optional |

## HTTP status interpretation

Generic handling should distinguish:

- 2xx with valid result structure: success;
- 2xx with challenge or login structure: Blocked or CaptchaDetected;
- 3xx: provider-specific; Google’s observed 302 is challenge-like, while DuckDuckGo’s observed 303 returns empty;
- 401/403: access denied or blocked;
- 402: access denied in the observed generic mapping;
- 429: RateLimited;
- 5xx: InvalidResponse or transient transport failure, depending on body and retry policy;
- all other 4xx: InvalidResponse unless a provider-specific mapping exists.

Do not use status alone when the provider has a stronger body signature.

## Provider suspension and cooldown

**Observation:** Provider state tracks continuous failures, a suspension end time, a reason, and a lock. The analyzed policy applies short increasing failure bans, with category-specific longer durations for CAPTCHA, access denial, and repeated throttling. Success resumes a suspended provider.

**Recommendation:** Implement a small circuit breaker:

1. On a transient connection failure, increment a failure counter.
2. On a block/CAPTCHA/rate limit, open the provider circuit immediately for a category-specific cooldown.
3. On generic parse failure, record the event but do not necessarily suspend after one occurrence.
4. Use bounded exponential backoff with jitter for repeated failures.
5. On a successful, structurally valid response, reset the consecutive-failure count.
6. Never let a suspended provider delay other providers.

Illustrative starting values:

| Failure class | Initial cooldown | Growth/cap |
| --- | ---: | --- |
| Transient connection | 5 s | exponential, cap 120 s |
| Rate limit | 30 s | provider Retry-After if safe, cap 15 min |
| CAPTCHA/block | 1 h | longer after repeats |
| Parse drift | no automatic suspension on first event | operator review after threshold |

These are recommendations; tune from deployment telemetry.

## Parsing failures

Parser behavior should be tiered:

### Item-level failure

Examples:

- missing title;
- one malformed href;
- one invalid encoded redirect;
- one oversized snippet.

Skip the item, increment a warning counter, and continue parsing the other blocks.

### Document-level failure

Examples:

- no expected result container and no valid empty-result marker;
- response is a challenge/login page;
- invalid encoding prevents safe parsing;
- result container exists but every item is structurally unusable.

Return ParseFailure, Blocked, or CaptchaDetected as appropriate.

### Transport-level failure

Examples:

- DNS or connection failure;
- TLS verification failure;
- read timeout;
- decompressed-size limit;
- cancellation.

Return the corresponding typed transport failure and discard the response.

## Retry safety

A retry is allowed only when:

- the provider request is idempotent or the provider’s form POST is known to be safe to repeat;
- remaining time is sufficient;
- the failure is classified transient;
- the retry count is within a hard limit;
- the provider is not blocked or challenged.

Do not retry malformed parser output or CAPTCHA pages. Do not retry a tokenless DuckDuckGo continuation request.

## Partial and all-provider failure

If one provider succeeds, return its results and mark the response degraded if another provider failed. If no provider succeeds:

    results = []
    degraded = true
    providers = typed diagnostics

The public tool should distinguish “no results from a healthy provider” from “all providers failed”, even if both result lists are empty.

## Safe diagnostics

Good diagnostic fields:

    provider
    phase
    kind
    status_code
    elapsed_ms
    retryable
    cooldown_active

Do not include:

- cookies;
- validation tokens;
- full query URLs;
- full HTML;
- arbitrary exception strings containing URLs or credentials;
- provider snippets in error labels.

## Recovery

Recovery actions belong outside the immediate search call:

- cooldown expiry;
- background trait refresh;
- circuit reset after a valid smoke test;
- dependency/client recreation after repeated connection failure;
- operator alert after parser regression threshold.

The search call itself should remain bounded and should not wait for recovery.
