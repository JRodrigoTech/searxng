# Open questions and live validation plan

## Purpose

The implementation mechanics are now specified by compatibility behavior. The remaining unknowns are primarily external: public search endpoints, markup, challenge behavior, transport acceptance, and provider-maintained locale resources can change without our code changing.

These questions must not be used as an excuse to redesign documented behavior before the parity suite is implemented.

## 1. Google endpoint health

Validate from the intended deployment egress:

- `https://www.google.com/wml/search` remains reachable;
- Nokia User-Agent + `chrome99_android` still returns expected result markup;
- the current result classes remain present;
- redirect-disabled 302 and `/sorry` signals still correspond to block/challenge behavior;
- locale parameters continue to produce expected language/region behavior.

If not, preserve old fixtures and investigate protocol drift before changing the adapter.

## 2. Bing HTML health

Validate:

- `/search` still returns `b_results/b_algo` structure;
- `ck/a?u=a1...` remains the common wrapper format where used;
- `setlang/cc` behavior remains effective;
- the `us/cn/ru` cc exclusions remain desirable;
- HTTP/3 is accepted/beneficial from intended egress;
- healthy zero-result pages can still be distinguished operationally from block pages through status/telemetry.

Do not invent page/time support merely because Bing's website supports it interactively.

## 3. Primary DuckDuckGo HTML health

Validate:

- `html.duckduckgo.com/html/` accepts the documented form/headers;
- a stable generated User-Agent remains compatible with continuation `vqd`;
- hidden `vqd` is still emitted when continuation is available;
- 3600-second token reuse remains accepted;
- `kl` form/cookie region behavior remains valid;
- `df` form/cookie time behavior remains valid;
- continuation offsets remain 10 then +15;
- `challenge-form` remains the challenge marker;
- Chinese continuation restriction is still necessary;
- 303 remains a meaningful empty outcome.

If continuation becomes unreliable, first-page-only operation is a safe capability reduction; do not send tokenless continuation requests.

## 4. DuckDuckGo safe-search semantics

The primary HTML adapter advertises safe-search capability but does not add a safe-search request value from the numeric setting in the audited builder.

Live research can determine whether the endpoint currently has a stable explicit field/cookie, but adding one would be a new protocol feature, not parity with the audited path. Until deliberately implemented, do not claim strict DDG safe-search enforcement at request level.

## 5. Secondary DuckDuckGo adapter viability

This optional adapter is real but disabled in the audited default configuration. Before implementing it for Overmind, validate:

- discovery page still exposes `deep_preload_link`;
- opaque `dp`/API URL behavior still requires discovery;
- JSON `results` still exposes `u/t/a` and continuation `n`;
- the arithmetic challenge grammar remains within the documented narrow parser;
- Firefox impersonation remains required/accepted.

This is not a blocker for primary V1.

## 6. Trait freshness

The compatibility design should ship generated provider trait snapshots. Establish a maintenance process for validating/updating them.

Questions:

- how often should snapshots be refreshed;
- should refresh run manually in development/CI or as a release task;
- which locales Overmind promises to support in tests;
- how to detect provider alias changes without adding runtime latency.

The per-query search path should not depend on live trait scraping.

## 7. Cache persistence decision

Primary DDG source behavior uses persistent provider cache state. Overmind may prefer in-memory TTL state initially.

Decision needed:

- exact persistence across runtime restarts, or
- process-local parity only.

If in-memory is chosen, document it as an operational deviation; request/state semantics within one runtime remain exact.

## 8. Hash-key representation decision

The audited result map uses the Python integer `hash(result)` directly as the dictionary key. This admits theoretical hash-collision merging.

For an independent implementation, a structured tuple containing the exact identity fields can preserve all ordinary duplicate semantics while eliminating collision ambiguity.

Decision needed:

- reproduce integer-hash storage literally, or
- use a structured identity key and classify it as a safety/internal deviation.

Recommendation for implementation engineering: structured key is acceptable only if parity tests prove identical behavior for all non-collision cases and documentation clearly records the difference.

## 9. Equal-score tie behavior

Exact source ordering uses stable sort with insertion order and no explicit tie-break. Native async task completion can therefore affect equal-score ordering.

Decide whether V1 must reproduce this exactly or whether Overmind wants a deterministic tie-break as a named later ranking mode.

The compatibility mode should preserve insertion semantics until an explicit product decision changes it.

## 10. Host URL output policy

Provider parity can produce decoded/normalized URLs without strict HTTP(S)-absolute validation. Before returning results to the agent, Overmind should define whether it:

- exposes parity output unchanged;
- filters non-HTTP(S) values only at serialization;
- marks unsafe/unusual URLs as omitted/quarantined.

Whatever policy is chosen must remain downstream of parity aggregation so it does not silently change score/dedup tests.

## 11. Body/resource limits

The audited source does not implement the custom response-body limits proposed in the earlier research draft. Overmind should choose practical limits for runtime safety based on measured provider pages.

Validate normal response-size distributions before setting caps. A host cap should fail a provider cleanly rather than truncate markup and emit plausible-but-wrong results.

## 12. Live smoke-test cadence

Define a low-rate schedule outside mandatory unit CI. Each run should record only safe operational facts:

    provider
    status
    content type
    body size
    elapsed time
    expected structure present
    result count
    challenge/block classification

Do not store raw queries, cookies, validation tokens, or full response bodies in routine telemetry.

## Implementation readiness gate

The primary V1 can begin now. None of the unresolved questions above prevents implementation because the documented parity behavior is sufficient for:

- transport;
- Google/Bing/DDG requests;
- parsers;
- primary DDG continuation state;
- normalization;
- duplicate merge;
- ranking/grouping;
- concurrency/deadlines;
- failure isolation.

Before declaring the tool production-ready, run live smoke validation from the actual deployment network and resolve any external protocol drift discovered there.