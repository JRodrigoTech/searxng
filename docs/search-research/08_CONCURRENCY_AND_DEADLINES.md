# Concurrency and deadlines

## Required semantics

The search operation has one total monotonic deadline. All providers begin as close together as practical. A provider that succeeds remains usable even if another provider fails or times out.

For a total timeout T:

    start = monotonic()
    deadline = start + T

Every request, retry, and parse phase receives only:

    remaining = max(0, deadline - monotonic())

No child operation may replace the deadline with a fresh T.

## Observed behavior

The analyzed orchestration creates one worker per provider and shares a result container protected by a re-entrant lock. It then waits for each worker using the remaining time calculated against the common start time. A late worker is marked timed out, and results completed after that point are not accepted into the search result.

This produces correct partial-failure semantics, but the workers are threads and late workers may continue doing network work after the caller has returned. The new design should make cancellation explicit.

## Recommended async algorithm

The preferred model for an async AI runtime is one event loop, one task per provider, and an explicit deadline controller:

    validate SearchRequest
    start = monotonic()
    deadline = start + total_timeout

    prepare all provider requests
    create one task for each eligible provider

    while tasks remain and monotonic() < deadline:
        wait for the first completed task
        for each completed task:
            if it succeeded before the deadline:
                normalize its results
                add them to the merge accumulator
            if it failed:
                record typed ProviderFailure

    for each unfinished task:
        cancel it
    await cancellation completion with a small bounded grace period

    finalize scores and deterministic ordering
    return successful results plus diagnostics

A task that completes after the deadline must be treated as timed out even if its future becomes readable before cancellation has fully propagated.

## Parsing and aggregation choices

There are two safe designs:

1. Parse inside each provider task and return immutable ProviderOutcome objects to the coordinator.
2. Return bounded response objects and parse them in the coordinator after completion.

Recommendation: parse inside the provider task because the parser is provider-specific and the response should be discarded quickly. The task returns only bounded provider results and telemetry. The coordinator performs all merge mutations on the event-loop thread, eliminating the need for a shared lock.

If a thread-based client is unavoidable, use a queue of immutable outcomes and protect the merge index with a lock. Never let provider adapters mutate a common result object without synchronization.

## Task model comparison

| Model | Advantages | Risks for this search |
| --- | --- | --- |
| asyncio.gather with return_exceptions | Simple, preserves all outcomes, easy partial-failure handling | Must wrap a separate deadline and explicit cancellation |
| asyncio.TaskGroup | Strong structured concurrency and automatic cancellation | One uncaught task exception can cancel siblings; every provider exception must be converted into a value before it escapes |
| concurrent.futures | Works with blocking clients | Threads continue after timeout unless explicitly designed; shared state needs locking |
| Worker threads | Compatible with synchronous adapters | Harder cancellation, late network work, more hidden state |

**Recommendation:** Use asyncio tasks with value-based ProviderOutcome results. A TaskGroup is acceptable only when each provider catches all expected failures and the coordinator deliberately handles the group’s cancellation semantics. gather plus an explicit wait-for-deadline loop is easier to audit for a three-provider partial-result contract.

## Per-provider time budget

The total deadline is authoritative. A provider may also have a configured cap:

    provider_deadline = min(global_deadline,
                            provider_start + provider_timeout)

The provider timeout must not be greater than the global remaining time. If the provider is started late because request preparation took time, it receives less wall-clock time.

Recommended default values for an AI tool:

- total deadline: 3 to 5 seconds;
- provider network cap: no more than the total deadline;
- retry budget: zero or one, only for transient connect errors;
- parse budget: included in the total deadline;
- final result limit: 10;
- provider parse cap: 20 to 50 valid result blocks.

These are recommendations, not extracted facts.

## Performance budget

The first-page V1 path normally creates three network requests, one per provider, and at most three concurrent provider tasks. Later DuckDuckGo pages add a request only when a valid cached token exists. Trait discovery and locale refresh should not be on the critical path of every search.

Parsing cost is approximately linear in the bounded response size and number of selected result blocks. The implementation should:

- cap each provider response before parsing;
- stop collecting after a provider parse cap, such as 20 to 50 valid blocks;
- avoid walking unrelated navigation trees;
- discard raw response bytes after provider parsing;
- keep the merged identity map bounded by the provider caps;
- apply the final output cap only after merge and rank so consensus is not lost prematurely.

Connection reuse should make the common path pay only request/response latency after warm-up. A client pool can be shared across the three providers, but cookies and provider state remain provider-scoped. If the chosen HTTP library cannot isolate cookie jars while sharing connections, use separate provider clients or explicit cookie headers.

Suggested V1 upper bounds:

| Resource | Recommendation |
| --- | ---: |
| Concurrent providers | 3 |
| Requests for first page | 3 |
| Provider result blocks parsed | 50 |
| Final results returned | 10 |
| Response body | 2 MiB compressed / 8 MiB decompressed |
| Total wall-clock deadline | 3 to 5 seconds |
| Retry attempts | 0 or 1, only for transient connect failure |

These values are conservative recommendations. Measure elapsed transport time, parser time, response bytes, and merge counts before increasing them.

## Failure timing

| Event | Required behavior |
| --- | --- |
| Provider fails before any result | Record failure; keep siblings running |
| Provider succeeds quickly | Keep its results even if siblings later fail |
| Provider times out | Cancel/ignore unfinished work; mark Timeout |
| Provider returns malformed HTML | Mark ParseFailure; keep siblings |
| All providers fail | Return empty results with diagnostics |
| A result merges after another provider finishes | Merge on the coordinator side; preserve both positions |
| Cancellation is refused by a blocking client | Do not wait beyond the bounded grace period; quarantine the client if necessary |

## Determinism

The order in which provider tasks complete must not change final ordering. Accumulate provider results with explicit provider names and positions, then rank after all accepted outcomes have been collected. Tie-break with canonical identity and a stable provider-name rule.

## Cancellation and resource cleanup

When the deadline expires:

1. Cancel unfinished tasks.
2. Stop starting retries.
3. Close or quarantine response streams.
4. Await task cancellation briefly.
5. Ensure late callbacks cannot mutate a completed response.

If the HTTP library cannot cancel a request safely, it must still stop accepting its result and close the client/stream when it becomes available. Reusing a poisoned connection after cancellation can create cross-request state leaks.

## Concurrency tests

Use controllable fake providers and a fake monotonic clock or event barriers to test:

- all three succeed;
- one immediate failure;
- two failures;
- one timeout;
- all timeout;
- a slow success before the deadline;
- a success after the deadline;
- cancellation of unfinished tasks;
- deterministic output despite different completion orders;
- no result mutation after SearchResponse completion.
