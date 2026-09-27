# Deterministic reference vectors

## Purpose

This file gives an implementation agent concrete, deterministic inputs and expected outputs for the simplified V0 search pipeline. The vectors are synthetic and protocol-shaped: they do not copy live provider pages. Their purpose is to verify that an independently written implementation follows the documented request, parse, normalization, merge, ranking, grouping, and error semantics.

These vectors complement live smoke tests. Passing them proves implementation behavior; it does not prove that external provider markup has not drifted.

# 1. Google request vector

Input:

    query = "python asyncio taskgroup"
    page = 1
    locale = all
    safe_search = 0
    time_range = None

Expected request method:

    GET

Expected endpoint:

    https://www.google.com/wml/search

Expected query pairs, ignoring URL-encoding order:

    q=python asyncio taskgroup
    sca_esv=1
    hl=en
    lr=
    cr=
    ie=utf8
    oe=utf8

Must be absent:

    start
    tbs
    safe
    num

Expected provider overrides:

    impersonate = chrome99_android
    allow_redirects = false
    User-Agent ∈ exact Nokia set from 04_GOOGLE_WEB_PROTOCOL.md

# 2. Google parser vector

Synthetic body:

    <?xml version="1.0"?>
    <html><body>
      <div class="zMzFAb">
        <a class="fuLhoc" href="/url?q=https%3A%2F%2Fexample.test%2Fdocs%3Fx%3D1%26y%3D2&sa=U&ved=abc">
          <span class="CVA68e">Example Docs</span>
        </a>
        <div class="taTFJ"><span class="FrIlee">Async   Python\nreference</span></div>
        <img src="https://encrypted-tbn.example/thumb"/>
      </div>
      <div class="zMzFAb">
        <span>not a normal result</span>
      </div>
    </body></html>

Expected provider output before common normalization:

    count = 1
    title = "Example Docs"
    url = "https://example.test/docs?x=1&y=2"
    content = "Async Python reference"
    thumbnail = "https://encrypted-tbn.example/thumb"

The second block is skipped because it lacks the required title element.

# 3. Bing request vector

Input:

    query = "python asyncio taskgroup"
    locale = all
    safe_search = 0

Expected method:

    GET

Expected endpoint:

    https://www.bing.com/search

Expected query pairs:

    q=python asyncio taskgroup
    adlt=off

Must be absent:

    setlang
    cc
    mkt

# 4. Bing wrapper vector

Synthetic destination:

    https://example.test/docs?x=1&y=2

UTF-8 bytes encoded with URL-safe base64 and without `=` padding:

    aHR0cHM6Ly9leGFtcGxlLnRlc3QvZG9jcz94PTEmeT0y

Wrapper `u` value:

    a1aHR0cHM6Ly9leGFtcGxlLnRlc3QvZG9jcz94PTEmeT0y

Synthetic href:

    https://www.bing.com/ck/a?u=a1aHR0cHM6Ly9leGFtcGxlLnRlc3QvZG9jcz94PTEmeT0y&ntb=1

Expected decoded href:

    https://example.test/docs?x=1&y=2

No extra absolute-URL validation belongs in the provider decoder.

# 5. Bing parser vector

Synthetic body:

    <html><body>
      <ol id="b_results">
        <li class="b_algo">
          <h2><a href="https://www.bing.com/ck/a?u=a1aHR0cHM6Ly9leGFtcGxlLnRlc3QvZG9jcz94PTEmeT0y&ntb=1">Example Docs Extended</a></h2>
          <p><span class="algoSlug_icon">decorative</span>Reference   snippet</p>
        </li>
        <li class="b_algo"><div>missing h2 link</div></li>
      </ol>
    </body></html>

Expected provider output:

    count = 1
    title = "Example Docs Extended"
    url = "https://example.test/docs?x=1&y=2"
    content = "Reference snippet"

The decorative exact-class `algoSlug_icon` span is removed before text extraction.

# 6. DuckDuckGo V0 request vector

Input query:

    "  python   asyncio\ttaskgroup  "

Expected DDG transformed query:

    "python asyncio taskgroup"

Expected method:

    POST

Expected endpoint:

    https://html.duckduckgo.com/html/

Expected form fields:

    q=python asyncio taskgroup
    b=
    kl=wt-wt

Expected cookies:

    none from `kl` or `df`

Expected explicit headers:

    Sec-Fetch-Dest: document
    Sec-Fetch-Mode: navigate
    Sec-Fetch-Site: same-origin
    Sec-Fetch-User: ?1
    Referer: https://html.duckduckgo.com/
    Content-Type: application/x-www-form-urlencoded

Expected common/final header:

    Accept-Language: en-US,en;q=0.9

Expected User-Agent grammar:

    Mozilla/5.0 (<one audited OS>; rv:<one audited Firefox version>) Gecko/20100101 Firefox/<same version>

The exact selected UA must remain stable for the lifetime of the provider instance.

# 7. DuckDuckGo parser vector

Synthetic body:

    <html><body>
      <form><input name="vqd" value="synthetic-vqd-1"/></form>
      <div id="links">
        <div class="web-result">
          <h2><a href="https://example.test/docs?x=1&y=2">Example Documentation</a></h2>
          <a class="result__snippet">Detailed   docs\nfor asyncio</a>
        </div>
        <div class="ad-result">
          <h2><a href="https://ads.example/">Ad</a></h2>
        </div>
      </div>
    </body></html>

Expected normal result output:

    count = 1
    title = "Example Documentation"
    url = "https://example.test/docs?x=1&y=2"
    content = "Detailed docs for asyncio"

Expected state side effect:

    cache synthetic-vqd-1 using transformed query + stable User-Agent
    expiry = 3600 seconds

The `ad-result` block is not selected.

# 8. Common normalization vector

Raw result:

    title = "  Alpha\t Beta\n Gamma  "
    content = "Alpha Beta Gamma"
    url = "https://example.test/docs?b=2&a=1#frag"

Expected normalized text:

    title = "Alpha Beta Gamma"
    content = ""

because normalized content equals normalized title.

Expected URL properties remain:

    scheme = https
    netloc = example.test
    path = /docs
    query = b=2&a=1
    fragment = frag

Query ordering and fragment remain unchanged.

# 9. Identity vector

These two results are duplicates in parity semantics:

    http://example.test/docs?x=1
    https://example.test/docs?x=1

when template and img_src are equal.

These are not duplicates:

    https://example.test/docs?x=1
    https://www.example.test/docs?x=1

These are not duplicates:

    https://example.test/docs?a=1&b=2
    https://example.test/docs?b=2&a=1

The conceptual identity string for an ordinary result is:

    default.html|example.test|/docs||x=1||

for URL `https://example.test/docs?x=1` and empty img_src.

Exact parity may use Python `hash(identity_string)` as the map key. A structured identity tuple is acceptable only as an explicitly documented collision-safety deviation.

# 10. Three-provider merge/rank vector

Provider outputs after normalization:

Google:

    position 1:
      title = "Example Docs"
      url = https://example.test/docs?x=1
      content = "short"
      thumbnail = https://thumb.example/a

    position 2:
      title = "Python Home"
      url = https://python.example/
      content = "Python"

Bing:

    position 1:
      title = "Example Documentation Extended"
      url = http://example.test/docs?x=1
      content = "a much longer explanatory snippet"

    position 2:
      title = "Asyncio Guide"
      url = https://async.example/guide
      content = "guide"

DuckDuckGo:

    position 1:
      title = "Example Documentation"
      url = https://example.test/docs?x=1
      content = "medium snippet"

Expected merge for Example identity:

    title = "Example Documentation Extended"
    content = "a much longer explanatory snippet"
    displayed scheme = https
    providers = {google, bing, duckduckgo}
    positions = [1, 1, 1] in whichever accepted insertion ordering occurs
    thumbnail = https://thumb.example/a if the Google-origin merged object remains the primary result; default-field semantics and completion order must be fixture-controlled when asserting this field

With all engine weights = 1.0 and ordinary priority:

    weight = 1 * len([1,1,1]) = 3
    score = 3/1 + 3/1 + 3/1 = 9.0

Python Home:

    positions = [2]
    score = 1/2 = 0.5

Asyncio Guide:

    positions = [2]
    score = 1/2 = 0.5

The two 0.5 results preserve insertion order in first-pass stable sorting. The final grouping pass may still alter relative placement depending on group key and current group state.

# 11. Same-provider duplicate vector

One provider emits:

    A @ position 1
    A @ position 3

and no other provider emits A.

Expected:

    positions = [1,3]
    engines = {that provider}
    weight = 2
    score = 2/1 + 2/3 = 2.6666666666666665

Do not collapse the two observations before scoring.

# 12. Google failure vectors

Classify as CAPTCHA/access denied when any one occurs:

- response host `sorry.google.com`;
- response path starts `/sorry`;
- response status = 302;
- body length < 2000 and body contains `/sorry/`.

Malformed one-result block is skipped locally; later Google result blocks still survive.

# 13. Bing malformed-wrapper vector

Synthetic wrapper:

    https://www.bing.com/ck/a?u=a1%%%%

If URL-safe base64 decoding raises for the selected runtime/library behavior, the exception escapes the provider parser and becomes provider-level failure. Do not catch it per item in compatibility mode.

# 14. DuckDuckGo failure vectors

HTTP status 303:

    expected = healthy empty provider result

Body containing:

    <form id="challenge-form"></form>

expected:

    provider CAPTCHA/access-denied classification
    explicit suspended_time = 0

# 15. Common HTTP error vectors

When status >= 400, classify in this order:

Cloudflare challenge candidate:

    status 429 or 503 AND
    body contains __cf_chl_jschl_tk__

or the audited challenge-platform triple.

Cloudflare CAPTCHA candidate:

    status 403 AND
    body contains __cf_chl_captcha_tk__

Cloudflare firewall:

    status 403 AND
    body contains <span class="cf-error-code">1020</span>

reCAPTCHA:

    status 503 AND
    body contains "https://www.google.com/recaptcha/

Then ordinary mapping:

    402/403 -> access denied
    429 -> too many requests
    other >=400 -> transport HTTP error

Cloudflare-specific classification requires a `Server` header beginning with `cloudflare` in the audited common layer.

# 16. Shared deadline vector

Synthetic workers:

    google completes at +0.10 s
    bing raises at +0.20 s
    duckduckgo completes at +3.20 s
    shared deadline = +3.00 s

Expected final accepted set:

    include google
    record bing failure
    mark duckduckgo timeout
    reject DDG late results even if worker later finishes

The total caller wait is approximately one shared deadline, not 3 seconds per provider.

# 17. Cancellation vector

Synthetic workers all sleeping for 2 seconds. Cancellation becomes true at +0.15 s.

Expected:

- coordinator detects cancellation during bounded polling;
- unfinished workers become inadmissible;
- runtime cancellation exception is propagated;
- no ordinary `SEARCH_UNAVAILABLE` observation replaces cancellation;
- any later worker completion cannot mutate accepted results.

# 18. V0 public response vector

If Google and DDG succeed and Bing fails:

    ok = true
    results = merged/ranked Google+DDG results
    providers.google.status = ok
    providers.bing.status = failed
    providers.duckduckgo.status = ok

If all three fail:

    ok = false
    error = SEARCH_UNAVAILABLE
    results = []

Provider diagnostics must be bounded and must not contain raw body, cookie, token, or complete request URL data.

# 19. Test ownership

An implementation agent should turn these vectors into tests before writing live transport code:

    tests/unit/tools/web_search/test_google.py
    tests/unit/tools/web_search/test_bing.py
    tests/unit/tools/web_search/test_duckduckgo.py
    tests/unit/tools/web_search/test_normalize.py
    tests/unit/tools/web_search/test_aggregate.py
    tests/unit/tools/web_search/test_coordinator.py
    tests/unit/tools/web_search/test_tool.py

Exact test paths may follow the repository's existing conventions, but the behavioral assertions above must remain covered.
