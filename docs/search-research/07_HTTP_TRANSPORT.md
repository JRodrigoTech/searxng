# HTTP transport

## Transport requirements

The three providers require only a narrow transport contract:

- asynchronous GET and POST;
- query and form encoding;
- explicit headers and cookies;
- response status, final URL, headers, and bounded body;
- TLS verification;
- compression decoding;
- optional HTTP version selection;
- connection reuse;
- per-call deadline;
- controlled redirect policy;
- typed transport exceptions.

The transport must not parse search result HTML or decide whether a response is a CAPTCHA. Those decisions belong to provider adapters.

## Observed transport characteristics

**Observation:** The analyzed transport uses an asynchronous curl-based client in a dedicated event-loop thread and exposes a synchronous bridge to provider code. It supports browser impersonation, pooled clients, proxies, local source addresses, and configurable HTTP/1.1, HTTP/2, or HTTP/3.

**Observation:** A client is keyed by settings such as TLS verification, redirect limit, source address, proxy selection, impersonation profile, curl options, and HTTP/3 selection. A pooled client has a default maximum of approximately ten concurrent connections unless configured otherwise.

**Observation:** The normal outgoing configuration enables HTTP/2, does not enable HTTP/3 globally, and uses a larger connection-pool setting at the application level. HTTP/3 is enabled for the Bing provider where no proxy prevents it.

**Recommendation:** An async runtime should use a native async client directly rather than recreating a synchronous bridge. Preserve the same observable contract, not the same internal implementation.

## Candidate libraries

| Choice | Strengths | Weaknesses | V1 status |
| --- | --- | --- | --- |
| httpx | Async API, pooling, HTTP/2 option, familiar timeout model | Browser impersonation is not its primary feature; HTTP/3 needs extra work | RECOMMENDED default if existing runtime already uses it |
| aiohttp | Mature async pooling and streaming | No native browser impersonation; HTTP/2 story is limited | VALID alternative |
| curl-cffi | Browser impersonation, HTTP/2/3 controls, close to observed behavior | Native binding/packaging complexity; sync/async lifecycle must be managed carefully | REQUIRED only if fingerprint fidelity proves necessary |
| urllib / standard library | Zero third-party dependency | Poor async ergonomics, no browser profile, more manual pooling and parsing | NOT_RECOMMENDED for this task |

The final selection depends on the host runtime. See 15_DEPENDENCIES.md.

## Client lifecycle

Recommended lifecycle:

1. Create one transport object per runtime.
2. Create one pooled client per materially different transport profile.
3. Reuse connections across searches.
4. Close clients during runtime shutdown.
5. Keep provider-specific cookies in explicit request/state storage when the client should not persist all cookies.

The observed client configuration discards accumulated cookies by default. This is useful for isolation but means a provider that requires state must pass its cookies explicitly or own a cookie jar.

## Redirect policy

There are two different redirects:

1. HTTP redirects from a provider response.
2. Result-link wrappers embedded in HTML.

The transport handles only the first. The provider parser handles the second.

Recommended defaults:

- Google: do not follow redirects; inspect status and final URL for sorry/CAPTCHA behavior.
- Bing: do not fetch wrapper destinations; decode the wrapper locally.
- DuckDuckGo HTML: do not follow result links; retain the form request’s redirect policy.
- Trait discovery: follow only the small, explicitly configured redirect budget.

Set an absolute redirect count and reject cross-scheme or unexpected-host redirects for provider discovery unless the provider contract documents them.

## Timeout model

Every transport call receives a duration computed from:

    remaining = max(0, deadline - monotonic())

The transport should subtract a small internal scheduling allowance if required by the client, but must not extend the caller’s deadline.

**Observation:** The synchronous bridge uses a default fallback timeout of roughly 120 seconds when no per-thread timeout is set, adds a small 0.2-second overhead, and subtracts elapsed time from the shared start time. The search configuration normally supplies a much smaller request timeout, around three seconds.

**Recommendation:** Require an explicit deadline from the coordinator for all provider calls. A missing deadline is a programming error in the search path, not a reason to use an unlimited request.

## Retries

The observed network layer:

- retries a disconnected connection once without using the normal retry budget;
- decrements a retry counter for other request exceptions;
- can recreate a client after a disconnected connection;
- can retry configured HTTP-error responses when retry-on-HTTP-error is enabled;
- does not automatically retry every HTTP status in the default configuration.

Recommended V1 policy:

| Condition | Retry |
| --- | --- |
| DNS/connect failure | At most one quick retry if remaining time permits |
| TLS failure | No immediate retry; classify and cool down |
| Read timeout | No retry after the total deadline; one retry only if substantial time remains |
| 429 | No tight retry; return RateLimited and schedule backoff |
| 403/anti-bot | No automatic retry |
| 5xx | One retry only when provider policy allows and deadline permits |
| Parse failure | Never retry the same bytes; a live smoke test should detect drift |

A retry is another provider request and must remain within both total deadline and per-provider request budget.

## TLS, proxies, and source addresses

- Verify TLS by default.
- Allow an explicit CA bundle when the host environment requires it.
- Treat disabling verification as an operational override, not a provider default.
- Support a configured proxy only at the transport boundary.
- Do not pass proxy credentials into telemetry.
- If source-address or Tor routing is used, treat it as deployment configuration and include only a redacted route label in metrics.

## Compression and response bounds

Most HTTP clients transparently decode gzip, brotli, and deflate when advertised. The adapter should parse decoded bytes, but the transport must enforce:

- maximum compressed body size where available;
- maximum decompressed body size;
- maximum header size;
- a parser input limit.

**Recommendation:** Start with a conservative per-provider body limit such as 2 MiB compressed and 8 MiB decompressed, then tune from fixtures and telemetry. A limit exceeded becomes InvalidResponse, not an unbounded read.

## HTTP version and browser profile

Browser impersonation is a coherent bundle: User-Agent, default headers, TLS/HTTP behavior, and sometimes ALPN. Do not set a Nokia User-Agent while separately impersonating an unrelated browser profile unless the provider-specific document explicitly requires it.

The selected providers currently use:

- Google: mobile User-Agent plus an Android Chrome 99 transport profile.
- Bing: a browser-like profile; HTTP/3 is an optional provider setting.
- DuckDuckGo HTML: stable generated User-Agent and explicit fetch/navigation headers.

These combinations are observations, not guarantees of future acceptance.

## Transport observability

Safe fields:

    provider
    method
    host
    status_code
    content_type
    http_version
    elapsed_ms
    bytes_read
    retry_count
    redirect_count

Do not log complete URLs containing queries, cookie values, validation tokens, authorization headers, or response bodies by default.
