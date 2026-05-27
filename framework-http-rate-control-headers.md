# framework-http-rate-control-headers.md

Comprehensive reference for the HTTP response headers that communicate rate limit state and backoff guidance to clients: the standardized `Retry-After` (RFC 9110) and the rate limit family covering both the widely deployed de facto convention `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` and the IETF standardization track `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, `RateLimit-Policy`. Includes the status code companions `429 Too Many Requests` and `503 Service Unavailable` that activate Retry-After, plus the full nginx `limit_req` and `limit_conn` configuration that produces them. Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to framework-http-caching-headers.md, framework-http-content-headers.md, framework-http-seo-headers.md, framework-http-security-headers.md, framework-http-performance-headers.md, framework-http-cors-headers.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx rate limiting, AI assistants generating or repairing rate limit config, API engineers shaping client backoff behavior, SEO operators ensuring Googlebot and AI crawlers are not accidentally throttled, and anyone troubleshooting "429s on legitimate users", "Googlebot dropped my pages", "client retry storm after maintenance", or "rate limit headers missing on API responses" anomalies on a self hosted stack.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Rate Control Mental Model (read this first)
5. Retry-After (the standardized backoff signal)
6. X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset (the de facto convention)
7. The IETF RateLimit-* Standardization (the future)
8. Status Code Pairing (429 vs 503 vs 3xx)
9. Nginx Rate Limiting Configuration (limit_req and limit_conn)
10. The Crawler Protection Rule (never rate limit Googlebot, ClaudeBot, GPTBot)
11. Client Backoff Patterns (exponential, jitter, honoring Retry-After)
12. How These Headers Interact
13. Asset Class And Use Case Recipes
14. Bubbles Nginx Reference Block (paste ready)
15. Audit Checklist (50+ items)
16. Common Pitfalls
17. Diagnostic Commands
18. Cross-References

---

## 1. DEFINITION

Rate control headers communicate two things to clients: (a) how much quota remains right now, so well behaved clients can self throttle, and (b) when to come back after being rejected, so retry storms do not amplify the problem they were meant to solve. The headers split into three families:

* **The backoff signal**: `Retry-After`. Standardized in RFC 9110. Sent with `429 Too Many Requests`, `503 Service Unavailable`, or `3xx` redirect responses. Tells the client (including search engine crawlers) exactly when it is safe to try again.
* **The rate limit announcement (de facto)**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. Widely deployed (GitHub, Twitter, Stripe, most public APIs) but never standardized. Sent on every response to let clients see their current quota state.
* **The rate limit announcement (standards track)**: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, `RateLimit-Policy`. IETF draft (draft-ietf-httpapi-ratelimit-headers, in late stages). Replaces the X- convention with un prefixed names. Modern APIs (GitLab, CircleCI, OKX, others) already emit both forms during the transition.

`Retry-After` is the only one with a hard standard. The X- and RateLimit-* families are conventions; clients have to know what your specific API uses. The standardization track aims to fix this so any RFC compliant client can read any server's rate limit state without API specific code.

---

## 2. WHY IT MATTERS

Six independent pressures push correct rate control headers from "operational hygiene" to "required infrastructure" in 2025 and forward.

**Search engines drop pages after 2 days of 429 or 503.** Google's official documentation states that returning 503 or 429 for more than 2 days will cause Google to drop those URLs from the index. The Retry-After header is what tells Googlebot "this is temporary, come back when X". Without it, Googlebot guesses, and after 48 hours of guessing wrong it stops trying. A Bubbles client site that accidentally rate limits Googlebot for a long weekend can lose its rankings.

**AI crawlers honor Retry-After.** ClaudeBot, GPTBot, PerplexityBot, and other LLM crawlers respect Retry-After (when they respect anything). For sites Joseph has optimized for AI discoverability, throttling these crawlers without proper Retry-After defeats the optimization investment. The same 2-day rule applies in spirit: persistent throttling teaches crawlers to skip the site.

**Retry storms amplify the problem.** Without Retry-After, clients retry on their own schedule. After a server briefly returns 503 during maintenance, every client retries within seconds, the next batch of 503s triggers the next round of retries, and the server stays down because it cannot drain the queue faster than retries arrive. Retry-After with a sensible value lets the storm spread out over minutes instead of seconds.

**Rate limit headers prevent self imposed denial.** Without X-RateLimit-* or RateLimit-* headers, well behaved clients have no way to throttle themselves. They must blindly send requests and react to 429s. A client that emits Server-Sent Events or polls frequently will hit limits constantly. With headers exposing remaining quota, the same client smooths its rate and never gets throttled.

**Nginx defaults are wrong for APIs.** nginx's `limit_req` and `limit_conn` modules return `503 Service Unavailable` by default. For a REST API, this is misleading: 503 implies the whole service is down, when really only this client exceeded its quota. The correct status is `429 Too Many Requests`. Fixing this is a one line config change (`limit_req_status 429`) that most deployments miss.

**Per IP rate limiting breaks legitimate clients sharing IPs.** Office networks, corporate VPNs, mobile carriers, and IPv6 with carrier grade NAT all put many users behind a single IP. Per IP rate limits configured for "individual user" levels block entire organizations. The fix is either a generous burst, per token rate limits when authenticated, or both.

**Cost of getting it wrong.** Misconfigured rate control produces silent ranking damage, real user pain, and runaway retry storms. Real examples:

* nginx rate limiting set to 10r/s with no burst, no Googlebot whitelist. Googlebot crawls the site, hits the limit, gets 503 for two days. Google drops 5,800 URLs from the index. Ranking recovery takes weeks.
* API returns 429 without Retry-After. Client library backs off exponentially starting at 1 second, doubling each retry. Hits the API 6 times in 90 seconds, all rate limited. With Retry-After: 60, would have hit once and succeeded.
* Office of 200 users behind one NAT IP. Rate limit at 30 requests per minute per IP. The first 30 users to load the morning dashboard succeed; the rest see 429s. Support tickets flood in.
* nginx returns 503 for rate limited API responses. Frontend monitoring alerts on "service unavailable" rate. On call engineer paged at 2 AM for what is actually expected client behavior.
* Maintenance window planned 30 minutes. Site returns 503 with no Retry-After. Browsers and CDN caches cache the 503. Even after maintenance ends, repeat visitors get cached 503s for hours.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the primary headers gets the same six part treatment:

1. **What it does**: the canonical RFC or IETF draft definition plus the practical implication.
2. **Syntax and values**: every legal format, what it means, when it is wrong.
3. **How to build it on Bubbles**: paste ready nginx config plus FastAPI sidecar code where the upstream is the natural emitter.
4. **How to verify it**: curl commands that trigger rate limits and read responses.
5. **How to troubleshoot**: the four or five failure modes seen in the field and how to recognize each.
6. **How to fix common breakage**: ordered repair steps.

The status code companions (429 and 503) get a dedicated section because they activate Retry-After. The nginx `limit_req` / `limit_conn` configuration gets its own dedicated section because it produces these headers and is where most operators spend their debugging time. The crawler protection rule gets its own section because it is the single most ranking impactful decision in this framework.

---

## 4. THE RATE CONTROL MENTAL MODEL (READ THIS FIRST)

Every cross origin or API request runs through the rate control decision tree. Internalize it and every header decision becomes obvious.

```
Request arrives at nginx
        |
        v
==================== IDENTIFICATION ====================
        |
        v
Who is the client?
   * IP based: $binary_remote_addr
   * Token based: $http_authorization (or extracted)
   * Session based: $cookie_session
   * Endpoint based: $request_uri
        |
        v
Is the client on the whitelist?
   * Googlebot (verified by reverse DNS)
   * ClaudeBot, GPTBot, PerplexityBot
   * Bubbles admin IPs
   * Local Tailscale network
   |
  YES (whitelisted)
   |
   v
Bypass rate limit, process normally
   |
   |
  NO (subject to rate limits)
        |
        v
==================== RATE LIMIT CHECK ====================
        |
        v
Has this client exceeded the per second rate?
   |             |
  NO            YES
   |             |
   |             v
   |          Is there burst capacity?
   |             |               |
   |            YES              NO
   |             |               |
   |             v               v
   |         Allow (queued)   Reject with 429 (or 503)
   |          or process       Set Retry-After: <seconds>
   |          immediately      Set RateLimit-Reset
   |         (with nodelay)    Log at warn level
        |          |
        v          |
==================== ANNOUNCE RATE LIMIT STATE =================
        |          |
        v          |
Decorate response with rate limit headers
   X-RateLimit-Limit: 100        (current quota)
   X-RateLimit-Remaining: 47     (what is left)
   X-RateLimit-Reset: 1742658600 (when it resets)
   AND modern equivalents:
   RateLimit-Limit: 100
   RateLimit-Remaining: 47
   RateLimit-Reset: 60
   RateLimit-Policy: 100;w=60
        |
        v
==================== CLIENT REACTION ====================
        |
        v
Well behaved client:
   Reads X-RateLimit-Remaining
   Slows down or pauses before hitting zero
   On 429, reads Retry-After and waits exactly that long
        |
        v
Misbehaving client (or no headers):
   Retries immediately or on hardcoded interval
   May get throttled into a longer cooldown
   May trigger fail2ban or equivalent
```

Five rules govern the system:

1. **Set Retry-After on every 429 and 503.** It tells crawlers it is temporary and tells well behaved clients exactly when to come back. Missing it is the difference between graceful recovery and a retry storm.
2. **Use 429 for rate limiting, 503 for actual outage.** They are different things. Conflating them confuses both crawlers and monitoring.
3. **Whitelist verified crawlers.** Googlebot, ClaudeBot, GPTBot are not enemies of your bandwidth budget; they are how your content gets discovered.
4. **Announce limits proactively.** Sending RateLimit-* headers on every response (not just 429) lets clients self throttle and never hit the limit.
5. **Migration: send both X-RateLimit-* and RateLimit-*.** The X- convention is what existing clients understand. The un prefixed form is the standardization track. Send both during the transition.

A correctly configured rate control stack throttles abusive clients without harming legitimate ones, communicates exact backoff windows to crawlers (preserving search ranking), and never produces unnecessary retry storms during maintenance.

---

## 5. RETRY-AFTER (THE STANDARDIZED BACKOFF SIGNAL)

### 5.1 What It Does

`Retry-After` tells the client how long to wait before retrying the request. Standardized in RFC 9110. Sent with three classes of responses:

* `503 Service Unavailable`: the service is temporarily down. Retry-After indicates expected duration of unavailability.
* `429 Too Many Requests`: the client exceeded a rate limit. Retry-After indicates when the limit window resets.
* `3xx` redirect: scheduled redirect with a minimum delay before following.

```
Retry-After: 120
Retry-After: Wed, 21 Oct 2026 07:28:00 GMT
```

Crawlers including Googlebot honor this header. Without it, crawlers guess at when to retry; with it, they wait precisely the indicated duration.

### 5.2 The Two Formats

| Format | Example | When to use |
|---|---|---|
| `<delta-seconds>` | `Retry-After: 60` | Relative wait. Most common. Use for rate limits, short maintenance |
| `<HTTP-date>` | `Retry-After: Wed, 21 Oct 2026 07:28:00 GMT` | Absolute time. Use for scheduled maintenance with known end time |

Delta seconds is recommended for most cases. It avoids clock skew issues between client and server. HTTP date is appropriate when the resume time is independent of the request time (a known maintenance window).

### 5.3 The 2 Day Rule For Search Engines

Per Google's official documentation: returning 503 or 429 for more than 2 days will cause Google to drop those URLs from the index. The mechanism:

* Days 1 to 2: Googlebot treats 503/429 as temporary, honors Retry-After, retries.
* Day 2+: Google starts treating sustained 5xx/429 as a permanent signal that the content is gone.
* Days later: URLs drop from the index.

Recovery: when the site returns 200, Googlebot recrawls. Recovery time depends on crawl budget; popular sites recrawl in hours, smaller sites can take weeks.

**The implication for Bubbles operators:** never rate limit Googlebot. Any maintenance window longer than 48 hours requires a different strategy than 503 (consider a static maintenance page returning 200 with a notice).

### 5.4 How To Build It On Bubbles

**Static 503 maintenance page with Retry-After:**

```nginx
server {
    server_name example.com;

    location / {
        # During maintenance, return 503 with Retry-After
        return 503;
    }

    error_page 503 /maintenance.html;
    location = /maintenance.html {
        root /var/www/sites/example.com;
        internal;
        add_header Retry-After "3600" always;  # 1 hour
        add_header Cache-Control "no-store" always;  # do not cache the 503
    }
}
```

**Programmatic Retry-After from FastAPI sidecar:**

```python
from fastapi import FastAPI, Response, status
from fastapi.responses import JSONResponse
import time

app = FastAPI()

# Simple in memory rate limiter (production should use Redis or similar)
client_buckets = {}

@app.middleware("http")
async def rate_limit(request, call_next):
    client_ip = request.client.host
    now = time.time()
    bucket = client_buckets.setdefault(client_ip, {"count": 0, "reset": now + 60})

    if now > bucket["reset"]:
        bucket["count"] = 0
        bucket["reset"] = now + 60

    if bucket["count"] >= 100:
        retry_after = int(bucket["reset"] - now)
        return JSONResponse(
            status_code=429,
            content={"error": "rate limit exceeded"},
            headers={
                "Retry-After": str(retry_after),
                "X-RateLimit-Limit": "100",
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(int(bucket["reset"])),
                "RateLimit-Limit": "100",
                "RateLimit-Remaining": "0",
                "RateLimit-Reset": str(retry_after),
                "RateLimit-Policy": "100;w=60",
            }
        )

    bucket["count"] += 1
    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = "100"
    response.headers["X-RateLimit-Remaining"] = str(100 - bucket["count"])
    response.headers["X-RateLimit-Reset"] = str(int(bucket["reset"]))
    response.headers["RateLimit-Limit"] = "100"
    response.headers["RateLimit-Remaining"] = str(100 - bucket["count"])
    response.headers["RateLimit-Reset"] = str(int(bucket["reset"] - now))
    response.headers["RateLimit-Policy"] = "100;w=60"
    return response
```

**Scheduled maintenance with absolute time:**

```nginx
server {
    location / {
        return 503;
    }

    error_page 503 /maintenance.html;
    location = /maintenance.html {
        root /var/www/sites/example.com;
        internal;
        # Maintenance ends at a specific time
        add_header Retry-After "Sat, 24 May 2026 22:00:00 GMT" always;
        add_header Cache-Control "no-store" always;
    }
}
```

### 5.5 How To Verify

```bash
# 1. Confirm Retry-After is sent on 429
curl -sI -X POST https://api.example.com/endpoint \
    -H "Authorization: Bearer test" \
    --data '...' | grep -iE "^(HTTP|retry-after)"
# After hitting the rate limit:
# HTTP/2 429
# retry-after: 60

# 2. Confirm Retry-After on 503 maintenance
curl -sI https://example.com/ | grep -iE "^(HTTP|retry-after)"
# HTTP/2 503
# retry-after: 3600

# 3. Test that Googlebot sees Retry-After
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/ | grep -i retry-after

# 4. Trigger rate limit deliberately to test response shape
for i in $(seq 1 200); do
    curl -so /dev/null -w "%{http_code}\n" https://api.example.com/endpoint
done | sort | uniq -c
# Expect: most 200, then 429s appear once burst is exhausted
```

### 5.6 Troubleshooting

**Symptom: Googlebot drops pages despite returning 503 with Retry-After.**
1. The 503 persisted longer than 2 days. Google treats sustained 5xx as permanent removal.
2. The Retry-After value was unrealistic (e.g., far future date). Googlebot ignores extreme values.
3. The 503 was returned to too many URLs simultaneously (server wide rather than scoped). Google interprets as site outage.
Fix: keep maintenance windows under 48 hours; use Retry-After values that match actual maintenance duration; for longer downtime, serve a 200 maintenance page instead.

**Symptom: 503s being cached by browsers despite Retry-After.**
The 503 response did not include `Cache-Control: no-store`. Browsers may cache 503s if not told otherwise.
Fix: always pair 503 with `Cache-Control: no-store, no-cache, must-revalidate`.

**Symptom: Clients retry immediately after 429 instead of waiting.**
1. Client library does not honor Retry-After.
2. The header value is invalid (non integer for delta seconds).
3. The value is 0 or negative.
Fix: ensure value is a positive integer of seconds, or a valid HTTP date. Update client to honor the header.

**Symptom: Retry-After date format rejected by some clients.**
Some clients are strict about RFC date format. The format is the same as the Date header: `Wed, 21 Oct 2026 07:28:00 GMT`. Note: no day name extensions, GMT timezone literal, comma after day name.

**Symptom: Browser displays a "service unavailable" error page during expected rate limiting.**
The client is a browser (not a script) and got a 429. The browser shows its own error page. This is correct behavior; the rate limit should not apply to browsers loading HTML pages, only to API endpoints or to clients clearly exceeding sensible thresholds.

### 5.7 How To Fix Common Breakage

**Case: Need to take site down for maintenance without losing Google rankings.**
Two options:

Option 1 (maintenance under 24 hours): serve 503 with Retry-After:

```nginx
return 503;
error_page 503 /maintenance.html;
location = /maintenance.html {
    internal;
    add_header Retry-After "3600" always;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
    root /var/www/sites/example.com;
}
```

Option 2 (maintenance over 24 hours): serve 200 with a static "we are working on it" page that includes the same content URLs in a sitemap. Google keeps the URLs indexed because they return 200.

**Case: API client library does not respect Retry-After.**
Wrap the client in a retry layer that does:

```python
import requests
import time

def request_with_backoff(method, url, **kwargs):
    for attempt in range(5):
        response = requests.request(method, url, **kwargs)
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "60"))
            time.sleep(retry_after)
            continue
        return response
    return response
```

---

## 6. X-RATELIMIT-LIMIT, X-RATELIMIT-REMAINING, X-RATELIMIT-RESET (THE DE FACTO CONVENTION)

### 6.1 What They Do

The X-RateLimit-* family announces the rate limit state to the client on every response. Not formally standardized but widely deployed since the early 2010s (GitHub, Twitter, Stripe, Shopify, Reddit, etc).

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 47
X-RateLimit-Reset: 1742658600
```

A client can read these on a successful response and decide to slow down before hitting the limit. On a 429 response, the same headers tell the client how to plan the next attempt.

### 6.2 The Three Headers

| Header | Value | Meaning |
|---|---|---|
| `X-RateLimit-Limit` | Integer | Total quota for the current window |
| `X-RateLimit-Remaining` | Integer | Requests remaining in the current window |
| `X-RateLimit-Reset` | Integer | When the window resets. Two common formats: Unix timestamp or seconds until reset |

The format of `X-RateLimit-Reset` is the single most inconsistent thing in this family. Two formats are common:

* **Unix epoch seconds**: `X-RateLimit-Reset: 1742658600` (GitHub, Reddit, Stripe)
* **Seconds until reset**: `X-RateLimit-Reset: 60` (Twitter formerly, some others)

The IETF standardization track resolves this by mandating "seconds until reset" (a delta). The X- convention is ambiguous; document which form your API uses.

**Bubbles convention:** use Unix epoch seconds for X-RateLimit-Reset (matches the most prominent deployments) and seconds until reset for the un prefixed RateLimit-Reset (matches the standardization track).

### 6.3 How To Build It On Bubbles

X-RateLimit-* headers must be set by the application layer because they depend on application state (how many requests this client has made). nginx can do basic IP based rate limiting but cannot count requests per token; that requires the upstream.

**FastAPI sidecar emitting full rate limit headers:**

```python
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse
import time
import asyncio
import redis.asyncio as redis

app = FastAPI()
r = redis.from_url("redis://127.0.0.1:6379")

WINDOW_SECONDS = 60
LIMIT = 100

async def check_rate_limit(client_id: str):
    """Returns (remaining, reset_epoch, allowed)."""
    now = int(time.time())
    window_start = now - (now % WINDOW_SECONDS)
    reset_epoch = window_start + WINDOW_SECONDS
    key = f"rl:{client_id}:{window_start}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, WINDOW_SECONDS + 5)
    count, _ = await pipe.execute()

    remaining = max(0, LIMIT - count)
    allowed = count <= LIMIT
    return remaining, reset_epoch, allowed

@app.middleware("http")
async def rate_limit(request: Request, call_next):
    # Identify client (token preferred, fall back to IP)
    auth = request.headers.get("authorization", "")
    if auth.startswith("Bearer "):
        client_id = f"token:{auth[7:32]}"
    else:
        client_id = f"ip:{request.client.host}"

    remaining, reset_epoch, allowed = await check_rate_limit(client_id)
    now = int(time.time())
    seconds_until_reset = reset_epoch - now

    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"error": "rate limit exceeded"},
            headers={
                "Retry-After": str(seconds_until_reset),
                # X- convention
                "X-RateLimit-Limit": str(LIMIT),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(reset_epoch),
                # IETF standardization track
                "RateLimit-Limit": str(LIMIT),
                "RateLimit-Remaining": "0",
                "RateLimit-Reset": str(seconds_until_reset),
                "RateLimit-Policy": f"{LIMIT};w={WINDOW_SECONDS}",
            }
        )

    response = await call_next(request)
    # X- convention
    response.headers["X-RateLimit-Limit"] = str(LIMIT)
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    response.headers["X-RateLimit-Reset"] = str(reset_epoch)
    # IETF standardization track
    response.headers["RateLimit-Limit"] = str(LIMIT)
    response.headers["RateLimit-Remaining"] = str(remaining)
    response.headers["RateLimit-Reset"] = str(seconds_until_reset)
    response.headers["RateLimit-Policy"] = f"{LIMIT};w={WINDOW_SECONDS}"
    return response
```

**Required CORS exposure** (from framework-http-cors-headers.md):

```nginx
add_header Access-Control-Expose-Headers "X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset, RateLimit-Policy, Retry-After" always;
```

Without exposing them, cross origin JavaScript cannot read these headers and the announcement is wasted.

### 6.4 How To Verify

```bash
# 1. Confirm headers are present on normal responses
curl -sI https://api.example.com/endpoint \
    -H "Authorization: Bearer test" | grep -iE "^(x-ratelimit|ratelimit|retry-after)"

# Expected:
# x-ratelimit-limit: 100
# x-ratelimit-remaining: 99
# x-ratelimit-reset: 1742658660
# ratelimit-limit: 100
# ratelimit-remaining: 99
# ratelimit-reset: 60
# ratelimit-policy: 100;w=60

# 2. Verify decrement on subsequent requests
for i in 1 2 3; do
    REMAINING=$(curl -sI https://api.example.com/endpoint \
                -H "Authorization: Bearer test" \
                | grep -i "x-ratelimit-remaining:" | grep -oE "[0-9]+")
    echo "Request $i: $REMAINING remaining"
done

# 3. Verify CORS exposure works
# Browser console on a different origin:
# fetch("https://api.example.com/endpoint", {headers: {"Authorization": "Bearer test"}})
#   .then(r => console.log("Remaining:", r.headers.get("X-RateLimit-Remaining")))
# Should print a number, not null
```

### 6.5 Troubleshooting

**Symptom: Headers absent from cross origin requests despite being set.**
The headers are not in `Access-Control-Expose-Headers`. Browser hides them from JavaScript.
Fix: add to Expose-Headers list (Section 6.3 nginx config).

**Symptom: X-RateLimit-Reset value confusion (Unix epoch vs delta).**
Clients written for GitHub style API expect Unix epoch; clients written for Twitter style expect delta.
Fix: document the format in API docs. For new APIs, send both forms (X-RateLimit-Reset as epoch, RateLimit-Reset as delta) for clarity.

**Symptom: Different clients see different remaining counts.**
The rate limit is per IP and they share an IP. Or the rate limit is per token and they share a token. Or the counter is incorrect.
Fix: verify the identification logic (IP vs token vs session) is what you intend. For multi tenant SaaS, per token rate limits are the norm.

**Symptom: Counter resets unexpectedly.**
The application restarted (counter lived in memory). The Redis backing store evicted the key. The window calculation is off.
Fix: use persistent storage for counters; configure Redis with appropriate maxmemory and policy.

### 6.6 How To Fix Common Breakage

**Case: Migrating from in memory counter to Redis.**

```python
# Before: in memory (lost on restart)
counters = {}
counters[client_id] = counters.get(client_id, 0) + 1

# After: Redis with TTL (survives restarts)
import redis.asyncio as redis
r = redis.from_url("redis://127.0.0.1:6379")

await r.incr(f"rl:{client_id}:{window_start}")
await r.expire(f"rl:{client_id}:{window_start}", WINDOW_SECONDS + 5)
```

**Case: Heterogeneous rate limits per endpoint.**
Some endpoints are cheap (search), others expensive (export). Configure per endpoint limits:

```python
LIMITS = {
    "/api/search": (1000, 60),       # 1000 per minute
    "/api/export": (10, 3600),       # 10 per hour
    "/api/upload": (50, 60),         # 50 per minute
}

def get_limit(path):
    for prefix, limit in LIMITS.items():
        if path.startswith(prefix):
            return limit
    return (100, 60)  # default
```

---

## 7. THE IETF RATELIMIT-* STANDARDIZATION (THE FUTURE)

### 7.1 Why The Standardization Matters

`X-RateLimit-*` was never standardized. Every major API implements it differently:

* Reset format: Unix epoch (GitHub, Stripe) vs delta seconds (Twitter).
* Window interpretation: sliding (some) vs fixed (others).
* What "Limit" means: per second vs per minute vs per hour.
* What "Remaining" means: current request count vs throughput percentage.

The IETF draft `draft-ietf-httpapi-ratelimit-headers` (in late drafts as of 2026) standardizes the format. The headers drop the `X-` prefix and define exact semantics. Modern APIs (GitLab, CircleCI, OKX) already emit both during the transition.

### 7.2 The Standardized Headers

```
RateLimit-Limit: 100
RateLimit-Remaining: 47
RateLimit-Reset: 60
RateLimit-Policy: 100;w=60
```

| Header | Format | Definition |
|---|---|---|
| `RateLimit-Limit` | Integer | The expiring limit for the current window |
| `RateLimit-Remaining` | Integer | Quantity of requests remaining in the current window |
| `RateLimit-Reset` | Integer | Number of seconds until the limit resets (delta only, not epoch) |
| `RateLimit-Policy` | Quoted policy | The policy in detail: `<limit>;w=<window_seconds>` |

### 7.3 The Policy Header

`RateLimit-Policy` describes the policy being applied. Useful when multiple windows are in effect or for documenting the policy without resorting to API docs.

```
RateLimit-Policy: 100;w=60
RateLimit-Policy: 100;w=60, 1000;w=3600
RateLimit-Policy: "api-default";q=100;w=60
```

The format is `<quota>;w=<window>` with optional policy name in quotes. Multiple policies can be listed.

### 7.4 The Single Combined RateLimit Header (Draft 10+)

The latest IETF draft direction is a single combined `RateLimit` header replacing the three:

```
RateLimit: "default";r=999;t=60;pk=:Ym9vbWVy:
```

Where:
* `r=` is remaining.
* `t=` is reset time (delta seconds).
* `pk=` is partition key (which quota bucket).

This format is not yet widely deployed. For Bubbles, stick with the three header form (RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset) plus RateLimit-Policy as the migration target. The X-RateLimit-* family is the present, the three RateLimit-* headers are the standardization track, the single combined header is the future once the RFC publishes.

### 7.5 How To Build It On Bubbles

See Section 6.3 FastAPI sidecar example which already emits both X- and un prefixed forms.

The key principle: **send both forms during the migration**. Clients written for the X- convention still work; clients written for the standard will work in the future. The header overhead is small.

### 7.6 Migration Glide Path

For new APIs in 2026:

1. **Now**: emit both `X-RateLimit-*` and `RateLimit-*` forms. Document both. Use whichever clients prefer.
2. **2027 and beyond**: continue emitting both. Most ecosystems still use X-.
3. **When the RFC publishes**: stop emitting X- in new versions; keep emitting on legacy endpoints. The transition will take years.

The cost of emitting both is one extra round of header lines. The benefit is forward compatibility without breaking existing clients. The cost benefit math says: emit both.

---

## 8. STATUS CODE PAIRING (429 VS 503 VS 3XX)

The status code that accompanies Retry-After matters because clients react to status code first, header second.

### 8.1 429 Too Many Requests

Use for: a specific client exceeded their rate quota. The service is operating normally; this specific request was rejected.

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1742658660

{"error": "rate_limit_exceeded", "message": "Try again in 60 seconds"}
```

Defined in RFC 6585. Specifically intended for rate limiting scenarios. Crawlers including Googlebot treat 429 as "throttle me, not a problem with the site".

**Critical:** nginx default for `limit_req` and `limit_conn` is 503, not 429. Configure with `limit_req_status 429` to return the correct status.

### 8.2 503 Service Unavailable

Use for: the entire service is temporarily unavailable. Maintenance, overload, dependency outage.

```
HTTP/1.1 503 Service Unavailable
Content-Type: text/html
Retry-After: 3600
Cache-Control: no-store, no-cache, must-revalidate

<html>...<body>We are doing scheduled maintenance...</body></html>
```

Defined in RFC 9110. Indicates a transient service wide problem. Crawlers treat 503 as "site is down, come back later". The 2-day rule applies: persistent 503 leads to index drops.

### 8.3 The Difference In One Sentence

429: "This specific client hit their quota; other clients are fine".

503: "This whole service is broken or down; everyone is affected".

Use the right one for the right scenario. Conflating them confuses monitoring, crawlers, and clients.

### 8.4 3xx Redirects With Retry-After

Less commonly used. A 3xx response with Retry-After tells the client to wait before following the redirect.

```
HTTP/1.1 301 Moved Permanently
Location: https://new.example.com/page
Retry-After: 60
```

Useful for staged redirect rollouts where the destination might be cold. Most clients ignore Retry-After on 3xx; the redirect is followed immediately.

### 8.5 How To Verify Status Code Pairing

```bash
# Test that rate limit produces 429 (not 503)
for i in $(seq 1 200); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" \
            -H "Authorization: Bearer test" \
            https://api.example.com/endpoint)
    if [ "$STATUS" != "200" ]; then
        echo "Request $i: $STATUS"
        break
    fi
done
# Expected: eventually "Request N: 429"
# Wrong: "Request N: 503" (means limit_req_status not set)

# Test that maintenance page produces 503
curl -sI https://example.com/  # during scheduled maintenance
# Expected: HTTP/2 503 with Retry-After
```

---

## 9. NGINX RATE LIMITING CONFIGURATION (LIMIT_REQ AND LIMIT_CONN)

nginx has two rate limiting modules:

* **`limit_req`**: requests per second per key. The most common.
* **`limit_conn`**: simultaneous connections per key.

Both produce 503 by default; both should be configured for 429 in API contexts.

### 9.1 The Leaky Bucket Model

`limit_req` implements a leaky bucket. Imagine a bucket that fills with incoming requests. The bucket leaks at the configured rate. If the bucket overflows, additional requests are rejected (or queued).

Configuration:

```nginx
http {
    # Zone definition: 10MB shared memory, rate 10 requests per second
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
}

server {
    location /api/ {
        # Apply zone, allow burst of 20 above the rate
        limit_req zone=api_limit burst=20 nodelay;
        limit_req_status 429;
    }
}
```

What this configuration does:

* `rate=10r/s`: the bucket leaks 10 requests per second.
* `burst=20`: the bucket can hold 20 requests above the steady rate before rejecting.
* `nodelay`: process burst requests immediately (no queuing). If the bucket is full, reject.

Without `nodelay`, burst requests are queued and processed at the leaked rate (spacing them out). With `nodelay`, burst requests are processed immediately as long as they fit in the bucket.

### 9.2 The Two Stage Pattern (nginx 1.15.7+)

The `delay` parameter (introduced 1.15.7) gives a middle ground: some burst is processed immediately, the rest is delayed:

```nginx
location /api/ {
    limit_req zone=api_limit burst=20 delay=10;
}
```

This means:

* First 10 burst requests: processed immediately (no delay).
* Next 10 burst requests: queued and spaced at the rate.
* Beyond burst 20: rejected.

Useful for accommodating typical web browser request patterns (initial page load with concurrent asset fetches) while still throttling sustained abuse.

### 9.3 The Three Common Patterns

**Pattern 1: API endpoint (strict, return 429):**

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_status 429;
}

location /api/ {
    limit_req zone=api burst=60 nodelay;
}
```

**Pattern 2: Login endpoint (very strict, prevent brute force):**

```nginx
http {
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    limit_req_status 429;
}

location /api/login {
    limit_req zone=login burst=5 nodelay;
}
```

**Pattern 3: General site (lenient, accommodate browsers):**

```nginx
http {
    limit_req_zone $binary_remote_addr zone=general:10m rate=100r/s;
    limit_req_status 429;
}

location / {
    limit_req zone=general burst=200 delay=100;
}
```

### 9.4 The Connection Limit Module

`limit_conn` controls simultaneous connections rather than request rate. Useful against download abuse (someone with 100 parallel connections downloading a large file):

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
    limit_conn_status 429;
}

location /downloads/ {
    limit_conn conn_per_ip 5;
}
```

Limit: any single IP may have at most 5 simultaneous connections to `/downloads/`.

### 9.5 Whitelisting Trusted IPs And Crawlers

Use a `geo` plus `map` combination to skip rate limiting for trusted sources:

```nginx
http {
    geo $rate_limit_skip {
        default 1;

        # Bubbles operator IPs
        100.90.0.0/16  0;   # Tailscale subnet
        127.0.0.1      0;
        ::1            0;

        # Office VPN range
        # 198.51.100.0/24  0;
    }

    map $rate_limit_skip $rate_limit_key {
        0 "";                       # whitelisted: empty key skips rate limit
        1 $binary_remote_addr;      # rate limited: use IP as key
    }

    limit_req_zone $rate_limit_key zone=api:10m rate=30r/s;
    limit_req_status 429;
}
```

When `$rate_limit_skip` is 0 (whitelisted), `$rate_limit_key` is empty, and nginx skips rate limiting entirely for that request.

### 9.6 Setting Retry-After On limit_req Rejections

nginx does not automatically set `Retry-After` on `limit_req` rejections. Use a custom error page:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_status 429;
}

server {
    location /api/ {
        limit_req zone=api burst=60 nodelay;

        proxy_pass http://127.0.0.1:9090;
    }

    error_page 429 = @rate_limited;

    location @rate_limited {
        internal;
        add_header Retry-After "60" always;
        add_header Content-Type "application/json" always;
        return 429 '{"error": "rate_limit_exceeded", "retry_after": 60}';
    }
}
```

The `@rate_limited` named location returns a JSON error body with Retry-After set. For HTML responses, swap the JSON for an HTML body.

### 9.7 Logging Rate Limit Events

Default nginx logs rate limit events at `error` level, which is noisy. Configure `warn` or `notice`:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_status 429;
    limit_req_log_level warn;
}
```

And ensure the error log level is at least `warn`:

```nginx
error_log /var/log/nginx/error.log warn;
```

Now rate limit events appear in the error log at warn level but not the noisier info events.

### 9.8 The Dry Run Mode

nginx 1.17.1+ supports `limit_req_dry_run on`: log what would have been rate limited without actually rejecting. Useful for tuning limits before enforcing:

```nginx
location /api/ {
    limit_req zone=api burst=60 nodelay;
    limit_req_dry_run on;  # log but do not reject
}
```

Run with dry_run for a week, observe the logs, adjust limits, then disable dry_run.

### 9.9 The Default Zone Sizing

`zone=name:10m` means 10 MB of shared memory. nginx documentation states 1 MB holds approximately 16,000 IP addresses. 10 MB holds about 160,000. For most Bubbles sites this is generous. For very high traffic sites, increase to 50m or 100m.

### 9.10 The 503 To 429 Conversion (One Line Fix)

The single most common configuration mistake is leaving the default 503. The fix:

```nginx
http {
    # Change default 503 to 429 for all limit_req and limit_conn rejections
    limit_req_status 429;
    limit_conn_status 429;
}
```

This single pair of lines should be in `nginx.conf` for every Bubbles deployment that serves APIs.

---

## 10. THE CRAWLER PROTECTION RULE (NEVER RATE LIMIT GOOGLEBOT, CLAUDEBOT, GPTBOT)

The single most ranking critical decision in this framework: **do not rate limit verified search and AI crawlers**. The cost of getting it wrong is index drops, ranking loss, and AI invisibility.

### 10.1 The Crawlers To Protect

Bubbles client sites depend on these crawlers for organic and AI visibility:

* **Googlebot** (search.google.com): the dominant search crawler.
* **Bingbot** (bing.com): the second search engine, also feeds DuckDuckGo and ChatGPT search.
* **ClaudeBot, Claude-Web** (anthropic.com): Anthropic's crawlers.
* **GPTBot, OAI-SearchBot** (openai.com): OpenAI's crawlers.
* **PerplexityBot, Perplexity-User** (perplexity.ai): Perplexity.
* **AmazonBot, Amazonbot** (amazon.com): for Alexa and product search.
* **Applebot** (apple.com): for Spotlight and Siri.
* **YandexBot, Baiduspider**: for international SEO (less relevant for NWA/SWMO clients).

### 10.2 Verifying Crawler Identity (User Agent Is Not Enough)

User Agent strings are trivially spoofed. Real verification uses reverse DNS plus forward DNS:

```bash
# 1. Resolve the request's IP back to a hostname
host 66.249.66.1
# Expected for Googlebot: crawl-66-249-66-1.googlebot.com

# 2. Resolve that hostname forward to an IP
host crawl-66-249-66-1.googlebot.com
# Expected: 66.249.66.1 (matches original)

# If both match AND the hostname is in *.googlebot.com or *.google.com, it is genuine Googlebot
```

For ClaudeBot, Anthropic publishes their IP ranges. For GPTBot, OpenAI publishes IP ranges. For Bingbot, the verification is similar to Googlebot (reverse DNS to msn.com or search.msn.com).

### 10.3 The nginx Pattern: User Agent Plus IP Whitelist

For Bubbles deployments, the practical approach combines user agent matching (for AI crawlers without easy reverse DNS verification) with a known good IP list:

```nginx
http {
    # Detect crawlers by user agent (basic but useful)
    map $http_user_agent $is_crawler {
        default 0;
        ~*googlebot                          1;
        ~*bingbot                            1;
        ~*claudebot                          1;
        ~*claude-web                         1;
        ~*gptbot                             1;
        ~*oai-searchbot                      1;
        ~*chatgpt-user                       1;
        ~*perplexitybot                      1;
        ~*perplexity-user                    1;
        ~*amazonbot                          1;
        ~*applebot                           1;
        ~*duckduckbot                        1;
        ~*facebookexternalhit                1;
        ~*twitterbot                         1;
        ~*linkedinbot                        1;
    }

    # Optionally verify crawler IPs as additional check
    geo $is_crawler_ip {
        default 0;
        66.249.64.0/19   1;     # Googlebot range (one of several)
        # Add Bing, Claude, OpenAI ranges as needed
    }

    # Skip rate limiting if either user agent OR IP matches
    map "$is_crawler$is_crawler_ip" $rate_limit_skip {
        "00"  0;        # not a crawler, apply rate limit
        default 1;       # any crawler signal, skip rate limit
    }

    map $rate_limit_skip $rate_limit_key {
        1 "";                       # crawler: empty key skips rate limit
        0 $binary_remote_addr;      # normal: use IP
    }

    limit_req_zone $rate_limit_key zone=api:10m rate=30r/s;
    limit_req_status 429;
}
```

This is permissive (allows anyone claiming to be a crawler to bypass rate limits via spoofed UA), but the risk is minimal for non sensitive endpoints. For sensitive endpoints (login, payment, admin), require strict IP verification.

### 10.4 The Bubbles Conservative Pattern: Whitelist UA Match Plus Generous Limits

For all Bubbles sites, the recommended pattern is permissive UA matching combined with generous overall limits:

```nginx
# Skip rate limiting for any UA matching a known crawler
map $http_user_agent $rate_limit_skip {
    default                  0;
    ~*googlebot              1;
    ~*bingbot                1;
    ~*claudebot              1;
    ~*gptbot                 1;
    ~*oai-searchbot          1;
    ~*perplexitybot          1;
    ~*applebot               1;
    ~*amazonbot              1;
}

map $rate_limit_skip $rate_limit_key {
    1 "";
    0 $binary_remote_addr;
}

# Generous limits for everyone else
limit_req_zone $rate_limit_key zone=site:10m rate=10r/s;
limit_req_status 429;

server {
    location / {
        limit_req zone=site burst=50 delay=20;
    }
}
```

If a malicious client spoofs a crawler UA to bypass rate limiting, the worst case is they get faster access to public site content (no real harm). For locations that DO need to apply to all (login, admin, payment), use a separate zone without UA whitelist.

### 10.5 The 503 Trap During Maintenance

If maintenance returns 503 for everyone (including Googlebot), the 2-day rule applies. The mitigation:

```nginx
# During maintenance: serve 503 to humans, 200 to crawlers (with maintenance notice)
map $http_user_agent $maintenance_response {
    default                  "503";
    ~*googlebot              "200";    # Googlebot sees 200 with notice
    ~*bingbot                "200";
    ~*claudebot              "200";
    ~*gptbot                 "200";
}

server {
    if ($maintenance_response = "503") {
        return 503;
    }
    # Crawlers fall through to normal handling (or a maintenance notice page)
}
```

This is unconventional and some SEO purists object. The alternative is to keep maintenance under 24 hours so the 2-day rule does not trigger, which is the cleaner approach.

### 10.6 How To Verify Crawler Bypass

```bash
# Simulate Googlebot user agent
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/ | head -3

# Try to trigger rate limit as Googlebot (should NOT see 429)
for i in $(seq 1 200); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" \
            -A "Mozilla/5.0 (compatible; Googlebot/2.1)" \
            https://example.com/)
    if [ "$STATUS" != "200" ]; then
        echo "Googlebot got: $STATUS at request $i (BAD)"
        break
    fi
done
echo "Googlebot completed all requests at 200 (GOOD)"

# Same test with a regular UA (should eventually 429)
for i in $(seq 1 200); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" \
            -A "Mozilla/5.0 (BadBot/1.0)" \
            https://example.com/)
    if [ "$STATUS" != "200" ]; then
        echo "BadBot got: $STATUS at request $i (correctly throttled)"
        break
    fi
done
```

---

## 11. CLIENT BACKOFF PATTERNS (EXPONENTIAL, JITTER, HONORING RETRY-AFTER)

The server only emits headers; the client must read and honor them. Documenting client side patterns is part of API design.

### 11.1 The Basic Retry Pattern

```python
import requests
import time

def request_with_retry(method, url, max_attempts=5, **kwargs):
    """Honor Retry-After; fall back to exponential backoff."""
    for attempt in range(max_attempts):
        response = requests.request(method, url, **kwargs)

        if response.status_code in (429, 503):
            retry_after = response.headers.get("Retry-After")
            if retry_after:
                # Honor server's directive
                try:
                    delay = int(retry_after)
                except ValueError:
                    # HTTP date format
                    from email.utils import parsedate_to_datetime
                    from datetime import datetime, timezone
                    target = parsedate_to_datetime(retry_after)
                    delay = max(0, (target - datetime.now(timezone.utc)).total_seconds())
            else:
                # Fall back to exponential backoff
                delay = 2 ** attempt

            time.sleep(delay)
            continue

        return response

    return response
```

### 11.2 Adding Jitter

If many clients hit a 429 simultaneously and all read the same Retry-After, they all retry at the same time, triggering another wave of 429s. Adding random jitter spreads them out:

```python
import random

def jittered_delay(retry_after_seconds):
    """Add up to 25% random delay to spread retries."""
    jitter = random.uniform(0, 0.25) * retry_after_seconds
    return retry_after_seconds + jitter
```

### 11.3 Proactive Self Throttling Using RateLimit-Remaining

Better than reactive retry on 429: read RateLimit-Remaining and slow down before hitting zero:

```python
def request_with_self_throttle(method, url, **kwargs):
    """Self throttle based on remaining quota."""
    response = requests.request(method, url, **kwargs)

    remaining = int(response.headers.get("X-RateLimit-Remaining", "999"))
    limit = int(response.headers.get("X-RateLimit-Limit", "1"))
    reset = response.headers.get("X-RateLimit-Reset")

    # Slow down if running low
    if remaining < limit * 0.1:  # under 10%
        # Calculate seconds until reset
        if reset:
            try:
                # Unix epoch format
                seconds_until_reset = int(reset) - int(time.time())
            except (ValueError, OverflowError):
                seconds_until_reset = 60

            # Spread remaining quota over time to reset
            if remaining > 0:
                delay = max(1, seconds_until_reset / remaining)
                time.sleep(delay)

    return response
```

### 11.4 Honoring The Standardized Format

For clients written for the IETF standardization track:

```python
def get_retry_delay(response):
    """Get retry delay from any of the rate limit header forms."""
    # Try Retry-After first (standardized)
    retry_after = response.headers.get("Retry-After")
    if retry_after:
        try:
            return int(retry_after)
        except ValueError:
            pass  # date format, parse separately

    # Try RateLimit-Reset (IETF standard, delta seconds)
    reset = response.headers.get("RateLimit-Reset")
    if reset:
        return int(reset)

    # Fall back to X-RateLimit-Reset (could be epoch)
    reset = response.headers.get("X-RateLimit-Reset")
    if reset:
        delta = int(reset) - int(time.time())
        if delta > 0:
            return delta
        return int(reset)  # might already be delta

    # No header, fall back to default
    return 60
```

---

## 12. HOW THESE HEADERS INTERACT

Several specific interactions matter.

### 12.1 Retry-After And RateLimit-Reset Both On The Same Response

For a 429 response, both are appropriate:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 60
RateLimit-Policy: 100;w=60
```

Retry-After is the standardized signal; RateLimit-Reset is the standardized rate limit detail. Both communicate "60 seconds until you can retry" in compatible ways. Smart clients prefer Retry-After (it is the RFC).

### 12.2 Vary: Authorization For Token Based Limits

If the rate limit varies per authorization token (which it usually does in multi tenant APIs), the response varies based on the Authorization header. Add Vary:

```nginx
add_header Vary "Authorization" always;
```

Without this, intermediate caches may serve cached rate limit data approved for token A to a request bearing token B.

### 12.3 CORS Expose-Headers Requirement

For cross origin requests to read the rate limit headers from JavaScript, they must be in Access-Control-Expose-Headers (see framework-http-cors-headers.md):

```nginx
add_header Access-Control-Expose-Headers "X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset, RateLimit-Policy, Retry-After" always;
```

Without exposure, browser JavaScript sees null for these headers even though the network panel shows them.

### 12.4 Cache-Control On 429 And 503 Responses

Always pair with `Cache-Control: no-store` to prevent intermediate caches from serving stale errors:

```nginx
location @rate_limited {
    add_header Retry-After "60" always;
    add_header Cache-Control "no-store, no-cache, must-revalidate" always;
    return 429 '...';
}
```

A cached 429 served to a different client (one who has not hit their limit) creates a confusing failure. A cached 503 served after maintenance ends extends the outage.

### 12.5 X-Robots-Tag On Rate Limited Responses

Add `X-Robots-Tag: noindex` to 429 and 503 responses so search engines do not accidentally index the error page:

```nginx
location @rate_limited {
    add_header Retry-After "60" always;
    add_header X-Robots-Tag "noindex, nofollow" always;
    return 429 '...';
}
```

This pairs with framework-http-seo-headers.md.

### 12.6 The limit_req And FastAPI Coordination

If both nginx and FastAPI implement rate limiting, they must agree on policy or one will mask the other:

* **nginx layer**: per IP, blunt instrument, fast (no application traversal).
* **FastAPI layer**: per token, fine grained, more expensive (runs in application).

Recommendation: nginx for crude DDoS protection (very generous limits, block obvious abuse). FastAPI for per token / per user limits announced via headers. They serve different purposes and do not conflict.

---

## 13. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 13.1 Standard Bubbles API rate limit (per IP, return 429)

```nginx
# In http block of nginx.conf
map $http_user_agent $rate_limit_skip {
    default                  0;
    ~*googlebot              1;
    ~*bingbot                1;
    ~*claudebot              1;
    ~*gptbot                 1;
    ~*oai-searchbot          1;
    ~*perplexitybot          1;
}

map $rate_limit_skip $rate_limit_key {
    1 "";
    0 $binary_remote_addr;
}

limit_req_zone $rate_limit_key zone=api:10m rate=30r/s;
limit_req_status 429;
limit_req_log_level warn;
```

```nginx
# In server block
location /api/ {
    limit_req zone=api burst=60 nodelay;
    proxy_pass http://127.0.0.1:9090;
}

error_page 429 = @rate_limited;

location @rate_limited {
    internal;
    add_header Retry-After "60" always;
    add_header Content-Type "application/json" always;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
    return 429 '{"error": "rate_limit_exceeded", "retry_after": 60}';
}
```

### 13.2 Login endpoint (strict, prevent brute force)

```nginx
# In http block
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
limit_req_status 429;
```

```nginx
# In server block
location /api/login {
    limit_req zone=login burst=5 nodelay;
    proxy_pass http://127.0.0.1:9090;
}

location /api/password-reset {
    limit_req zone=login burst=3 nodelay;
    proxy_pass http://127.0.0.1:9090;
}
```

Plus paired with fail2ban for IPs that persistently hit the login limit:

```ini
# /etc/fail2ban/jail.d/nginx-rate-limit.conf
[nginx-rate-limit]
enabled = true
filter = nginx-rate-limit
action = iptables-multiport[name=NginxRateLimit, port="http,https"]
logpath = /var/log/nginx/error.log
maxretry = 10
findtime = 600
bantime = 3600
```

### 13.3 Static site general protection (lenient, accommodate browsers)

```nginx
limit_req_zone $rate_limit_key zone=site:10m rate=20r/s;
limit_req_status 429;
```

```nginx
server {
    location / {
        limit_req zone=site burst=50 delay=25;
        try_files $uri $uri/ $uri.html =404;
    }

    error_page 429 = @rate_limited_html;

    location @rate_limited_html {
        internal;
        add_header Retry-After "30" always;
        add_header Cache-Control "no-store" always;
        return 429 '<html><body><h1>Slow down!</h1><p>Too many requests. Try again in 30 seconds.</p></body></html>';
    }
}
```

### 13.4 Download protection (limit_conn for parallel downloads)

```nginx
limit_conn_zone $binary_remote_addr zone=download_conn:10m;
limit_conn_status 429;
```

```nginx
location /downloads/ {
    limit_conn download_conn 3;
    limit_rate 1m;  # 1 MB/s per connection cap as a bonus
}

error_page 429 = @too_many_conns;

location @too_many_conns {
    internal;
    add_header Retry-After "60" always;
    return 429 '{"error": "too_many_connections", "max": 3}';
}
```

### 13.5 Scheduled maintenance with Retry-After and Googlebot 200 fallback

```nginx
map $http_user_agent $maintenance_response {
    default                  "503";
    ~*googlebot              "200";
    ~*bingbot                "200";
    ~*claudebot              "200";
    ~*gptbot                 "200";
    ~*perplexitybot          "200";
}

server {
    location / {
        if ($maintenance_response = "503") {
            return 503;
        }
        # Crawlers see the maintenance notice with 200 status
        root /var/www/sites/example.com;
        try_files /maintenance.html =503;
    }

    error_page 503 /maintenance.html;
    location = /maintenance.html {
        internal;
        add_header Retry-After "1800" always;  # 30 minutes
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
        root /var/www/sites/example.com;
    }
}
```

### 13.6 FastAPI sidecar with proper rate limit headers using slowapi

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

app = FastAPI()
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.exception_handler(RateLimitExceeded)
async def rate_limit_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=429,
        content={"error": "rate_limit_exceeded"},
        headers={
            "Retry-After": "60",
            "X-RateLimit-Limit": "100",
            "X-RateLimit-Remaining": "0",
            "RateLimit-Limit": "100",
            "RateLimit-Remaining": "0",
            "RateLimit-Reset": "60",
            "RateLimit-Policy": "100;w=60",
        }
    )

@app.get("/api/data")
@limiter.limit("100/minute")
async def data(request: Request):
    return {"data": "..."}
```

### 13.7 thatwebhostingguy.com wildcard subdomain protection

```nginx
# Wildcard subdomains share a single rate limit zone
limit_req_zone $rate_limit_key zone=wildcard:50m rate=50r/s;
limit_req_status 429;

server {
    server_name *.thatwebhostingguy.com;

    location / {
        limit_req zone=wildcard burst=100 delay=50;
        # ... rest of config ...
    }
}
```

50 MB zone holds approximately 800,000 IPs which is generous for the demo platform.

### 13.8 CORS exposed rate limit headers for SPA

```nginx
# For an API consumed by a JS frontend on a different origin
add_header Access-Control-Expose-Headers "X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset, RateLimit-Policy, Retry-After" always;
add_header Vary "Origin, Authorization" always;
```

The frontend can now read rate limit state and self throttle:

```javascript
async function apiCall(url, options) {
    const response = await fetch(url, options);
    const remaining = parseInt(response.headers.get("X-RateLimit-Remaining") || "999");
    const limit = parseInt(response.headers.get("X-RateLimit-Limit") || "1");

    if (remaining < limit * 0.1) {
        console.warn(`Rate limit at ${remaining}/${limit}, slowing down`);
    }

    if (response.status === 429) {
        const retryAfter = parseInt(response.headers.get("Retry-After") || "60");
        await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
        return apiCall(url, options);
    }

    return response;
}
```

### 13.9 Search endpoint with permissive crawler access

```nginx
location /search {
    # Crawlers indexing search pages should not be throttled
    if ($is_crawler) {
        proxy_pass http://127.0.0.1:9090;
        break;
    }

    # Regular users: rate limited
    limit_req zone=search burst=30 delay=15;
    proxy_pass http://127.0.0.1:9090;
}
```

Note: `if ($is_crawler) { proxy_pass; break; }` is one of the unsafe `if` patterns. Better approach: use map plus a wrapper location, or rely on the global skip pattern from Section 13.1.

### 13.10 Dry run mode for tuning a new rate limit

```nginx
limit_req_zone $rate_limit_key zone=new_api:10m rate=10r/s;
limit_req_status 429;

location /api/v2/ {
    limit_req zone=new_api burst=20 nodelay;
    limit_req_dry_run on;  # log but do not enforce
    proxy_pass http://127.0.0.1:9090;
}
```

Run for a week. Check error log for `limiting requests, dry_run` entries. Tune the rate and burst values based on observed traffic, then remove `limit_req_dry_run on;` to enforce.

---

## 14. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete rate control stanza, layered with the previous six frameworks.

```nginx
# /etc/nginx/nginx.conf (http context)
http {
    # Crawler detection
    map $http_user_agent $rate_limit_skip {
        default                  0;
        ~*googlebot              1;
        ~*bingbot                1;
        ~*duckduckbot            1;
        ~*claudebot              1;
        ~*claude-web             1;
        ~*gptbot                 1;
        ~*oai-searchbot          1;
        ~*chatgpt-user           1;
        ~*perplexitybot          1;
        ~*perplexity-user        1;
        ~*amazonbot              1;
        ~*applebot               1;
        ~*facebookexternalhit    1;
        ~*twitterbot             1;
        ~*linkedinbot            1;
    }

    map $rate_limit_skip $rate_limit_key {
        1 "";
        0 $binary_remote_addr;
    }

    # Rate limit zones
    limit_req_zone $rate_limit_key zone=site:10m rate=20r/s;
    limit_req_zone $rate_limit_key zone=api:10m rate=30r/s;
    limit_req_zone $rate_limit_key zone=login:10m rate=1r/s;

    # Connection limit zone
    limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;

    # Use 429 instead of 503 for all rate limit rejections
    limit_req_status 429;
    limit_conn_status 429;
    limit_req_log_level warn;
    limit_conn_log_level warn;
}
```

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

    # Security baseline
    include snippets/common-security-headers.conf;

    root /var/www/sites/example.com;
    index index.html;

    # ===== Site general (lenient) =====
    location / {
        limit_req zone=site burst=50 delay=25;
        limit_conn conn_per_ip 20;
        try_files $uri $uri/ $uri.html =404;
    }

    # ===== API endpoints (moderate, return 429) =====
    location /api/ {
        limit_req zone=api burst=60 nodelay;
        limit_conn conn_per_ip 20;

        # Per CORS framework
        if ($cors_origin) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Credentials "true" always;
            add_header Access-Control-Expose-Headers "X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset, RateLimit-Policy, Retry-After, Server-Timing, ETag, Link, X-Total-Count" always;
            add_header Vary "Origin, Authorization" always;
        }

        add_header X-Robots-Tag "noindex, nofollow" always;
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ===== Login (strict) =====
    location /api/login {
        limit_req zone=login burst=5 nodelay;
        # All other API headers inherited or repeated
        proxy_pass http://127.0.0.1:9090;
    }

    location /api/password-reset {
        limit_req zone=login burst=3 nodelay;
        proxy_pass http://127.0.0.1:9090;
    }

    # ===== Downloads (connection limited) =====
    location /downloads/ {
        limit_conn conn_per_ip 3;
        limit_rate_after 5m;
        limit_rate 2m;  # 2 MB/s after first 5 MB
        # Other download specific config
    }

    # ===== Error handlers =====
    error_page 429 = @rate_limited;
    error_page 503 = @maintenance;

    location @rate_limited {
        internal;
        add_header Retry-After "60" always;
        add_header Content-Type "application/json" always;
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
        return 429 '{"error": "rate_limit_exceeded", "retry_after": 60}';
    }

    location @maintenance {
        internal;
        add_header Retry-After "1800" always;
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
        try_files /maintenance.html =503;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify:

```bash
# Crawler bypass works
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1)" https://example.com/ | head -3

# Rate limit triggers and returns 429 with Retry-After
for i in $(seq 1 100); do
    curl -so /dev/null -w "%{http_code} " https://example.com/api/data
done; echo
```

---

## 15. AUDIT CHECKLIST

Run through these 50 items for any production rate control configuration.

### Retry-After

1. [ ] Retry-After present on every 429 response.
2. [ ] Retry-After present on every 503 response.
3. [ ] Values are positive integers (or valid HTTP dates).
4. [ ] Values reflect actual quota window reset times (not arbitrary or stale).
5. [ ] No Retry-After on success responses (2xx) where it would be confusing.
6. [ ] 503 responses paired with `Cache-Control: no-store` to prevent stale errors.
7. [ ] Maintenance pages return 503 with Retry-After AND complete in under 48 hours.

### X-RateLimit-* family

8. [ ] X-RateLimit-Limit set on every API response.
9. [ ] X-RateLimit-Remaining decrements correctly across requests.
10. [ ] X-RateLimit-Reset format documented (Unix epoch vs delta seconds).
11. [ ] Headers added to Access-Control-Expose-Headers for cross origin access.
12. [ ] Per token rate limits implemented (not just per IP) where authentication exists.

### IETF RateLimit-* family

13. [ ] RateLimit-Limit emitted in parallel with X-RateLimit-Limit.
14. [ ] RateLimit-Remaining emitted in parallel with X-RateLimit-Remaining.
15. [ ] RateLimit-Reset uses delta seconds format (matches standard).
16. [ ] RateLimit-Policy describes the policy (quota + window).
17. [ ] Both X- and un prefixed forms exposed via CORS.

### Status codes

18. [ ] Nginx `limit_req_status 429` set (not the default 503).
19. [ ] Nginx `limit_conn_status 429` set (not the default 503).
20. [ ] 429 used for rate limiting (per client quota).
21. [ ] 503 used for service outage (whole service unavailable).
22. [ ] No 503 served for more than 48 hours (to avoid Google deindex).

### Nginx configuration

23. [ ] limit_req zones defined with reasonable sizes (10m default).
24. [ ] limit_req_zone keys appropriate ($binary_remote_addr, not $remote_addr).
25. [ ] Burst parameter set on every limit_req usage.
26. [ ] Use of nodelay or delay matches the use case (API: nodelay; browsers: delay).
27. [ ] Log level set to `warn` not the default `error`.
28. [ ] error_log level supports warn (`error_log /var/log/nginx/error.log warn`).
29. [ ] Custom 429 error page returns Retry-After.
30. [ ] Custom 503 error page returns Retry-After.

### Crawler protection

31. [ ] User agent whitelist for known crawlers (Googlebot, Bingbot, ClaudeBot, GPTBot, PerplexityBot, etc).
32. [ ] Crawler whitelist applied via `map` to skip rate limiting.
33. [ ] Tested that Googlebot UA bypasses rate limits.
34. [ ] Sensitive endpoints (login, payment) NOT whitelisted by UA (strict on those).
35. [ ] No 429/503 served to Googlebot in normal operation.

### Application layer

36. [ ] FastAPI sidecar emits rate limit headers (when responsible for per token limits).
37. [ ] Rate limit storage is persistent (Redis or similar), not in process memory.
38. [ ] Rate limit keys are sensible (per token preferred over per IP for authenticated APIs).
39. [ ] Different endpoints have different limits where appropriate (cheap vs expensive).

### Client coordination

40. [ ] API documentation describes rate limit headers and format.
41. [ ] API documentation includes example backoff code.
42. [ ] Client libraries (if shipped) honor Retry-After.
43. [ ] Client libraries add jitter to retry delays.

### Cross cutting

44. [ ] Vary: Authorization set when rate limit is per token.
45. [ ] X-Robots-Tag: noindex on 429 and 503 error pages.
46. [ ] No Set-Cookie on 429 responses (avoid setting cookies for failed requests).
47. [ ] No 429 cached by intermediate proxies (Cache-Control: no-store).
48. [ ] Monitoring distinguishes 429 (expected) from 503 (problem).
49. [ ] Alerts on sustained 503 above threshold but not on routine 429.
50. [ ] Quarterly review of rate limit thresholds against actual traffic patterns.

A site that passes all 50 has correctly configured rate control for production.

---

## 16. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Nginx returning 503 for rate limited requests.**
Symptom: monitoring alerts on "service unavailable" rate but the service is fine.
Why it breaks: nginx defaults to 503 for `limit_req` rejections.
Fix: `limit_req_status 429;` in http block.

**Pitfall 2: Googlebot getting rate limited for two days, pages drop from index.**
Symptom: rankings collapse, search traffic drops, GSC shows crawl errors.
Why it breaks: no crawler whitelist; aggressive limit applied to everyone equally.
Fix: implement user agent whitelist (Section 10.3). Re submit affected URLs via GSC after fix.

**Pitfall 3: 429 with no Retry-After.**
Symptom: clients retry immediately, triggering more 429s, ratchet effect.
Why it breaks: clients have no signal to wait. Default behavior is "retry now".
Fix: always set Retry-After on 429 responses.

**Pitfall 4: Maintenance 503 cached by browser.**
Symptom: site is back up but users still see 503 from cache.
Why it breaks: missing Cache-Control: no-store.
Fix: add `add_header Cache-Control "no-store" always;` to 503 responses.

**Pitfall 5: X-RateLimit headers invisible to JavaScript.**
Symptom: frontend shows null for response.headers.get("X-RateLimit-Remaining").
Why it breaks: missing from Access-Control-Expose-Headers.
Fix: add to Expose-Headers in CORS configuration.

**Pitfall 6: Per IP limits break office networks.**
Symptom: large office reports "rate limit exceeded" while individual users are not heavy.
Why it breaks: many users share one NAT IP; per IP limits are too low for the aggregate.
Fix: increase per IP limits for unauthenticated endpoints; use per token limits for authenticated.

**Pitfall 7: X-RateLimit-Reset format confusion.**
Symptom: client backoff uses wrong duration (treating epoch as seconds, or vice versa).
Why it breaks: no standard for this header; format is API specific.
Fix: document the format. Use Unix epoch for X-, delta for RateLimit-Reset. Both makes intent clear.

**Pitfall 8: Sustained 503 for over 48 hours during scheduled work.**
Symptom: Google deindexes URLs.
Why it breaks: 2-day rule.
Fix: keep maintenance under 48 hours, OR return 200 with maintenance notice for crawlers.

**Pitfall 9: Rate limit zone too small, hash collisions.**
Symptom: legitimate clients occasionally rate limited despite being under threshold.
Why it breaks: 10m zone overflows for very high traffic sites, causing key eviction.
Fix: increase zone size (50m, 100m) for high traffic.

**Pitfall 10: Using `if ($is_crawler) { ... }` with proxy_pass.**
Symptom: unexpected behavior, sometimes works sometimes does not.
Why it breaks: `if` plus `proxy_pass` is the nginx if antipattern.
Fix: use map plus separate locations, or the global skip pattern.

**Pitfall 11: Login endpoint with no special rate limit.**
Symptom: credential stuffing attack succeeds because attacker can try thousands of password combinations per minute.
Why it breaks: general rate limit (10r/s) is too permissive for login.
Fix: separate strict zone for login (1r/s, burst 5, nodelay).

**Pitfall 12: limit_conn defending against parallel download misses HTTP/2.**
Symptom: client uses HTTP/2 multiplexing, all requests come over 1 connection, bypasses limit_conn.
Why it breaks: limit_conn counts connections, HTTP/2 multiplexes many requests per connection.
Fix: combine limit_conn with limit_req for layered defense.

**Pitfall 13: Counter resets on application restart.**
Symptom: rate limits become ineffective after every deploy; clients see Limit reset to maximum.
Why it breaks: in process counters lost on restart.
Fix: persist counters in Redis or similar shared storage.

**Pitfall 14: Rate limit headers absent on error responses.**
Symptom: client got 500 and has no info about quota state.
Why it breaks: error responses bypass middleware that adds the headers.
Fix: add headers in exception handlers as well.

**Pitfall 15: Retry-After value of 0.**
Symptom: client retries immediately, defeating the rate limit.
Why it breaks: integer 0 means no wait.
Fix: minimum value of 1; for very short delays use 1 second.

---

## 17. DIAGNOSTIC COMMANDS

Reference of every command useful for rate control investigation.

### Inspect rate limit headers

```bash
# Single request, all rate control headers
curl -sI -H "Authorization: Bearer test" https://api.example.com/data \
    | grep -iE "^(x-ratelimit|ratelimit|retry-after|cache-control|vary)"

# Pretty print
curl -sI -H "Authorization: Bearer test" https://api.example.com/data \
    | grep -iE "^(x-ratelimit|ratelimit|retry-after)" \
    | sort
```

### Trigger rate limit deliberately

```bash
# Send 200 requests rapidly, count status codes
for i in $(seq 1 200); do
    curl -so /dev/null -w "%{http_code}\n" \
         -H "Authorization: Bearer test" \
         https://api.example.com/data
done | sort | uniq -c
# Expect: most 200, some 429 once limit exceeded
```

### Verify crawler bypass

```bash
# As Googlebot: should not get 429
for i in $(seq 1 500); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" \
             -A "Mozilla/5.0 (compatible; Googlebot/2.1)" \
             https://example.com/)
    [ "$STATUS" != "200" ] && { echo "Bad: got $STATUS at $i"; break; }
done

# As regular UA: should eventually 429
for i in $(seq 1 500); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" \
             https://example.com/)
    if [ "$STATUS" = "429" ]; then
        echo "Correctly throttled at request $i"
        break
    fi
done
```

### Check Retry-After value

```bash
# Trigger rate limit then read Retry-After
for i in $(seq 1 100); do curl -so /dev/null https://api.example.com/data; done
curl -sI https://api.example.com/data | grep -iE "retry-after|^HTTP"
# Expected:
# HTTP/2 429
# retry-after: 60
```

### Test exponential backoff with Retry-After

```bash
# Bash retry helper
request_with_retry() {
    local url=$1
    local attempts=0
    while [ $attempts -lt 5 ]; do
        local headers=$(curl -sI "$url" 2>/dev/null)
        local status=$(echo "$headers" | head -1 | awk '{print $2}')

        if [ "$status" = "200" ]; then
            echo "Success after $attempts retries"
            return 0
        fi

        if [ "$status" = "429" ] || [ "$status" = "503" ]; then
            local retry=$(echo "$headers" | grep -i retry-after | tr -d '\r\n' | awk '{print $2}')
            retry=${retry:-60}
            echo "Got $status, waiting $retry seconds"
            sleep "$retry"
            attempts=$((attempts + 1))
            continue
        fi

        echo "Unexpected status: $status"
        return 1
    done
}
```

### Server side investigation

```bash
# Show rate limit zones
nginx -T 2>/dev/null | grep -E "limit_req_zone|limit_conn_zone"

# Show rate limit usages
nginx -T 2>/dev/null | grep -E "limit_req |limit_conn "

# Show status overrides
nginx -T 2>/dev/null | grep -E "limit_req_status|limit_conn_status"

# Check crawler whitelist exists
nginx -T 2>/dev/null | grep -A 30 "map \$http_user_agent"

# Find rate limit events in log
sudo grep -i "limiting requests" /var/log/nginx/error.log | tail -20

# Count rate limit events by IP
sudo grep -i "limiting requests" /var/log/nginx/error.log \
    | grep -oE 'client: [0-9.]+' | sort | uniq -c | sort -rn | head -10

# Per minute rate limit event rate
sudo grep -i "limiting requests" /var/log/nginx/error.log \
    | awk '{print $1, $2}' | cut -d: -f1,2 | uniq -c | tail -10

# Apply changes
nginx -t && systemctl reload nginx
```

### Browser DevTools quick reference

In Chrome DevTools Network panel:

1. Make a request that hits rate limit.
2. Click the 429 response in the request list.
3. Headers tab shows Retry-After, X-RateLimit-*, RateLimit-*.
4. Console panel: errors related to rate limiting appear.

Useful console commands:

```javascript
// Check current rate limit state
async function checkRateLimit(url) {
    const r = await fetch(url, {credentials: "include"});
    console.log({
        status: r.status,
        limit: r.headers.get("X-RateLimit-Limit"),
        remaining: r.headers.get("X-RateLimit-Remaining"),
        reset: r.headers.get("X-RateLimit-Reset"),
        retryAfter: r.headers.get("Retry-After"),
    });
}

// Self throttling client
async function safeApiCall(url, options = {}) {
    const r = await fetch(url, options);

    if (r.status === 429) {
        const retry = parseInt(r.headers.get("Retry-After") || "60");
        console.warn(`Rate limited, waiting ${retry}s`);
        await new Promise(resolve => setTimeout(resolve, retry * 1000));
        return safeApiCall(url, options);
    }

    return r;
}
```

---

## 18. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): Cache-Control no-store pairing with 429/503 responses.
* [framework-http-content-headers.md](framework-http-content-headers.md): Content-Type for JSON error bodies on 429.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): X-Robots-Tag noindex on rate limit error pages. The 2-day Google deindex rule is critical context here.
* [framework-http-security-headers.md](framework-http-security-headers.md): rate limiting is complementary to CSP and HSTS for defense in depth.
* [framework-http-performance-headers.md](framework-http-performance-headers.md): Server-Timing can include rate limit lookup time. Server header suppression also applies to error responses.
* [framework-http-cors-headers.md](framework-http-cors-headers.md): Access-Control-Expose-Headers must list rate limit headers for cross origin reads. Vary: Authorization for per token limits.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. The 2-day rule for crawlers is also covered there.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook.
* [framework-fail2ban-deployment.md](framework-fail2ban-deployment.md): pairing with fail2ban for persistent abusers.
* [framework-bot-management.md](framework-bot-management.md): broader bot management strategy including crawler verification.
* RFC 9110 (HTTP Semantics, Retry-After): https://www.rfc-editor.org/rfc/rfc9110
* RFC 6585 (Additional HTTP Status Codes, 429): https://www.rfc-editor.org/rfc/rfc6585
* IETF draft RateLimit headers: https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
* nginx limit_req module: https://nginx.org/en/docs/http/ngx_http_limit_req_module.html
* nginx limit_conn module: https://nginx.org/en/docs/http/ngx_http_limit_conn_module.html
* Google crawl rate troubleshooting: https://developers.google.com/search/docs/crawling-indexing/troubleshoot-crawling-errors
* MDN Retry-After: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After
* MDN 429 Too Many Requests: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
* MDN 503 Service Unavailable: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/503

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Bubbles minimum viable rate control

```nginx
# http context
http {
    map $http_user_agent $rate_limit_skip {
        default              0;
        ~*googlebot          1;
        ~*bingbot            1;
        ~*claudebot          1;
        ~*gptbot             1;
        ~*oai-searchbot      1;
        ~*perplexitybot      1;
    }
    map $rate_limit_skip $rate_limit_key {
        1 "";
        0 $binary_remote_addr;
    }
    limit_req_zone $rate_limit_key zone=site:10m rate=20r/s;
    limit_req_zone $rate_limit_key zone=api:10m rate=30r/s;
    limit_req_zone $rate_limit_key zone=login:10m rate=1r/s;
    limit_req_status 429;
    limit_conn_status 429;
    limit_req_log_level warn;
}

# server context
location / {
    limit_req zone=site burst=50 delay=25;
}
location /api/ {
    limit_req zone=api burst=60 nodelay;
}
location /api/login {
    limit_req zone=login burst=5 nodelay;
}

error_page 429 = @rate_limited;
location @rate_limited {
    internal;
    add_header Retry-After "60" always;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
    return 429 '{"error": "rate_limit_exceeded", "retry_after": 60}';
}
```

### Header purpose table

| Header | One line purpose |
|---|---|
| Retry-After | When to come back after 429/503 |
| X-RateLimit-Limit | Current quota (de facto convention) |
| X-RateLimit-Remaining | Requests left in window |
| X-RateLimit-Reset | When window resets (Unix epoch or delta) |
| RateLimit-Limit | Standardized version of X-RateLimit-Limit |
| RateLimit-Remaining | Standardized version of X-RateLimit-Remaining |
| RateLimit-Reset | Standardized: delta seconds (not epoch) |
| RateLimit-Policy | Quota and window in detail |

### Status code decision

| Situation | Status |
|---|---|
| Specific client over quota | 429 |
| Entire service down | 503 |
| Service down for over 48 hours | NOT 503, serve 200 with notice |

### The 2-day rule

Google drops URLs from the index after 48 hours of sustained 429 or 503. Always:

1. Whitelist Googlebot from rate limits.
2. Keep maintenance windows under 48 hours.
3. For longer downtime, return 200 with a maintenance page.

### Five commands every operator should know

```bash
# 1. View rate limit state
curl -sI -H "Authorization: Bearer test" https://api.example.com/data | grep -iE "ratelimit|retry-after"

# 2. Trigger rate limit
for i in $(seq 1 200); do curl -so /dev/null -w "%{http_code} " https://api.example.com/data; done; echo

# 3. Verify Googlebot bypass
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1)" https://example.com/ | head -3

# 4. Check rate limit events in nginx error log
sudo grep "limiting requests" /var/log/nginx/error.log | tail -20

# 5. Apply changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Rate limit triggers correctly
for i in $(seq 1 500); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" https://example.com/api/data)
    [ "$STATUS" = "429" ] && { echo "Rate limited at $i"; break; }
done

# 2. Retry-After is present and reasonable
curl -sI -X GET https://api.example.com/data -H "X-Test-Many: yes"
# Look for: retry-after: <positive integer>

# 3. Googlebot does not get throttled
COUNT_429=0
for i in $(seq 1 200); do
    STATUS=$(curl -so /dev/null -w "%{http_code}" \
             -A "Mozilla/5.0 (compatible; Googlebot/2.1)" \
             https://example.com/)
    [ "$STATUS" = "429" ] && COUNT_429=$((COUNT_429 + 1))
done
echo "Googlebot 429 count: $COUNT_429 (should be 0)"
```

If all three pass AND no recent Googlebot 429 events in error log, the rate control stack is correctly wired.

---

End of framework-http-rate-control-headers.md.
