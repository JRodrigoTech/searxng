# Clean-room web-search subsystem: documentation index

## Objective

This document set specifies an independent, minimal Python search subsystem for an AI-agent tool. The subsystem sends one query to Google Web Search, Bing Web Search, and DuckDuckGo Web Search, executes eligible providers concurrently, converts their replies into one canonical result model, merges equivalent destinations, ranks the merged set, and returns a bounded structured response.

The specification is deliberately narrower than a general search website. It excludes a web UI, a public HTTP server, a database-backed application, media-search categories, arbitrary page fetching, user preference pages, and provider plug-in infrastructure.

All protocol observations are time-sensitive. A live endpoint, selector, challenge page, or browser fingerprint can change without notice. Statements are labelled as follows:

- **Observation** — behavior verified in the analyzed code path.
- **Inference** — a conclusion drawn from multiple observations; validate before treating it as a protocol guarantee.
- **Recommendation** — proposed behavior for the new implementation.
- **Unknown** — not established by the available evidence and requiring live validation or an explicit product decision.

## Scope and target flow

    search(request)
      |
      +--> prepare eligible provider requests
      |
      +--> Google       \
      +--> Bing          +--> concurrent transport calls
      +--> DuckDuckGo  /
                         |
                         +--> provider-specific parsing
                         |
                         +--> canonical normalization
                         |
                         +--> URL identity and duplicate merge
                         |
                         +--> deterministic ranking
                         |
                         +--> bounded SearchResponse

The request deadline begins when the public search operation begins. A successful provider remains useful when another provider fails, is blocked, or exceeds the deadline.

## Documentation map

| File | Purpose |
| --- | --- |
| 00_INDEX.md | Scope, reading order, V1 checklist, completion gate |
| 01_TARGET_ARCHITECTURE.md | Independent component architecture and boundaries |
| 02_SEARCH_PIPELINE.md | End-to-end execution sequence and data flow |
| 03_PROVIDER_CONTRACT.md | Adapter, transport, parser, capability, and failure contracts |
| 04_GOOGLE_WEB_PROTOCOL.md | Google public-web request and parsing behavior |
| 05_BING_WEB_PROTOCOL.md | Bing public-web request, locale, parsing, and redirect decoding |
| 06_DUCKDUCKGO_WEB_PROTOCOL.md | DuckDuckGo HTML flow, token state, pagination, and blocks |
| 07_HTTP_TRANSPORT.md | Minimal transport, pooling, redirects, retry, TLS, and compression policy |
| 08_CONCURRENCY_AND_DEADLINES.md | Shared deadline and partial-failure algorithm |
| 09_RESULT_MODEL_AND_NORMALIZATION.md | Raw, provider, canonical, merged, and final result representations |
| 10_URL_IDENTITY_AND_DEDUPLICATION.md | Conservative canonical identity and duplicate rules |
| 11_MERGE_AND_RANKING.md | Field merge policy, provenance, observed scoring, and V1 ranking |
| 12_FAILURE_AND_RESILIENCE.md | Typed failures, retry policy, suspension, and degradation |
| 13_SECURITY_BOUNDARIES.md | Untrusted search data and transport/security boundaries |
| 14_TEST_STRATEGY.md | Unit, fixture, concurrency, regression, and optional live tests |
| 15_DEPENDENCIES.md | Dependency choices, alternatives, and V1 minimum |
| 16_IMPLEMENTATION_BLUEPRINT.md | Proposed package shape, APIs, responsibilities, and call sequence |
| 17_OPEN_QUESTIONS.md | Items that require live validation or product decisions |

## Recommended reading order

1. This index.
2. The target architecture and pipeline.
3. The provider contract.
4. The three provider protocol documents.
5. Transport and deadline behavior.
6. Result identity, merge, and ranking.
7. Failure, security, testing, and dependency decisions.
8. The implementation blueprint.
9. Open questions before implementation begins.

## ESSENTIAL_V1 checklist

- [ ] Accept a non-empty query and a bounded result limit.
- [ ] Execute Google, Bing, and DuckDuckGo with independent provider adapters.
- [ ] Use one monotonic total deadline for the search operation.
- [ ] Start eligible provider requests concurrently.
- [ ] Isolate provider exceptions and preserve successful results.
- [ ] Support first-page search for all three providers.
- [ ] Support Google time and safe-search parameters.
- [ ] Support Bing market/region and safe-search parameters.
- [ ] Support DuckDuckGo region, first-page HTML POST, and the token needed for safe pagination.
- [ ] Detect provider block, CAPTCHA, malformed-response, timeout, and transport failures without attempting to defeat challenges.
- [ ] Normalize title, snippet, destination URL, provider, and 1-based provider position.
- [ ] Keep provider provenance and all observed positions after a merge.
- [ ] Use an explicit canonical identity separate from the display URL.
- [ ] Apply conservative duplicate matching that does not remove meaningful query parameters.
- [ ] Use a deterministic, unit-testable ranking rule.
- [ ] Enforce a per-provider parse cap and a final output cap.
- [ ] Emit structured telemetry without cookies, tokens, or response bodies.
- [ ] Add sanitized provider fixtures and protocol regression tests.
- [ ] Keep arbitrary page fetching out of this component.

## Important boundaries

The three services do not expose one uniform protocol:

- Google uses a mobile/legacy public representation that returns an XML-like HTML document and uses a redirect-style result URL.
- Bing uses a standard HTML result page and may wrap result destinations in a base64url value.
- DuckDuckGo’s active no-JavaScript flow is a form POST and has a query/User-Agent-bound validation token for later pages.

The adapter layer must retain these differences. The coordinator should know only capabilities, deadlines, typed results, and typed failures.

## Major uncertainties

The largest unresolved issues are intentionally collected in 17_OPEN_QUESTIONS.md:

- Whether each public endpoint remains available from the deployment network.
- Whether current anti-automation behavior is IP-based, fingerprint-based, or both.
- Whether an explicit safe-search mapping for DuckDuckGo HTML is still supported.
- Whether Google’s public mobile representation continues to emit the observed classes.
- Whether Bing’s redirect wrapper format remains stable.
- Which locale fallback policy is acceptable for the product.
- Whether the runtime can use an asynchronous HTTP stack and HTML parser without conflicting dependencies.

## Definition of done

The documentation is complete when a separate engineer can implement the subsystem without reopening the analyzed repository, while understanding which statements are observations and which are recommendations. An implementation is complete when it satisfies the ESSENTIAL_V1 checklist, passes the test plan, returns partial results on provider failure, and exposes no untrusted provider text as trusted agent instructions.
