# framework-http-caching-headers.md

Comprehensive reference for the six HTTP response headers that control how browsers, shared caches, CDNs, search engine crawlers, and AI crawlers cache and revalidate web content. Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to UNIVERSAL-RANKING-FRAMEWORK.md and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx, AI assistants generating or repairing nginx config, and anyone troubleshooting caching anomalies on a self hosted stack.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Caching Mental Model (read this first)
5. Cache-Control (the primary lever)
6. ETag (Google's preferred caching signal as of late 2025)
7. Last-Modified (the date based fallback)
8. Expires (legacy, do not author new ones)
9. Vary (cache key correctness)
10. Age (intermediate cache transparency)
11. Conditional Requests: how the headers actually talk to each other
12. Asset Class Recipes (HTML, JSON, CSS/JS, images, fonts, service worker, sitemaps)
13. Bubbles Nginx Reference Block (paste ready)
14. Audit Checklist (50+ items)
15. Common Pitfalls
16. Diagnostic Commands (curl, wget, browser devtools)
17. Cross-References

---

## 1. DEFINITION

HTTP caching headers are response headers a server sends along with a resource to tell every cache between the origin and the end client how long the response stays fresh, how to verify it is still current, and which request variations require a separate cached copy. The six headers covered here split into three concerns:

* **Freshness directives**: `Cache-Control`, `Expires`. They answer "how long can a cache reuse this without asking?"
* **Validators**: `ETag`, `Last-Modified`. They answer "did this resource change since the last time the client saw it?"
* **Cache key and transparency**: `Vary`, `Age`. They answer "does this response depend on request headers?" and "how stale is the copy I just got?"

Together these headers form a single coherent system. Configuring any one in isolation produces incorrect behavior. The strongest sites tune all six in concert.

---

## 2. WHY IT MATTERS

Three independent pressures push correct caching headers from "nice to have" to "required infrastructure" in 2025 and forward.

**Crawl budget.** Googlebot and every major AI crawler (GPTBot, ClaudeBot, PerplexityBot, Applebot, OAI SearchBot, CCBot) revisit production URLs constantly. When a server returns a proper 304 Not Modified response to a conditional request, the crawler skips the body, saves bandwidth, and uses the saved capacity to crawl deeper into the site. Google explicitly published a December 2024 blog post titled "Allow us to cache, pretty please" and updated its crawler documentation in November 2025 to emphasize that ETag is the preferred caching signal because it avoids date formatting bugs. Sites that respond 200 OK on every request waste crawl budget and slow indexation of new and updated pages.

**Core Web Vitals and LCP.** Cached resources skip TLS handshakes, TCP setup, and full body transfers. Returning visitors and crawlers that revalidate against a strong validator see sub 50 ms response times instead of full page loads. This pulls LCP down measurably and contributes to the page experience signal Google factors into ranking.

**Origin server load.** Bubbles serves 181 live hostnames from a single VPS. Without proper caching, every request lands on disk and consumes CPU for TLS, compression, and SSI processing. Correct headers offload 70 to 95 percent of return traffic to client and intermediate caches, leaving CPU headroom for new visitors and crawl spikes.

**Cost of getting it wrong.** Misconfigured caching is one of the few infrastructure mistakes that can silently destroy a site. Examples seen in the wild and recoverable by following this framework:

* A page goes live, the editor changes the price an hour later, and the price is wrong for the next 30 days because someone set `Cache-Control: max-age=2592000` on HTML.
* A logged in user sees another user's account dashboard because a shared cache stored a response that should have been `private`.
* A site disappears from Google because `must-revalidate` plus a misconfigured upstream returns 504 to Googlebot on every conditional request.
* A site looks fast in Lighthouse but every revisit triggers a full page download because `Vary: User-Agent` exploded the cache key space.

Every example above is preventable with the rules in this document.

---

## 3. WHAT THIS COVERS

Each of the six headers gets the same six part treatment:

1. **What it does**: the canonical RFC 9111 definition plus the practical implication.
2. **Syntax and directives**: every legal value, what it means, and when it is wrong.
3. **How to build it on Bubbles**: paste ready nginx config.
4. **How to verify it**: curl commands that prove the header is set and behaving.
5. **How to troubleshoot**: the four or five failure modes seen in the field and how to recognize each.
6. **How to fix common breakage**: ordered repair steps.

Recipes for the most common asset classes are then collected in one place (Section 12) so a builder can copy a block per file type without rederiving from first principles.

---

## 4. THE CACHING MENTAL MODEL (READ THIS FIRST)

Every cache that touches a response runs the same decision loop. Internalize this loop and every header decision becomes obvious.

```
Request comes in
        |
        v
Is there a stored response for this URL?
   |                            |
  NO                           YES
   |                            |
   v                            v
Forward to origin       Is the stored response fresh?
                              (max-age, Expires, heuristic)
                                    |                    |
                                   YES                  NO
                                    |                    |
                                    v                    v
                            Serve from cache     Revalidate with origin
                            (add Age header)     using ETag and/or Last-Modified
                                                         |
                                                         v
                                            Origin returns 304 or 200
                                                         |
                                                         v
                                            304: serve stored body
                                            200: replace stored body
```

The two questions a cache asks are independent:

1. **Freshness**: is this response still allowed to be reused without checking? Answered by `Cache-Control` (`max-age`, `s-maxage`) or `Expires`.
2. **Validation**: when freshness expires, how do I check cheaply if the body still matches? Answered by `ETag` and `Last-Modified`.

`Vary` modifies the cache key (the lookup ID for "is there a stored response for this URL"). `Age` is informational only and reports how long the response has been sitting in an intermediate cache.

Three cache audiences exist and need different rules:

* **Private cache**: a single browser. Honors `private` and `public`. Stores per user content safely.
* **Shared cache**: a CDN, reverse proxy, or corporate proxy. Honors `s-maxage`, `public`, `private`. Must never store `private` responses.
* **Crawler cache**: Googlebot, ClaudeBot, GPTBot, and others. Honors `ETag` and `Last-Modified` conditional requests as the primary signal. Google ignores most other directives.

A correct header set must produce sensible behavior in all three audiences simultaneously.

---

## 5. CACHE-CONTROL (THE PRIMARY LEVER)

### 5.1 What It Does

`Cache-Control` is the single most important caching header. It tells every cache how to handle the response: whether to store it, how long it stays fresh, which audiences are allowed to store it, and what to do when freshness expires. Defined in RFC 9111. Supersedes `Expires` in every cache that understands HTTP/1.1, which is every cache made in the last 25 years.

### 5.2 Syntax And Directives

The header is a comma separated list of directives. Order does not matter. Spaces are optional.

```
Cache-Control: public, max-age=31536000, immutable
Cache-Control: private, max-age=0, must-revalidate
Cache-Control: no-store
Cache-Control: s-maxage=3600, max-age=60, stale-while-revalidate=30
```

Directive reference, response side only (request side directives like `max-stale` and `only-if-cached` are out of scope for server configuration):

**`max-age=<seconds>`**. How long the response stays fresh in any cache. Most important single directive. Counted from the moment the origin generated the response. Common values: `0` (immediately stale, must revalidate), `60` (one minute), `3600` (one hour), `86400` (one day), `604800` (one week), `2592000` (30 days), `31536000` (one year, the practical maximum).

**`s-maxage=<seconds>`**. Overrides `max-age` for shared caches only (CDNs, reverse proxies). Private browser cache ignores this directive and uses `max-age` instead. Useful when you want a CDN to hold a longer copy than the browser, but Bubbles is a self hosted origin with no CDN, so `s-maxage` is rarely needed here.

**`public`**. Any cache (private or shared) is allowed to store this response. Required when authentication is involved and you want the response cached anyway. Often redundant when `s-maxage` or `must-revalidate` is present.

**`private`**. Only a private (browser) cache may store this response. Shared caches must not store it. Required for any response that contains user specific data: logged in dashboards, personalized prices, shopping cart contents, session aware HTML.

**`no-cache`**. Confusingly named. Does not mean "do not cache". Means "you may store this, but you must revalidate with the origin before serving". The response will be checked on every reuse via conditional request. Combine with `ETag` or `Last-Modified` to make this efficient.

**`no-store`**. The real "do not cache". Caches must not store any part of the response. Use sparingly. Appropriate for responses containing secrets, single use tokens, or financial transaction confirmations. Overuse destroys performance.

**`must-revalidate`**. Once the response goes stale, the cache must revalidate before serving. If revalidation fails (origin unreachable), the cache must return a 504 Gateway Timeout instead of serving the stale copy. Combined with `max-age=0` produces an "always revalidate" policy.

**`proxy-revalidate`**. Same as `must-revalidate` but applies only to shared caches. Allows private browser caches to serve stale.

**`no-transform`**. Caches and intermediaries must not modify the response body (no recompression, no minification, no image conversion). Useful when bytes matter exactly, such as cryptographic signatures or precomputed hashes.

**`immutable`**. The response body will never change during its freshness lifetime. The browser will not even send a conditional request when the user clicks reload. Pair with `max-age=31536000` and a content hashed URL. Supported in all modern browsers including Chrome, Firefox, and Safari. Crawlers do not act on `immutable`.

**`stale-while-revalidate=<seconds>`**. Once the response goes stale, the cache may serve the stale copy for this many additional seconds while revalidating in the background. Hides revalidation latency from end users. Excellent for slow APIs and dynamic HTML where eventual consistency is acceptable.

**`stale-if-error=<seconds>`**. If revalidation fails (origin returns 5xx or times out), the cache may serve the stale copy for this many additional seconds. Acts as a soft circuit breaker.

**`must-understand`**. The cache should store the response only if it understands the caching requirements for the status code. Niche. Skip unless you have a specific reason.

### 5.3 Directive Conflict Resolution

When directives appear to conflict, the most restrictive wins:

* `no-store` beats everything. If `no-store` is present, the response is not stored, period.
* `private` beats `public`. If both somehow appear (a misconfiguration), shared caches treat the response as `private`.
* `no-cache` plus `max-age=N` means store the response, mark it as stale immediately for serving purposes, revalidate every time.
* `must-revalidate` plus `max-age=N` means serve fresh for N seconds, then revalidate; if revalidation fails, return 504.

### 5.4 How To Build It On Bubbles

The four nginx mechanisms for setting `Cache-Control`:

**1. `expires` directive (sets `Cache-Control: max-age=` and `Expires` together):**

```nginx
location ~* \.(css|js|woff2|jpg|png|webp|avif|svg)$ {
    expires 1y;
    add_header Cache-Control "public, immutable" always;
}
```

The `expires` directive emits both `Expires: <date>` and `Cache-Control: max-age=<seconds>`. The added `add_header` line appends the rest of the directives. Use `always` so the header is sent even on 4xx and 5xx responses.

**2. `add_header` directive (full manual control):**

```nginx
location / {
    add_header Cache-Control "public, max-age=300, stale-while-revalidate=60" always;
}
```

Direct and explicit. The trade off: this does not set `Expires` automatically. Modern clients do not need `Expires`, but legacy crawlers and some intermediate caches still read it. For belt and suspenders, set both.

**3. `expires` with negative or `off` for no caching:**

```nginx
location = /sw.js {
    expires off;
    add_header Cache-Control "no-store, no-cache, must-revalidate" always;
}

location ~ ^/(login|account|admin) {
    expires -1;
    add_header Cache-Control "private, no-store" always;
}
```

`expires off` disables nginx's automatic `Expires` and `Cache-Control` generation, leaving your manual `add_header` value in place. `expires -1` emits `Expires: <past date>` and `Cache-Control: no-cache`.

**4. Map based dynamic policy:**

```nginx
map $sent_http_content_type $cache_policy {
    default                              "public, max-age=300";
    "~^text/html"                        "public, max-age=0, must-revalidate";
    "~^application/json"                 "private, max-age=0, no-cache";
    "~^application/javascript"           "public, max-age=31536000, immutable";
    "~^image/"                           "public, max-age=2592000";
    "~^font/"                            "public, max-age=31536000, immutable";
}

server {
    add_header Cache-Control $cache_policy always;
}
```

Powerful when you cannot rely on URL extension to determine content type, such as when a backend serves dynamic content with varied types from the same path.

### 5.5 The `add_header` Inheritance Trap

This is the most common source of "I set the header but it is not appearing" complaints in nginx. The rule:

> An `add_header` directive in a nested block (location, if, limit_except) **replaces** all `add_header` directives in parent blocks unless every parent header is also redeclared at the inner level.

If you have this:

```nginx
server {
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Strict-Transport-Security "max-age=31536000" always;

    location ~* \.js$ {
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }
}
```

The `.js` files will get `Cache-Control` but **will not** get `X-Frame-Options` or `Strict-Transport-Security`. The location block's `add_header` wiped out the parent inheritance.

The fix is to put shared headers in a `conf.d/*.conf` snippet and `include` it inside every location block, or to use the `more_set_headers` directive from the `headers_more` nginx module which does append correctly. The Bubbles convention is the include approach:

```nginx
# /etc/nginx/snippets/common-headers.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Then in each location:

```nginx
location ~* \.js$ {
    include snippets/common-headers.conf;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}
```

### 5.6 How To Verify

Six quick commands:

```bash
# 1. See the full response header set
curl -sI https://example.com/path/to/asset.css

# 2. Just the caching header
curl -sI https://example.com/page.html | grep -i cache-control

# 3. Force a fresh request (skip local DNS and connection cache)
curl -sI --resolve example.com:443:169.155.162.118 https://example.com/

# 4. Check that the header survives compression
curl -sI -H "Accept-Encoding: gzip" https://example.com/page.html

# 5. Check the header from outside the local network
curl -sI https://example.com/page.html
# compare to running the same command on Bubbles itself

# 6. Test what Googlebot sees
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" https://example.com/
```

### 5.7 Troubleshooting

**Symptom: Header not appearing in curl output.**
Causes ranked by frequency:
1. `add_header` inheritance trap (Section 5.5). Move the header to a snippet and include in every location.
2. The location block did not match the URL. Add `return 200 "DEBUG: matched location X\n";` and `add_header X-Debug-Match "X" always;` to confirm.
3. Missing `always` keyword. Without `always`, headers do not apply to 4xx and 5xx responses.
4. Nginx config did not reload. Run `nginx -t && systemctl reload nginx`.
5. A downstream proxy is stripping the header. Run the curl directly against Bubbles using `--resolve`.

**Symptom: Header appears but the browser still requests on every page load.**
Causes:
1. `Cache-Control: no-cache` is forcing revalidation. This is by design. Change to `max-age=N` if no revalidation is desired.
2. `Cache-Control: max-age=0` with `must-revalidate`. Same outcome. Same fix.
3. `Vary: *` or `Vary: User-Agent` is preventing the browser from matching its cached copy. See Section 9.
4. The URL keeps changing (cache busting query strings appended on every request). Check page source.

**Symptom: Header appears but a CDN or upstream proxy is returning stale content.**
Bubbles has no CDN in front of it, so this should not happen in this stack. If it does, the cause is almost always a corporate proxy at the visitor's end. There is no server side fix; document for the visitor.

**Symptom: Different headers on HTTP and HTTPS.**
You have two server blocks (port 80 and port 443) and updated only one. Mirror the caching config in both, or redirect all HTTP to HTTPS at the server block level and only configure caching in the HTTPS block.

### 5.8 How To Fix Common Breakage

**Case: HTML is being served from browser cache after editing.**
The HTML had a long `max-age` and the browser is serving the stale copy. Fix:

```nginx
# In the location block that serves HTML
location ~* \.html$ {
    add_header Cache-Control "public, max-age=0, must-revalidate" always;
}

# Or if you serve clean URLs through try_files:
location / {
    add_header Cache-Control "public, max-age=0, must-revalidate" always;
    try_files $uri $uri/ $uri.html =404;
}
```

After deploying, run `nginx -t && systemctl reload nginx`. Visitors with the stale copy will revalidate on their next request and pick up the new content.

**Case: Static assets are being re downloaded on every visit.**
Fingerprinted assets (logo.abc123.png, main.f4e1.js) should be served `immutable`. If they are not, add:

```nginx
location ~* \.(css|js|woff2|jpg|jpeg|png|webp|avif|svg|gif|ico)$ {
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}
```

**Case: A user reported seeing another user's data.**
This is a `private` versus `public` failure. Find every endpoint that returns user specific data and ensure:

```nginx
location ~ ^/(account|dashboard|cart|orders) {
    add_header Cache-Control "private, no-store" always;
}
```

`no-store` is the strongest available. Use it for anything that includes session data, even if you also use `private`.

---

## 6. ETAG (GOOGLE'S PREFERRED CACHING SIGNAL AS OF LATE 2025)

### 6.1 What It Does

`ETag` (Entity Tag) is an opaque identifier the server assigns to a specific representation of a resource. When the client asks again, it sends the saved ETag in an `If-None-Match` request header. If the server's current ETag for that URL matches, the server returns `304 Not Modified` with no body. If it does not match, the server returns `200 OK` with the new body and a new ETag.

Google's crawler documentation (updated November 2025) explicitly states that ETag is preferred over Last-Modified for crawler caching because ETag avoids date formatting bugs. When both ETag and Last-Modified are present, Google's crawlers use the ETag value per the HTTP standard. The recommendation is to set both for compatibility with other consumers.

### 6.2 Syntax And Strength

Format: a quoted string, optionally prefixed with `W/` for weak comparison.

```
ETag: "abc123def456"
ETag: W/"abc123def456"
```

**Strong ETag**: guarantees byte for byte identity. Two resources with the same strong ETag are interchangeable at the byte level. Required for range requests (HTTP 206 partial content), which matters for video, large downloads, and PDFs.

**Weak ETag**: indicates semantic equivalence. Two resources with the same weak ETag represent the same logical entity but the bytes may differ (different whitespace, different gzip compression level, different image optimization). Cannot be used for range requests but is sufficient for all standard cache validation.

### 6.3 How Nginx Generates ETag

Nginx generates ETags automatically for static files served from disk. The default format is:

```
ETag: "<last-modified-hex>-<content-length-hex>"
```

Example: a 4096 byte file modified at Unix timestamp 0x67510a12 produces `ETag: "67510a12-1000"`. This is generated cheaply with no hashing. It is a strong ETag.

Nginx does **not** generate ETags for:

* Responses from upstream servers (proxy_pass, fastcgi_pass, uwsgi_pass). The upstream must generate them.
* Dynamic responses generated by SSI, sub_filter, image_filter, or any other content modification.
* Responses where the body is altered after generation (notably gzip on the fly, which forces a strong to weak ETag conversion).

### 6.4 The Gzip ETag Conversion (the trap everyone hits)

When nginx applies gzip compression on the fly, the bytes change. A strong ETag is no longer accurate. Nginx handles this two ways depending on configuration:

**Default behavior (nginx 1.7.3 and later)**: nginx weakens the ETag, converting `ETag: "abc"` into `ETag: W/"abc"`. The cache still works for validation purposes. Range requests against the gzipped representation will not work, but range requests on text content are rare.

**Strong ETag preservation**: set `gzip_vary on;` and ensure your upstream emits properly quoted strong ETags. Nginx will then keep the strong ETag intact for clients that did not negotiate gzip.

**`gzip_proxied no_etag`**: tells nginx to skip gzipping any response that has an ETag, preserving the strong ETag at the cost of losing compression for those responses. Rarely the right tradeoff.

The Bubbles default is to accept the weak ETag conversion. Weak ETags satisfy Google's crawler caching requirements and all browser cache validation. The strong ETag is only required if you serve byte exact range requests, which Bubbles does not.

### 6.5 How To Build It On Bubbles

Static files: ETag is on by default. Verify with:

```bash
curl -sI https://example.com/css/style.css | grep -i etag
# Expected: ETag: W/"67510a12-1234" or similar
```

To explicitly enable or disable:

```nginx
# Explicitly on (this is the default for static)
location ~* \.(css|js|jpg|png|webp|svg|woff2)$ {
    etag on;
}

# Explicitly off (when you do not want a validator at all)
location = /random.html {
    etag off;
}
```

For dynamic responses from a FastAPI sidecar or other upstream, the upstream must emit the ETag and nginx must pass it through:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass_header ETag;
}
```

A FastAPI route that emits a content hashed ETag:

```python
import hashlib
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/api/reviews")
async def reviews(request: Request):
    body = await fetch_reviews()
    payload = json.dumps(body, sort_keys=True).encode()
    etag = '"' + hashlib.sha256(payload).hexdigest()[:16] + '"'

    if request.headers.get("if-none-match") == etag:
        return Response(status_code=304)

    return Response(
        content=payload,
        media_type="application/json",
        headers={"ETag": etag, "Cache-Control": "public, max-age=300"},
    )
```

### 6.6 How To Verify

```bash
# 1. Confirm ETag is present
curl -sI https://example.com/style.css | grep -i etag

# 2. Capture the ETag and replay as If-None-Match
ETAG=$(curl -sI https://example.com/style.css | grep -i etag | awk '{print $2}' | tr -d '\r')
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css
# Expected: HTTP/2 304

# 3. Check what Googlebot sees
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     -H "If-None-Match: $ETAG" \
     https://example.com/style.css
# Expected: HTTP/2 304 with content-length: 0

# 4. Verify ETag changes when content changes
md5sum /var/www/sites/example.com/style.css
curl -sI https://example.com/style.css | grep -i etag
# Edit the file
echo "/* trailing comment */" >> /var/www/sites/example.com/style.css
curl -sI https://example.com/style.css | grep -i etag
# The two ETag values must differ
```

### 6.7 Troubleshooting

**Symptom: ETag not present in response.**
1. The location is serving a dynamic upstream that does not emit ETag. Either add ETag at the upstream or use `Last-Modified` as the validator.
2. `etag off;` is set in the location or a parent block.
3. The response was modified after generation by SSI, sub_filter, or image_filter. ETag is intentionally stripped because the modification breaks byte identity.
4. The `If-Modified-Since` or `If-None-Match` middleware is consuming the headers. Check `proxy_pass_header` settings.

**Symptom: ETag never matches, every request returns 200 OK.**
1. The ETag value changes on every request. This happens when the upstream generates it from a timestamp at request time rather than from the body. Fix at the upstream: hash the body, not the clock.
2. Strong to weak conversion is happening mid path. The client sent `If-None-Match: "abc"` but the server compares against `W/"abc"` and rejects with strict comparison. Use weak comparison or stop weakening.
3. Different nginx workers are generating different ETags. Should not happen for static files (deterministic from filesystem) but can happen if an upstream uses a random seed in the ETag.
4. A CDN or proxy is rewriting the ETag. Not applicable to Bubbles (no CDN).

**Symptom: `If-None-Match` is being sent but server returns 200 instead of 304.**
1. Server is comparing ETag values byte for byte and the quotes do not match. RFC requires the quoted form (`"abc"`) including the quotes in the comparison.
2. Multiple representations exist (gzip and identity, mobile and desktop) and `Vary` is correctly differentiating them, producing different ETags. Expected behavior. The cache will store one copy per variant.
3. The body changed but the URL is the same. The 200 is correct.

**Symptom: ETag is weak when it should be strong (range requests failing).**
1. Gzip on the fly is weakening it. Either disable gzip for that content type, set `gzip_proxied no_etag`, or pre compress the asset with `gzip_static on` and serve the precompressed file directly.
2. SSI is enabled on the location. Disable `ssi on;` for the file type.

### 6.8 How To Fix Common Breakage

**Case: Googlebot is re downloading every page on every crawl.**
ETag is missing or invalid. Verify:

```bash
curl -sI -A "Googlebot" https://example.com/page-being-recrawled.html | grep -iE "etag|last-modified"
```

If ETag is missing, ensure the location block does not have `etag off;`, ensure the response is not SSI processed, and ensure the upstream (if any) emits an ETag header.

**Case: ETag values look like garbage (random characters every time).**
The upstream is generating ETag from non deterministic input (timestamp, process ID, random seed). Fix at the upstream: generate from a hash of the body or from a version stamp that changes only when content changes.

**Case: Crawl rate dropped after enabling caching.**
This is expected and desirable. Google reduces crawl rate when 304 responses prove the content has not changed. New and updated pages still get full attention because their ETag values differ from what Google has cached.

---

## 7. LAST-MODIFIED (THE DATE BASED FALLBACK)

### 7.1 What It Does

`Last-Modified` reports the date and time the server believes the resource was last changed. When the client asks again, it sends the saved date in an `If-Modified-Since` request header. If the server's current `Last-Modified` is equal to or earlier than the date sent, the server returns 304 Not Modified.

Functionally similar to ETag, but Google explicitly prefers ETag because Last-Modified is prone to date formatting bugs and has only one second resolution (responses changed within the same second are indistinguishable). Google still supports it and recommends setting both ETag and Last-Modified for compatibility.

### 7.2 Syntax And Format

The date must follow the HTTP standard format, which is exactly one of three legal formats with the IMF fixdate format being the only one anyone uses today:

```
Last-Modified: Wed, 21 Oct 2026 07:28:00 GMT
```

The required structure:

* Three letter weekday name, comma, space.
* Two digit day, space.
* Three letter month name, space.
* Four digit year, space.
* Two digit hour, colon, two digit minute, colon, two digit second, space.
* The literal string `GMT`.

Any deviation (lowercase month, missing leading zeros, timezone other than GMT, ISO 8601 format) is technically invalid and may be ignored by strict crawlers. Google has historically been tolerant but explicitly recommends the exact format above.

### 7.3 How Nginx Generates Last-Modified

For static files, nginx automatically emits `Last-Modified` based on the filesystem mtime of the file. Format is always correct. No configuration required.

For dynamic responses, the upstream must emit it. If the upstream omits it, nginx will not synthesize one.

### 7.4 How To Build It On Bubbles

Static files: on by default. Verify with:

```bash
curl -sI https://example.com/style.css | grep -i last-modified
```

To explicitly disable for a location:

```nginx
location ~* \.html$ {
    if_modified_since off;
}
```

This stops nginx from honoring `If-Modified-Since` and returning 304. Rarely useful. The header still appears in responses.

To explicitly set on a dynamic response from a FastAPI sidecar:

```python
from email.utils import format_datetime
from datetime import datetime, timezone

last_modified = format_datetime(datetime.now(timezone.utc), usegmt=True)
return Response(
    content=body,
    headers={"Last-Modified": last_modified, "ETag": etag}
)
```

`email.utils.format_datetime` with `usegmt=True` produces the exact RFC compliant format.

### 7.5 The mtime Gotcha For Deploys

This is the most common Last-Modified bug. When you deploy a new build with `rsync` or `cp`, the filesystem mtime of unchanged files often gets updated to the deploy time. Result: every revalidation against `Last-Modified` returns 200 OK with a full body, because Last-Modified moved forward even though the content did not change.

Fixes ranked by quality:

**1. Use `rsync` with `--times` and `--checksum`:**

```bash
rsync -avzc --times /local/build/ user@bubbles:/var/www/sites/example.com/
```

`-c` checksums files and only copies changed ones, preserving mtime on unchanged files. `--times` preserves mtime on copied files.

**2. Use content addressed asset URLs (fingerprinting):**

When asset URLs include a content hash (`main.f4e1c2.js`), the Last-Modified value does not matter for cache invalidation. New content gets a new URL, old content keeps its long max-age plus immutable. This is the recommended pattern for any non HTML asset.

**3. Touch only changed files:**

```bash
find /var/www/sites/example.com -newer /tmp/last-deploy -type f -exec touch -d "$(date)" {} \;
```

Updates mtime only on files that actually changed since the last deploy.

**4. Rely on ETag instead:**

Bubbles ETag is generated from mtime AND content length, so a touched but unchanged file produces a new ETag too. This means ETag suffers the same problem if mtime is the only input. The real fix is to use content hashing in the URL or in the ETag generation.

### 7.6 How To Verify

```bash
# 1. Confirm Last-Modified is present and formatted correctly
curl -sI https://example.com/page.html | grep -i last-modified
# Expected: Last-Modified: Wed, 21 Oct 2026 07:28:00 GMT

# 2. Replay it as If-Modified-Since
LM=$(curl -sI https://example.com/page.html | grep -i last-modified | cut -d' ' -f2- | tr -d '\r')
curl -sI -H "If-Modified-Since: $LM" https://example.com/page.html
# Expected: HTTP/2 304

# 3. Validate the format is exactly RFC compliant
date -u -d "$LM" "+%a, %d %b %Y %H:%M:%S GMT"
# Output should match LM character for character

# 4. Check what changes after a file edit
stat /var/www/sites/example.com/page.html | grep Modify
echo "" >> /var/www/sites/example.com/page.html
curl -sI https://example.com/page.html | grep -i last-modified
# New Last-Modified must be later than old
```

### 7.7 Troubleshooting

**Symptom: Last-Modified header missing.**
1. Location serves a dynamic upstream that does not emit it. Fix at upstream.
2. `if_modified_since off;` is set somewhere upstream. Search the config: `grep -r if_modified_since /etc/nginx/`.
3. The file does not exist or is being served from a memory cache without filesystem stat. Should not happen on Bubbles.

**Symptom: Last-Modified date is wrong (in the future, or far in the past).**
1. Server clock is wrong. Run `timedatectl status` and ensure NTP is synced. On Bubbles: `systemctl status chrony` or `systemctl status systemd-timesyncd`.
2. The file's mtime was set manually with `touch -d` for testing and never reset.
3. The file was restored from backup with the backup's mtime instead of "now".

**Symptom: Every deploy invalidates all caches.**
mtime is being updated on every file by the deploy process. See Section 7.5.

**Symptom: Browser caches but crawler does not.**
Google explicitly prefers ETag. If only Last-Modified is set, Google still uses it but you have lower validation reliability. Add ETag (Section 6).

**Symptom: 304 returned but client renders blank page.**
A 304 response must have an empty body. If your application is sending a body with a 304 status, the client may treat it as a malformed response. Fix at the upstream: return only headers on 304.

### 7.8 How To Fix Common Breakage

**Case: Last-Modified is older than the actual change date.**
The file was edited in place and the editor did not update mtime, or the file is being served from a stale cached copy. Run `touch` on the file or rebuild the deploy artifact.

**Case: Last-Modified format is wrong (e.g. ISO 8601).**
A custom upstream is emitting a non standard date. Fix the upstream code to use the correct format. In Python: `email.utils.format_datetime(dt, usegmt=True)`. In Node: `new Date().toUTCString()`. In Go: `time.Now().UTC().Format(http.TimeFormat)`.

**Case: Both ETag and Last-Modified set, both look right, but client revalidation is failing.**
Most clients (and Google) check ETag first. If the ETag has changed but Last-Modified has not, the response is 200 with new body. If the ETag is the same but Last-Modified has moved forward, the response is still 304 because content matches. The two headers are not required to agree on freshness state. If you suspect a client bug, test with curl and only one validator at a time.

---

## 8. EXPIRES (LEGACY, DO NOT AUTHOR NEW ONES)

### 8.1 What It Does

`Expires` is the HTTP/1.0 predecessor to `Cache-Control: max-age`. It gives an absolute date and time after which the response is considered stale.

```
Expires: Wed, 21 Oct 2026 07:28:00 GMT
```

Every HTTP/1.1 cache understands `Cache-Control: max-age` and uses it in preference to `Expires` when both are present. Effectively, `Expires` is only consulted in two scenarios:

1. A pure HTTP/1.0 client (none exist in production today).
2. A misconfigured intermediate that ignores `Cache-Control` (rare bug).

### 8.2 Why It Exists And Why Nginx Still Emits It

The nginx `expires` directive emits both `Cache-Control: max-age=N` and `Expires: <date>` together. This is harmless. The `Expires` value is computed once per response and serves as the belt and suspenders backup for the few clients that might need it.

You should not author `Expires` manually. If nginx is doing it for you via the `expires` directive, leave it alone. If you have a manual `add_header Expires` somewhere, remove it and let `expires` handle it.

### 8.3 The Date Math Problem

Because `Expires` is an absolute date, it is wrong the moment a clock disagreement appears between client and server. If the server's clock is one hour ahead, every cache that uses `Expires` will consider the response stale one hour early. `Cache-Control: max-age` is relative to when the response was received, so it is immune to clock skew.

This is the primary reason `max-age` won.

### 8.4 How To Build It On Bubbles

Use nginx's `expires` directive (which sets both headers correctly):

```nginx
location ~* \.(css|js)$ {
    expires 1y;
}
```

Nginx output:

```
Cache-Control: max-age=31536000
Expires: Wed, 24 May 2027 14:00:00 GMT
```

To suppress `Expires`:

```nginx
location ~* \.(css|js)$ {
    expires off;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}
```

`expires off` prevents nginx from emitting either header. Your manual `add_header Cache-Control` provides the freshness directive. `Expires` is omitted, which is fine.

### 8.5 How To Verify

```bash
curl -sI https://example.com/style.css | grep -iE "expires|cache-control"
```

You should see both lines, or just `Cache-Control` if you opted out of `Expires`. Never see only `Expires` without `Cache-Control`; that is a misconfiguration.

### 8.6 Troubleshooting

**Symptom: `Expires` and `Cache-Control` disagree.**
You have a manual `add_header Expires` overriding the nginx generated value. Remove the manual line and let `expires` handle both.

**Symptom: `Expires: 0` or `Expires: -1` appearing.**
This is the legacy way of saying "already expired". Modern equivalent: `Cache-Control: no-cache` or `Cache-Control: max-age=0, must-revalidate`. Either is fine; do not need both.

**Symptom: Page caches longer than expected on some clients.**
A buggy intermediate might be honoring `Expires` over `Cache-Control`. Set both to the same lifetime and the conflict disappears.

### 8.7 How To Fix Common Breakage

**Case: You inherited a config with hardcoded `Expires` dates.**
Replace them with `expires 1y;` or whatever lifetime is correct. Hardcoded dates eventually pass and produce silent cache failure.

**Case: `Expires` value is in the past on a fresh response.**
The server is computing it wrong. If using nginx `expires`, this should never happen. If using a manual `add_header Expires "..."`, the date is hardcoded and stale. Replace with nginx `expires` directive.

---

## 9. VARY (CACHE KEY CORRECTNESS)

### 9.1 What It Does

`Vary` tells caches which request headers must match for a cached response to be reused. The cache key for a stored response becomes `(URL, list of values from headers named in Vary)`.

```
Vary: Accept-Encoding
```

The cache stores one copy per distinct `Accept-Encoding` value. A browser sending `Accept-Encoding: gzip, br` gets a different cached entry than a browser sending `Accept-Encoding: gzip`. Without `Vary: Accept-Encoding`, the cache would serve the same gzipped body to a client that did not request gzip, producing a corrupted response.

### 9.2 The Cache Key Explosion Problem

`Vary` is a double edged sword. Every header listed in `Vary` multiplies the number of cached copies the cache must store. A few examples:

* `Vary: Accept-Encoding`: 3 to 5 variants (identity, gzip, br, deflate, zstd). Manageable.
* `Vary: Accept-Encoding, Accept-Language`: 3 to 5 times maybe 20 languages = up to 100 variants. Risky.
* `Vary: User-Agent`: thousands of distinct user agent strings, each producing its own cached copy. Cache hit rate collapses. **Almost always wrong.**
* `Vary: *`: special value meaning "vary on every aspect of the request including unspecified ones". Functionally disables caching for that response. Rarely the right answer; use `Cache-Control: no-store` instead.

The rule: list only headers that genuinely change the response body. Three values are commonly correct:

1. `Accept-Encoding`: required when you serve gzipped, brotli, or zstd compressed responses.
2. `Accept`: required when content negotiation actually serves different formats (rare on Bubbles).
3. `Cookie` or specific cookie names via `Vary: Cookie`: when authentication state changes the response. Even then, prefer `Cache-Control: private`.

### 9.3 How Nginx Builds Vary

**Automatic via gzip_vary:**

```nginx
http {
    gzip on;
    gzip_vary on;
    gzip_types text/html text/css application/javascript application/json image/svg+xml;
}
```

`gzip_vary on;` tells nginx to add `Vary: Accept-Encoding` to any response it compresses. This is the correct way to handle the most common Vary requirement. Do not also add `Vary: Accept-Encoding` via `add_header` (nginx will deduplicate but the manual approach is fragile if compression is disabled later).

**Manual:**

```nginx
location /api/locale-aware/ {
    add_header Vary "Accept-Language, Accept-Encoding" always;
}
```

When you need to vary on something beyond `Accept-Encoding`, set it manually. Same `add_header` inheritance rules apply (Section 5.5).

**Brotli compression:**

If you have brotli enabled (via `ngx_brotli` module), it adds `Vary: Accept-Encoding` automatically when responses are compressed, same as gzip.

### 9.4 How To Verify

```bash
# 1. Check Vary is present on compressible content
curl -sI https://example.com/page.html | grep -i vary

# 2. Verify the response actually varies
curl -s -H "Accept-Encoding: identity" https://example.com/page.html | wc -c
curl -s -H "Accept-Encoding: gzip" https://example.com/page.html | wc -c
# Sizes should differ; gzip version smaller

# 3. Check Vary is not overly broad
curl -sI https://example.com/page.html | grep -i vary
# Expected: Vary: Accept-Encoding (one value)
# Bad: Vary: User-Agent or Vary: *

# 4. Check that compression actually applies
curl -sI -H "Accept-Encoding: gzip" https://example.com/page.html | grep -i content-encoding
# Expected: content-encoding: gzip
```

### 9.5 Troubleshooting

**Symptom: Page Speed Insights or GTmetrix warns "Specify a Vary: Accept-Encoding header".**
The audit is checking for the literal `Vary: Accept-Encoding` header on compressible responses. Fix:

```nginx
http {
    gzip on;
    gzip_vary on;
    gzip_types text/html text/plain text/css application/javascript application/json application/xml application/ld+json image/svg+xml;
}
```

`nginx -t && systemctl reload nginx`. Re run the audit.

**Symptom: Random clients see corrupted (binary gibberish) responses.**
A cache somewhere served a gzipped body to a client that did not request gzip, and no `Vary: Accept-Encoding` was present to prevent the mix up. Add `gzip_vary on;` and ensure no upstream is stripping the header.

**Symptom: Cache hit rate at the CDN is unexpectedly low.**
Not applicable on Bubbles (no CDN). If you ever put a CDN in front and hit rates collapse, the first suspect is `Vary: User-Agent` or `Vary: Cookie` causing key explosion.

**Symptom: Logged in users see other users' personalized HTML.**
You are caching authenticated responses without varying on the auth cookie. Fix: do not cache authenticated responses at all. Add `Cache-Control: private, no-store` to any response that contains session data.

### 9.6 How To Fix Common Breakage

**Case: Vary header missing on compressed responses.**

```nginx
# In http {} block
gzip on;
gzip_vary on;
gzip_min_length 1100;
gzip_comp_level 6;
gzip_types text/html text/css application/javascript application/json application/xml application/ld+json image/svg+xml application/rss+xml font/ttf font/otf application/x-font-ttf;
```

`nginx -t && systemctl reload nginx`. Verify with curl.

**Case: Vary: User-Agent is appearing.**
Search for it: `grep -r "Vary" /etc/nginx/`. Remove the offending `add_header Vary "User-Agent"` line. If it is coming from an upstream, fix the upstream or strip the header:

```nginx
location / {
    proxy_pass http://upstream;
    proxy_hide_header Vary;
    add_header Vary "Accept-Encoding" always;
}
```

**Case: Vary: * is being returned.**
Same fix as above. Replace with the actual headers that affect the response, or convert to `Cache-Control: no-store` if the response truly should not be cached.

---

## 10. AGE (INTERMEDIATE CACHE TRANSPARENCY)

### 10.1 What It Does

`Age` is an informational header set by intermediate caches. It reports how many seconds have elapsed since the response was generated at the origin. The origin server itself never sets `Age`; only proxies, CDNs, and shared caches do.

```
Age: 234
```

Meaning: this response has been sitting in a cache for 234 seconds. Combined with the response's `Cache-Control: max-age`, you can compute how much fresh life remains: `max-age minus Age = seconds until stale`.

### 10.2 When You Will See Age

On Bubbles directly: never. Bubbles is the origin. Nginx as an origin does not emit `Age`.

On Bubbles acting as a reverse proxy (with `proxy_cache` enabled): yes. The proxy cache layer emits `Age` for responses it serves from its cache.

From an intermediate (corporate proxy, mobile carrier, ISP cache): yes. Visitors behind these caches may see `Age` values that explain why content seems slightly stale.

### 10.3 How To Build It (If You Run Reverse Proxy Caching)

Most Bubbles sites do not run proxy caching, but if you set one up to front a FastAPI sidecar:

```nginx
proxy_cache_path /var/cache/nginx/sidecar levels=1:2 keys_zone=sidecar:10m max_size=1g inactive=60m;

server {
    location /api/ {
        proxy_cache sidecar;
        proxy_cache_valid 200 5m;
        proxy_pass http://127.0.0.1:9090;
        add_header X-Cache-Status $upstream_cache_status always;
    }
}
```

The `Age` header will be emitted automatically for cache hits. `X-Cache-Status` (HIT, MISS, EXPIRED, STALE, BYPASS, REVALIDATED) is added by us for visibility.

### 10.4 How To Interpret

* `Age: 0`: response just generated, no caching benefit yet.
* `Age: <small number>`: response is fresh, served from a nearby cache. Good.
* `Age` close to `max-age`: response is about to expire; cache will revalidate soon.
* `Age` greater than `max-age`: response is stale. Server allowed it via `stale-while-revalidate` or `stale-if-error`, or the cache is misbehaving.
* `Age` from an unexpected source: an intermediate is caching responses you did not authorize. Verify your `Cache-Control` and `Vary` are restrictive enough.

### 10.5 How To Verify

```bash
# Run from various network locations
curl -sI https://example.com/page.html | grep -iE "age|cache-control"

# From the same network repeatedly to see if Age increments
for i in 1 2 3 4 5; do
    curl -sI https://example.com/page.html | grep -i "age:"
    sleep 30
done
```

On a direct Bubbles request you expect no `Age` header. If you see one, an intermediate is involved.

### 10.6 Troubleshooting

**Symptom: Age header is appearing on Bubbles responses but Bubbles is the origin.**
1. An ISP cache or corporate proxy is sitting between you and Bubbles. Test from a different network.
2. You have `proxy_cache` configured somewhere on Bubbles itself. Search: `grep -r proxy_cache /etc/nginx/`.
3. The browser cache is sometimes mistaken for a proxy cache; check devtools to confirm the response came from network.

**Symptom: Age is always 0.**
Either the cache always misses (low hit rate, check `X-Cache-Status`) or the response is being regenerated on every request (check `Cache-Control` for `no-store` or `no-cache`).

**Symptom: Age is higher than max-age.**
The cache is serving stale content. This is allowed when `stale-while-revalidate` or `stale-if-error` is set. If neither is set, the cache is misbehaving.

### 10.7 How To Fix Common Breakage

**Case: Visitors report seeing old content but Bubbles shows the new content.**
An intermediate cache is between them and Bubbles. Have them check headers; if `Age` is high, the response is from a cache. They can hard reload (Ctrl Shift R) to bypass the browser cache, but cannot bypass a corporate proxy. Server side, ensure your `Cache-Control` is reasonable so the proxy expires the response soon.

**Case: You want to instrument cache behavior on Bubbles.**
Add an `X-Cache-Status` header anywhere proxy caching is active. Combine with `Age` to see exactly what each cache layer is doing. Use `curl -sI` to make it visible.

---

## 11. CONDITIONAL REQUESTS: HOW THE HEADERS ACTUALLY TALK TO EACH OTHER

The six response headers above produce these request headers from clients on subsequent visits:

| Response header sent first | Request header sent on revisit | Server response |
|---|---|---|
| `ETag: "abc"` | `If-None-Match: "abc"` | 304 if match, 200 if not |
| `Last-Modified: Wed, 21 Oct...` | `If-Modified-Since: Wed, 21 Oct...` | 304 if not modified since, 200 if modified |
| `Cache-Control: max-age=300` | (none; cache serves locally for 300 sec) | n/a |
| `Vary: Accept-Encoding` | (none; affects cache lookup only) | n/a |

When both `ETag` and `Last-Modified` are present, clients (and Google) prefer `If-None-Match` over `If-Modified-Since`. The server should check `If-None-Match` first and only fall back to `If-Modified-Since` if the first is absent.

A 304 response must include:

* The status line `HTTP/2 304` or `HTTP/1.1 304 Not Modified`.
* The current `ETag` and/or `Last-Modified` (so the client can update its stored copy).
* `Cache-Control` (so the client knows the new freshness window).
* No body. Body must be exactly zero bytes.

Nginx handles all of this automatically for static files. Upstream applications must implement it themselves.

### 11.1 Full Conditional Request Test

```bash
# Step 1: get the initial response and capture validators
RESP=$(curl -sI https://example.com/style.css)
ETAG=$(echo "$RESP" | grep -i etag | awk '{print $2}' | tr -d '\r')
LM=$(echo "$RESP" | grep -i last-modified | cut -d' ' -f2- | tr -d '\r')

echo "ETag: $ETAG"
echo "Last-Modified: $LM"

# Step 2: replay with If-None-Match (ETag based)
echo "--- ETag revalidation ---"
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -1
# Expected: HTTP/2 304

# Step 3: replay with If-Modified-Since (Last-Modified based)
echo "--- Last-Modified revalidation ---"
curl -sI -H "If-Modified-Since: $LM" https://example.com/style.css | head -1
# Expected: HTTP/2 304

# Step 4: replay with a stale validator
echo "--- Stale validator (should return 200) ---"
curl -sI -H "If-None-Match: \"old-etag-value\"" https://example.com/style.css | head -1
# Expected: HTTP/2 200

# Step 5: simulate Googlebot
echo "--- Googlebot view ---"
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     -H "If-None-Match: $ETAG" \
     https://example.com/style.css | head -3
# Expected: HTTP/2 304, with etag and last-modified echoed
```

If all four show the expected status, your caching is correctly wired up.

---

## 12. ASSET CLASS RECIPES

Each block is a paste ready location stanza. Copy the ones relevant to your build into the server block.

### 12.1 HTML (dynamic content, must always be current)

```nginx
location / {
    try_files $uri $uri/ $uri.html =404;
    add_header Cache-Control "public, max-age=0, must-revalidate" always;
    # ETag and Last-Modified are auto-generated by nginx for static .html files
}
```

Rationale: HTML changes whenever the editor publishes a new version. Setting `max-age=0, must-revalidate` makes the browser send a conditional request on every page load. The 304 response is tiny (under 200 bytes) so the cost is negligible.

### 12.2 CSS and JS (fingerprinted, immutable)

```nginx
# For fingerprinted assets like main.f4e1c2.css
location ~* \.(css|js)$ {
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}
```

Rationale: when the URL changes for every new version, the old URL never needs to be revalidated. `immutable` tells the browser not to even send a conditional request on user reload.

If your build pipeline does NOT fingerprint assets:

```nginx
location ~* \.(css|js)$ {
    expires 1h;
    add_header Cache-Control "public, max-age=3600, must-revalidate" always;
}
```

Shorter freshness, forced revalidation. Less efficient but safe.

### 12.3 Fonts (immutable, long lived)

```nginx
location ~* \.(woff2|woff|ttf|otf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
    add_header Access-Control-Allow-Origin "*" always;
}
```

Rationale: fonts essentially never change. The CORS header is required if the font is served from a different origin than the HTML using it.

### 12.4 Images (long max-age, mutable)

```nginx
location ~* \.(jpg|jpeg|png|gif|webp|avif|svg|ico)$ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000" always;
}
```

Rationale: images change occasionally (CMS replace, brand refresh). 30 days strikes a balance. If your images are fingerprinted, add `immutable` and bump to 1 year.

### 12.5 Service Worker (never cache)

```nginx
location = /sw.js {
    expires off;
    add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0" always;
    add_header Service-Worker-Allowed "/" always;
}
```

Rationale: a stale service worker is catastrophic; it serves a stale version of the entire site forever until manually unregistered. Must always come fresh from origin.

### 12.6 Sitemaps and robots.txt (short fresh window)

```nginx
location = /robots.txt {
    expires 1h;
    add_header Cache-Control "public, max-age=3600" always;
}

location = /sitemap.xml {
    expires 10m;
    add_header Cache-Control "public, max-age=600" always;
}

location ~* /sitemaps/.*\.xml$ {
    expires 10m;
    add_header Cache-Control "public, max-age=600" always;
}
```

Rationale: crawlers fetch these frequently. Short cache lets new pages get discovered quickly. Long enough cache avoids beating up the server during a crawl burst.

### 12.7 JSON APIs (varies by API)

Public, read only API:

```nginx
location /api/public/ {
    expires 5m;
    add_header Cache-Control "public, max-age=300, stale-while-revalidate=60" always;
    add_header Vary "Accept-Encoding" always;
}
```

Authenticated, user specific API:

```nginx
location /api/private/ {
    expires off;
    add_header Cache-Control "private, no-store" always;
}
```

### 12.8 PDFs and downloadable documents

```nginx
location ~* \.(pdf|zip|tar|gz)$ {
    expires 1d;
    add_header Cache-Control "public, max-age=86400" always;
    add_header X-Content-Type-Options "nosniff" always;
}
```

Rationale: documents change occasionally. 1 day cache is safe. Disable sniffing to prevent browsers from rendering PDFs inline when not intended.

### 12.9 The llms.txt, ai.txt, security.txt family

```nginx
location ~* /(llms\.txt|ai\.txt|humans\.txt|\.well-known/security\.txt)$ {
    expires 1d;
    add_header Cache-Control "public, max-age=86400" always;
}
```

Rationale: AI policy files and security disclosures change rarely. One day is fine.

### 12.10 RSS and Atom feeds

```nginx
location ~* /(feed|rss|atom)/?$ {
    expires 15m;
    add_header Cache-Control "public, max-age=900" always;
    add_header Vary "Accept-Encoding" always;
}
```

Rationale: feed readers poll every 15 to 60 minutes. Short cache matches the polling rate.

---

## 13. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete caching stanza for a typical Bubbles site, ready to drop into `/etc/nginx/sites-available/example.com`:

```nginx
# /etc/nginx/sites-available/example.com
# Caching configuration aligned with Google's November 2025 crawler guidance.

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    listen 443 quic reuseport;
    listen [::]:443 quic reuseport;

    server_name example.com www.example.com;

    root /var/www/sites/example.com;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Tell clients HTTP/3 is available
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # Global compression with proper Vary handling
    gzip on;
    gzip_vary on;
    gzip_min_length 1100;
    gzip_comp_level 6;
    gzip_types text/html text/plain text/css application/javascript application/json application/xml application/ld+json image/svg+xml application/rss+xml application/atom+xml font/ttf font/otf application/x-font-ttf;

    # Brotli (if module installed)
    brotli on;
    brotli_comp_level 6;
    brotli_types text/html text/plain text/css application/javascript application/json application/xml application/ld+json image/svg+xml application/rss+xml application/atom+xml font/ttf font/otf;

    # ETag on for all static (default, explicit for clarity)
    etag on;

    # ------- ASSET CLASS RECIPES -------

    # HTML: revalidate on every visit, body almost always 304
    location ~* \.html$ {
        expires 0;
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        include snippets/common-security-headers.conf;
    }

    # Fingerprinted CSS and JS
    location ~* \.(css|js)$ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        include snippets/common-security-headers.conf;
    }

    # Fonts
    location ~* \.(woff2|woff|ttf|otf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        add_header Access-Control-Allow-Origin "*" always;
        include snippets/common-security-headers.conf;
    }

    # Images
    location ~* \.(jpg|jpeg|png|gif|webp|avif|svg|ico)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000" always;
        include snippets/common-security-headers.conf;
    }

    # Service worker: never cache
    location = /sw.js {
        expires off;
        add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0" always;
        add_header Service-Worker-Allowed "/" always;
        include snippets/common-security-headers.conf;
    }

    # Crawler entry files
    location = /robots.txt {
        expires 1h;
        add_header Cache-Control "public, max-age=3600" always;
    }

    location = /sitemap.xml {
        expires 10m;
        add_header Cache-Control "public, max-age=600" always;
    }

    location ~* /(llms\.txt|ai\.txt|humans\.txt|\.well-known/security\.txt)$ {
        expires 1d;
        add_header Cache-Control "public, max-age=86400" always;
    }

    # FastAPI sidecar proxy on port 9090
    location /api/ {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass_header ETag;
        proxy_pass_header Last-Modified;
        add_header Vary "Accept-Encoding" always;
        include snippets/common-security-headers.conf;
    }

    # Default location for clean URLs
    location / {
        try_files $uri $uri/ $uri.html =404;
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        include snippets/common-security-headers.conf;
    }

    # 404 (do not cache)
    error_page 404 /404.html;
    location = /404.html {
        internal;
        add_header Cache-Control "public, max-age=300" always;
    }
}

# HTTP to HTTPS redirect (do not cache)
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    add_header Cache-Control "no-store" always;
    return 301 https://$server_name$request_uri;
}
```

The shared security header snippet at `/etc/nginx/snippets/common-security-headers.conf`:

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

---

## 14. AUDIT CHECKLIST

Run through these 50 items for any production site. Each item is a single curl command or visual inspection. Pass means the listed expected value appears.

### Cache-Control

1. [ ] HTML response has `Cache-Control: public, max-age=0, must-revalidate` (or similar revalidate policy).
2. [ ] CSS/JS response has `Cache-Control: public, max-age=31536000, immutable`.
3. [ ] Font response has `Cache-Control: public, max-age=31536000, immutable`.
4. [ ] Image response has `Cache-Control: public, max-age=2592000` or longer.
5. [ ] Service worker (sw.js) has `Cache-Control: no-store, no-cache, must-revalidate`.
6. [ ] Robots.txt has `Cache-Control: public, max-age=3600`.
7. [ ] Sitemap.xml has `Cache-Control: public, max-age=600` (10 min) or similar short window.
8. [ ] Any authenticated endpoint has `Cache-Control: private, no-store`.
9. [ ] No response has BOTH `no-cache` AND `max-age=31536000` (contradictory).
10. [ ] No `Cache-Control` is duplicated in the response (would indicate a config bug).

### ETag

11. [ ] HTML response has an `ETag` header.
12. [ ] CSS/JS response has an `ETag` header.
13. [ ] Image response has an `ETag` header.
14. [ ] FastAPI sidecar responses pass through `ETag` (test `/api/...` endpoint).
15. [ ] ETag value is stable across multiple requests for unchanged content (`curl` twice, diff the headers).
16. [ ] ETag value changes when content changes (`echo "" >> file; curl` again, ETag differs).
17. [ ] `If-None-Match` replay returns `HTTP/2 304` with empty body.
18. [ ] Googlebot user agent gets the same ETag and same 304 behavior.

### Last-Modified

19. [ ] HTML response has `Last-Modified` header.
20. [ ] CSS/JS response has `Last-Modified` header.
21. [ ] Last-Modified date is in RFC compliant format (`Wed, 21 Oct 2026 07:28:00 GMT`).
22. [ ] Last-Modified value is in the past, not the future.
23. [ ] `If-Modified-Since` replay returns `HTTP/2 304` with empty body.
24. [ ] Last-Modified does NOT update on deploys of unchanged files (validate with mtime check before and after).

### Expires

25. [ ] Either `Expires` is present alongside `Cache-Control: max-age=N` (both generated by nginx `expires` directive), OR neither is present.
26. [ ] Expires date is consistent with `max-age` (within a couple seconds of `now + max-age`).
27. [ ] No hardcoded `Expires` dates in any config (search `grep -r "add_header Expires" /etc/nginx/`).

### Vary

28. [ ] Compressible responses (HTML, CSS, JS) have `Vary: Accept-Encoding`.
29. [ ] No response has `Vary: User-Agent` (would explode cache key).
30. [ ] No response has `Vary: *` (effectively disables caching).
31. [ ] `gzip_vary on;` is set in the http or server block.
32. [ ] gzip is actually applied (response has `Content-Encoding: gzip` when client requests it).
33. [ ] Request without `Accept-Encoding: gzip` returns uncompressed body.
34. [ ] Request with `Accept-Encoding: gzip` returns compressed body.

### Age

35. [ ] Direct request to Bubbles (`curl --resolve`) does NOT include an `Age` header (Bubbles is origin).
36. [ ] If `proxy_cache` is enabled anywhere, `X-Cache-Status` header is added for observability.

### Conditional requests behavior

37. [ ] First request returns `HTTP/2 200` with body.
38. [ ] Second request with `If-None-Match: <etag>` returns `HTTP/2 304` with empty body.
39. [ ] Second request with stale `If-None-Match: "wrong"` returns `HTTP/2 200` with full body.
40. [ ] 304 response includes `ETag`, `Cache-Control`, and `Last-Modified` headers.
41. [ ] 304 response body is zero bytes.

### Crawler behavior

42. [ ] Googlebot user agent gets the same caching headers as a regular browser.
43. [ ] Googlebot user agent gets a 304 on conditional request.
44. [ ] ClaudeBot, GPTBot, PerplexityBot user agents are not blocked from caching by `Cache-Control` directives.
45. [ ] Crawl rate in Search Console is steady or increasing (a sudden drop after enabling caching is normal and good; a sudden drop without that change suggests a problem).

### Deploy hygiene

46. [ ] Deploys use `rsync -c` or fingerprinted URLs to avoid spurious mtime changes.
47. [ ] CSS and JS asset URLs include a content hash (`style.f4e1c2.css`) or version param.
48. [ ] Service worker is updated on every deploy (cache busted via filename or version constant inside).
49. [ ] After deploy, `nginx -t && systemctl reload nginx` ran cleanly.
50. [ ] After deploy, sample of three URLs spot checked with curl for correct headers.

A site that passes all 50 is ready for production crawling.

---

## 15. COMMON PITFALLS

Patterns to recognize and avoid.

**Pitfall 1: Manual Expires with hardcoded date.**
Symptom: `add_header Expires "Wed, 31 Dec 2025 23:59:59 GMT" always;` in a config from years ago.
Why it breaks: the date passes and everything is permanently "expired", forcing revalidation on every request.
Fix: replace with nginx `expires 1y;` directive.

**Pitfall 2: `Cache-Control: no-cache` confused with "do not cache".**
Symptom: developer wanted to prevent caching, used `no-cache`, but content is still cached.
Why it breaks: `no-cache` means "always revalidate" not "do not store". The cache stores the response and revalidates on every request.
Fix: if you truly do not want storage, use `no-store`. If revalidation is fine, leave it alone.

**Pitfall 3: `add_header` in a location block silently drops parent `add_header` directives.**
Symptom: security headers disappear on specific file types.
Why it breaks: nginx inheritance rule.
Fix: snippet file pattern (see Section 5.5 and 13).

**Pitfall 4: Long max-age on HTML.**
Symptom: editor updates a page, change does not appear for hours or days.
Why it breaks: `max-age=86400` on HTML traps the old version in every browser and intermediate cache.
Fix: `max-age=0, must-revalidate` for HTML. Editor updates appear on next page load.

**Pitfall 5: `immutable` on non fingerprinted asset.**
Symptom: edit to `style.css` does not appear; users see old version forever (until they manually clear cache).
Why it breaks: `immutable` tells the browser the content will never change. Browser ignores user reload.
Fix: rename to `style.f4e1c2.css` so the URL changes when content changes, then `immutable` is safe.

**Pitfall 6: ETag changes on every request.**
Symptom: every revalidation returns 200 OK with full body, crawl budget wasted.
Why it breaks: upstream is generating ETag from a timestamp or random number.
Fix: generate ETag from a hash of the body (`sha256` of the JSON payload, for example), or use nginx's automatic ETag for static files.

**Pitfall 7: Vary: User-Agent kills cache hit rate.**
Symptom: cache hit rate is 5 percent when it should be 90 percent.
Why it breaks: every distinct user agent string produces a separate cache key.
Fix: remove `Vary: User-Agent`. If you must serve different content per device, do it at the URL level (m.example.com or /mobile/...).

**Pitfall 8: `Vary: *` "to be safe".**
Symptom: nothing caches anywhere; every request is full.
Why it breaks: `*` means "this response varies on every possible request aspect", which is treated as "this response cannot be cached".
Fix: list specific headers or use `Cache-Control: no-store`.

**Pitfall 9: Deploy bumps mtime on every file.**
Symptom: every deploy invalidates every cache, even for unchanged files.
Why it breaks: `cp -r` or naive deploy script touches every file with current time.
Fix: `rsync -c --times` or build pipeline that only writes changed files.

**Pitfall 10: SSI processed pages have no ETag.**
Symptom: pages with `<!--# include -->` directives never return 304.
Why it breaks: nginx strips ETag when the body is modified post generation; the on disk content is no longer authoritative.
Fix: pre render SSI at build time (use a static site generator), or set Last-Modified manually from the upstream, or accept the lost validator and rely on a short `max-age`.

**Pitfall 11: Service worker cached forever.**
Symptom: site is stuck on an old version even after deploys; users must manually unregister the service worker.
Why it breaks: `sw.js` was served with a long `max-age` and is being served from browser cache instead of fetched fresh.
Fix: see Section 12.5. Service worker MUST be `no-store, no-cache, must-revalidate`.

**Pitfall 12: Authenticated response cached as public.**
Symptom: user A sees user B's account dashboard or order history.
Why it breaks: response containing user specific data was served `Cache-Control: public, max-age=300` and a shared cache stored it.
Fix: every authenticated endpoint MUST be `Cache-Control: private, no-store`. Audit immediately if this is suspected; the privacy implications are serious.

**Pitfall 13: Mixing `max-age` and `s-maxage` inconsistently.**
Symptom: CDN cache and browser cache disagree on freshness, causing intermittent stale content.
Why it breaks: shared cache uses `s-maxage`, private cache uses `max-age`, and the two values are not coherent.
Fix: not applicable on Bubbles (no CDN). If you ever front Bubbles with a cache, set both values explicitly and verify behavior.

**Pitfall 14: Stale validators after restoring from backup.**
Symptom: after a backup restore, conditional requests start returning 200 instead of 304 for unchanged content.
Why it breaks: the restore set file mtime to "now" instead of preserving original mtime.
Fix: restore with `tar --preserve` or `rsync --times`. Verify with `stat`.

**Pitfall 15: Ignoring the `always` keyword.**
Symptom: caching headers appear on 200 responses but not on 404 or 500.
Why it breaks: `add_header` without `always` only applies to 200, 201, 204, 206, 301, 302, 303, 304, 307, 308.
Fix: append `always` to every `add_header` directive.

---

## 16. DIAGNOSTIC COMMANDS

Reference of every command useful for caching header investigation. Each one has a comment explaining what it tells you.

### Inspect headers

```bash
# Just the headers, no body
curl -sI https://example.com/page.html

# Just the caching family
curl -sI https://example.com/page.html | grep -iE "cache|etag|last-modified|expires|vary|age"

# Headers AND first lines of body
curl -si https://example.com/page.html | head -30

# Headers as Googlebot
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" https://example.com/

# Headers from a specific IP (bypass DNS)
curl -sI --resolve example.com:443:169.155.162.118 https://example.com/

# Headers including the negotiated protocol
curl -sI --http2 https://example.com/page.html
curl -sI --http3 https://example.com/page.html
```

### Test conditional requests

```bash
# Capture ETag and replay
ETAG=$(curl -sI https://example.com/style.css | awk -F': ' '/^[Ee]tag/{print $2}' | tr -d '\r')
echo "ETag is: $ETAG"
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -1

# Capture Last-Modified and replay
LM=$(curl -sI https://example.com/style.css | awk -F': ' '/^[Ll]ast-[Mm]odified/{print $2}' | tr -d '\r')
echo "Last-Modified is: $LM"
curl -sI -H "If-Modified-Since: $LM" https://example.com/style.css | head -1

# Verify body is empty on 304
curl -s -o /tmp/304-body -H "If-None-Match: $ETAG" https://example.com/style.css
wc -c /tmp/304-body
# Expected: 0 /tmp/304-body
```

### Test compression and Vary

```bash
# Test with and without gzip
curl -s -H "Accept-Encoding: identity" https://example.com/page.html | wc -c
curl -s -H "Accept-Encoding: gzip" https://example.com/page.html | wc -c

# Confirm Content-Encoding is gzip
curl -sI -H "Accept-Encoding: gzip" https://example.com/page.html | grep -i content-encoding

# Confirm Vary header is present
curl -sI -H "Accept-Encoding: gzip" https://example.com/page.html | grep -i vary

# Test brotli
curl -sI -H "Accept-Encoding: br" https://example.com/page.html | grep -iE "content-encoding|vary"
```

### Cache behavior across time

```bash
# Watch Age increase from an intermediate cache
for i in $(seq 1 10); do
    echo "=== Request $i at $(date +%T) ==="
    curl -sI https://example.com/page.html | grep -iE "age|cache-control"
    sleep 30
done
```

### Bubbles server side investigation

```bash
# Check nginx config syntax
nginx -t

# Show active config for a server name
nginx -T 2>/dev/null | grep -A 50 "server_name example.com"

# Find every place a header is set
grep -rn "Cache-Control" /etc/nginx/

# Find every add_header in a specific site config
grep -n "add_header" /etc/nginx/sites-available/example.com

# Watch access logs for cache related status codes
tail -f /var/log/nginx/access.log | grep -E " 304 | 200 "

# Count 304 vs 200 ratio over last 1000 requests
tail -1000 /var/log/nginx/access.log | awk '{print $9}' | sort | uniq -c | sort -rn

# Reload after changes
nginx -t && systemctl reload nginx
```

### Browser dev tools quick reference

When using Chrome DevTools Network panel:

* Set "Disable cache" off to see real caching behavior.
* Look at the "Size" column: "(memory cache)" or "(disk cache)" means the browser served from cache without contacting the server.
* "304" in the Status column means conditional revalidation succeeded.
* Click a request, then "Headers" tab, then look for "Response Headers" to see all the caching family.
* Hard reload (Ctrl Shift R) sends `Cache-Control: no-cache` from the client, bypassing fresh cached responses but still allowing conditional revalidation.

---

## 17. CROSS-REFERENCES

* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Section 24 (Bubbles Nginx config) inherits the caching stanzas defined here.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook. References this framework for any caching configuration decision.
* [framework-corewebvitals.md](framework-corewebvitals.md): LCP, INP, CLS targets. Caching headers directly affect LCP for repeat visitors.
* [framework-crawlbudget.md](framework-crawlbudget.md): how Google allocates crawl resources. Proper 304 responses are the primary lever for increasing crawl efficiency.
* [framework-robotstxt-crawlbudget.md](framework-robotstxt-crawlbudget.md): robots.txt as a caching surface (Section 12.6 here).
* [framework-security-headers.md](framework-security-headers.md): HSTS, CSP, X-Frame-Options. Configured alongside caching headers via the shared `common-security-headers.conf` snippet.
* [framework-compression.md](framework-compression.md): gzip and brotli configuration. Tightly coupled to ETag (gzip weakens ETags) and Vary (Vary: Accept-Encoding is required when compression is active).
* Google official documentation: https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers
* Google December 2024 blog post on caching: https://developers.google.com/search/blog/2024/12/crawling-december-caching
* RFC 9111 (HTTP Caching): https://www.rfc-editor.org/rfc/rfc9111
* MDN Cache-Control reference: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

| Asset type | Cache-Control | ETag | Last-Modified | Expires | Vary |
|---|---|---|---|---|---|
| HTML | `public, max-age=0, must-revalidate` | auto | auto | auto | `Accept-Encoding` |
| CSS/JS fingerprinted | `public, max-age=31536000, immutable` | auto | auto | auto | `Accept-Encoding` |
| CSS/JS unfingerprinted | `public, max-age=3600, must-revalidate` | auto | auto | auto | `Accept-Encoding` |
| Fonts | `public, max-age=31536000, immutable` | auto | auto | auto | none |
| Images | `public, max-age=2592000` | auto | auto | auto | none |
| Service worker | `no-store, no-cache, must-revalidate` | off | off | off | none |
| robots.txt | `public, max-age=3600` | auto | auto | auto | none |
| sitemap.xml | `public, max-age=600` | auto | auto | auto | none |
| Public API JSON | `public, max-age=300, stale-while-revalidate=60` | manual at upstream | manual at upstream | none | `Accept-Encoding` |
| Authenticated API | `private, no-store` | none | none | none | none |
| PDF/zip | `public, max-age=86400` | auto | auto | auto | none |

Three commands every operator should know by heart:

```bash
# Validate the config is syntactically correct
nginx -t

# Apply the config without dropping connections
systemctl reload nginx

# Combined: only reload if syntax check passes
nginx -t && systemctl reload nginx
```

And two curl invocations that prove caching works end to end:

```bash
# First request: should be 200 with body
curl -sI https://example.com/style.css | head -1

# Second request with ETag: should be 304 with no body
ETAG=$(curl -sI https://example.com/style.css | awk -F': ' '/^[Ee]tag/{print $2}' | tr -d '\r')
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -1
```

If the first returns 200 and the second returns 304, caching is correctly wired. Everything else in this document is detail behind that core test.

---

End of framework-http-caching-headers.md.
