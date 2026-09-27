# Security boundaries

## Threat model

Search responses come from external services and from arbitrary third-party web publishers. Titles, snippets, URLs, thumbnail URLs, dates, and metadata are untrusted data. The search subsystem is an input source for an AI agent and must not turn provider text into instructions.

## Search data is not authority

The caller and downstream agent must treat all of the following as untrusted:

- result titles;
- snippets;
- provider suggestions;
- URL path/query text;
- page names and authors;
- result metadata;
- challenge-page text.

The response schema should make this boundary obvious. A result is data, not a command, policy, or system message. The tool description should tell the agent not to follow instructions embedded in result text.

## HTML and script safety

- Parse HTML with a non-executing parser.
- Never evaluate provider JavaScript.
- Never interpret script nodes as result content.
- Convert selected text nodes to plain text.
- Escape result text at the presentation layer.
- Do not render arbitrary provider HTML in a privileged UI.

The inactive DuckDuckGo JSON-style path contains an arithmetic challenge in response content. The new subsystem must not execute it. Treat it as anti-automation evidence and return a typed failure.

## URL safety

The search component does not fetch result destinations. This is an important security boundary:

    search result discovery  !=  arbitrary page fetching

If a later feature fetches a result URL, it needs a separate SSRF policy:

- validate scheme;
- reject userinfo;
- resolve DNS and check every address;
- block localhost, loopback, link-local, multicast, private, and reserved ranges unless explicitly allowed;
- re-check after redirects;
- bound redirect count, body size, and response time;
- isolate credentials and proxy settings;
- use a separate capability and audit trail.

Result URL parsing alone is not sufficient SSRF protection because DNS rebinding and redirect chains can change the destination.

## Redirect safety

Provider wrapper decoding is local string processing. It must not perform a network request.

Rules:

- accept only known wrapper formats;
- decode one bounded layer;
- require an absolute HTTP(S) destination;
- reject control characters and credentials;
- preserve the decoded destination only after validation;
- never treat a provider redirect host as the final identity.

## Response-size and decompression safety

Enforce limits before and after decompression. Protect against:

- very large HTML responses;
- compressed data that expands unexpectedly;
- deeply nested or malformed markup;
- huge numbers of DOM nodes;
- large attribute values;
- invalid byte sequences.

Use a parser configuration that does not fetch external entities or resources. XML-like Google output must be parsed without network-enabled entity expansion.

## Encoding safety

- Prefer the response charset when trustworthy.
- Use a safe replacement policy for invalid bytes.
- Reject undecodable control-heavy responses.
- Normalize text for display only after decoding.
- Avoid logging undecoded bytes.

Do not treat an encoding failure as permission to reinterpret arbitrary bytes as executable content.

## Cookies and validation tokens

Cookies and DuckDuckGo validation tokens are transport state, not search results:

- store them only in provider-scoped state;
- apply TTL and query/User-Agent binding;
- redact them from logs and telemetry;
- do not return them to the caller;
- clear them on suspected block or token mismatch;
- do not share them across unrelated providers.

## Logging and telemetry injection

Provider titles, snippets, hosts, and error text can contain newlines, control characters, or misleading structured-log syntax. Before logging:

- use structured fields rather than string interpolation;
- normalize or escape control characters;
- truncate values;
- prefer host and status over full URLs;
- use a low-cardinality failure code.

Never include arbitrary provider text in a metric name or label.

## Network and TLS

- TLS verification is enabled by default.
- A custom CA bundle is explicit deployment configuration.
- Disabling verification must be visible in configuration and telemetry.
- Proxy configuration is a trust boundary; do not log proxy credentials.
- Browser impersonation does not replace TLS verification.

## Prompt-injection resistance

The downstream agent may see snippets. The tool response should:

- label them as quoted external content;
- keep provider diagnostics separate from result text;
- never concatenate snippets into an instruction template;
- avoid hidden metadata fields that can be interpreted as tool commands;
- preserve source/provider labels for citation and skepticism.

The search tool should not automatically summarize or obey text found in result pages.

## Resource isolation

Bound:

- number of providers;
- number of result blocks parsed;
- text and metadata lengths;
- total response bytes;
- redirects;
- retries;
- total time.

Use separate cancellation and cleanup paths so an uncooperative provider cannot hold the AI runtime indefinitely.

## Security review checklist

- [ ] No provider JavaScript is executed.
- [ ] No result URL is fetched by search.
- [ ] Known wrappers are decoded locally and bounded.
- [ ] HTTP(S) scheme and host are validated.
- [ ] Cookies and tokens are redacted.
- [ ] HTML/XML parsers cannot fetch external resources.
- [ ] Response and decompression limits are enforced.
- [ ] Logs escape provider-controlled text.
- [ ] Provider text is labelled untrusted in the tool contract.
- [ ] TLS verification remains on by default.
