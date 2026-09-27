# Search pipeline

## Compatibility objective

The pipeline must preserve both provider behavior and the order in which common aggregation steps happen. Changing phase order can change positions, duplicate identity, scores, and final ordering even when provider parsers return the same links.

## End-to-end sequence

For an ordinary search:

1. record one monotonic search start;
2. resolve selected primary providers;
3. skip missing/uninitialized/suspended providers;
4. apply common capability checks for page and time range;
5. prepare common request params including locale, safe-search, page, time range, and generic `Accept-Language`;
6. derive one shared `actual_timeout`;
7. start every eligible provider concurrently;
8. each provider mutates its prepared request with endpoint-specific URL, headers, cookies, form/query fields, and transport profile;
9. each provider performs HTTP work using the remaining common search budget;
10. common HTTP errors are classified before provider parsing;
11. provider parser emits main results and optional side channels;
12. late providers are prevented from inserting results after timeout;
13. accepted results are normalized as they enter the shared aggregate;
14. accepted main-result position is assigned per provider using accepted-result ordinal;
15. normalized identity is looked up in the global merge map;
16. duplicate fields/provenance/positions are merged immediately;
17. when collection is complete, scores are calculated;
18. results are stable-sorted by score descending;
19. category/template/image grouping performs the second ordering pass;
20. only then apply the caller's final result limit;
21. apply explicit host output/security policy;
22. serialize the bounded result to the agent.

## Search request fields

The common provider input conceptually carries:

    query
    page                # one-based
    safe_search         # 0, 1, 2
    time_range          # none/day/week/month/year
    locale
    provider category
    provider-specific state handle

Providers do not receive arbitrary raw parameter dictionaries from the model.

## Eligibility behavior

The common capability gate rejects/skips a provider when:

- requested page > 1 and provider does not advertise paging;
- requested page exceeds provider maximum page;
- a time range is requested and provider does not advertise time-range support.

For the primary three:

| Provider | Paging | Max | Time range |
| --- | --- | ---: | --- |
| Google | yes | 50 | yes |
| Bing | no | — | no |
| DuckDuckGo HTML | yes | common configured max unless otherwise bounded | yes |

Thus a page-2 or time-filtered aggregate can legitimately omit Bing rather than sending invented parameters.

## Common request defaults

Before provider mutation, online requests use:

    method=GET
    headers={}
    data={}
    json={}
    content=b""
    url=""
    cookies={}
    allow_redirects=false
    max_redirects=0
    soft_max_redirects=0
    auth=None
    verify=None
    raise_for_httperror=true

The generic layer adds `Accept-Language` when enabled.

Provider request builders then mutate this request. A builder can intentionally leave `url` empty/none, in which case no network call is made.

## Provider request/parse summaries

### Google

    prepare locale/safe/time/page
      -> GET WML endpoint
      -> Nokia UA + chrome99_android
      -> generic HTTP layer
      -> Google sorry/302 checks
      -> strip optional XML declaration
      -> XPath result parsing
      -> /url?q= unwrap
      -> typed main results

Individual malformed result blocks are isolated and skipped.

### Bing

    prepare q/adlt/setlang/cc
      -> GET /search
      -> default browser transport, conditional HTTP3
      -> generic HTTP layer
      -> b_algo parsing
      -> optional ck/a base64url decode
      -> legacy main-result dictionaries

Malformed recognized base64 can escape the parser and become provider-level failure.

### Primary DuckDuckGo

    preprocess query / stable UA
      -> POST HTML endpoint
      -> region/time form+cookies
      -> first-page or vqd continuation fields
      -> generic HTTP layer
      -> 303 empty OR challenge-form check
      -> cache hidden vqd
      -> web-result extraction
      -> typed main results
      -> optional zero-click answer

## Provider-local position semantics

Positions are assigned by the common result container, not from DOM indexes.

For each provider call:

    main_count = 0

For each accepted normal main result:

    main_count += 1
    position = main_count

Side channels do not increment the main position. Provider parser items skipped before insertion do not increment it either.

This position then immediately participates in duplicate merge.

## Normalization before merge

Every main result is normalized before hashing:

- provider provenance initialization;
- whitespace collapse;
- title/content limits;
- content==title clearing;
- URL parsing/default-scheme/IDNA behavior.

Do not calculate identity from raw parser output and normalize afterward; that changes duplicates.

## Global merge while providers finish

There is one shared identity map. Results are inserted as provider workers finish and are extended into the container.

No separate stage waits for all provider lists and then pre-deduplicates each provider. Same-provider and cross-provider duplicates use the same merge function.

This insertion order also supplies stable-sort tie order later, so an async reimplementation that buffers all providers and then processes them in a fixed provider-name order would alter exact tie behavior.

## Shared deadline

All eligible provider workers start before waits. Every wait/network operation consumes time relative to one search start.

A worker that exceeds the caller's shared deadline is marked timed out. If it later finishes, common insertion logic sees the timeout marker and does not add its result set.

A native async implementation should reproduce accepted-result timing semantics even if it can cancel work more effectively.

## Error path

Expected provider/transport exceptions are captured inside the provider worker. The provider is recorded as unresponsive/failed according to common handling. Siblings continue.

A search can therefore return:

    results != []
    plus one or more provider failures

A provider returning a healthy empty list is distinct from a provider exception.

## Aggregation close/order

After accepted providers finish or time out:

1. calculate score for every merged main result;
2. stable sort descending score;
3. run the grouping pass using category, template, image marker, group count 8, distance 20;
4. project/limit results.

Skipping step 3 is not parity.

## Final host boundary

Provider-compatible aggregation can be followed by Overmind-specific output checks, such as restricting schemes before exposing URLs to another capability. Such policies must not feed back into compatibility ranking unless intentionally designed and separately tested.

## Pipeline golden test

A single integration fixture should execute fake Google/Bing/DDG results through the full sequence and assert:

- provider accepted positions;
- normalization;
- identities;
- same-provider/cross-provider merges;
- exact scores;
- stable equal-score insertion behavior;
- final grouping order;
- final output limit;
- sibling survival on one provider failure;
- late result rejection.

That integration fixture is the strongest guard against future “cleanup” refactors accidentally changing search behavior.