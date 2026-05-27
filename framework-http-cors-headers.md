# framework-http-cors-headers.md

Comprehensive reference for the five HTTP response headers that implement Cross Origin Resource Sharing (CORS): `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Expose-Headers`, and `Access-Control-Max-Age`. Includes the companion `Access-Control-Allow-Credentials` header and the preflight OPTIONS request flow that ties all of them together. Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to framework-http-caching-headers.md, framework-http-content-headers.md, framework-http-seo-headers.md, framework-http-security-headers.md, framework-http-performance-headers.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx CORS, AI assistants generating or repairing CORS config, frontend engineers diagnosing "blocked by CORS policy" console errors, security auditors checking for permissive origin reflection vulnerabilities, and anyone troubleshooting "preflight failing", "credentials with wildcard", "custom header rejected", or "JavaScript cannot read response header" anomalies on a self hosted stack.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The CORS Request Flow Mental Model (read this first)
5. Access-Control-Allow-Origin (who may read this response)
6. Access-Control-Allow-Methods (which HTTP methods are allowed cross origin)
7. Access-Control-Allow-Headers (which request headers may be sent)
8. Access-Control-Expose-Headers (which response headers JavaScript may read)
9. Access-Control-Max-Age (how long the preflight is cached)
10. Access-Control-Allow-Credentials (the critical companion)
11. The Preflight Request Lifecycle
12. How These Headers Interact
13. Asset Class And Use Case Recipes
14. Bubbles Nginx Reference Block (paste ready)
15. Audit Checklist (50+ items)
16. Common Pitfalls
17. Diagnostic Commands (curl, browser DevTools, CORS testers)
18. Cross-References

---

## 1. DEFINITION

CORS headers tell browsers which cross origin sites are allowed to read responses from this origin and what they are allowed to do. The browser enforces the rules; the server only declares them. Without explicit CORS opt in, the browser's same origin policy blocks reading any cross origin response, even if the request itself succeeded. Defined by the WHATWG Fetch standard.

The five headers split into three concerns:

* **Permission**: `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`. They answer "may this origin read me, and may credentials accompany the request?"
* **Capability negotiation**: `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Expose-Headers`. They answer "what verbs may be used, what request headers are permitted, what response headers may JavaScript read?"
* **Performance**: `Access-Control-Max-Age`. It answers "how long may the browser cache this preflight permission grant?"

CORS is a one way conversation initiated by the browser, not a security mechanism the browser can be told to ignore. A server saying "allow everyone" does not make CORS less secure; it makes it less restrictive. The browser still enforces the boundary. The defense in depth here is on the requesting page's origin, not on the responding origin.

---

## 2. WHY IT MATTERS

Six independent pressures push correct CORS configuration from "afterthought" to "required infrastructure" in 2025 and forward.

**Modern frontends are multi origin by default.** A typical Bubbles client site might serve HTML from `example.com`, fonts from `fonts.gstatic.com`, analytics from `googletagmanager.com`, AdSense from `pagead2.googlesyndication.com`, and a FastAPI sidecar JSON endpoint from `api.example.com`. Every one of those cross origin requests requires CORS approval at the destination. Misconfigure any of them and the feature breaks silently with only a console error to debug.

**Wildcard plus credentials is a permanent footgun.** The combination `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true` is **rejected by every modern browser**. The browser refuses to honor the credentials. This breaks login flows, authenticated API calls, and CSRF tokens. The fix requires either dropping credentials or explicitly enumerating origins, often via dynamic Origin reflection (which itself is dangerous if done wrong).

**Origin reflection without validation is the most common CORS vulnerability.** "Just echo back whatever Origin the browser sent" is the lazy fix for "I need to allow many origins". It opens the door to credential theft from any attacker controlled site. Every CORS related CVE in the last five years has involved unvalidated origin reflection. Security scanners specifically test for this.

**Preflight requests double your API latency if cache is missing.** Without `Access-Control-Max-Age` (or with the default 5 seconds), every API call from a single page application triggers an OPTIONS preflight before the actual request. A 100 ms API turns into 200 ms of perceived latency. Browser cache caps vary (Chrome 7200 seconds, Firefox 86400 seconds, Safari 300 seconds), so the server should send the maximum useful value.

**Custom headers are common and easy to break.** Any frontend that adds `Authorization`, `X-Requested-With`, `X-CSRF-Token`, or any application specific header to a cross origin request triggers a preflight. The server must list those exact header names in `Access-Control-Allow-Headers`. Forgetting one breaks the entire request with a confusing error.

**JavaScript cannot read most response headers by default.** Even when CORS allows the request, the requesting JavaScript can only read seven "safe" response headers (`Cache-Control`, `Content-Language`, `Content-Type`, `Expires`, `Last-Modified`, `Pragma`, `Content-Length`) unless the server explicitly lists others in `Access-Control-Expose-Headers`. This means custom headers like `X-Total-Count` (used for pagination), `Server-Timing`, or rate limit headers are invisible to the frontend until the server opts in.

**Cost of getting it wrong.** Misconfigured CORS produces silent UX failures and serious security incidents. Real examples:

* Login form on `app.example.com` posts to `api.example.com`. API responds with `Access-Control-Allow-Origin: *` and `Access-Control-Allow-Credentials: true`. Browser refuses, login fails. Users blame the site.
* Frontend code reads `X-Total-Count` for pagination. Server sends the header but does not list it in `Access-Control-Expose-Headers`. JavaScript sees `undefined`. Pagination breaks for cross origin clients.
* Server reflects `Origin` header back as `Access-Control-Allow-Origin` without validation. Attacker on `evil.com` makes authenticated request, reads the response, exfiltrates user data.
* API call latency doubles after migration to a separate API subdomain because every request now preflights and `Max-Age` was forgotten.
* OPTIONS preflight returns 200 with no CORS headers because `if ($request_method = OPTIONS)` was used inside a location that also has `proxy_pass`. The combination is one of the nginx if pitfalls that produces unexpected behavior.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the five primary headers gets the same six part treatment:

1. **What it does**: the canonical Fetch standard definition plus the practical implication.
2. **Syntax and values**: every legal value, what it means, and when it is wrong.
3. **How to build it on Bubbles**: paste ready nginx config, plus FastAPI sidecar code where the upstream is the natural emitter.
4. **How to verify it**: curl commands that simulate preflight and actual requests.
5. **How to troubleshoot**: the four or five failure modes seen in the field and how to recognize each.
6. **How to fix common breakage**: ordered repair steps.

The companion `Access-Control-Allow-Credentials` header gets its own dedicated section because it changes the rules for several other headers. The preflight request lifecycle is documented separately because understanding it is essential for correct configuration.

---

## 4. THE CORS REQUEST FLOW MENTAL MODEL (READ THIS FIRST)

Every cross origin request in the browser runs through this decision tree. Internalize it and every CORS header decision becomes obvious.

```
JavaScript on https://app.example.com calls fetch("https://api.example.com/data", {...})
        |
        v
Browser examines the request
        |
        v
Is this a "simple" request?
   * Method is GET, HEAD, or POST
   * No custom headers beyond the safe list
   * Content-Type is text/plain, application/x-www-form-urlencoded, or multipart/form-data
   |                                 |
  YES                                NO
   |                                 |
   v                                 v
Send actual request immediately    Send OPTIONS preflight first
with Origin: https://app.example.com   with:
   |                                   - Origin: https://app.example.com
   |                                   - Access-Control-Request-Method: PUT
   |                                   - Access-Control-Request-Headers: authorization, x-csrf
   |                                 |
   |                                 v
   |                              Server responds to OPTIONS with:
   |                                 - Access-Control-Allow-Origin: <origin>
   |                                 - Access-Control-Allow-Methods: ...
   |                                 - Access-Control-Allow-Headers: ...
   |                                 - Access-Control-Max-Age: 7200
   |                                 - Status 204 No Content (or 200)
   |                                 |
   |                                 v
   |                              Browser checks:
   |                                 - Does Allow-Origin match the page's origin?
   |                                 - Is the requested method in Allow-Methods?
   |                                 - Are all requested headers in Allow-Headers?
   |                                 |             |
   |                                YES           NO
   |                                 |             |
   |                                 |             v
   |                                 |          Block actual request
   |                                 |          Console error
   |                                 v
   |                              Cache preflight result for Max-Age seconds
   |                              Send actual request
   |                                 |
   v                                 v
========================================================================
Both paths converge on the actual request and response
========================================================================
        |
        v
Server receives actual request, processes, responds
   Response includes:
   - Access-Control-Allow-Origin: <origin>
   - Access-Control-Allow-Credentials: true (if applicable)
   - Access-Control-Expose-Headers: x-total-count, server-timing (if needed)
   - Vary: Origin (if Allow-Origin is not "*")
        |
        v
Browser checks:
   - Does Allow-Origin match this page's origin?
   |                |
  YES              NO
   |                |
   v                v
JavaScript sees   Block response from JavaScript
the response      Throw TypeError: Failed to fetch
   |              (the actual response was received but is hidden)
   v
JavaScript can read safe response headers
JavaScript can read any header listed in Access-Control-Expose-Headers
JavaScript CANNOT read other headers (Set-Cookie, internal headers, etc)
```

Five rules govern the system:

1. **The browser enforces CORS, not the server.** The server only declares. If the server says "anyone may read", the browser still applies its rules to whoever asked.
2. **Preflight asks permission; actual request collects the data.** They are separate HTTP exchanges. Both must carry correct CORS response headers.
3. **Wildcard and credentials are mutually exclusive.** Browser refuses the combination. Either drop credentials or name the origin explicitly.
4. **Vary: Origin is required for dynamic origin allow lists.** Without it, intermediate caches and the browser's own cache may serve a response approved for one origin to a different origin.
5. **Custom request and response headers must be explicitly allowed and exposed.** Defaults cover only the basic web. Anything custom requires opt in by name.

A correctly configured CORS stack lets approved origins read responses fully (including custom headers), blocks unapproved origins decisively, caches preflight permissions to minimize OPTIONS overhead, and never reflects origins without validation.

---

## 5. ACCESS-CONTROL-ALLOW-ORIGIN (WHO MAY READ THIS RESPONSE)

### 5.1 What It Does

`Access-Control-Allow-Origin` tells the browser which origin (the protocol plus host plus port) is allowed to read the response from JavaScript. Without this header, the response is blocked from cross origin JavaScript regardless of HTTP status.

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Origin: null
```

This header is the foundation of CORS. Every CORS enabled response must include it. The browser compares the value against the page's origin. If they match (or the value is `*` and credentials are not in use), JavaScript may read the response. Otherwise, the response is blocked even though it was received.

### 5.2 Values

| Value | Meaning |
|---|---|
| `*` | Any origin may read. Cannot be combined with credentials. Suitable for public APIs and public assets |
| `<origin>` | A single specific origin: scheme, host, optional port. Example: `https://app.example.com`. Compatible with credentials |
| `null` | The literal string "null". Used for sandboxed iframes and file:// URLs. Rarely the right choice |

**Important: only one origin per header.** Unlike `Access-Control-Allow-Methods` (comma separated list), `Access-Control-Allow-Origin` accepts exactly one value. To support multiple origins, the server must dynamically reflect the requesting origin (with validation) or send a different value per request.

### 5.3 The Dynamic Reflection Pattern (For Multiple Allowed Origins)

The Bubbles canonical pattern for supporting multiple specific origins:

```nginx
http {
    # Define allowed origins
    map $http_origin $cors_origin {
        default                                "";
        "~^https://(app\.|admin\.)?example\.com$"    "$http_origin";
        "~^https://partner\.example\.com$"           "$http_origin";
        "~^https://staging\.example\.com$"           "$http_origin";
    }
}

server {
    location /api/ {
        # Only set the header when the origin matches
        if ($cors_origin) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Vary "Origin" always;
        }
    }
}
```

The map directive validates the origin against the allow list. If the request's `Origin` header matches one of the patterns, `$cors_origin` is set to that origin value and reflected back. If it does not match, `$cors_origin` is empty and no CORS header is added (effectively rejecting the cross origin request).

**Vary: Origin is essential.** Because the response varies based on the request's Origin header, every cache between the server and client must understand that responses are not interchangeable across origins. Without Vary, a cached response approved for `app.example.com` might be served to a browser on `evil.com`.

### 5.4 The Reflection Vulnerability

The naive version of "support multiple origins" is to reflect whatever the browser sends, with no validation:

```nginx
# DANGEROUS: do not do this
add_header Access-Control-Allow-Origin $http_origin always;
add_header Access-Control-Allow-Credentials "true" always;
```

This says "whoever you are, you may read my responses with credentials". An attacker hosting JavaScript on `evil.com` can now make authenticated requests to this API from a victim's browser and read the responses, exfiltrating any data the victim has access to. Every CORS related CVE in the last several years involves this pattern or a variant.

**The fix:** always validate the origin against an explicit allow list (Section 5.3). Never reflect blindly.

### 5.5 How To Build It On Bubbles

**Public API for any origin:**

```nginx
location /api/public/ {
    add_header Access-Control-Allow-Origin "*" always;
    # No credentials, no need for Vary
}
```

**Same origin only (effectively no CORS):**

```nginx
location /api/private/ {
    # No Access-Control-Allow-Origin header
    # Browser blocks cross origin requests by default
}
```

**Specific single origin:**

```nginx
location /api/ {
    add_header Access-Control-Allow-Origin "https://app.example.com" always;
    add_header Vary "Origin" always;
}
```

**Multiple specific origins (the Bubbles canonical pattern):**

```nginx
# In nginx.conf http context
map $http_origin $cors_origin {
    default                                              "";
    "~^https://(app\.|admin\.|api\.)?example\.com$"      "$http_origin";
    "~^https://partner\.example\.com$"                   "$http_origin";
}

# In server block location
location /api/ {
    if ($cors_origin) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Vary "Origin" always;
    }
}
```

**FastAPI sidecar with proper validation:**

```python
from fastapi import FastAPI, Request, Response
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# FastAPI's built in middleware handles the allow list correctly
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://app.example.com",
        "https://admin.example.com",
        "https://partner.example.com",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization", "X-CSRF-Token"],
    expose_headers=["X-Total-Count", "Server-Timing"],
    max_age=7200,
)
```

When using the FastAPI middleware approach, nginx should not add CORS headers itself; it would conflict with the upstream. Either nginx handles CORS OR the upstream does, not both.

### 5.6 How To Verify

```bash
# 1. Simulate a CORS request with an Origin header
curl -sI -H "Origin: https://app.example.com" https://api.example.com/data | grep -iE "access-control|^vary"

# Expected:
# access-control-allow-origin: https://app.example.com
# vary: Origin

# 2. Test from a disallowed origin
curl -sI -H "Origin: https://evil.com" https://api.example.com/data | grep -i access-control-allow-origin
# Expected: no access-control-allow-origin header

# 3. Verify wildcard public API
curl -sI -H "Origin: https://anything.com" https://api.example.com/public/data | grep -i access-control-allow-origin
# Expected: access-control-allow-origin: *

# 4. Confirm Vary: Origin when reflecting
curl -sI -H "Origin: https://app.example.com" https://api.example.com/data | grep -i vary
# Expected: vary: Origin (or contains Origin)

# 5. Test in browser DevTools
# Open DevTools Console on https://app.example.com:
# fetch("https://api.example.com/data").then(r => r.json()).then(console.log)
# Network tab: check Access-Control-Allow-Origin in response
# Console: should NOT show "blocked by CORS policy"
```

### 5.7 Troubleshooting

**Symptom: Console error "blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present".**
Causes ranked by frequency:
1. The header is not being set. Verify with curl.
2. The header is being set on the actual request response but not on the preflight OPTIONS response. Both must have it. See Section 11.
3. The origin sent by the browser does not match the allow list (typo, missing port, http vs https mismatch).
4. An upstream is overriding the value back to empty.

**Symptom: Header present in curl but browser still shows CORS error.**
1. Browser hard cached a previous failed preflight. Clear cache or test in incognito.
2. The actual request returned a non 2xx status. Browser treats it as a CORS failure even if the headers are correct, in some cases.
3. The Origin value in the curl test does not match what the browser sends. Verify by looking at the Origin in DevTools Network tab.

**Symptom: "The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '*' when the request's credentials mode is 'include'".**
Wildcard plus credentials is rejected. See Section 10.

**Symptom: Origin reflection returns wrong value (browser blocks).**
1. The reflection is happening but with the wrong case (`origin` vs `Origin` in the regex).
2. The map directive does not match. Test the regex with curl against a sample origin.
3. The `if ($cors_origin)` check is inside a nested location that does not see the map variable. Move to the outer scope.

**Symptom: Cache serves response approved for origin A to a request from origin B.**
`Vary: Origin` is missing. Add it whenever `Access-Control-Allow-Origin` is anything other than `*`.

### 5.8 How To Fix Common Breakage

**Case: Login fails with "wildcard plus credentials" error.**
Replace wildcard with specific origin reflection:

```nginx
map $http_origin $cors_origin {
    default                            "";
    "~^https://app\.example\.com$"     "$http_origin";
}

location /api/auth/ {
    if ($cors_origin) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Vary "Origin" always;
    }
}
```

**Case: New partner integration needs API access.**
Add their origin to the map:

```nginx
map $http_origin $cors_origin {
    default                                            "";
    "~^https://(app\.|admin\.)?example\.com$"          "$http_origin";
    "~^https://partner\.example\.com$"                 "$http_origin";
    "~^https://new-partner\.example\.com$"             "$http_origin";  # added
}
```

`nginx -t && systemctl reload nginx`. The new partner can immediately make CORS requests.

**Case: Security scan flags origin reflection without validation.**
The current config uses `$http_origin` directly. Replace with a map based allow list (Section 5.3 and 5.5).

---

## 6. ACCESS-CONTROL-ALLOW-METHODS (WHICH HTTP METHODS ARE ALLOWED CROSS ORIGIN)

### 6.1 What It Does

`Access-Control-Allow-Methods` lists which HTTP methods the actual request may use. Sent in the preflight response. The browser checks the requested method (announced via `Access-Control-Request-Method`) against this list and blocks the actual request if absent.

```
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
Access-Control-Allow-Methods: *
```

This header only matters on preflight (OPTIONS) responses. It is ignored on actual request responses.

### 6.2 Values

The value is a comma separated list of HTTP method names.

| Value | Meaning |
|---|---|
| `GET, POST, PUT, DELETE, PATCH, OPTIONS` | Specific allowed methods. Comma separated |
| `*` | Any method. Cannot be combined with credentials. Recently supported in most browsers |
| (absent) | No methods explicitly allowed. The browser may still permit "simple" methods (GET, HEAD, POST) |

**Always include OPTIONS** if you support preflight, even though OPTIONS is the preflight method itself. Some browsers expect it in the allow list.

### 6.3 How To Build It On Bubbles

For a typical REST API:

```nginx
location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token" always;
        add_header Access-Control-Max-Age "7200" always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Vary "Origin" always;
        return 204;
    }
    # actual request handling
    proxy_pass http://127.0.0.1:9090;
}
```

The `if ($request_method = OPTIONS)` pattern is **one of the explicitly safe uses** of nginx `if` because it combines `if` with `return`, not with `proxy_pass`. The nginx `if` antipattern warning applies only to combinations with content handlers.

For a read only public API:

```nginx
location /api/public/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin "*" always;
        add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS" always;
        add_header Access-Control-Max-Age "86400" always;
        return 204;
    }
}
```

### 6.4 How To Verify

```bash
# 1. Simulate a preflight
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: PUT" \
     -H "Access-Control-Request-Headers: Authorization" \
     https://api.example.com/data

# Expected response:
# HTTP/2 204
# access-control-allow-origin: https://app.example.com
# access-control-allow-methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
# access-control-allow-headers: Content-Type, Authorization, X-CSRF-Token
# access-control-max-age: 7200

# 2. Verify PUT is allowed
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: PUT" \
     https://api.example.com/data | grep -i access-control-allow-methods
# Look for PUT in the response

# 3. Test that DELETE works (after preflight passes)
curl -X DELETE -H "Origin: https://app.example.com" https://api.example.com/data/123 | head
```

### 6.5 Troubleshooting

**Symptom: "Method PUT is not allowed by Access-Control-Allow-Methods".**
The OPTIONS preflight response does not include PUT in the allow list.
Fix: add the method to the list.

**Symptom: Methods listed but browser still blocks.**
1. The methods are listed on the actual request response, not the preflight. Add to OPTIONS specifically.
2. The list contains a typo or wrong case.
3. The method is allowed but a required header is not (check Allow-Headers separately).

**Symptom: OPTIONS returns 405 Method Not Allowed.**
The location does not handle OPTIONS. Add the `if ($request_method = OPTIONS)` block as shown in Section 6.3.

### 6.6 How To Fix Common Breakage

**Case: Need to add PATCH method for partial updates.**
Add PATCH to the list:

```nginx
add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
```

Reload nginx. Browsers cache preflights, so existing tabs may need a refresh.

**Case: API endpoint is read only.**
Restrict to read methods:

```nginx
add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS" always;
```

---

## 7. ACCESS-CONTROL-ALLOW-HEADERS (WHICH REQUEST HEADERS MAY BE SENT)

### 7.1 What It Does

`Access-Control-Allow-Headers` lists which request headers the actual request may include. Sent in the preflight response. The browser checks the requested headers (announced via `Access-Control-Request-Headers`) against this list and blocks the actual request if any are not allowed.

```
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-Token, X-Requested-With
Access-Control-Allow-Headers: *
```

### 7.2 The Always Allowed Headers

These headers are always allowed without being listed (they belong to the "CORS safelisted request headers"):

* `Accept`
* `Accept-Language`
* `Content-Language`
* `Content-Type` (only with values `text/plain`, `application/x-www-form-urlencoded`, or `multipart/form-data`)
* `Range`

Any other request header (`Authorization`, `Content-Type: application/json`, `X-CSRF-Token`, etc) triggers a preflight and must appear in `Access-Control-Allow-Headers`.

### 7.3 Values

The value is a comma separated list of header names. Case insensitive.

| Value | Meaning |
|---|---|
| `Content-Type, Authorization` | Specific allowed custom request headers |
| `*` | Any header. Cannot be combined with credentials (since 2021+) |
| (absent) | Only the safelisted headers allowed |

### 7.4 How To Build It On Bubbles

For a typical API with Authorization and CSRF protection:

```nginx
location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token, X-Requested-With" always;
        # ... other CORS preflight headers ...
        return 204;
    }
}
```

For an API that accepts custom application headers:

```nginx
location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token, X-Tenant-ID, X-Request-ID, X-Client-Version" always;
        return 204;
    }
}
```

### 7.5 How To Verify

```bash
# 1. Simulate preflight requesting custom headers
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: authorization, content-type, x-csrf-token" \
     https://api.example.com/data | grep -i access-control-allow-headers

# Expected: access-control-allow-headers: Content-Type, Authorization, X-CSRF-Token

# 2. Test with a header NOT in the list
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: x-secret-header" \
     https://api.example.com/data

# Browser would block; nginx still returns 204 with the configured allow list
# (browser does the enforcement, not nginx)
```

### 7.6 Troubleshooting

**Symptom: "Request header field X-Custom-Header is not allowed by Access-Control-Allow-Headers in preflight response".**
The header is not in the allow list. Add it.

**Symptom: Authorization header rejected despite being listed.**
1. Header name typo. Check exact spelling (case insensitive but spelling matters).
2. The preflight is going to a different location than the actual request. Both must have CORS headers.
3. An intermediate is stripping the header before it reaches the configured CORS handler.

**Symptom: Browser sends preflight even for simple POST.**
The Content-Type is `application/json` (which is not in the safelisted list). This is correct browser behavior. Configure preflight handling.

### 7.7 How To Fix Common Breakage

**Case: Adding a new custom header to API requests.**
Add it to the allow list:

```nginx
add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token, X-New-Header" always;
```

Reload nginx. Browsers with cached preflights will preflight again on next request after cache expires.

**Case: Want to be permissive about headers (development/internal API).**
Use the wildcard if credentials are not required:

```nginx
add_header Access-Control-Allow-Headers "*" always;
```

Or enumerate liberally. Note: with credentials enabled, wildcard does not work; you must list every header.

---

## 8. ACCESS-CONTROL-EXPOSE-HEADERS (WHICH RESPONSE HEADERS JAVASCRIPT MAY READ)

### 8.1 What It Does

`Access-Control-Expose-Headers` lists which response headers the requesting JavaScript may read via the Fetch API or XMLHttpRequest. Without this header, only the "safelisted" response headers are visible to JavaScript. Sent on the actual response (not preflight).

```
Access-Control-Expose-Headers: X-Total-Count
Access-Control-Expose-Headers: X-Total-Count, X-RateLimit-Limit, X-RateLimit-Remaining, Server-Timing
Access-Control-Expose-Headers: *
```

### 8.2 The Always Visible Headers

These response headers are always readable from JavaScript on a CORS response:

* `Cache-Control`
* `Content-Language`
* `Content-Length`
* `Content-Type`
* `Expires`
* `Last-Modified`
* `Pragma`

**Everything else is invisible to JavaScript unless listed in Expose-Headers.** Common headers that surprise developers by being invisible:

* `Authorization` (response side)
* `Location` (after a 3xx redirect)
* `Set-Cookie` (always invisible to script; this is a security feature, cannot be exposed)
* `X-Total-Count` and any pagination headers
* `X-RateLimit-*` headers
* `Server-Timing`
* `ETag` (counterintuitively, this is NOT in the safelist; must be exposed)
* `Link`
* Custom application headers

### 8.3 Values

| Value | Meaning |
|---|---|
| `<header>, <header>` | Specific headers JavaScript may read |
| `*` | Any header. Cannot be combined with credentials |
| (absent) | Only safelisted headers visible |

### 8.4 How To Build It On Bubbles

For pagination plus rate limiting plus Server-Timing:

```nginx
location /api/ {
    if ($request_method = OPTIONS) {
        # Preflight handling
        return 204;
    }

    # Actual request: expose custom response headers
    add_header Access-Control-Allow-Origin $cors_origin always;
    add_header Access-Control-Expose-Headers "X-Total-Count, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, Server-Timing, ETag, Link" always;
    add_header Vary "Origin" always;
    add_header Access-Control-Allow-Credentials "true" always;

    proxy_pass http://127.0.0.1:9090;
}
```

For a FastAPI sidecar with CORS middleware:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    expose_headers=[
        "X-Total-Count",
        "X-RateLimit-Limit",
        "X-RateLimit-Remaining",
        "X-RateLimit-Reset",
        "Server-Timing",
        "ETag",
        "Link",
    ],
    max_age=7200,
)
```

### 8.5 How To Verify

```bash
# 1. Confirm header is on actual responses
curl -sI -H "Origin: https://app.example.com" https://api.example.com/users | grep -iE "access-control-expose|x-total"

# Expected:
# access-control-expose-headers: X-Total-Count, Server-Timing, ETag
# x-total-count: 1247

# 2. Test in browser DevTools
# Console on https://app.example.com:
# fetch("https://api.example.com/users", {credentials: "include"})
#   .then(r => {
#     console.log("Total:", r.headers.get("X-Total-Count"));
#     console.log("Rate limit:", r.headers.get("X-RateLimit-Remaining"));
#   })
# Without Expose-Headers: both show null
# With Expose-Headers listing them: both show actual values
```

### 8.6 Troubleshooting

**Symptom: JavaScript reads response.headers.get("X-Total-Count") and gets null.**
The header is on the response but not exposed. Add it to `Access-Control-Expose-Headers`.

**Symptom: ETag returns null in JavaScript despite being in response.**
ETag is NOT in the default safelist (despite being common). Must be explicitly exposed.

**Symptom: Set-Cookie cannot be read from JavaScript.**
By design. The browser never exposes Set-Cookie to script regardless of CORS configuration. This is a fundamental security boundary, not a CORS issue.

**Symptom: Expose-Headers wildcard does not work.**
1. The wildcard is being used with credentials (not allowed). Drop credentials or enumerate headers.
2. Older browser (some still treat `*` as a literal header name in this context). Enumerate for safety.

### 8.7 How To Fix Common Breakage

**Case: Pagination component cannot read total count.**
The server sends `X-Total-Count: 1247` but JavaScript sees null. Add to expose list:

```nginx
add_header Access-Control-Expose-Headers "X-Total-Count, Link" always;
```

Frontend code now reads `response.headers.get("X-Total-Count")` successfully.

**Case: RUM library cannot read Server-Timing.**
Same fix: add `Server-Timing` to the expose list.

---

## 9. ACCESS-CONTROL-MAX-AGE (HOW LONG THE PREFLIGHT IS CACHED)

### 9.1 What It Does

`Access-Control-Max-Age` tells the browser how long to cache the preflight (OPTIONS) response. While the cache is valid, the browser skips the preflight and sends the actual request directly. Sent in the OPTIONS preflight response.

```
Access-Control-Max-Age: 7200
Access-Control-Max-Age: 86400
```

Without this header, browsers apply a default of 5 seconds, which means every API call from a single page application triggers a preflight after the cache expires. This doubles perceived latency for every cross origin request.

### 9.2 Browser Caps

Browsers enforce their own upper limits, capping whatever the server sends:

| Browser | Max value honored |
|---|---|
| Firefox | 86400 (24 hours) |
| Chromium (v76+) | 7200 (2 hours) |
| Safari | 600 (10 minutes) |

For maximum benefit across all browsers, send `Access-Control-Max-Age: 86400`. Firefox uses the full day, Chrome caps to 2 hours, Safari to 10 minutes. The server side cost of being generous is zero.

### 9.3 What The Preflight Cache Stores

The cached entry is keyed on:

* URL (or the request path).
* Origin (the requesting page's origin).
* Combination of `Access-Control-Request-Method` and `Access-Control-Request-Headers` from the preflight.

A change in any of these triggers a fresh preflight. So a single page that makes both GET and POST requests with different headers will preflight twice.

### 9.4 How To Build It On Bubbles

In the preflight handler:

```nginx
location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token" always;
        add_header Access-Control-Max-Age "86400" always;  # 24 hours, browsers cap as needed
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Vary "Origin" always;
        return 204;
    }
}
```

For a high traffic API, Max-Age is one of the cheapest performance wins available.

### 9.5 How To Verify

```bash
# 1. Confirm Max-Age in preflight response
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     https://api.example.com/data | grep -i access-control-max-age
# Expected: access-control-max-age: 86400

# 2. Verify browser caches the preflight
# In Chrome DevTools Network panel:
# Make a cross origin request, observe OPTIONS preflight
# Make the same request again within 2 hours, observe NO OPTIONS preflight
# After 2 hours, observe preflight again

# 3. Test that changing the method invalidates cache
# OPTIONS for GET cached
# Then POST: new OPTIONS preflight (different method)
```

### 9.6 Troubleshooting

**Symptom: Every API call triggers an OPTIONS preflight despite Max-Age.**
1. Max-Age is missing or set to 0. Check the preflight response.
2. The request varies (different method, different custom header each time). Each unique combination preflights separately.
3. Browser cache cleared between requests.
4. The page was opened in a private/incognito window with disabled cache.

**Symptom: Preflight cache works in Firefox but not Chrome.**
Chrome caps Max-Age at 7200 (2 hours). After 2 hours, preflight happens again regardless of server value.

**Symptom: Preflight cache works for one URL but not for similar URLs.**
The cache is per URL (or per path depending on implementation). Two endpoints that share the same allow list still each get their own cached preflight.

### 9.7 How To Fix Common Breakage

**Case: Single page app feels slow on cross origin API calls.**
Confirm Max-Age is set:

```bash
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     https://api.example.com/users | grep -i max-age
```

If missing or low, set to 86400:

```nginx
add_header Access-Control-Max-Age "86400" always;
```

API latency on cached preflights drops by half (one round trip eliminated per request).

---

## 10. ACCESS-CONTROL-ALLOW-CREDENTIALS (THE CRITICAL COMPANION)

### 10.1 What It Does

`Access-Control-Allow-Credentials` tells the browser that the response is allowed to be read even when the request was made with credentials (cookies, HTTP authentication, client TLS certificates). Without this header (or with value `false`), JavaScript that made a `fetch(url, {credentials: "include"})` call cannot read the response, and the browser refuses to send cookies in the first place.

```
Access-Control-Allow-Credentials: true
```

Only `true` is a meaningful value. Setting to `false` is equivalent to omitting.

### 10.2 The Wildcard Prohibition

When `Access-Control-Allow-Credentials: true` is set, **all CORS headers must use specific values, not wildcards**. The browser strictly enforces this:

| Header | With credentials | Without credentials |
|---|---|---|
| Access-Control-Allow-Origin | Specific origin only | Specific or `*` |
| Access-Control-Allow-Methods | Specific methods only | Specific or `*` |
| Access-Control-Allow-Headers | Specific headers only | Specific or `*` |
| Access-Control-Expose-Headers | Specific headers only | Specific or `*` |

Combining `*` with `true` causes the browser to refuse the response entirely with an error.

### 10.3 When To Use Credentials

* **Use credentials** when the API requires authentication via cookies or HTTP auth and the request comes from a browser based application on a different origin.
* **Do not use credentials** for public APIs, third party widgets, or anonymous data endpoints.

The browser only sends credentials when the calling code explicitly requests them:

```javascript
fetch("https://api.example.com/data", { credentials: "include" })
```

Without `credentials: "include"`, the browser does not send cookies and does not require `Access-Control-Allow-Credentials` from the server.

### 10.4 How To Build It On Bubbles

For an authenticated API used by a frontend on a different origin:

```nginx
map $http_origin $cors_origin {
    default                                  "";
    "~^https://app\.example\.com$"           "$http_origin";
}

location /api/ {
    if ($request_method = OPTIONS) {
        if ($cors_origin) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Credentials "true" always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token" always;
            add_header Access-Control-Max-Age "86400" always;
            add_header Vary "Origin" always;
            return 204;
        }
        return 204;  # generic OPTIONS response for non allow listed origins
    }

    # Actual request
    if ($cors_origin) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Vary "Origin" always;
    }
    proxy_pass http://127.0.0.1:9090;
}
```

### 10.5 How To Verify

```bash
# 1. Confirm Allow-Credentials on both preflight and actual
curl -sI -X OPTIONS -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     https://api.example.com/data | grep -i access-control-allow-credentials

curl -sI -H "Origin: https://app.example.com" \
     https://api.example.com/data | grep -i access-control-allow-credentials

# Both should show: access-control-allow-credentials: true

# 2. Verify wildcard is NOT used
curl -sI -H "Origin: https://app.example.com" \
     https://api.example.com/data | grep -i access-control-allow-origin
# Should show specific origin, NOT *

# 3. Test in browser with credentials
# Console on https://app.example.com:
# fetch("https://api.example.com/data", {credentials: "include"})
#   .then(r => console.log(r.status))
# Should succeed and return JSON
```

### 10.6 Troubleshooting

**Symptom: "The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '*' when the request's credentials mode is 'include'".**
Server is sending `*` for origin while also setting credentials. Replace wildcard with specific origin reflection.

**Symptom: Cookie not sent on cross origin request despite credentials: include.**
1. Server is not sending `Access-Control-Allow-Credentials: true`.
2. Cookie does not have `SameSite=None; Secure` attribute (required for cross site cookies in modern browsers).
3. Site is on HTTP, not HTTPS (SameSite=None requires Secure which requires HTTPS).

**Symptom: Set-Cookie response header sent but browser does not store cookie.**
1. Same as above (SameSite=None; Secure required).
2. Server's `Access-Control-Allow-Credentials` is missing or false.
3. Cookie's Domain attribute does not include the requesting origin.

### 10.7 How To Fix Common Breakage

**Case: Cross origin login fails to set session cookie.**
Three things must align:

1. Server sets `Access-Control-Allow-Credentials: true` and specific Allow-Origin.
2. Cookie is set with `SameSite=None; Secure`.
3. Frontend uses `credentials: "include"` in fetch.

```python
# FastAPI example
from fastapi.responses import JSONResponse

@app.post("/api/login")
async def login(...):
    response = JSONResponse({"status": "ok"})
    response.set_cookie(
        key="session",
        value=session_token,
        max_age=3600,
        secure=True,        # required for SameSite=None
        httponly=True,
        samesite="none",    # required for cross site
    )
    return response
```

```javascript
// Frontend
fetch("https://api.example.com/login", {
    method: "POST",
    credentials: "include",  // critical
    body: JSON.stringify({...})
});
```

```nginx
# Nginx
location /api/login {
    if ($cors_origin) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Vary "Origin" always;
    }
    proxy_pass http://127.0.0.1:9090;
}
```

---

## 11. THE PREFLIGHT REQUEST LIFECYCLE

The OPTIONS preflight is the single most misunderstood part of CORS. A full walkthrough:

### 11.1 When Does A Preflight Happen

A preflight is triggered for any "non simple" request. A simple request meets ALL of these:

* Method is GET, HEAD, or POST (other methods preflight).
* Request headers are limited to the safelisted set (custom headers preflight).
* If POST, Content-Type is `text/plain`, `application/x-www-form-urlencoded`, or `multipart/form-data` (`application/json` preflights).

Practically, **almost every modern API call triggers a preflight** because:

* JSON APIs use `Content-Type: application/json` (not safelisted).
* Authenticated APIs use `Authorization` header (not safelisted).
* REST APIs use PUT, PATCH, DELETE (not safelisted methods).

### 11.2 The Preflight Request

The browser sends an OPTIONS request with these special headers:

```
OPTIONS /api/users/123 HTTP/2
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: authorization, content-type
```

* `Origin`: which origin is asking.
* `Access-Control-Request-Method`: which method the actual request will use.
* `Access-Control-Request-Headers`: which custom headers the actual request will include.

### 11.3 The Preflight Response

The server must respond with the CORS headers granting permission:

```
HTTP/2 204
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-Token
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 7200
Vary: Origin
```

Status 204 (No Content) is preferred. 200 also works. The response body is ignored.

### 11.4 What The Browser Does Next

1. **Match check.** Browser verifies:
   * `Allow-Origin` matches the page's origin (or is `*` without credentials).
   * Requested method is in `Allow-Methods`.
   * All requested headers are in `Allow-Headers`.

2. **If all match:** browser caches the result for `Max-Age` seconds, then sends the actual request.

3. **If any does not match:** browser blocks the actual request, fires a CORS error, and the JavaScript catches a `TypeError: Failed to fetch`. No actual request is sent.

### 11.5 The Actual Request

If preflight passed, the browser sends the actual request:

```
PUT /api/users/123 HTTP/2
Host: api.example.com
Origin: https://app.example.com
Authorization: Bearer abc...
Content-Type: application/json

{"name": "Joseph"}
```

Note: `Origin` is included on the actual request too, not just preflight.

The server processes the request and returns the response with CORS headers (the response side ones, not the preflight ones):

```
HTTP/2 200
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: X-Total-Count, ETag
Vary: Origin
Content-Type: application/json
ETag: "abc123"
X-Total-Count: 1
...
```

### 11.6 The Nginx Pattern

The full nginx CORS handler combining preflight and actual:

```nginx
location /api/ {
    # Preflight handling
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token" always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Max-Age "86400" always;
        add_header Vary "Origin" always;
        add_header Content-Length 0;
        add_header Content-Type "text/plain charset=UTF-8";
        return 204;
    }

    # Actual request handling
    if ($cors_origin) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Expose-Headers "X-Total-Count, Server-Timing, ETag" always;
        add_header Vary "Origin" always;
    }
    proxy_pass http://127.0.0.1:9090;
}
```

The `if` for OPTIONS plus `return` is **safe**. The `if` for `$cors_origin` followed by `add_header` plus `proxy_pass` is also safe because `if` here only conditions on `add_header`, not on `proxy_pass`. The dangerous combination is `if` with content handlers like `proxy_pass` directly inside the `if` block.

### 11.7 How To Verify The Whole Flow

```bash
# Step 1: simulate preflight
echo "=== PREFLIGHT ==="
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: PUT" \
     -H "Access-Control-Request-Headers: authorization, content-type" \
     https://api.example.com/users/123

# Expected response with all preflight CORS headers, status 204

# Step 2: simulate actual request
echo "=== ACTUAL ==="
curl -sI -X PUT \
     -H "Origin: https://app.example.com" \
     -H "Authorization: Bearer test" \
     -H "Content-Type: application/json" \
     https://api.example.com/users/123

# Expected response with actual response CORS headers
```

---

## 12. HOW THESE HEADERS INTERACT

The CORS headers form a coherent permission grant. Several specific interactions matter.

### 12.1 The Wildcard And Credentials Mutual Exclusion

The single most important rule. With `Access-Control-Allow-Credentials: true`, all of these must be specific:

* `Access-Control-Allow-Origin`: cannot be `*`.
* `Access-Control-Allow-Methods`: cannot be `*`.
* `Access-Control-Allow-Headers`: cannot be `*`.
* `Access-Control-Expose-Headers`: cannot be `*`.

Without credentials, wildcards work everywhere.

### 12.2 Vary: Origin Is Required For Dynamic Origin

Whenever `Access-Control-Allow-Origin` is set based on the request's Origin header (not hardcoded), `Vary: Origin` must be set so caches do not cross contaminate.

```nginx
if ($cors_origin) {
    add_header Access-Control-Allow-Origin $cors_origin always;
    add_header Vary "Origin" always;  # ESSENTIAL
}
```

### 12.3 Preflight Headers vs Actual Response Headers

The preflight response and the actual response carry different sets of CORS headers:

| Header | On preflight (OPTIONS) | On actual response |
|---|---|---|
| Access-Control-Allow-Origin | Yes | Yes |
| Access-Control-Allow-Credentials | Yes | Yes |
| Access-Control-Allow-Methods | Yes | (ignored) |
| Access-Control-Allow-Headers | Yes | (ignored) |
| Access-Control-Max-Age | Yes | (ignored) |
| Access-Control-Expose-Headers | (ignored) | Yes |
| Vary: Origin | Yes | Yes |

Some headers matter only on preflight, some only on actual. Setting them all on both responses is harmless and simpler to configure.

### 12.4 Cache-Control On Preflight Responses

The browser caches the preflight response according to `Max-Age`. Browser cache may also be governed by `Cache-Control` for some implementations. Set both:

```nginx
if ($request_method = OPTIONS) {
    add_header Access-Control-Max-Age "86400" always;
    add_header Cache-Control "public, max-age=86400" always;
    return 204;
}
```

### 12.5 Authorization Header And Preflight

Any request with `Authorization` header triggers a preflight because Authorization is not safelisted. This means JWT based API calls always preflight. Combine with `Max-Age` to mitigate.

### 12.6 CORS And Service Workers

Service workers can intercept fetch requests and modify CORS behavior. If the page has a service worker, debugging CORS gets harder because the network panel may show different headers than the actual server response. Unregister the service worker during CORS debugging.

---

## 13. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 13.1 Public read only API (no credentials)

```nginx
location /api/public/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin "*" always;
        add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS" always;
        add_header Access-Control-Max-Age "86400" always;
        return 204;
    }

    add_header Access-Control-Allow-Origin "*" always;
    proxy_pass http://127.0.0.1:9090;
}
```

### 13.2 Authenticated API with specific frontend origin

```nginx
# http context
map $http_origin $cors_origin {
    default                                  "";
    "~^https://app\.example\.com$"           "$http_origin";
}

# server context
location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token" always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Max-Age "86400" always;
        add_header Vary "Origin" always;
        return 204;
    }

    if ($cors_origin) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Expose-Headers "X-Total-Count, Server-Timing, ETag, Link" always;
        add_header Vary "Origin" always;
    }
    proxy_pass http://127.0.0.1:9090;
}
```

### 13.3 Multi tenant API with several frontend origins

```nginx
map $http_origin $cors_origin {
    default                                                "";
    "~^https://(app\.|admin\.|partner\.)?example\.com$"    "$http_origin";
    "~^https://staging\.example\.com$"                     "$http_origin";
    "~^https://localhost(:[0-9]+)?$"                       "$http_origin";  # dev only
}
```

### 13.4 Embeddable widget for any partner

```nginx
location /widget/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin "*" always;
        add_header Access-Control-Allow-Methods "GET, OPTIONS" always;
        add_header Access-Control-Max-Age "86400" always;
        return 204;
    }

    add_header Access-Control-Allow-Origin "*" always;
    add_header Access-Control-Expose-Headers "X-Widget-Version" always;
    # No credentials, so wildcard origin is fine
}
```

### 13.5 Web fonts on a CDN subdomain

```nginx
server {
    server_name fonts.example.com;

    location ~* \.(woff2|woff|ttf|otf)$ {
        add_header Access-Control-Allow-Origin "*" always;
        add_header Cross-Origin-Resource-Policy "cross-origin" always;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }
}
```

Fonts loaded from a different origin require CORS even though they are not "scripted". The `<link rel="preload" as="font">` element and `@font-face` rules both invoke CORS mode.

### 13.6 Images served from a different origin

```nginx
location ~* \.(jpg|jpeg|png|webp|avif)$ {
    add_header Access-Control-Allow-Origin "*" always;
    add_header Cache-Control "public, max-age=2592000" always;
}
```

Required if the page uses `crossorigin` attribute on `<img>` or if the image is accessed via `<canvas>` (canvas tainting).

### 13.7 Same origin only (no CORS at all)

```nginx
location /api/internal/ {
    # No CORS headers
    # Browser blocks all cross origin access by default
    proxy_pass http://127.0.0.1:9090;
}
```

This is appropriate when there is no legitimate cross origin caller.

### 13.8 Development mode with permissive localhost

```nginx
map $http_origin $cors_origin {
    default                                  "";
    "~^https?://localhost(:[0-9]+)?$"        "$http_origin";
    "~^https?://127\.0\.0\.1(:[0-9]+)?$"     "$http_origin";
    "~^https://app\.example\.com$"           "$http_origin";  # production
}
```

Allows http://localhost:5173 (Vite), http://localhost:3000 (Next.js dev), etc, in addition to production.

### 13.9 FastAPI sidecar handling CORS entirely

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://app.example.com",
        "https://admin.example.com",
        "http://localhost:5173",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization", "X-CSRF-Token"],
    expose_headers=["X-Total-Count", "Server-Timing", "ETag"],
    max_age=7200,
)
```

When using FastAPI's middleware, nginx should pass through without adding its own CORS headers:

```nginx
location /api/ {
    # Do NOT add CORS headers here; the upstream handles them
    proxy_pass http://127.0.0.1:9090;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Mixing nginx and FastAPI CORS produces duplicate headers, which browsers reject.

### 13.10 Bubbles cross subdomain pattern

For thatwebhostingguy.com wildcard subdomains that need to call back to the main domain:

```nginx
server {
    server_name api.thatdeveloperguy.com;

    map $http_origin $cors_origin {
        default                                            "";
        "~^https://[a-z0-9-]+\.thatwebhostingguy\.com$"    "$http_origin";
        "~^https://[a-z0-9-]+\.thatdeveloperguy\.com$"     "$http_origin";
        "~^https://thatdeveloperguy\.com$"                 "$http_origin";
    }

    location / {
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Methods "GET, POST, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type" always;
            add_header Access-Control-Max-Age "86400" always;
            add_header Vary "Origin" always;
            return 204;
        }

        if ($cors_origin) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Vary "Origin" always;
        }
        proxy_pass http://127.0.0.1:9090;
    }
}
```

---

## 14. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete CORS stanza, layered with the previous five frameworks.

```nginx
# /etc/nginx/nginx.conf (http context)
http {
    # CORS origin allow list (define once, use site by site)
    map $http_origin $cors_origin {
        default                                              "";
        "~^https://(app\.|admin\.)?example\.com$"            "$http_origin";
        "~^https://partner\.example\.com$"                   "$http_origin";
    }
}
```

```nginx
# /etc/nginx/sites-available/api.example.com

server {
    listen 443 ssl;
    listen 443 quic;
    http2 on;
    http3 on;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    # HTTP/3 advertisement (from framework-http-performance-headers.md)
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # Security baseline (from framework-http-security-headers.md)
    include snippets/common-security-headers.conf;

    # API location with full CORS handling
    location / {
        # Preflight (OPTIONS) handler
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token, X-Request-ID" always;
            add_header Access-Control-Allow-Credentials "true" always;
            add_header Access-Control-Max-Age "86400" always;
            add_header Vary "Origin" always;
            add_header Cache-Control "public, max-age=86400" always;
            return 204;
        }

        # Actual request handling: CORS headers plus pass to upstream
        if ($cors_origin) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Credentials "true" always;
            add_header Access-Control-Expose-Headers "X-Total-Count, X-RateLimit-Remaining, Server-Timing, ETag, Link" always;
            add_header Vary "Origin" always;
        }

        # SEO: API endpoints should not be indexed (from framework-http-seo-headers.md)
        add_header X-Robots-Tag "noindex, nofollow" always;

        # Cache control (from framework-http-caching-headers.md)
        add_header Cache-Control "public, max-age=0, must-revalidate" always;

        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass_header Server-Timing;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify the full flow:

```bash
# Preflight
curl -sI -X OPTIONS -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: PUT" \
     -H "Access-Control-Request-Headers: authorization" \
     https://api.example.com/users/123

# Actual
curl -sI -X PUT -H "Origin: https://app.example.com" \
     -H "Authorization: Bearer test" \
     https://api.example.com/users/123
```

Both should return the expected CORS headers.

---

## 15. AUDIT CHECKLIST

Run through these 50 items for any production CORS configuration.

### Access-Control-Allow-Origin

1. [ ] Header present on every CORS enabled endpoint (both preflight and actual response).
2. [ ] Header value is specific (not `*`) for any endpoint with credentials.
3. [ ] Multi origin support uses map based validation, not raw `$http_origin` reflection.
4. [ ] No origin reflection without validation against an allow list.
5. [ ] Allow list regex matches exactly (anchored with `^` and `$`).
6. [ ] Vary: Origin header set whenever Allow-Origin is dynamic.
7. [ ] Origins listed include exact scheme (https://, not just example.com).
8. [ ] Origins listed include port when non standard.

### Access-Control-Allow-Methods

9. [ ] Preflight response includes the methods the API actually accepts.
10. [ ] OPTIONS is always in the list (some browsers require it).
11. [ ] No methods listed that the API does not accept (avoid misleading the browser).
12. [ ] PUT, DELETE, PATCH explicitly listed if the API uses them.

### Access-Control-Allow-Headers

13. [ ] All custom request headers used by the frontend are listed.
14. [ ] Content-Type is listed (required for JSON APIs).
15. [ ] Authorization is listed (required for token based auth).
16. [ ] X-CSRF-Token (or equivalent) is listed if used.
17. [ ] Header names match the case used by the frontend (case insensitive but spelled correctly).

### Access-Control-Expose-Headers

18. [ ] Custom response headers used by the frontend are explicitly exposed.
19. [ ] X-Total-Count or pagination headers exposed if pagination is used.
20. [ ] Rate limit headers (X-RateLimit-*) exposed if rate limiting is announced.
21. [ ] Server-Timing exposed if RUM relies on it for cross origin endpoints.
22. [ ] ETag exposed if client side validation uses it.

### Access-Control-Max-Age

23. [ ] Max-Age header present on every preflight response.
24. [ ] Value is at least 7200 (Chrome cap) to maximize cross browser benefit.
25. [ ] Value 86400 sent to take full advantage of Firefox cap (Chrome still caps at 7200).
26. [ ] No accidentally setting Max-Age to 0 or -1 in production.

### Access-Control-Allow-Credentials

27. [ ] Set to `true` only on endpoints that require authenticated cross origin access.
28. [ ] When set, Allow-Origin is specific (not wildcard).
29. [ ] When set, all other Allow-* and Expose-Headers use specific values, not wildcards.
30. [ ] Cookies used for auth have SameSite=None; Secure attributes.
31. [ ] Frontend uses `credentials: "include"` in fetch when needed.

### Preflight handling

32. [ ] OPTIONS requests return 204 No Content (or 200).
33. [ ] Preflight response includes all required CORS headers.
34. [ ] The `if ($request_method = OPTIONS)` pattern is used safely (with `return`, not `proxy_pass`).
35. [ ] Preflight is handled at the nginx layer (faster) when possible, not always forwarded to upstream.

### Configuration hygiene

36. [ ] No duplicate Allow-Origin headers (would indicate nginx and upstream both adding).
37. [ ] `add_header` inheritance trap not triggered (use snippet pattern from framework-http-security-headers.md).
38. [ ] All add_header lines use `always` keyword.
39. [ ] No CORS headers leaking on non CORS endpoints (e.g. main HTML pages).
40. [ ] No origins in the allow list that no longer exist or have been deprecated.

### Security

41. [ ] No `Access-Control-Allow-Origin: *` with credentials anywhere.
42. [ ] No `Access-Control-Allow-Origin: null` unless specifically required.
43. [ ] Origin allow list does not include unintended subdomains via overly broad regex.
44. [ ] Pen test does not report origin reflection vulnerability.
45. [ ] StackHawk or similar scanner reports no CORS misconfigurations.

### Cross cutting

46. [ ] `nginx -t` passes without warnings.
47. [ ] `nginx -T` shows expected CORS directives.
48. [ ] curl preflight test returns expected headers.
49. [ ] curl actual request test returns expected headers.
50. [ ] Browser console shows no CORS errors on production traffic.

A site that passes all 50 has correctly configured CORS for security and performance.

---

## 16. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Wildcard plus credentials.**
Symptom: browser console "Access-Control-Allow-Origin cannot be wildcard when credentials are included".
Why it breaks: browser explicitly refuses the combination.
Fix: replace `*` with specific origin via map based validation.

**Pitfall 2: Origin reflection without validation.**
Symptom: scanner reports CORS misconfiguration; potential credential theft from any origin.
Why it breaks: `add_header Access-Control-Allow-Origin $http_origin always;` says "whoever you are, you may read with credentials". Attacker hosts JavaScript on evil.com, makes authenticated request to victim's browser, reads response.
Fix: use map based allow list (Section 5.3 and 5.5). Never reflect without validation.

**Pitfall 3: Missing Vary: Origin with dynamic Allow-Origin.**
Symptom: cache (CDN, proxy, browser) serves response approved for origin A to a request from origin B.
Why it breaks: cache key does not include Origin; one stored response serves all requesters.
Fix: always add `Vary: Origin` when Allow-Origin is dynamic.

**Pitfall 4: Missing preflight handler returning 405.**
Symptom: OPTIONS request returns 405 Method Not Allowed.
Why it breaks: location does not handle OPTIONS, falls through to default handler.
Fix: add `if ($request_method = OPTIONS) { return 204; }` block with CORS headers.

**Pitfall 5: Preflight returns 200 with body instead of 204.**
Symptom: works but logs are noisy, some scanners flag the unnecessary body.
Why it breaks: 200 is allowed by spec; 204 is preferred and cleaner.
Fix: use `return 204;` instead of letting nginx fall through to a default 200 response.

**Pitfall 6: Custom request header missing from Allow-Headers.**
Symptom: "Request header field X-Custom is not allowed".
Why it breaks: every custom header must be in the allow list.
Fix: add the header to Access-Control-Allow-Headers.

**Pitfall 7: Custom response header invisible to JavaScript.**
Symptom: `response.headers.get("X-Total-Count")` returns null despite the header being on the response.
Why it breaks: only safelisted response headers are visible by default; everything else requires opt in.
Fix: add the header to Access-Control-Expose-Headers.

**Pitfall 8: Missing Max-Age, every request preflights.**
Symptom: SPA feels slow on cross origin API calls.
Why it breaks: default cache is 5 seconds; every API call doubles latency from preflight.
Fix: add `Access-Control-Max-Age 86400`.

**Pitfall 9: nginx and upstream both adding CORS headers.**
Symptom: duplicate Allow-Origin header; browser rejects.
Why it breaks: nginx adds one, FastAPI middleware adds another. Browser sees two values, treats as invalid.
Fix: pick one layer to handle CORS. Either nginx or upstream, not both.

**Pitfall 10: `if ($request_method = OPTIONS)` combined with proxy_pass.**
Symptom: unexpected behavior, preflight succeeds but actual request fails.
Why it breaks: `if` plus `proxy_pass` is one of the nginx if antipatterns. The combination produces non obvious behavior.
Fix: keep `if` with `return` only. Use separate handling for the actual request.

**Pitfall 11: Forgotten OPTIONS in Allow-Methods.**
Symptom: preflight succeeds but some browsers (Safari) refuse the actual request.
Why it breaks: OPTIONS should be listed even though it is the preflight method itself.
Fix: include OPTIONS in the methods list.

**Pitfall 12: Cookie SameSite=None missing Secure.**
Symptom: cross origin auth works on https but fails on http; mobile users with mixed networks see intermittent auth failures.
Why it breaks: SameSite=None requires Secure attribute, which requires HTTPS.
Fix: ensure cookies are set with `Secure` attribute and site is HTTPS only (which Bubbles already is via HSTS).

**Pitfall 13: Overly broad regex allows unintended subdomains.**
Symptom: scanner reports `https://attacker.example.com.evil.com` is allowed.
Why it breaks: regex `~^https://.*\.example\.com$` matches `attacker.example.com.evil.com` because `.` matches anything.
Fix: anchor the end properly and escape dots: `~^https://[a-z0-9-]+\.example\.com$`.

**Pitfall 14: Allow list missing development origins.**
Symptom: works in production, breaks in local development.
Why it breaks: localhost or 127.0.0.1 not in the allow list.
Fix: add development origins conditionally:

```nginx
map $http_origin $cors_origin {
    default                                  "";
    "~^https://app\.example\.com$"           "$http_origin";
    # Development: comment out in production if security concern
    "~^https?://localhost(:[0-9]+)?$"        "$http_origin";
}
```

**Pitfall 15: Set-Cookie expected to be readable from JavaScript.**
Symptom: JavaScript cannot read the auth cookie.
Why it breaks: Set-Cookie is intentionally never exposed to script regardless of CORS. This is a fundamental browser security boundary.
Fix: do not attempt to read Set-Cookie from script. Use HttpOnly cookies for sessions and let the browser handle them automatically with `credentials: "include"`.

---

## 17. DIAGNOSTIC COMMANDS

Reference of every command useful for CORS investigation.

### Simulate preflight

```bash
# Simulate full preflight
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: PUT" \
     -H "Access-Control-Request-Headers: authorization, content-type" \
     https://api.example.com/users/123

# Look for the response headers
curl -sI -X OPTIONS \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     https://api.example.com/data | grep -iE "^(access-control|vary):"

# Test from different origins
for origin in "https://app.example.com" "https://evil.com" "https://localhost:5173"; do
    echo "=== Origin: $origin ==="
    curl -sI -X OPTIONS -H "Origin: $origin" \
         -H "Access-Control-Request-Method: POST" \
         https://api.example.com/data | grep -i access-control-allow-origin
done
```

### Simulate actual request

```bash
# Actual GET with origin
curl -sI -H "Origin: https://app.example.com" \
     https://api.example.com/data | grep -iE "^(access-control|vary):"

# Actual POST with auth
curl -sI -X POST -H "Origin: https://app.example.com" \
     -H "Authorization: Bearer test" \
     -H "Content-Type: application/json" \
     https://api.example.com/users \
     -d '{"name":"test"}'

# Confirm Expose-Headers and the actual headers being exposed
curl -sI -H "Origin: https://app.example.com" \
     https://api.example.com/users | grep -iE "^(access-control-expose|x-total|server-timing|etag):"
```

### Test the full flow

```bash
# Bash function: test a full CORS flow
test_cors() {
    local origin=$1
    local method=$2
    local url=$3

    echo "=== Preflight: $method $url from $origin ==="
    curl -sI -X OPTIONS -H "Origin: $origin" \
         -H "Access-Control-Request-Method: $method" \
         -H "Access-Control-Request-Headers: authorization, content-type" \
         "$url" | grep -iE "^(HTTP|access-control|vary):"

    echo ""
    echo "=== Actual: $method $url from $origin ==="
    curl -sI -X "$method" -H "Origin: $origin" \
         -H "Authorization: Bearer test" \
         "$url" | grep -iE "^(HTTP|access-control|vary):"
}

test_cors "https://app.example.com" "PUT" "https://api.example.com/users/123"
test_cors "https://evil.com" "PUT" "https://api.example.com/users/123"
```

### Browser DevTools quick reference

In Chrome DevTools:

1. **Network tab**: shows preflight (OPTIONS) and actual requests as separate entries.
2. Click the OPTIONS request: Headers tab shows the CORS response.
3. Click the actual request: Headers tab shows CORS response and any errors.
4. **Console tab**: CORS errors appear in red with the specific reason.

Useful console commands:

```javascript
// Test a CORS request and read response
fetch("https://api.example.com/data", {
    method: "GET",
    credentials: "include",
    headers: {"Authorization": "Bearer test"}
})
.then(r => {
    console.log("Status:", r.status);
    console.log("X-Total-Count:", r.headers.get("X-Total-Count"));
    console.log("Server-Timing:", r.headers.get("Server-Timing"));
    return r.json();
})
.then(data => console.log("Data:", data))
.catch(e => console.error("Error:", e));

// List what headers are visible
fetch("https://api.example.com/data")
    .then(r => {
        for (const [k, v] of r.headers.entries()) {
            console.log(k, ":", v);
        }
    });
```

### Server side investigation

```bash
# Find CORS directives in active config
nginx -T 2>/dev/null | grep -iE "access-control|http_origin|cors_origin"

# Check map definitions
nginx -T 2>/dev/null | grep -A 10 "map \$http_origin"

# Find any wildcard plus credentials combinations (security check)
nginx -T 2>/dev/null | awk '/Access-Control-Allow-Origin.*\*/{f=1} /Access-Control-Allow-Credentials.*true/{if(f)print "DANGER: wildcard + credentials in same scope"} /server {/{f=0}'

# Audit for any raw $http_origin usage (unvalidated reflection)
nginx -T 2>/dev/null | grep "Access-Control-Allow-Origin.*\$http_origin"
# Any output here is potentially a vulnerability

# Apply changes
nginx -t && systemctl reload nginx
```

### External CORS testers

```bash
# Online testers
echo "Visit: https://cors-test.codehappy.dev/"
echo "Visit: https://cors-anywhere.herokuapp.com/"

# Self hosted with a test page
cat > /tmp/cors-test.html << 'EOF'
<!DOCTYPE html>
<html><body>
<h1>CORS Test</h1>
<button onclick="testCors()">Test</button>
<pre id="out"></pre>
<script>
async function testCors() {
    const out = document.getElementById("out");
    out.textContent = "Testing...";
    try {
        const r = await fetch("https://api.example.com/data", {
            method: "GET",
            credentials: "include",
            headers: {"Authorization": "Bearer test"}
        });
        out.textContent = `Status: ${r.status}\n`;
        for (const [k, v] of r.headers.entries()) {
            out.textContent += `${k}: ${v}\n`;
        }
    } catch (e) {
        out.textContent = `Error: ${e.message}`;
    }
}
</script>
</body></html>
EOF
# Serve from a different origin to test cross origin behavior
python3 -m http.server -d /tmp 8080
# Visit http://localhost:8080/cors-test.html in browser
```

---

## 18. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): Cache-Control, ETag, Last-Modified, Expires, Vary, Age. Vary: Origin is part of the CORS pattern and shares mechanics with Vary: Accept-Encoding from the caching framework.
* [framework-http-content-headers.md](framework-http-content-headers.md): Content-Type, Content-Language, Content-Encoding, Content-Length, Content-Disposition. Content-Type values determine whether a request preflights (application/json preflights; form encoded does not).
* [framework-http-seo-headers.md](framework-http-seo-headers.md): X-Robots-Tag, Link, Location. API endpoints should typically be noindex.
* [framework-http-security-headers.md](framework-http-security-headers.md): HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, CORP, COEP, COOP. CORS is closely related to but distinct from CORP. CORS is what the origin says about who may read its responses; CORP is what the resource owner says about who may load it at all.
* [framework-http-performance-headers.md](framework-http-performance-headers.md): Server-Timing, Timing-Allow-Origin, Server, Alt-Svc. Timing-Allow-Origin is the CORS equivalent for the Resource Timing API. Server-Timing requires Access-Control-Expose-Headers to be readable from cross origin JavaScript.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook.
* [framework-fastapi-sidecar.md](framework-fastapi-sidecar.md): FastAPI patterns including CORSMiddleware usage.
* WHATWG Fetch (CORS spec): https://fetch.spec.whatwg.org/
* MDN CORS guide: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
* MDN Access-Control-Allow-Origin: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Origin
* MDN Access-Control-Allow-Credentials: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Credentials
* MDN Access-Control-Max-Age: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Max-Age
* enable-cors.org: https://enable-cors.org/
* nginx `if` is evil article: https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Bubbles default CORS pattern for authenticated API

```nginx
# http context
map $http_origin $cors_origin {
    default                                  "";
    "~^https://app\.example\.com$"           "$http_origin";
}

# server / location context
location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-CSRF-Token" always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Max-Age "86400" always;
        add_header Vary "Origin" always;
        return 204;
    }

    if ($cors_origin) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Expose-Headers "X-Total-Count, Server-Timing, ETag, Link" always;
        add_header Vary "Origin" always;
    }
    proxy_pass http://127.0.0.1:9090;
}
```

### Public read only API

```nginx
location /api/public/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin "*" always;
        add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS" always;
        add_header Access-Control-Max-Age "86400" always;
        return 204;
    }
    add_header Access-Control-Allow-Origin "*" always;
    proxy_pass http://127.0.0.1:9090;
}
```

### Header decision matrix

| Header | Preflight | Actual response | With credentials? |
|---|---|---|---|
| Access-Control-Allow-Origin | Yes | Yes | Specific value (no `*`) |
| Access-Control-Allow-Methods | Yes | n/a | Specific value (no `*`) |
| Access-Control-Allow-Headers | Yes | n/a | Specific value (no `*`) |
| Access-Control-Allow-Credentials | Yes | Yes | `true` |
| Access-Control-Max-Age | Yes | n/a | Specific seconds value |
| Access-Control-Expose-Headers | n/a | Yes | Specific value (no `*`) |
| Vary: Origin | Yes | Yes | Required when Allow-Origin is dynamic |

### Five rules to memorize

1. Never combine `*` with credentials. Browser refuses.
2. Always validate origin against an allow list. Never reflect blindly.
3. Always set `Vary: Origin` when Allow-Origin is dynamic.
4. Always set `Max-Age` (use 86400) to avoid every API call preflighting.
5. Custom request headers go in Allow-Headers; custom response headers go in Expose-Headers.

### Five commands every operator should know

```bash
# 1. Simulate preflight
curl -sI -X OPTIONS -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     https://api.example.com/data

# 2. Simulate actual request
curl -sI -H "Origin: https://app.example.com" \
     https://api.example.com/data | grep -i access-control

# 3. Test from disallowed origin (should not get Allow-Origin)
curl -sI -H "Origin: https://evil.com" \
     https://api.example.com/data | grep -i access-control-allow-origin

# 4. Find any unvalidated origin reflection (security audit)
nginx -T 2>/dev/null | grep "Access-Control-Allow-Origin.*\$http_origin"
# Any output is potentially a vulnerability

# 5. Apply changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Preflight succeeds
curl -sI -X OPTIONS -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: PUT" \
     https://api.example.com/data | head -1
# Expected: HTTP/2 204

# 2. Disallowed origin gets no Allow-Origin
curl -sI -H "Origin: https://evil.com" \
     https://api.example.com/data | grep -i access-control-allow-origin | wc -l
# Expected: 0

# 3. Vary: Origin is set on dynamic responses
curl -sI -H "Origin: https://app.example.com" \
     https://api.example.com/data | grep -i vary | grep -i origin
# Expected: vary: Origin
```

If all three produce expected output AND browser DevTools shows no CORS errors on production traffic, the CORS stack is correctly wired.

---

End of framework-http-cors-headers.md.
