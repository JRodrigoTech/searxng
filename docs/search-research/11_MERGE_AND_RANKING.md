# Merge and ranking

## Observed merge behavior

When two results have the same observed identity, the analyzed behavior:

- combines provider names in a set;
- appends each provider position to an unlabelled position list;
- prefers the longer title;
- prefers the longer snippet/content;
- fills empty fields from the other result;
- prefers a secure URL scheme over a non-secure one;
- preserves other populated fields unless a better field is selected.

The behavior does not retain a provider-to-position association in the position list. The independent design below should improve that loss of context.

## Independent merge model

For every identity, retain:

    providers: set[str]
    observations: list[
        {
          provider: str,
          position: int,
          url: str,
          title: str,
          snippet: str
        }
    ]

The provider-local normalizer must collapse duplicate identities from one provider before this stage. Thus a provider contributes at most one ranking observation per canonical destination.

## Field merge policy

| Field | Merge rule |
| --- | --- |
| Identity | Must be equal under the configured URL policy |
| Display URL | Prefer HTTPS; otherwise prefer the first valid URL; preserve all observed URLs |
| Title | Prefer non-empty; then prefer the longer bounded title; tie-break by earliest observation |
| Snippet | Prefer non-empty; then prefer the longer bounded snippet; tie-break by earliest observation |
| Published date | Prefer a valid date; if two conflict, keep earliest-observed value and record a conflict flag |
| Thumbnail | Prefer a valid non-empty value; do not fetch it |
| Metadata | Union allowlisted keys; do not overwrite a non-empty value with empty data |
| Providers | Set union |
| Positions | Keep labelled observations |

“Longer” is a useful heuristic for the observed behavior, but not a claim that longer text is more accurate. The bounds and tie-break rule make it deterministic.

## Merge examples

### Different titles and snippets

Provider A:

    title: API guide
    snippet: Short description
    position: 1

Provider B:

    title: Complete API guide for clients
    snippet: Longer description with useful context
    position: 2

Merged:

    title: Complete API guide for clients
    snippet: Longer description with useful context
    providers: A, B
    observations: A@1, B@2

### HTTP and HTTPS

Provider A returns http://example.test/docs at position 2.
Provider B returns https://example.test/docs at position 1.

If the identity policy treats HTTP and HTTPS as equivalent, the merged display URL is HTTPS, both observations remain, and the ranking sees positions 2 and 1.

### Conflicting published dates

Provider A reports January 1.
Provider B reports January 3.

Do not invent a reconciliation. Keep the first valid date, add a bounded conflict marker, or leave the merged date unset. The choice must be stable and tested.

## Observed ranking semantics

The analyzed score calculation has these steps:

1. Start with weight 1.
2. Multiply by the configured weight of every distinct provider present in the merged result.
3. Multiply by the number of provider positions/observations.
4. For normal priority, add weight divided by each position.
5. For high priority, add weight once per position.
6. For low priority, leave the score at zero.

With provider weights all equal to one, the normal-priority formula is:

    score = number_of_observations * sum(1 / position)

With provider weights:

    score = (number_of_observations
             * product(provider_weight for each provider))
             * sum(1 / position)

This product behavior can grow quickly when more than one provider weight exceeds one. Treat it as an observation to reproduce in compatibility tests, not as the default recommendation for a new subsystem.

## Worked ranking example

Assume equal provider weights and normal priority:

Provider A:

    R1 at position 1
    R2 at position 2

Provider B:

    R2 at position 1
    R1 at position 3

Provider C:

    R1 at position 2

R1 appears three times at positions 1, 3, and 2:

    observation count = 3
    position sum = 1/1 + 1/3 + 1/2
                   = 1 + 0.333333 + 0.5
                   = 1.833333
    score(R1) = 3 * 1.833333
              = 5.5

R2 appears twice at positions 2 and 1:

    observation count = 2
    position sum = 1/2 + 1/1
                   = 0.5 + 1
                   = 1.5
    score(R2) = 2 * 1.5
              = 3.0

Therefore the observed score order is R1, then R2 before any later category-grouping pass.

## Additional observed ordering

After score sorting, the analyzed result container performs a second grouping pass for certain category/template/image groups. It can move related results closer together and limits the number of results per group and a distance window.

**Recommendation:** Do not carry this application-specific grouping into V1. It makes ranking harder to explain and can violate the caller’s expectation that score order is final. If grouping is later required, make it a named post-ranker with its own tests.

## Recommended simplified V1 ranking

Use distinct-provider observations, one per provider, and a capped consensus multiplier:

    base = sum(provider_weight / position
               for each distinct provider observation)

    consensus = 1 + 0.35 * min(distinct_provider_count - 1, 2)

    score = base * consensus

Recommended default weights start at one for all three providers. The multiplier then becomes:

| Distinct providers | Consensus factor |
| ---: | ---: |
| 1 | 1.00 |
| 2 | 1.35 |
| 3 | 1.70 |

Properties:

- position one contributes more than position two;
- agreement increases confidence;
- the consensus bonus is capped;
- a single provider cannot receive multiple consensus bonuses for duplicate blocks;
- the formula is deterministic and easy to test.

Alternative scoring functions can be evaluated later, but the implementation should store component contributions for explainability.

## Tie-breaking

After descending score, sort by:

1. distinct provider count descending;
2. best provider position ascending;
3. canonical identity ascending;
4. display title using Unicode code-point order.

Do not use task completion order or dictionary insertion order as an implicit tie-break.

## Priority classes

The analyzed application can mark certain results high or low priority through unrelated host rules. This is NOT_NEEDED for the web-search V1. If a future caller supplies priority, make it explicit in SearchRequest and document how it interacts with the normal score; do not silently import host heuristics.
