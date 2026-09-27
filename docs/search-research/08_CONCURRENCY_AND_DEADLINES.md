# Concurrency and deadlines

## Compatibility target

The externally important behavior is one shared search deadline, concurrent provider start, partial-result preservation, and rejection of results that arrive after the caller has stopped waiting. The audited implementation achieves this with worker threads around a synchronous bridge to an async HTTP loop.

An embedded async runtime may use native asyncio instead of threads, but it must preserve these observable semantics.

## Search clock

The search operation records one monotonic start time before standard provider execution:

    search_start = monotonic()

Request preparation determines one `actual_timeout` for the entire selected-provider search.

The default candidate timeout is the maximum configured timeout among eligible selected providers. It is then constrained by the optional caller query timeout and the optional global maximum request timeout:

- neither provided -> provider-derived default;
- query timeout only -> `min(default_timeout, query_timeout)`;
- global max only -> `min(default_timeout, global_max)`;
- both -> `min(query_timeout, global_max)`.

The active default per-engine timeout originates from the outgoing request timeout, 3.0 seconds, unless an engine overrides it.

## Capability filtering before workers

Before launching a provider worker:

1. a processor must exist;
2. the provider must not currently be suspended;
3. common capability validation must accept the requested page/time range;
4. request parameters must be constructible.

Page > 1 is skipped when the provider does not advertise paging. A configured maximum page is enforced. A time-range query is skipped when the provider does not advertise time-range support.

This means Bing is excluded from page > 1 or time-filtered searches in the audited path, while Google and primary DuckDuckGo remain eligible according to their capability flags.

## Worker launch

The audited orchestrator:

1. builds the full request list;
2. assigns one unique search identifier;
3. starts one `threading.Thread` per eligible provider before waiting for any of them;
4. passes each worker the same `search_start` and `actual_timeout`;
5. marks each worker initially as not timed out.

All providers therefore run concurrently even though waits are performed sequentially afterward.

## Waiting algorithm

After all workers are started, the caller iterates the matching worker threads. Before each join it computes:

    remaining = max(
        0,
        actual_timeout - (monotonic_now - search_start)
    )

and joins that worker for only `remaining` seconds.

If the worker is still alive after the join:

    worker._timeout = true
    record provider as unresponsive: "timeout"

The next worker receives a newly computed remaining budget from the **same** search start.

**PARITY MUST:** never give a later provider a fresh full timeout merely because an earlier join completed or timed out.

## Late-result suppression

A timed-out worker may continue running because Python threads cannot be forcibly cancelled safely.

When provider execution eventually tries to extend the shared result container, the common processor inspects the current worker's timeout marker:

- if timed out -> record timeout/error and do **not** insert its results;
- otherwise -> insert results and timing, then reset that provider's suspended state on success.

Thus late network work can continue internally, but late results do not mutate the accepted result set.

For a native-async port, cancellation can stop more work, but accepted-result semantics must remain identical: a result completing after the shared deadline must not enter the final result set.

## Provider failure isolation

Each online provider catches its own expected transport/provider errors. A failure in one worker records that provider as unresponsive and does not raise through the search orchestrator to terminate sibling workers.

Therefore:

    Google success
    Bing failure
    DuckDuckGo success

still produces the Google and DuckDuckGo results.

If all providers fail, the result container remains empty while provider-error diagnostics record failures.

## Shared result synchronization

The audited result container uses a re-entrant lock around duplicate lookup/merge and several diagnostic mutations. Provider threads can finish in any order and safely merge into the shared map.

A native-async implementation can avoid the lock by returning immutable provider outcomes to a single coordinator task, but it must reproduce result insertion semantics exactly, including provider-local position assignment before global merge.

## Network timeout interaction

Inside each worker, the common processor installs the same search start and timeout into network thread-local state.

Each network request then derives its effective wait from the common start. The sync bridge adds approximately 0.2 seconds of scheduling overhead and subtracts elapsed search time. Additional requests made inside a provider therefore consume the remaining search budget rather than receiving a new full search timeout.

This matters for:

- trait/request helpers executed in the search worker;
- retries;
- stateful continuation calls;
- any secondary request performed by a provider.

## Async port with parity semantics

A suitable implementation in Overmind can be:

    start = monotonic()
    deadline = start + actual_timeout

    create one provider task for every eligible provider

    while tasks remain:
        remaining = deadline - monotonic()
        if remaining <= 0:
            break
        wait for completions up to remaining
        accept only tasks completed before deadline
        convert provider exceptions to provider outcomes

    mark unfinished providers timed out
    cancel unfinished tasks where possible
    ignore all post-deadline results
    merge/rank only accepted provider results

Using `asyncio.gather(return_exceptions=True)` is acceptable if wrapped by the same single deadline. `TaskGroup` is acceptable only if expected provider failures are caught inside each task so one provider exception does not automatically cancel siblings.

## Completion order and deterministic expectations

The audited result map is mutated as provider workers finish. Python's sort is stable, and there is no explicit canonical tie-break after equal scores. Consequently, exact equal-score ordering can depend on insertion/completion order.

This is an important compatibility fact. The earlier proposal to add a canonical-identity/title tie-break would improve determinism but would change parity.

**PARITY MODE:** preserve stable-sort/insertion behavior.

**OPTIONAL DEVIATION:** a deterministic tie-break may be added later, but it must be named, tested, and excluded from source-parity golden tests.

## Timing diagnostics

For accepted successful provider results, timing records include:

- total elapsed time from common search start;
- accumulated HTTP load time tracked for that worker.

A provider marked late does not get to insert its successful result set merely because the underlying work later completes.

## Golden tests

Use fake providers/barriers to prove:

1. all providers are started before waiting begins;
2. the total wall-clock budget is shared, not multiplied by provider count;
3. a fast success survives a sibling failure;
4. a fast success survives a sibling timeout;
5. a worker finishing after the deadline cannot insert results;
6. provider exceptions remain local to that provider;
7. page/time capability filtering occurs before worker launch;
8. multiple network calls in one provider see a shrinking remaining budget;
9. equal-score tie order follows insertion order in parity mode;
10. an async implementation produces the same accepted-result set and timeout classification as the thread-based baseline.

## Non-parity enhancements

True task cancellation, bounded cancellation grace, response-body caps, and explicit structured outcomes are appropriate host-runtime improvements. They are safe additions only if tests prove they do not alter successful in-deadline provider behavior, merge order where parity is required, or request fingerprints.