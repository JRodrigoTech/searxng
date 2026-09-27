# Security boundaries

## Purpose

The search tool receives untrusted internet data and feeds an AI agent. Security controls must therefore be explicit. At the same time, this documentation targets behavioral compatibility with the audited search pipeline. Security hardening that changes which results survive must be clearly separated from provider-parity behavior.

## Trust boundary

All provider-derived values are untrusted data:

- titles;
- snippets/content;
- URLs;
- thumbnails;
- suggestions;
- answer text;
- dates and metadata;
- challenge/error page content.

They must never become system/developer instructions or acquire tool authority. The tool returns data; downstream context compilation must preserve that distinction.

## Search versus fetch

This component discovers search results. It does not fetch arbitrary result destinations:

    web search != page fetch

A later page-fetch tool requires its own SSRF controls, redirect policy, DNS/IP checks, body limits, credential isolation, and authorization boundary. Search-result URLs alone are not trusted destinations.

## Provider parsing

The primary three adapters do not require a JavaScript runtime:

- Google: tolerant HTML parsing of the mobile representation;
- Bing: HTML parsing;
- primary DuckDuckGo: HTML parsing.

The secondary DuckDuckGo JSON/script adapter contains a narrow arithmetic challenge reconstruction. It parses a small provider-generated grammar and computes a number; it does not run arbitrary provider JavaScript in a browser/runtime. For exact secondary-adapter parity that narrow behavior is documented separately. Do not generalize it into `eval`, a JavaScript VM, or arbitrary script execution.

## URL behavior: parity versus host validation

Provider-parity behavior is intentionally permissive in places:

- Google unwraps its known redirect and returns the decoded string without an absolute-HTTP(S) check;
- Bing decodes its known wrapper and likewise does not perform an absolute-HTTP(S) check;
- common result normalization can assign `http` when a parsed result lacks a scheme;
- identity preserves most URL spelling details and ignores scheme for ordinary-result hashing.

Therefore a rule such as “provider parser must reject every non-absolute/non-HTTP URL” would change compatibility.

Recommended boundary:

    provider parse
      -> source-compatible normalization/merge/rank
      -> host output validator
      -> agent-facing SearchResult

The host validator may reject or quarantine unsafe schemes/userinfo/control characters before exposing results to later capabilities. Tests must distinguish that host policy from provider parity.

## Provider wrapper decoding

Known wrappers are decoded locally; decoding must not fetch the wrapped destination.

Compatibility decoders are bounded by their known formats:

- Google `/url?q=` transformation;
- Bing `ck/a?u=a1...` base64url transformation;
- primary DuckDuckGo uses direct result hrefs.

If the host applies a stricter post-decode URL policy, it belongs after the parity decoder.

## HTML/XML parser safety

Use non-executing parsing. The parser must not intentionally retrieve external entities/resources. Provider markup must be treated as bytes/text input only.

The exact provider selectors should be run only against bounded responses from their fixed provider origins. Do not expose a general XPath/selector facility to the agent.

## Response-size hardening

The audited source path does not provide the bespoke compressed/decompressed limits proposed in the first research draft. Overmind should still impose reasonable transport/body limits because the search capability processes external data.

Such limits are a host security boundary and must be sized so normal compatibility fixtures succeed. A body exceeding a host cap should produce a bounded provider failure, not partial parsing of unbounded input.

## Cookies and token state

Provider state includes region/time cookies, validation tokens, trait caches, and request identity.

Rules:

- scope state by provider;
- never expose cookies or `vqd` in the agent result;
- preserve query/User-Agent binding where required;
- preserve TTL semantics;
- avoid logging token values;
- do not let one provider mutate another provider's state;
- if the host uses in-memory state instead of persistent cache, document that as a persistence deviation.

## Logging

Provider-controlled values can contain control characters or misleading syntax. Prefer structured logs with bounded/redacted values.

Safe operational fields include:

    provider
    phase
    status_code
    elapsed_ms
    result_count
    timeout
    failure_class

Avoid normal logs containing:

- raw query URLs;
- cookies;
- validation tokens;
- complete HTML/JSON bodies;
- proxy credentials;
- arbitrary snippets as metric labels.

## TLS and transport identity

TLS verification remains enabled by default. Browser impersonation changes request fingerprint but does not replace certificate verification.

The compatibility transport uses provider-specific browser/TLS profiles. Security hardening must not silently replace them with unrelated fingerprints and then attribute resulting blocks to provider instability.

## Prompt-injection boundary

The final tool response should structurally identify provider-derived fields as external content. Do not concatenate snippets into trusted instruction strings.

If the agent later fetches a result page, that fetched page remains untrusted even when several search providers agreed on its URL.

## SSRF requirements for a future fetch capability

A separate page-fetch capability should, at minimum:

- permit only expected schemes;
- reject userinfo;
- resolve DNS and validate every resolved address;
- block loopback, link-local, private, multicast, and reserved ranges unless explicitly authorized;
- re-evaluate redirects;
- bound redirect count/time/body;
- isolate credentials and proxy configuration;
- defend against DNS rebinding where practical;
- maintain its own authorization and audit trail.

None of those checks should be implemented by secretly causing the search parser to fetch result URLs.

## Resource bounds

Host-level bounds should cover:

- enabled provider count;
- search deadline;
- response bytes;
- parsed result count;
- title/content lengths;
- state/cache growth;
- retries;
- final tool result count.

Compatibility normalization already limits title/content to 200/1200 characters. Additional result-count/body limits are host controls.

## Security/parity test matrix

Tests should prove both layers independently:

| Case | Compatibility expectation | Host expectation |
| --- | --- | --- |
| Google decoded unusual URL | decoder reproduces source output | output policy may quarantine it |
| Bing decoded non-HTTP string | parser reproduces decoded string if decoding succeeds | output policy may reject it |
| Provider snippet contains instructions | preserved as ordinary text | never promoted to trusted instruction |
| DDG `vqd` | used internally with exact binding/TTL | never returned/logged |
| Secondary DDG arithmetic challenge | narrow parser can reproduce it when that adapter is enabled | no arbitrary JS execution |
| Result destination | never fetched by search | separate fetch authorization required |

## Checklist

- [ ] Provider text remains untrusted.
- [ ] Search does not fetch arbitrary result URLs.
- [ ] Compatibility wrapper decoding is local only.
- [ ] Host URL validation is separated from provider-parity parsing.
- [ ] No arbitrary JavaScript execution exists.
- [ ] Cookies/tokens are provider-scoped and redacted.
- [ ] TLS verification remains enabled.
- [ ] Request fingerprint parity is tested.
- [ ] Response/resource limits are host-level and bounded.
- [ ] A future fetch tool receives independent SSRF controls.