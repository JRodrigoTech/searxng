# Target architecture

## Design goal

Build a small embedded search library with one public search operation and three primary provider adapters. The architecture may be idiomatic for the host runtime, but the behavior inside the compatibility pipeline must reproduce the audited provider, normalization, merge, ranking, deadline, and failure semantics.

The target is not a search website and not a server. It is a callable tool implementation.

## Component diagram

    Search Tool API
        |
        v
    SearchCoordinator
        |
        +--> Capability/Suspension Gate
        +--> Shared Deadline
        +--> LocaleTraits
        +--> Provider Registry
                    |
                    +--> GoogleProvider
                    +--> BingProvider
                    +--> DuckDuckGoHtmlProvider
                    |
                    +--> SearchTransport
                    +--> ProviderState
        |
        v
    CommonNormalizer
        |
        v
    IdentityMerge
        |
        v
    CompatibilityRanker
        |
        v
    CompatibilityGrouping
        |
        v
    Host Output Policy
        |
        v
    SearchResponse

## Architectural invariant: provider details stop at provider boundary

The coordinator must not contain provider-specific:

- endpoints;
- HTML/XPath selectors;
- cookies;
- wrapper formats;
- CAPTCHA signatures;
- User-Agent lists;
- continuation-token fields.

Each provider owns those details.

The aggregation layer likewise must not know how a result was scraped; it sees normalized result fields, provider identity, position, and ranking metadata.

## Search Tool API

Responsibilities:

- validate caller input;
- map the public call to an internal `SearchRequest`;
- invoke the coordinator;
- apply only explicit host output/security policy after compatibility ordering;
- serialize a bounded result for the agent.

It must not:

- fetch result destinations;
- reinterpret snippets as instructions;
- expose provider cookies/tokens;
- select arbitrary provider protocol fields supplied by the model.

## SearchCoordinator

Responsibilities:

- capture the common monotonic start;
- derive the common search timeout;
- check provider capability and suspension state;
- start all eligible providers concurrently;
- isolate failures;
- reject late results;
- feed accepted provider outputs through normalization and aggregation in accepted insertion/completion order;
- close/rank/group the aggregate;
- apply final result limit.

The source implementation uses worker threads. The target can use asyncio directly as long as shared-deadline, partial-failure, late-result, and parity-sensitive insertion-order behavior are preserved.

## Provider contract

A provider needs conceptual operations equivalent to:

    capabilities(request) -> eligible | unsupported
    prepare(request, context) -> PreparedRequest | no-request | failure
    execute(prepared, transport, state, deadline) -> ProviderOutput

The actual Python interface may combine them, but tests must separately verify request preparation and response parsing.

## SearchTransport

Compatibility responsibilities:

- curl-compatible async HTTP;
- browser/TLS impersonation profiles;
- HTTP/2 and conditional HTTP/3;
- explicit cookies/headers;
- redirect policy;
- pooled client reuse keyed by transport settings;
- shared remaining-time propagation;
- generic HTTP error classification;
- retry behavior.

Transport must not know:

- result XPath selectors;
- provider wrapper decoding;
- global ranking;
- prompt/security interpretation of snippets.

## LocaleTraits

The locale layer owns:

- provider trait snapshots;
- all-locale sentinels;
- provider aliases;
- best-fit language/region mapping.

It must produce exactly the provider codes expected by adapter request builders.

Trait refresh is an offline/background maintenance concern. Ordinary search must work from the bundled snapshot.

## ProviderState

Primary DuckDuckGo requires provider-scoped TTL state for `vqd`. The cache identity includes the transformed query and stable User-Agent.

The state boundary must prevent:

- cross-provider state access;
- accidental User-Agent/token mismatch;
- unbounded cache growth;
- token exposure through public results.

An optional secondary DDG adapter uses additional page-URL cache state, but it is not part of primary V1.

## CommonNormalizer

Responsibilities are exact compatibility transformations:

- common text whitespace/length rules;
- duplicate content/title suppression;
- parsed URL initialization;
- missing-scheme behavior;
- observed IDNA behavior;
- provider provenance initialization.

Do not add product canonicalization here.

## IdentityMerge

Responsibilities:

- construct an identity equivalent to the audited fields;
- detect both same-provider and cross-provider duplicates through one global map;
- append every accepted duplicate position;
- merge title/content/default fields/provenance/scheme exactly.

No one-vote-per-provider policy belongs here in parity mode.

## CompatibilityRanker

Responsibilities:

- configured provider weights;
- exact weight-product and position-count formula;
- priority behavior;
- score assignment when aggregation closes;
- first-pass stable descending score sort.

No alternate consensus formula is used in the default compatibility path.

## CompatibilityGrouping

After score sort, reproduce the second ordering pass using:

- primary provider category;
- template;
- image marker from thumbnail/img_src;
- group capacity 8;
- max distance 20.

This is a distinct stage and should have its own tests because omitting it changes output order.

## Host Output Policy

Only after compatibility ordering should Overmind-specific controls act on agent-facing output, for example:

- reject/quarantine unsafe URL schemes;
- enforce final serialized field bounds;
- attach labelled provenance;
- remove internal diagnostic state.

Keeping this boundary after compatibility aggregation allows security hardening without corrupting provider/ranking parity tests.

## Data ownership

| Data | Owner | Lifetime |
| --- | --- | --- |
| Query/options | SearchRequest | one search |
| Provider capabilities | provider | process/configuration lifetime |
| Trait snapshot | LocaleTraits | release/process lifetime |
| HTTP pools | SearchTransport | runtime lifetime |
| Stable DDG User-Agent | provider instance/module-equivalent | runtime lifetime |
| DDG `vqd` | ProviderState | 3600 s |
| Raw provider response | provider execution | one request |
| Normalized result | compatibility pipeline | one search |
| Aggregated result | compatibility pipeline | one search |
| Suspension state | provider health layer | across searches |
| Agent-facing result | tool response | one call |

## Primary V1 versus optional later work

### Primary V1

- Google web adapter;
- Bing web adapter;
- DuckDuckGo HTML adapter;
- curl-compatible transport profiles;
- trait mapping;
- shared deadline;
- exact normalization/dedupe/merge/ranking/grouping;
- provider failure isolation;
- tool serialization.

### Optional later

- secondary DuckDuckGo JSON/script adapter;
- trait refresh command/job;
- richer provider diagnostics;
- persistent cache if V1 initially uses in-memory state;
- alternative ranking modes clearly separated from compatibility.

## Non-goals

- web server/UI;
- arbitrary fetch/crawler;
- provider plugin ecosystem;
- browser automation;
- search index;
- generic JavaScript runtime;
- copying a larger application's application/session layers.

## Architecture acceptance criteria

A new implementation is architecturally correct only if provider protocol tests and global compatibility tests can be run separately. Replacing an internal mechanism is allowed; changing its observable behavior is not, unless recorded as an explicit deviation.