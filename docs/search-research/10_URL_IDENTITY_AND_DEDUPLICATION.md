# URL identity and duplicate detection

## Separate identity from display

The system needs two related but different values:

- display_url — the destination presented to the caller;
- canonical_identity — a stable key used only to decide whether two results represent the same destination.

The display URL should preserve useful user-facing information. The identity key may normalize host case, default ports, or provider wrappers without rewriting the display URL.

## Observed identity behavior

**Observation:** The analyzed result merge key is built from a result template, parsed network location, path, parameters, query, fragment, and image-source value. The URL scheme is not included. The key is used directly in a dictionary.

Consequences:

- http and https versions of the same host/path/query merge;
- host case is not independently normalized by this key;
- default ports are not removed;
- www and non-www remain different;
- trailing slashes remain different;
- repeated slashes remain different;
- query parameter ordering remains different;
- tracking parameters remain part of identity;
- fragments remain part of identity;
- different image sources can make otherwise equal image-style results distinct;
- the dictionary key does not perform a second explicit equality check.

This is an implementation observation, not the recommended long-term identity policy.

## Recommended conservative identity policy

The V1 identity function should be explicit and collision-resistant:

    canonical_identity(url, policy) -> string

### Parsing

Reject:

- missing scheme;
- missing hostname;
- control characters;
- unsupported schemes;
- invalid port syntax.

Accept only HTTP and HTTPS for ordinary web results. Preserve non-standard schemes as a typed invalid-result condition rather than attempting to fetch them.

### Authority

- Lowercase the hostname.
- Convert Unicode hostnames to their IDNA ASCII form for identity.
- Remove the default port 80 for http and 443 for https.
- Preserve non-default ports.
- Preserve the distinction between www.example.test and example.test.
- Preserve userinfo only by rejecting it for a normal search result; userinfo in a result URL is not useful and can create credential leakage.

### Scheme

For the observed interoperability requirement, http and https may share an identity when authority, path, query, and fragment otherwise match. Do not equate either with another scheme.

This policy has a trade-off: an HTTP resource and an HTTPS resource can differ in content or access policy. If the caller requires security-sensitive separation, make scheme-sensitive identity a configuration option. The display merge policy should prefer https when the two values are otherwise equal.

### Path

- Preserve path case.
- Preserve repeated slashes.
- Preserve encoded slash versus literal slash.
- Normalize an empty path to / only when the URL parser requires it.
- Do not remove a trailing slash from a non-root path.
- Do not resolve dot segments unless a standards-tested URL library is used and the product accepts the semantic risk.

### Percent encoding

- Normalize hexadecimal case in percent escapes if the URL library provides a safe operation.
- Decode only unreserved characters for identity.
- Keep encoded reserved characters such as %2F distinct from literal slash.
- Never decode arbitrary query values before comparison.

### Query

V1 should retain query parameter names, values, and order after syntactic encoding normalization. This is the safest rule because parameters can be order-sensitive and many are semantic.

Tracking-parameter removal is a later, allowlisted policy, not a generic “drop everything that looks like tracking” rule. If enabled, begin with parameters that are unambiguously analytics-only, apply it only to known HTTP hosts, and keep both pre- and post-cleaning identities available for debugging.

Do not remove:

- q, query, id, item, page, p, path, version, or other content selectors;
- unknown parameters;
- signed or opaque parameters;
- parameters on hosts that have not been tested.

### Fragment

Keep fragments by default for conservative identity because a fragment can identify a document section or a client-side route. A page-identity mode may drop fragments when the product explicitly treats all anchors as one page. The mode must be tested separately.

## Provider wrapper removal

Unwrap before identity:

| Provider | Wrapper |
| --- | --- |
| Google | Relative /url?q= link; URL-decode and stop before &sa=U |
| Bing | ck/a wrapper; read u, remove a1, pad, URL-safe base64-decode |
| DuckDuckGo HTML | Direct result href in the active path |

If unwrapping produces another provider wrapper, do not recursively fetch it. Either apply one bounded decode rule or reject the item.

## Duplicate algorithm

1. Validate and unwrap the destination.
2. Produce canonical_identity.
3. Look up the identity in a map.
4. If absent, insert a new merged record.
5. If present, merge fields and append a labelled provider observation.
6. Keep the earliest provider position for each provider.
7. Recalculate score after all provider outcomes have been accepted.

The lookup map should store the complete identity string, not only a language-runtime hash. If a hash is used for indexing, compare the strings after a hash match to avoid collision-dependent merging.

## Examples

### Same destination, different scheme

    https://example.test/docs
    http://example.test/docs

Under the recommended HTTP/HTTPS-equivalence policy: merge, display HTTPS, retain both observed URLs and both providers/positions.

### Same host, different tracking parameter

    https://example.test/docs?article=7
    https://example.test/docs?article=7&utm_source=search

V1 conservative policy: do not merge unless an allowlisted host/parameter policy explicitly removes the tracking parameter. Retain both results if not proven equivalent.

### Same page, different fragment

    https://example.test/docs#intro
    https://example.test/docs#api

V1 conservative policy: keep separate identities. Optional page-identity mode may merge them later.

### www difference

    https://www.example.test/docs
    https://example.test/docs

Keep separate. DNS or application-level equivalence is not established by URL spelling.

### Query order

    https://example.test/search?a=1&b=2
    https://example.test/search?b=2&a=1

Keep separate in V1. Add host-specific query sorting only with evidence that the parameters are order-independent.

## Display URL selection

When merged records have different URLs:

1. Prefer a valid HTTPS URL over an HTTP URL when identity policy considers them equal.
2. Prefer the URL with fewer provider wrapper artifacts.
3. Prefer the first observed URL as a stable fallback.
4. Never replace a URL with an unvalidated decoded value.

The selected display URL must remain absolute and must not contain credentials.

## Future normalization layers

Later versions may add:

- a host-specific tracking allowlist;
- a configurable fragment policy;
- canonical-host aliases explicitly maintained by product configuration;
- provider-specific URL parameter normalization.

Do not add a broad heuristic normalizer without fixture tests; false-positive merging is harder to detect than missed deduplication.
