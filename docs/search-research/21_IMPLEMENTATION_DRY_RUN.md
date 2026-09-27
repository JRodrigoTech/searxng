# Implementation dry-run: coding-agent perspective

## Purpose

This document simulates the first implementation pass as if the coding agent had no access to the analyzed source repository. It exists to expose missing decisions before code is written.

The dry-run assumes the agent can read only `docs/search-research/` plus the target Overmind repository.

## Can the agent start without reopening the analyzed source?

For simplified V0: **yes**.

The required behavior is now fully specified for:

- public `web.search(query, limit)` contract;
- Overmind Tool identity and registration;
- first-page Google request/parser;
- first-page Bing request/parser;
- first-page primary DuckDuckGo request/parser;
- transport fingerprint requirements;
- common deadline and failure isolation;
- text/URL normalization;
- duplicate identity;
- merge semantics;
- ranking formula;
- second grouping pass;
- bounded response serialization;
- cancellation;
- dependency ownership surfaces;
- deterministic synthetic reference vectors.

Full multi-locale, pagination, external bangs, and secondary DDG remain outside V0 by explicit contract.

# Phase 0 — repository inspection

Before editing code, inspect the target repository for:

    overmind/tools/base.py
    overmind/tools/registry.py
    overmind/extensions/builtin.py
    overmind/runtime_stargate/production_policy.py
    overmind/execution.py
    tests/unit/tools/
    tests/unit/runtime/
    scripts/runtime-manifest.json
    electron/runtime/python-requirements.in
    electron/runtime/python-requirements.lock
    .github/workflows/ci.yml

Confirm current conventions rather than inventing a parallel registration or dependency system.

# Phase 1 — tests before production code

Create pure tests first for all deterministic mechanics.

## 1.1 Normalization tests

Implement no production logic yet. Write expected cases from `09_RESULT_MODEL_AND_NORMALIZATION.md` and `20_REFERENCE_VECTORS.md`:

- whitespace collapse;
- exact title/content truncation;
- content==title clearing;
- non-string URL handling;
- default-scheme behavior;
- IDNA behavior;
- identity-preserved query/fragment/port spelling.

## 1.2 Identity/merge/rank tests

Write:

- HTTP/HTTPS duplicate case;
- www non-duplicate;
- query-order non-duplicate;
- same-provider duplicate case;
- three-provider consensus case;
- exact score values;
- secure-suffixed scheme merge;
- stable ties;
- grouping pass.

These tests define the most valuable reusable core before any live network work.

## 1.3 Provider request/fixture tests

Write fake response/request tests from `20_REFERENCE_VECTORS.md`.

At this stage no internet access is necessary.

## 1.4 Coordinator tests

Use fake provider functions with barriers/events. Prove:

- all workers start;
- one shared 3-second budget;
- sibling survival;
- late-result rejection;
- cancellation propagation.

# Phase 2 — internal models

Implement project-owned data structures. A minimal set is enough:

    SearchRequest
    PreparedRequest
    ProviderResult
    AggregateResult
    ProviderDiagnostic

Do not recreate a large framework.

Important defaults for ordinary V0 main results:

    template = "default.html"
    priority = ""
    category = "general"
    img_src = ""
    thumbnail = ""

Keep `thumbnail` distinct from `img_src` because identity uses `img_src`, while final grouping considers image presence from either thumbnail or img_src.

# Phase 3 — normalization and aggregator

Implement this before providers.

Suggested responsibility boundary:

    normalize.py
      normalize_text()
      normalize_main_result()

    aggregate.py
      identity_key()
      merge_result()
      calculate_score()
      order_results()
      compatibility_group()

For exact collision parity, raw `hash(identity_string)` can be used. For the target project a structured identity tuple is safer and produces identical normal behavior; if selected, mark this as the documented collision-safety deviation and retain a test proving the exact identity fields.

Do not add URL cleanup beyond the specification.

# Phase 4 — transport

Install/pin:

    curl_cffi
    lxml

Do not add Babel in V0.

The first implementation needs a narrow transport wrapper, not a general crawler:

    request(method, url, *, params/data, headers, cookies,
            impersonate, default_headers, allow_redirects,
            http_version_policy, timeout)

The wrapper should:

- keep TLS verification on;
- support provider impersonation;
- disable ordinary search redirects;
- classify common HTTP errors before provider parser invocation;
- avoid implicit cookie accumulation;
- expose only bounded status/body/headers to adapters;
- use remaining shared deadline for timeout.

If implementation uses synchronous `curl_cffi.Session` in one worker per provider, it avoids adding async-loop plumbing to Overmind's synchronous Tool contract. Reuse clients only when settings/fingerprints are identical.

# Phase 5 — Google adapter

Implement strictly from `04_GOOGLE_WEB_PROTOCOL.md` plus V0 narrowing in `18_SIMPLIFIED_V0_CONTRACT.md`.

Recommended test-driven sequence:

1. request URL/query vector;
2. random Nokia UA constrained to exact list;
3. `chrome99_android` profile;
4. sorry/302 detection;
5. optional XML declaration removal;
6. XPath parse fixture;
7. redirect decode fixture;
8. per-block error isolation.

Do not implement locale refresh, paging, safe/time controls, or arbitrary output count in V0.

# Phase 6 — Bing adapter

Sequence:

1. request vector `q/adlt=off` only;
2. no locale params in V0 all-locale mode;
3. result block parser;
4. decorative icon removal;
5. wrapper decode vector;
6. malformed recognized wrapper provider-level failure.

Do not add paging/time support.

HTTP/3 is transport behavior. If exact HTTP/3 support proves awkward on the target platform, preserve request/parser behavior, record HTTP/2-only as an explicit transport deviation, and keep a future compatibility test. Do not block V0 logic on QUIC availability.

# Phase 7 — DuckDuckGo adapter

V0 first-page implementation does not need continuation state to function, but should still parse/cache `vqd` if present.

Sequence:

1. stable UA generation from exact OS/version data;
2. query whitespace normalization;
3. first-page form/header vector;
4. no `kl`/`df` cookies in all-locale/no-time mode;
5. challenge-form classification;
6. 303 empty behavior;
7. `vqd` extraction/cache;
8. exact `web-result` parsing;
9. zero-click output ignored by public V0 projection.

Do not import the external-bang table. V0 rejects bang-like public syntax before provider execution.

# Phase 8 — coordinator

Construct one coordinator object owned by the Tool/service.

Suggested algorithm:

    def search_sync(request, cancellation):
        start = monotonic()
        deadline = start + 3.0
        accepted = shared aggregator
        health = provider health state

        build eligible provider workers
        start all workers

        while unfinished:
            cancellation.raise_if_cancelled()
            remaining = deadline - monotonic()
            if remaining <= 0:
                mark unfinished timed out
                break
            poll/join for min(remaining, small_quantum)

        cancellation.raise_if_cancelled()
        close aggregator
        order/group
        limit
        return response

Each worker checks an admission flag under lock before extending the aggregator. Coordinator closes admission on timeout/cancellation.

This is easier to reason about than letting threads mutate state after caller completion.

# Phase 9 — failure/suspension state

Use process-memory health state in V0.

Represent at least:

    suspended_until
    reason

Classify generic transport/provider exceptions according to `12_FAILURE_AND_RESILIENCE.md`.

Do not retry CAPTCHA/access denied aggressively.

Successful accepted completion clears provider suspension state according to documented parity behavior.

# Phase 10 — Tool wrapper

Implement:

    WebSearchTool.execute(arguments, cancellation=...)

Validation occurs before constructing a network search.

Return partial success whenever at least one provider succeeds and produces/validly returns a result set, even if siblings fail.

A provider healthy-empty is not the same as failure. If all providers are healthy-empty, the Tool may return:

    ok = true
    results = []

`SEARCH_UNAVAILABLE` is reserved for a condition where every eligible provider fails, times out, or is suspended rather than merely returning no result.

This distinction must be tested.

# Phase 11 — registration and authority

Update only existing ownership points:

- built-in Tool descriptor;
- first-party Tool operation policy.

Do not touch Agent or ContextCompiler to know about search.

Test exact Tool Surface authorization.

# Phase 12 — dependencies and packaging

Update direct runtime dependency declarations and the portable-runtime lock closure atomically.

The coding agent should expect to touch at least:

    electron/runtime/python-requirements.in
    electron/runtime/python-requirements.lock
    scripts/runtime-manifest.json
    .github/workflows/ci.yml

plus any baseline/verification files that explicitly assert the dependency set.

Do not hand-edit a generated lock if the repository has an established command for regenerating it; use the repository's documented dependency workflow.

# Phase 13 — live smoke tests

Only after unit/golden tests pass, perform optional network tests:

    query = "python asyncio taskgroup"

Assertions should be structural, not content-rank brittle:

- at least one provider can complete from deployment egress;
- provider result URLs are parseable;
- Google endpoint does not immediately enter sorry/302 path;
- Bing parser finds ordinary result blocks if Bing responds normally;
- DDG returns either ordinary results, healthy-empty, or an explicitly classified block state;
- aggregate output is bounded and serializable.

Live provider content is not a golden fixture.

# Phase 14 — final compatibility audit

Before merge, answer all of these from tests/code:

1. Does Google level-0 safe search omit `safe`?
2. Does Google split `&sa=U` before percent-decoding?
3. Does Google use Nokia UA plus `chrome99_android`?
4. Does Bing V0 omit `setlang`, `cc`, and `mkt` for all-locale?
5. Can a malformed recognized Bing wrapper fail only Bing, not the whole search?
6. Is DDG UA stable across requests?
7. Is DDG first-page query whitespace collapsed exactly as documented?
8. Are all three providers started before coordinator waiting begins?
9. Is there exactly one shared deadline?
10. Can late workers mutate no accepted results?
11. Does normalization precede identity?
12. Are same-provider duplicates retained as extra positions?
13. Is the exact score formula used?
14. Is grouping run after score sort?
15. Is final `limit` applied after grouping?
16. Does cancellation remain an execution cancellation rather than a search error?
17. Is `web.search` authorized only through the exact `tools/web_search` Tool identity?
18. Are `curl_cffi` and `lxml` present in every runtime packaging surface?
19. Are raw HTML, cookies, vqd tokens, and full provider error bodies absent from Tool output/log metadata?
20. Can all deterministic tests run without internet access?

If any answer is unknown, implementation is not complete.

# Files the coding agent should expect to create

A compact implementation can reasonably fit in:

    overmind/tools/web_search/__init__.py
    overmind/tools/web_search/tool.py
    overmind/tools/web_search/models.py
    overmind/tools/web_search/coordinator.py
    overmind/tools/web_search/transport.py
    overmind/tools/web_search/normalize.py
    overmind/tools/web_search/aggregate.py
    overmind/tools/web_search/errors.py
    overmind/tools/web_search/providers/__init__.py
    overmind/tools/web_search/providers/google.py
    overmind/tools/web_search/providers/bing.py
    overmind/tools/web_search/providers/duckduckgo.py

plus tests and dependency/registration edits.

Do not create server, daemon, Docker, database, browser automation, provider plugin framework, or generic web-fetch subsystem for this V0.

# Dry-run verdict

A coding agent can now implement the simplified V0 using the documentation set plus Overmind itself, without consulting the analyzed search repository.

Remaining external uncertainties are operational, not specification gaps: whether current provider endpoints still accept the documented fingerprints from the deployment network and whether QUIC/HTTP3 is available on the selected packaged runtime.
