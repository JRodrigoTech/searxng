# Open questions and validation plan

## Purpose

These questions are intentionally not hidden behind confident prose. The public web is mutable, and several protocol details are brittle. Each question has a suggested validation method and a safe fallback.

## Highest priority live checks

### 1. Are all three endpoints reachable from deployment?

Question:

- Can the deployment network reach the Google mobile endpoint, Bing HTML endpoint, and DuckDuckGo HTML endpoint without a login or challenge?

Validation:

- run one low-rate smoke query per provider from the actual deployment egress;
- record status, body size, content type, and whether the expected result container exists.

Fallback:

- disable the failing provider through configuration;
- preserve other provider results;
- do not add challenge-solving code.

### 2. Does the current Google representation still emit the observed structure?

Question:

- Are the result-block, title, snippet, and thumbnail class structures still present?

Validation:

- capture a small sanitized response fragment;
- run the parser fixture;
- compare result count and wrapper links.

Fallback:

- return ParseFailure and suspend only after a configurable regression threshold;
- investigate a new representation separately.

### 3. Does Google accept the mobile User-Agent/profile pairing?

Question:

- Does the fixed Nokia User-Agent with Android Chrome transport profile continue to produce normal results rather than a consent or challenge page?

Validation:

- compare a single fixed profile against a controlled smoke query;
- do not rotate rapidly.

Fallback:

- select a different coherent profile only as an explicitly tested configuration.

### 4. Is Bing wrapper decoding still base64url with the a1 prefix?

Question:

- Do current result links still use ck/a and u values beginning with a1?

Validation:

- inspect a sanitized result href;
- test padding and UTF-8 handling.

Fallback:

- skip malformed wrapper results and retain direct absolute links;
- do not follow wrappers over the network.

### 5. Does DuckDuckGo still require vqd for later pages?

Question:

- Is the token query/User-Agent relationship and one-hour TTL still valid?

Validation:

- first-page fixture with hidden vqd;
- same-query continuation from the same stable User-Agent;
- separate-query and changed User-Agent negative tests;
- expiry test.

Fallback:

- support first-page search only and report later pages as unavailable.

## DuckDuckGo questions

### Safe-search field

The active HTML form did not show a verified safe-search parameter or cookie despite advertising safe-search capability. Determine whether the current endpoint accepts a stable field. Until then, strict safe-search must either exclude this provider or be labelled best effort.

### Cookie names and values

The observed path uses kl for region and df for time. Verify that the current endpoint still reads them and that a form field is not also required.

### Challenge source

Determine whether challenge behavior is primarily IP-based, User-Agent-bound, token-bound, or a combination. The implementation does not need to identify the exact cause to degrade safely.

### Chinese pagination

Verify whether the zh page-two restriction remains necessary. Keep it as a capability override until repeated live tests show safe support.

### JSON-style alternative

Decide whether the separate d.js JSON-style path adds enough reliability to justify a second adapter. The arithmetic challenge path is not a V1 requirement and must not be executed.

## Google questions

- Does omitting num remain correct?
- Is sca_esv=1 still accepted?
- Does cr=country... still provide the desired restriction?
- Should the all-locale request omit lr/cr or send empty values?
- Are status 302 and sorry URL checks sufficient for current blocks?
- Which result dates, if any, are stable enough to expose?
- Is a trait refresh from preferences worth its privacy and latency cost?

## Bing questions

- Is the current active request really more reliable without mkt?
- Do US, CN, and RU still require cc omission?
- Can page and time filters be implemented through a stable public HTML request?
- Which body markers distinguish a normal zero-result response from a challenge?
- Does Bing require cookies or a referer from the deployment network?
- Is optional HTTP/3 beneficial or merely a fingerprint difference?

## Locale questions

- Which caller locale format is guaranteed by the host runtime?
- Should an unsupported region fall back to language-only or to all-locale?
- Does the product require strict geographic targeting or only an approximate market?
- Should the locale trait tables be static, refreshed asynchronously, or bundled per release?
- How should script tags such as zh-Hant be mapped when no exact provider trait exists?

## Ranking and product decisions

- Are all providers equally trusted, or should weights be configurable?
- Should HTTP and HTTPS share identity for this product?
- Should fragments be preserved or dropped for page identity?
- Is a snippet merge based on longer text acceptable, or should provider preference win?
- Should provider diagnostics be returned to the AI agent or only logged?
- What final result limit is appropriate for token budget and tool latency?
- Should a zero-result healthy provider be distinct from an empty blocked response?

## Runtime and dependency decisions

- Does the host already have an async HTTP client and HTML parser?
- Can the runtime use curl-based browser impersonation wheels on every deployment platform?
- Does the host permit an event-loop-owned client, or must the library expose a client lifecycle?
- Are HTTP/2 and HTTP/3 materially helpful for the target egress?
- Can the host provide structured metrics without adding a new metrics dependency?

## Security decisions

- Will a later tool fetch result pages, or must search remain strictly URL discovery?
- If fetching is added, where will SSRF validation and network isolation live?
- Are result snippets stored, cached, or transmitted to another service?
- What retention policy applies to provider diagnostics?
- Is disabling TLS verification ever allowed in production?

## Evidence needed before declaring protocol stability

For each provider, retain a small internal validation record containing:

- date and deployment egress region;
- request parameter names and values;
- redacted header profile;
- status and content type;
- result-container observation;
- redirect-wrapper example with synthetic or sanitized destination;
- parser result count;
- block/challenge outcome if applicable.

Do not store cookies, validation tokens, full query URLs, or full response bodies in the record.

## Decision rule

When a live result disagrees with these documents:

1. Preserve the last known-good fixture.
2. Record the new response shape in a sanitized fixture.
3. Mark the old statement as obsolete or provider-version-specific.
4. Update the adapter contract and regression tests.
5. Keep the coordinator and result model unchanged unless the semantic contract actually changed.

The subsystem is ready for implementation when all ESSENTIAL_V1 items are either verified live or explicitly guarded behind a typed capability/failure outcome, and no open question would cause silent data corruption or unsafe URL handling.
