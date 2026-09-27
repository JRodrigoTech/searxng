# Open questions and live validation plan

## Purpose

The simplified Overmind V0 is now fully specified for implementation. Remaining unknowns fall into two classes:

- **operational/live uncertainties** — external providers may drift or block the documented protocol;
- **later/full-compatibility decisions** — locale traits, paging, persistence, external bangs, or optional providers that V0 intentionally does not expose.

None of the items below is a specification blocker for simplified V0 unless explicitly marked otherwise.

## V0 blocker status

Current status:

    specification blockers: none

The V0 coding agent can implement solely from this documentation set plus the target Overmind repository.

What still requires live validation is whether external services currently accept the documented fingerprints from the intended deployment network.

## 1. Google live endpoint health

Validate from intended deployment egress:

- `https://www.google.com/wml/search` remains reachable;
- Nokia User-Agent + `chrome99_android` still returns expected markup;
- current result classes remain present;
- redirect-disabled 302 and `/sorry` signals still correspond to block/challenge behavior.

For V0 use the fixed all-locale vector from `18`/`20`; broad locale behavior is later work.

If live behavior differs, preserve offline fixture/vector expectations until protocol drift is understood. Do not silently mutate parser/request logic from one failed live run.

## 2. Bing live endpoint health

Validate:

- `/search` returns the documented `b_results/b_algo` structure;
- `ck/a?u=a1...` remains used where wrapper links appear;
- V0 all-locale `q + adlt=off` remains accepted;
- HTTP/3 is available/beneficial on target packaging if desired.

HTTP/3 is not a V0 correctness blocker. If unavailable, record HTTP/2-only transport as a named transport deviation while preserving request/parser semantics.

## 3. Primary DuckDuckGo HTML health

Validate:

- `html.duckduckgo.com/html/` accepts the documented POST form/headers;
- stable generated User-Agent remains accepted;
- hidden `vqd` is still emitted where observed;
- `challenge-form` remains a meaningful challenge marker;
- 303 remains the documented healthy-empty outcome.

V0 uses page one only, so continuation viability is not a V0 blocker.

## 4. Full-reference locale trait snapshot

Not required for V0 because locale is fixed to `all`.

Later compatibility work should ship generated provider trait snapshots and tests. Decide:

- refresh cadence;
- release/CI/manual generation workflow;
- promised locale set;
- drift detection.

Per-query live trait scraping should remain unnecessary.

## 5. DuckDuckGo continuation state

Not required for V0.

Later validate:

- hidden `vqd` extraction;
- query + stable-UA binding;
- 3600-second reuse;
- continuation offsets;
- Chinese continuation restriction;
- persistence choice.

A failure of continuation must never cause V0 to send tokenless page-two requests.

## 6. External bang support

V0 rejects whitespace-delimited tokens beginning with `!` and does not ship the external-bang dataset.

Later support requires a maintained bang recognition table if exact quoting behavior is desired. Do not approximate “recognized bang” with arbitrary prefix matching while claiming compatibility.

## 7. DuckDuckGo safe-search semantics

Not exposed in V0.

The primary HTML adapter advertises safe capability but the audited builder does not add a request value from the numeric safe setting. Any future explicit DDG safe field/cookie would be a new protocol feature unless directly supported by a later audited implementation.

## 8. Secondary DuckDuckGo adapter

Not part of V0.

Before later implementation validate:

- discovery page `deep_preload_link`;
- opaque `dp`/API URL discovery;
- JSON `results` fields and continuation;
- narrow arithmetic challenge grammar;
- Firefox impersonation viability.

## 9. Cache persistence

V0 uses process-local health/suspension state and may capture DDG `vqd` in memory only.

Later decide whether provider state must survive runtime restart. Persistence changes operational continuity, not page-one V0 request semantics.

## 10. Identity map representation

The audited implementation uses Python integer `hash(result)` as the map key. A structured tuple of the exact identity fields avoids theoretical hash collision merging.

For V0, either choice is implementable. If a tuple is used, record this as the documented collision-safety deviation and prove all non-collision duplicate vectors remain identical.

This does not block coding.

## 11. Equal-score ties

Compatibility sorting is stable and has no explicit canonical tie-break. Worker completion/insertion order can therefore influence equal-score results.

V0 should preserve this behavior. A deterministic secondary tie-break would be a separate future ranking mode, not an invisible cleanup.

## 12. Host URL output policy

Provider parity can yield URLs without an extra strict HTTP(S)-absolute validation stage.

For V0, do not add a pre-ranking URL cleanup that changes identity/score. If Overmind later chooses output filtering/quarantine, apply it downstream and label it as host policy.

## 13. Response body/resource limits

The audited search path does not provide the custom body limits proposed in early drafts. Overmind may later add host safety caps after measuring normal provider response sizes.

A cap should fail a provider cleanly rather than truncate parser input and emit plausible partial results.

This is not a V0 parity blocker.

## 14. Dependency/platform live check

Before merge, verify the chosen pinned `curl_cffi` and `lxml` versions against Overmind's supported CPython/platform matrix and regenerate the repository's normal dependency lock/manifest surfaces.

This is an implementation/packaging acceptance step, not a search-protocol unknown.

## 15. Live smoke-test cadence

Use low-rate tests outside mandatory offline unit CI. Record only safe operational facts:

    provider
    status
    content type
    body size
    elapsed time
    expected structure present
    result count
    challenge/block classification

Do not store raw queries, cookies, validation tokens, or full response bodies in routine telemetry.

## V0 implementation readiness gate

The V0 implementation can begin now because documentation contains:

- exact public Tool scope;
- exact Overmind registration/authorization path;
- exact page-one request vectors for Google/Bing/DDG;
- exact parser selectors/error boundaries;
- exact common normalization;
- exact duplicate identity and merge;
- exact score/grouping rules;
- shared deadline and late-result suppression;
- cancellation semantics;
- failure isolation and suspension values;
- deterministic synthetic vectors;
- coding-agent dry-run/build order.

The remaining risks are external service drift and normal implementation defects, not missing V0 specification.