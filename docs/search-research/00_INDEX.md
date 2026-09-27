# Clean-room web-search subsystem: documentation index

## Objective

This document set specifies an independent Python web-search subsystem for an AI-agent tool. The first implementation target is **behavioral compatibility** with the audited mechanics for Google Web Search, Bing Web Search, and the primary DuckDuckGo HTML web path.

The system sends eligible providers concurrently, normalizes their replies, merges duplicate destinations, reproduces the observed ranking/order logic, and returns a bounded structured response to the caller.

The implementation is independent in package names and architecture. The compatibility target concerns observable behavior and algorithms, not copying another application's modules or server/UI infrastructure.

## Documentation rule

Each technical statement belongs to one of these categories:

- **PARITY MUST** — reproduce this behavior in the compatibility implementation.
- **Audited behavior** — verified behavior from the analyzed executable code path.
- **Host policy / deliberate deviation** — a useful change for the target AI runtime that is not source parity.
- **Live validation** — external provider behavior that can drift even when our implementation is correct.

When recommendations conflict with audited behavior, parity wins for the first implementation unless the deviation is explicitly named and tested.

## Target flow

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

## Primary provider set

### Google

- mobile/WML public representation;
- GET;
- Nokia User-Agent selected from a fixed set;
- Android Chrome 99 transport impersonation;
- language/region/time/safe mappings;
- explicit `/sorry`/302 challenge detection;
- XPath extraction and Google redirect decoding.

### Bing

- standard HTML web search;
- GET;
- `q/adlt/setlang/cc` request shape;
- no paging/time support in this adapter;
- optional HTTP/3 transport behavior;
- `b_algo` extraction;
- `ck/a?u=a1...` decoding.

### DuckDuckGo primary

- no-JavaScript HTML endpoint;
- POST form;
- stable generated User-Agent;
- navigation fetch headers;
- region/time form+cookie state;
- `vqd` continuation state bound to transformed query + User-Agent;
- HTML challenge detection;
- direct result extraction.

There is also a separate disabled-by-default DuckDuckGo JSON/script adapter. It is documented completely in `06_DUCKDUCKGO_WEB_PROTOCOL.md` but is not required for the initial three-provider V1.

## Documentation map

| File | Purpose |
| --- | --- |
| `00_INDEX.md` | Scope, parity rules, reading order, completion gate |
| `01_TARGET_ARCHITECTURE.md` | Independent component architecture while preserving compatibility semantics |
| `02_SEARCH_PIPELINE.md` | Exact end-to-end execution and phase order |
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
| `16_IMPLEMENTATION_BLUEPRINT.md` | Concrete package/API/build sequence |
| `17_OPEN_QUESTIONS.md` | Only external/live questions still needing validation |

## Reading order for the implementation agent

1. `00_INDEX.md`
2. `16_IMPLEMENTATION_BLUEPRINT.md`
3. `07_HTTP_TRANSPORT.md`
4. `04`, `05`, `06` provider protocols
5. `09`, `10`, `11` normalization/dedupe/ranking
6. `08` concurrency/deadlines
7. `12` failure/suspension
8. `03` provider contract
9. `14` tests
10. `13`, `15`, `17`

## Compatibility-critical facts that must not be simplified away

- Google safe-search level 0 omits the `safe` parameter rather than sending `safe=off`.
- Google combines a Nokia HTTP User-Agent with `chrome99_android` transport impersonation.
- Google wrapper decoding splits `&sa=U` before percent-decoding.
- Bing does not use its `mkt` helper in ordinary web search.
- Bing malformed recognized base64 wrappers can fail the provider because decoding is not locally isolated.
- Primary DuckDuckGo uses a stable User-Agent and a 3600-second query+UA-bound `vqd` cache.
- Primary DuckDuckGo page 1 and continuation forms differ materially.
- Search providers share one deadline; a late worker cannot add results.
- Common normalization happens before identity/merge.
- Ordinary result identity excludes scheme but includes netloc/path/params/query/fragment/img_src/template.
- Same-provider duplicates are not pre-collapsed; they can append additional positions.
- Ranking uses product of engine weights, number of positions, and reciprocal positions.
- Score sorting is followed by a separate category/template/image grouping pass.
- Equal-score parity has no explicit canonical tie-break; insertion order can matter.

## V1 checklist

- [ ] One async callable tool-facing search entry point.
- [ ] Google, Bing, and primary DuckDuckGo adapters.
- [ ] `curl_cffi`-class browser impersonation behavior.
- [ ] Generated/verified locale trait snapshot and best-fit mapping.
- [ ] Shared monotonic deadline and partial failures.
- [ ] Exact provider request/parse golden fixtures.
- [ ] Exact common text/URL normalization.
- [ ] Exact global identity/dedup semantics.
- [ ] Exact merge and score formula.
- [ ] Exact final grouping pass.
- [ ] Primary DuckDuckGo TTL state and continuation support.
- [ ] Failure/suspension behavior represented accurately.
- [ ] Final output cap applied only after aggregation/order.
- [ ] Provider data marked untrusted at the tool boundary.
- [ ] Search does not fetch arbitrary result destinations.
- [ ] Optional live smoke suite from deployment egress.

## Definition of done

Documentation is implementation-ready when a new coding agent can build the primary tool without reopening the analyzed source repository and can reproduce all parity golden tests.

Implementation is parity-ready when all fixture/request/normalization/dedupe/ranking/deadline tests pass and live smoke tests demonstrate that current provider endpoints still accept the documented protocol from the intended deployment network.

External search services can change independently. A live failure does not automatically mean the implementation is wrong; fixture parity and live protocol health must be diagnosed separately.