# Test strategy

## Goal

The test suite must prove that the embedded implementation reproduces the audited search behavior before any Overmind-specific hardening or projection is applied. Provider protocols are brittle, so exact request/parse fixtures matter more than broad mocked “returns some results” tests.

Use five layers:

1. pure compatibility unit tests;
2. sanitized provider-response fixtures;
3. fake transport/request-shape tests;
4. concurrency/deadline/merge integration tests;
5. optional low-rate live smoke tests.

## Common normalization golden tests

Assert exact behavior for:

- spaces/tabs/newlines collapse to one space;
- title limit 200 chars with final-word truncation plus `" …"`;
- content limit 1200 chars with the same rule;
- content identical to title becomes empty;
- missing URL scheme receives `http` through common parsed-URL normalization;
- IDNA netloc beginning `xn--` is decoded to Unicode;
- query order, fragments, ports, `www`, and trailing slashes are retained;
- provider engine enters the provenance set;
- positions count only accepted main results, beginning at 1.

## Google golden tests

### Request

Assert:

- endpoint `/wml/search`;
- `q` and `sca_esv=1`;
- `hl/lr/cr/ie/oe` trait output;
- page 1 omits `start`; page 2 sends `start=10`;
- `day/week/month/year -> qdr:d/w/m/y`;
- safe 0 omits `safe` entirely;
- safe 1 -> `medium`, safe 2 -> `high`;
- `num` is absent;
- User-Agent belongs to the exact Nokia set;
- browser profile is `chrome99_android`;
- online request redirects remain disabled;
- generic `Accept-Language` shape is preserved;
- do not assert helper-only `Accept: */*` or `CONSENT=YES+` as values actually applied by this web request.

### CAPTCHA/sorry

Test:

- host `sorry.google.com`;
- path `/sorry...`;
- HTTP 302;
- body under 2000 chars containing `/sorry/`;
- ordinary response not falsely classified.

### Parser

Fixtures for:

- XML declaration removal;
- current result block/title/snippet/thumbnail selectors;
- missing title -> item skip;
- missing href -> item skip;
- malformed block exception -> item skip, later block survives;
- suggestions side channel;
- `/url?q=` direct decode.

### URL unwrap

Golden test must prove order:

    strip prefix -> split literal &sa=U -> percent-decode

Do not add a provider-level absolute-URL assertion in the parity test.

## Bing golden tests

### Request

Assert:

- endpoint `/search`;
- `q`;
- `adlt=off/moderate/strict`;
- `setlang` from trait region;
- `cc` present for ordinary regions;
- `cc` absent for `us`, `cn`, `ru`;
- no `mkt` in ordinary web request;
- page > 1 rejected by capability layer;
- time filter rejected by capability layer;
- provider network enables HTTP/3 when the transport conditions allow it.

### Parser

Fixtures for:

- `ol#b_results > li.b_algo`;
- missing link -> skip;
- empty href/title -> skip;
- paragraph extraction;
- exact removal of `span.algoSlug_icon`;
- direct href unchanged.

### Wrapper decoding

Test:

- exact `https://www.bing.com/ck/a?` prefix;
- missing `u` leaves wrapper href;
- `u` without `a1` leaves wrapper href;
- `a1` stripping;
- URL-safe base64 padding;
- UTF-8 decode with replacement;
- no absolute-URL validation afterward;
- malformed base64 exception escapes the parser and is classified at provider level, not silently skipped.

### Empty page

A successful HTTP page with zero matching `b_algo` blocks must produce an empty result list, not an invented ParseFailure.

## Primary DuckDuckGo HTML golden tests

### Request preprocessing

Assert:

- 499-char query accepted;
- 500-char query produces no request;
- recognized external bangs become quoted;
- query whitespace is normalized by bang preprocessing;
- process/provider User-Agent is stable across searches/pages.

### First page

Assert form/header state:

- method POST;
- HTML endpoint with trailing slash;
- `q` and `b=""`;
- no continuation fields;
- exact Sec-Fetch navigation headers;
- Referer;
- form content type;
- all-region `kl=wt-wt` with no `kl` cookie;
- specific region `kl` in both form and cookie;
- time filter `df` in both form and cookie;
- no invented safe-search field.

### Continuation

Assert:

- token cache key depends on transformed query + exact User-Agent;
- token TTL 3600 seconds;
- missing token raises the CAPTCHA/access-denied type with zero suspension;
- locale starting `zh` produces no page>1 request;
- `nextParams`, `api=d.js`, `o=json`, `v=l`, `vqd`;
- offsets page2=10, page3=25, page4=40;
- `dc=s+1`.

### Parser

Assert:

- 303 -> empty result list;
- `form#challenge-form` -> CAPTCHA/access-denied with zero suspension;
- hidden `vqd` extraction/cache;
- exact `div#links > div.web-result` selection;
- ads not selected;
- title/href/snippet extraction;
- structurally missing indexed href can escalate to provider-level failure;
- zero-click answer emitted only when not matching diagnostic phrases.

## Secondary DuckDuckGo JSON/script golden tests

This adapter is optional for V1 but its documentation is complete enough to implement later. Test:

- first-page discovery GET;
- Firefox impersonation;
- `default_headers=false`;
- `deep_preload_link` extraction;
- first-page URL cache TTL 7200 seconds;
- insertion of `o=json` into `d.js` URL;
- exact script/no-cors/same-site headers;
- JSON fields `u/t/a`;
- item without `u` skipped;
- continuation `n` builds next page URL;
- next-page cache TTL 3600 seconds;
- direct page skipping fails without a cached next-page URL;
- narrow arithmetic challenge fixture reproduces the audited calculation/follow-up without a general JS evaluator.

## URL identity/dedup golden tests

Assert parity identity fields:

    template + netloc + path + params + query + fragment + img_src

and verify:

- scheme difference alone merges;
- `www` difference does not merge;
- different query order does not merge;
- different fragment does not merge;
- trailing slash difference does not merge;
- `img_src` difference does not merge;
- thumbnail difference alone does not necessarily split identity;
- same-provider duplicate uses the same global merge path and appends a second position;
- no tracking-parameter removal;
- no provider-local pre-dedup.

If implementation replaces Python integer-hash keys with structured keys for collision safety, run all ordinary equivalence cases against both models and mark the key representation as a deliberate internal safety deviation.

## Merge/ranking golden tests

Test exact field merge:

- longer title wins;
- longer content wins;
- empty/default fields are filled according to model semantics;
- provider provenance unions;
- secure-suffixed scheme can replace insecure scheme;
- every duplicate appends position.

Test exact score:

    product(engine weights)
    * len(positions)
    * sum(1/position)

for ordinary priority.

Include:

- one engine;
- two engines;
- three engines;
- non-1.0 weights;
- duplicate occurrences from the same engine;
- low priority;
- high priority.

Golden R1/R2 example with equal weights must remain:

    R1 = 5.5
    R2 = 3.0

## Final grouping golden tests

After score sorting, test the second pass using:

    group key = category + template + image marker
    max_count = 8
    max_distance = 20

Verify that:

- grouping can change score order;
- image-bearing and non-image groups differ;
- primary/origin `engine` determines category lookup;
- equal-score first-pass order remains stable/insertion-based;
- no synthetic tie-break is introduced in parity mode.

## Concurrency/deadline tests

Use controllable fake providers to verify:

- all eligible providers start before joins/waits;
- shared deadline is not multiplied by provider count;
- one provider failure preserves siblings;
- one provider timeout preserves fast successes;
- late result is rejected;
- search start is common to all network budgets;
- extra provider network calls see less remaining time;
- suspended provider is skipped pre-dispatch;
- success resets suspended state;
- changing completion order can change only behaviors that are insertion-order-dependent (for example exact-score ties), not score computation itself.

## Failure/suspension tests

Assert active configured behavior:

- generic immediate failure suspension: 5 s;
- access denied: 180 s;
- CAPTCHA: 3600 s;
- too many requests: 180 s;
- Cloudflare CAPTCHA: 1,296,000 s;
- Cloudflare firewall: 86,400 s;
- reCAPTCHA: 604,800 s;
- primary DuckDuckGo missing-vqd/challenge: explicit 0 s.

Also test generic HTTP mappings for 402/403/429 and recognized challenge signatures.

## Transport fingerprint tests

Test:

- default Chrome impersonation;
- Google `chrome99_android` + Nokia UA;
- Bing conditional HTTP/3;
- primary DDG stable explicit UA plus default browser transport;
- secondary DDG Firefox + no default headers;
- redirects disabled through online-provider params;
- `discard_cookies=true` behavior represented by explicit provider cookies;
- disconnected pooled connection special retry;
- configured generic retries=0;
- shrinking timeout from one search start.

## Optional live smoke tests

Run low-rate, manually enabled tests from the intended deployment egress. For each provider:

- one benign short query;
- record status, body size, content type, elapsed time;
- verify expected request/profile and structural selector;
- verify at least one result when the query should obviously return results;
- stop/back off on block/challenge;
- never make live availability a mandatory unit-CI gate.

Live tests detect protocol drift; sanitized fixtures define implementation behavior.

## Clean separation of host hardening

Maintain two test groups:

    parity/*
    host_policy/*

Examples of host policy: body caps, HTTP(S)-only agent output, richer labelled provenance, cancellation improvements. A host-policy test must not replace or silently weaken the parity golden tests.