# Test strategy

## Test layers

Use five layers:

1. Pure unit tests for mapping, identity, merge, and ranking.
2. Sanitized fixture tests for provider parsers.
3. Fake-transport tests for request construction and failure handling.
4. Deterministic concurrency tests with controlled tasks.
5. Optional live smoke tests that are never required for ordinary CI.

No test should depend on a real CAPTCHA, an external redirect, or an unbounded live response.

## Unit tests: common models

Cover:

- empty and whitespace-only query rejection;
- limit lower and upper bounds;
- one-based provider positions;
- whitespace collapse;
- title/snippet truncation at word boundaries;
- duplicate title/snippet suppression;
- absolute URL validation;
- metadata bounds;
- provider-local duplicate collapse;
- labelled provenance observations.

## Unit tests: Google

### Request construction

Assert decoded parameter maps for:

- first page;
- page two offset 10;
- each time range d, w, m, y;
- safe off, moderate, strict;
- locale en-US;
- locale es-ES;
- an all-locale request;
- a Chinese locale alias;
- omission of num;
- presence of sca_esv, ie, and oe;
- fixed Accept and consent cookie;
- stable request-level browser profile;
- redirect following disabled.

### Response parser

Fixtures should contain small sanitized blocks for:

- two normal results;
- missing title;
- missing href;
- a Google wrapper URL;
- a direct absolute URL;
- a snippet and an encrypted thumbnail;
- malformed one-result block followed by a valid block;
- XML declaration before HTML;
- an empty but structurally valid result page.

### Block detection

Fixture or synthetic response tests for:

- sorry.google.com final URL;
- /sorry path;
- HTTP 302 with challenge-like body;
- short body containing /sorry/;
- ordinary empty response that must not be misclassified without evidence.

## Unit tests: Bing

### Request construction

Assert:

- q and adlt values;
- setlang for en-US and es-ES;
- cc for es;
- cc omission for us, cn, and ru;
- no unsupported time/page parameters are advertised;
- optional market helper is not accidentally sent if the active contract does not use it.

### Redirect decoding

Test:

- valid a1 base64url with missing padding;
- valid payload containing query parameters;
- valid UTF-8;
- malformed base64;
- invalid UTF-8 replaced safely;
- wrapper without a1;
- non-wrapper absolute URL;
- decoded value that is not absolute.

### Response parser

Fixtures should include:

- normal b_results with multiple b_algo items;
- missing h2 link;
- missing href;
- decorative algoSlug_icon span;
- empty snippet;
- malformed wrapper followed by a valid result;
- challenge/login document with no result list.

## Unit tests: DuckDuckGo

### Request construction

Assert:

- query length 499 accepted;
- query length 500 rejected without network;
- first-page POST fields q and b;
- Content-Type;
- stable User-Agent reused across page one and page two;
- Sec-Fetch-* fields;
- Referer;
- kl region cookie;
- df time cookie;
- page two token fields;
- offsets for pages 2, 3, and 4;
- Chinese locale page-two suppression;
- no tokenless continuation request.

### Token state

Test:

- token extraction from hidden vqd input;
- token cache hit for the same query/User-Agent;
- cache miss for a different query;
- cache miss for a different User-Agent;
- TTL expiry;
- invalid token clearing after a challenge;
- concurrent access to the same state key;
- no token value in telemetry.

### Response parser

Fixtures should include:

- normal web-result blocks;
- ad-style block excluded;
- missing title;
- missing snippet;
- direct destination href;
- challenge-form;
- 303 response;
- vqd hidden input;
- optional zero-click abstract;
- diagnostic zero-click text excluded.

## Locale and filter tests

Use a table-driven suite:

| Input | Expected behavior |
| --- | --- |
| no locale | provider fallback |
| en-US | provider-specific English/US values |
| es-ES | provider-specific Spanish/Spain values |
| zh-CN | provider alias values |
| unsupported language | fallback or UnsupportedLocale according to policy |
| safe off | provider off mapping |
| safe strict | provider strict mapping or explicit unsupported outcome |
| time day/week/month/year | provider-specific values |
| Bing time filter | UnsupportedCapability in V1 |

## Canonical URL and deduplication tests

Test:

- Google wrapper versus direct destination;
- Bing wrapper versus direct destination;
- HTTP versus HTTPS under the selected policy;
- host case;
- default ports;
- non-default ports;
- www distinction;
- root and non-root trailing slashes;
- repeated slashes;
- encoded slash versus literal slash;
- query order;
- unknown query parameter preservation;
- allowlisted tracking parameter policy if later enabled;
- fragments preserved by default;
- userinfo rejection;
- IDNA host normalization;
- hash collision simulation with explicit identity comparison.

## Merge and ranking tests

Test:

- one provider result;
- same destination from two providers;
- same destination from all providers;
- different titles and snippets;
- HTTPS preference;
- labelled positions;
- conflicting dates;
- provider-local duplicate collapse;
- deterministic tie-break;
- observed score compatibility formula;
- recommended simplified formula;
- consensus cap;
- a provider cannot obtain multiple consensus bonuses from duplicate blocks.

The worked R1/R2 example in 11_MERGE_AND_RANKING.md should be a golden test with expected scores 5.5 and 3.0 for equal provider weights under the observed formula.

## Concurrency tests

Use fake providers with barriers and controllable delays:

- all providers succeed;
- one fails immediately;
- two fail;
- one times out;
- all time out;
- slow success before the deadline;
- completion after the deadline;
- cancellation requested at deadline;
- a task raises an unexpected exception;
- completion order changes but final order does not;
- late task cannot mutate a completed response.

## Fixture design

Fixtures must be:

- small;
- sanitized;
- stored by provider and protocol case;
- free of real cookies, tokens, personal data, and large tracking URLs;
- representative of the external document structure;
- accompanied by the failure or behavior the fixture proves.

Do not copy entire provider pages. A handful of result blocks and the surrounding container is sufficient.

## Regression tests for brittle selectors

Every provider parser should have:

- a fixture for the currently observed selector structure;
- a fixture with unrelated navigation/advertising elements;
- a missing-field fixture;
- a zero-result or block fixture;
- an assertion that a structural change produces ParseFailure or an explicit warning rather than silently returning plausible but wrong data.

Add a test that checks the parser’s result count is not zero solely because a fixture contains no expected selector.

## Transport tests

With a fake server or mocked transport, test:

- GET query encoding;
- POST form encoding;
- header and cookie isolation;
- compressed response decoding;
- body-size limit;
- invalid encoding;
- redirect budget;
- TLS error classification;
- retry count and deadline propagation;
- HTTP/2/HTTP/3 profile selection as a configuration decision;
- client reuse and close.

## Optional live smoke tests

Run periodically, outside mandatory unit CI, from a controlled environment:

- one short benign query per provider;
- verify status and at least one structurally valid result;
- verify wrapper decoding;
- verify locale and safe-search request construction;
- record elapsed time and parser warnings;
- never solve or repeatedly trigger challenges;
- stop or back off when a block/CAPTCHA is detected.

Live tests should be rate-limited, manually enabled in CI, and must not fail a code change solely because a public provider is temporarily unavailable.

## Property and fuzz testing

Useful properties:

- URL identity is deterministic;
- identity does not contain raw cookies or tokens;
- wrapper decoding never performs I/O;
- malformed base64 never crashes the coordinator;
- parser output is bounded;
- ranking is deterministic under provider completion permutation;
- a provider failure does not remove another provider’s results.

Fuzz HTML fragments, URL query strings, Unicode whitespace, invalid UTF-8, and truncated wrapper values.
