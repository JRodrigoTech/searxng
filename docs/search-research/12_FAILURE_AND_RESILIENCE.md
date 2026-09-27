# Failure and resilience

## Compatibility target

Provider failures are isolated: one provider can fail, time out, be blocked, or be suspended without invalidating successful sibling providers. The compatibility layer should preserve the observed classifications and suspension semantics before the host converts them into its own tool diagnostics.

## Common online-provider exception handling

The online provider processor handles these broad groups:

1. TLS/SSL errors;
2. transport timeout / async timeout;
3. other request exceptions;
4. provider CAPTCHA, too-many-requests, and access-denied exceptions;
5. any other exception.

Transport-related failures and explicit access-denied/CAPTCHA/rate-limit failures request provider suspension. Generic unexpected parser exceptions are recorded but do not use the same `suspend=True` path.

No provider exception is supposed to terminate sibling searches.

## Generic HTTP classification

Before provider-specific parsing, the common HTTP error layer recognizes:

- provider-independent Cloudflare challenge patterns;
- Cloudflare firewall/access-denied patterns;
- a reCAPTCHA pattern;
- HTTP 402/403 as access denied;
- HTTP 429 as too many requests;
- other failing statuses through the underlying HTTP client's status exception.

Provider parsers can add stronger signatures, for example:

- Google: host/path `/sorry`, any 302, or short `/sorry/` body;
- primary DuckDuckGo: `form#challenge-form`;
- primary DuckDuckGo missing continuation `vqd`: explicit CAPTCHA-class exception with zero suspension.

Bing's ordinary web parser has no independent challenge selector in the audited path.

## Suspension state

Each provider processor owns/reuses a thread-safe suspended-status object keyed according to its network identity. It stores:

    continuous_errors
    suspend_end_time
    suspend_reason

### Generic suspension duration

When suspension is requested without an explicit access-denied duration, the current active configuration uses:

    ban_time_on_fail = 5 seconds
    max_ban_time_on_fail = 120 seconds

The current implementation chooses `min(max_ban_time_on_fail, ban_time_on_fail)`, so the immediate generic duration is 5 seconds. Although a continuous-error counter is incremented, this specific method does not itself implement exponential growth.

**PARITY MUST:** do not describe the current generic failure suspension as exponential backoff.

### Active configured access/challenge durations

The audited runtime configuration overrides schema defaults with:

| Failure | Active duration |
| --- | ---: |
| Access denied / HTTP 402-403 | 180 s |
| CAPTCHA | 3600 s |
| Too many requests / HTTP 429 | 180 s |
| Cloudflare CAPTCHA | 1,296,000 s |
| Cloudflare firewall/access denied | 86,400 s |
| reCAPTCHA | 604,800 s |

These are configuration values, not universal protocol constants. Compatibility fixtures should distinguish code behavior from deployment policy.

The settings schema's fallback defaults differ for several classes (for example generic access denied/CAPTCHA/too-many), so an implementation that wants this audited deployment behavior should use the active values above or make them explicit configuration.

## Explicit zero-suspension DuckDuckGo cases

Primary DuckDuckGo deliberately raises its CAPTCHA/access-denied class with:

    suspended_time = 0

when:

- a continuation page is requested without a cached `vqd`;
- an HTML response contains `challenge-form`.

That exception still records a provider failure, but its explicit duration overrides the normal CAPTCHA suspension period.

**PARITY MUST:** do not replace these zero-duration cases with the global one-hour CAPTCHA cooldown.

## Success recovery

When a provider successfully finishes and its result set is accepted before timeout, the processor resumes that provider:

    continuous_errors = 0
    suspend_end_time = 0
    suspend_reason = ""

A late worker marked timed out does not insert its results and therefore does not follow the normal accepted-success path.

## Suspended-provider selection

Before starting a search worker, the orchestrator checks provider suspension state. A currently suspended provider is not started; it is instead recorded as an unresponsive/suspended engine for that search.

This makes suspension a pre-dispatch capability/health gate rather than a delay inside the search call.

## Provider-specific failure behavior

### Google

Provider parser raises its CAPTCHA/access-denied type for:

- response host `sorry.google.com`;
- response path beginning `/sorry`;
- any HTTP 302 reaching the parser;
- body shorter than 2,000 chars containing `/sorry/`.

Individual malformed result blocks are caught and skipped, so one bad Google item normally does not fail the whole provider.

### Bing

Missing result link/title is skipped. However malformed base64 inside a recognized `u=a1...` wrapper is not locally caught. That exception can leave the parser and is then handled as a generic provider exception by the online processor.

A successful HTML response with zero matching result blocks returns an empty list rather than an explicit parse failure.

### Primary DuckDuckGo

- query length >= 500 produces no URL/request;
- 303 returns empty results;
- missing continuation token raises zero-duration CAPTCHA/access-denied;
- Chinese continuation can produce no URL/request;
- `challenge-form` raises zero-duration CAPTCHA/access-denied;
- structural exceptions during indexed result extraction can escape and become generic provider failures.

### Secondary DuckDuckGo JSON/script adapter

- missing preload URL means no request URL;
- JSON/parser failures can escape to common provider handling;
- its narrow arithmetic challenge path can perform one provider-specific follow-up request.

## Timeout versus transport failure

The search orchestrator can mark a worker timed out because the common search deadline elapsed even if the underlying network thread is still alive. Separately, the network/client can raise a timeout during provider execution.

Both prevent successful result insertion for that provider, but they arise at different layers. The host diagnostics may unify them as `Timeout` while retaining a low-cardinality phase field.

## Empty result versus failure

Compatibility requires preserving these distinctions:

- healthy provider returns zero matching results -> success with empty result list;
- provider request builder sets URL to none due to a known unsupported state -> provider does no network call and contributes no results;
- exception -> provider error/unresponsive diagnostic;
- suspended provider -> no worker launched;
- timed-out provider -> late results ignored.

Do not convert every empty list into `ParseFailure`.

## Host-facing typed diagnostics

Overmind can project internal conditions into a simpler enum such as:

    Timeout
    ConnectionFailure
    AccessDenied
    CaptchaDetected
    RateLimited
    ParserFailure
    Suspended
    UnsupportedCapability

This projection is useful, but it must not alter provider behavior. Preserve an internal reason/code sufficient to test source parity.

## Retry interaction

Configured generic network retries are zero in the audited defaults. One disconnected pooled connection may still be recreated and retried once without consuming that retry budget. Challenge/access-denied/rate-limit paths should not be wrapped in a new aggressive retry loop by the tool.

## Golden tests

Test:

- one provider failure preserves sibling results;
- suspended provider is skipped before worker launch;
- successful accepted result resets suspension state;
- generic transport failure uses active 5-second generic suspension when no explicit duration exists;
- active 180/3600/180-second configured access/CAPTCHA/rate-limit values;
- Cloudflare/reCAPTCHA configured durations;
- primary DuckDuckGo missing-vqd/challenge uses explicit zero suspension;
- Google malformed item is skipped locally;
- malformed Bing wrapper can escalate to provider failure;
- HTTP-200 zero-result page remains distinguishable from provider failure;
- late timed-out worker cannot recover by inserting results afterward.

## Deliberate deviations

Exponential circuit breakers, jitter, adaptive health scores, or long parser-regression cooldowns can be added by the host later. They are operational enhancements, not the audited failure policy, and should be disabled in compatibility tests.