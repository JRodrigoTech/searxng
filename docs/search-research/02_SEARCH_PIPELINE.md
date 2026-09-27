# Search pipeline

## End-to-end sequence

The following sequence is the implementation target for an ordinary web search:

1. Validate the query and output limit.
2. Capture start = monotonic() and compute deadline = start + total_timeout.
3. Resolve the requested provider set.
4. Parse language and region input once.
5. Ask each provider for a capability-aware request.
6. Drop providers that cannot support the requested page or time filter, recording an unsupported-capability outcome.
7. Start all remaining provider operations concurrently.
8. Give each operation the remaining time at the instant its network operation begins.
9. Decode and parse each completed response in its provider task.
10. Assign provider positions to valid main results in provider order.
11. Normalize URLs, text, thumbnails, dates, and metadata.
12. Insert normalized results into the merge index using canonical identity.
13. Aggregate provider provenance and positions.
14. Calculate scores.
15. Apply deterministic ordering and the final limit.
16. Return successful results plus compact provider diagnostics.

## Observed execution behavior

### Search clock

**Observation:** The search clock starts at entry to the search operation, before provider requests are dispatched. Provider requests use one shared start time and a timeout derived from the default provider timeout, an optional query timeout, and an optional maximum request timeout.

**Observation:** Each worker is joined with the remaining time, not with a fresh full timeout. A worker that is still running after the deadline is marked as timed out. The underlying thread may finish later, but its result is no longer accepted.

**Recommendation:** Preserve the shared-deadline semantic, but use cancellable asynchronous tasks in the new implementation. Never reset the clock for parsing, retries, or a later provider.

### Provider selection

**Observation:** A provider is skipped when it is absent, suspended, uninitialized, or cannot build a request for the requested page/time combination. The active configuration snapshot also disables Google and Bing while leaving the DuckDuckGo HTML adapter enabled; this is configuration, not a protocol limitation.

**Recommendation:** Make provider selection explicit in the SearchRequest. The default may include all three, but a deployment can disable a provider without changing the adapter.

### Request preparation

Each adapter receives the same conceptual input:

    query
    page            # one-based
    language
    region
    safe_search
    time_range
    remaining_time

The adapter returns one of:

- ProviderRequest — ready for transport.
- UnsupportedCapability — the provider cannot honor the requested option.
- ProviderFailure — request preparation itself failed.

Unsupported options should be reported distinctly from network errors. For example, Bing’s active adapter has no page/time capability in the analyzed path, while Google and DuckDuckGo do.

### Network and parse execution

The provider operation should combine:

1. request construction;
2. transport call;
3. HTTP/status/block inspection;
4. response decoding;
5. provider-specific extraction;
6. provider-result validation.

This keeps a provider’s anti-bot signatures and redirect rules out of generic transport code.

### Result insertion

Provider positions are assigned to valid main results in the order in which the provider parser emits them. A result that is later merged still retains the position it had in each provider’s list.

If the provider emits an item with no title, no absolute destination, or an unusable redirect, the item is skipped and parsing continues where safe. A malformed whole document becomes a provider parse failure.

## First-page fast path

The recommended V1 path is one request per provider:

| Provider | V1 first-page request | Expected document |
| --- | --- | --- |
| Google | One GET | XML-like HTML mobile result page |
| Bing | One GET | Standard HTML result page |
| DuckDuckGo | One POST | No-JavaScript HTML result page |

The first-page path avoids DuckDuckGo token acquisition for the initial request. A later page needs provider state and should be enabled only after the first page has succeeded.

## Provider-specific state in the pipeline

- Google has process-start locale traits and a rotating/fixed set of mobile User-Agent values.
- Bing has process-start region traits; its active search request uses the selected market in setlang and optionally cc.
- DuckDuckGo has process-start traits, a stable process-level User-Agent, cookies for region/time, and a validation token cached by query plus User-Agent.

State lookup must be bounded and must not block the entire search on a slow trait refresh. A stale or missing trait map should fall back to a documented default.

## Partial failure semantics

Example:

    Google       succeeds in 300 ms
    Bing         fails in 700 ms
    DuckDuckGo   is unfinished at the deadline

The SearchResponse contains Google results, Bing’s typed failure, and DuckDuckGo’s timeout outcome. The search operation itself is successful if at least one provider produced usable results; the response may still carry an overall degraded status.

If every provider fails, return an empty bounded result list plus typed diagnostics. Do not invent a successful result or expose a raw exception as the only explanation.

## Provider diagnostics

The coordinator should record, per provider:

    provider
    prepared
    started_at
    elapsed_ms
    status
    raw_result_count
    normalized_result_count
    merged_result_count
    failure_type
    timeout
    blocked
    parse_error

The returned tool response may expose a safe summary such as providers_succeeded and providers_degraded. Detailed transport state belongs in structured logs or metrics.

## Explicitly excluded paths

- External bang/redirect commands are not part of the ordinary search contract.
- Answer, infobox, suggestion, and correction objects are optional side channels, not required V1 main results.
- A provider’s JavaScript API endpoint is not used merely because it can return JSON.
- Search result URLs are not fetched after search.
