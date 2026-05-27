# framework-http-2xx-status-codes.md

Comprehensive reference for the HTTP 2xx Success status codes that crawlers and clients encounter, with crawler reaction semantics, indexing implications, and nginx plus FastAPI implementation patterns. Covers `200 OK` (the only indexable success code), `201 Created` (REST creation), `202 Accepted` (async operations, grouped with 201 by Google), `204 No Content` (the silent success that Googlebot treats as soft 404), and `206 Partial Content` (range requests for streaming and resumable downloads). Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to the eight HTTP header frameworks (caching, content, SEO, security, performance, CORS, rate control, request headers), UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

**This is the first framework in the HTTP status codes series.** The eight previous frameworks documented response and request headers. This one and its siblings (3xx redirects, 4xx client errors, 5xx server errors) document what the status line itself says and how crawlers interpret it. Status codes and headers interlock: 304 Not Modified pairs with `If-Modified-Since` from the request headers framework; 410 Gone pairs with `X-Robots-Tag` for deindexing; 429 already covered in framework-http-rate-control-headers.md but the 5xx sibling will revisit.

Audience: humans configuring nginx and FastAPI to return the right status code for the right scenario, AI assistants generating server logic that crawlers understand correctly, SEO operators auditing indexing problems caused by status code misuse (especially soft 404s), and anyone troubleshooting "page returns 200 but does not get indexed", "video does not seek properly", or "DELETE endpoint returns wrong code".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The 2xx Mental Model (read this first)
5. 200 OK (the only indexable success code)
6. 201 Created (REST creation, Location header pairing)
7. 202 Accepted (asynchronous and queued operations)
8. 204 No Content (the silent success, soft 404 trap)
9. 206 Partial Content (range requests, streaming, resumable downloads)
10. The Soft 404 Trap (deep dive on the most ranking damaging 2xx pattern)
11. How Major Crawlers React To Each 2xx Code
12. How These Codes Interact With Each Other And With Headers
13. Asset Class And Use Case Recipes
14. Bubbles Nginx Reference Block (paste ready)
15. Audit Checklist (50+ items)
16. Common Pitfalls
17. Diagnostic Commands
18. Cross-References

---

## 1. DEFINITION

2xx status codes signal that the server received, understood, and accepted the request. The action requested was successful in some sense. The exact sense varies: 200 means a full response is enclosed; 201 means a new resource was created; 202 means the request was queued for later processing; 204 means no body is being returned; 206 means part of a larger resource is being returned in response to a Range request. Defined across RFC 9110 (HTTP Semantics).

For crawlers, 2xx codes split into three categories of indexability:

* **Indexable**: 200 OK. The content in the response body may be considered for indexing.
* **Wait and see**: 201 Created, 202 Accepted. Googlebot waits a limited time and then passes whatever it received to the indexing pipeline. The result is usually a soft 404 if the body is empty.
* **Not indexable**: 204 No Content, 206 Partial Content. Googlebot signals to the indexing pipeline that no body was received (204) or only a fragment was received (206). Search Console typically shows a soft 404 for 204 responses on URLs in the index.

The practical implication is one of the most ranking damaging patterns in HTTP: **a 200 OK with empty or near empty content is treated by Google as a soft 404**. The server says success; the indexer sees nothing useful; the URL gets flagged. Tracking and avoiding this pattern is core SEO hygiene for any Bubbles client site.

---

## 2. WHY IT MATTERS

Six independent pressures push correct 2xx status code usage from "default behavior" to "actively managed signal" in 2025 and forward.

**The soft 404 pattern silently breaks indexing.** A page that returns 200 with empty or thin content gets soft 404 flagged by Google. The flag does not prevent indexing in itself, but it strongly suggests the page should not be ranked. Bubbles client sites with dynamic category pages, search results pages, or "no products available" states are prime candidates. Joseph's earlier work removing 5,800 thin location pages from indexing addressed exactly this category of problem.

**REST API conventions depend on correct status codes.** A POST that creates a resource should return 201 Created with a `Location` header pointing to the new resource. A DELETE that succeeds should return 204 No Content. Returning 200 for both confuses API consumers and breaks tooling (Postman collections, OpenAPI validators, client SDKs).

**Video and audio streaming requires correct 206 handling.** HTML5 video players, podcast clients, and audio scrubbers all use HTTP Range requests for seeking. A server that returns 200 with the full file instead of 206 with the requested byte range produces broken seek behavior (the player rebuffers from the start instead of jumping). Pre 2026 mobile devices with bandwidth limits suffer most.

**Beacon and analytics endpoints belong on 204.** Endpoints that receive `sendBeacon()` data, RUM telemetry, or CSP violation reports should return 204. The body is irrelevant; the client only cares that the data was accepted. Returning 200 with an empty body works but wastes a few bytes per request.

**Conditional GET turns 200s into 304s.** A correctly configured server returns 200 with fresh content for first visits and 304 Not Modified (covered in framework-http-3xx-status-codes.md later in the series) for revalidation. This pair is one of the largest crawl budget savings available. Without it, every revisit gets a full 200 with the same content.

**Async operations need 202.** Endpoints that queue work for later processing (image processing, email sending, report generation) should return 202 Accepted with a `Location` or `Content-Location` header pointing to where the client can check status. Returning 200 with the final result blocks the client until the work is done; returning 200 with "queued" status leaves the client guessing about completion.

**Cost of getting it wrong.** Misconfigured 2xx codes produce silent SEO damage and confused API clients. Real examples:

* Bubbles client site had 5,800+ dynamically generated location pages with thin content. Each returned 200 OK with a paragraph of templated text. Google soft 404 flagged all of them. The site's overall quality signal degraded; rankings on the genuine pages dropped. Removing them via X-Robots-Tag noindex plus 410 Gone took weeks to recover.
* E commerce category page returns 200 with "no products available" when out of stock. Soft 404. Lost ranking for the category term.
* DELETE /api/users/123 returns 200 with `{"status": "ok"}`. Some client libraries treat it as containing the deleted user object and try to parse a non existent `id` field. Errors all over the client. Returning 204 with no body is correct.
* Video player on the marketing page rebuffers from the start every time the user seeks. Server is not returning 206 in response to Range requests. Investigation reveals the upstream framework does not handle Range; nginx does, but the file is being served by the upstream.
* Async report generation returns 200 immediately with "report queued". Client thinks the report is ready and tries to download from the Location URL. Gets 404 because the report has not been generated yet.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the five status codes (the four Joseph listed plus 202 because Google groups it with 201) gets the same six part treatment:

1. **What it means**: the canonical RFC definition plus practical implication.
2. **When to use it**: which scenarios warrant this specific code.
3. **How to return it in nginx and FastAPI**: paste ready config and code.
4. **How crawlers react**: indexing pipeline behavior per major crawler.
5. **How to verify**: curl commands and observation patterns.
6. **Common misuse and how to fix**: the typical wrong choices and their replacements.

The soft 404 trap gets its own dedicated section because it is the most ranking damaging pattern in this family. The crawler reaction table in Section 11 is the quick reference Joseph needs when auditing client sites.

---

## 4. THE 2xx MENTAL MODEL (READ THIS FIRST)

A request arrives, the server processes it, the response status code communicates what happened. For 2xx codes, the question is not whether the request succeeded but in what sense. Internalize the distinctions.

```
Request arrives at server
        |
        v
==================== PROCESSING ====================
        |
        v
What just happened?
   |
   |---> Existing resource was returned in full
   |     |
   |     v
   |     200 OK
   |     Body: the resource itself
   |     Indexable: YES (if content is substantive)
   |     Crawler: passes to indexing pipeline
   |
   |---> A new resource was just created
   |     |
   |     v
   |     201 Created
   |     Body: representation of new resource (or empty)
   |     Header: Location pointing to new resource
   |     Crawler: waits for content, may show soft 404 in Search Console
   |
   |---> Request accepted, will process asynchronously
   |     |
   |     v
   |     202 Accepted
   |     Body: status info (or empty)
   |     Header: Location pointing to status endpoint
   |     Crawler: waits for content, may show soft 404
   |
   |---> Success, no body to return
   |     |
   |     v
   |     204 No Content
   |     Body: MUST be empty
   |     Crawler: signals indexing pipeline that no content was received
   |     Search Console: typically shows soft 404 for indexed URLs
   |
   |---> Partial response to Range request
   |     |
   |     v
   |     206 Partial Content
   |     Body: requested byte range only
   |     Header: Content-Range describing the slice
   |     Crawler: not for indexing; this is mid stream content
```

Five rules govern the system:

1. **200 is for substantive content.** If the page has no content to display, do not return 200. Return 404, 410, or something else appropriate.
2. **201 needs a Location header.** Creating a resource without telling the client where it lives is unhelpful.
3. **204 must have no body.** Browsers and intermediaries enforce this. Returning a body with 204 produces inconsistent behavior.
4. **206 requires Content-Range.** The client needs to know which slice it received.
5. **The status code is a contract.** Clients and crawlers make decisions based on it. Wrong code, wrong decision, wrong outcome.

A correctly configured server returns 200 only for substantive content, uses 201 with Location for creation, uses 202 with Location for async work, uses 204 for body less success, and lets nginx handle 206 for Range requests automatically.

---

## 5. 200 OK (THE ONLY INDEXABLE SUCCESS CODE)

### 5.1 What It Means

200 OK is the standard success response with content in the body. The request was understood, the server processed it, and the result is enclosed. Defined in RFC 9110.

Per Google's official documentation: "200 (success): Google passes on the content to the indexing pipeline. The indexing systems may index the content, but that is not guaranteed."

The key phrase is "passes on the content". 200 is permission to consider the content; it is not a guarantee of indexing. Whether the content is actually indexed depends on quality, uniqueness, internal linking, and many other factors.

### 5.2 When To Use 200

* The resource exists and has substantive content to return.
* A successful GET, HEAD, or successful idempotent PUT/PATCH that returns the updated resource.
* A POST that returns the result of an operation (search, calculation, validation), not creating a resource.
* The default success state for almost any browser navigation.

### 5.3 When NOT To Use 200

* **The page exists but has no content.** Return 404 or 410 instead. Returning 200 produces a soft 404.
* **A resource was created.** Use 201 with Location.
* **The operation completed but there is no body to return.** Use 204.
* **Async processing started but is not complete.** Use 202 with status URL.
* **The page is intentionally empty (placeholder, coming soon, login required).** Return 401, 403, or 404 depending on the situation. A "coming soon" page that returns 200 with one paragraph of text will be soft 404 flagged.

### 5.4 The "Indexable" Qualifier

"Indexable" does not mean "will be indexed". A 200 response is indexable if:

* The content is substantive (not thin, not a placeholder).
* No `X-Robots-Tag: noindex` header is set (see framework-http-seo-headers.md).
* No `<meta name="robots" content="noindex">` tag is in the HTML.
* No robots.txt disallow is blocking the URL from being crawled at all.
* The content is not a near duplicate of another already indexed URL.

The minimum bar for indexability:

* Distinct, substantive content (more than a paragraph).
* A semantic HTML structure with H1, paragraphs, and meaningful internal links.
* Title and meta description that match the content.
* No noindex directive anywhere.

### 5.5 How To Return 200

**In nginx:**

```nginx
# 200 is the default for successful responses; no explicit directive needed
location / {
    try_files $uri $uri/ $uri.html =404;
    # If found, nginx returns 200; if not, 404
}

# Explicit 200 with custom body
location = /healthz {
    add_header Content-Type "application/json" always;
    return 200 '{"status": "ok"}';
}
```

**In FastAPI:**

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse, HTMLResponse

app = FastAPI()

@app.get("/")
async def index():
    # 200 is the default
    return {"data": "..."}

@app.get("/page", response_class=HTMLResponse)
async def page():
    return "<html><body>Substantive content here</body></html>"
```

### 5.6 How To Verify

```bash
# Confirm 200 returned
curl -sI https://example.com/ | head -1
# Expected: HTTP/2 200

# Verify body has content
SIZE=$(curl -s -o /dev/null -w "%{size_download}" https://example.com/)
echo "Body size: $SIZE bytes"
# Expected: substantive size (typical HTML page: 5000+ bytes)

# Verify no noindex
curl -sI https://example.com/ | grep -i x-robots-tag
# Expected: max-snippet etc; NOT noindex unless intentional

curl -s https://example.com/ | grep -i 'name="robots"'
# Expected: index, follow (or absent); NOT noindex
```

### 5.7 Crawler Reaction

Per Google's documentation:

> "Google passes on the content to the indexing pipeline. The indexing systems may index the content, but that is not guaranteed."

The Bubbles audit pattern: a 200 OK on a thin page is a soft 404 risk. Section 10 covers detection and mitigation.

### 5.8 How To Fix Common Breakage

**Case: marketing page returns 200 but is soft 404 flagged.**
The page has too little unique content. Either:
1. Add substantive, unique content (the right answer for valuable pages).
2. Return 410 Gone if the page should no longer exist.
3. Add `X-Robots-Tag: noindex` if the page must remain accessible to users but should not be in the index.

**Case: dynamic search results page with no results.**
Do not return 200 with "no results found". Instead:
1. Return 200 with related categories or popular alternatives (substantive content).
2. Return 404 if no related content exists.
3. Or add `X-Robots-Tag: noindex` to the empty results variant.

---

## 6. 201 CREATED (REST CREATION, LOCATION HEADER PAIRING)

### 6.1 What It Means

201 Created is returned by a POST (or PUT) that successfully creates a new resource. The response indicates the resource was created and the `Location` header points to it. Defined in RFC 9110.

```
POST /api/users HTTP/2
Content-Type: application/json

{"name": "Joseph", "email": "joseph@thatdeveloperguy.com"}

----

HTTP/2 201 Created
Location: /api/users/12847
Content-Type: application/json

{"id": 12847, "name": "Joseph", "email": "joseph@thatdeveloperguy.com"}
```

The body typically contains a representation of the newly created resource. The `Location` header is the canonical pointer.

### 6.2 When To Use 201

* POST to a collection that creates a new resource (`POST /api/users`).
* PUT to a URL that creates a new resource at that location (`PUT /api/users/12847` where the resource did not exist before).
* The result of any operation that brings a new resource into existence.

### 6.3 When NOT To Use 201

* The resource already existed. Use 200 (resource returned) or 204 (no body) for idempotent updates.
* The request was accepted for processing but the resource has not been created yet. Use 202.
* Nothing was created (read only operation). Use 200.

### 6.4 How To Return 201

**In nginx:**

201 is typically returned by the upstream, not nginx directly. Nginx passes it through:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    # Pass through 201 and Location header automatically
}
```

For static endpoints that always return 201 (rare):

```nginx
location = /api/always-create {
    add_header Location "/api/new-resource/abc" always;
    add_header Content-Type "application/json" always;
    return 201 '{"id": "abc"}';
}
```

**In FastAPI:**

```python
from fastapi import FastAPI, status
from fastapi.responses import JSONResponse

app = FastAPI()

@app.post("/api/users", status_code=status.HTTP_201_CREATED)
async def create_user(name: str, email: str):
    user_id = await db_create_user(name, email)
    return JSONResponse(
        status_code=201,
        content={"id": user_id, "name": name, "email": email},
        headers={"Location": f"/api/users/{user_id}"}
    )
```

Or with the `Response` parameter to set headers:

```python
from fastapi import Response

@app.post("/api/users", status_code=201)
async def create_user(name: str, email: str, response: Response):
    user_id = await db_create_user(name, email)
    response.headers["Location"] = f"/api/users/{user_id}"
    return {"id": user_id, "name": name, "email": email}
```

### 6.5 How To Verify

```bash
# Create a resource, observe 201 with Location
curl -sI -X POST https://api.example.com/users \
    -H "Content-Type: application/json" \
    -d '{"name": "test"}' | head -5

# Expected:
# HTTP/2 201
# location: /api/users/12847
# content-type: application/json

# Follow the Location to verify resource exists
LOCATION=$(curl -sI -X POST https://api.example.com/users \
    -H "Content-Type: application/json" \
    -d '{"name": "test"}' | grep -i "^location:" | awk '{print $2}' | tr -d '\r')

curl -sI "https://api.example.com$LOCATION" | head -1
# Expected: HTTP/2 200
```

### 6.6 Crawler Reaction

Per Google's documentation:

> "201 (created), 202 (accepted): Googlebot waits for the content for a limited time, then passes on whatever it received to the indexing pipeline. The timeout is user agent dependent."

In practice: crawlers never POST. They GET. A 201 response only happens during normal crawl when the URL itself returns 201 to a GET (extremely unusual). If you somehow return 201 to a GET request, Googlebot will wait for content and may flag soft 404 if nothing arrives.

For API only endpoints that return 201 to POST: crawlers do not see this. Add `X-Robots-Tag: noindex` to the entire `/api/` location anyway.

### 6.7 How To Fix Common Breakage

**Case: POST that creates a resource returns 200 instead of 201.**
Standard REST convention violation. Update the endpoint to return 201 with Location:

```python
@app.post("/api/users", status_code=201)  # was 200
async def create_user(...):
    ...
    return JSONResponse(
        status_code=201,
        content={...},
        headers={"Location": f"/api/users/{user_id}"}
    )
```

**Case: 201 returned without Location header.**
Add the Location header pointing to the new resource. API consumers depend on it.

---

## 7. 202 ACCEPTED (ASYNCHRONOUS AND QUEUED OPERATIONS)

### 7.1 What It Means

202 Accepted means the request has been received and queued for processing, but processing is not complete. The actual result will be available later. Defined in RFC 9110.

```
POST /api/reports HTTP/2
Content-Type: application/json

{"type": "monthly-sales", "month": "2026-05"}

----

HTTP/2 202 Accepted
Location: /api/reports/job-abc123/status

{"job_id": "abc123", "status": "queued"}
```

The `Location` header points to where the client can poll for completion. The response body typically includes a job ID and current status.

### 7.2 When To Use 202

* Long running operations: report generation, video transcoding, email campaign sending.
* Operations that involve external systems: payment processing, third party API calls with latency.
* Batch operations: bulk imports, batch deletions.
* Anything where blocking the client until completion is impractical.

### 7.3 When NOT To Use 202

* The operation is fast enough to complete synchronously. Use 200 or 201.
* The operation failed. Use 4xx or 5xx.
* The operation is idempotent and can return cached results immediately. Use 200.

### 7.4 The 202 Pattern (Status Polling)

The typical 202 flow:

1. Client POSTs `/api/reports` with parameters.
2. Server returns 202 with `Location: /api/reports/job-abc123/status` and `{"job_id": "abc123", "status": "queued"}`.
3. Client polls `/api/reports/job-abc123/status`:
   * While processing: 200 with `{"status": "processing", "progress": 45}`.
   * On completion: 200 with `{"status": "complete", "result_url": "/api/reports/job-abc123/result"}`.
   * On failure: 200 with `{"status": "failed", "error": "..."}` (or 500 for unexpected failures).
4. Client GETs the result URL to retrieve the actual output.

Alternative pattern (webhook based):

1. Client POSTs `/api/reports` with a webhook URL.
2. Server returns 202 with job ID.
3. Server processes asynchronously.
4. On completion, server POSTs to the webhook URL with the result.

### 7.5 How To Return 202

**In FastAPI:**

```python
from fastapi import FastAPI, BackgroundTasks
from fastapi.responses import JSONResponse
import uuid

app = FastAPI()

@app.post("/api/reports", status_code=202)
async def create_report(request: ReportRequest, background_tasks: BackgroundTasks):
    job_id = str(uuid.uuid4())

    # Queue the work
    background_tasks.add_task(generate_report, job_id, request)

    return JSONResponse(
        status_code=202,
        content={"job_id": job_id, "status": "queued"},
        headers={"Location": f"/api/reports/{job_id}/status"}
    )

@app.get("/api/reports/{job_id}/status")
async def report_status(job_id: str):
    status = await get_job_status(job_id)
    return {
        "job_id": job_id,
        "status": status["state"],
        "progress": status["progress"],
        "result_url": status.get("result_url"),
    }
```

For production, use a real job queue (Celery, RQ, Dramatiq) rather than BackgroundTasks for durability.

### 7.6 How To Verify

```bash
# Submit and observe 202
RESPONSE=$(curl -sI -X POST https://api.example.com/reports \
    -H "Content-Type: application/json" \
    -d '{"type": "test"}')

echo "$RESPONSE" | head -3
# Expected:
# HTTP/2 202
# location: /api/reports/job-xxx/status

# Poll the status URL
STATUS_URL=$(echo "$RESPONSE" | grep -i "^location:" | awk '{print $2}' | tr -d '\r')
curl -s "https://api.example.com$STATUS_URL"
# Expected: JSON with current status
```

### 7.7 Crawler Reaction

Same as 201: Googlebot waits a limited time, then passes whatever it received to the indexing pipeline. Crawlers do not POST so they never trigger 202 from typical endpoints.

For API endpoints that return 202, add `X-Robots-Tag: noindex` to the location.

---

## 8. 204 NO CONTENT (THE SILENT SUCCESS, SOFT 404 TRAP)

### 8.1 What It Means

204 No Content means the request was successful and there is intentionally no body to return. Defined in RFC 9110. The response MUST have no body; clients and intermediaries enforce this.

```
DELETE /api/users/12847 HTTP/2

----

HTTP/2 204 No Content
```

The status line is the entire response. No body. The client knows the operation succeeded.

### 8.2 When To Use 204

* Successful DELETE operations.
* Successful PUT or PATCH that updates a resource without returning the updated representation.
* CORS preflight responses (the `OPTIONS` handler from framework-http-cors-headers.md).
* Beacon endpoints (`navigator.sendBeacon()` targets, RUM telemetry, CSP violation reports).
* Health check endpoints that just need to confirm "alive" without metadata.
* "Like", "favorite", "follow" type actions that change state without needing to return the new state.

### 8.3 When NOT To Use 204

* The response should have a body. Use 200.
* The URL is a normal page that crawlers might encounter. Returning 204 for a navigable URL triggers soft 404.
* You need to return error information. Use 4xx with a body explaining the error.
* The body might be empty depending on the resource. Be consistent: always 200 with possibly empty body, or always 204. Mixing is confusing.

### 8.4 The Critical Crawler Behavior

Per Google's documentation:

> "204 (no content): Googlebot signals the indexing pipeline that it received no content. Search Console may show a soft 404 error in the site's Page Indexing report."

**This is a trap for content URLs.** A normal page URL that returns 204 will be flagged as soft 404. If the URL was previously indexed (returning 200 with content), changing it to return 204 effectively deindexes it via soft 404 over time.

For API endpoints and beacon endpoints, this is not a concern because crawlers do not crawl them (and you should `X-Robots-Tag: noindex` them anyway).

### 8.5 How To Return 204

**In nginx:**

```nginx
# OPTIONS preflight (from framework-http-cors-headers.md)
location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
        add_header Access-Control-Max-Age "86400" always;
        return 204;  # explicit 204
    }
    # ...
}

# Beacon endpoint
location = /beacon {
    return 204;
}
```

**In FastAPI:**

```python
from fastapi import FastAPI, Response, status

app = FastAPI()

@app.delete("/api/users/{user_id}", status_code=204)
async def delete_user(user_id: int):
    await db_delete_user(user_id)
    return Response(status_code=204)
    # Note: no body. FastAPI ensures Response with status 204 has empty body.

@app.post("/rum")
async def rum_beacon(request: Request):
    body = await request.body()
    # Process beacon data...
    return Response(status_code=204)
```

### 8.6 The 204 Cannot Have Body Rule

Per RFC 9110, 204 responses MUST NOT have a body. Browsers, proxies, and clients enforce this. If you accidentally include a body:

* Some clients ignore it (correct behavior per spec).
* Some clients fail to parse (treating the body as the start of the next response).
* nginx may strip the body or fail to send.

The rule is absolute: 204 means no body. If you have a body to return, use 200.

### 8.7 How To Verify

```bash
# Confirm 204 status and empty body
curl -sI -X DELETE https://api.example.com/users/12847 | head -1
# Expected: HTTP/2 204

# Verify body is empty
SIZE=$(curl -s -X DELETE https://api.example.com/users/12847 -o /dev/null -w "%{size_download}")
echo "Body size: $SIZE"
# Expected: 0

# Verify Content-Length is 0 (or absent)
curl -sI -X DELETE https://api.example.com/users/12847 | grep -i content-length
# Expected: content-length: 0 (or no Content-Length at all)

# Test that OPTIONS preflight returns 204
curl -sI -X OPTIONS -H "Origin: https://app.example.com" \
    -H "Access-Control-Request-Method: POST" \
    https://api.example.com/data | head -1
# Expected: HTTP/2 204
```

### 8.8 Crawler Reaction Per Operator

Per Google: signals indexing pipeline no content was received. Search Console may show soft 404 on Page Indexing report.

Bingbot: similar to Google (no content to index).

ClaudeBot, GPTBot, OAI-SearchBot: treat 204 as "no content available". For training crawlers, this means the URL is skipped. For search/retrieval crawlers, the cached representation (if any) is the most recent 200.

**The Bubbles audit pattern for 204:** any URL that returns 204 should either be an API endpoint with `X-Robots-Tag: noindex` OR should never appear in robots.txt, sitemap, or internal navigation. The combination of "204 + indexable URL" is always a misconfiguration.

### 8.9 How To Fix Common Breakage

**Case: page returns 204 instead of 200 because the upstream returned empty.**
The upstream returned an empty response but the wrapper sent 204 instead of 200. The fix depends on intent:
1. If the page should exist with content: fix the upstream to actually return content.
2. If the page should not exist: return 404 instead.
3. If the URL is a legitimate API endpoint: add `X-Robots-Tag: noindex` so it never gets indexed in the first place.

**Case: DELETE returns 200 with body, breaking REST conventions.**
Change to 204:

```python
@app.delete("/api/users/{user_id}", status_code=204)
async def delete_user(user_id: int):
    await db_delete_user(user_id)
    return Response(status_code=204)
```

API clients now get the standard "operation succeeded, no body" pattern.

**Case: CSP report endpoint returns 200 with empty body.**
Change to 204:

```python
@app.post("/csp-report")
async def csp_report(request: Request):
    body = await request.body()
    # log it
    return Response(status_code=204)
```

Saves a few bytes per request and matches the RFC intent.

---

## 9. 206 PARTIAL CONTENT (RANGE REQUESTS, STREAMING, RESUMABLE DOWNLOADS)

### 9.1 What It Means

206 Partial Content is returned when the server is fulfilling a Range request, returning only the specified byte range of the resource rather than the full content. Defined in RFC 9110.

```
GET /video.mp4 HTTP/2
Range: bytes=1024-4096

----

HTTP/2 206 Partial Content
Content-Type: video/mp4
Content-Range: bytes 1024-4096/2097152
Content-Length: 3073
Accept-Ranges: bytes
```

The client gets bytes 1024 through 4096 of a 2097152 byte resource. Used for:

* Video streaming (HTML5 video player seek behavior).
* Audio streaming and scrubbing.
* Resumable downloads (curl `-C` continues from where it left off).
* Large file downloads in chunks (parallel download tools).
* HTTP based RSS readers fetching only new content.

### 9.2 The Three Headers That Define 206

| Header | Direction | Purpose |
|---|---|---|
| `Range` | Request | Client specifies byte range it wants |
| `Content-Range` | Response | Server confirms range delivered |
| `Accept-Ranges` | Response | Server advertises range support |

```
Range: bytes=0-1023            (request first 1024 bytes)
Range: bytes=1024-             (request from byte 1024 to end)
Range: bytes=-1024             (request last 1024 bytes)
Range: bytes=0-1023,2048-4095  (request multiple ranges)

Content-Range: bytes 0-1023/47022    (delivered bytes 0 to 1023 of 47022 total)
Content-Range: bytes */47022          (used in 416 response, file size only)

Accept-Ranges: bytes           (server supports byte range requests)
Accept-Ranges: none            (server does NOT support range requests)
```

For multiple ranges in one request, the response uses `Content-Type: multipart/byteranges` with each range as a separate part.

### 9.3 The 416 Range Not Satisfiable Companion

If the requested range is invalid (start past end of file, end before start, etc), the server returns 416:

```
HTTP/2 416 Range Not Satisfiable
Content-Range: bytes */47022
```

The `*` in Content-Range indicates the range is unsatisfiable; the second number is the actual file size so the client can adjust.

### 9.4 When 206 Is Used Automatically

For static files, nginx returns 206 automatically when a Range request comes in:

```nginx
location /downloads/ {
    root /var/www/sites/example.com;
    # nginx handles Range requests automatically
    # Returns 206 when client sends Range header
    # Returns 200 when no Range header
}
```

No special configuration needed. nginx reads the requested byte range, returns 206 with the correct slice.

For dynamic responses, the upstream must handle Range parsing and 206 response building.

### 9.5 When To Use 206 Explicitly

* Streaming video served from object storage (S3, R2): the upstream must handle Range and proxy ranges through.
* Custom streaming endpoints in FastAPI: the upstream code handles Range.
* Audio waveform fetching: get the first chunk to compute the waveform, then full stream.

### 9.6 How To Implement 206 In FastAPI

```python
import os
from pathlib import Path
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import StreamingResponse, Response

app = FastAPI()

VIDEO_DIR = Path("/var/www/sites/example.com/videos")

@app.get("/api/video/{video_id}")
async def stream_video(video_id: str, request: Request):
    video_path = VIDEO_DIR / f"{video_id}.mp4"
    if not video_path.exists():
        raise HTTPException(status_code=404, detail="video not found")

    file_size = video_path.stat().st_size
    range_header = request.headers.get("range")

    # No Range: return full content (200)
    if not range_header:
        return StreamingResponse(
            video_stream(video_path, 0, file_size - 1),
            status_code=200,
            media_type="video/mp4",
            headers={
                "Accept-Ranges": "bytes",
                "Content-Length": str(file_size),
            }
        )

    # Parse Range header
    try:
        units, ranges = range_header.split("=", 1)
        if units != "bytes":
            raise ValueError("only byte ranges supported")
        start_str, end_str = ranges.split("-", 1)
        start = int(start_str) if start_str else 0
        end = int(end_str) if end_str else file_size - 1
    except (ValueError, IndexError):
        # Malformed Range: return 416
        return Response(
            status_code=416,
            headers={"Content-Range": f"bytes */{file_size}"}
        )

    # Validate range
    if start >= file_size or end >= file_size or start > end:
        return Response(
            status_code=416,
            headers={"Content-Range": f"bytes */{file_size}"}
        )

    chunk_size = end - start + 1

    return StreamingResponse(
        video_stream(video_path, start, end),
        status_code=206,
        media_type="video/mp4",
        headers={
            "Accept-Ranges": "bytes",
            "Content-Range": f"bytes {start}-{end}/{file_size}",
            "Content-Length": str(chunk_size),
        }
    )


def video_stream(path: Path, start: int, end: int):
    """Yield bytes from `start` to `end` inclusive."""
    chunk_size = 64 * 1024  # 64 KB chunks
    with open(path, "rb") as f:
        f.seek(start)
        remaining = end - start + 1
        while remaining > 0:
            data = f.read(min(chunk_size, remaining))
            if not data:
                break
            yield data
            remaining -= len(data)
```

### 9.7 How To Verify 206

```bash
# Request a specific byte range
curl -sI -H "Range: bytes=0-1023" https://example.com/video.mp4 | head -5
# Expected:
# HTTP/2 206
# accept-ranges: bytes
# content-range: bytes 0-1023/<total>
# content-length: 1024

# Request from middle
curl -sI -H "Range: bytes=1024-2047" https://example.com/video.mp4 | head -5

# Request invalid range (should 416)
curl -sI -H "Range: bytes=999999999-" https://example.com/video.mp4 | head -3
# Expected:
# HTTP/2 416
# content-range: bytes */<size>

# Verify partial download works
curl -s -H "Range: bytes=0-99" https://example.com/video.mp4 | wc -c
# Expected: 100 (one hundred bytes)

# Test resumable download (curl -C)
curl -O https://example.com/large-file.zip   # downloads, gets interrupted
curl -O -C - https://example.com/large-file.zip  # continues from where it stopped
```

### 9.8 Crawler Reaction To 206

Crawlers do not typically request byte ranges. When they encounter video or audio files, they:

* **Image and asset crawlers (Googlebot-Image, Googlebot-Video)**: may request Range for video metadata extraction (first chunk to read file headers).
* **Search crawlers (Googlebot, Bingbot)**: typically GET full files.
* **AI crawlers (ClaudeBot, GPTBot)**: typically GET full text resources; not optimized for video.

If a crawler does send Range and gets 206 back, no indexing implication. The crawler reassembles the full file or gives up.

### 9.9 Common Misuse And How To Fix

**Case: video player rebuffers from start every time user seeks.**
The video URL is not returning 206 in response to Range. Verify:

```bash
curl -sI -H "Range: bytes=0-1023" https://example.com/video.mp4 | head -3
```

If response is `HTTP/2 200` (instead of 206), the server is not handling Range. For static files served by nginx, this should not happen (nginx handles automatically). If the file is served by an upstream that does not handle Range, either:

1. Move the file to a static location served directly by nginx.
2. Implement Range handling in the upstream (Section 9.6).
3. Use the `X-Accel-Redirect` pattern to have nginx serve the file with Range support.

**Case: download manager cannot resume large file.**
The server does not advertise `Accept-Ranges: bytes`. nginx adds this automatically for static files. For dynamic responses, the upstream must add it.

**Case: 416 returned when range is valid.**
The range parsing logic in the upstream has a bug. Common: off by one errors with inclusive end byte. Range `bytes=0-1023` requests 1024 bytes (0 through 1023 inclusive).

---

## 10. THE SOFT 404 TRAP (DEEP DIVE ON THE MOST RANKING DAMAGING 2xx PATTERN)

### 10.1 What A Soft 404 Is

A soft 404 is Google's classification for any URL that returns a 2xx status code but contains effectively no content. The page exists per the server (200 OK), but the content does not justify treating it as a valid page. Google's documentation:

> "If the server responded with a 2xx status code, the content received in the response may be considered for indexing. If the content suggests an error, for example an empty page or an error message, Search Console will show a soft 404 error."

The classifier looks for:

* Very short or near empty body.
* Templates with placeholder content ("Coming soon", "Page under construction").
* Error messages embedded in 200 responses ("Sorry, no results found").
* Login walls that render nothing for unauthenticated requests.
* Dynamic pages that returned empty data.

### 10.2 Why Soft 404s Damage Ranking

A soft 404 is worse than a hard 404 for SEO:

* **Hard 404**: Google sees the URL does not exist, stops crawling it, removes from index quickly.
* **Soft 404**: Google sees the URL but flags it as low quality, keeps trying to crawl periodically, but the URL drags down the site's quality signal.

The aggregate effect: a site with many soft 404 URLs gets treated as a lower quality site overall. Genuine pages may rank worse because they are mixed with many soft 404 pages.

### 10.3 Common Soft 404 Patterns

**Pattern 1: dynamic location pages.**
A site has 5,800 URLs of the form `/services/<city>` for every city in a five state area. Each page contains the same template with just the city name swapped in. Thin content. All soft 404.

The fix: either generate substantive content for each (challenging at scale) or noindex+ 410 the pages that lack genuine value.

**Pattern 2: out of stock product pages.**
Ecommerce site has 200 product URLs. 30 are out of stock and show "Currently unavailable". Server returns 200. Soft 404.

The fix: keep the page indexable IF the product will return; add product description, related products, customer reviews. Soft 404 risk reduced. If the product is permanently gone, return 410.

**Pattern 3: empty search results.**
Site allows `/search?q=anything` URLs. Empty searches return 200 with "no results found". Soft 404 (and also gets indexed for random queries).

The fix: noindex search results pages site wide via `X-Robots-Tag: noindex`. Optionally return 404 for queries with zero results.

**Pattern 4: client side rendered pages.**
SPA returns a minimal HTML shell with a `<div id="app"></div>` and JavaScript that fetches and renders content. Googlebot's initial fetch sees the empty shell. Soft 404.

The fix: SSR (server side rendering) or SSG (static site generation). For Bubbles, this means generating static HTML at build time, not relying on client side JavaScript.

**Pattern 5: login walls.**
A page like `/account/settings` returns 200 with empty body for unauthenticated requests (redirecting via JavaScript to login). Soft 404 flagged.

The fix: return 401 Unauthorized or 403 Forbidden with appropriate body for unauthenticated requests. Or noindex the page.

### 10.4 Detecting Soft 404s

**In Google Search Console:**

1. Page Indexing report.
2. Filter for "Soft 404" status.
3. Review the URL list.
4. Click each URL to see why Google flagged it.

**Via crawler simulation:**

```bash
# Check a specific URL for soft 404 risk
URL=https://example.com/services/some-city
RESPONSE=$(curl -s "$URL")
SIZE=${#RESPONSE}
WORD_COUNT=$(echo "$RESPONSE" | wc -w)

echo "URL: $URL"
echo "Response size: $SIZE bytes"
echo "Word count: $WORD_COUNT"

if [ $WORD_COUNT -lt 100 ]; then
    echo "WARNING: thin content (under 100 words). Possible soft 404."
fi
```

**Via systematic audit:**

```bash
#!/bin/bash
# audit-thin-pages.sh: scan sitemap for thin content URLs

SITE=https://example.com
SITEMAP=$SITE/sitemap.xml

curl -s "$SITEMAP" | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | while read url; do
    SIZE=$(curl -so /dev/null -w "%{size_download}" "$url")
    if [ "$SIZE" -lt 5000 ]; then
        echo "THIN: $url ($SIZE bytes)"
    fi
done
```

A page under 5 KB of HTML is suspicious. Most substantive pages are 10 KB or more.

### 10.5 The Bubbles Fix Pattern (Joseph's Previous Work)

Joseph's prior infrastructure overhaul addressed exactly this problem: 5,800+ thin location pages were noindex/410 cleaned across the client fleet. The systematic approach:

1. **Identify**: GSC Page Indexing report filtered to Soft 404; cross reference with sitemap and internal links.
2. **Classify**: keep, improve, noindex, or 410?
   * Keep + improve: high value pages that need more content.
   * Noindex: low value pages users might still need (login, account, internal tools).
   * 410: pages that should never have existed and have no remaining purpose.
3. **Execute via nginx**:
   ```nginx
   # Noindex tier
   location ~* ^/services/(very-specific-city|another-city)/ {
       add_header X-Robots-Tag "noindex, follow" always;
       try_files $uri $uri/ $uri.html =404;
   }

   # 410 tier (permanent removal)
   location ~* ^/old-removed-section/ {
       return 410;
   }
   ```
4. **Submit removal**: GSC URL Removals tool for the most critical noindex URLs (faster than waiting for recrawl).
5. **Monitor**: GSC Page Indexing report week over week. Soft 404 count should drop, "Indexed" count should hold steady or grow for valuable pages.

### 10.6 The Prevention Pattern

For new Bubbles client sites:

1. **No URLs without substantive content.** Before publishing a page, ensure it has at least 300 words of unique, useful content.
2. **Dynamic pages get noindex by default.** Search results, filters, sort orders all start with `X-Robots-Tag: noindex` and only get indexability when the canonical version is identified.
3. **Empty states return 404 or 410.** Out of stock products, expired events, removed pages all return appropriate 4xx codes.
4. **Server side rendering for critical content.** Never rely on client side JavaScript to populate the body for indexable pages.
5. **Audit at deployment.** Run a thin content scan before launching; flag any pages under 5 KB for review.

This was already baked into the SEO BUILD REFERENCE v2.4 process.

---

## 11. HOW MAJOR CRAWLERS REACT TO EACH 2XX CODE

Quick reference table for auditing.

| Code | Googlebot | Bingbot | ClaudeBot | GPTBot | OAI-SearchBot | PerplexityBot |
|---|---|---|---|---|---|---|
| 200 | Indexable (subject to quality) | Indexable | Considered for training | Considered for training | Considered for retrieval | Considered for retrieval |
| 201 | Wait for content, may soft 404 | Wait briefly | Treated as 200 | Treated as 200 | Treated as 200 | Treated as 200 |
| 202 | Wait for content, may soft 404 | Wait briefly | Treated as 200 | Treated as 200 | Treated as 200 | Treated as 200 |
| 204 | Soft 404 in Search Console | No content to index | Skip | Skip | Cached representation retained | Cached representation retained |
| 206 | Not for indexing (partial) | Not for indexing | N/A (text crawlers) | N/A | N/A | N/A |

**Key implications for SEO:**

* Only 200 with substantive content gets indexed.
* 201, 202, 204 on normal page URLs is always a misconfiguration for SEO.
* 206 is for media; not relevant to typical SEO.

---

## 12. HOW THESE CODES INTERACT WITH EACH OTHER AND WITH HEADERS

Several specific interactions matter.

### 12.1 200 Plus X-Robots-Tag: noindex

A page can return 200 OK but be excluded from indexing via `X-Robots-Tag: noindex`. This is preferred over 204 (which causes soft 404) for pages that should be reachable but not indexed:

```
HTTP/2 200 OK
X-Robots-Tag: noindex
Content-Type: text/html

<html>...login page or internal tool...</html>
```

### 12.2 201 Plus Location

201 without Location is incomplete. Always include the canonical URL of the created resource.

### 12.3 202 Plus Location Plus Polling

202 should point to a status URL. The client polls that URL until completion. The status URL returns 200 with current state until the operation completes.

### 12.4 204 Plus Cache-Control

For body less responses to mutating operations (DELETE, PUT), set `Cache-Control: no-store` to ensure intermediaries do not cache and serve the 204 to other requests.

```python
@app.delete("/api/users/{user_id}", status_code=204)
async def delete_user(user_id: int):
    await db_delete_user(user_id)
    return Response(
        status_code=204,
        headers={"Cache-Control": "no-store"}
    )
```

### 12.5 206 Plus Cache-Control

Partial responses can be cached, but caches must understand Range. Most browser caches do; some intermediate proxies do not. For static files, `Cache-Control: public, max-age=2592000` works fine with 206 because nginx handles caching correctly.

For dynamic 206 responses, include `Vary: Range` is not standard; range caching is handled by the cache implementation.

### 12.6 206 Plus ETag

Range requests can use conditional headers. The `If-Range` header lets the client say "only serve this range if the resource has not changed since I last fetched":

```
GET /video.mp4 HTTP/2
Range: bytes=1024-4096
If-Range: "abc123"
```

If the ETag still matches, server returns 206. If it changed, server returns 200 with the full content (the cached partial is now invalid).

### 12.7 The Status Code Plus Content-Type Coupling

For 200, 201, 202: include Content-Type matching the body. For 204: no body so Content-Type is irrelevant (though servers may include it harmlessly). For 206: Content-Type matches the requested range (e.g., `video/mp4`).

For multi range 206: `Content-Type: multipart/byteranges; boundary=<unique>`.

---

## 13. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 13.1 Standard substantive page (200)

```nginx
location / {
    try_files $uri $uri/ $uri.html =404;
    # 200 returned automatically if file exists
    # nginx adds appropriate headers
}
```

```python
@app.get("/", response_class=HTMLResponse)
async def home():
    return generate_substantive_html()  # 200 default
```

### 13.2 REST resource creation (201 with Location)

```python
@app.post("/api/articles", status_code=201)
async def create_article(article: Article, response: Response):
    new_id = await db_create_article(article)
    response.headers["Location"] = f"/api/articles/{new_id}"
    return {"id": new_id, **article.dict()}
```

### 13.3 Async report generation (202 with polling URL)

```python
@app.post("/api/reports", status_code=202)
async def create_report(req: ReportRequest, background_tasks: BackgroundTasks):
    job_id = str(uuid.uuid4())
    await queue_job(job_id, req)
    return JSONResponse(
        status_code=202,
        content={"job_id": job_id, "status": "queued"},
        headers={"Location": f"/api/reports/{job_id}/status"}
    )

@app.get("/api/reports/{job_id}/status")
async def report_status(job_id: str):
    status = await get_job_status(job_id)
    if status["state"] == "complete":
        return {
            "job_id": job_id,
            "status": "complete",
            "result_url": status["result_url"]
        }
    return {"job_id": job_id, "status": status["state"], "progress": status["progress"]}
```

### 13.4 DELETE returning 204

```python
@app.delete("/api/users/{user_id}", status_code=204)
async def delete_user(user_id: int):
    deleted = await db_delete_user(user_id)
    if not deleted:
        raise HTTPException(status_code=404)
    return Response(status_code=204, headers={"Cache-Control": "no-store"})
```

### 13.5 Beacon endpoint (204)

```nginx
location = /beacon {
    proxy_pass http://127.0.0.1:9090;
    proxy_set_header X-Real-IP $remote_addr;
    # Upstream returns 204
}
```

```python
@app.post("/beacon")
async def beacon(request: Request):
    body = await request.body()
    # Process beacon data
    log_beacon_event(body)
    return Response(status_code=204)
```

### 13.6 CSP report endpoint (204)

```python
@app.post("/csp-report")
async def csp_report(request: Request):
    try:
        report = await request.json()
        logger.warning(f"CSP violation: {report}")
    except Exception:
        pass  # never fail a report
    return Response(status_code=204)
```

### 13.7 OPTIONS preflight (204)

```nginx
if ($request_method = OPTIONS) {
    add_header Access-Control-Allow-Origin $cors_origin always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
    add_header Access-Control-Max-Age "86400" always;
    add_header Vary "Origin" always;
    return 204;
}
```

### 13.8 Video streaming with 206 (static file via nginx)

```nginx
location /videos/ {
    root /var/www/sites/example.com;
    # nginx handles Range automatically
    # Returns 206 for Range requests, 200 for full requests

    # Cache video files long term
    add_header Cache-Control "public, max-age=2592000" always;
    add_header Accept-Ranges "bytes" always;
}
```

### 13.9 Video streaming with 206 from upstream (FastAPI handler)

See Section 9.6 for the complete implementation.

### 13.10 Health check (200 with minimal body)

```nginx
location = /healthz {
    access_log off;  # do not log health checks
    add_header Content-Type "application/json" always;
    return 200 '{"status": "ok"}';
}
```

200 with body (not 204) so monitoring tools can verify response content beyond just status code.

### 13.11 Login wall (401 not 204 not 200)

```python
@app.get("/account/{path:path}")
async def account(request: Request, path: str):
    user = await get_current_user(request)
    if not user:
        # NOT 200 with login redirect
        # NOT 204 (soft 404)
        # 401 Unauthorized with redirect info
        return JSONResponse(
            status_code=401,
            content={"error": "auth required", "login_url": "/login"},
            headers={"WWW-Authenticate": 'Bearer realm="account"'}
        )
    # ... show account page
```

This returns 401 which crawlers correctly understand as "this requires auth, do not index".

### 13.12 Out of stock product (multiple correct answers)

Option A: keep page indexable with substantive content:

```python
@app.get("/products/{product_id}")
async def product(product_id: str):
    product = await get_product(product_id)
    if not product:
        raise HTTPException(status_code=404)

    # Even if out of stock, return full product page with:
    # - Description, images, reviews
    # - Related products
    # - "Notify when available" form
    # - Recently viewed
    return HTMLResponse(render_product(product))  # 200 with substantive content
```

Option B: noindex the out of stock variant:

```python
if not product.in_stock:
    return HTMLResponse(
        render_product(product),
        headers={"X-Robots-Tag": "noindex"}
    )
```

Option C: 410 if product is permanently discontinued:

```python
if product.discontinued:
    raise HTTPException(status_code=410)
```

Each is correct for different scenarios. Returning 200 with "out of stock" message and no other content is the soft 404 trap.

---

## 14. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete 2xx aware configuration for a Bubbles client site.

```nginx
# /etc/nginx/sites-available/example.com

server {
    listen 443 ssl;
    listen 443 quic;
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

    # ===== STATIC ASSETS (200 with cache, automatic Range for videos) =====
    location ~* \.(css|js|woff2|jpg|jpeg|png|webp|avif|svg)$ {
        # 200 default; nginx adds ETag, Last-Modified for conditional GET (304)
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        add_header Accept-Ranges "bytes" always;
    }

    location ~* \.(mp4|webm|mp3|ogg|m4a)$ {
        # Range request support automatic
        # 200 for full requests, 206 for Range requests
        add_header Cache-Control "public, max-age=2592000" always;
        add_header Accept-Ranges "bytes" always;
    }

    # ===== HTML PAGES (200 with no special handling) =====
    location ~* \.html$ {
        # 200 default; substantive content expected
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
    }

    # ===== HEALTH CHECK (200 with json) =====
    location = /healthz {
        access_log off;
        add_header Content-Type "application/json" always;
        return 200 '{"status": "ok"}';
    }

    # ===== BEACON (204) =====
    location = /beacon {
        access_log off;
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header X-Real-IP $remote_addr;
        # Upstream returns 204
    }

    # ===== CSP REPORT (204) =====
    location = /csp-report {
        proxy_pass http://127.0.0.1:9090;
        # Upstream returns 204
    }

    # ===== RUM BEACON (204) =====
    location = /rum {
        access_log off;
        proxy_pass http://127.0.0.1:9090;
        client_max_body_size 16k;
    }

    # ===== API (FastAPI sidecar returns 200/201/202/204 as appropriate) =====
    location /api/ {
        # OPTIONS preflight returns 204
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
            add_header Access-Control-Max-Age "86400" always;
            add_header Vary "Origin" always;
            return 204;
        }

        # API endpoints noindex regardless of status
        add_header X-Robots-Tag "noindex, nofollow" always;

        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ===== ROOT (200 with substantive content) =====
    location / {
        try_files $uri $uri/ $uri.html =404;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify each pattern:

```bash
# 200 with substantive content
curl -sI https://example.com/ | head -1
curl -s -o /dev/null -w "Size: %{size_download} bytes\n" https://example.com/

# 200 from health check
curl -sI https://example.com/healthz | head -1

# 204 from OPTIONS preflight
curl -sI -X OPTIONS -H "Origin: https://app.example.com" \
    -H "Access-Control-Request-Method: POST" \
    https://example.com/api/data | head -1
# Expected: HTTP/2 204

# 206 from Range request on video
curl -sI -H "Range: bytes=0-1023" https://example.com/videos/hero.mp4 | head -3
# Expected: HTTP/2 206, content-range header

# 201 from POST to API (if upstream supports it)
curl -sI -X POST https://example.com/api/items \
    -H "Content-Type: application/json" \
    -d '{"name": "test"}' | head -3
# Expected (for a creation endpoint): HTTP/2 201, location header
```

---

## 15. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### 200 OK

1. [ ] Every URL that returns 200 has substantive content (not thin, not placeholder).
2. [ ] No URL returns 200 with "no results found" or "out of stock" as the only content.
3. [ ] Soft 404 count in GSC Page Indexing is monitored monthly.
4. [ ] All indexable URLs have at least 300 words of unique content.
5. [ ] Client side rendered URLs have server side rendered fallback for crawlers.
6. [ ] No URLs return 200 with empty body for unauthenticated requests (use 401 instead).
7. [ ] Login pages and account pages have X-Robots-Tag: noindex if reachable.
8. [ ] No login walls hidden behind 200 with JavaScript redirect.

### 201 Created

9. [ ] All POST endpoints that create resources return 201 (not 200).
10. [ ] Every 201 response includes a Location header pointing to the new resource.
11. [ ] The Location URL actually exists and returns 200 when fetched.
12. [ ] Response body is the representation of the new resource (or empty if minimal).
13. [ ] API endpoints noindex via X-Robots-Tag (covered in /api/ location).

### 202 Accepted

14. [ ] Long running operations return 202 (not blocking 200).
15. [ ] Every 202 response includes Location pointing to status URL.
16. [ ] Status URL responds with current state (queued, processing, complete, failed).
17. [ ] Status URL also returns final result location when complete.
18. [ ] Job queue is durable (Redis, RQ, Celery, not just BackgroundTasks).

### 204 No Content

19. [ ] DELETE endpoints return 204 (not 200 with empty body).
20. [ ] OPTIONS preflight returns 204.
21. [ ] Beacon endpoints (/beacon, /rum, /csp-report) return 204.
22. [ ] 204 responses have empty body (no JSON, no HTML).
23. [ ] No 204 responses on URLs that crawlers might index.
24. [ ] 204 responses on mutating endpoints include Cache-Control: no-store.

### 206 Partial Content

25. [ ] Video files served from static location (nginx auto Range support).
26. [ ] Audio files served from static location (same).
27. [ ] Large download files have Accept-Ranges: bytes header.
28. [ ] curl -H "Range: bytes=0-1023" returns 206 on streamable assets.
29. [ ] Invalid ranges return 416 Range Not Satisfiable.
30. [ ] Content-Range header present and correct on 206 responses.
31. [ ] HTML5 video player seek works without rebuffering from start.

### Soft 404 prevention

32. [ ] Thin pages identified via GSC Page Indexing > Soft 404 list.
33. [ ] Thin pages classified: improve, noindex, or 410.
34. [ ] Noindex tier configured in nginx for low value but accessible URLs.
35. [ ] 410 tier configured for permanently removed URLs.
36. [ ] Dynamic search results pages noindex by default.
37. [ ] Filter and sort parameter URLs noindex (or canonical to base URL).
38. [ ] Empty category pages return 404 or 410, not 200 with "no items".

### Crawler interaction

39. [ ] Bubbles crawler analytics distinguish 200, 204, etc.
40. [ ] Daily report includes any 204 traffic from crawlers (should be near zero).
41. [ ] Any 5xx in crawler logs investigated (covered in framework-http-5xx-status-codes.md when built).

### Cross cutting

42. [ ] nginx -t passes without warnings.
43. [ ] nginx -T | grep "return " shows expected status codes.
44. [ ] curl tests against major URL patterns return expected status codes.
45. [ ] FastAPI sidecar status_code parameter set explicitly where appropriate (not defaulting to 200).
46. [ ] No accidental fall through to 200 in error paths.
47. [ ] Error handlers in FastAPI return correct status (4xx or 5xx, not 200 with error body).
48. [ ] Healthcheck endpoint returns 200 with body for monitoring tools.
49. [ ] Logging captures status code (already in standard log format).
50. [ ] Monitoring alerts on unexpected 2xx mix (sudden spike in 204 or 206 on crawled URLs).

A site that passes all 50 has correctly configured 2xx status code usage for SEO, API conventions, and streaming.

---

## 16. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: 200 OK for "no results found" page.**
Symptom: empty search results pages flagged as soft 404 in GSC.
Why it breaks: 200 with thin content = soft 404 by Google's classifier.
Fix: noindex empty results, OR show related categories/popular items as substantive content, OR return 404.

**Pitfall 2: 204 returned for normal page URL.**
Symptom: previously indexed pages drop from index after change to 204.
Why it breaks: Google explicitly treats 204 on a URL as "no content to index".
Fix: only use 204 on API endpoints, beacons, OPTIONS preflight. Never on navigable pages.

**Pitfall 3: 201 without Location header.**
Symptom: API client cannot find the resource just created.
Why it breaks: REST convention violation; client expects Location.
Fix: always set `Location: /api/resource/<id>` on 201 responses.

**Pitfall 4: 200 returned with `{"error": "..."}` body.**
Symptom: API client treats failed request as success.
Why it breaks: status code is the canonical signal; body is metadata.
Fix: return 4xx or 5xx for errors, not 200 with error body.

**Pitfall 5: 200 returned for "coming soon" placeholder page.**
Symptom: Google indexes the placeholder, ranks it generically.
Why it breaks: 200 with thin content = soft 404 risk or low quality signal.
Fix: noindex the placeholder, or return 503 with Retry-After (more honest).

**Pitfall 6: DELETE returns 200 with deleted object body.**
Symptom: API client SDK fails to parse the response, expecting void.
Why it breaks: REST convention says DELETE returns 204 (or 200 with operation result, but never the deleted object).
Fix: return 204 (no body) for successful DELETE.

**Pitfall 7: 204 with body sneaked in.**
Symptom: some clients fail to parse subsequent responses; pipeline desync.
Why it breaks: RFC says 204 MUST have empty body.
Fix: ensure server returns no body; use Response(status_code=204) in FastAPI.

**Pitfall 8: video player rebuffers from start every seek.**
Symptom: poor video UX, high bandwidth costs.
Why it breaks: server returning 200 with full file instead of 206 for Range.
Fix: serve video from static nginx location (auto Range support), or implement Range in upstream.

**Pitfall 9: 200 returned for client side rendered SPA shell.**
Symptom: SPAs flagged as soft 404 across Page Indexing report.
Why it breaks: Googlebot sees `<div id="app"></div>` and nothing else.
Fix: server side render or static generate for indexable pages. SPA pattern stays for app only routes.

**Pitfall 10: 202 without status URL.**
Symptom: client has no way to know when async operation completes.
Why it breaks: client must poll something; without Location they have nowhere to look.
Fix: include Location pointing to status endpoint.

**Pitfall 11: API endpoint returning 200 to crawlers (no noindex).**
Symptom: API endpoints (with JSON bodies) showing in search results.
Why it breaks: 200 plus indexable URL plus no noindex = indexable.
Fix: add `X-Robots-Tag: noindex, nofollow` to /api/ location.

**Pitfall 12: 206 returned without Content-Range header.**
Symptom: clients misinterpret the range delivered.
Why it breaks: spec requires Content-Range on 206.
Fix: always include Content-Range with start, end, and total size.

**Pitfall 13: 200 returned for unauthenticated request to private URL.**
Symptom: empty or minimal body indexed as soft 404; users redirected via JS.
Why it breaks: should be 401 (or 403) with appropriate body.
Fix: return 401 with WWW-Authenticate header for unauthenticated requests.

**Pitfall 14: Range request returns 200 with full file.**
Symptom: streaming and resumable downloads do not work; bandwidth wasted.
Why it breaks: server does not parse Range header; returns full content.
Fix: for static files use nginx (handles Range automatically); for upstream implement Range handling.

**Pitfall 15: 200 with empty JSON body returned where 204 should be used.**
Symptom: API client unnecessarily processes empty object.
Why it breaks: 200 implies meaningful body; 204 means body intentionally absent.
Fix: use 204 for operations that have no meaningful return value.

---

## 17. DIAGNOSTIC COMMANDS

Reference of every command useful for 2xx status code investigation.

### Inspect status codes across a site

```bash
# Status code summary for top URLs
SITE=https://example.com
for path in / /about /services /blog /contact /api/health; do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "$SITE$path")
    SIZE=$(curl -so /dev/null -w "%{size_download}" "$SITE$path")
    echo "$path: $STATUS ($SIZE bytes)"
done
```

### Detect thin content (soft 404 risk)

```bash
# Scan sitemap for thin pages
curl -s https://example.com/sitemap.xml | \
    grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | \
    while read url; do
        SIZE=$(curl -so /dev/null -w "%{size_download}" "$url")
        if [ "$SIZE" -lt 5000 ]; then
            echo "THIN: $url ($SIZE bytes)"
        fi
    done

# Count words in a specific page
URL=https://example.com/some-page
WORDS=$(curl -s "$URL" | sed 's/<[^>]*>//g' | wc -w)
echo "$URL: $WORDS words"
# Under 300 words is soft 404 risk
```

### Verify 201 with Location

```bash
RESPONSE=$(curl -sI -X POST https://api.example.com/items \
    -H "Content-Type: application/json" \
    -d '{"name": "test"}')

STATUS=$(echo "$RESPONSE" | head -1 | awk '{print $2}')
LOCATION=$(echo "$RESPONSE" | grep -i "^location:" | awk '{print $2}' | tr -d '\r')

echo "Status: $STATUS"
echo "Location: $LOCATION"

# Verify Location resource exists
if [ -n "$LOCATION" ]; then
    FOLLOWUP=$(curl -so /dev/null -w "%{http_code}" "https://api.example.com$LOCATION")
    echo "Location returns: $FOLLOWUP"
fi
```

### Verify 204 has empty body

```bash
# DELETE should return 204 with no body
RESPONSE=$(curl -sI -X DELETE https://api.example.com/users/test)
STATUS=$(echo "$RESPONSE" | head -1 | awk '{print $2}')
CONTENT_LENGTH=$(echo "$RESPONSE" | grep -i "^content-length:" | awk '{print $2}' | tr -d '\r')

echo "Status: $STATUS"
echo "Content-Length: ${CONTENT_LENGTH:-(not set)}"
echo "Expected: 204 with content-length 0 or absent"

# Also test that no body comes back
BODY_SIZE=$(curl -s -X DELETE https://api.example.com/users/test -o /dev/null -w "%{size_download}")
echo "Body size: $BODY_SIZE bytes"
echo "Expected: 0"
```

### Verify 206 Range support

```bash
# Test that Range requests get 206
URL=https://example.com/video.mp4

echo "=== Full request ==="
curl -sI "$URL" | head -3
# Expected: HTTP/2 200, accept-ranges: bytes

echo "=== Range request (first 1KB) ==="
curl -sI -H "Range: bytes=0-1023" "$URL" | head -4
# Expected: HTTP/2 206, content-range: bytes 0-1023/<total>

echo "=== Range request (last 1KB) ==="
curl -sI -H "Range: bytes=-1024" "$URL" | head -4

echo "=== Invalid range ==="
curl -sI -H "Range: bytes=999999999-" "$URL" | head -3
# Expected: HTTP/2 416

# Verify partial bytes downloaded
BYTES=$(curl -s -H "Range: bytes=0-99" "$URL" | wc -c)
echo "Bytes downloaded: $BYTES (expected 100)"
```

### Audit for soft 404 patterns

```bash
# Find URLs returning 200 but containing common soft 404 phrases
URL=https://example.com
SUSPICIOUS_PHRASES=("no results found" "out of stock" "page not found" "coming soon" "under construction")

for phrase in "${SUSPICIOUS_PHRASES[@]}"; do
    echo "=== Checking for '$phrase' ==="
    # Sample 20 URLs from sitemap
    curl -s "$URL/sitemap.xml" | grep -oE "<loc>[^<]+</loc>" | head -20 | \
        sed 's/<[^>]*>//g' | while read u; do
            if curl -s "$u" | grep -qi "$phrase"; then
                STATUS=$(curl -so /dev/null -w "%{http_code}" "$u")
                if [ "$STATUS" = "200" ]; then
                    echo "SOFT 404 RISK: $u contains '$phrase' (status 200)"
                fi
            fi
        done
done
```

### Status code distribution from logs

```bash
# Top status codes in the access log today
sudo awk -v date="$(date '+%d/%b/%Y')" '$0 ~ date {print $9}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -10

# 2xx breakdown
sudo grep -oE '" [0-9]{3} ' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -15

# Soft 404 candidates: 200 with low response size
sudo awk '$9 == 200 && $10 < 5000 {print $7, $10}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -20
```

### Server side investigation

```bash
# Find explicit return statements in nginx config
nginx -T 2>/dev/null | grep -E "return [0-9]+"

# Find status_code in FastAPI source
grep -rn "status_code" /opt/bubbles/api/

# Check what GSC reports for the domain
# (Manual: visit Google Search Console > Page Indexing > Soft 404)

# Apply changes
nginx -t && systemctl reload nginx
```

---

## 18. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): `Last-Modified` and `ETag` pair with conditional GET requests; the 304 Not Modified response (in framework-http-3xx-status-codes.md when built) is the conditional success.
* [framework-http-content-headers.md](framework-http-content-headers.md): Content-Type for 200/201/202 responses; Content-Range for 206; Content-Length 0 (or absent) for 204.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): `X-Robots-Tag` controls indexability of 200 responses; 410 Gone (covered in 4xx framework) is the SEO friendly removal.
* [framework-http-security-headers.md](framework-http-security-headers.md): security headers apply to all 2xx responses; CSP report endpoint returns 204.
* [framework-http-performance-headers.md](framework-http-performance-headers.md): Server-Timing on 2xx responses for backend observability.
* [framework-http-cors-headers.md](framework-http-cors-headers.md): OPTIONS preflight returns 204 with CORS headers.
* [framework-http-rate-control-headers.md](framework-http-rate-control-headers.md): 429 and 503 (covered there) are the 4xx and 5xx companions to 2xx success; Retry-After applies to those.
* [framework-http-request-headers.md](framework-http-request-headers.md): `If-Modified-Since` and `If-None-Match` request headers determine when 200 becomes 304; `Range` request header determines when 200 becomes 206.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Soft 404s degrade overall site quality signals.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including thin content prevention.
* [framework-http-3xx-status-codes.md] (next in series): 301, 302, 303, 304, 307, 308 redirects.
* [framework-http-4xx-status-codes.md] (later in series): 400, 401, 403, 404, 410, 429.
* [framework-http-5xx-status-codes.md] (later in series): 500, 502, 503, 504.
* Google HTTP status codes documentation: https://developers.google.com/search/docs/crawling-indexing/http-network-errors
* MDN 200 OK: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/200
* MDN 201 Created: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/201
* MDN 202 Accepted: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/202
* MDN 204 No Content: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/204
* MDN 206 Partial Content: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/206
* RFC 9110 (HTTP Semantics): https://www.rfc-editor.org/rfc/rfc9110

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Status code decision tree

```
Operation succeeded. What now?
   |
   |---> Returning a substantive resource (page, JSON data) ........... 200 OK
   |
   |---> Just created a new resource (POST /api/users) ............... 201 Created (+ Location)
   |
   |---> Operation queued, will complete later (long running job) ..... 202 Accepted (+ Location for status)
   |
   |---> Operation succeeded, no body to return (DELETE, OPTIONS) ..... 204 No Content (empty body)
   |
   |---> Range request fulfilled (video Range, resumable download) .... 206 Partial Content (+ Content-Range)
```

### Indexability quick reference

| Code | Indexable? | Use for crawled URLs? |
|---|---|---|
| 200 | YES (if substantive) | YES |
| 201 | Waits, may soft 404 | NO (creation is POST only) |
| 202 | Waits, may soft 404 | NO (async is POST only) |
| 204 | NO (soft 404) | NEVER |
| 206 | NO (partial) | NEVER (media only) |

### Five rules to memorize

1. 200 needs substantive content. Thin = soft 404.
2. 201 needs Location header. Always.
3. 204 needs empty body. Always.
4. 206 needs Content-Range. Always.
5. API endpoints noindex regardless of status code.

### Five commands every operator should know

```bash
# 1. Check status of a URL
curl -sI https://example.com/page | head -1

# 2. Check response size (thin = soft 404 risk)
curl -s -o /dev/null -w "%{size_download} bytes\n" https://example.com/page

# 3. Test Range support
curl -sI -H "Range: bytes=0-1023" https://example.com/video.mp4 | head -3

# 4. Find soft 404 candidates in logs
sudo awk '$9 == 200 && $10 < 5000 {print $7, $10}' /var/log/nginx/access.log | sort -u | head -20

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Substantive page returns 200 with real content
SIZE=$(curl -s -o /dev/null -w "%{size_download}" https://example.com/)
[ $SIZE -gt 5000 ] && echo "OK: substantive content" || echo "FAIL: thin"

# 2. DELETE returns 204 with empty body
RESPONSE=$(curl -sI -X DELETE https://api.example.com/test-resource/123)
echo "$RESPONSE" | head -1
echo "$RESPONSE" | grep -i content-length
# Expected: HTTP/2 204; content-length: 0 (or absent)

# 3. Video supports Range (206)
curl -sI -H "Range: bytes=0-1023" https://example.com/videos/test.mp4 | head -3
# Expected: HTTP/2 206; content-range: bytes 0-1023/...
```

If all three pass AND GSC Page Indexing > Soft 404 is empty or stable, the 2xx layer is correctly wired.

---

End of framework-http-2xx-status-codes.md.
