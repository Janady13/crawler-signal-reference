# framework-http-3xx-status-codes.md

Comprehensive reference for the HTTP 3xx Redirection status codes, the second framework in the status codes series after framework-http-2xx-status-codes.md. Covers `301 Moved Permanently` (strong canonical signal), `302 Found` (temporary, original URL retained), `303 See Other` (explicit method change to GET), `304 Not Modified` (conditional GET success, not actually a redirect), `307 Temporary Redirect` (preserves request method), and `308 Permanent Redirect` (preserves method, modern 301). Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to the eight HTTP header frameworks plus framework-http-2xx-status-codes.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx redirects, AI assistants generating redirect logic that preserves SEO equity, SEO operators planning site migrations or domain consolidations, security auditors checking for open redirect vulnerabilities, and anyone troubleshooting "ranking dropped after migration", "POST request becoming GET after redirect", "redirect chain showing in scanner reports", or "Google not transferring authority to new URL".

This framework also documents the **14 paying client subdomain redirect pattern** Joseph deployed in the recent Bubbles infrastructure overhaul (handledtax.thatwebhostingguy.com to handledtax.com, arcounselingandwellness.thatwebhostingguy.com to arcounselingandwellness.com, etc) as the canonical reference for similar future migrations.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The 3xx Mental Model (read this first)
5. The Method Preservation Matrix
6. 301 Moved Permanently (the canonical permanent redirect)
7. 302 Found (the temporary redirect with historical ambiguity)
8. 303 See Other (explicit method change to GET)
9. 304 Not Modified (the conditional GET success, not actually a redirect)
10. 307 Temporary Redirect (preserves request method, 302's strict cousin)
11. 308 Permanent Redirect (preserves request method, 301's modern equivalent)
12. The Canonicalization Signal Strength Hierarchy
13. The One Hop Rule (redirect chains are crawl budget killers)
14. The HTTP To HTTPS Plus HSTS Interaction
15. The Open Redirect Vulnerability (security)
16. The Bubbles Subdomain To Canonical Domain Pattern
17. How Major Crawlers React To Each 3xx Code
18. Asset Class And Use Case Recipes
19. Bubbles Nginx Reference Block (paste ready)
20. Audit Checklist (50+ items)
21. Common Pitfalls
22. Diagnostic Commands
23. Cross-References

---

## 1. DEFINITION

3xx status codes signal that the client must take additional action to complete the request. In practice, this means "go fetch a different URL". The response carries a `Location` header pointing to the new URL (for redirects) or signals that the cached version is still valid (for 304). Defined in RFC 9110.

The 3xx family splits along two axes:

**Permanence axis:**

* **Permanent**: 301, 308. The change is forever. Update bookmarks, internal links, sitemaps, canonical tags. Search engines transfer ranking signals to the new URL.
* **Temporary**: 302, 303, 307. The original URL remains canonical. Search engines keep the original indexed and continue crawling it.
* **Special**: 304 is not a redirect. It tells the client to use its cached copy.

**Method preservation axis:**

* **Method may change**: 301, 302. Per spec, the client MAY change POST to GET; in practice most browsers DO change. Used historically for navigation redirects.
* **Method explicitly forced to GET**: 303. Used after POST to redirect to a result page.
* **Method explicitly preserved**: 307, 308. POST stays POST, PUT stays PUT, DELETE stays DELETE. Used for API endpoints and form submissions.

The combination creates a 2x2 grid of redirect semantics (plus 303 and 304 as special cases):

```
                          Method may change          Method preserved
                          ──────────────────         ─────────────────
Permanent (301/308)       301 Moved Permanently      308 Permanent Redirect
Temporary (302/307)       302 Found                  307 Temporary Redirect
```

For SEO, the permanence axis is the critical one: 301 and 308 transfer ranking equity; 302 and 307 do not (the original URL stays in the index). For APIs and form submissions, the method preservation axis is critical: 307/308 keep POST intact; 301/302 may downgrade to GET.

---

## 2. WHY IT MATTERS

Eight independent pressures push correct 3xx usage from "default behavior" to "actively managed signal" in 2025 and forward.

**Permanent redirects transfer almost all ranking equity.** Google has confirmed since 2016 that PageRank is not lost across 30x redirects (no dampening). Site migrations using 301 or 308 retain nearly all accumulated authority. Misconfigured migrations using 302 or temporary redirects do not.

**Temporary redirects keep the original URL indexed.** Google treats 302 and 307 as signals that the original URL should remain canonical. The destination URL gets less attention. Sites that should be canonicalizing to a new domain but use 302 instead see the old URL persist in search results indefinitely.

**Redirect chains drain crawl budget.** Each hop in a chain is a request the crawler must follow before reaching content. Googlebot follows up to 10 hops then gives up. AI crawlers (ClaudeBot, GPTBot) typically follow fewer. A long chain on a high traffic page wastes thousands of crawler fetches per day.

**Method preservation matters for APIs and forms.** A client posts data to `/api/v1/users`, gets 301 to `/api/v2/users`. Browser changes POST to GET on the second request. Data is lost. The fix is 308 (which preserves POST) or designing API migrations to support both versions during transition.

**Browser caches redirects aggressively.** A 301 cached by a browser persists across sessions and is hard to clear. Bad 301s deployed accidentally affect users for weeks. Bubbles operators have to assume any 301 they deploy will be cached widely.

**HSTS interacts with HTTPS redirects.** A 301 from HTTP to HTTPS bootstraps HSTS. Once the browser has HSTS for the origin, it never even attempts HTTP again. This is correct behavior but means rollback is hard if HTTPS has problems.

**Open redirect vulnerabilities are common.** An endpoint like `/redirect?url=...` that takes any URL and returns 302 is a phishing tool: attackers craft `https://yoursite.com/redirect?url=https://evil.com/fake-login` and the URL appears legitimate to victims. Every site needs an allow list for redirect destinations.

**304 Not Modified saves enormous crawler bandwidth.** When the client sends `If-Modified-Since` or `If-None-Match` and the resource has not changed, returning 304 with no body is the correct response. Servers that always return 200 with full content waste crawler budget on every revisit.

**Cost of getting it wrong.** Misconfigured 3xx codes produce silent ranking damage, broken API integrations, and security incidents. Real examples:

* Site migration to new domain uses 302 instead of 301 because the developer was "not sure if permanent yet". Original domain stays indexed. New domain accumulates no ranking signal for months. Total traffic stays flat or drops despite better content.
* API endpoint moved from `/api/v1/` to `/api/v2/`. Used 301. Client POST requests become GET requests. Data lost. Three weeks of partial outage before someone correlates the migration with the data loss.
* Site had a 4-hop chain: `http://example.com/old` to `https://example.com/old` to `https://www.example.com/old` to `https://www.example.com/new`. Googlebot followed once, then 5,800 thin location pages drained the rest of the day's crawl budget. New content discovered weeks later than necessary.
* Open redirect on `/click?to=` parameter. Phishing campaigns used the trusted domain as a wrapper. Email security gateways flagged the site as a phishing source. Deliverability collapsed across the entire domain for two months.
* Conditional GET not implemented. Googlebot's revisits return 200 with full body every time. Crawl budget depleted. New pages discovered slowly.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the six status codes (300 is omitted, almost never used) gets the same six part treatment:

1. **What it means**: the canonical RFC definition plus practical implication.
2. **When to use it**: which scenarios warrant this specific code.
3. **How to return it in nginx and FastAPI**: paste ready config and code.
4. **How crawlers react**: indexing pipeline behavior per major crawler.
5. **How to verify**: curl commands and observation patterns.
6. **Common misuse and how to fix**: typical wrong choices and replacements.

Sections 12 to 16 are deep dives on cross cutting concerns: the canonicalization signal hierarchy, the one hop rule, the HSTS interaction, the open redirect vulnerability, and the Bubbles subdomain to canonical domain pattern (the 14 paying client redirect setup).

---

## 4. THE 3xx MENTAL MODEL (READ THIS FIRST)

A 3xx response says "this is not where the content is; go elsewhere". The client looks at the Location header (for redirects) and fetches that URL instead. Internalize the decision tree.

```
You need the client to look at a different URL. Why?
   |
   |---> The resource has permanently moved
   |     |
   |     |---> Method preservation matters (POST stays POST) .......... 308 Permanent Redirect
   |     |
   |     |---> Method preservation does NOT matter (typical) ........... 301 Moved Permanently
   |
   |---> The resource is temporarily at a different URL
   |     |
   |     |---> Method preservation matters (POST stays POST) .......... 307 Temporary Redirect
   |     |
   |     |---> Method preservation does NOT matter (typical) ........... 302 Found
   |
   |---> Request was POST; want the client to GET something else ........ 303 See Other
   |
   |---> Client has cached this, version is still fresh ................ 304 Not Modified
```

Six rules govern the system:

1. **Permanent (301/308) for SEO equity transfer.** Use these only when the move is genuinely permanent. The signal is hard to reverse.
2. **Temporary (302/307) for short term redirects.** Login flows, A/B tests, maintenance pages.
3. **303 after POST to force GET.** The classic post/redirect/get pattern to prevent double form submission.
4. **308 over 301 for API endpoints.** Method preservation prevents data loss.
5. **One hop only.** No chains. Update internal links to point at the final destination.
6. **304 needs no Location header.** It is not a redirect; it is "use your cache".

A correctly configured server uses permanent redirects sparingly and only when warranted, uses temporary redirects for genuinely temporary situations, preserves request methods for API and form endpoints, and never builds redirect chains.

---

## 5. THE METHOD PRESERVATION MATRIX

The single most critical operational detail in the 3xx family. Per RFC 9110:

| Status Code | Method Preservation |
|---|---|
| 301 Moved Permanently | Client **MAY** rewrite the method. Browsers historically rewrite POST to GET |
| 302 Found | Client **MAY** rewrite the method. Browsers historically rewrite POST to GET |
| 303 See Other | Client **MUST** use GET for the followup |
| 307 Temporary Redirect | Client **MUST** preserve the original method |
| 308 Permanent Redirect | Client **MUST** preserve the original method |

The "MAY rewrite" wording for 301/302 is responsible for years of subtle bugs. In practice:

* **Browsers** following 301 or 302 in response to a POST: **change to GET** (the de facto standard, not the spec).
* **HTTP libraries** (Python requests, curl, etc): behavior varies. Some preserve POST, some change to GET, some require explicit configuration.
* **API consumers**: never assume. Test the exact behavior.

For unambiguous behavior, use 307 (temporary, method preserved) or 308 (permanent, method preserved). They were specifically added in HTTP/1.1 and HTTP/1.1 RFC 7538 to remove the 301/302 ambiguity.

### 5.1 When Method Preservation Matters

* **API endpoints with POST/PUT/PATCH/DELETE**: client sends data; redirect to a new URL needs to carry the same data and method.
* **Form submissions**: HTML forms POSTing to an endpoint. If the endpoint moves, the form data must arrive at the new endpoint.
* **Webhook endpoints**: third party services POSTing JSON to a registered URL. If the URL changes, the POST must reach the new location intact.
* **Idempotent updates (PUT)**: client PUTs new data; redirect must deliver the PUT to the new URL.

### 5.2 When Method Preservation Does NOT Matter

* **Marketing site URLs**: humans clicking links (always GET).
* **Sitemap URLs**: crawlers GETting.
* **Image and asset URLs**: GETs.
* **Login flow redirects after authentication**: typically GET to the dashboard.

For these, 301 (permanent) or 302 (temporary) is fine because the client was going to GET anyway.

### 5.3 The 303 Special Case

303 is unique: regardless of the original method, the client MUST use GET to follow it. The canonical use case is the post/redirect/get (PRG) pattern:

1. Client POSTs form data to `/submit`.
2. Server processes the data (saves to database, sends email).
3. Server responds 303 to `/thank-you`.
4. Browser GETs `/thank-you`.
5. User refreshes the page: it GETs `/thank-you` again, not POSTing the form a second time.

Without 303, refreshing the post action page might re submit the form. With 303, the URL changes to the thank you page, refresh is safe.

In practice, 302 is often used for the same purpose (and works because browsers convert POST to GET on 302). 303 is the explicit, spec compliant version.

---

## 6. 301 MOVED PERMANENTLY (THE CANONICAL PERMANENT REDIRECT)

### 6.1 What It Means

301 Moved Permanently signals that the resource has been permanently relocated. The Location header points to the new canonical URL. Clients should update bookmarks, internal links should be updated, search engines should treat the new URL as canonical. Defined in RFC 9110.

```
HTTP/2 301 Moved Permanently
Location: https://example.com/new-url
```

### 6.2 When To Use 301

* **Site migration to a new domain.** `oldexample.com` to `newexample.com`.
* **HTTP to HTTPS redirect.** `http://example.com/*` to `https://example.com/*`.
* **WWW to non-WWW (or vice versa).** Pick a canonical hostname, redirect the other.
* **Trailing slash canonicalization.** `/page` to `/page/` or `/page/` to `/page`. Pick one.
* **URL restructuring.** `/blog/2024/01/article` to `/articles/article`.
* **Subdomain to canonical domain.** Joseph's pattern: `handledtax.thatwebhostingguy.com` to `handledtax.com`.
* **Removing duplicate URLs.** Multiple URLs serving the same content; pick one as canonical, redirect the rest.

### 6.3 When NOT To Use 301

* **Temporary maintenance or A/B testing.** Use 302 or 307 (the redirect is short term).
* **Login flow redirects.** Use 302 or 307 (login state can change).
* **POST/PUT/DELETE endpoint relocation.** Use 308 (method preservation).
* **You are not sure if the move is permanent.** Use 302 first; convert to 301 later once confirmed.

### 6.4 How To Return 301

**In nginx, three common patterns:**

Pattern 1: redirect entire server block:

```nginx
server {
    listen 443 ssl;
    server_name www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    return 301 https://example.com$request_uri;
}
```

Pattern 2: redirect specific path:

```nginx
location = /old-page {
    return 301 https://example.com/new-page;
}

location /old-section/ {
    return 301 https://example.com/new-section$request_uri;
}
```

Pattern 3: redirect entire domain:

```nginx
server {
    listen 443 ssl;
    server_name oldexample.com www.oldexample.com;

    ssl_certificate /etc/letsencrypt/live/oldexample.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/oldexample.com/privkey.pem;

    return 301 https://newexample.com$request_uri;
}
```

**In FastAPI:**

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.get("/old-page")
async def old_page():
    return RedirectResponse(url="/new-page", status_code=301)
```

### 6.5 How To Verify

```bash
# Confirm 301 status and Location
curl -sI https://example.com/old-page | head -3
# Expected:
# HTTP/2 301
# location: https://example.com/new-page

# Follow the redirect and confirm destination returns 200
curl -sIL https://example.com/old-page | head -10
# Expected:
# HTTP/2 301
# location: https://example.com/new-page
# (then)
# HTTP/2 200 (the destination)

# Verify the redirect transfers method (for browsers, POST becomes GET)
curl -X POST -sIL https://example.com/old-page | grep -E "^HTTP|^Location"
# Note: curl by default follows with GET after 301
```

### 6.6 Crawler Reaction

Per Google's documentation:

> "301 (moved permanently): Googlebot follows the redirect, and the indexing pipeline uses the redirect as a strong signal that the redirect target should be canonical."

Practical implication: 301 is the strongest canonical signal you can send. Combined with internal link updates, sitemap updates, and canonical tags pointing to the new URL, Google will transfer ranking signals fully within weeks (sometimes days for high authority sites).

Bing's behavior: similar to Google. Bing explicitly says "persistent 302 redirects get treated as 301", confirming that the permanent vs temporary distinction is what matters most.

ClaudeBot, GPTBot, OAI-SearchBot, PerplexityBot: follow 301 and treat the destination as the content URL. The old URL is effectively removed from their indices over time.

### 6.7 How To Fix Common Breakage

**Case: site migration used 302 instead of 301; new domain not ranking.**
Change to 301:

```nginx
# Was:
return 302 https://newexample.com$request_uri;
# Now:
return 301 https://newexample.com$request_uri;
```

Submit the change in Google Search Console (Change of Address tool) to accelerate. The transition may still take weeks.

**Case: old URL still appears in search results despite 301.**
The 301 is correctly returning but other signals contradict it. Check:
1. Internal links: do they still point to the old URL? Update them.
2. Sitemap: does it list the old URL? Remove it.
3. Canonical tags: do they point to the old URL? Update to new.
4. External backlinks: this is harder; over time, Google figures it out.

**Case: 301 cached aggressively, hard to remove.**
Browsers cache 301s. Once cached, changing the server response does not affect cached redirects. Options:
1. Wait it out (browser cache expires eventually).
2. Have users clear cache.
3. For aggressive cleanup, change the URL itself (cache miss on new URL).

---

## 7. 302 FOUND (THE TEMPORARY REDIRECT WITH HISTORICAL AMBIGUITY)

### 7.1 What It Means

302 Found signals that the resource is temporarily at a different URL. The original URL is still canonical and should be used for future requests. Defined in RFC 9110.

```
HTTP/2 302 Found
Location: https://example.com/temporary-location
```

The historical name was "Moved Temporarily" but RFC 7231 changed it to "Found" to reduce confusion with 301.

### 7.2 The Method Ambiguity

The HTTP/1.0 spec did not clearly define whether 302 should preserve the request method. Browsers implemented "change POST to GET" because that matched what users expected for navigation. The HTTP/1.1 spec acknowledged this as the de facto behavior. To fix the ambiguity, 303 (forces GET) and 307 (preserves method) were introduced.

In 2026, 302's method behavior depends on the client:

* **Browsers**: change POST to GET (de facto since 1996).
* **curl**: change POST to GET unless told otherwise.
* **Python requests**: change POST to GET on 302.
* **Strict HTTP libraries**: may follow the RFC strictly and preserve method.

For unambiguous behavior, use 307 or 303 instead.

### 7.3 When To Use 302

* **Login flow**: unauthenticated user requests `/dashboard`, gets 302 to `/login?return=/dashboard`. After login, 302 back to `/dashboard`. The original URL is still canonical.
* **A/B testing**: route some users to a variant URL temporarily. The canonical URL should not change.
* **Geo redirect**: send users to a regional variant temporarily. The main URL is still canonical.
* **Maintenance redirect**: route to a maintenance page during a scheduled window.
* **Short URL services**: `short.ly/abc` to `https://full-url.com/some-path`. Some short URL services use 302 because the destination might change.

### 7.4 When NOT To Use 302

* **Permanent moves**: use 301 or 308.
* **Site migrations**: use 301.
* **HTTPS upgrade**: use 301.
* **POST endpoints that must preserve method**: use 307.

### 7.5 How To Return 302

**In nginx:**

```nginx
# Maintenance redirect (temporary)
location / {
    return 302 https://example.com/maintenance;
}

# Login redirect (temporary)
location /dashboard {
    if ($cookie_session = "") {
        return 302 https://example.com/login?return=$request_uri;
    }
    try_files $uri =404;
}
```

**In FastAPI:**

```python
@app.get("/dashboard")
async def dashboard(request: Request):
    if not is_authenticated(request):
        return RedirectResponse(
            url="/login?return=/dashboard",
            status_code=302
        )
    return render_dashboard()
```

Note: FastAPI's `RedirectResponse` defaults to 307 (which preserves method). For 302 behavior, explicitly set `status_code=302`.

### 7.6 How To Verify

```bash
# Confirm 302
curl -sI https://example.com/dashboard | head -3
# Expected:
# HTTP/2 302
# location: /login?return=/dashboard

# Verify method behavior (browser would change POST to GET)
curl -X POST -sIL https://example.com/some-302-endpoint | grep -E "^HTTP|^Location"

# Test with explicit method preservation flag (curl --request)
# Most HTTP libraries are unpredictable on 302; test the actual client you care about
```

### 7.7 Crawler Reaction

Per Google's documentation:

> "302 (found): Googlebot follows the redirect, and the indexing pipeline uses the redirect as a weak signal that the redirect target should be canonical."

Practical implications:

* Original URL stays in the index.
* Destination URL gets crawled but may not be indexed.
* If 302 persists for an extended period, Google reassesses and may eventually treat the destination as canonical (Bing does this explicitly).

The Bubbles audit pattern: any 302 that has been in place for more than a few months should be reviewed. Is it actually temporary? If not, convert to 301.

### 7.8 How To Fix Common Breakage

**Case: 302 used for a permanent move; new URL not ranking.**
Convert to 301:

```nginx
# Was:
return 302 https://example.com/new;
# Now:
return 301 https://example.com/new;
```

**Case: API client expects POST to be preserved through 302, but it becomes GET.**
The client's library is following the de facto behavior. Two fixes:
1. Server side: switch to 307 (forces method preservation).
2. Client side: configure the HTTP library to preserve POST on 302 (varies by library).

The cleaner fix is the server side change to 307.

---

## 8. 303 SEE OTHER (EXPLICIT METHOD CHANGE TO GET)

### 8.1 What It Means

303 See Other signals that the response to the request can be found at another URL, which the client MUST retrieve with a GET request. Defined in RFC 9110. Used in the post/redirect/get pattern.

```
POST /api/orders HTTP/2
Content-Type: application/json

{"product_id": 12847, "quantity": 1}

----

HTTP/2 303 See Other
Location: /orders/abc123/confirmation
```

The client (browser) MUST follow with `GET /orders/abc123/confirmation`, even though the original request was POST.

### 8.2 When To Use 303

* **Post/redirect/get (PRG) pattern after form submission.** Prevents double submission on refresh.
* **Result of an action is at a different URL.** POST to `/transactions/transfer` returns 303 to `/accounts/12847/recent-activity`.
* **The response is logically separate from the action.** "I processed your request; here is where you can see the result."

### 8.3 When NOT To Use 303

* **You are returning the resource that was just created.** Use 201 with Location pointing to the new resource (see framework-http-2xx-status-codes.md). The client GETs that URL when it wants the full representation.
* **The result is at the same URL.** Use 200 with the response body.
* **The method should be preserved.** Use 307 or 308.

### 8.4 The PRG Pattern Explained

The post/redirect/get pattern is one of the foundational web UX patterns:

1. User submits a form with POST `/checkout`.
2. Server processes the order (charges card, creates order record).
3. Server responds 303 to `/orders/abc123/confirmation`.
4. Browser GETs `/orders/abc123/confirmation`.
5. User sees the confirmation page.
6. **Critical:** if user presses refresh, browser GETs `/orders/abc123/confirmation` again. Order is NOT charged twice.

Without 303 (or 302's de facto behavior of changing POST to GET), refreshing after a form submission would re submit the form, leading to duplicate orders, double posts, etc.

### 8.5 How To Return 303

**In nginx (unusual; typically the upstream does this):**

```nginx
# Rare; usually 303 comes from FastAPI sidecar
location = /submit-special {
    return 303 /thank-you;
}
```

**In FastAPI:**

```python
from fastapi.responses import RedirectResponse

@app.post("/checkout")
async def checkout(order: OrderForm):
    order_id = await process_order(order)
    return RedirectResponse(
        url=f"/orders/{order_id}/confirmation",
        status_code=303
    )
```

The explicit `status_code=303` is required; FastAPI's `RedirectResponse` default is 307.

### 8.6 How To Verify

```bash
# Simulate a form POST and observe 303
curl -sI -X POST -d "field=value" https://example.com/submit | head -3
# Expected:
# HTTP/2 303
# location: /thank-you

# Follow it; should be a GET
curl -sIL -X POST -d "field=value" https://example.com/submit | grep -E "^HTTP|^>"
# Look for: GET /thank-you (curl converted to GET after 303)
```

### 8.7 Crawler Reaction

Per Google's documentation, 303 is grouped with other 3xx codes in the documentation table. In practice:

> "303 (see other): Googlebot follows the redirect."

Crawlers do not POST so they rarely encounter 303. When they do (e.g., a third party link points to a POST result URL), they follow it as a GET to the Location target.

For SEO purposes: 303 is invisible to crawlers in normal operation.

---

## 9. 304 NOT MODIFIED (THE CONDITIONAL GET SUCCESS, NOT ACTUALLY A REDIRECT)

### 9.1 What It Means

304 Not Modified is the conditional GET success response. The client sent `If-Modified-Since` or `If-None-Match` (covered in framework-http-request-headers.md); the resource has not changed; the server returns 304 with no body. The client uses its cached copy.

```
GET /style.css HTTP/2
If-None-Match: "abc123"

----

HTTP/2 304 Not Modified
ETag: "abc123"
Cache-Control: public, max-age=86400
```

**304 is not a redirect.** It is grouped in the 3xx range historically but does not change the URL. The client uses its cached copy of the same URL.

### 9.2 When To Return 304

* The client sent `If-None-Match` and the resource's current ETag matches.
* The client sent `If-Modified-Since` and the resource's `Last-Modified` is not later than the request value.
* Both: ETag takes precedence.

### 9.3 The Strict Requirements

304 responses have specific requirements per RFC 9110:

* **MUST NOT** have a body.
* **MUST** include the `ETag` header if the resource has one (so the client knows the validator is still current).
* **MUST** include the `Last-Modified` header if the resource has one.
* **MAY** include `Cache-Control`, `Date`, `Expires`, `Vary`, and other caching related headers.
* **MUST NOT** include `Content-Type`, `Content-Length` (with values > 0), or other body describing headers.

### 9.4 How To Return 304

**In nginx (automatic for static files):**

```nginx
location /assets/ {
    root /var/www/sites/example.com;
    # nginx auto generates ETag from file mtime + size
    # On If-None-Match match, returns 304 automatically
    # On If-Modified-Since match, returns 304 automatically
}
```

No special config required. nginx handles conditional GET for static files.

**In FastAPI:**

```python
from email.utils import formatdate, parsedate_to_datetime
from datetime import datetime, timezone
import hashlib
from fastapi import FastAPI, Request, Response

app = FastAPI()

@app.get("/dynamic-resource")
async def dynamic_resource(request: Request):
    content = generate_content()
    etag = '"' + hashlib.md5(content.encode()).hexdigest()[:16] + '"'

    # Check If-None-Match (ETag takes precedence)
    inm = request.headers.get("if-none-match", "")
    if etag in [tag.strip() for tag in inm.split(",") if tag.strip()]:
        return Response(
            status_code=304,
            headers={
                "ETag": etag,
                "Cache-Control": "public, max-age=300",
            }
        )

    # Fall back to If-Modified-Since
    last_modified = datetime(2026, 5, 25, 14, 30, 0, tzinfo=timezone.utc)
    ims = request.headers.get("if-modified-since")
    if ims:
        try:
            ims_date = parsedate_to_datetime(ims)
            if last_modified <= ims_date:
                return Response(
                    status_code=304,
                    headers={
                        "ETag": etag,
                        "Last-Modified": formatdate(last_modified.timestamp(), usegmt=True),
                    }
                )
        except (TypeError, ValueError):
            pass

    return Response(
        content=content,
        media_type="text/html",
        headers={
            "ETag": etag,
            "Last-Modified": formatdate(last_modified.timestamp(), usegmt=True),
            "Cache-Control": "public, max-age=300",
        }
    )
```

### 9.5 How To Verify

```bash
# Get current ETag
ETAG=$(curl -sI https://example.com/style.css | grep -i "^etag:" | awk '{print $2}' | tr -d '\r')
echo "Current ETag: $ETAG"

# Send If-None-Match and verify 304
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -3
# Expected:
# HTTP/2 304
# etag: "..."
# (no content-length > 0)

# Verify body is empty
BYTES=$(curl -s -H "If-None-Match: $ETAG" https://example.com/style.css | wc -c)
echo "Body bytes: $BYTES (expected 0)"

# Send stale ETag and verify 200
curl -sI -H 'If-None-Match: "stale"' https://example.com/style.css | head -3
# Expected: HTTP/2 200 (with full body)
```

### 9.6 Crawler Reaction

Per Google's documentation:

> "304 (not modified): Googlebot signals the indexing pipeline that the content is the same as last time it was crawled."

Crawler implications:

* The crawler does not redownload the body.
* The crawler keeps the cached representation as the current content.
* Crawl budget is preserved (304 response is tiny: just headers).

This is the **single largest crawl budget optimization available** for a self hosted Bubbles site. Implementing conditional GET on dynamic responses can double or triple effective crawl coverage.

### 9.7 The Crawl Budget Math

Suppose Googlebot crawls 1000 URLs per day on your site. If 80% of those URLs have not changed since the last crawl:

* **Without 304**: 1000 full responses, average 10 KB each = 10 MB transferred per day.
* **With 304**: 200 full responses (10 KB each = 2 MB) + 800 304 responses (200 bytes each = 0.16 MB) = 2.16 MB transferred per day.

That is roughly **80% bandwidth saved** and **80% server CPU saved**. The crawl budget freed up by 304s is spent on new and changed pages, accelerating their discovery.

### 9.8 Common Misuse And How To Fix

**Case: 304 returned with body.**
A bug in the framework or custom code. Per spec, 304 MUST NOT have a body. Some clients fail to parse subsequent responses if 304 has a body. Fix:

```python
# WRONG
return Response(status_code=304, content="something")

# RIGHT
return Response(status_code=304, headers={"ETag": etag})
```

**Case: 304 missing ETag header.**
Without ETag in the response, the client does not know which version the 304 corresponds to. Always include the current ETag:

```python
return Response(
    status_code=304,
    headers={"ETag": current_etag}
)
```

**Case: server always returns 200 even when client sends If-None-Match.**
Server is ignoring conditional GET headers. Either nginx is not configured (try `etag on;`) or the upstream is bypassing the check. See Section 9.4.

---

## 10. 307 TEMPORARY REDIRECT (PRESERVES REQUEST METHOD, 302'S STRICT COUSIN)

### 10.1 What It Means

307 Temporary Redirect is the strict version of 302. Same semantics (resource temporarily elsewhere, original URL still canonical), but the client MUST preserve the original request method. Defined in RFC 9110.

```
POST /api/v1/data HTTP/2
Content-Type: application/json

{"item": "value"}

----

HTTP/2 307 Temporary Redirect
Location: /api/v1/data-staging
```

The browser MUST send `POST /api/v1/data-staging` with the same body. Compare to 302 where the browser would typically send GET (losing the body).

### 10.2 When To Use 307

* **API endpoint temporary relocation where method matters.** During a migration window, redirect POST to PUT (or to a different POST URL).
* **Webhook receivers temporarily forwarding to a backup endpoint.** Method preservation critical.
* **A/B testing on POST endpoints.** Route a percentage to a variant endpoint without changing method.
* **Form action endpoint relocation during deploy.** Form POSTs preserve.

### 10.3 When NOT To Use 307

* **Permanent moves**: use 308 (or 301 if method does not matter).
* **GET only resources**: use 302 (simpler, equivalent for GET).
* **Post/redirect/get pattern**: use 303 (the explicit method change).

### 10.4 The 302 vs 307 Decision

For GET requests, 302 and 307 are equivalent. The difference only matters for non GET methods. The rule:

* **Endpoint serves GET only**: 302 is fine. 307 is also fine (no difference for GET).
* **Endpoint may receive POST/PUT/PATCH/DELETE**: 307 is required.

For Bubbles client sites, most public URLs are GET only (humans clicking links). 302 is fine for those. For API endpoints and form processing endpoints, 307 is the safer default.

### 10.5 How To Return 307

**In nginx:**

```nginx
location /api/v1/data {
    return 307 /api/v1/data-staging;
}
```

**In FastAPI:**

```python
from fastapi.responses import RedirectResponse

@app.post("/api/v1/data")
async def relocated(request: Request):
    return RedirectResponse(
        url="/api/v1/data-staging",
        status_code=307
    )
```

Note: FastAPI's `RedirectResponse` defaults to 307, so the `status_code` parameter can be omitted in this case.

### 10.6 How To Verify

```bash
# POST to 307 endpoint, verify method preserved on followup
curl -sIL -X POST -d "data=test" https://example.com/api/v1/data 2>&1 | grep -E "^HTTP|^> POST|^> GET|^Location"

# Expected behavior: second request is also POST
# > POST /api/v1/data-staging HTTP/2
# (not GET)
```

### 10.7 Crawler Reaction

Similar to 302: Googlebot follows the redirect, uses it as a weak signal that the destination should be canonical. Original URL retained in the index.

For API endpoints (which crawlers should not be crawling anyway, per X-Robots-Tag: noindex), this is moot.

---

## 11. 308 PERMANENT REDIRECT (PRESERVES REQUEST METHOD, 301'S MODERN EQUIVALENT)

### 11.1 What It Means

308 Permanent Redirect is the strict version of 301. Same SEO semantics (permanent, strong canonical signal), but the client MUST preserve the original request method. Defined in RFC 7538.

```
POST /api/v1/users HTTP/2
Content-Type: application/json

{"name": "Joseph"}

----

HTTP/2 308 Permanent Redirect
Location: /api/v2/users
```

The browser MUST send `POST /api/v2/users` with the same body.

### 11.2 When To Use 308

* **Permanent API endpoint relocation where method matters.** `/api/v1/users` to `/api/v2/users` and all client POSTs must continue to work.
* **Form action endpoint permanent move.** Form submissions must route to the new endpoint.
* **Webhook receiver permanent move.** Webhook senders must continue posting.

### 11.3 When NOT To Use 308

* **Method preservation does not matter.** Use 301 (equivalent for GET, more widely understood).
* **The move is temporary.** Use 307 (temporary equivalent).

### 11.4 The 301 vs 308 Decision

For GET requests, 301 and 308 are equivalent. Google treats them identically for canonical signal strength. The difference only matters for non GET methods. The rule:

* **Endpoint is GET only**: 301 is fine. 308 is also fine.
* **Endpoint may receive POST/PUT/PATCH/DELETE**: 308 is required.

**No SEO upgrade from switching 301 to 308.** Google treats them equivalently. The only reason to use 308 over 301 is method preservation for non GET endpoints.

### 11.5 How To Return 308

**In nginx:**

```nginx
# Permanent API migration with method preservation
location /api/v1/ {
    return 308 /api/v2/$request_uri;
}

# Note: $request_uri starts with /api/v1/..., so this produces /api/v2//api/v1/...
# Use a regex or explicit handling instead:

location /api/v1/users {
    return 308 /api/v2/users;
}

# Or using a rewrite pattern (one location per major endpoint)
```

For more complex API migrations, FastAPI is usually the cleaner approach.

**In FastAPI:**

```python
from fastapi.responses import RedirectResponse

@app.api_route("/api/v1/users", methods=["GET", "POST", "PUT", "DELETE"])
async def v1_users_migrated():
    return RedirectResponse(
        url="/api/v2/users",
        status_code=308
    )
```

### 11.6 How To Verify

```bash
# Permanent API migration: POST should remain POST
curl -sIL -X POST -d '{"name":"test"}' \
    -H "Content-Type: application/json" \
    https://api.example.com/api/v1/users 2>&1 | grep -E "^HTTP|^> POST|^Location"

# Expected:
# HTTP/2 308
# Location: /api/v2/users
# > POST /api/v2/users HTTP/2
# (second request is POST with same body)
```

### 11.7 Crawler Reaction

Per Google's documentation: 308 is treated the same as 301 for canonical signal purposes. Strong signal that the destination is canonical.

In practice: 308 is invisible to crawlers because crawlers GET (and 301/308 are equivalent for GET).

---

## 12. THE CANONICALIZATION SIGNAL STRENGTH HIERARCHY

A summary of how strongly each redirect signals "use the destination as canonical":

| Signal Strength | Redirects | Notes |
|---|---|---|
| **Strongest** | 301, 308 | Google fully transfers ranking; old URL drops from index over weeks |
| **Weak** | 302, 307 | Google retains old URL as canonical; destination crawled but may not be indexed |
| **Method specific** | 303 | Forces GET; for the PRG pattern, not for canonicalization |
| **Not a canonical signal** | 304 | Same URL; no canonical change |

**Bing's adaptive behavior:** Bing treats persistent 302 as if it were 301 (after a few months). Conversely, "flapping" 301 (where the redirect target keeps changing) gets demoted to 302 strength.

**Google's behavior:** Google does similar reassessment over time but does not document it explicitly. A 302 that has been in place for a year may eventually be treated as canonical, but the path is unpredictable. The clean approach is to use the correct redirect type from the start.

**Practical rule for the Bubbles audit:** every redirect in the config should be examined. If it has been in place for more than 90 days and is genuinely permanent, it should be 301 or 308. If it is genuinely temporary, it should remain 302 or 307.

---

## 13. THE ONE HOP RULE (REDIRECT CHAINS ARE CRAWL BUDGET KILLERS)

### 13.1 The Problem

A redirect chain is a sequence of redirects that the client must follow before reaching the final destination:

```
http://example.com/old-page
   |
   v (301)
https://example.com/old-page
   |
   v (301)
https://www.example.com/old-page
   |
   v (301)
https://www.example.com/new-page
```

Each hop costs:

* **A round trip** from client to server (latency).
* **A crawler request** (crawl budget).
* **A weakened canonical signal** (the longer the chain, the less Google trusts the destination).

Googlebot follows up to 10 hops then gives up. Most modern AI crawlers follow fewer (PerplexityBot, ClaudeBot may give up after 3 to 5). Browsers follow up to 20 by default but flag warnings.

### 13.2 The Rule

**Maximum one hop from any source URL to the final destination.** Period.

The chain above should be collapsed to:

```
http://example.com/old-page
   |
   v (single 301)
https://www.example.com/new-page
```

### 13.3 How To Eliminate Chains

Audit existing redirects:

```bash
# Find chains by following redirects with curl
for url in $(curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g'); do
    HOPS=$(curl -sIL "$url" | grep -c "^HTTP")
    if [ "$HOPS" -gt 2 ]; then
        echo "CHAIN ($HOPS hops): $url"
    fi
done
```

Combine multiple redirects into one:

```nginx
# BEFORE: three redirects
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;  # hop 1
}

server {
    listen 443 ssl;
    server_name www.example.com;
    return 301 https://example.com$request_uri;  # hop 2
}

server {
    listen 443 ssl;
    server_name example.com;

    location /old-page {
        return 301 /new-page;  # hop 3
    }
}

# AFTER: collapsed
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;  # single redirect from HTTP
}

server {
    listen 443 ssl;
    server_name www.example.com;
    return 301 https://example.com$request_uri;  # www to non-www in one hop
}

server {
    listen 443 ssl;
    server_name example.com;

    location = /old-page {
        return 301 /new-page;  # direct redirect to final
    }
}

# Old URLs that should go to a new specific page:
# location = /http-www-old-page { return 301 https://example.com/new-page; }
```

The principle: every old URL goes directly to its final destination in one step. No intermediate hops.

### 13.4 The Internal Links Audit

Even with clean server side redirects, internal links pointing at old URLs cause unnecessary hops. Audit:

```bash
# Find HTML files containing references to URLs that get redirected
grep -r "https://example.com/old-page" /var/www/sites/example.com/ \
    | grep -v ".git"

# For each match, update to the new URL
```

For a Bubbles client site migration, this is a project: update every HTML file, every CMS database row, every sitemap entry to use the final URL. The redirect remains as a safety net for external links Joseph cannot control.

---

## 14. THE HTTP TO HTTPS PLUS HSTS INTERACTION

### 14.1 The Standard Pattern

Every Bubbles site upgrades HTTP to HTTPS via 301:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;
    # ...
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}
```

The 301 from HTTP to HTTPS is the bootstrap. After the first visit, HSTS (set on the HTTPS response) takes over and the browser never even tries HTTP again.

### 14.2 The HSTS Permanence

HSTS is one way (covered in framework-http-security-headers.md). Once a browser has HSTS for an origin:

* All requests to `http://` are automatically upgraded to `https://` before leaving the browser.
* Certificate errors cannot be clicked through.
* The behavior persists for the duration of `max-age`.

For HSTS preloaded sites, the behavior is permanent even on first visit (the browser ships with the preload list).

Operational implication: rolling back from HTTPS is hard. The 301 plus HSTS combination is effectively permanent.

### 14.3 The Single Hop From HTTP

For the HSTS bootstrap to be clean, the HTTP to HTTPS redirect must be a single hop:

```nginx
# CORRECT: one hop from HTTP to final HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

# WRONG: two hops
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://www.example.com$request_uri;  # hop 1: HTTP to HTTPS www
}

server {
    listen 443 ssl;
    server_name www.example.com;
    return 301 https://example.com$request_uri;  # hop 2: www to non-www
}
```

The correct pattern: HTTP to HTTPS canonical (non www) in one hop. www subdomain HTTPS to non www HTTPS as a separate one hop.

### 14.4 Verification

```bash
# Should be exactly one hop from HTTP
curl -sIL http://example.com/ | grep -c "^HTTP"
# Expected: 2 (the 301 and the final 200)

# Should be no upgrade attempt if HSTS is cached
curl -sI -H "Origin: https://example.com" http://example.com/
# Browser would auto upgrade; curl does not unless told

# Check HSTS is being sent on HTTPS responses
curl -sI https://example.com/ | grep -i strict-transport-security
# Expected: max-age=63072000; includeSubDomains; preload
```

---

## 15. THE OPEN REDIRECT VULNERABILITY (SECURITY)

### 15.1 What It Is

An open redirect is an endpoint that takes a URL as a parameter and redirects to it without validation:

```python
# DANGEROUS: open redirect
@app.get("/redirect")
async def redirect_to(url: str):
    return RedirectResponse(url=url, status_code=302)
```

A request to `https://example.com/redirect?url=https://evil.com/fake-login` returns 302 to evil.com. The URL in the browser bar reads as the trusted domain. Phishing attacks use this pattern routinely.

### 15.2 Why It Is Dangerous

Phishing emails want to use trusted domains to evade spam filters:

* Email contains link to `https://example.com/redirect?url=...`.
* Spam filter sees example.com is trusted.
* User clicks; browser hits example.com first, then redirects to evil.com.
* User's phishing detection trigger (the URL bar) shows the trusted domain in the link they clicked.

Email security gateways flag domains as phishing sources when open redirects are abused. The reputation damage to the trusted domain is real.

### 15.3 The Three Defense Patterns

**Pattern 1: Allow list of destinations.**

```python
ALLOWED_REDIRECT_HOSTS = {
    "example.com",
    "www.example.com",
    "app.example.com",
    "partner.example.com",
}

from urllib.parse import urlparse

@app.get("/redirect")
async def redirect_to(url: str):
    parsed = urlparse(url)
    if parsed.netloc not in ALLOWED_REDIRECT_HOSTS:
        raise HTTPException(status_code=400, detail="redirect destination not allowed")
    return RedirectResponse(url=url, status_code=302)
```

**Pattern 2: Path only redirects.**

```python
@app.get("/redirect")
async def redirect_to(path: str):
    if not path.startswith("/") or path.startswith("//"):
        raise HTTPException(status_code=400, detail="invalid path")
    return RedirectResponse(url=path, status_code=302)
```

Only allows redirects within the same origin. Cannot redirect to external sites.

**Pattern 3: Signed redirects.**

```python
import hmac
import hashlib

REDIRECT_SECRET = "..."  # from secret manager

@app.get("/redirect")
async def redirect_to(url: str, sig: str):
    expected_sig = hmac.new(
        REDIRECT_SECRET.encode(),
        url.encode(),
        hashlib.sha256
    ).hexdigest()[:16]

    if not hmac.compare_digest(sig, expected_sig):
        raise HTTPException(status_code=400, detail="invalid signature")

    return RedirectResponse(url=url, status_code=302)
```

The application can generate signed URLs for legitimate redirects; attackers cannot forge them.

### 15.4 The Bubbles Pattern

For Bubbles client sites that need internal redirects (login flow `return_to` parameter, etc), use Pattern 2 (path only):

```python
def safe_redirect_target(target: str, default: str = "/") -> str:
    """Return target if it is a safe internal path, else default."""
    if not target:
        return default
    if not target.startswith("/"):
        return default
    if target.startswith("//"):  # protocol relative URL
        return default
    if target.startswith("/\\"):  # backslash trick
        return default
    return target

@app.post("/login")
async def login(form: LoginForm, return_to: str = "/"):
    success = await authenticate(form)
    if success:
        safe_target = safe_redirect_target(return_to)
        return RedirectResponse(url=safe_target, status_code=302)
    raise HTTPException(status_code=401)
```

The `safe_redirect_target` helper prevents any redirect to a different origin.

---

## 16. THE BUBBLES SUBDOMAIN TO CANONICAL DOMAIN PATTERN

Joseph deployed this pattern during the recent infrastructure overhaul, redirecting 14 paying client subdomains from the `thatwebhostingguy.com` demo platform to their canonical custom domains.

### 16.1 The Pattern

Each paying client follows the same lifecycle:

1. **Stage 1 (Demo)**: client is on a subdomain of `thatwebhostingguy.com`. Example: `handledtax.thatwebhostingguy.com`. AdSense injection enabled; not yet paying.
2. **Stage 2 (Production)**: client buys their own domain (e.g., `handledtax.com`). The thatwebhostingguy.com subdomain stays as a 301 to the canonical domain. AdSense disabled on the redirect. SEO equity transfers to the new canonical.

### 16.2 The Nginx Configuration

```nginx
# /etc/nginx/sites-available/thatwebhostingguy-wildcard

server {
    listen 443 ssl;
    listen 443 quic;
    http2 on;
    http3 on;
    server_name *.thatwebhostingguy.com thatwebhostingguy.com;

    ssl_certificate /etc/letsencrypt/live/thatwebhostingguy.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/thatwebhostingguy.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # ============= 14 PAYING CLIENT 301 REDIRECTS =============
    # These subdomains are paying clients with their own canonical domains.
    # 301 transfers SEO equity from the demo subdomain to the production domain.

    if ($host = "arcounselingandwellness.thatwebhostingguy.com") {
        return 301 https://arcounselingandwellness.com$request_uri;
    }
    if ($host = "eurekabathworks.thatwebhostingguy.com") {
        return 301 https://eurekabathworks.com$request_uri;
    }
    if ($host = "heritagehardwoodfloors.thatwebhostingguy.com") {
        return 301 https://heritagehardwoodfloorsnwa.com$request_uri;
    }
    if ($host = "localliving.thatwebhostingguy.com") {
        return 301 https://locallivingrealtynwa.com$request_uri;
    }
    if ($host = "janieseecleaning.thatwebhostingguy.com") {
        return 301 https://showmecleannwa.com$request_uri;
    }
    if ($host = "showmecleannwa.thatwebhostingguy.com") {
        return 301 https://showmecleannwa.com$request_uri;
    }
    if ($host = "wecoverusa.thatwebhostingguy.com") {
        return 301 https://wecoverusa.com$request_uri;
    }
    if ($host = "white-river-cabins.thatwebhostingguy.com") {
        return 301 https://whiterivercabins.com$request_uri;
    }
    if ($host = "whiterivercabins.thatwebhostingguy.com") {
        return 301 https://whiterivercabins.com$request_uri;
    }
    if ($host = "greenoughsguideservice.thatwebhostingguy.com") {
        return 301 https://greenoughsguideservice.com$request_uri;
    }
    if ($host = "beyondastep.thatwebhostingguy.com") {
        return 301 https://beyondastep.com$request_uri;
    }
    if ($host = "idsofnwa.thatwebhostingguy.com") {
        return 301 https://idsofnwa.com$request_uri;
    }
    if ($host = "marshallese-voices.thatwebhostingguy.com") {
        return 301 https://marshallese-voices.com$request_uri;
    }
    if ($host = "nwapoolice.thatwebhostingguy.com") {
        return 301 https://nwapoolice.com$request_uri;
    }
    if ($host = "handledtax.thatwebhostingguy.com") {
        return 301 https://handledtax.com$request_uri;
    }
    if ($host = "handledtaxes.thatwebhostingguy.com") {
        return 301 https://handledtax.com$request_uri;
    }

    # ============= NON PAYING SUBDOMAINS (DEMO TIER) =============
    # All other subdomains serve from their wildcard subdirectory with AdSense

    set $subdomain "";
    if ($host ~ "^([^.]+)\.thatwebhostingguy\.com$") {
        set $subdomain $1;
    }

    root /var/www/sites/$subdomain;
    index index.html;

    # AdSense injection for non paying surfaces
    sub_filter '</head>' '<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-1158174357019592" crossorigin="anonymous"></script></head>';
    sub_filter_once on;

    # Phase 1 wildcard noindex (do not let demo subdomains compete with canonical sites)
    add_header X-Robots-Tag "noindex, follow" always;

    location / {
        try_files $uri $uri/ $uri.html =404;
    }
}
```

### 16.3 Verification

```bash
# Verify each paying client subdomain 301s to canonical
for sub in arcounselingandwellness eurekabathworks heritagehardwoodfloors localliving showmecleannwa wecoverusa whiterivercabins greenoughsguideservice beyondastep idsofnwa marshallese-voices nwapoolice handledtax; do
    DOMAIN="$sub.thatwebhostingguy.com"
    DEST=$(curl -sI "https://$DOMAIN/" | grep -i "^location:" | awk '{print $2}' | tr -d '\r')
    STATUS=$(curl -sI "https://$DOMAIN/" | head -1 | awk '{print $2}')
    echo "$DOMAIN -> $STATUS -> $DEST"
done

# Expected output (each line):
# arcounselingandwellness.thatwebhostingguy.com -> 301 -> https://arcounselingandwellness.com/
# etc.
```

### 16.4 The SEO Recovery Timeline

After deploying the 301 redirects:

* **Week 1**: Googlebot encounters the 301s. Crawls the destination URLs. Starts transferring ranking signals.
* **Month 1**: Canonical sites begin ranking for terms previously held by the subdomain demos.
* **Month 3**: Demo subdomain URLs drop out of search results.
* **Month 6**: SEO equity fully transferred. Old subdomain URLs effectively deindexed.

This is the standard timeline for clean 301 migrations. The Bubbles 14 client deployment was on this trajectory at the time of the recent overhaul.

---

## 17. HOW MAJOR CRAWLERS REACT TO EACH 3XX CODE

Quick reference for auditing.

| Code | Googlebot | Bingbot | ClaudeBot | GPTBot |
|---|---|---|---|---|
| 301 | Strong canonical signal; transfers ranking | Strong canonical signal | Treats destination as content URL | Treats destination as content URL |
| 302 | Weak canonical signal; keeps origin indexed | Persistent 302 eventually treated as 301 | Follows; less aggressive than 301 | Follows; less aggressive than 301 |
| 303 | Follows | Follows | Follows (rare, only after POST) | Follows |
| 304 | Uses cached representation; saves crawl budget | Uses cached | Uses cached if maintaining cache | Uses cached if maintaining cache |
| 307 | Weak signal (same as 302) | Treats like 302 | Follows | Follows |
| 308 | Strong signal (same as 301) | Strong signal | Follows | Follows |

**Key implications for Bubbles SEO:**

* For migrations: use 301 (or 308 if method matters).
* For login flows and A/B tests: 302 or 307.
* For HSTS bootstrap: 301 (the HTTP to HTTPS upgrade).
* For canonical hostname enforcement (www vs non www): 301.

---

## 18. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 18.1 Standard HTTP to HTTPS plus canonical hostname (Bubbles default)

```nginx
# HTTP to HTTPS (one hop)
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

# www HTTPS to non www HTTPS (one hop)
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    return 301 https://example.com$request_uri;
}

# Canonical server
server {
    listen 443 ssl;
    listen 443 quic;
    http2 on;
    http3 on;
    server_name example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    # ... rest of config ...
}
```

### 18.2 Specific page permanent redirect (301)

```nginx
location = /old-page {
    return 301 /new-page;
}

# Or with regex for pattern matching
location ~* ^/blog/2024/[0-9]+/(.+)$ {
    return 301 /articles/$1;
}
```

### 18.3 Login flow redirect (302 with return_to)

```python
@app.get("/dashboard")
async def dashboard(request: Request):
    if not is_authenticated(request):
        return RedirectResponse(
            url=f"/login?return_to=/dashboard",
            status_code=302
        )
    return render_dashboard()

@app.post("/login")
async def login(form: LoginForm, return_to: str = "/"):
    if await authenticate(form):
        safe_target = safe_redirect_target(return_to)
        return RedirectResponse(url=safe_target, status_code=303)
    raise HTTPException(status_code=401)
```

### 18.4 Maintenance window (302 with Retry-After)

```nginx
server {
    location / {
        return 302 /maintenance;
    }

    location = /maintenance {
        root /var/www/sites/example.com;
        try_files /maintenance.html =503;
        add_header Retry-After "3600" always;
        add_header Cache-Control "no-store" always;
    }
}
```

After maintenance: remove the `return 302 /maintenance;` and reload nginx.

### 18.5 Post/redirect/get pattern (303)

```python
@app.post("/checkout")
async def checkout(order: OrderForm):
    order_id = await process_order(order)
    return RedirectResponse(
        url=f"/orders/{order_id}/confirmation",
        status_code=303
    )

@app.get("/orders/{order_id}/confirmation")
async def confirmation(order_id: str):
    order = await get_order(order_id)
    return render_confirmation(order)
```

### 18.6 API endpoint permanent migration with method preservation (308)

```python
@app.api_route("/api/v1/users", methods=["GET", "POST", "PUT", "DELETE"])
async def v1_users_migrated():
    return RedirectResponse(
        url="/api/v2/users",
        status_code=308
    )
```

### 18.7 API endpoint temporary migration with method preservation (307)

```python
@app.api_route("/api/v1/payments", methods=["POST", "PUT"])
async def v1_payments_temp():
    return RedirectResponse(
        url="/api/v1/payments-backup",
        status_code=307
    )
```

### 18.8 Conditional GET for dynamic content (304)

```python
from email.utils import formatdate
import hashlib

@app.get("/api/data")
async def api_data(request: Request):
    content = generate_data()
    body = json.dumps(content).encode()
    etag = '"' + hashlib.md5(body).hexdigest()[:16] + '"'

    inm = request.headers.get("if-none-match", "")
    if etag in [tag.strip() for tag in inm.split(",") if tag.strip()]:
        return Response(
            status_code=304,
            headers={"ETag": etag, "Cache-Control": "public, max-age=60"}
        )

    return Response(
        content=body,
        media_type="application/json",
        headers={"ETag": etag, "Cache-Control": "public, max-age=60"}
    )
```

### 18.9 Sitemap migration: redirect old sitemap to new

```nginx
location = /sitemap.xml {
    return 301 /sitemap-new.xml;
}

location = /sitemap-new.xml {
    root /var/www/sites/example.com;
    try_files $uri =404;
    add_header Content-Type "application/xml" always;
}
```

### 18.10 Safe internal redirect helper

```python
from urllib.parse import urlparse

def safe_redirect_target(target: str, default: str = "/") -> str:
    """Return target if it is a safe internal path, else default."""
    if not target:
        return default
    if not target.startswith("/"):
        return default
    if target.startswith("//"):  # protocol relative URL
        return default
    if target.startswith("/\\"):  # backslash trick
        return default
    parsed = urlparse(target)
    if parsed.scheme or parsed.netloc:
        return default
    return target
```

### 18.11 Bubbles subdomain to canonical pattern (from Section 16)

See Section 16.2 for the complete reference block.

### 18.12 Trailing slash canonicalization

Pick one (with or without trailing slash) and redirect the other:

```nginx
# Option A: enforce no trailing slash
rewrite ^/(.*)/$ /$1 permanent;

# Option B: enforce trailing slash on directories only
location / {
    try_files $uri $uri/ $uri.html =404;
}
```

### 18.13 Removed section permanent redirect to category

```nginx
# Old section removed; redirect to category
location /old-section/ {
    return 301 /category/related;
}
```

Note: redirect to a relevant category, NOT to the homepage. Redirecting deleted pages to the homepage is the soft 404 trap (covered in framework-http-2xx-status-codes.md).

---

## 19. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete 3xx aware configuration for a Bubbles client site.

```nginx
# ============= HTTP to HTTPS canonical (single 301) =============
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

# ============= www HTTPS to non www HTTPS (separate single 301) =============
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    return 301 https://example.com$request_uri;
}

# ============= CANONICAL SERVER =============
server {
    listen 443 ssl;
    listen 443 quic;
    listen [::]:443 ssl;
    listen [::]:443 quic;
    http2 on;
    http3 on;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
    include snippets/common-security-headers.conf;

    root /var/www/sites/example.com;
    index index.html;

    # ============= LEGACY URL 301 REDIRECTS =============
    # Each legacy URL: single 301 to final destination

    location = /old-home {
        return 301 /;
    }

    location = /services-page {
        return 301 /services/;
    }

    location ~* ^/blog/[0-9]+/[0-9]+/(.+)$ {
        return 301 /articles/$1;
    }

    location ~* ^/old-product/(.+)$ {
        return 301 /products/$1;
    }

    # ============= TRAILING SLASH NORMALIZATION =============
    # Pick: with slash or without. Below: enforce with slash on directories
    # No explicit redirect rule; nginx try_files handles via 404 if mismatched

    # ============= API ENDPOINT MIGRATION (308 preserves method) =============
    location = /api/v1/users {
        return 308 /api/v2/users;
    }

    # ============= STATIC ASSETS (conditional GET produces 304 automatically) =============
    location ~* \.(css|js|woff2|jpg|jpeg|png|webp|avif|svg)$ {
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        # nginx returns 304 automatically on If-None-Match / If-Modified-Since match
    }

    # ============= HTML PAGES =============
    location ~* \.html$ {
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        # nginx returns 304 if client validates with current ETag
    }

    # ============= LOGIN FLOW (302 with safe return_to) =============
    location = /dashboard {
        # Upstream handles 302 to /login?return_to=/dashboard
        proxy_pass http://127.0.0.1:9090;
    }

    location = /login {
        proxy_pass http://127.0.0.1:9090;
        # FastAPI handles authentication and 303 post-redirect-get
    }

    # ============= API =============
    location /api/ {
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            return 204;
        }
        add_header X-Robots-Tag "noindex, nofollow" always;
        proxy_pass http://127.0.0.1:9090;
    }

    # ============= ROOT =============
    location / {
        try_files $uri $uri/ $uri.html =404;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify the redirect chain is exactly one hop:

```bash
# HTTP to canonical HTTPS: should be exactly one redirect
curl -sIL http://example.com/ | grep -c "^HTTP"
# Expected: 2 (the 301 + the final 200)

# WWW HTTPS to non-WWW HTTPS: also exactly one redirect
curl -sIL https://www.example.com/ | grep -c "^HTTP"
# Expected: 2 (the 301 + the final 200)

# Legacy URL to new URL: also exactly one redirect
curl -sIL https://example.com/old-home | grep -c "^HTTP"
# Expected: 2 (the 301 + the final 200)

# Trace each redirect explicitly
curl -sIL https://example.com/old-home | grep -E "^HTTP|^Location:"
```

---

## 20. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### Canonical redirect setup

1. [ ] HTTP to HTTPS uses 301 (not 302).
2. [ ] HTTP to HTTPS is a single hop to the canonical hostname.
3. [ ] WWW to non WWW (or vice versa) uses 301.
4. [ ] WWW HTTPS to non WWW HTTPS is a single hop.
5. [ ] HSTS is set on the canonical HTTPS responses.
6. [ ] Trailing slash policy is consistent (chosen and enforced).

### Permanent redirects (301/308)

7. [ ] All site migrations use 301 (or 308 if method preservation needed).
8. [ ] API endpoint migrations use 308 (method preserved).
9. [ ] Legacy URLs redirect to relevant new URLs, NOT to the homepage.
10. [ ] No 301 redirects to error pages or thin content (would create soft 404 chains).
11. [ ] All 301 redirects verified working with curl -sIL.
12. [ ] Internal links updated to point to the final URL, not via redirect.
13. [ ] Sitemap lists final URLs, not redirected ones.
14. [ ] Canonical tags point to final URLs.

### Temporary redirects (302/307)

15. [ ] Login flows use 302 or 303.
16. [ ] Maintenance redirects use 302 with Retry-After.
17. [ ] No 302 redirects in place longer than 90 days without review.
18. [ ] A/B test redirects use 302 (not 301).
19. [ ] Geo redirects use 302 (canonical URL stays in index).

### Method preserving redirects (307/308)

20. [ ] API endpoints that accept POST/PUT/PATCH/DELETE use 307 (temp) or 308 (perm).
21. [ ] Form action endpoints that may be redirected use 307/308.
22. [ ] Webhook receiver endpoints use 308 for permanent moves.

### 303 (post/redirect/get)

23. [ ] Form submissions follow PRG pattern: POST to 303 to GET.
24. [ ] No double submission on browser refresh.

### 304 (conditional GET)

25. [ ] Static assets return 304 when If-None-Match matches.
26. [ ] Static assets return 304 when If-Modified-Since matches.
27. [ ] Dynamic responses implement conditional GET where appropriate.
28. [ ] 304 responses include current ETag (or Last-Modified).
29. [ ] 304 responses have empty body.
30. [ ] ETag generation is content based (not timestamp based).

### Chain prevention

31. [ ] No redirect chains exceeding one hop.
32. [ ] Audit script run periodically: `for url in $(sitemap); do count hops; done`.
33. [ ] Mixed redirect types (301to302) eliminated; one type per chain.

### Security

34. [ ] No open redirect on any endpoint accepting URL parameters.
35. [ ] If redirect endpoint exists: allow list OR path only OR signed.
36. [ ] `safe_redirect_target()` helper used in FastAPI sidecar.
37. [ ] No redirects to user supplied protocol relative URLs (//).

### Bubbles specific

38. [ ] The 14 paying client subdomain 301s verified working.
39. [ ] Wildcard subdomain noindex applied to non paying tier.
40. [ ] AdSense not injected on the 301 responses (only on serving demo content).

### Verification

41. [ ] `curl -sIL` confirms expected single hop on every redirect.
42. [ ] HSTS header present on HTTPS responses after the 301.
43. [ ] GSC Page Indexing shows no redirect chain errors.
44. [ ] Screaming Frog or similar shows no chains exceeding one hop.
45. [ ] Browser DevTools Network tab shows single hop on critical paths.

### Cross cutting

46. [ ] nginx -t passes without warnings.
47. [ ] nginx -T | grep "return 30" shows expected status codes.
48. [ ] No accidental 302 where 301 is intended.
49. [ ] Monitoring alerts on unexpected redirect status codes.
50. [ ] Quarterly review of all redirects to verify they are still needed.

A site that passes all 50 has correctly configured 3xx redirects for SEO equity, API correctness, security, and crawl efficiency.

---

## 21. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: 302 used for permanent migration.**
Symptom: new domain not ranking; old URL stays in search results.
Why it breaks: 302 is weak signal; Google keeps old URL canonical.
Fix: change to 301. Submit Change of Address in GSC.

**Pitfall 2: 301 used for temporary maintenance redirect.**
Symptom: site URL "stuck" at maintenance page even after maintenance ends.
Why it breaks: browsers and crawlers cache 301 aggressively.
Fix: use 302 (or 503 with Retry-After) for temporary redirects.

**Pitfall 3: Redirect chain (4+ hops).**
Symptom: slow page loads; crawl budget wasted.
Why it breaks: each hop is a round trip.
Fix: collapse chains. Every old URL to final URL in one hop.

**Pitfall 4: POST endpoint redirected with 301; data lost.**
Symptom: form submissions or API calls result in empty data at new endpoint.
Why it breaks: browsers convert POST to GET on 301.
Fix: use 308 instead.

**Pitfall 5: Open redirect on `/redirect?url=`.**
Symptom: phishing campaigns abuse the domain; deliverability collapses.
Why it breaks: any external URL accepted.
Fix: allow list, path only, or signed redirects.

**Pitfall 6: 301 to homepage for all removed pages.**
Symptom: removed product pages all redirect to homepage; soft 404 by Google.
Why it breaks: irrelevant destination; Google detects no canonical match.
Fix: 410 for permanently removed pages, or 301 to a relevant category.

**Pitfall 7: 304 returned with body.**
Symptom: some HTTP clients fail to parse subsequent responses.
Why it breaks: RFC says 304 MUST have no body.
Fix: return empty body from server.

**Pitfall 8: 304 missing ETag header.**
Symptom: client cannot determine which version 304 refers to.
Why it breaks: ETag is the validator; without it, next revalidation fails.
Fix: always include current ETag on 304 response.

**Pitfall 9: HTTP to HTTPS redirect is two hops (HTTP to HTTPS www to HTTPS canonical).**
Symptom: extra latency on every first visit.
Why it breaks: two separate 301s instead of one.
Fix: redirect HTTP directly to canonical HTTPS in one step.

**Pitfall 10: `return 302` with `if` containing `proxy_pass`.**
Symptom: redirect intermittent or wrong.
Why it breaks: nginx `if + proxy_pass` antipattern.
Fix: use map or location matching instead of conditional if blocks.

**Pitfall 11: 302 for HTTPS canonical (instead of 301).**
Symptom: HTTPS variant not picked up as canonical.
Why it breaks: 302 is weak signal; original (HTTP) stays canonical in Google's mind.
Fix: 301.

**Pitfall 12: Conditional GET ignored on dynamic responses.**
Symptom: crawlers download full body on every visit.
Why it breaks: upstream code does not check If-None-Match / If-Modified-Since.
Fix: implement conditional GET in upstream (Section 9.4).

**Pitfall 13: Browser cached 301 prevents rollback.**
Symptom: changing the server returns the new behavior, but browsers still follow the old 301.
Why it breaks: 301 cached client side persistently.
Fix: change the URL itself (cache miss on new URL). Or wait for cache expiry.

**Pitfall 14: Redirect destination is 404.**
Symptom: 301 to a page that no longer exists.
Why it breaks: chain ends at 404; bad UX, lost SEO equity.
Fix: audit redirect destinations periodically; remove or update broken ones.

**Pitfall 15: HSTS preloaded but redirect target is HTTP.**
Symptom: certificate error or refused connection on first visit.
Why it breaks: HSTS forces HTTPS but redirect points to HTTP.
Fix: all redirect targets must be HTTPS.

---

## 22. DIAGNOSTIC COMMANDS

Reference of every command useful for 3xx investigation.

### Inspect redirect behavior

```bash
# Show redirect chain
curl -sIL https://example.com/old-url | grep -E "^HTTP|^Location:"

# Count hops
HOPS=$(curl -sIL https://example.com/old-url | grep -c "^HTTP")
echo "Hops: $((HOPS - 1))"  # subtract final 200

# Get just the redirect status and destination
curl -sI https://example.com/old-url | head -3
```

### Audit all redirects in a sitemap

```bash
# Find any URLs that redirect (should be none in a healthy sitemap)
SITEMAP=https://example.com/sitemap.xml

curl -s "$SITEMAP" | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | \
    while read url; do
        STATUS=$(curl -so /dev/null -w "%{http_code}" "$url")
        if [ "$STATUS" != "200" ]; then
            DEST=$(curl -sI "$url" | grep -i "^location:" | awk '{print $2}' | tr -d '\r')
            echo "$STATUS $url -> $DEST"
        fi
    done
```

### Find chains exceeding one hop

```bash
# Audit sitemap for chains
curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | \
    while read url; do
        HOPS=$(curl -sIL "$url" 2>/dev/null | grep -c "^HTTP")
        if [ "$HOPS" -gt 2 ]; then
            echo "CHAIN ($((HOPS - 1)) hops): $url"
        fi
    done
```

### Test method preservation

```bash
# 301: method changes (typical browser behavior)
curl -sIL -X POST -d "key=value" https://example.com/301-endpoint 2>&1 | \
    grep -E "^> POST|^> GET|^HTTP|^Location"
# Expect: POST becomes GET on the followup

# 308: method preserved
curl -sIL -X POST -d "key=value" https://example.com/308-endpoint 2>&1 | \
    grep -E "^> POST|^> GET|^HTTP|^Location"
# Expect: POST remains POST on the followup
```

### Test conditional GET (304)

```bash
# Get current ETag
ETAG=$(curl -sI https://example.com/style.css | grep -i "^etag:" | awk '{print $2}' | tr -d '\r')

# Verify 304 returned with matching ETag
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -3
# Expected: HTTP/2 304

# Verify body is empty
BYTES=$(curl -s -H "If-None-Match: $ETAG" https://example.com/style.css | wc -c)
echo "Body bytes: $BYTES (expected 0)"

# Verify 200 returned with stale ETag
curl -sI -H 'If-None-Match: "stale"' https://example.com/style.css | head -1
# Expected: HTTP/2 200
```

### Verify open redirect protection

```bash
# Test that /redirect?url=... rejects external URLs
curl -sI "https://example.com/redirect?url=https://evil.com" | head -3
# Expected: 400 Bad Request OR 200 with safe behavior, NOT 302 to evil.com

# Test path only redirect
curl -sI "https://example.com/login?return_to=/dashboard" | head -3
# Expected: passthrough OR 302 to /dashboard
curl -sI "https://example.com/login?return_to=https://evil.com" | head -3
# Expected: rejected or sanitized
```

### Server side investigation

```bash
# Find all return 3xx in nginx config
nginx -T 2>/dev/null | grep -E "return 30[0-9]+"

# Show all server_name patterns
nginx -T 2>/dev/null | grep "server_name"

# Find any 302 that might need conversion to 301
nginx -T 2>/dev/null | grep -B 5 "return 302" | grep -E "location|server_name"

# Apply changes
nginx -t && systemctl reload nginx
```

### Browser DevTools quick reference

In Chrome DevTools Network panel:

1. Enable "Preserve log" to see redirects across navigation.
2. The redirect chain shows as multiple request rows.
3. Click each request to see Status code and Location header.

The Network panel's "Doc" filter shows just HTML/redirect requests, hiding asset noise.

For checking HSTS interaction:

1. Navigate to `chrome://net-internals/#hsts`.
2. Query the domain to see HSTS state.
3. "Delete domain" if testing rollback.

---

## 23. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): `Last-Modified` and `ETag` pair with conditional GET; 304 is the conditional success.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): `Location` header is the redirect target; open redirect security pattern covered there is expanded in Section 15 here.
* [framework-http-security-headers.md](framework-http-security-headers.md): HSTS interaction (Section 14 here); HSTS forces HTTPS, making the HTTP to HTTPS redirect effectively permanent.
* [framework-http-request-headers.md](framework-http-request-headers.md): `If-Modified-Since` and `If-None-Match` request headers are what trigger 304 responses.
* [framework-http-2xx-status-codes.md](framework-http-2xx-status-codes.md): the previous status code framework; 201 Created with Location is analogous to 301 in that both use Location, but for different purposes.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. 301 redirect equity transfer is fundamental to site migrations.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including the 14 paying client subdomain pattern.
* [framework-http-4xx-status-codes.md] (next in series): 400, 401, 403, 404, 410, 422, 429.
* [framework-http-5xx-status-codes.md] (later in series): 500, 502, 503, 504.
* Google HTTP status codes documentation: https://developers.google.com/search/docs/crawling-indexing/http-network-errors
* Google Change of Address tool documentation: https://support.google.com/webmasters/answer/9370220
* MDN 301: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/301
* MDN 302: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/302
* MDN 303: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/303
* MDN 304: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/304
* MDN 307: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/307
* MDN 308: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/308
* RFC 9110 (HTTP Semantics): https://www.rfc-editor.org/rfc/rfc9110
* RFC 7538 (308 Permanent Redirect): https://www.rfc-editor.org/rfc/rfc7538

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Redirect decision matrix

```
Is the move permanent?
   |
  YES                           NO
   |                             |
   v                             v
Does method matter?            Does method matter?
(POST stays POST?)             (POST stays POST?)
   |                             |
  YES to 308                    YES to 307
   |                             |
  NO to 301                     NO to 302

Special cases:
* After POST, want GET response to 303
* Client has cached, version fresh to 304 (not really a redirect)
```

### SEO signal strength

| Redirect | SEO Signal | Use For |
|---|---|---|
| 301 | Strong (full equity transfer) | Permanent site/URL moves |
| 308 | Strong (same as 301) | Permanent API moves with method preservation |
| 302 | Weak (origin retained) | Login, A/B test, maintenance |
| 307 | Weak (same as 302) | Temporary API moves with method preservation |
| 303 | Not a canonical signal | Post/redirect/get pattern |
| 304 | Not a redirect | Conditional GET cache validation |

### Five rules to memorize

1. **301/308 for permanent**. 302/307 for temporary.
2. **307/308 preserve method**. 301/302 may change POST to GET.
3. **One hop only**. No chains.
4. **No open redirects**. Allow list or path only.
5. **HTTPS canonical in single hop from HTTP**. HSTS handles future visits.

### Five commands every operator should know

```bash
# 1. Check redirect status and destination
curl -sI https://example.com/old-url | head -3

# 2. Follow chain and count hops
curl -sIL https://example.com/old-url | grep -c "^HTTP"

# 3. Verify method preservation
curl -sIL -X POST https://example.com/api/v1/endpoint 2>&1 | grep -E "^> POST|^> GET|^HTTP"

# 4. Test conditional GET (304)
ETAG=$(curl -sI https://example.com/style.css | grep -i etag | awk '{print $2}' | tr -d '\r')
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -1

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. HTTP to canonical HTTPS is exactly one hop
HOPS=$(curl -sIL http://example.com/ | grep -c "^HTTP")
[ "$HOPS" = "2" ] && echo "OK: one hop" || echo "FAIL: $((HOPS - 1)) hops"

# 2. No 302 for permanent canonicalization
nginx -T 2>/dev/null | grep -B 5 "return 302" | grep -iE "https|www|canonical" && echo "WARN: possible misuse of 302"

# 3. Method preserved through 308
curl -sIL -X POST -d "data=test" https://example.com/api/v1/users 2>&1 | \
    grep -E "^> POST.*v2|^> GET.*v2" | head -1
# Expected: POST /api/v2/users (method preserved)
```

If all three pass AND no redirect chains exceed one hop, the 3xx layer is correctly wired.

---

End of framework-http-3xx-status-codes.md.
