# Merge and ranking

## Compatibility target

This document defines the exact merge and ordering behavior that the embedded search tool should reproduce before projecting results to the agent.

There are three distinct phases:

1. normalize and merge equal identities;
2. compute scores when the container closes;
3. sort by score, then run a category/template/image grouping pass.

The third phase is easy to miss and can change the final order even after scoring.

## Merge behavior

When a duplicate arrives, the existing result is mutated.

### Content

If the incoming `content` is longer than the existing content, replace the existing content.

### Title

If the incoming title is longer than the existing title, replace the existing title.

### Other default fields

The result model copies fields from the incoming result according to its default-field semantics when the existing result lacks a usable value. Exact field-copy details depend on typed versus legacy result representation, but this happens before provenance/scheme updates.

### Engine provenance

Add the incoming result's engine name to the existing result's `engines` set.

### URL scheme preference

If the existing parsed URL scheme does not end with `s`, but the incoming duplicate's scheme does end with `s`, replace the existing scheme with the incoming secure-suffixed scheme and regenerate the displayed URL.

This is broader than only `http -> https`; the source test is literally whether the scheme string ends in `s`.

### Position

After merge, append the incoming provider-local position to the existing unlabelled `positions` list.

There is no “best position only” reduction and no one-position-per-provider rule.

## Exact score calculation

Start:

    weight = 1.0

For every engine name in the merged result's `engines` set, when that engine has a configured weight:

    weight *= engine_weight

Then:

    weight *= len(positions)

Initialize:

    score = 0

For each position:

- priority `low`: contribute nothing;
- priority `high`: add `weight`;
- ordinary priority: add `weight / position`.

For normal web results, the formula is therefore:

    score = (
        product(weight_of_each_engine)
        * number_of_positions
        * sum(1 / position for position in positions)
    )

With all provider weights at the default 1.0:

    score = len(positions) * sum(1 / position)

## Important scoring consequences

- repeated appearances increase score through both the reciprocal-position sum and the multiplier `len(positions)`;
- duplicate occurrences from one provider can also increase score because positions are not provider-labelled and are not collapsed first;
- engine weights are multiplied together, not added;
- engine set iteration order does not affect multiplication mathematically, aside from ordinary floating-point effects;
- low-priority results score zero;
- high-priority results ignore actual numeric position except through the number of positions already included in `weight`.

## Worked example

Equal engine weights, ordinary priority:

Provider A:

    R1 @ 1
    R2 @ 2

Provider B:

    R2 @ 1
    R1 @ 3

Provider C:

    R1 @ 2

For R1:

    positions = [1, 3, 2]
    weight = 1 * 3 = 3
    score = 3/1 + 3/3 + 3/2
          = 3 + 1 + 1.5
          = 5.5

For R2:

    positions = [2, 1]
    weight = 1 * 2 = 2
    score = 2/2 + 2/1
          = 1 + 2
          = 3.0

First-pass score order is R1 then R2.

## Container close

When the result container closes, score is calculated once for every merged main result. The score is also attributed into engine metrics for every engine in that result's provenance set.

No further duplicate merge should occur after close.

## First ordering pass

Main results are sorted by:

    score descending

Python's sort is stable. There is no explicit secondary key such as provider count, canonical identity, title, or URL.

Therefore equal-score ordering follows the insertion order of the merged values, which can depend on provider completion/insertion timing.

**PARITY MUST:** do not introduce an explicit tie-break in compatibility mode.

## Second ordering/grouping pass

After score sorting, the audited container performs another pass.

For each result, it determines a category using the first configured category of the result's single primary/origin `engine` field when available.

It builds a group key equivalent to:

    category
    + template
    + image-marker

where the image marker is present if either `thumbnail` or `img_src` is non-empty.

The grouping state uses:

    max_count = 8
    max_distance = 20

When a compatible group has already been seen, still has group capacity, and is less than `max_distance` from the current output position, the result is inserted at the group's stored index rather than appended normally. Stored group indexes are then adjusted and the group's remaining count is decremented.

Otherwise the result is appended and a new/current group state is recorded.

### Why this matters for the three web providers

All three providers normally belong to the general/web family, but image-marker differences can split groups. Google ordinary web results can include a thumbnail while Bing and primary DuckDuckGo generally do not populate that field in the audited parser. Therefore this second pass can change pure score order.

A tool that returns `sorted(results, key=score)` and stops is not exact parity.

## Primary engine versus provenance set

A merged result contains:

- `engine`: one primary/origin engine value;
- `engines`: all engines that contributed to the merged result.

The grouping pass derives category from the singular `engine`, not from every provider in `engines`. This distinction must be preserved if exact final ordering matters.

## Output limit

The audited result container itself computes and exposes the ordered collection; product/UI layers can then limit what they display. For Overmind, apply the tool's final `limit` **after** compatibility merge, score calculation, and grouping so early truncation does not remove consensus information.

## Compatibility golden tests

Tests must cover:

- longer title wins;
- longer content wins;
- secure-suffixed scheme replaces insecure scheme when identities merge;
- provider engine set union;
- every duplicate appends its position;
- same-provider duplicates also append positions;
- exact normal-priority formula;
- non-default engine weights multiply;
- low priority produces zero;
- high priority adds the common weighted value per position;
- stable equal-score ordering follows insertion order;
- post-score grouping can reorder results;
- grouping key distinguishes image-bearing from non-image result groups;
- `max_count=8` and `max_distance=20` behavior;
- final result limit is applied after compatibility ordering.

## Deliberate deviations

A simpler reciprocal-rank score, capped consensus bonus, one-vote-per-provider policy, explicit deterministic tie-break, or removal of the grouping pass may be desirable for a future product ranking model. None is source parity. Keep alternative ranking behind an explicit mode and never use it in compatibility golden tests.