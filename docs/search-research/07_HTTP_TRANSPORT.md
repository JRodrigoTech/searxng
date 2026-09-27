# HTTP transport

## Compatibility target

The audited search path relies on a curl-based asynchronous transport with browser impersonation support. For a parity implementation, transport behavior is part of the provider protocol rather than a replaceable detail.

A different HTTP library may work, but it must be treated as an explicit compatibility deviation and validated against live/fixture tests. The baseline for reproducing the observed behavior is `curl_cffi` or an equivalent implementation that can match the same browser/TLS profiles.

## Architecture

The audited stack combines:

- asynchronous curl clients;
- a dedicated event loop;
- a synchronous bridge used by engine worker threads;
- provider/network-specific pooled clients;
- browser/TLS impersonation;
- HTTP/1.1, HTTP/2, and conditional HTTP/3;
- explicit cookies/headers passed per request;
- shared search deadlines propagated through thread-local timeout state.

The embedded tool does not need to reproduce the synchronous bridge if it is already async. It **does** need to reproduce the observable request/timeout semantics.

## Client construction

A pooled client is keyed by transport-affecting values including:

    TLS verification
    maximum redirects
    local source address
    proxy selection
    impersonation profile
    curl options
    HTTP/3 enablement

Reusing a client with materially different values would change transport behavior and must be avoided.

The client baseline is:

    verify = configured value, true by default
    max_clients = configured pool size, or 10 fallback
    discard_cookies = true
    response_class = project response wrapper

Cookies therefore do not accumulate implicitly in the client; provider state must be supplied explicitly when required.

## Browser impersonation

Default transport impersonation is:

    chrome

When impersonation is active, the client enables the impersonated browser's default headers.

Provider overrides relevant to the tool are:

| Provider/path | Observed profile |
| --- | --- |
| Google web | `chrome99_android`, plus explicit Nokia `User-Agent` |
| Bing web | default browser impersonation; provider enables HTTP/3 |
| DuckDuckGo HTML | default browser impersonation plus stable explicit generated `User-Agent` and explicit navigation headers |
| DuckDuckGo secondary JSON/script | `firefox` with `default_headers=false` |

**PARITY MUST:** preserve provider-specific profile choices. Do not independently normalize User-Agent and TLS/browser identity into a supposedly more coherent fingerprint unless live evidence justifies a deliberate deviation.

## HTTP versions

Transport selection is:

1. if HTTP/2 is disabled: HTTP/1.1;
2. else if HTTP/3 is enabled for the network/provider and no proxy is configured: HTTP/3;
3. otherwise: HTTP/2.

The default outgoing configuration enables HTTP/2. Bing explicitly enables HTTP/3; this takes effect only when the proxy condition permits it.

## HTTPS restriction

When plain HTTP is disabled, curl protocol options restrict both direct requests and redirect protocols to HTTPS. This is a transport-level boundary independent from provider result URLs.

## Generic online request defaults

Each online provider request starts from:

    method = GET
    headers = {}
    data = {}
    json = {}
    content = b""
    url = ""
    cookies = {}
    allow_redirects = false
    max_redirects = 0
    soft_max_redirects = 0
    auth = None
    verify = None
    raise_for_httperror = true

Provider code mutates these values before the network call.

Although a convenience network `GET` helper defaults to following redirects when called directly, the ordinary online-provider processor explicitly passes `allow_redirects=false` from these request params. Therefore Google/Bing/DDG search calls through this path do not silently inherit the convenience helper's redirect-following default.

## Accept-Language

Before provider request construction, the generic online processor adds `Accept-Language` when that provider allows it.

If a parsed locale exists:

    <language>,<language>-<territory-or-language>;q=0.7,en;q=0.3

Otherwise:

    en-US,en;q=0.9

A provider may then preserve or override individual headers.

## Request execution

The online processor builds network arguments from:

    headers
    cookies
    auth
    verify override
    allow_redirects
    max_redirects
    curl_options
    impersonate
    default_headers

For POST it additionally forwards non-empty form data, JSON, or binary content according to the prepared request.

The response receives the complete search request params before provider parsing so stateful parsers can inspect submitted query/form/header values.

## Shared timeout semantics

The search worker installs a thread-local:

    timeout_limit
    search_start_time

For each network call, the sync/async bridge computes approximately:

    timeout = configured_timeout_or_thread_timeout_or_120
    timeout += 0.2
    timeout -= monotonic_now - search_start_time

The extra 0.2 seconds is bridge overhead; it does not create a fresh per-request search budget because elapsed time since the common start is subtracted.

The application default request timeout is 3.0 seconds. Search orchestration may choose a different total `actual_timeout` from selected engine timeouts, a query timeout, and an optional maximum-request timeout. The search-level deadline remains authoritative.

**PARITY MUST:** retries and secondary provider requests consume the remaining common search budget rather than receiving a new full timeout.

## Retries

Default configured retry count is zero.

The network layer nevertheless has one special behavior: if an existing connection is reported as disconnected, it closes the client and retries once without consuming the configured retry counter. Subsequent connection failures follow the normal retry budget.

Other request exceptions decrement the configured retry counter. HTTP-status retries occur only when a network has `retry_on_http_error` configured; the ordinary baseline does not retry every HTTP failure.

This distinction matters for compatibility tests: “retries=0” does not mean an already-disconnected pooled connection is never retried.

## HTTP error handling

With `raise_for_httperror=true`, the common layer classifies known challenge/error responses before the provider parser receives them. Relevant behavior includes:

- recognized Cloudflare challenge/CAPTCHA signatures;
- recognized reCAPTCHA signature;
- HTTP 402/403 -> access denied;
- HTTP 429 -> too-many-requests;
- other failing status codes -> underlying HTTP error.

Provider-specific checks, such as Google's 302/sorry handling or DuckDuckGo HTML's `challenge-form`, happen in provider code.

## Pooling and state isolation

The audited default pool configuration is 100 maximum connections at the application level. Individual clients use the configured value, falling back to 10 when unspecified.

Because `discard_cookies=true`, connection reuse must not be confused with cookie/session persistence. DuckDuckGo's continuation state and cookies are explicit provider-owned state.

## Proxy and source-address behavior

Networks may rotate configured proxies and local source addresses. These values are part of the client-cache key, so changing them can select/create another client. HTTP/3 is disabled when proxies are in use.

The initial embedded tool does not need proxy rotation unless Overmind already exposes it, but omitting proxy/source-IP features does not justify changing the protocol fingerprint of direct requests.

## Response body behavior

The audited transport consumes normal responses through curl's response abstraction; it does not implement the proposed custom compressed/decompressed size caps in the original research docs. Size caps are valuable for the host runtime but are **not an observed parity behavior**.

If Overmind imposes body limits for safety, document them as a host boundary around the compatibility adapter and ensure ordinary provider fixtures remain below the cap.

## Recommended implementation shape

For an async host runtime:

    SearchTransport
      -> curl_cffi.AsyncSession pools
      -> provider transport profile
      -> request with common remaining deadline

There is no need to recreate the extra event-loop thread or synchronous bridge merely to imitate internal architecture. Behavioral equivalence requires:

- the same browser/TLS profiles;
- the same HTTP version policy;
- the same explicit headers/cookies;
- redirects disabled for ordinary online searches;
- equivalent HTTP-error classification;
- equivalent retry behavior where relevant;
- shared-deadline semantics;
- isolated provider state.

## Compatibility tests

Test at minimum:

- default impersonation is Chrome-family;
- Google selects `chrome99_android` while sending a Nokia UA;
- secondary DDG selects Firefox and disables default headers;
- Bing selects HTTP/3 only with HTTP/2 enabled, provider HTTP/3 enabled, and no proxy;
- normal online requests pass `allow_redirects=false`;
- explicit cookies do not persist accidentally through a client jar;
- disconnected pooled connection receives the special recreation/retry;
- configured retries=0 does not introduce generic repeated requests;
- each request's timeout shrinks relative to one shared search start;
- TLS verification defaults on;
- client reuse key changes with proxy/profile/HTTP3/verification settings.

## Deliberate deviations

A host may add response-size limits, stronger cancellation, structured telemetry, or other safety features. Keep those outside the provider-protocol semantics and test that they do not mutate request fingerprints or accepted results under normal conditions.