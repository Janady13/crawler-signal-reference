# framework-http-4xx-status-codes.md

Comprehensive reference for the HTTP 4xx Client Error status codes, the third framework in the status codes series after framework-http-2xx-status-codes.md and framework-http-3xx-status-codes.md. Covers `400 Bad Request` (the catch all for malformed requests), `401 Unauthorized` (the auth challenge), `403 Forbidden` (the auth denial that crawlers back off from), `404 Not Found` (the soft removal that can persist for months in the index), `410 Gone` (the surgical removal that deindexes in days), `422 Unprocessable Entity` (the validation failure), `429 Too Many Requests` (cross referenced to framework-http-rate-control-headers.md), `451 Unavailable For Legal Reasons` (the legal compliance signal), plus brief coverage of 405, 408, 411, 413, 414, 415. Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to the eight HTTP header frameworks plus framework-http-2xx-status-codes.md and framework-http-3xx-status-codes.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx error responses, AI assistants generating server logic that crawlers and clients interpret correctly, SEO operators planning bulk URL removals or pruning campaigns, security auditors reviewing auth and access denials, API engineers distinguishing validation failures from malformed requests, and anyone troubleshooting "404 pages stuck in Google index for months", "Googlebot kept crawling deleted pages", "API returns 500 when it should return 400", or "auth flow returns 401 when it should be 403".

This framework documents **the 5,800 thin page noindex plus 410 cleanup pattern** Joseph executed during the recent Bubbles infrastructure overhaul as the canonical reference for SEO pruning at scale.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The 4xx Mental Model (read this first)
5. 400 Bad Request (the malformed request catch all)
6. 401 Unauthorized (the auth challenge)
7. 403 Forbidden (the auth denial, Googlebot backs off)
8. 404 Not Found (the soft removal, weeks to months)
9. 410 Gone (the surgical removal, days)
10. 422 Unprocessable Entity (the validation failure)
11. 429 Too Many Requests (cross reference to framework-http-rate-control-headers.md)
12. 451 Unavailable For Legal Reasons (the legal compliance signal)
13. Other 4xx Codes (405, 408, 411, 413, 414, 415 briefly)
14. The 401 vs 403 Distinction (deep dive)
15. The 404 vs 410 Distinction (THE SEO CENTERPIECE)
16. The 400 vs 422 Distinction (validation semantics)
17. The Bubbles Thin Content Removal Pattern (Joseph's 5,800 page cleanup)
18. How Major Crawlers React To Each 4xx Code
19. Asset Class And Use Case Recipes
20. Bubbles Nginx Reference Block (paste ready)
21. Audit Checklist (50+ items)
22. Common Pitfalls
23. Diagnostic Commands
24. Cross-References

---

## 1. DEFINITION

4xx status codes signal that the client made a mistake. The request was malformed, the client lacks authorization, the resource does not exist, the action is forbidden, or the request is otherwise rejected. Defined across RFC 9110 and individual RFCs for specific codes. The 4xx category is the most operationally diverse of the status code families: each code has distinct semantics, distinct client reactions, and distinct crawler interpretations.

The eight codes Joseph listed plus 400 and 422 split into five functional groups:

* **Malformed request**: 400 Bad Request, 422 Unprocessable Entity. The request reached the server but cannot be processed as is.
* **Authentication and authorization**: 401 Unauthorized, 403 Forbidden. The client identity is missing or insufficient.
* **Resource state**: 404 Not Found, 410 Gone. The resource is absent (404 may return; 410 will not).
* **Rate and capacity**: 429 Too Many Requests. The client is sending too fast.
* **Legal compliance**: 451 Unavailable For Legal Reasons. The resource is blocked for legal reasons.

For SEO and crawler interaction, the critical pairs are 404 vs 410 (removal speed) and 401 vs 403 (Googlebot's reaction to access denial). For API design, the critical pairs are 400 vs 422 (validation semantics) and 401 vs 403 (auth state). For legal and compliance, 451 is the formal signal.

---

## 2. WHY IT MATTERS

Eight independent pressures push correct 4xx code usage from "default error" to "actively chosen signal" in 2025 and forward.

**404 vs 410 timing is days vs months in the index.** Per Google's actual behavior, 404 URLs are retried on a slowing schedule (24 hours, 7 days, 30 days, 90 days) and persist in the index for weeks to months. 410 URLs are dropped within days; retry frequency drops by approximately 92% after the second consecutive 410. For bulk removal campaigns (Joseph's 5,800 thin page cleanup), this difference is measured in months of crawl budget waste.

**403 makes Googlebot back off.** Per Google's documentation, 403 is treated as "access denied" and Googlebot reduces crawl rate to the affected URLs. Persistent 403 leads to deindexing. Misuse of 403 (returning it when 404 is correct) effectively hides pages from Google.

**401 vs 403 distinction matters for both security and crawlers.** 401 says "you need to authenticate"; 403 says "I won't tell you why or you're authenticated but lack permission". Crawlers treat them differently. Returning 401 when 403 is correct leaks information ("this URL exists and accepts authentication"); returning 403 when 401 is correct can prevent legitimate auth flows.

**400 vs 422 distinguishes wire format from semantic errors.** 400 is for malformed requests (syntax errors, missing required fields, invalid JSON). 422 is for well formed requests with semantically invalid data (email format wrong, password too short, foreign key doesn't exist). API clients use this distinction to retry differently: 400 means fix the request structure; 422 means fix the data.

**429 ties directly to crawl rate.** Persistent 429 to Googlebot for more than 2 days drops URLs from the index (per Google's documentation, also covered in framework-http-rate-control-headers.md). The crawler protection patterns from the rate control framework prevent this; this framework documents 429 from the status code side.

**451 is the only spec compliant legal removal signal.** GDPR right to erasure, DMCA takedowns, court orders for URL removal: 451 is the documented response. Search Console may explicitly note "Page removed because of legal complaint" when 451 is encountered.

**API clients depend on correct 4xx codes.** OpenAPI validators, Postman collections, SDK generators all expect specific codes for specific scenarios. Mixing them (returning 400 for everything, returning 200 with error bodies) breaks tooling.

**Cost of getting it wrong.** Misconfigured 4xx codes produce silent SEO damage, security incidents, API integration failures, and crawl budget waste. Real examples:

* Site removed 5,800 thin location pages by deleting them. Server returned 404 (default behavior). After 90 days, 4,200 URLs still in the index. After 180 days, 800 URLs still in the index. With 410, the same cleanup would have completed in 2 to 3 weeks.
* API returned 400 for "user not found" instead of 404. API client retry logic treated 400 as "fix the request and retry"; client looped sending the same request indefinitely.
* Login wall returned 401 with "WWW-Authenticate: Basic" header on every page. Browsers prompted users for HTTP Basic auth credentials despite the site using cookie auth. Users confused; conversion dropped.
* Public profile page returned 403 to all users including the profile owner. Should have been 401 for unauthenticated, 200 for authenticated. Profile owners locked out of their own pages.
* GDPR right to erasure request resulted in a 404. Google retried the URL for 60 days. The deleted user's data leaked through Google's snippet cache for 45 days. 451 would have triggered explicit legal removal.
* Site under DDoS responded with 429 with no Retry-After. Crawlers (including Googlebot) retried aggressively. Site fell offline. 503 with Retry-After would have signaled the right behavior.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The eight codes from Joseph's list plus 400 and 422 (essential for completeness) each get the same six part treatment:

1. **What it means**: the canonical RFC definition plus practical implication.
2. **When to use it**: which scenarios warrant this specific code.
3. **How to return it in nginx and FastAPI**: paste ready config and code.
4. **How crawlers react**: indexing pipeline behavior per major crawler.
5. **How to verify**: curl commands and observation patterns.
6. **Common misuse and how to fix**: typical wrong choices and replacements.

Sections 14 to 17 are deep dives on cross cutting concerns: the 401 vs 403 distinction, the 404 vs 410 distinction (the most ranking impactful), the 400 vs 422 distinction, and the Bubbles thin content removal pattern (Joseph's 5,800 page cleanup as the canonical reference).

---

## 4. THE 4xx MENTAL MODEL (READ THIS FIRST)

A request reaches the server and the server decides the client is at fault. The exact reason determines the code. Internalize the decision tree.

```
The request cannot be fulfilled. Whose fault is it?
   |
  Server's fault                     Client's fault
   |                                  |
   v                                  v
5xx codes (next framework)         4xx codes (this framework)
                                      |
                                      v
==================== WHAT DID THE CLIENT DO WRONG? ====================
                                      |
   +-------- The request itself is malformed --------+
   |                                                  |
   |       Wire format wrong (bad JSON, missing field) ... 400 Bad Request
   |                                                  |
   |       Wire format OK, data semantically invalid ... 422 Unprocessable Entity
   |                                                  |
   |       HTTP method not allowed for this URL ........ 405 Method Not Allowed
   |                                                  |
   |       Payload too large ........................... 413 Payload Too Large
   |                                                  |
   |       Content type not supported .................. 415 Unsupported Media Type
   |                                                  |
   +--------------------------------------------------+
   |
   +-------- The client is not authorized --------+
   |                                               |
   |       No credentials provided ................. 401 Unauthorized
   |                                               |
   |       Credentials provided but insufficient .... 403 Forbidden
   |                                               |
   +-----------------------------------------------+
   |
   +-------- The resource doesn't exist (or never will) --------+
   |                                                             |
   |       Not currently here, might return ......................... 404 Not Found
   |                                                             |
   |       Permanently gone, will never return ...................... 410 Gone
   |                                                             |
   +-------------------------------------------------------------+
   |
   +-------- The client is sending too fast --------+
   |                                                 |
   |       Rate limit exceeded ........................ 429 Too Many Requests
   |                                                 |
   +-------------------------------------------------+
   |
   +-------- Legal reasons prevent serving --------+
                                                    |
           Court order, DMCA, GDPR, etc ............... 451 Unavailable For Legal Reasons
```

Six rules govern the system:

1. **Use the most specific code that fits.** 404 is correct for "not found"; 400 is wrong even though it is technically true the request "didn't match a resource".
2. **404 for might return, 410 for never returning.** The distinction shapes crawler behavior dramatically.
3. **401 needs WWW-Authenticate header.** Without it, clients cannot complete the auth challenge.
4. **403 hides the reason.** Do not leak why; "you can't" is the full message.
5. **429 needs Retry-After.** Tells crawlers when to come back.
6. **451 is the only correct legal removal signal.** Not 404, not 410, not 403.

A correctly configured server returns the precise 4xx code matching the situation, includes appropriate response headers (WWW-Authenticate, Retry-After, etc), and writes useful body content where allowed (not for 401 challenges, never excessive for 429).

---

## 5. 400 BAD REQUEST (THE MALFORMED REQUEST CATCH ALL)

### 5.1 What It Means

400 Bad Request signals that the server cannot process the request because of something the client did wrong at the protocol or format level. Defined in RFC 9110.

Common causes:

* Malformed JSON in the body.
* Missing required headers.
* Invalid URL encoding.
* Invalid query parameters (wrong type, missing).
* Invalid HTTP version or method syntax.

```
HTTP/2 400 Bad Request
Content-Type: application/json

{"error": "invalid_request", "detail": "expected JSON body"}
```

### 5.2 When To Use 400

* JSON body parsing failed.
* Required header missing (`Content-Type` for a POST with body).
* URL parameter has wrong format (`?page=abc` when expecting integer).
* Invalid HTTP method syntax.
* Request line malformed.

### 5.3 When NOT To Use 400

* Resource not found: use 404.
* Authentication required: use 401.
* Validation of well formed input failed: use 422.
* Server side error: use 5xx.

### 5.4 How To Return 400

**In FastAPI:**

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=400,
        content={
            "error": "invalid_request",
            "detail": "request validation failed",
            "errors": exc.errors(),
        }
    )

@app.get("/items")
async def get_items(page: int):
    # FastAPI automatically returns 422 for type mismatch
    # For business logic errors, raise HTTPException with 400
    if page < 1:
        raise HTTPException(status_code=400, detail="page must be positive")
    return {"items": []}
```

Note: FastAPI defaults to 422 for request validation errors (mismatched types, missing required fields). To use 400 instead, override the validation handler as shown.

### 5.5 How To Verify

```bash
# Send malformed JSON, expect 400
curl -sI -X POST -H "Content-Type: application/json" \
    -d 'not valid json' \
    https://api.example.com/data | head -1
# Expected: HTTP/2 400

# Missing required header
curl -sI -X POST https://api.example.com/data | head -1
# May return 400 or 415 depending on server
```

### 5.6 Crawler Reaction

Per Google's documentation: 4xx codes (except 429) are treated as "client error". For 400 specifically, Googlebot treats the URL as unprocessable and may reduce crawl frequency. Persistent 400 leads to URL removal from index (but slowly; 410 is preferred for explicit removal).

For Bubbles client sites, 400 should be rare on indexable URLs. If a crawled URL returns 400, something is wrong (probably a misconfigured URL parameter parser).

### 5.7 Common Misuse And How To Fix

**Case: 400 returned for "user not found".**
Wrong code. The request was valid; the resource is not found. Fix:

```python
# WRONG
if not user:
    raise HTTPException(status_code=400, detail="user not found")

# RIGHT
if not user:
    raise HTTPException(status_code=404, detail="user not found")
```

**Case: 400 returned for "you don't have permission".**
Wrong code. Use 401 (no auth) or 403 (insufficient auth):

```python
# WRONG
if not user.has_permission("read", resource):
    raise HTTPException(status_code=400, detail="permission denied")

# RIGHT
if not user.has_permission("read", resource):
    raise HTTPException(status_code=403, detail="permission denied")
```

---

## 6. 401 UNAUTHORIZED (THE AUTH CHALLENGE)

### 6.1 What It Means

401 Unauthorized signals that the request requires authentication and the client did not provide it (or provided invalid credentials). Defined in RFC 9110. The response MUST include a `WWW-Authenticate` header describing how to authenticate.

```
HTTP/2 401 Unauthorized
WWW-Authenticate: Bearer realm="api"
Content-Type: application/json

{"error": "auth_required", "detail": "please authenticate"}
```

The name is historically misleading. "Unauthorized" actually means "unauthenticated". The correct code for "authenticated but lacks permission" is 403 Forbidden.

### 6.2 When To Use 401

* No Authorization header present and the endpoint requires authentication.
* Invalid Authorization header (expired token, malformed token, wrong scheme).
* Session cookie missing or expired and the endpoint requires session.
* Any case where the client could potentially gain access by providing valid credentials.

### 6.3 When NOT To Use 401

* The client is authenticated but lacks permission: use 403.
* The client should not be able to authenticate at all (e.g., banned account): use 403.
* The resource does not exist: use 404 (do not leak existence via 401 vs 404 distinction).

### 6.4 The Required WWW-Authenticate Header

Per RFC 9110, 401 responses MUST include `WWW-Authenticate`:

```
WWW-Authenticate: Bearer realm="api", error="invalid_token"
WWW-Authenticate: Basic realm="Internal Tools"
WWW-Authenticate: Cookie realm="session"
```

The header describes:

* The authentication scheme (Bearer, Basic, Digest, custom).
* The realm (a string identifying the protection space).
* Optional error details (`invalid_token`, `expired_token`, etc).

For browser facing endpoints using session cookies, the Bubbles convention is a custom "Cookie" scheme:

```
WWW-Authenticate: Cookie realm="bubbles-session"
```

For API endpoints using JWT or similar tokens:

```
WWW-Authenticate: Bearer realm="api"
```

Browsers see `Basic` and prompt the user with the native auth dialog. For Bubbles client sites using cookie auth, avoid Basic in WWW-Authenticate or browsers will prompt unnecessarily.

### 6.5 How To Return 401

**In nginx (for HTTP Basic auth on a specific location):**

```nginx
location /admin/ {
    auth_basic "Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
    # nginx returns 401 with WWW-Authenticate: Basic automatically
}
```

**In FastAPI:**

```python
from fastapi import FastAPI, HTTPException, Request, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

app = FastAPI()
security = HTTPBearer()

async def get_current_user(creds: HTTPAuthorizationCredentials = Depends(security)):
    user = await verify_token(creds.credentials)
    if not user:
        raise HTTPException(
            status_code=401,
            detail="invalid token",
            headers={"WWW-Authenticate": 'Bearer realm="api", error="invalid_token"'}
        )
    return user

@app.get("/api/me")
async def me(user = Depends(get_current_user)):
    return user
```

For cookie based auth without browser prompting:

```python
async def get_session_user(request: Request):
    session = request.cookies.get("session")
    if not session:
        raise HTTPException(
            status_code=401,
            detail="auth required",
            headers={"WWW-Authenticate": 'Cookie realm="bubbles-session"'}
        )
    user = await load_session(session)
    if not user:
        raise HTTPException(
            status_code=401,
            detail="invalid session",
            headers={"WWW-Authenticate": 'Cookie realm="bubbles-session"'}
        )
    return user
```

### 6.6 How To Verify

```bash
# No auth, expect 401 with WWW-Authenticate
curl -sI https://api.example.com/private | grep -iE "^(HTTP|WWW-Authenticate)"
# Expected:
# HTTP/2 401
# www-authenticate: Bearer realm="api"

# With valid auth, expect 200
curl -sI -H "Authorization: Bearer validtoken" https://api.example.com/private | head -1
# Expected: HTTP/2 200

# With invalid auth, expect 401 with error info
curl -sI -H "Authorization: Bearer invalid" https://api.example.com/private | head -3
# Expected: HTTP/2 401 with WWW-Authenticate: Bearer realm="api", error="invalid_token"
```

### 6.7 Crawler Reaction

Per Google's documentation: 401 is treated as access denied. Googlebot does not authenticate (it has no credentials). Persistent 401 leads to URL removal from index. For protected content that should not be indexed, 401 is appropriate.

For Bubbles client sites: pages requiring login should return 401 with proper WWW-Authenticate. Crawlers correctly understand "this is not for you" and skip it.

### 6.8 The Login Wall Antipattern

A common mistake: returning 200 with empty body or a redirect to /login for unauthenticated requests. This produces soft 404 (covered in framework-http-2xx-status-codes.md). The fix is 401:

```python
# WRONG (soft 404)
if not user:
    return HTMLResponse("<html>Please log in</html>", status_code=200)

# WRONG (302 to login, but page still shows in index)
if not user:
    return RedirectResponse(url="/login", status_code=302)

# RIGHT (401 with proper challenge)
if not user:
    raise HTTPException(
        status_code=401,
        detail="auth required",
        headers={"WWW-Authenticate": 'Cookie realm="bubbles-session"'}
    )
```

For pages that humans need to see the login screen for, the convention is:

* Return 401 if the request looks like a programmatic client (API call).
* Render the login page directly with `X-Robots-Tag: noindex` if the request looks like a browser navigation.

Or, simpler: always return 401 and let JavaScript on the client side handle the redirect to /login. Crawlers see 401 and skip.

---

## 7. 403 FORBIDDEN (THE AUTH DENIAL, GOOGLEBOT BACKS OFF)

### 7.1 What It Means

403 Forbidden signals that the server understood the request but refuses to authorize it. Defined in RFC 9110. The client may or may not be authenticated; the point is that the request will not be served regardless. The response typically does not explain why.

```
HTTP/2 403 Forbidden
Content-Type: application/json

{"error": "forbidden"}
```

### 7.2 When To Use 403

* Authenticated user lacks permission to access the resource.
* Banned IP address or user account.
* IP based access control (admin endpoints restricted to office VPN).
* CSRF check failed.
* Server detected abusive behavior (probing, scanning).
* "I will not serve this content" for any reason that is not auth, not rate limit, not absent.

### 7.3 When NOT To Use 403

* No credentials provided: use 401.
* Resource does not exist: use 404 (do not leak existence by returning 403 vs 404).
* Rate limit exceeded: use 429.
* Legal removal: use 451.

### 7.4 The 401 vs 403 Decision

The fundamental question: **can the client gain access by authenticating?**

* **YES (client just needs to log in)**: 401.
* **NO (client is authenticated but lacks permission, or won't be served regardless)**: 403.

Examples:

* Request to /admin without auth: 401 (login could grant access if user has admin role).
* Request to /admin with regular user auth: 403 (this user lacks admin role; authenticating again won't help).
* Request from banned IP: 403 (no amount of authentication helps).
* Request to user's own private settings without auth: 401.
* Request to another user's private settings (with valid auth, just wrong user): 403.

### 7.5 The Critical Crawler Behavior

Per Google's documentation and observed behavior: 403 signals to Googlebot that the URL is access denied. Googlebot:

* **Reduces crawl rate** to URLs returning 403.
* **Treats 403 as a removal candidate** if persistent. After repeated 403s over weeks, the URL drops from the index.
* **Does not retry as aggressively** as 404.

This makes 403 useful for SEO scenarios where:

* You want to deny access (admin pages, internal tools) but do not want to use 410 (which suggests "gone").
* You want to signal "this is access controlled" without revealing what is behind it.

The Bubbles practical use: admin endpoints (`/admin/*`, `/api/internal/*`) returning 403 keep Googlebot from wasting crawl budget on them.

### 7.6 How To Return 403

**In nginx (IP based access control):**

```nginx
location /admin/ {
    # Allow only office and Tailscale ranges
    allow 100.90.0.0/16;    # Tailscale
    allow 127.0.0.1;
    allow ::1;
    # Office VPN range
    # allow 198.51.100.0/24;
    deny all;
    # Returns 403 for denied IPs

    proxy_pass http://127.0.0.1:9090;
}
```

**In nginx (denied user agents):**

```nginx
# Block specific user agents
map $http_user_agent $is_bad_bot {
    default 0;
    ~*Bytespider 1;
    ~*PetalBot   1;
    ~*MJ12bot    1;
    ~*AhrefsBot  1;
    ~*SemrushBot 1;
}

location / {
    if ($is_bad_bot) {
        return 403;
    }
    # ...
}
```

**In FastAPI:**

```python
@app.get("/api/admin/users")
async def admin_users(user = Depends(get_current_user)):
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="forbidden")
    return await list_all_users()

@app.get("/api/users/{user_id}/private-data")
async def user_private(user_id: int, current = Depends(get_current_user)):
    if current.id != user_id and not current.is_admin:
        raise HTTPException(status_code=403, detail="forbidden")
    return await get_private_data(user_id)
```

### 7.7 How To Verify

```bash
# Authenticated but lacks permission
curl -sI -H "Authorization: Bearer user_token" \
    https://api.example.com/api/admin/users | head -1
# Expected: HTTP/2 403

# Without auth (should be 401, not 403)
curl -sI https://api.example.com/api/admin/users | head -1
# Expected: HTTP/2 401

# Banned IP (if testing)
curl -sI --interface 10.0.0.99 https://example.com/admin/ | head -1
# Expected: HTTP/2 403
```

### 7.8 The "I Refuse To Tell You Why" Convention

403 responses should not explain why. Detailed reasons leak information useful to attackers:

```json
// BAD (information disclosure)
{
    "error": "forbidden",
    "reason": "user_id 12847 does not have role 'admin' which is required for /admin/users",
    "your_roles": ["editor", "viewer"]
}

// GOOD (minimal)
{
    "error": "forbidden"
}
```

For Bubbles client sites: keep 403 bodies short. For internal admin tools where information disclosure is not a concern, expanded messages are fine.

---

## 8. 404 NOT FOUND (THE SOFT REMOVAL, WEEKS TO MONTHS)

### 8.1 What It Means

404 Not Found signals that the server cannot find the requested resource. Defined in RFC 9110. The resource might exist in the future; the server does not commit to permanent absence.

```
HTTP/2 404 Not Found
Content-Type: text/html

<html><body><h1>404 Not Found</h1><p>The page you requested does not exist.</p></body></html>
```

### 8.2 When To Use 404

* The URL pattern does not match any resource.
* A resource that existed has been deleted but might come back (or you are not sure).
* Default response for any "I don't know what this is" scenario.
* Typo or invalid URL.

### 8.3 When NOT To Use 404

* Permanently removed pages where SEO removal speed matters: use 410.
* Resource exists but client lacks permission: use 403.
* Resource exists but request is malformed: use 400.
* Legal removal: use 451.

### 8.4 The Critical Crawler Retry Behavior

Per Google's documented behavior and observed timing:

> "404 Not Found tells the client that the server cannot locate the requested resource at this specific moment. Crucially, it leaves the door open for the future."

Googlebot's retry schedule for 404 URLs:

| Hours since last crawl | Retry behavior |
|---|---|
| 24 hours | First retry |
| 7 days | Second retry |
| 30 days | Third retry |
| 90 days | Fourth retry |
| 180+ days | Sporadic retries |

The URL stays in the index throughout this period, marked as "Not found (404)" in Google Search Console's Page Indexing report. For high authority sites (Joseph notes this from prior observation), Googlebot may continue retrying 404 URLs for **months**.

**For bulk removal scenarios, this is a problem.** 5,800 thin URLs returning 404 will linger in the index and drain crawl budget for 3+ months.

### 8.5 The Custom 404 Page Requirement

Default nginx 404 pages are minimal HTML. For user experience, provide a useful custom 404:

```nginx
server {
    error_page 404 /404.html;

    location = /404.html {
        root /var/www/sites/example.com;
        internal;
        # do NOT add X-Robots-Tag: index here; the URL is 404, not a content URL
    }
}
```

The custom 404 page itself returns 404 (not 200). Common mistake: returning 200 with the 404 design (a soft 404 covered in framework-http-2xx-status-codes.md).

### 8.6 How To Return 404

**In nginx:**

```nginx
location / {
    try_files $uri $uri/ $uri.html =404;
    # If no file found, returns 404
}
```

**In FastAPI:**

```python
@app.get("/articles/{slug}")
async def article(slug: str):
    article = await get_article(slug)
    if not article:
        raise HTTPException(status_code=404, detail="article not found")
    return article
```

### 8.7 How To Verify

```bash
# Expect 404 for non existent URL
curl -sI https://example.com/this-does-not-exist | head -1
# Expected: HTTP/2 404

# Verify custom 404 page content
curl -s https://example.com/this-does-not-exist | head -20
# Expected: substantive HTML, not a blank or generic page

# Verify custom 404 returns 404 status (NOT 200 with 404 content)
STATUS=$(curl -so /dev/null -w "%{http_code}" https://example.com/this-does-not-exist)
echo "Status: $STATUS"
# Expected: 404
```

### 8.8 Crawler Reaction Summary

* **Googlebot**: 404 URL stays in index, retried on slowing schedule for weeks to months.
* **Bingbot**: similar to Googlebot, retries less aggressively.
* **ClaudeBot, GPTBot, OAI-SearchBot**: treat 404 as "not currently available", may retry; less aggressive than search crawlers.

### 8.9 Common Misuse And How To Fix

**Case: deleted page redirected to homepage (returning 301 to /).**
Wrong. Redirecting to homepage creates a soft 404 (Google detects "no canonical match"). Fix:

```python
# WRONG (soft 404)
return RedirectResponse(url="/", status_code=301)

# RIGHT (404 for unknown URL, OR 301 to relevant content)
raise HTTPException(status_code=404, detail="not found")
# OR
return RedirectResponse(url="/category/related", status_code=301)
```

**Case: catchall route returns 200 with "page not found" message.**
Wrong (soft 404). The catchall must return 404 status:

```python
@app.exception_handler(404)
async def custom_404(request, exc):
    return HTMLResponse(
        content=render_404_template(),
        status_code=404,  # MUST be 404, not 200
    )
```

---

## 9. 410 GONE (THE SURGICAL REMOVAL, DAYS)

### 9.1 What It Means

410 Gone signals that the resource is permanently no longer available. The server explicitly commits to "this URL will never return". Defined in RFC 9110.

```
HTTP/2 410 Gone
Content-Type: text/html

<html><body><h1>This Page Is Gone</h1><p>The content at this URL was removed and will not return.</p></body></html>
```

The semantic difference from 404 is the permanence commitment.

### 9.2 When To Use 410

* Bulk pruning of low quality content (the Bubbles thin page cleanup pattern).
* Removed product pages where the product is discontinued forever.
* Spam or hacked content cleanup.
* Old URL patterns from a previous CMS that should not return.
* Any "I know this is gone forever" scenario.

### 9.3 When NOT To Use 410

* Maybe the URL will come back: use 404.
* The content moved: use 301 to the new URL.
* Auth required: use 401 or 403.
* Legal removal: use 451.

### 9.4 The Critical Crawler Speed Difference

Per industry research and Google's documented behavior:

> "After the second consecutive time Googlebot encounters a 410 on a specific URL, the retry frequency drops by an average of 92% compared to the baseline. Google effectively trusts the 410 signal much faster than the 404."

Practical timing:

| Status code | Time to drop from index | Retry frequency |
|---|---|---|
| 404 | 2-4 weeks typical, up to 6 months for high authority sites | 24h, 7d, 30d, 90d, ongoing |
| 410 | Days (typically 1-2 weeks) | Drops 92% after second consecutive 410 |

John Mueller (Google) has confirmed: "A 410 will sometimes fall out a little bit faster than a 404. But usually, we're talking on the order of a couple days or so." In aggregate across thousands of URLs, this becomes meaningful weeks of saved crawl budget.

### 9.5 How To Return 410

**In nginx (single URL):**

```nginx
location = /old-removed-page {
    return 410;
}
```

**In nginx (pattern):**

```nginx
# All URLs under /old-section/ are permanently gone
location ^~ /old-section/ {
    return 410;
}

# All URLs matching a pattern
location ~ ^/services/(extremely-thin-city|another-thin-city)/ {
    return 410;
}
```

**In nginx (bulk via map):**

```nginx
map $request_uri $is_gone {
    default 0;
    ~^/old-section/ 1;
    ~^/spam-cleanup/ 1;
    /old-page-1 1;
    /old-page-2 1;
    # ... thousands of entries possible
}

server {
    location / {
        if ($is_gone) {
            return 410;
        }
        try_files $uri $uri/ $uri.html =404;
    }
}
```

**With custom 410 page:**

```nginx
location ^~ /old-section/ {
    return 410;
}

error_page 410 /410.html;
location = /410.html {
    root /var/www/sites/example.com;
    internal;
    add_header X-Robots-Tag "noindex" always;
}
```

**In FastAPI:**

```python
@app.get("/articles/{slug}")
async def article(slug: str):
    article = await get_article(slug)
    if not article:
        # Check if this URL was explicitly removed
        if await is_permanently_removed(slug):
            raise HTTPException(status_code=410, detail="content removed")
        raise HTTPException(status_code=404, detail="not found")
    return article
```

### 9.6 The Bulk 410 Pattern (For Joseph's Cleanup Style)

For large scale URL removal (Joseph's 5,800 page cleanup), the pattern is:

1. **Generate the removal list.** Export the URLs to remove from GSC, database, or sitemap diff.
2. **Generate nginx config.** Build a `map` or `location` block listing each URL.
3. **Deploy and reload.** `nginx -t && systemctl reload nginx`.
4. **Submit URL removal in GSC.** For the most critical URLs, accelerate via Google Search Console URL Removals tool.
5. **Monitor GSC Coverage report.** Track "Not found (410)" count weekly.

```bash
# Generate nginx map from list of URLs to remove
echo "map \$request_uri \$is_gone {" > /etc/nginx/snippets/gone-urls.conf
echo "    default 0;" >> /etc/nginx/snippets/gone-urls.conf
while read url; do
    echo "    $url 1;" >> /etc/nginx/snippets/gone-urls.conf
done < /tmp/urls-to-remove.txt
echo "}" >> /etc/nginx/snippets/gone-urls.conf

nginx -t && systemctl reload nginx
```

Then include this snippet in `nginx.conf` at the http level, and check `$is_gone` in each server block.

### 9.7 How To Verify

```bash
# Expect 410 for explicitly removed URL
curl -sI https://example.com/old-removed-page | head -1
# Expected: HTTP/2 410

# Verify custom 410 page if configured
curl -s https://example.com/old-removed-page | head -10

# Test that 410 URLs are NOT returning 404
# (404 would cause Google to retry for months; 410 drops in days)
for url in /old-page-1 /old-page-2 /old-page-3; do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "https://example.com$url")
    if [ "$STATUS" = "410" ]; then
        echo "OK: $url returns 410"
    else
        echo "WRONG: $url returns $STATUS (expected 410)"
    fi
done
```

### 9.8 Common Misuse And How To Fix

**Case: bulk removed content returning 404 instead of 410.**
The cleanup will work but take 3 to 5 times longer. Fix by switching to 410:

```nginx
# WRONG (just removing files, gets 404 by default)
# Files deleted, no nginx config

# RIGHT (explicit 410)
location ^~ /removed-section/ {
    return 410;
}
```

**Case: 410 used for a page that might come back.**
410 is a commitment. If the content might return, use 404 or rename and 301 redirect when ready.

**Case: 410 used instead of 301 to a new URL.**
Lost SEO opportunity. If the content moved, redirect:

```nginx
# WRONG: visitors lost
location = /old-product-page { return 410; }

# RIGHT: visitors and equity directed to new page
location = /old-product-page { return 301 /products/replacement; }
```

---

## 10. 422 UNPROCESSABLE ENTITY (THE VALIDATION FAILURE)

### 10.1 What It Means

422 Unprocessable Entity signals that the server understands the request (syntax is correct) but cannot process the contained instructions (semantics are invalid). Defined in RFC 9110. Common for API validation failures.

```
HTTP/2 422 Unprocessable Entity
Content-Type: application/json

{
    "error": "validation_failed",
    "errors": [
        {"field": "email", "msg": "invalid email format"},
        {"field": "password", "msg": "must be at least 8 characters"}
    ]
}
```

### 10.2 When To Use 422

* Field validation failed (email format, password length, date range).
* Foreign key references non existent resource.
* Business rule violation (e.g., trying to book a date in the past).
* Conflict between fields (e.g., end_date before start_date).

### 10.3 When NOT To Use 422

* JSON malformed or syntax broken: use 400.
* Auth missing or invalid: use 401 or 403.
* Resource not found: use 404.
* Action conflicts with current state of another resource: 409 Conflict.

### 10.4 The 400 vs 422 Distinction

The cleanest distinction:

* **400 Bad Request**: the request itself is malformed at the wire/syntax level. The client must fix the structure.
* **422 Unprocessable Entity**: the request is well formed but the data is invalid for business or schema reasons. The client must fix the data.

Examples:

| Scenario | Code |
|---|---|
| JSON body has trailing comma | 400 |
| JSON body is valid but `email` field is `null` and required | 422 |
| Required `Content-Type` header missing | 400 |
| `Content-Type` is `application/json` but body parses as `application/xml` | 400 |
| All fields present but `password` is only 4 characters | 422 |
| `start_date` is "yesterday" and business rule says future only | 422 |

FastAPI defaults to 422 for request validation errors (type mismatch, missing fields). This is FastAPI's documented convention; the strictest reading of RFC 9110 would call this 400, but 422 is widely used by APIs and matches user expectations.

### 10.5 How To Return 422

**In FastAPI (default):**

```python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr, Field

app = FastAPI()

class CreateUser(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    password: str = Field(min_length=8)

@app.post("/api/users")
async def create_user(user: CreateUser):
    # FastAPI automatically returns 422 for validation errors
    new_id = await db_create_user(user)
    return {"id": new_id, **user.dict(exclude={"password"})}
```

FastAPI's default validation error response:

```json
{
    "detail": [
        {
            "type": "string_too_short",
            "loc": ["body", "password"],
            "msg": "String should have at least 8 characters",
            "input": "abc",
            "ctx": {"min_length": 8}
        }
    ]
}
```

**Custom 422 for business rule violations:**

```python
@app.post("/api/bookings")
async def create_booking(booking: BookingRequest):
    if booking.start_date >= booking.end_date:
        raise HTTPException(
            status_code=422,
            detail={
                "error": "validation_failed",
                "errors": [{"field": "end_date", "msg": "must be after start_date"}]
            }
        )
    if booking.start_date < date.today():
        raise HTTPException(
            status_code=422,
            detail={
                "error": "validation_failed",
                "errors": [{"field": "start_date", "msg": "must be in the future"}]
            }
        )
    # ...
```

### 10.6 How To Verify

```bash
# Send invalid email, expect 422
curl -sI -X POST -H "Content-Type: application/json" \
    -d '{"name":"test","email":"not-an-email","password":"validpassword"}' \
    https://api.example.com/users | head -1
# Expected: HTTP/2 422

# See error detail
curl -s -X POST -H "Content-Type: application/json" \
    -d '{"name":"test","email":"not-an-email","password":"validpassword"}' \
    https://api.example.com/users | python3 -m json.tool
```

### 10.7 Crawler Reaction

Like 400: not for crawled URLs in typical scenarios. API endpoints returning 422 should have `X-Robots-Tag: noindex` (already in the standard `/api/` location).

---

## 11. 429 TOO MANY REQUESTS (CROSS REFERENCE TO FRAMEWORK-HTTP-RATE-CONTROL-HEADERS.MD)

### 11.1 What It Means

429 Too Many Requests signals that the client has exceeded a rate limit. Defined in RFC 6585. Should always be paired with `Retry-After` indicating when the client can try again.

```
HTTP/2 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1742658660
Content-Type: application/json

{"error": "rate_limit_exceeded", "retry_after": 60}
```

### 11.2 Where To Look For The Full Treatment

The full treatment of 429 (when to use, how to configure nginx `limit_req_status 429`, FastAPI rate limiter implementations, the crawler whitelist pattern that protects Googlebot from being throttled) is in **framework-http-rate-control-headers.md**, particularly:

* Section 5: Retry-After header
* Section 8: Status code pairing (429 vs 503)
* Section 9: nginx limit_req configuration
* Section 10: The crawler protection rule (never rate limit Googlebot, ClaudeBot, GPTBot)

### 11.3 The Critical 2 Day Rule

The single most ranking critical detail: per Google's documentation, returning 429 (or 503) for more than 2 days will cause Google to drop those URLs from the index. The mitigation is the crawler whitelist pattern documented in framework-http-rate-control-headers.md Section 10.

### 11.4 Quick Reference: nginx 429 Config

The minimal correct configuration:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_status 429;  # MUST be 429 not the default 503
    limit_req_log_level warn;
}

server {
    location /api/ {
        limit_req zone=api burst=60 nodelay;
        # ... rest of API config
    }

    error_page 429 = @rate_limited;

    location @rate_limited {
        internal;
        add_header Retry-After "60" always;
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
        return 429 '{"error": "rate_limit_exceeded", "retry_after": 60}';
    }
}
```

For the complete pattern including crawler whitelisting, see framework-http-rate-control-headers.md.

---

## 12. 451 UNAVAILABLE FOR LEGAL REASONS (THE LEGAL COMPLIANCE SIGNAL)

### 12.1 What It Means

451 Unavailable For Legal Reasons signals that the resource is blocked due to legal requirements. Defined in RFC 7725 (with a nod to Ray Bradbury's "Fahrenheit 451"). Specifically intended for court orders, takedown notices, geographic restrictions imposed by law, and similar legal mandates.

```
HTTP/2 451 Unavailable For Legal Reasons
Content-Type: text/html
Link: <https://example.com/legal/takedown/12847>; rel="blocked-by"

<html><body><h1>Unavailable For Legal Reasons</h1>
<p>This content has been removed pursuant to a legal order.</p>
<p>For details: <a href="/legal/takedown/12847">/legal/takedown/12847</a></p>
</body></html>
```

### 12.2 When To Use 451

* **GDPR right to erasure**: a user's personal data was deleted following a GDPR request.
* **DMCA takedown**: content removed due to copyright complaint.
* **Court order**: judicial order to remove specific content or block access.
* **Geographic restriction by law**: content not available in a specific jurisdiction due to local law.
* **Trademark dispute resolution**: domain or content removed pending dispute resolution.

### 12.3 When NOT To Use 451

* **Voluntary content removal**: use 410 (no legal compulsion).
* **Generic copyright issue without formal DMCA**: use 410.
* **Access control / paywall**: use 401 or 403.
* **Maintenance**: use 503.

### 12.4 The Optional Link Header

RFC 7725 defines an optional `Link` header pointing to information about the block:

```
Link: <https://example.com/legal/info/12847>; rel="blocked-by"
```

This can describe who imposed the block (court, government agency, the entity that filed the takedown). Useful for transparency and the Lumen Database type registries.

### 12.5 How To Return 451

**In nginx:**

```nginx
# Specific URL legally removed
location = /legally-removed-content {
    add_header Link '<https://example.com/legal/takedown/12847>; rel="blocked-by"' always;
    return 451;
}

# Or with custom error page
error_page 451 /451.html;

location = /451.html {
    root /var/www/sites/example.com;
    internal;
}
```

**In FastAPI:**

```python
@app.get("/content/{content_id}")
async def get_content(content_id: int):
    content = await get_content_by_id(content_id)
    if not content:
        raise HTTPException(status_code=404)

    if content.legally_blocked:
        raise HTTPException(
            status_code=451,
            detail="Content unavailable for legal reasons",
            headers={
                "Link": f'<https://example.com/legal/takedown/{content.takedown_id}>; rel="blocked-by"',
            }
        )

    return content
```

### 12.6 How To Verify

```bash
# Expect 451 with Link header
curl -sI https://example.com/legally-removed-content | grep -iE "^(HTTP|Link)"
# Expected:
# HTTP/2 451
# link: <https://example.com/legal/takedown/...>; rel="blocked-by"
```

### 12.7 Crawler Reaction

Per Google's documentation: 451 may be treated similarly to 404/410 (content not available) but Search Console may explicitly show "Page removed because of legal complaint" indicator. The content drops from the index. This is the spec compliant way to signal legal removal.

For GDPR right to erasure specifically, 451 is the correct response (the content was removed under legal compulsion). 410 is acceptable but does not convey the legal context.

### 12.8 The DMCA Context

For DMCA takedowns in the US:

1. Receive valid DMCA takedown notice.
2. Remove the content.
3. Server returns 451 for the URL going forward.
4. Optional: file a counter notice if disputed.

Compare to GDPR:

1. Receive valid GDPR right to erasure request.
2. Delete user data associated with the request.
3. URLs that exposed that data return 451 (or 410 if more appropriate).

---

## 13. OTHER 4xx CODES (BRIEFLY)

Joseph did not list these, but they appear in real traffic and the Bubbles audit should account for them.

### 13.1 405 Method Not Allowed

The HTTP method (GET, POST, etc) is not allowed for this URL.

```
HTTP/2 405 Method Not Allowed
Allow: GET, POST

(body optional)
```

The `Allow` header MUST list the supported methods. nginx returns 405 automatically when a method is not in the location's allowed set; FastAPI returns 405 automatically when no endpoint matches the method on a URL.

### 13.2 408 Request Timeout

The server closed the connection because the client did not send the request in time. Rare in modern traffic. Browsers may retry after 408.

```nginx
# nginx default behavior, controlled by:
client_header_timeout 60s;
client_body_timeout 60s;
```

### 13.3 411 Length Required

The server requires a `Content-Length` header and the client did not send one. Required for some PUT/POST scenarios. Generally rare.

### 13.4 413 Payload Too Large

The request body exceeds the server's limit.

```nginx
# nginx config
client_max_body_size 10m;  # default is 1m
# Returns 413 if request exceeds this
```

For file upload endpoints, set `client_max_body_size` appropriately (e.g., `100m` for multi MB uploads).

### 13.5 414 URI Too Long

The request URI exceeds the server's limit. Typically only seen with extremely long query strings or buffer overflow attacks. nginx defaults handle most cases; tune `large_client_header_buffers` if needed.

### 13.6 415 Unsupported Media Type

The server cannot process the request body because its `Content-Type` is not supported.

```python
@app.post("/api/upload")
async def upload(request: Request):
    content_type = request.headers.get("content-type", "")
    if not content_type.startswith("application/json"):
        raise HTTPException(
            status_code=415,
            detail="content-type must be application/json"
        )
    # ...
```

### 13.7 409 Conflict

Mentioned for completeness: the request conflicts with the current state of the resource. Common for duplicate creation attempts (e.g., POST creating a user with an email that already exists).

```python
@app.post("/api/users")
async def create_user(user: CreateUser):
    if await user_exists_with_email(user.email):
        raise HTTPException(
            status_code=409,
            detail="user with this email already exists"
        )
    # ...
```

---

## 14. THE 401 VS 403 DISTINCTION (DEEP DIVE)

### 14.1 The Fundamental Question

For every access denied scenario, ask: **can the client gain access by providing different credentials?**

| Scenario | Can different credentials help? | Status |
|---|---|---|
| No Authorization header | Yes, valid token would work | 401 |
| Expired token | Yes, refresh token | 401 |
| Token for wrong user | Yes, correct user's token | depends (see below) |
| Banned account | No, regardless of credentials | 403 |
| Banned IP | No, regardless of credentials | 403 |
| Authenticated but lacks role | No, the role is what's missing | 403 |
| Resource is admin only | No (unless current user IS admin) | 403 |
| CSRF check failed | No, the request itself is the issue | 403 |

### 14.2 The "Wrong User's Token" Edge Case

If user A's token is presented for accessing user B's private data:

* If the endpoint is "user's own data" pattern: 403 (this user just isn't the right user; different credentials might help).
* If the endpoint requires admin role: 403 (this user lacks admin; different role required).

The 401 path would apply if the token was outright invalid (expired, malformed, signed by wrong key).

### 14.3 The Information Leak Concern

For sensitive resources, you may want to mask whether the resource exists at all. Two patterns:

**Pattern A: Always 404 for non owned resources.**

```python
@app.get("/api/users/{user_id}/private")
async def user_private(user_id: int, current = Depends(get_current_user)):
    user = await get_user(user_id)
    if not user:
        raise HTTPException(status_code=404)

    if current.id != user.id and not current.is_admin:
        # Pretend the user doesn't exist (hides existence)
        raise HTTPException(status_code=404)

    return await get_private_data(user_id)
```

**Pattern B: Honest 403 with no detail.**

```python
@app.get("/api/users/{user_id}/private")
async def user_private(user_id: int, current = Depends(get_current_user)):
    user = await get_user(user_id)
    if not user:
        raise HTTPException(status_code=404)

    if current.id != user.id and not current.is_admin:
        raise HTTPException(status_code=403, detail="forbidden")

    return await get_private_data(user_id)
```

Pattern A hides resource existence (useful for high security applications). Pattern B is honest but tells the client the resource exists. Choose based on threat model.

### 14.4 The Crawler Difference

Crawlers react differently:

* **401**: "auth required, I can't authenticate, I'll back off"
* **403**: "I'm denied, I'll reduce crawl rate and may deindex"
* **404**: "this URL doesn't exist, I'll retry later"

For pages crawlers should not see and should not retry:

* **401** if the page conceptually requires auth (login walls, account pages).
* **403** if the page is hard restricted (admin tools, internal APIs).
* **410** if the page is gone forever and you want fast removal.

---

## 15. THE 404 VS 410 DISTINCTION (THE SEO CENTERPIECE)

### 15.1 The Single Most Important Distinction In 4xx

For any URL that no longer exists, the choice between 404 and 410 determines how quickly the URL drops from the search index. The difference is **days vs months**.

### 15.2 The Timeline Comparison

| Time elapsed | 404 status | 410 status |
|---|---|---|
| 0 hours | First Googlebot encounter | First Googlebot encounter |
| 24 hours | First retry | First retry (still 410) |
| 7 days | Second retry, still in index | URL dropped from index |
| 30 days | Third retry, may show in "Not found" | Gone, retry frequency dropped 92% |
| 90 days | Fourth retry, may still be in index | Forgotten |
| 6+ months | Still retrying, possibly still in index for high authority sites | Forgotten |

### 15.3 When Each Is The Right Answer

**Use 404 when:**

* The URL might be a typo or accident.
* You are not sure if the content will return.
* You inherited a site and do not know the URL's history.
* The URL pattern is "anything not explicitly matched" (the default fallback).
* You want a generic "not found" response for unknown URLs.

**Use 410 when:**

* You are intentionally and permanently removing specific URLs.
* You are doing bulk content pruning (the Bubbles thin page cleanup).
* The content was spam, hacked, or otherwise undesired.
* The URL pattern is one you specifically know should never serve content.
* You want fast removal from the index.

### 15.4 The Bulk Removal Pattern

For Joseph's 5,800 thin page cleanup scenario:

**Step 1: Identify URLs to remove.**

```bash
# Export from GSC Page Indexing > Soft 404 or low quality URLs
# Or generate from sitemap diff between old and new states
# Save to /tmp/urls-to-remove.txt, one URL per line
```

**Step 2: Categorize.**

For each URL, decide:

* **301 to relevant new page**: when there's a semantic replacement.
* **410 for permanent removal**: when there's no replacement and the URL should disappear.
* **noindex but keep accessible**: when users still need to reach it but Google should not index.

**Step 3: Generate nginx config.**

```bash
# For permanent removals
cat /tmp/urls-to-410.txt | while read url; do
    echo "    \"$url\" 1;"
done > /etc/nginx/snippets/410-urls.conf

# Then in nginx.conf http context:
# include /etc/nginx/snippets/gone-urls-map.conf;

# Where gone-urls-map.conf is:
cat > /etc/nginx/snippets/gone-urls-map.conf <<EOF
map \$request_uri \$is_gone {
    default 0;
    include /etc/nginx/snippets/410-urls.conf;
}
EOF
```

In each server block:

```nginx
location / {
    if ($is_gone) {
        return 410;
    }
    try_files $uri $uri/ $uri.html =404;
}
```

**Step 4: Deploy and monitor.**

```bash
nginx -t && systemctl reload nginx

# Verify a few URLs return 410
for url in $(head -5 /tmp/urls-to-remove.txt); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "https://example.com$url")
    echo "$url: $STATUS"
done

# Monitor GSC Coverage report weekly
# "Not found (410)" count should drop steadily
```

### 15.5 The "Stuck" 404 Problem

If you have URLs that have been 404 for months but are still in the index:

1. **Confirm they actually return 404 (or 410)** via direct curl.
2. **Submit URL removal in GSC** for the most critical ones (URL Removals tool).
3. **Convert to 410** for faster removal:

```nginx
# Was returning 404 by default; convert to explicit 410
location ^~ /old-section/ {
    return 410;
}
```

4. **Resubmit the sitemap** without those URLs.
5. **Wait 2 to 4 weeks** for Google to recrawl and process the change.

### 15.6 The Custom 410 Page

For user experience, provide a custom 410 page:

```nginx
error_page 410 /410.html;

location = /410.html {
    root /var/www/sites/example.com;
    internal;
    add_header X-Robots-Tag "noindex" always;
}
```

```html
<!-- /var/www/sites/example.com/410.html -->
<!doctype html>
<html lang="en">
<head>
<title>Content Removed</title>
</head>
<body>
<h1>This content has been removed</h1>
<p>The page you were looking for has been permanently removed and will not return.</p>
<p><a href="/">Return to homepage</a> or <a href="/blog/">browse our latest articles</a>.</p>
</body>
</html>
```

Note: substantive enough that if accidentally indexed, it is not a soft 404.

---

## 16. THE 400 VS 422 DISTINCTION (VALIDATION SEMANTICS)

### 16.1 The Cleanest Split

* **400**: the request is malformed. Fix the request structure (JSON syntax, missing headers, wrong content type).
* **422**: the request is well formed. Fix the data values.

### 16.2 Decision Examples

| Scenario | Code |
|---|---|
| JSON syntax error in body | 400 |
| Missing `Content-Type` header | 400 |
| Required field absent | 422 (FastAPI default) |
| Field present but wrong type | 422 (FastAPI default) |
| Email field present and well formed, but business rejects | 422 |
| Foreign key references non existent resource | 422 |
| Date in past when business requires future | 422 |
| HTTP method wrong | 405 |
| Body too large | 413 |
| Body Content-Type not supported | 415 |

### 16.3 The FastAPI Default

FastAPI's request validation (Pydantic based) defaults to 422 for any validation error including missing fields and type mismatches. This is somewhat more permissive than the strict RFC reading (which might suggest 400 for missing required fields).

For consistency with FastAPI conventions: use 422 for all schema validation failures. Use 400 only when the request format itself is broken (JSON parse errors).

### 16.4 The Error Body Convention

422 responses typically include detailed error breakdown:

```json
{
    "detail": [
        {
            "type": "string_too_short",
            "loc": ["body", "password"],
            "msg": "String should have at least 8 characters",
            "input": "abc"
        }
    ]
}
```

400 responses are often less structured:

```json
{
    "error": "invalid_request",
    "detail": "JSON parse error: expected ',' or '}'"
}
```

For API clients, the difference matters: 422 detail can be displayed inline next to specific form fields; 400 typically requires a generic error message.

---

## 17. THE BUBBLES THIN CONTENT REMOVAL PATTERN (JOSEPH'S 5,800 PAGE CLEANUP)

### 17.1 The Background

The thatdeveloperguy.com infrastructure overhaul included a 5,800+ thin location page cleanup. The pages were dynamically generated `/services/<city>/` templates with minimal unique content per city. Google was soft 404 flagging them and dragging down the overall site quality signal.

### 17.2 The Cleanup Approach

The pages fell into three buckets:

1. **High value cities** (Northwest Arkansas and Southwest Missouri core market): kept, content beefed up to substantive per page.
2. **Adjacent market cities** (still relevant but not core): X-Robots-Tag: noindex, follow. Users could still reach them; Google would stop indexing.
3. **Far flung cities** (irrelevant to ThatDeveloperGuy's actual service area): 410 Gone. Hard removal.

### 17.3 The Nginx Implementation

```nginx
# /etc/nginx/snippets/services-cleanup.conf

# Bucket 3: cities to 410 Gone
map $request_uri $service_410 {
    default 0;
    ~*^/services/(detroit|chicago|denver|portland|seattle)/ 1;
    # ... thousands of city slugs ...
    ~*^/services/(other-far-flung-cities)/ 1;
}

# Bucket 2: cities to noindex but keep accessible
map $request_uri $service_noindex {
    default 0;
    ~*^/services/(little-rock|fayetteville-edge|tulsa)/ 1;
    # ... adjacent market cities ...
}

# Bucket 1: core cities (no map needed, default serve)
```

In the server block:

```nginx
server {
    # ...

    location ~* ^/services/ {
        if ($service_410) {
            return 410;
        }

        if ($service_noindex) {
            add_header X-Robots-Tag "noindex, follow" always;
        }

        try_files $uri $uri/ $uri.html =404;
    }

    # Generic 410 page
    error_page 410 /410.html;
    location = /410.html {
        root /var/www/sites/thatdeveloperguy.com;
        internal;
        add_header X-Robots-Tag "noindex" always;
    }
}
```

### 17.4 The Verification Pattern

```bash
# Verify 410s are returning correctly
for url in /services/detroit /services/chicago /services/portland; do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "https://thatdeveloperguy.com$url")
    echo "$url: $STATUS"
done
# Expected: 410 for each

# Verify noindex tier
curl -sI https://thatdeveloperguy.com/services/little-rock/ | grep -i x-robots-tag
# Expected: x-robots-tag: noindex, follow

# Verify kept pages serve normally
curl -sI https://thatdeveloperguy.com/services/cassville/ | head -1
# Expected: HTTP/2 200
```

### 17.5 The Timeline Of Recovery

The 5,800 page cleanup followed approximately this timeline:

* **Week 1**: deployment of 410s and noindex. Googlebot starts encountering the new responses.
* **Week 2 to 4**: 410 URLs drop from the index rapidly (within days of the second consecutive 410). GSC "Not found (410)" count climbs initially then drops.
* **Month 2**: noindex URLs gradually deindex. The "noindex" count grows as URLs are reprocessed.
* **Month 3**: most URLs in both removal tiers are out of the index. Site quality signal improves.
* **Month 4 to 6**: rankings on the kept (substantive) pages improve as the site quality signal stabilizes.

This pattern is now documented as the canonical Bubbles approach for any client experiencing thin content soft 404 issues.

### 17.6 The Repeatable Audit Workflow

For any new Bubbles client showing soft 404 issues:

1. **Export GSC Coverage report.** Filter to "Soft 404" and "Not indexed".
2. **Classify each URL.** Keep + improve, noindex, or 410?
3. **Build the nginx map config.** Replicate the pattern from Section 17.3.
4. **Deploy and reload.** `nginx -t && systemctl reload nginx`.
5. **Submit URL removal in GSC** for the most critical 410s (URL Removals tool).
6. **Update sitemap** to exclude removed URLs.
7. **Monitor weekly.** GSC Coverage report; "Not found (410)" should rise then fall.

---

## 18. HOW MAJOR CRAWLERS REACT TO EACH 4XX CODE

Quick reference table.

| Code | Googlebot | Bingbot | ClaudeBot | GPTBot | PerplexityBot |
|---|---|---|---|---|---|
| 400 | Reduces crawl frequency over time | Treats as error | Skip | Skip | Skip |
| 401 | Access denied, reduces crawl, may deindex | Similar | Skip | Skip | Skip |
| 403 | Access denied, **backs off**, may deindex | Similar | Skip | Skip | Skip |
| 404 | Retries on slowing schedule (24h, 7d, 30d, 90d), drops slowly | Retries less | Caches negative result | Caches negative | Caches negative |
| 410 | Drops in days, retry frequency drops 92% | Treats like 404 | Caches negative result | Caches negative | Caches negative |
| 422 | Like 400 | Like 400 | Skip | Skip | Skip |
| 429 | Reduces crawl rate, honors Retry-After, **2 day rule** | Reduces crawl rate | Honors Retry-After | Honors Retry-After | Honors Retry-After |
| 451 | Treats as removal, may show "removed for legal complaint" in GSC | Similar | Removes from cache | Removes from cache | Removes from cache |

**Key implications for Bubbles SEO:**

* For bulk removal: **410** is the strongest signal (days vs months).
* For access denial of public URLs: **403** is the back off signal.
* For login walls: **401** (with WWW-Authenticate) tells crawlers "not for you".
* For rate limiting: **429** with Retry-After AND crawler whitelist (per framework-http-rate-control-headers.md).
* For legal removal: **451** is the spec compliant signal.

---

## 19. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 19.1 Default 404 with custom page

```nginx
server {
    root /var/www/sites/example.com;

    location / {
        try_files $uri $uri/ $uri.html =404;
    }

    error_page 404 /404.html;
    location = /404.html {
        root /var/www/sites/example.com;
        internal;
    }
}
```

### 19.2 Bulk 410 for content cleanup

```nginx
# In http context
map $request_uri $is_gone {
    default 0;
    ~*^/old-section/    1;
    ~*^/removed-blog/   1;
    /specific-removed-page-1 1;
    /specific-removed-page-2 1;
}

# In server block
location / {
    if ($is_gone) {
        return 410;
    }
    try_files $uri $uri/ $uri.html =404;
}

error_page 410 /410.html;
location = /410.html {
    root /var/www/sites/example.com;
    internal;
    add_header X-Robots-Tag "noindex" always;
}
```

### 19.3 Admin endpoint with 401 then 403

```python
from fastapi import Depends

async def get_current_user(request: Request):
    session = request.cookies.get("session")
    if not session:
        raise HTTPException(
            status_code=401,
            detail="auth required",
            headers={"WWW-Authenticate": 'Cookie realm="bubbles"'}
        )
    user = await load_session(session)
    if not user:
        raise HTTPException(
            status_code=401,
            detail="invalid session",
            headers={"WWW-Authenticate": 'Cookie realm="bubbles"'}
        )
    return user

@app.get("/api/admin/users")
async def admin_users(user = Depends(get_current_user)):
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="forbidden")
    return await list_all_users()
```

### 19.4 IP based access control with 403

```nginx
location /admin/ {
    allow 100.90.0.0/16;    # Tailscale
    allow 127.0.0.1;
    allow ::1;
    deny all;

    proxy_pass http://127.0.0.1:9090;
    proxy_set_header X-Real-IP $remote_addr;
}
```

### 19.5 Block scraper bots with 403

```nginx
# In http context
map $http_user_agent $is_unwanted_bot {
    default      0;
    ~*Bytespider 1;
    ~*PetalBot   1;
    ~*MJ12bot    1;
    ~*SemrushBot 1;
    ~*AhrefsBot  1;
    ~*DotBot     1;
}

# In server block
if ($is_unwanted_bot) {
    return 403;
}
```

### 19.6 Validation with 422 (FastAPI)

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import date

class CreateBooking(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    start_date: date
    end_date: date

@app.post("/api/bookings")
async def create_booking(booking: CreateBooking):
    if booking.end_date <= booking.start_date:
        raise HTTPException(
            status_code=422,
            detail=[{"loc": ["end_date"], "msg": "must be after start_date"}]
        )
    if booking.start_date < date.today():
        raise HTTPException(
            status_code=422,
            detail=[{"loc": ["start_date"], "msg": "cannot be in the past"}]
        )
    # ...
```

### 19.7 JSON parsing error returning 400

```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    # If the body is malformed JSON, return 400 instead of 422
    if any("JSONDecodeError" in str(err.get("type", "")) for err in exc.errors()):
        return JSONResponse(
            status_code=400,
            content={"error": "invalid_request", "detail": "malformed JSON body"}
        )
    return JSONResponse(
        status_code=422,
        content={"detail": exc.errors()}
    )
```

### 19.8 Rate limit with 429 and Retry-After

See framework-http-rate-control-headers.md Section 13.1. Quick reference:

```nginx
limit_req_status 429;

error_page 429 = @rate_limited;
location @rate_limited {
    internal;
    add_header Retry-After "60" always;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
    return 429 '{"error": "rate_limit_exceeded", "retry_after": 60}';
}
```

### 19.9 Legal removal with 451

```nginx
# Specific URL legally removed
location = /legally-removed-content {
    add_header Link '<https://example.com/legal/takedown/12847>; rel="blocked-by"' always;
    return 451;
}

error_page 451 /451.html;
location = /451.html {
    root /var/www/sites/example.com;
    internal;
    add_header X-Robots-Tag "noindex" always;
}
```

### 19.10 File upload size limit with 413

```nginx
location /api/upload {
    client_max_body_size 100m;
    proxy_pass http://127.0.0.1:9090;
}

# Custom 413 error page
error_page 413 /413.json;
location = /413.json {
    internal;
    add_header Content-Type "application/json" always;
    return 413 '{"error": "payload_too_large", "max_size_mb": 100}';
}
```

### 19.11 Content type enforcement with 415

```python
@app.post("/api/upload")
async def upload(request: Request):
    content_type = request.headers.get("content-type", "")
    if not content_type.startswith("multipart/form-data"):
        raise HTTPException(
            status_code=415,
            detail="content-type must be multipart/form-data"
        )
    # ...
```

### 19.12 Method enforcement with 405

```nginx
location = /api/items {
    limit_except GET POST {
        deny all;  # returns 403; nginx limit_except is different from method restriction
    }

    # For proper 405 with Allow header, handle in upstream:
    proxy_pass http://127.0.0.1:9090;
}
```

FastAPI returns 405 automatically when no endpoint matches the method:

```python
@app.get("/api/items")
async def get_items():
    return []

# PUT /api/items will get 405 Method Not Allowed
# Allow header will list: GET (and any other defined methods on this URL)
```

---

## 20. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete 4xx aware configuration for a Bubbles client site.

```nginx
# /etc/nginx/snippets/4xx-error-pages.conf
# Custom error pages and 4xx handling

error_page 404 /404.html;
error_page 410 /410.html;
error_page 429 = @rate_limited;
error_page 451 /451.html;

location = /404.html {
    root /var/www/sites/example.com;
    internal;
}

location = /410.html {
    root /var/www/sites/example.com;
    internal;
    add_header X-Robots-Tag "noindex" always;
}

location = /451.html {
    root /var/www/sites/example.com;
    internal;
    add_header X-Robots-Tag "noindex" always;
}

location @rate_limited {
    internal;
    add_header Retry-After "60" always;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
    return 429 '{"error": "rate_limit_exceeded", "retry_after": 60}';
}
```

```nginx
# /etc/nginx/sites-available/example.com

# In http context, the gone-urls map (built from the cleanup list)
# This would be in nginx.conf or an http-level snippet

http {
    map $request_uri $is_gone {
        default 0;
        ~*^/old-section/ 1;
        ~*^/removed-content/ 1;
        # ... add bulk removal URLs here ...
    }

    # Blocked bot map
    map $http_user_agent $is_blocked_bot {
        default      0;
        ~*Bytespider 1;
        ~*PetalBot   1;
        ~*MJ12bot    1;
        ~*SemrushBot 1;
    }
}

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
    include snippets/4xx-error-pages.conf;

    root /var/www/sites/example.com;
    index index.html;

    # Body size limit
    client_max_body_size 10m;

    # Block unwanted bots
    if ($is_blocked_bot) {
        return 403;
    }

    # ============= 410 GONE FOR REMOVED URLs =============
    location / {
        if ($is_gone) {
            return 410;
        }
        try_files $uri $uri/ $uri.html =404;
    }

    # ============= ADMIN (403 if not from allowed IPs) =============
    location /admin/ {
        allow 100.90.0.0/16;    # Tailscale
        allow 127.0.0.1;
        deny all;

        proxy_pass http://127.0.0.1:9090;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # ============= API (401/403 from upstream) =============
    location /api/ {
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
            return 204;
        }
        add_header X-Robots-Tag "noindex, nofollow" always;
        proxy_pass http://127.0.0.1:9090;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify each pattern:

```bash
# 404 for unknown URL
curl -sI https://example.com/does-not-exist | head -1
# Expected: HTTP/2 404

# 410 for explicitly removed URL
curl -sI https://example.com/old-section/anything | head -1
# Expected: HTTP/2 410

# 403 from blocked bot UA
curl -sI -A "Mozilla/5.0 Bytespider" https://example.com/ | head -1
# Expected: HTTP/2 403

# 403 from disallowed IP for /admin (assuming you're not on Tailscale)
curl -sI https://example.com/admin/ | head -1
# Expected: HTTP/2 403

# 401 from API (no auth)
curl -sI https://example.com/api/me | head -3
# Expected: HTTP/2 401 with WWW-Authenticate

# 413 from oversized upload
dd if=/dev/zero of=/tmp/big.bin bs=1M count=20
curl -sI -X POST --data-binary @/tmp/big.bin https://example.com/api/upload | head -1
# Expected: HTTP/2 413

# 422 from invalid data
curl -sI -X POST -H "Content-Type: application/json" \
    -d '{"email":"not-email","password":"abc"}' \
    https://example.com/api/users | head -1
# Expected: HTTP/2 422
```

---

## 21. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### 400 Bad Request

1. [ ] Used for malformed requests (JSON parse errors, missing headers).
2. [ ] Not used for "resource not found" (use 404 instead).
3. [ ] Not used for "permission denied" (use 401/403 instead).
4. [ ] Error body explains what was wrong with the request format.

### 401 Unauthorized

5. [ ] Used for unauthenticated requests to protected endpoints.
6. [ ] **WWW-Authenticate header always present on 401 responses.**
7. [ ] Realm and scheme correctly specified in WWW-Authenticate.
8. [ ] Cookie based auth uses custom Cookie scheme (not Basic, to avoid browser prompts).
9. [ ] Login pages return 401, not 200 with login form (avoids soft 404).
10. [ ] Expired tokens return 401 with `error="invalid_token"` in WWW-Authenticate.

### 403 Forbidden

11. [ ] Used for authenticated requests lacking permission.
12. [ ] Used for IP based access denial.
13. [ ] Used for blocked user agents (scraper bots).
14. [ ] Response body minimal (no information leak).
15. [ ] Admin endpoints return 403 to non admins.
16. [ ] CSRF check failures return 403.
17. [ ] **Not used in place of 401 for unauthenticated requests.**

### 404 Not Found

18. [ ] Used for URLs that do not match any resource.
19. [ ] Custom 404 page is substantive (no soft 404).
20. [ ] Custom 404 page returns 404 status (NOT 200).
21. [ ] **Deleted pages that are gone forever use 410, NOT 404.**
22. [ ] Sitemap does not include 404 URLs.
23. [ ] Internal links do not point to 404 URLs.
24. [ ] GSC "Not found (404)" count monitored.

### 410 Gone

25. [ ] Used for bulk removal of low quality content.
26. [ ] Used for permanently removed pages.
27. [ ] Used for spam/hacked content cleanup.
28. [ ] **Not used for pages that might come back.**
29. [ ] Bulk 410 map config in place if cleanup project active.
30. [ ] Custom 410 page provided.
31. [ ] Custom 410 page has `X-Robots-Tag: noindex`.
32. [ ] GSC URL Removals tool used for critical removals to accelerate.
33. [ ] GSC "Not found (410)" count tracked weekly during cleanup.

### 422 Unprocessable Entity

34. [ ] Used for validation failures on well formed requests.
35. [ ] Response body includes per field error detail.
36. [ ] FastAPI default 422 handler returning useful detail.

### 429 Too Many Requests

37. [ ] **Retry-After header always present on 429 responses.**
38. [ ] nginx `limit_req_status 429` set (not default 503).
39. [ ] Crawler whitelist in place (Googlebot, ClaudeBot, GPTBot not throttled).
40. [ ] X-Robots-Tag: noindex on rate limit error page.
41. [ ] No 429 to crawlers for more than 2 days (Google deindex rule).

### 451 Unavailable For Legal Reasons

42. [ ] Used for GDPR right to erasure compliance.
43. [ ] Used for DMCA takedown responses.
44. [ ] Optional Link header with rel="blocked-by".

### Other 4xx codes

45. [ ] 413 returned for oversized uploads (client_max_body_size configured).
46. [ ] 415 returned for unsupported Content-Type.
47. [ ] 405 returned with Allow header when method wrong.

### Cross cutting

48. [ ] All 4xx responses include `X-Robots-Tag: noindex` if the error page itself might be indexed.
49. [ ] `nginx -t` passes without warnings.
50. [ ] `nginx -T | grep "return 4"` shows expected status codes; no accidental 4xx where 2xx/3xx intended.

A site that passes all 50 has correctly configured 4xx status code usage for SEO, security, API design, and operational hygiene.

---

## 22. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: 404 used for bulk content removal.**
Symptom: removed URLs persist in Google index for 3+ months.
Why it breaks: 404 says "might come back"; Google retries for weeks.
Fix: use 410 for explicitly permanent removal. The 92% retry frequency drop after second consecutive 410 produces deindex in days.

**Pitfall 2: 401 returned without WWW-Authenticate header.**
Symptom: clients cannot complete the auth flow.
Why it breaks: RFC requires WWW-Authenticate on 401.
Fix: always include the header with scheme and realm.

**Pitfall 3: Login wall returns 200 with login form.**
Symptom: login page indexed as soft 404; account pages flagged in GSC.
Why it breaks: 200 with non substantive content is soft 404.
Fix: return 401 with WWW-Authenticate for programmatic clients; render login page directly with `X-Robots-Tag: noindex` for browsers.

**Pitfall 4: 403 used when 401 is correct.**
Symptom: legitimate users hit "forbidden" before they have a chance to authenticate.
Why it breaks: 403 means "I won't serve this regardless of credentials"; 401 means "I will if you authenticate".
Fix: 401 first; 403 only when authentication won't help.

**Pitfall 5: 401 used when 403 is correct.**
Symptom: authenticated user gets challenged again instead of denied.
Why it breaks: 401 prompts auth retry; the user is already authenticated.
Fix: 403 for "you're authenticated but lack permission".

**Pitfall 6: WWW-Authenticate uses Basic scheme on cookie auth site.**
Symptom: browser shows native auth dialog instead of using login form.
Why it breaks: Browsers see `Basic` and prompt with their own dialog.
Fix: use custom `Cookie` scheme or Bearer with appropriate realm.

**Pitfall 7: Custom 404 page returns 200.**
Symptom: thousands of "soft 404" URLs in GSC despite being deleted.
Why it breaks: page exists per the server (200), content is "not found" message.
Fix: ensure the error page handler returns 404 status, not 200.

**Pitfall 8: 410 used for a page that might come back.**
Symptom: when the page comes back, it has lost all SEO equity.
Why it breaks: 410 is a permanent commitment.
Fix: use 404 if uncertain. Reserve 410 for genuine permanent removal.

**Pitfall 9: 429 without Retry-After.**
Symptom: clients retry immediately, exacerbating the rate limit problem.
Why it breaks: without Retry-After, clients have no signal to back off.
Fix: always include Retry-After on 429.

**Pitfall 10: 429 sent to Googlebot persistently.**
Symptom: site URLs drop from Google index over 2-4 days.
Why it breaks: Google's 2 day rule: sustained 429/503 leads to index removal.
Fix: implement the crawler whitelist (framework-http-rate-control-headers.md Section 10).

**Pitfall 11: 400 used for resource not found.**
Symptom: API clients retry with same request, infinite loop.
Why it breaks: 400 suggests "fix the request"; client cannot fix.
Fix: 404 for resource not found.

**Pitfall 12: 403 error body leaks information.**
Symptom: attackers learn role names, user IDs, permission details from error responses.
Why it breaks: detailed errors enable enumeration attacks.
Fix: minimal 403 body: `{"error": "forbidden"}` is sufficient.

**Pitfall 13: 422 used where 400 is correct.**
Symptom: API clients confused about whether to fix request or data.
Why it breaks: 422 is for valid format with invalid data; 400 is for invalid format.
Fix: use 400 for JSON parse errors; 422 for validation.

**Pitfall 14: Deleted pages redirect to homepage with 301.**
Symptom: Google flags as soft 404; SEO equity lost.
Why it breaks: redirect target is irrelevant; Google detects no canonical match.
Fix: 410 for removal, or 301 to a relevant new page (not homepage).

**Pitfall 15: 451 not used for legal removals.**
Symptom: legally removed content shows 404 or 410; legal transparency missing.
Why it breaks: 451 is the spec compliant signal for legal mandates.
Fix: 451 with optional Link rel="blocked-by" for transparency.

---

## 23. DIAGNOSTIC COMMANDS

Reference of every command useful for 4xx investigation.

### Inspect specific status codes

```bash
# Check what status a URL returns
curl -sI https://example.com/some-url | head -1

# Check for 401 with WWW-Authenticate
curl -sI https://example.com/protected | grep -iE "^(HTTP|WWW-Authenticate)"

# Check for 410 vs 404 (the SEO critical pair)
for url in /old-page-1 /old-page-2 /old-page-3; do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "https://example.com$url")
    echo "$url: $STATUS"
done
```

### Bulk audit removed URLs

```bash
# Verify list of URLs all return 410
while read url; do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "https://example.com$url")
    if [ "$STATUS" != "410" ]; then
        echo "WRONG: $url returns $STATUS (expected 410)"
    fi
done < /tmp/urls-to-410.txt
```

### Status code distribution from logs

```bash
# Top status codes today
sudo awk -v d="$(date '+%d/%b/%Y')" '$0 ~ d {print $9}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -10

# 4xx specific
sudo grep -oE '" 4[0-9]{2} ' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Most common 404 URLs
sudo awk '$9 == 404 {print $7}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -20

# 403 by user agent (find what's being blocked)
sudo awk '$9 == 403 {for(i=11;i<=NF;i++) printf "%s ", $i; print ""}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -10
```

### Test auth challenge (401)

```bash
# Get the WWW-Authenticate header for a protected resource
curl -sI https://api.example.com/private | grep -iE "^(HTTP|WWW-Authenticate)"

# Test with invalid token
curl -sI -H "Authorization: Bearer invalid_token" https://api.example.com/private | \
    grep -iE "^(HTTP|WWW-Authenticate)"
# Expected: 401 with WWW-Authenticate: Bearer realm="api", error="invalid_token"

# Test with valid token
curl -sI -H "Authorization: Bearer $VALID_TOKEN" https://api.example.com/private | head -1
# Expected: HTTP/2 200
```

### Test 422 validation responses

```bash
# Invalid email
curl -s -X POST -H "Content-Type: application/json" \
    -d '{"email":"not-an-email","password":"validpass123"}' \
    https://api.example.com/users | python3 -m json.tool

# Missing required field
curl -s -X POST -H "Content-Type: application/json" \
    -d '{"email":"valid@example.com"}' \
    https://api.example.com/users | python3 -m json.tool
```

### Find soft 404 candidates (200 returned for what should be 4xx)

```bash
# URLs returning 200 with very thin body (potential soft 404)
sudo awk '$9 == 200 && $10 < 500 {print $7, $10}' /var/log/nginx/access.log | \
    sort -u | head -20

# Cross check with sitemap
curl -s https://example.com/sitemap.xml | \
    grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | \
    while read url; do
        SIZE=$(curl -so /dev/null -w "%{size_download}" "$url")
        if [ "$SIZE" -lt 500 ]; then
            echo "THIN: $url ($SIZE bytes)"
        fi
    done
```

### Server side investigation

```bash
# Find all explicit 4xx returns in nginx config
nginx -T 2>/dev/null | grep -E "return 4[0-9]{2}"

# Find all error_page directives
nginx -T 2>/dev/null | grep "error_page"

# Find the gone URL map
nginx -T 2>/dev/null | grep -A 20 "map.*is_gone"

# Check for limit_req_status
nginx -T 2>/dev/null | grep "limit_req_status"
# Expected: limit_req_status 429;

# Apply changes
nginx -t && systemctl reload nginx
```

### GSC monitoring (manual but critical)

In Google Search Console:

1. **Page Indexing report**: track "Not found (404)", "Not found (410)", "Soft 404" counts week over week.
2. **URL Removals tool**: for critical 410 removals, submit removal request to accelerate.
3. **Index Coverage**: monitor "Valid" count to confirm legitimate pages are not being removed.

---

## 24. CROSS-REFERENCES

* [framework-http-2xx-status-codes.md](framework-http-2xx-status-codes.md): the soft 404 trap (200 with thin content) is the upstream of many 4xx misuses.
* [framework-http-3xx-status-codes.md](framework-http-3xx-status-codes.md): 301 redirect to relevant new page is the alternative to 410 for moved content.
* [framework-http-caching-headers.md](framework-http-caching-headers.md): 4xx responses should generally have `Cache-Control: no-store` to avoid serving stale errors.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): `X-Robots-Tag: noindex` on 4xx error pages prevents accidental indexing.
* [framework-http-security-headers.md](framework-http-security-headers.md): security headers apply to 4xx responses; auth interaction.
* [framework-http-rate-control-headers.md](framework-http-rate-control-headers.md): the full 429 treatment including crawler whitelist and the 2-day Google deindex rule.
* [framework-http-request-headers.md](framework-http-request-headers.md): `Authorization` request header triggers 401/403 decisions; `Host` header missing or mismatched can cause 400.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Soft 404 prevention and bulk cleanup are core SEO hygiene topics.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including the thin content cleanup pattern.
* [framework-http-5xx-status-codes.md] (next in series): 500, 502, 503, 504.
* Google HTTP status codes documentation: https://developers.google.com/search/docs/crawling-indexing/http-network-errors
* Google URL Removals tool: https://search.google.com/search-console/remove-outdated-content
* MDN 400: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/400
* MDN 401: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/401
* MDN 403: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/403
* MDN 404: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/404
* MDN 410: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/410
* MDN 422: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/422
* MDN 429: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
* MDN 451: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/451
* RFC 9110 (HTTP Semantics): https://www.rfc-editor.org/rfc/rfc9110
* RFC 7725 (451 Unavailable For Legal Reasons): https://www.rfc-editor.org/rfc/rfc7725
* RFC 6585 (Additional HTTP Status Codes, 429): https://www.rfc-editor.org/rfc/rfc6585

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Status code decision tree

```
Client did something wrong. What?
   |
   |---> Request malformed (bad JSON, missing header) ........... 400
   |
   |---> Request well formed but data invalid .................... 422
   |
   |---> No auth credentials ..................................... 401 (+ WWW-Authenticate)
   |
   |---> Has auth, lacks permission .............................. 403
   |
   |---> Resource not here, might return ......................... 404
   |
   |---> Resource gone forever (bulk removal) .................... 410
   |
   |---> Client too fast ......................................... 429 (+ Retry-After)
   |
   |---> Legal mandate to block .................................. 451 (+ optional Link)
   |
   |---> Method not allowed ...................................... 405 (+ Allow)
   |
   |---> Body too large .......................................... 413
   |
   |---> Content-Type not supported .............................. 415
```

### SEO removal speed

| Code | Time to drop from Google index |
|---|---|
| 410 | Days (typically 1-2 weeks) |
| 404 | Weeks to months |
| 403 | Weeks (Googlebot backs off) |
| 451 | Days (treated as legal removal) |

### The 401 vs 403 question

**Can the client gain access by authenticating differently?**

* **YES** to 401 (with WWW-Authenticate)
* **NO** to 403

### The 404 vs 410 question

**Will this URL ever return?**

* **Maybe** to 404
* **No, never** to 410 (faster deindex)

### Five rules to memorize

1. 401 needs WWW-Authenticate. Always.
2. 429 needs Retry-After. Always.
3. 410 over 404 for permanent removal. 92% retry frequency drop after second consecutive 410.
4. 403 backs Googlebot off. 401 tells them "not for you".
5. 451 for legal removal. Optional Link rel="blocked-by".

### Five commands every operator should know

```bash
# 1. Check status of a URL
curl -sI https://example.com/url | head -1

# 2. Verify 401 has WWW-Authenticate
curl -sI https://example.com/protected | grep -iE "^(HTTP|WWW-Authenticate)"

# 3. Find 4xx hot spots in logs
sudo awk '$9 ~ /^4/ {print $9, $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# 4. Audit bulk 410 deployment
while read url; do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "https://example.com$url")
    [ "$STATUS" != "410" ] && echo "WRONG: $url ($STATUS)"
done < /tmp/urls-to-410.txt

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. 410 for removed URLs (not 404)
for url in /old-page-1 /old-page-2; do
    STATUS=$(curl -so /dev/null -w "%{http_code}" "https://example.com$url")
    [ "$STATUS" = "410" ] && echo "OK: $url is 410" || echo "FAIL: $url is $STATUS"
done

# 2. 401 has WWW-Authenticate
HAS_AUTH=$(curl -sI https://example.com/api/private | grep -i "www-authenticate" | wc -l)
[ "$HAS_AUTH" = "1" ] && echo "OK: 401 has WWW-Authenticate" || echo "FAIL: missing header"

# 3. 429 has Retry-After
HAS_RETRY=$(curl -sI -H "X-Force-Rate-Limit: 1" https://example.com/api/data 2>/dev/null | grep -i "retry-after" | wc -l)
[ "$HAS_RETRY" = "1" ] && echo "OK: 429 has Retry-After" || echo "verify rate limiting"
```

If all three pass AND GSC Page Indexing > Not found (410) drops weekly during cleanup campaigns, the 4xx layer is correctly wired.

---

End of framework-http-4xx-status-codes.md.
