# URL identity and duplicate detection

## Compatibility target

Duplicate detection is intentionally simple and must be documented exactly because it directly affects consensus and ranking.

The audited implementation does **not** build a sophisticated canonical URL string. It normalizes result fields first, then uses the result object's Python hash as the dictionary key for duplicate merging.

## Main-result identity

For ordinary typed web results, the hash input is equivalent to concatenating:

    template
    parsed_url.netloc
    parsed_url.path
    parsed_url.params
    parsed_url.query
    parsed_url.fragment
    img_src

The URL **scheme is omitted** from identity.

Legacy ordinary web results use the same identity fields.

Therefore two ordinary results merge when these identity components produce the same Python hash, even if their URL schemes differ.

## What is and is not normalized before identity

Common result normalization occurs before hashing. Relevant behavior is:

- a missing scheme is replaced with `http` in the parsed URL;
- an IDNA netloc beginning with `xn--` is decoded to Unicode;
- title/content normalization does not participate in identity;
- scheme does not participate in the main-result hash;
- netloc does participate;
- path participates;
- URL params participate;
- query participates exactly as parsed;
- fragment participates;
- `img_src` participates;
- `thumbnail` does **not** participate;
- `template` participates.

There is no generic removal of:

- `www.`;
- default ports;
- tracking parameters;
- trailing slashes;
- repeated slashes;
- fragments;
- reordered query parameters.

There is no generic lowercase-host rewrite in the result identity layer.

## Consequences

### HTTP and HTTPS

These can merge when all other identity fields match because scheme is excluded.

    http://example.test/a
    https://example.test/a

can therefore be duplicates.

### Query ordering

These remain distinct:

    https://example.test/?a=1&b=2
    https://example.test/?b=2&a=1

because the query string is part of identity as parsed.

### Fragment

These remain distinct:

    https://example.test/docs#one
    https://example.test/docs#two

### `www`

These remain distinct:

    https://www.example.test/docs
    https://example.test/docs

### Trailing slash

These normally remain distinct:

    https://example.test/docs
    https://example.test/docs/

### Image source

Two otherwise equal results can remain distinct if their `img_src` values differ. A normal Google web thumbnail is stored in `thumbnail`, not necessarily `img_src`, so that thumbnail alone does not change ordinary identity.

## Provider wrapper handling before identity

Wrapper decoding occurs in provider parsing, before common normalization/hash:

- Google: only `/url?q=...` wrapper, split `&sa=U` before percent-decoding;
- Bing: only exact `https://www.bing.com/ck/a?` wrapper, `u=a1...` URL-safe-base64 decode;
- primary DuckDuckGo HTML: result href is used directly.

The common identity layer does not independently re-run those provider decoders.

## Dictionary-key semantics

The global main-result map is keyed directly by:

    hash(result)

an integer Python hash.

When a key is absent, the result is inserted and its positions list is initialized. When a key already exists, the new result is merged into the existing result and the new position is appended.

There is no second full identity-string equality comparison after an integer-hash match.

This creates a theoretical Python-hash collision risk. Replacing the map key with the full structured identity would be a robustness improvement, but it is **not exact parity**.

## No separate provider-local deduplication stage

There is no independent pre-pass that first collapses duplicates within each provider. Results enter the shared container in provider output order. The same global identity map handles both:

- duplicates from the same provider;
- duplicates across different providers.

If one provider emits the same identity multiple times, multiple positions can therefore be appended and influence the score. The source does not enforce “one vote per provider”.

**PARITY MUST:** do not silently add provider-local duplicate collapse before ranking.

## Position and provenance effects

On first insertion:

    positions = [provider_local_position]

On every duplicate merge:

    merge fields/provenance
    positions.append(provider_local_position)

The positions list is unlabelled in the compatibility model. Provider names are held separately in the `engines` set. Therefore the original structure cannot reconstruct which exact position came from which engine after merge.

An embedded tool may maintain an auxiliary labelled observation list for explainability, but scoring must still consume the compatibility positions list if exact parity is required.

## Secure-scheme preference during merge

Although scheme is ignored for identity, merge can alter the displayed URL. If the current merged URL scheme does not end in `s` and the duplicate's scheme does end in `s`, the merged parsed URL adopts the duplicate's scheme.

Thus HTTP/HTTPS identity equivalence and secure-scheme display preference are separate behaviors.

## Exact parity algorithm

Conceptually:

    normalize(result)
    key = python_hash(template, netloc, path, params, query, fragment, img_src)

    if key not present:
        result.positions = [position]
        map[key] = result
    else:
        merge_existing_with(result)
        map[key].positions.append(position)

The implementation can use an equivalent structured key instead of Python's randomized runtime hash only if the product deliberately prefers collision safety over byte-for-byte internal behavior. If so, golden tests should prove that all non-collision ordinary cases match.

## Golden tests

Compatibility tests must cover:

- identical URL from two providers merges;
- HTTP and HTTPS merge;
- differing `www` does not merge;
- differing query order does not merge;
- differing fragment does not merge;
- differing trailing slash does not merge;
- differing non-default port does not merge;
- differing `img_src` does not merge;
- differing thumbnail alone does not necessarily split identity;
- same-provider duplicate is merged by the same global mechanism and appends another position;
- wrapper decoding happens before identity;
- scheme-preference chooses the `...s` variant during merge;
- parity mode performs no tracking-parameter stripping or canonical-host aliasing.

## Deliberate deviations

Potential improvements include full-string keys to avoid hash collisions, HTTP(S)-only validation, tracking cleanup, provider-local duplicate collapse, and labelled positions. None should be hidden inside a supposed compatibility implementation. Add them only as named post-/pre-processing policies with separate tests.