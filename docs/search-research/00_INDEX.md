# Clean-room web-search subsystem: documentation index

## Objective

This document set specifies an independent Python web-search subsystem for an AI-agent tool. It contains two deliberately separated targets:

1. **Full compatibility reference** — documents the audited mechanics of Google Web Search, Bing Web Search, the primary DuckDuckGo HTML path, common transport, normalization, deduplication, ranking, deadlines, failure isolation, locale traits, and continuation behavior.
2. **Simplified Overmind V0** — the implementation target to build first: one bounded `web.search(query, limit)` Tool using first-page Google, Bing, and primary DuckDuckGo concurrently while preserving the audited provider mechanics and exact aggregation/ranking behavior that still apply.

The implementation is independent in package names and architecture. Compatibility concerns observable behavior and algorithms, not copying another application's modules, server, UI, or framework.

## Documentation rule

Each technical statement belongs to one of these categories:

- **PARITY MUST** — reproduce this behavior when the corresponding compatibility feature is implemented.
- **Audited behavior** — verified behavior from the analyzed executable code path.
- **V0 contract** — exact behavior required by the first Overmind implementation, including deliberate feature reductions.
- **Host policy / deliberate deviation** — a useful target-runtime change that is not source parity.
- **Live validation** — external provider behavior that can drift even when our implementation is correct.

When a general/full-reference document conflicts with `18_SIMPLIFIED_V0_CONTRACT.md` on whether a feature exists in V0, the V0 contract decides **scope**. The provider/common documents still decide the **mechanics** of every behavior V0 retains.

## Full-reference target flow

    search(request)
       |
       v
    capability + suspension filtering
       |
       v
    shared monotonic search deadline
       |
       +--> Google
       +--> Bing
       +--> DuckDuckGo HTML
              (concurrently)
       |
       v
    provider-specific parsing
       |
       v
    exact common normalization
       |
       v
    global identity merge
       |
       v
    exact score calculation
       |
       v
    score sort + grouping pass
       |
       v
    final result limit
       |
       v
    explicit host output/security policy
       |
       v
    SearchResponse to agent

## Simplified Overmind V0

Public Tool:

    web.search({
      "query": string,
      "limit": integer = 10
    })

Fixed internal profile:

    page = 1
    locale = all
    safe_search = 0
    time_range = None
    shared_timeout = 3.0 seconds
    providers = google + bing + primary duckduckgo HTML
    provider weights = 1.0

V0 intentionally does not expose locale, paging, safe search, time range, external bangs, secondary DDG, or arbitrary result-page fetching. These are explicit scope reductions, not undocumented omissions.

## Primary provider set

### Google

- mobile/WML public representation;
- GET;
- Nokia User-Agent selected from a fixed set per request;
- Android Chrome 99 transport impersonation;
- V0 fixed all-locale request resolves to `hl=en`, empty `lr`, empty `cr`;
- explicit `/sorry`/302 challenge detection;
- XPath extraction and Google redirect decoding.

### Bing

- standard HTML web search;
- GET;
- V0 request uses `q` + `adlt=off` and omits `setlang`, `cc`, `mkt` for all-locale;
- no paging/time support in this adapter;
- provider transport can prefer HTTP/3 when available;
- `b_algo` extraction;
- `ck/a?u=a1...` decoding.

### DuckDuckGo primary

- no-JavaScript HTML endpoint;
- POST form;
- stable generated Firefox-formatted User-Agent built from the exact audited OS/version corpus;
- navigation fetch headers;
- V0 sends first-page `q`, empty `b`, `kl=wt-wt`;
- V0 still may capture hidden `vqd` but does not require continuation state to search page one;
- HTML challenge detection;
- direct result extraction.

A separate disabled-by-default DuckDuckGo JSON/script adapter is documented in `06_DUCKDUCKGO_WEB_PROTOCOL.md`; it is not part of V0.

## Documentation map

| File | Purpose |
| --- | --- |
| `00_INDEX.md` | Scope, precedence, reading order, completion gates |
| `01_TARGET_ARCHITECTURE.md` | Independent component architecture while preserving compatibility semantics |
| `02_SEARCH_PIPELINE.md` | Full-reference end-to-end execution and phase order |
| `03_PROVIDER_CONTRACT.md` | Common provider/transport/capability contract |
| `04_GOOGLE_WEB_PROTOCOL.md` | Exact Google request, fingerprint, parser, block, unwrap behavior |
| `05_BING_WEB_PROTOCOL.md` | Exact Bing request/parser/wrapper/failure behavior |
| `06_DUCKDUCKGO_WEB_PROTOCOL.md` | Primary HTML path plus secondary JSON/script path |
| `07_HTTP_TRANSPORT.md` | curl/browser impersonation, HTTP versions, pooling, retries, deadlines |
| `08_CONCURRENCY_AND_DEADLINES.md` | Shared deadline, concurrent dispatch, late-result suppression |
| `09_RESULT_MODEL_AND_NORMALIZATION.md` | Exact common result normalization and positions |
| `10_URL_IDENTITY_AND_DEDUPLICATION.md` | Exact identity fields and global dedupe behavior |
| `11_MERGE_AND_RANKING.md` | Exact merge, score formula, stable sort, grouping pass |
| `12_FAILURE_AND_RESILIENCE.md` | Failure isolation and suspension timings |
| `13_SECURITY_BOUNDARIES.md` | Untrusted data and separation of parity from host hardening |
| `14_TEST_STRATEGY.md` | Compatibility golden suite and live smoke tests |
| `15_DEPENDENCIES.md` | Compatibility dependency baseline |
| `16_IMPLEMENTATION_BLUEPRINT.md` | Full-reference architecture plus explicit V0 implementation path |
| `17_OPEN_QUESTIONS.md` | External/full-reference questions that are not V0 blockers |
| `18_SIMPLIFIED_V0_CONTRACT.md` | **Normative first implementation contract** |
| `19_OVERMIND_INTEGRATION.md` | **Normative current Overmind Tool/RuntimeStargate integration contract** |
| `20_REFERENCE_VECTORS.md` | **Deterministic request/parser/merge/rank/deadline test vectors** |
| `21_IMPLEMENTATION_DRY_RUN.md` | **Coding-agent build order and pre-merge audit** |

## Reading order for the V0 coding agent

The coding agent implementing Overmind V0 should read in this order:

1. `18_SIMPLIFIED_V0_CONTRACT.md`
2. `19_OVERMIND_INTEGRATION.md`
3. `21_IMPLEMENTATION_DRY_RUN.md`
4. `20_REFERENCE_VECTORS.md`
5. `04_GOOGLE_WEB_PROTOCOL.md`
6. `05_BING_WEB_PROTOCOL.md`
7. `06_DUCKDUCKGO_WEB_PROTOCOL.md` — primary HTML sections only for V0
8. `07_HTTP_TRANSPORT.md`
9. `09_RESULT_MODEL_AND_NORMALIZATION.md`
10. `10_URL_IDENTITY_AND_DEDUPLICATION.md`
11. `11_MERGE_AND_RANKING.md`
12. `08_CONCURRENCY_AND_DEADLINES.md`
13. `12_FAILURE_AND_RESILIENCE.md`
14. `13_SECURITY_BOUNDARIES.md`
15. `14_TEST_STRATEGY.md`
16. `15_DEPENDENCIES.md`
17. `16_IMPLEMENTATION_BLUEPRINT.md` for later/full-reference context
18. `17_OPEN_QUESTIONS.md`

## Compatibility-critical facts that V0 must not simplify away

- Google safe-search level 0 means the `safe` parameter is absent.
- Google combines a Nokia HTTP User-Agent with `chrome99_android` transport impersonation.
- Google wrapper decoding splits literal `&sa=U` before percent-decoding.
- V0 all-locale Google resolves to `hl=en`, `lr=`, `cr=`.
- V0 all-locale Bing sends `q/adlt=off` and omits `setlang`, `cc`, and `mkt`.
- Primary DuckDuckGo uses one stable generated User-Agent for runtime/provider lifetime.
- V0 DuckDuckGo still performs its provider-specific whitespace collapse before POST.
- Search providers share one deadline; a late worker cannot add results.
- Cancellation closes result admission and remains runtime cancellation rather than `SEARCH_UNAVAILABLE`.
- Common normalization happens before identity/merge.
- Ordinary result identity excludes scheme but includes netloc/path/params/query/fragment/img_src/template.
- Same-provider duplicates are not pre-collapsed; they can append additional positions.
- Ranking uses product of engine weights, number of positions, and reciprocal positions; with V0 unit weights this reduces to `len(positions) * sum(1/position)`.
- Score sorting is followed by the separate category/template/image grouping pass.
- Equal-score parity has no explicit canonical tie-break; insertion order can matter.
- Caller `limit` is applied after score sort and grouping, not before aggregation.

## V0 checklist

- [ ] `WebSearchTool` follows current synchronous Tool contract.
- [ ] Canonical identity `tools/web_search`, operation `web.search`.
- [ ] Exact built-in descriptor and first-party Tool authorization added.
- [ ] Public schema only `query` and optional `limit`.
- [ ] Google, Bing, and primary DuckDuckGo page-one adapters.
- [ ] `curl_cffi` browser impersonation + `lxml` parser available in every runtime packaging surface.
- [ ] One shared monotonic 3-second budget and partial failure behavior.
- [ ] Cooperative Overmind cancellation during coordinator waits.
- [ ] Deterministic vectors from `20_REFERENCE_VECTORS.md` encoded as tests before live transport work.
- [ ] Exact common text/URL normalization.
- [ ] Exact global identity/dedup semantics.
- [ ] Exact merge and score formula.
- [ ] Exact final grouping pass.
- [ ] Failure/suspension behavior represented in process memory.
- [ ] Final output cap applied only after aggregation/order.
- [ ] Provider data marked untrusted at the Tool boundary.
- [ ] Search does not fetch arbitrary result destinations.
- [ ] Optional low-rate live smoke suite from intended deployment egress.

## V0 definition of done

V0 is implementation-ready when a new coding agent can build it using only this documentation set plus the target Overmind repository, without reopening the analyzed search repository.

V0 is implementation-complete when:

- all deterministic vectors pass offline;
- Overmind registration/authorization/cancellation tests pass;
- dependency packaging/locks are correct for the supported runtime;
- all three provider adapters obey their exact page-one request/parser contracts;
- partial failures and healthy-empty providers are distinguished correctly;
- no late provider can mutate accepted results;
- output is bounded and JSON serializable;
- optional live smoke tests show current endpoint viability from deployment egress.

## Full-reference definition of done

Full compatibility extends beyond V0 and additionally requires tested locale trait snapshots, paging where supported, primary DDG continuation state, exact bang recognition if exposed, and any other full-surface capabilities explicitly selected for implementation.

External search services can change independently. A live failure does not automatically mean the implementation is wrong; deterministic fixture parity and live protocol health must be diagnosed separately.