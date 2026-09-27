# Result model and normalization

## Why several models are needed

Provider documents contain provider-specific fields and sometimes wrapper URLs. A single mutable object cannot safely represent all stages. Use four conceptual stages:

    RawProviderItem
        |
        v
    ProviderResult
        |
        v
    NormalizedResult
        |
        v
    MergedResult
        |
        v
    SearchResponse item

The names are design terminology, not a requirement to copy any existing class hierarchy.

## Raw provider item

This is the parser’s short-lived representation. It may contain:

- raw title nodes or text;
- raw snippet text;
- raw href;
- provider wrapper URL;
- raw thumbnail;
- provider-specific date/author fields;
- parser warnings.

Raw items must not leave the provider adapter. They may contain HTML-derived text and are untrusted.

## ProviderResult

ProviderResult is the first validated representation:

    ProviderResult
      provider: fixed provider identifier
      title: non-empty text
      destination_url: absolute URL
      snippet: optional text
      provider_position: positive integer
      published_at: optional instant/date
      thumbnail: optional absolute URL
      metadata: bounded provider-neutral map

The destination URL must already have known provider wrappers removed. If an item cannot produce an absolute URL, it is not a ProviderResult.

## NormalizedResult

The normalizer applies common invariants:

    NormalizedResult
      title
      url
      display_url
      snippet
      provider
      provider_position
      published_at
      thumbnail
      metadata
      identity

For V1, url and display_url may be the same value. Keeping both conceptual fields prevents a future identity rewrite from changing what the caller sees.

### Required invariants

- title is non-empty after whitespace normalization;
- url is absolute and has an allowed HTTP-family scheme;
- provider is one of the enabled provider identifiers;
- provider_position starts at one;
- snippet is optional and bounded;
- thumbnail is optional and must not be fetched by the search subsystem;
- identity is calculated only after redirect unwrapping and URL validation;
- metadata is bounded in size and value length.

## MergedResult

MergedResult contains one display record plus provenance:

    MergedResult
      title
      url
      snippet
      identity
      providers: set of provider identifiers
      observations:
        - provider
          provider_position
          observed_url
          title
          snippet
      score
      published_at
      thumbnail
      metadata

The observations list is important. A provider set alone cannot explain where a result appeared, and an unlabelled list of positions cannot tell which provider produced a position.

## SearchResponse

    SearchResponse
      query
      results: bounded list[MergedResult]
      providers: bounded provider diagnostics
      degraded: boolean
      elapsed_ms

Do not return raw HTML, cookies, validation tokens, transport exceptions, or unbounded metadata.

## Text normalization

**Observation:** The analyzed result path collapses runs of whitespace and trims both ends. Titles are bounded at approximately 200 characters; snippets/content are bounded at approximately 1,200 characters, with word-boundary truncation and an ellipsis. When snippet text equals the title, the snippet is cleared.

**Recommendation:** Keep these limits configurable constants, apply them after HTML-to-text conversion, and test Unicode whitespace. Do not cut a UTF-8 byte sequence in the middle of a code point.

Suggested V1 rules:

1. Convert HTML nodes to text without interpreting text as markup after extraction.
2. Replace repeated whitespace with one ordinary space.
3. Strip leading/trailing whitespace.
4. Reject empty titles.
5. Truncate title and snippet by Unicode character count at word boundaries.
6. Preserve the original text only in bounded, opt-in debug metadata.

## URL normalization at result boundary

The normalizer should:

1. unwrap Google and Bing provider wrappers;
2. parse the URL;
3. require a scheme and network location;
4. lowercase the hostname;
5. normalize IDNA representation for identity;
6. preserve a safe display form;
7. remove no query parameters by default;
8. calculate identity using the separate URL policy.

The normalizer must not fetch the destination to discover canonical tags. Search and page fetching are separate capabilities.

## Dates and thumbnails

Published dates are optional. A provider date may be retained only when the parser can identify a real date field. Do not infer a publication date from arbitrary snippet prose in V1.

Thumbnails are metadata, not evidence that the result URL is safe. Preserve a thumbnail URL only if it is absolute and within a bounded field length. Never fetch or render it inside the search subsystem.

## Metadata policy

Useful metadata includes:

- provider-specific result type;
- a redacted source marker;
- a parser warning code;
- an observed result feature such as “has thumbnail”.

Do not preserve:

- raw DOM fragments;
- all provider attributes;
- cookies;
- query-bound validation tokens;
- full wrapper URLs containing tracking data;
- arbitrary script text.

## Provider-local duplicate handling

Before cross-provider merging, collapse repeated identical identities from the same provider. Keep the lowest provider position and, if necessary, merge the provider’s own fields. This prevents one provider from receiving an artificial consensus bonus for emitting the same destination twice.

## Main results versus side channels

The analyzed behavior also supports suggestions, answer boxes, infoboxes, engine data, and other result categories. Those are not required for V1 main web results. If added later, use a tagged union or separate arrays rather than allowing an answer object to masquerade as a normal web result.

## Normalization examples

Input:

    title: "  A   useful   page "
    snippet: "A useful page"
    href: "https://example.test/a"
    provider_position: 1

Normalized:

    title: "A useful page"
    snippet: ""
    url: "https://example.test/a"
    provider_position: 1

Input:

    title: "Example"
    href: "/url?q=https%3A%2F%2Fexample.test%2Fdocs&sa=U"

Normalized:

    title: "Example"
    url: "https://example.test/docs"
