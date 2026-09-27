# Target architecture

## Design goal

Build a small library, embedded in an existing asynchronous AI runtime, with one public search operation and three provider adapters. Provider-specific HTTP details must stop at the adapter boundary. The coordinator must not contain CSS selectors, cookies, provider redirect decoding, or token-generation rules.

## Component diagram

    Search API
        |
        v
    SearchCoordinator
        |
        +--> LocaleMapper
        +--> ProviderRegistry
        +--> DeadlineController
        +--> ProviderTaskGroup
                         |
                         +--> GoogleProvider
                         +--> BingProvider
                         +--> DuckDuckGoProvider
                                      |
                                      +--> SearchTransport
                                      +--> ProviderStateStore
        |
        +--> ResultNormalizer
        +--> UrlIdentity
        +--> ResultMerger
        +--> ResultRanker
        +--> SearchResponse

The transport is shared, but every provider owns its request construction and response parser. The state store is a narrow capability used by DuckDuckGo for validation-token caching; it must not become a general application session store.

## Responsibilities

### Search API

**Responsibility**

- Validate the public request.
- Supply defaults.
- Call the coordinator.
- Return a typed response suitable for an AI tool.

**Must not know**

- Provider-specific endpoint URLs.
- HTML selectors.
- Cookie names.
- Redirect wrapper formats.
- Anti-bot page signatures.

### SearchCoordinator

**Responsibility**

- Establish the monotonic deadline.
- Resolve requested providers and locale capabilities.
- Ask each adapter to build a provider request.
- Dispatch eligible work concurrently.
- Convert exceptions into typed provider outcomes.
- Feed successful provider results to normalization, merge, and ranking.
- Enforce the final result limit.

**Must not know**

- How a provider obtains or decodes a destination URL.
- How a provider extracts a validation token.
- The raw response document structure.

### SearchProvider

Each provider implements four conceptual operations:

1. capabilities() — supported page, time, language, region, and safe-search features.
2. prepare(request, deadline) — produce an immutable provider request or an explicit unsupported outcome.
3. execute(provider_request, transport, deadline) — perform the provider call and decode provider-level state.
4. parse(response) — return provider results, optional provider metadata, and typed parse/block failures.

The actual Python API may combine operations, but the boundary should remain visible in tests.

### SearchTransport

**Responsibility**

- Async GET and POST.
- Pooled connections.
- TLS verification.
- compression decoding.
- bounded response reads.
- redirect policy.
- per-call remaining-time handling.
- optional HTTP/2 or HTTP/3 selection.

**Must not know**

- Which response class means a CAPTCHA.
- How a search result is ranked.
- Which cookie maps to a locale.

### LocaleMapper

**Responsibility**

- Parse user language/region input.
- Produce provider-specific language, market, or region values.
- Apply a documented fallback.
- Report unsupported locale as data, not as an exception that destroys the entire search.

Provider locale mappings are not interchangeable. Google, Bing, and DuckDuckGo use different strings and different notions of “market”.

### ResultNormalizer

**Responsibility**

- Convert provider output to a common result.
- Collapse whitespace.
- Require an absolute destination URL.
- Preserve 1-based provider position.
- Remove known provider redirect wrappers before identity calculation.
- Bound text and metadata.

### UrlIdentity

**Responsibility**

- Produce a conservative stable identity string.
- Leave the display URL available for the caller.
- Avoid merging URLs merely because they look similar.

See 10_URL_IDENTITY_AND_DEDUPLICATION.md.

### ResultMerger

**Responsibility**

- Detect equal identities.
- Preserve every provider and position observation.
- Select the best available display fields according to deterministic rules.
- Preserve useful metadata instead of overwriting it arbitrarily.

### ResultRanker

**Responsibility**

- Calculate a reproducible score from provider, provider position, and consensus.
- Use explicit provider weights.
- Apply deterministic tie-breaking.
- Be independent of HTML parsing and transport state.

## Recommended data ownership

| Data | Owner | Lifetime |
| --- | --- | --- |
| Query, limit, locale, filters | SearchRequest | One search |
| Per-provider capability flags | Provider adapter | Process lifetime or static configuration |
| HTTP connection pool | SearchTransport | Process/runtime lifetime |
| DuckDuckGo validation token | ProviderStateStore | Bounded TTL |
| Raw response bytes | Provider execution | One request; discard after parsing |
| Provider result | Provider adapter | One search |
| Canonical result | Normalizer/merger | One search |
| Telemetry counters | Observability sink | Configurable |

Do not put cookies, validation tokens, or raw HTML into the returned SearchResponse.

## V1 and later

### ESSENTIAL_V1

- One asynchronous public method.
- Three fixed providers.
- First-page support for all providers.
- Provider-specific safe search where verified.
- Google and DuckDuckGo time filters where verified.
- Bing market/region support.
- Shared total deadline.
- Typed partial failure.
- Conservative URL identity.
- Provenance-preserving merge.
- Deterministic ranking.

### USEFUL_LATER

- Provider-specific page counts beyond the first page.
- Periodic trait/locale refresh.
- Persistent encrypted token cache across process restarts.
- Adaptive provider weights.
- Circuit breaking across searches.
- More detailed provider health metrics.
- Optional answer/infobox results as separate result types.

### NOT_NEEDED

- Search website templates.
- User-facing engine selection pages.
- General category registries.
- Media, files, maps, news, and social result types.
- A plugin installation mechanism.
- A public server endpoint.
- Arbitrary result-page fetching.

## Architectural invariants

1. A provider can fail without invalidating another provider’s results.
2. A provider cannot mutate another provider’s request or state.
3. A deadline is monotonic and never extended by a slow provider.
4. A provider position starts at one and is assigned before duplicate merging.
5. A merged result keeps provider provenance and all observed positions.
6. Display URL and identity URL are separate values.
7. Provider text is untrusted data.
8. The parser never executes JavaScript supplied by a provider.
9. The output is bounded even if a response is unexpectedly large.
