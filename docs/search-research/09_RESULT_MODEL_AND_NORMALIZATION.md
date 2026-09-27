# Result model and normalization

## Compatibility target

Provider parsers emit either typed main-result objects or legacy dictionary-shaped results. The common aggregation layer normalizes them before duplicate detection and ranking. For compatibility, normalization must occur **before hashing/merge** and must preserve the exact fields used by result identity.

## Main-result fields relevant to web search

A normalized main result carries at least:

    url
    parsed_url
    title
    content
    thumbnail
    img_src
    engine
    engines
    positions
    priority
    score
    template
    category

Optional metadata such as publication date, author, or other fields can exist but is not required from the three audited general-web adapters.

The public tool may later project this into a smaller `SearchResult`, but merge/ranking compatibility should operate on equivalent internal fields first.

## Engine provenance

Before normal main-result insertion:

1. the result's `engine` is filled from the provider when absent;
2. common field normalization runs;
3. normalization adds the non-empty engine name to the result's `engines` set.

When duplicates merge, the `engines` set is unioned. The single `engine` field remains a primary/origin engine value and is not the same thing as the complete provenance set.

## Exact text normalization

Runs matching spaces, tabs, or newlines are collapsed to one ordinary space and surrounding whitespace is stripped.

Limits are:

    title   = 200 characters
    content = 1200 characters

When a field exceeds its limit:

1. take the first `max_length` characters;
2. split at the final space in that truncated slice;
3. keep the text before that space;
4. append `" …"`.

This is character-based Python string behavior, not byte truncation.

After normalization, if `content == title`, content becomes the empty string.

**PARITY MUST:** apply these transformations before hashing/merge because longer-title/content merge decisions operate on normalized values.

## Exact URL normalization

When `url` exists but `parsed_url` does not:

    parsed_url = urllib.parse.urlparse(url)

If `url` is not a string, the result's URL is cleared and parsed URL becomes absent.

When a parsed URL exists:

1. if the complete network-location string begins with `xn--`, decode that netloc through IDNA into Unicode;
2. if the URL has no scheme, set scheme to `http`;
3. preserve the parsed path;
4. regenerate the string URL from the parsed result.

Important consequences:

- the common normalizer does not require an HTTP/HTTPS scheme;
- a schemeless URL can become `http:<...>` according to `urlparse` structure rather than being rejected;
- hostname case is not explicitly canonicalized here;
- default ports are not removed;
- query order is not changed;
- fragments are retained;
- tracking parameters are retained;
- `www.` is retained;
- trailing/repeated slashes are retained;
- IDNA decoding occurs only when the entire parsed netloc starts with `xn--` in this implementation.

A stricter host security layer can validate result URLs before exposing/fetching them, but that is a deliberate behavioral boundary after compatibility aggregation, not the observed normalizer.

## Publication dates

When `publishedDate` exists, normalization attempts to generate a formatted string using:

    %Y-%m-%d %H:%M:%S%z

If Python raises `ValueError` for the date, `publishedDate` is cleared.

The three main web adapters covered by V1 generally do not populate a publication date for ordinary results.

## Provider position assignment

Positions are **not** raw DOM indexes and are not assigned inside all provider parsers.

The result container keeps a `main_count` separately for each provider call. For every accepted main result, it increments this count and passes that count as the result position into merge.

Side channels such as suggestions, answers, corrections, infoboxes, and engine data do not consume a normal main-result position.

A malformed/skipped result that never reaches main-result insertion likewise does not consume a position.

Therefore provider position is:

    ordinal among accepted main results from that provider

starting at 1.

## Typed and legacy provider outputs

Google and primary DuckDuckGo emit typed main-result objects. Bing returns legacy dictionaries containing `url`, `title`, and `content`; the aggregation layer wraps these into a legacy-result compatibility type and then applies the same common normalization.

The new tool does not need to reproduce two Python class hierarchies, but its tests must prove equivalent normalized values before dedupe/ranking.

## Side-channel results

The common container distinguishes normal results from:

- suggestions;
- answers;
- corrections;
- infoboxes;
- provider engine data.

These do not flow through the ordinary main-result hash/ranking path.

For the initial tool response, returning only ranked main results is acceptable. That is an output-surface simplification, not a reason to change how provider parsers identify their normal results.

## Public tool projection

After compatibility merge/ranking, a compact external result can be projected as:

    SearchResult:
      title
      url
      snippet        # normalized internal content
      score
      providers
      positions
      thumbnail?

The projection must not re-run a second independent dedupe/ranking algorithm.

If the host wants labelled `provider -> position` provenance, capture that information during insertion in an auxiliary structure without changing the compatibility `positions` list used by scoring.

## Golden tests

Test exact behavior for:

- whitespace collapse of spaces/tabs/newlines;
- 200/1200 character limits and word-boundary ellipsis;
- content equal to title becomes empty;
- non-string URL becomes empty;
- missing scheme receives `http` through parsed-url replacement;
- IDNA netloc beginning with `xn--` becomes Unicode;
- query, fragment, port, `www`, trailing slash, and query order remain untouched;
- engine is added to provenance during normalization;
- first accepted main result gets position 1 regardless of skipped malformed/side-channel entries;
- Bing legacy output normalizes to the same internal field semantics as typed provider output.

## Deliberate host deviations

Absolute-URL enforcement, HTTP(S)-only output, ASCII-IDNA canonicalization, tracking removal, or richer provenance are reasonable Overmind boundaries, but they must be applied after/paralleling the compatibility model and covered by separate tests. They are not part of the observed normalization behavior.