# Overmind integration contract for `web.search`

## Purpose

This document specifies how the simplified web-search implementation is integrated into the current Overmind runtime without changing Overmind's trust boundaries.

The implementation agent may use idiomatic internal code, but the integration points in this document are architectural requirements for the current runtime.

## Existing Tool boundary

Overmind Tool implementations expose:

    name: str
    schema: dict
    execute(arguments, *, cancellation=None) -> dict

The current Tool protocol is synchronous. `ToolRegistry` performs exact-name dispatch and serializes the returned observation. The RuntimeStargate authorizes the Tool invocation before dispatch; the Tool implementation does not receive Runtime, Agent, Session, ContextCompiler, registries, schedulers, or a Core service locator.

The search component therefore belongs to Tool role, not Plugin role, for the first implementation.

## Canonical identity and operation

Use:

    component_identity = "tools/web_search"
    canonical tool name = "web.search"

These values must remain consistent across:

- the Tool implementation;
- the Tool schema;
- built-in extension discovery;
- first-party Tool operation policy;
- registration/authorization tests.

## Recommended package placement

Use the existing Tool package hierarchy:

    overmind/tools/web_search/
        __init__.py
        tool.py
        models.py
        coordinator.py
        transport.py
        normalize.py
        aggregate.py
        errors.py
        providers/
            __init__.py
            google.py
            bing.py
            duckduckgo.py

Optional test-private helpers/fixtures should live under the test tree, not inside production modules.

Names are recommendations; responsibility boundaries are the important part.

## Tool schema

V0 public parameters:

    query: string
    limit: integer, optional, default 10

Suggested schema constraints:

    query:
      type: string
      minLength: 1
      maxLength: 499

    limit:
      type: integer
      minimum: 1
      maximum: 10

    required: [query]
    additionalProperties: false

Runtime code must still validate values itself because schema validation is not a substitute for defensive Tool input checks.

V0 rejects query strings containing recognized bang syntax at the public contract level by using a conservative rule: any whitespace-delimited token beginning with `!` yields `UNSUPPORTED_QUERY_SYNTAX`. This avoids depending on the full external-bang dataset while preserving ordinary-query behavior.

## Tool execution

Conceptual shape:

    class WebSearchTool:
        component_identity = "tools/web_search"
        name = "web.search"
        schema = ...

        def execute(self, arguments, *, cancellation=None):
            validate arguments
            if cancellation:
                cancellation.raise_if_cancelled()
            return self._service.search_sync(...)

Do not turn `execute()` into an async method; the current Tool contract invokes it synchronously.

## SearchCoordinator execution model

For V0 use worker threads around synchronous `curl_cffi` requests or a private synchronous transport wrapper. This has several advantages in the current runtime:

- it matches the current synchronous Tool contract;
- it preserves the audited shared-deadline/thread semantics naturally;
- it avoids creating/destroying an asyncio loop on every Tool call;
- it avoids introducing a long-lived private event-loop service solely for one Tool.

An async internal transport can be adopted later if the runtime itself gains an async Tool contract. Do not block that future evolution by exposing thread details outside the search package.

## Cancellation integration

The current Overmind cancellation object is cooperative. The Tool must check it:

1. before provider dispatch;
2. during coordinator wait loops;
3. before expensive post-processing if cancellation becomes visible;
4. before returning the final observation.

Recommended wait loop:

    while unfinished workers and before shared deadline:
        cancellation.raise_if_cancelled()
        join/poll workers for a short bounded quantum

Do not wait for the entire three-second provider timeout in a single blocking join if a cancellation token is available.

On cancellation:

- mark unfinished provider results as no longer admissible;
- do not allow late workers to mutate the aggregate;
- propagate Overmind's existing cancellation exception;
- do not convert cancellation into an ordinary `{ok:false}` search failure.

The Runtime already understands cancellation as execution control, not provider failure.

## Built-in extension discovery

Add a factory equivalent in responsibility to existing built-in Tool factories:

    def _web_search(_root: Path):
        from overmind.tools.web_search import WebSearchTool
        return (WebSearchTool(),)

Add the component descriptor:

    ComponentDescriptor(
        "tools/web_search",
        "tool",
        "tools/web_search",
        _web_search,
        True,
    )

The factory must return a non-empty tuple of Tool objects and each Tool's `component_identity` must equal the descriptor identity.

## RuntimeStargate production policy

Add exactly this first-party Tool grant:

    ("tools/web_search", "web.search")

Do not authorize broader wildcard networking operations through RuntimeStargate merely because this Tool performs network I/O internally. Tool Surface authorizes the bounded Tool invocation; the Tool owns only its private capability implementation.

The Tool must not receive RuntimeStargate, Runtime, Application Surface ports, Agent, Session, or other Core objects.

## Registration path

At startup, normal built-in discovery should:

    discover component
      -> construct WebSearchTool family
      -> validate component identity
      -> register `web.search` in ToolRegistry
      -> freeze/commit the catalog according to normal startup flow
      -> include component identity in RuntimeStargate enablement

No search-specific registration path should be added to Core.

## Dependencies and portable runtime

Current Overmind portable runtime dependency closure is explicitly pinned and hash-locked. V0 search adds direct runtime dependencies:

    curl_cffi
    lxml

V0 does not need Babel because locale is fixed to `all`.

The implementation work must update all dependency ownership surfaces that Overmind already keeps aligned, including:

- the direct Python runtime requirements input;
- the generated hash-locked requirements file;
- the runtime manifest dependency mapping used by bootstrap/portable installation;
- CI install steps or shared dependency bootstrap logic where those dependencies are currently enumerated explicitly;
- any architectural baseline tests/docs that assert the runtime closure.

Do not simply add an import and assume developer-machine packages are sufficient.

Pin tested versions compatible with the project's current CPython runtime and target platform. The compatibility research snapshot used `curl_cffi 0.16.3` and `lxml 6.1.3`; the implementation agent must verify these exact pins against Overmind's supported Python/platform build before committing the lock update.

## Tool observation

The Tool returns a JSON-serializable dictionary. Suggested success shape:

    {
      "ok": true,
      "query": "python asyncio taskgroup",
      "results": [...],
      "providers": {...},
      "elapsed_ms": 420
    }

Suggested invalid-argument examples:

    {"ok": false, "error": "INVALID_ARGUMENTS"}
    {"ok": false, "error": "QUERY_TOO_LONG"}
    {"ok": false, "error": "UNSUPPORTED_QUERY_SYNTAX"}

Search/network provider failures should normally remain inside the provider diagnostics so partial successful results can still be returned.

If all providers fail, use a bounded shape such as:

    {
      "ok": false,
      "error": "SEARCH_UNAVAILABLE",
      "query": "...",
      "results": [],
      "providers": {...},
      "elapsed_ms": ...
    }

Do not include raw exception reprs that can contain provider-controlled data, request URLs, cookies, or tokens.

## Event metadata

If the Tool implements `event_metadata`, keep it low-cardinality and comfortably below Overmind's Tool metadata byte limit.

Useful fields:

    result_count
    provider_success_count
    provider_failure_count
    elapsed_ms

Avoid:

    query
    URLs
    snippets
    cookies
    tokens
    raw provider errors

## Tests required in Overmind

Add tests for:

- `WebSearchTool.name == "web.search"`;
- schema name matches canonical Tool name;
- component identity is `tools/web_search`;
- built-in discovery can construct the family;
- ToolRegistry accepts the family;
- duplicate/invalid registration rules remain unchanged;
- RuntimeStargate production policy authorizes only the exact `(component, operation)` pair;
- disabling the component removes it through the normal extension mechanism;
- cancellation propagates as execution cancellation rather than ordinary search failure;
- observations remain JSON serializable and bounded;
- production code never receives unrestricted Core objects.

## Architectural non-goals

Do not modify Core to understand:

- Google;
- Bing;
- DuckDuckGo;
- HTTP headers;
- search result ranking;
- provider state;
- search URLs.

Those are private implementation details of the Tool capability.

Do not add a public local HTTP server, Docker service, browser process, or daemon for search.

## Integration definition of done

The integration is correct when:

- `web.search` appears in the committed Tool schema surface when the component is enabled;
- RuntimeStargate authorizes the exact Tool operation through the normal Tool Surface;
- the Tool executes without access to unrestricted Core state;
- cancellation works through the existing token;
- search dependencies are present in developer, CI, bootstrap, and portable-runtime closures;
- disabling the component uses the existing extension configuration path;
- existing Tool/Plugin architectural tests remain green.
