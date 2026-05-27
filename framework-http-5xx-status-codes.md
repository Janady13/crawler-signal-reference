# framework-http-5xx-status-codes.md

Comprehensive reference for the HTTP 5xx Server Error status codes, the **final framework in the status codes series** after framework-http-2xx-status-codes.md, framework-http-3xx-status-codes.md, and framework-http-4xx-status-codes.md. Covers `500 Internal Server Error` (the application crashed), `502 Bad Gateway` (the upstream is unreachable or returned garbage), `503 Service Unavailable` (the maintenance code with Retry-After), and `504 Gateway Timeout` (the upstream took too long), plus brief coverage of 505, 507, 508, 511, and the Cloudflare specific 520 through 527 codes Bubbles operators may encounter when reviewing third party systems. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to the eight HTTP header frameworks plus the three previous status code frameworks, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

**This framework completes the 12 file Bubbles HTTP wire reference.** Combined with the eight header frameworks and three previous status code frameworks, every HTTP response Bubbles can produce (and every request it can receive) is now documented with paste ready nginx config, FastAPI patterns, audit checklists, and common pitfalls.

Audience: humans diagnosing production 5xx incidents on Bubbles, AI assistants generating incident response runbooks, SREs configuring self healing for FastAPI sidecars, SEO operators planning maintenance windows that preserve search rankings, and anyone troubleshooting "site returns 502 randomly", "500 error after deployment", "504 timeout during database migration", or "Googlebot stopped crawling after the holiday maintenance".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The 5xx Mental Model (read this first)
5. 500 Internal Server Error (the application crashed)
6. 502 Bad Gateway (the upstream is unreachable or broken)
7. 503 Service Unavailable (the maintenance code, Retry-After mandatory)
8. 504 Gateway Timeout (the upstream is too slow)
9. Other 5xx Codes (505, 507, 508, 511 briefly)
10. Cloudflare Specific 5xx Codes (520 through 527, briefly)
11. The 500 vs 502 vs 504 Distinction (deep dive)
12. The 503 Maintenance Pattern (deep dive, 2 day Google deindex rule)
13. The Bubbles FastAPI Sidecar Diagnostic Pattern
14. Self Healing With Systemd And Nginx Upstream Backup
15. How Major Crawlers React To Each 5xx Code
16. Asset Class And Use Case Recipes
17. Bubbles Nginx Reference Block (paste ready)
18. Audit Checklist (50+ items)
19. Common Pitfalls
20. Diagnostic Commands
21. Cross-References
22. The Complete 12 File Bubbles HTTP Wire Reference (series summary)

---

## 1. DEFINITION

5xx status codes signal that the server failed to fulfill an apparently valid request. The client did nothing wrong; the server has a problem. Defined across RFC 9110 and individual RFCs for specific codes.

The four codes Joseph listed split into two functional groups:

* **Application or origin failure**: 500 Internal Server Error. The server side application threw an uncaught exception, ran into a bug, or otherwise crashed handling the request.
* **Proxy or upstream failure**: 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout. nginx (acting as the front end) cannot get a usable response from the upstream (FastAPI sidecar on port 9090, in the Bubbles architecture).

For SEO, the critical code is **503 Service Unavailable**: it is the only 5xx code that is "intentional" (the operator chose to serve it), and it pairs with `Retry-After` from framework-http-rate-control-headers.md to signal "I will be back" to crawlers. The other 5xx codes are unintentional failures that should be fixed, not configured.

For incident response, the critical distinction is **502 vs 504**: 502 means the upstream is not even accepting connections (process crashed, port not bound); 504 means the upstream accepted the connection but did not respond in time (slow query, deadlock, runaway loop).

---

## 2. WHY IT MATTERS

Six independent pressures push correct 5xx code handling from "default error" to "actively managed signal" in 2025 and forward.

**503 with Retry-After is the only way to do planned maintenance gracefully.** Per Google's official documentation: "When the crawl rate goes down, stop returning 503 or 429 HTTP response status codes for crawl requests; returning 503 or 429 for more than 2 days will cause Google to drop those URLs from the index." The 503 with Retry-After pattern is the contract: Googlebot sees "temporary, come back in N seconds", waits, retries, and the URL stays in the index. Without Retry-After, Googlebot guesses; after 48 hours of guessing wrong, the URL drops.

**500 errors signal real bugs that need fixing.** Unlike 4xx codes which are client errors (and often legitimate), 5xx codes are server problems. A site with persistent 500 errors has uncaught exceptions in the application. Monitoring 5xx rates is core SRE hygiene.

**502 vs 504 diagnostic distinction saves debugging time.** "Site returns 5xx randomly" is a useless symptom; "site returns 502 specifically when FastAPI process restarts" or "site returns 504 specifically on the /reports endpoint" points directly at the cause. Knowing the distinction shortens incident response.

**Crawlers eventually deindex persistent 5xx.** Sustained 500/502/504 leads to URL removal from the search index, just like sustained 503/429. For Bubbles client sites, 5xx during marketing campaign deployment is a real ranking risk.

**Self healing prevents most 5xx in the first place.** systemd `Restart=always`, nginx upstream backup, proper timeouts, and circuit breakers eliminate most transient 5xx errors before crawlers or users notice them.

**Cloudflare specific 5xx codes are not Bubbles concerns.** Bubbles runs self hosted with nginx directly facing the internet (no Cloudflare in front, per standing rule). The 520 through 527 codes only appear when Cloudflare is in the path, which is never for Bubbles client sites. But operators may encounter them when reviewing third party systems or migration sources.

**Cost of getting it wrong.** Misconfigured 5xx responses produce silent ranking damage, prolonged outages, and confused monitoring. Real examples:

* Bubbles client deployed a code change. FastAPI sidecar crashed on startup due to a missing environment variable. nginx returned 502 for every request. systemd was not configured with `Restart=always`. The site was down for 6 hours before Joseph noticed. Googlebot crawled, got 502 on 47 URLs in that window. The URLs were deindexed within 4 days.
* Long running database migration during business hours. Endpoint took 90 seconds to complete. nginx `proxy_read_timeout` was 60 seconds. nginx returned 504 to every client. The migration finished (the database accepted the work) but every request appeared to fail. Users saw error pages; the migration "succeeded" but the user experience said "broken".
* Maintenance window: server taken down for 4 hours of cleanup. No 503 returned; just connection refused. Crawlers retried, got nothing, scheduled the URL as failed. After the window, URLs took 3 weeks to recrawl back to normal frequency.
* Holiday weekend maintenance: site returned 503 with Retry-After: 86400 (24 hours). Maintenance overran to 50 hours. Beyond the 2 day rule. Googlebot dropped 200 URLs from the index. Recovery took 4 weeks.
* Site running on a tiny server hit 100% CPU during a flash crowd. nginx queued connections; the queue filled. New connections got 502. The server itself was fine; the proxy buffers were exhausted.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The four codes Joseph listed plus essential context for others get the standard treatment:

1. **What it means**: the canonical RFC definition plus practical implication.
2. **What causes it**: the typical scenarios that produce this code.
3. **How to handle it in nginx and FastAPI**: paste ready config and code.
4. **How crawlers react**: indexing pipeline behavior per major crawler.
5. **How to diagnose**: bash commands and log inspection patterns.
6. **How to prevent or recover**: ordered remediation steps.

Sections 11 to 14 are deep dives on the most operationally critical concerns: the 500 vs 502 vs 504 diagnostic distinction, the 503 maintenance pattern, the Bubbles FastAPI sidecar diagnostic workflow, and the self healing configuration.

Section 22 is the series summary tying together all 12 frameworks.

---

## 4. THE 5xx MENTAL MODEL (READ THIS FIRST)

Something failed on the server side. Where exactly? Internalize the decision tree.

```
A 5xx is being returned. Where does it originate?
   |
   |---> The upstream application (FastAPI sidecar) generated it
   |     |
   |     |---> Application threw an uncaught exception ............... 500 Internal Server Error
   |     |
   |     |---> Application is intentionally signaling unavailability . 503 Service Unavailable
   |     |
   |     |---> Application returned something nginx couldn't parse ... 502 Bad Gateway (rare from FastAPI; more from misconfigured backends)
   |
   |---> Nginx generated it because the upstream had a problem
   |     |
   |     |---> Connection refused (upstream not listening) ............ 502 Bad Gateway
   |     |
   |     |---> Upstream sent invalid HTTP response ..................... 502 Bad Gateway
   |     |
   |     |---> Upstream timed out (didn't respond in time) ............. 504 Gateway Timeout
   |     |
   |     |---> Upstream returned 503 (signaling unavailability) ........ 503 (passed through)
   |
   |---> The operator chose to signal maintenance
                                                                            |
            503 with Retry-After (the only "intentional" 5xx) ................. 503 Service Unavailable

==================== CRAWLER RECOVERY STRATEGY ====================
   |
   |---> Transient 5xx (rare, intermittent) ............... Crawlers retry; URL stays in index
   |
   |---> Sustained 5xx (hours to 2 days) ................... Crawlers reduce crawl rate
   |
   |---> 5xx beyond 2 days ................................ URLs drop from index
```

Six rules govern the system:

1. **500 means application bug; fix the bug.**
2. **502 means upstream is unreachable; fix the upstream (or proxy config).**
3. **503 is the only 5xx you choose. Pair with Retry-After. Keep under 48 hours.**
4. **504 means upstream too slow; increase timeout OR speed up upstream OR return 202 + status URL.**
5. **systemd Restart=always prevents most 502.**
6. **The 2 day rule is hard: sustained 5xx for >48 hours = index loss.**

A correctly configured server produces 500/502/504 rarely (only on genuine bugs or unavoidable upstream failures), uses 503 with Retry-After for planned maintenance, recovers automatically via systemd, and stays under the 48 hour threshold for any sustained 5xx state.

---

## 5. 500 INTERNAL SERVER ERROR (THE APPLICATION CRASHED)

### 5.1 What It Means

500 Internal Server Error is the generic "the server encountered an unexpected condition and could not fulfill the request". Defined in RFC 9110. Catch all for "something went wrong on the server side" when no more specific 5xx code applies.

```
HTTP/2 500 Internal Server Error
Content-Type: application/json

{"error": "internal_server_error"}
```

### 5.2 What Causes 500

* **Uncaught exception** in the FastAPI application (Python traceback, type error, key error).
* **Database connection failure** that the application doesn't handle gracefully.
* **Bug in business logic** (division by zero, null pointer, infinite loop hitting timeout).
* **Configuration error** that the app discovers at request time.
* **Missing template, missing static asset** referenced by code.

500 is "this should not have happened, the app crashed handling this specific request". It is almost always a bug, not an intentional state.

### 5.3 The Distinction From Other 5xx

* 500 vs 502: 500 means the app ran but failed. 502 means the app did not run at all (or returned garbage to nginx).
* 500 vs 503: 500 is unintentional crash. 503 is intentional unavailability signal.
* 500 vs 504: 500 is app failure. 504 is app slowness (took longer than nginx timeout).

### 5.4 How nginx Handles 500 From Upstream

When the FastAPI sidecar returns 500 to nginx, nginx passes it through by default:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    # 500 from upstream passes through unchanged
}
```

For custom 500 error pages:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_intercept_errors on;  # let nginx use error_page even for upstream errors
}

error_page 500 502 503 504 /5xx.html;

location = /5xx.html {
    root /var/www/sites/example.com;
    internal;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
}
```

### 5.5 How To Generate 500 In FastAPI (And Why You Usually Shouldn't)

Generally, 500 should happen by accident (uncaught exception) rather than be returned explicitly. The right pattern is:

* Catch the exception.
* Log it.
* Return an appropriate code: 422 for validation, 503 for "we're broken right now", 500 only if truly unexpected.

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
import logging

app = FastAPI()
logger = logging.getLogger(__name__)

@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    """Catch any unhandled exception and return 500 cleanly."""
    logger.exception(f"Unhandled exception on {request.url.path}: {exc}")
    return JSONResponse(
        status_code=500,
        content={"error": "internal_server_error"},
        headers={
            "Cache-Control": "no-store",
            "X-Robots-Tag": "noindex",
        }
    )

@app.get("/api/example")
async def example():
    # If this raises uncaught, the handler above returns 500
    return await some_potentially_failing_operation()
```

### 5.6 How To Diagnose 500

```bash
# 1. Find 500 events in nginx access log
sudo awk '$9 == 500 {print $4, $7, $11}' /var/log/nginx/access.log | tail -50

# 2. Cross reference with nginx error log
sudo tail -100 /var/log/nginx/error.log | grep -i "upstream\|error"

# 3. Check FastAPI application logs
sudo journalctl -u fastapi-sidecar --since "1 hour ago" | grep -iE "error|traceback|exception"

# 4. Test the specific endpoint that's 500ing
curl -v https://example.com/api/the-endpoint 2>&1 | tail -30

# 5. If consistent 500 on specific endpoint, check the code path
# - was there a recent deployment?
# - did dependencies change?
# - is the database accessible?
```

### 5.7 Crawler Reaction

Per Google's documentation, 5xx (including 500) is treated as server error. Googlebot retries on a schedule. Persistent 500 leads to URL removal from the index over weeks.

For Bubbles: every 500 should be considered a bug to fix, not a state to manage. Monitoring should alert on 500 rate above baseline.

### 5.8 The Prevention Pattern

```python
# Top level exception handler catches anything not handled by route specific logic
@app.exception_handler(Exception)
async def catch_all(request: Request, exc: Exception):
    logger.exception(f"Unhandled: {exc}")
    return JSONResponse(status_code=500, content={"error": "internal_server_error"})

# Route specific error handling for predictable failures
@app.get("/api/data")
async def get_data():
    try:
        result = await db.fetch_one("SELECT ...")
        if not result:
            raise HTTPException(status_code=404, detail="not found")
        return result
    except DatabaseError as e:
        logger.error(f"Database error: {e}")
        # Convert to a 503 with Retry-After if this is recoverable
        raise HTTPException(
            status_code=503,
            detail="database temporarily unavailable",
            headers={"Retry-After": "30"}
        )
    # Anything else falls through to the top level 500 handler
```

---

## 6. 502 BAD GATEWAY (THE UPSTREAM IS UNREACHABLE OR BROKEN)

### 6.1 What It Means

502 Bad Gateway signals that nginx, acting as a gateway or proxy, received an invalid response from the upstream server. Defined in RFC 9110.

```
HTTP/2 502 Bad Gateway
Content-Type: text/html

<html><body><center><h1>502 Bad Gateway</h1></center></body></html>
```

### 6.2 What Causes 502

In the Bubbles architecture (nginx + FastAPI sidecar on port 9090):

* **FastAPI process not running.** systemd hasn't started it, it crashed and didn't restart, port not bound.
* **FastAPI listening on wrong port** (binding to 9091 instead of 9090).
* **FastAPI process bound to localhost only** but nginx trying to reach a different interface.
* **Connection refused on port 9090** (firewall, port already in use by another process).
* **FastAPI returned malformed HTTP response** (rare; FastAPI is generally compliant).
* **Upstream connection reset** during response (the process died mid response).

Compare to 504: 502 means "I couldn't even talk to upstream usefully"; 504 means "I talked to upstream but it never finished talking back".

### 6.3 How nginx Returns 502

Automatic when proxy_pass cannot reach the upstream:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    # If 127.0.0.1:9090 connection refused OR FastAPI returns garbage,
    # nginx returns 502 to the client.
}
```

The nginx error log records why:

```
2026/05/25 02:45:12 [error] 1234#0: *567 connect() failed (111: Connection refused) while connecting to upstream, client: 1.2.3.4, server: example.com, request: "GET /api/data HTTP/2.0", upstream: "http://127.0.0.1:9090/api/data", host: "example.com"
```

The `connect() failed (111: Connection refused)` is the smoking gun: FastAPI is not listening.

### 6.4 The Diagnostic Sequence

When 502 appears:

```bash
# 1. Is FastAPI running?
systemctl status fastapi-sidecar
# Look for: Active: active (running) OR Active: failed

# 2. Is it listening on the expected port?
sudo ss -tlnp | grep :9090
# Expected: LISTEN ... 127.0.0.1:9090

# 3. Can nginx reach upstream from localhost?
curl -sI http://127.0.0.1:9090/api/health
# Expected: HTTP/1.1 200 OK (if not, FastAPI is the problem)

# 4. Check the most recent FastAPI logs for crash reasons
sudo journalctl -u fastapi-sidecar --since "10 min ago" | tail -50

# 5. Check nginx error log for the 502 details
sudo tail -50 /var/log/nginx/error.log | grep upstream
```

### 6.5 The Self Healing Pattern (systemd)

The single most important prevention for 502: configure systemd to restart FastAPI automatically when it crashes.

```ini
# /etc/systemd/system/fastapi-sidecar.service
[Unit]
Description=FastAPI sidecar for Bubbles
After=network.target

[Service]
Type=simple
User=bubbles
WorkingDirectory=/opt/bubbles
ExecStart=/opt/bubbles/.venv/bin/uvicorn app:app --host 127.0.0.1 --port 9090
Restart=always
RestartSec=5
StartLimitInterval=60
StartLimitBurst=10

# Resource limits (prevent runaway process)
MemoryMax=2G
CPUQuota=200%

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=fastapi-sidecar

[Install]
WantedBy=multi-user.target
```

Key directives:

* **`Restart=always`**: restart on any exit (crash, signal, normal exit).
* **`RestartSec=5`**: wait 5 seconds before restart (avoid rapid loop).
* **`StartLimitInterval=60` + `StartLimitBurst=10`**: if it crashes 10 times in 60 seconds, give up (prevents thrashing).

Apply:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now fastapi-sidecar
sudo systemctl status fastapi-sidecar
```

With this configured, transient FastAPI crashes produce a brief 502 (a few seconds) before automatic recovery. Without it, a single crash can produce hours of 502 if no one is watching.

### 6.6 The Nginx proxy_next_upstream Pattern

For sites with multiple FastAPI backends (failover), nginx can automatically try a backup:

```nginx
upstream fastapi_backend {
    server 127.0.0.1:9090 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:9091 backup;  # backup server, only used when primary fails
}

server {
    location /api/ {
        proxy_pass http://fastapi_backend;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;
    }
}
```

For a single instance Bubbles setup, this is overkill. For high availability deployments, it eliminates 502 from individual instance failures.

### 6.7 How To Verify Recovery

```bash
# After a 502 incident, verify recovery:

# 1. systemd shows running
systemctl is-active fastapi-sidecar
# Expected: active

# 2. Port is bound
sudo ss -tlnp | grep :9090

# 3. Health check returns 200
curl -sI http://127.0.0.1:9090/healthz | head -1
curl -sI https://example.com/healthz | head -1

# 4. No recent 502s in access log
sudo awk -v since="$(date -d '5 min ago' '+%d/%b/%Y:%H:%M')" '
    $4 ~ since && $9 == 502 {count++}
    END {print "502 count in last 5 min:", count+0}
' /var/log/nginx/access.log
```

### 6.8 Crawler Reaction

Same as 500: treated as server error, retried, removed from index over time if persistent.

For Bubbles: a brief 502 (seconds) is invisible to crawlers. Sustained 502 (hours) is a real ranking risk. systemd `Restart=always` is the single most important prevention.

---

## 7. 503 SERVICE UNAVAILABLE (THE MAINTENANCE CODE, RETRY-AFTER MANDATORY)

### 7.1 What It Means

503 Service Unavailable signals that the server is temporarily unable to handle the request. Defined in RFC 9110. **The only 5xx code that is "intentional"** (you choose to return it for maintenance, overload, or deliberate unavailability).

```
HTTP/2 503 Service Unavailable
Content-Type: text/html
Retry-After: 3600
Cache-Control: no-store
X-Robots-Tag: noindex

<html><body><h1>We are doing scheduled maintenance</h1>
<p>We will be back online in approximately 1 hour. Thank you for your patience.</p></body></html>
```

### 7.2 When To Use 503

* **Planned maintenance** (the primary use case).
* **Overload** (when the server cannot accept more work). nginx returns this automatically for `limit_req` and `limit_conn` by default, though the rate control framework recommends overriding to 429.
* **Dependency failure** where the application chooses to fail fast rather than serve broken data.
* **Initial deployment** before the application is ready (use during the cutover window).

### 7.3 The Mandatory Retry-After

For 503, `Retry-After` is functionally mandatory. Without it:

* Crawlers retry on their own schedule (variable).
* Browsers and intermediaries may cache the 503.
* Per Google: after 2 days of sustained 503 without proper signaling, URLs drop from the index.

With Retry-After:

* Crawlers wait exactly that long before retrying.
* Crawlers treat the response as transient.
* The URL stays in the index (for up to 2 days).

```
Retry-After: 3600              # 1 hour in seconds
Retry-After: Wed, 25 May 2026 16:00:00 GMT    # absolute time
```

### 7.4 The 2 Day Google Deindex Rule

The single most ranking critical detail in the entire 5xx framework, cross referenced from framework-http-rate-control-headers.md Section 5.3 and framework-http-4xx-status-codes.md Section 11.3:

> Per Google: "When the crawl rate goes down, stop returning 503 or 429 HTTP response status codes for crawl requests; returning 503 or 429 for more than 2 days will cause Google to drop those URLs from the index."

**Operational implication for Bubbles maintenance windows:**

* Maintenance under 24 hours: 503 with Retry-After is safe.
* Maintenance 24 to 48 hours: 503 with Retry-After is risky; Google may start treating it as permanent.
* Maintenance over 48 hours: **do not use 503**. Either serve a 200 maintenance page with substantive content, or use the crawler whitelist pattern (return 200 to crawlers, 503 to humans).

### 7.5 The Standard 503 Maintenance Pattern

**Step 1: prepare a maintenance HTML page.**

```html
<!-- /var/www/sites/example.com/maintenance.html -->
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Maintenance In Progress</title>
<meta name="robots" content="noindex, follow">
</head>
<body>
<h1>We will be back soon</h1>
<p>Scheduled maintenance is in progress. We expect to be back online by approximately Sunday at 8 PM Central.</p>
<p>If you need urgent assistance, please email admin@thatdeveloperguy.com.</p>
</body>
</html>
```

**Step 2: configure nginx to serve 503 with Retry-After.**

```nginx
server {
    listen 443 ssl;
    # ...

    location / {
        return 503;
    }

    error_page 503 /maintenance.html;
    location = /maintenance.html {
        root /var/www/sites/example.com;
        internal;
        add_header Retry-After "3600" always;          # 1 hour
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
    }
}
```

**Step 3: deploy.**

```bash
nginx -t && systemctl reload nginx
```

**Step 4: verify.**

```bash
# Verify 503 with Retry-After
curl -sI https://example.com/ | grep -iE "^(HTTP|Retry-After|Cache-Control)"
# Expected:
# HTTP/2 503
# retry-after: 3600
# cache-control: no-store
```

**Step 5: end maintenance, remove the return 503 block, reload.**

```bash
# Comment out or remove the `return 503;` line and the error_page 503 directive
nginx -t && systemctl reload nginx
```

### 7.6 The Crawler Friendly Maintenance Pattern (For Maintenance Over 24 Hours)

If maintenance must exceed 24 hours, serve a 200 maintenance page to crawlers and 503 to humans:

```nginx
map $http_user_agent $maintenance_response {
    default                  "503";
    ~*googlebot              "200";
    ~*bingbot                "200";
    ~*duckduckbot            "200";
    ~*applebot               "200";
    ~*claudebot              "200";
    ~*gptbot                 "200";
    ~*oai-searchbot          "200";
    ~*claude-searchbot       "200";
    ~*perplexitybot          "200";
}

server {
    location / {
        if ($maintenance_response = "503") {
            return 503;
        }
        # Crawlers fall through to serve the maintenance notice as 200
        root /var/www/sites/example.com;
        try_files /maintenance-200.html =503;
    }

    error_page 503 /maintenance.html;

    location = /maintenance.html {
        internal;
        add_header Retry-After "3600" always;
        add_header Cache-Control "no-store" always;
        root /var/www/sites/example.com;
    }

    location = /maintenance-200.html {
        internal;
        # For crawlers: 200 with the notice, no robots restriction
        # Substantive enough that it doesn't get soft 404 flagged
        root /var/www/sites/example.com;
    }
}
```

The `maintenance-200.html` page should be substantive (a real page about the maintenance, the company, the timeline) to avoid soft 404 flagging.

### 7.7 The 503 vs 504 Distinction

Both can result from upstream unavailability, but they signal different states:

* **503**: explicit "I'm intentionally unavailable" (set by the operator, comes from nginx config or the app).
* **504**: implicit "the upstream took too long" (set automatically by nginx when timeout exceeded).

For maintenance: use 503 explicitly. Do not let 504 happen accidentally during long running operations.

### 7.8 Crawler Reaction Per Operator

* **Googlebot**: honors Retry-After, retries after the specified duration, URL stays indexed up to 2 days.
* **Bingbot**: similar.
* **ClaudeBot, GPTBot**: honor Retry-After, retry after specified duration.
* **PerplexityBot**: honors Retry-After.

All major crawlers handle 503 with Retry-After correctly. The 2 day rule (or its equivalent) applies across all of them.

### 7.9 How To Diagnose Unintentional 503

If 503 appears unexpectedly (not from maintenance):

```bash
# 1. Check if limit_req is firing
sudo grep "limiting requests" /var/log/nginx/error.log | tail -20
# limit_req returns 503 by default; should be 429 per framework-http-rate-control-headers.md

# 2. Check if FastAPI is returning 503
sudo journalctl -u fastapi-sidecar --since "30 min ago" | grep -i "503"

# 3. Check overall 503 rate
sudo awk -v since="$(date -d '1 hour ago' '+%d/%b/%Y:%H')" '
    $4 ~ since && $9 == 503 {count++}
    END {print "503 in last hour:", count+0}
' /var/log/nginx/access.log
```

---

## 8. 504 GATEWAY TIMEOUT (THE UPSTREAM IS TOO SLOW)

### 8.1 What It Means

504 Gateway Timeout signals that nginx, acting as a proxy, did not receive a timely response from the upstream server. Defined in RFC 9110.

```
HTTP/2 504 Gateway Timeout
Content-Type: text/html

<html><body><center><h1>504 Gateway Time-out</h1></center></body></html>
```

### 8.2 What Causes 504

* **Slow database query**: FastAPI is waiting on Postgres; query takes longer than `proxy_read_timeout`.
* **External API call**: FastAPI calls a third party service; the call exceeds the timeout.
* **Runaway operation**: a request triggered an infinite loop or very expensive computation.
* **Lock contention**: thread or process waiting for a lock that another request holds.
* **Insufficient resources**: server CPU at 100%, all worker processes busy.

Compare to 502: 502 means nginx couldn't even talk to upstream. 504 means nginx talked to upstream but the conversation never finished.

### 8.3 The Nginx Timeout Configuration

nginx defaults are aggressive for production:

```nginx
# nginx defaults (rarely shown in config files)
proxy_connect_timeout 60s;    # how long to wait to connect to upstream
proxy_send_timeout 60s;       # how long to wait to send request to upstream
proxy_read_timeout 60s;       # how long to wait for response from upstream
```

For typical web requests, 60 seconds is generous; most should complete in under 5 seconds.

For long running endpoints, the choice is:

**Option A: increase the timeout.**

```nginx
location /api/long-running-report {
    proxy_pass http://127.0.0.1:9090;
    proxy_read_timeout 300s;   # 5 minutes
    proxy_send_timeout 60s;
    proxy_connect_timeout 10s;
}
```

This works but blocks the client for the full duration. Bad UX for any operation over 30 seconds.

**Option B: return 202 Accepted with async processing.**

The correct pattern per framework-http-2xx-status-codes.md Section 7: long running operations should be async. Return 202 immediately with a status URL; the client polls. No timeout concern.

**Option C: streaming response.**

For operations that can stream output incrementally:

```python
from fastapi.responses import StreamingResponse

@app.get("/api/large-export")
async def large_export():
    async def generate():
        for chunk in iter_chunks():
            yield chunk

    return StreamingResponse(generate(), media_type="application/json")
```

nginx keeps the connection open as long as data is flowing.

### 8.4 How To Diagnose 504

```bash
# 1. Find 504 events
sudo awk '$9 == 504 {print $4, $7, $11}' /var/log/nginx/access.log | tail -20

# 2. Check nginx error log for upstream timeout
sudo grep -i "upstream timed out" /var/log/nginx/error.log | tail -20

# Look for: "upstream timed out (110: Connection timed out) while reading response header from upstream"

# 3. Identify the slow endpoint
sudo awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -5

# 4. Profile the slow endpoint
# Check FastAPI logs for what it was doing
sudo journalctl -u fastapi-sidecar | grep "/the-slow-endpoint" | tail -10

# 5. Check upstream resource usage
top -p $(pgrep -f uvicorn)
```

### 8.5 The Bubbles Standard Timeouts

For typical Bubbles client sites:

```nginx
# In server block, for /api/ location
location /api/ {
    proxy_pass http://127.0.0.1:9090;

    # Standard timeouts (more aggressive than nginx defaults)
    proxy_connect_timeout 5s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}

# For specific known long endpoints
location /api/reports/generate {
    proxy_pass http://127.0.0.1:9090;
    proxy_read_timeout 300s;  # 5 minutes for report generation
}

# For streaming
location /api/stream {
    proxy_pass http://127.0.0.1:9090;
    proxy_read_timeout 86400s;  # 24 hours; keep alive as long as needed
    proxy_buffering off;
}
```

### 8.6 Crawler Reaction

Same as 500/502: treated as server error, crawler retries, sustained 504 leads to URL removal.

For Bubbles: a 504 means a specific endpoint is too slow. Fix the slow endpoint; do not just raise the timeout indefinitely.

### 8.7 The 503 vs 504 Operator Choice

When you know an endpoint is going to be slow (long running report, batch operation), the choice is:

* **Do not use 504**: it implies failure. The client gets a generic timeout.
* **Use 202 Accepted**: signal async processing, return status URL. The client knows to poll.
* **Use 503 with Retry-After**: signal "come back later". Less appropriate for normal long running work; better for "temporarily overloaded".

The 504 is what happens when you don't actively choose. Avoid it by designing endpoints to complete within reasonable bounds.

---

## 9. OTHER 5xx CODES (BRIEFLY)

Joseph did not list these, but they appear occasionally.

### 9.1 505 HTTP Version Not Supported

The server does not support the HTTP version of the request. Extremely rare in 2026; modern clients and servers all support HTTP/1.1, HTTP/2, and HTTP/3. Possible cause: very old HTTP/1.0 client hitting a strict HTTP/2 only server.

For Bubbles: never seen in practice. If encountered, check nginx `listen` directives.

### 9.2 507 Insufficient Storage

The server is out of disk space to complete the request. Rare for Bubbles (disk monitoring should catch this before 507). Common in WebDAV scenarios.

Detection:

```bash
df -h /var/www
# If usage > 95%, problem.
```

### 9.3 508 Loop Detected

Used by WebDAV when an infinite loop is detected in resource processing. Not relevant to typical Bubbles workloads.

### 9.4 511 Network Authentication Required

Indicates the client must authenticate to gain network access. Used by captive portals (hotel WiFi, etc). Not relevant to Bubbles client sites.

### 9.5 599 Network Connect Timeout Error

Non standard but sometimes returned by load balancers. Not part of any RFC. Treat as 504 if encountered.

---

## 10. CLOUDFLARE SPECIFIC 5xx CODES (520 THROUGH 527, BRIEFLY)

Bubbles does not use Cloudflare (per the standing rule "Never recommend Cloudflare or third party CDN in front of Bubbles"). But operators may encounter these codes when reviewing third party systems or migration sources. Brief reference:

| Code | Cloudflare Meaning |
|---|---|
| 520 Unknown Error | The origin returned an empty, unknown, or unexpected response to Cloudflare |
| 521 Web Server Is Down | The origin refused the connection from Cloudflare |
| 522 Connection Timed Out | TCP handshake with origin did not complete |
| 523 Origin Is Unreachable | Cloudflare cannot find the origin's IP (DNS or routing problem) |
| 524 A Timeout Occurred | TCP connection completed but origin did not respond in 100 seconds |
| 525 SSL Handshake Failed | TLS handshake between Cloudflare and origin failed |
| 526 Invalid SSL Certificate | Origin's SSL cert could not be validated |
| 527 Railgun Error | Cloudflare's Railgun service failed (deprecated) |

These codes are generated by Cloudflare's edge, not by the origin server. None of them apply to Bubbles directly.

If a Bubbles client previously used Cloudflare and migrated to Bubbles, the 5xx pattern shifts:

* Cloudflare 521 (origin refused connection) becomes nginx 502 from Bubbles.
* Cloudflare 524 (origin timeout) becomes nginx 504 from Bubbles.
* Cloudflare 525/526 (SSL issues) become normal SSL handshake failures (no 5xx, just connection errors).

For Joseph's migrations off Cloudflare: the new 5xx pattern is simpler (just 500/502/503/504), and the diagnostics are local (look at Bubbles logs, not Cloudflare dashboard).

---

## 11. THE 500 VS 502 VS 504 DISTINCTION (DEEP DIVE)

The three "unintentional" 5xx codes signal different failure modes. Knowing the distinction shortens incident response.

### 11.1 The Decision Table

| Symptom | Status | What it means | Where to look |
|---|---|---|---|
| Application threw exception, returned 500 | 500 | App ran and failed | FastAPI logs |
| Nginx cannot connect to upstream | 502 | App not listening on expected port | `systemctl status fastapi-sidecar`, `ss -tlnp` |
| Upstream sent malformed HTTP | 502 | App returned garbage (rare) | Compare expected vs actual response from upstream |
| Upstream connection succeeded but response timed out | 504 | App accepted request but never responded | FastAPI logs, slow query log |
| Upstream connection succeeded but response was incomplete | 502 | App died mid response | systemd journal for crash |

### 11.2 The Bash One Liners

```bash
# Find the most recent 5xx errors and classify
sudo awk '$9 ~ /^5/ {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 502 vs 504 breakdown in last hour
sudo awk -v since="$(date -d '1 hour ago' '+%d/%b/%Y:%H')" '
    $4 ~ since && $9 == 502 {n502++}
    $4 ~ since && $9 == 504 {n504++}
    $4 ~ since && $9 == 500 {n500++}
    END {
        print "500 errors:", n500+0
        print "502 errors:", n502+0
        print "504 errors:", n504+0
    }
' /var/log/nginx/access.log
```

### 11.3 The Diagnostic Sequence

For any 5xx incident:

```bash
# Step 1: classify
sudo awk '$9 ~ /^5/' /var/log/nginx/access.log | tail -5

# Step 2: by status
# If 500: check FastAPI logs
sudo journalctl -u fastapi-sidecar -p err --since "10 min ago"

# If 502: check FastAPI is running and listening
systemctl is-active fastapi-sidecar
sudo ss -tlnp | grep :9090

# If 504: check what's slow
sudo grep "upstream timed out" /var/log/nginx/error.log | tail -10

# Step 3: cross reference with deployment events
sudo journalctl -u fastapi-sidecar --since "1 hour ago" | grep -i "stopped\|started\|restarted"
```

### 11.4 The Recovery Action Per Code

* **500**: Fix the bug. Check FastAPI logs for traceback, identify the cause, deploy a fix.
* **502**: Restart FastAPI. `systemctl restart fastapi-sidecar`. Then investigate why it died.
* **504**: Identify slow endpoint, fix the slowness (query optimization, indexing, async processing).

If 500 or 502 is intermittent, systemd `Restart=always` should be in place. If it isn't, that is the immediate fix.

---

## 12. THE 503 MAINTENANCE PATTERN (DEEP DIVE, 2 DAY GOOGLE DEINDEX RULE)

The single SEO critical 5xx pattern. This section consolidates everything from framework-http-rate-control-headers.md Section 5.3 and framework-http-4xx-status-codes.md Section 11.3 into the canonical 503 handling reference.

### 12.1 The Rule

Sustained 503 (or 429) for more than 2 days causes Google to drop the URLs from the index.

Practical implications:

* Maintenance windows must be planned for under 48 hours.
* Retry-After must be set on every 503.
* The pattern is: short downtime to 503 with Retry-After; long downtime to 200 with maintenance notice (or crawler split pattern from Section 7.6).

### 12.2 The Standard Pattern (Under 24 Hours)

```nginx
# /etc/nginx/sites-available/example.com.maintenance

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        return 503;
    }

    error_page 503 /maintenance.html;
    location = /maintenance.html {
        root /var/www/sites/example.com;
        internal;
        add_header Retry-After "3600" always;
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
    }
}
```

### 12.3 The Deployment Workflow

```bash
# Before maintenance:
# 1. Save current config
sudo cp /etc/nginx/sites-available/example.com /etc/nginx/sites-available/example.com.before-maintenance

# 2. Deploy maintenance config
sudo cp /etc/nginx/sites-available/example.com.maintenance /etc/nginx/sites-available/example.com
sudo nginx -t && sudo systemctl reload nginx

# 3. Verify maintenance is showing
curl -sI https://example.com/ | grep -iE "^(HTTP|Retry-After)"

# During maintenance: do the work

# After maintenance:
# 1. Restore the original config
sudo cp /etc/nginx/sites-available/example.com.before-maintenance /etc/nginx/sites-available/example.com
sudo nginx -t && sudo systemctl reload nginx

# 2. Verify back online
curl -sI https://example.com/ | head -1
# Expected: HTTP/2 200
```

### 12.4 The Crawler Split Pattern (For 24 to 48 Hours)

For maintenance windows approaching the 2 day limit, use the crawler split pattern from Section 7.6 to serve crawlers a 200 response with substantive maintenance notice. Maintains index presence even during extended downtime.

### 12.5 The Hard No Pattern (Over 48 Hours)

Maintenance over 48 hours: do not use 503 at all. Options:

* **Serve a 200 placeholder.** Substantive content explaining the situation. Keep URLs returning 200.
* **Use 301 to a temporary mirror site.** SEO equity stays put, traffic routes elsewhere temporarily.
* **Phased shutdown.** Pull URLs out of sitemap, let them naturally drop in priority before downtime.

### 12.6 The Monitoring During Maintenance

```bash
# Watch GSC for sudden index changes
# (Manual monitoring; check daily during maintenance)

# Monitor 503 rate from access log
watch -n 60 'sudo awk "{ if (\$9 == 503) c++ } END { print \"503 count:\", c+0 }" /var/log/nginx/access.log'

# Verify Retry-After is consistently set
curl -sI https://example.com/ | grep -i retry-after
# Should match the planned duration
```

### 12.7 The Post Maintenance Recovery

After maintenance ends:

1. **Confirm 200 returns immediately.** `curl -sI https://example.com/` should show 200.
2. **Clear browser caches.** 503 responses with no Cache-Control may be cached briefly. The `Cache-Control: no-store` prevents this, but verify.
3. **Submit sitemap for recrawl.** GSC > Sitemaps > resubmit signals fresh crawl.
4. **Monitor crawl rate.** GSC Crawl Stats should return to baseline within hours to days.
5. **Verify index unchanged.** GSC Page Indexing > "Valid" count should be the same as before maintenance.

---

## 13. THE BUBBLES FASTAPI SIDECAR DIAGNOSTIC PATTERN

A consolidated diagnostic workflow for the most common Bubbles 5xx scenarios: FastAPI sidecar issues.

### 13.1 The Bubbles Architecture Recap

```
Internet
   |
   v
nginx (port 443, 80)
   |
   v (proxy_pass http://127.0.0.1:9090)
FastAPI (uvicorn, port 9090)
   |
   v
PostgreSQL, Redis, etc.
```

When 5xx appears, the failure is at one of these layers.

### 13.2 The Layered Diagnostic

**Layer 1: nginx is up.**

```bash
systemctl is-active nginx
# Expected: active

# If not, start it:
sudo systemctl start nginx
```

**Layer 2: nginx accepts connections.**

```bash
curl -sI https://example.com/ -o /dev/null
echo $?
# Expected: 0 (success)
```

**Layer 3: nginx can reach upstream.**

```bash
curl -sI http://127.0.0.1:9090/ -o /dev/null
echo $?
# Expected: 0 (success)
# If fails: upstream is down
```

**Layer 4: FastAPI sidecar is running.**

```bash
systemctl is-active fastapi-sidecar
# Expected: active
```

**Layer 5: FastAPI is listening on 9090.**

```bash
sudo ss -tlnp | grep :9090
# Expected: LISTEN 127.0.0.1:9090
```

**Layer 6: FastAPI is processing requests.**

```bash
curl -sI http://127.0.0.1:9090/healthz | head -1
# Expected: HTTP/1.1 200 OK
```

**Layer 7: FastAPI logs show no recent errors.**

```bash
sudo journalctl -u fastapi-sidecar --since "5 min ago" -p err
# Expected: no output (no errors)
```

**Layer 8: Downstream dependencies (database, Redis) are accessible.**

```bash
# Test database connection
sudo -u bubbles psql -d bubbles -c "SELECT 1;" 2>&1
# Expected: result row "1"

# Test Redis if used
redis-cli ping
# Expected: PONG
```

### 13.3 The Single Command Diagnostic Script

```bash
#!/bin/bash
# /usr/local/bin/bubbles-health-check.sh
# Quick health check for the Bubbles FastAPI sidecar stack

echo "=== Bubbles Health Check ==="
echo ""

echo "[nginx]"
echo "  Status: $(systemctl is-active nginx)"
nginx -t 2>&1 | grep -v "ok" | head -2

echo "[FastAPI sidecar]"
echo "  Status: $(systemctl is-active fastapi-sidecar)"
echo "  Port 9090: $(sudo ss -tlnp | grep -c :9090) listener(s)"

echo "[Upstream reachable from nginx]"
LOCAL=$(curl -so /dev/null -w "%{http_code}" http://127.0.0.1:9090/healthz)
echo "  Local healthz: $LOCAL"

echo "[Public reachable]"
PUBLIC=$(curl -so /dev/null -w "%{http_code}" https://example.com/healthz)
echo "  Public healthz: $PUBLIC"

echo "[Recent 5xx in last 10 min]"
sudo awk -v since="$(date -d '10 min ago' '+%d/%b/%Y:%H:%M')" '
    $4 >= ("["since) && $9 ~ /^5/ {count++}
    END {print "  Count:", count+0}
' /var/log/nginx/access.log

echo "[Recent FastAPI errors in last 10 min]"
ERR_COUNT=$(sudo journalctl -u fastapi-sidecar --since "10 min ago" -p err -q 2>&1 | wc -l)
echo "  Error log lines: $ERR_COUNT"

echo ""
echo "=== End Health Check ==="
```

Make executable and run:

```bash
sudo chmod +x /usr/local/bin/bubbles-health-check.sh
sudo bubbles-health-check.sh
```

### 13.4 The Restart Sequence

When something is wrong:

```bash
# 1. Restart FastAPI sidecar first (most common fix for 502)
sudo systemctl restart fastapi-sidecar
sleep 3
systemctl is-active fastapi-sidecar
curl -sI http://127.0.0.1:9090/healthz | head -1

# 2. If still broken, reload nginx
sudo nginx -t && sudo systemctl reload nginx

# 3. If still broken, restart nginx fully
sudo systemctl restart nginx

# 4. Verify
curl -sI https://example.com/ | head -1
```

### 13.5 The Common Causes Index

For Bubbles 5xx specifically:

| Symptom | Most likely cause | Fix |
|---|---|---|
| 502 immediately after deploy | FastAPI failed to start (config error, missing dep) | Check `journalctl -u fastapi-sidecar` for traceback |
| 502 intermittent | FastAPI crashes; systemd restarts | Find crash cause; verify Restart=always |
| 502 during high traffic | uvicorn workers saturated | Increase `--workers` count, vertical scale, or add queue |
| 500 after dependency update | New version of library has breaking change | Check pip freeze diff, roll back |
| 504 on specific endpoint | Slow query, third party API call | Profile the endpoint, optimize or move async |
| 503 unexpectedly | Rate limit firing (nginx default for limit_req) | Set `limit_req_status 429` (per rate control framework) |
| All 5xx simultaneously | nginx config error or system level failure | `nginx -t`, check disk space, check load |

---

## 14. SELF HEALING WITH SYSTEMD AND NGINX UPSTREAM BACKUP

The single most important 5xx prevention: automatic recovery from transient failures.

### 14.1 The Systemd Self Healing Configuration

The production grade `/etc/systemd/system/fastapi-sidecar.service`:

```ini
[Unit]
Description=FastAPI sidecar for Bubbles
Documentation=https://bubbles.thatdeveloperguy.com/docs/fastapi-sidecar
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=bubbles
Group=bubbles
WorkingDirectory=/opt/bubbles

# Environment
Environment="PYTHONUNBUFFERED=1"
EnvironmentFile=/etc/bubbles/sidecar.env

# Start command
ExecStart=/opt/bubbles/.venv/bin/uvicorn app:app \
    --host 127.0.0.1 \
    --port 9090 \
    --workers 4 \
    --log-level info \
    --access-log

# Restart behavior
Restart=always
RestartSec=5
StartLimitInterval=300
StartLimitBurst=10

# Resource limits
MemoryMax=2G
CPUQuota=200%
TasksMax=512

# Filesystem security
ProtectSystem=strict
ReadWritePaths=/var/log/bubbles /var/lib/bubbles
NoNewPrivileges=true

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=fastapi-sidecar

[Install]
WantedBy=multi-user.target
```

### 14.2 The Nginx Self Healing Configuration

```nginx
upstream fastapi_backend {
    server 127.0.0.1:9090 max_fails=3 fail_timeout=30s;
    # Optional: backup instance on a different port
    # server 127.0.0.1:9091 backup;

    keepalive 32;  # connection pool
}

server {
    # ...

    location /api/ {
        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # required for keepalive

        # Retry on failure (try upstream again, or next upstream if backup configured)
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;

        # Timeouts (standard Bubbles)
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        # Other proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Key behaviors:

* `max_fails=3 fail_timeout=30s`: if 3 failures within 30 seconds, nginx marks the upstream "down" for 30 seconds.
* `proxy_next_upstream`: retry on these conditions.
* `keepalive 32`: maintain 32 idle connections to upstream for faster responses.

### 14.3 The Watchdog Pattern

For paranoid setups, an external watchdog that restarts the service if health checks fail:

```bash
#!/bin/bash
# /usr/local/bin/bubbles-watchdog.sh
# Run via cron every minute

HEALTH_URL=http://127.0.0.1:9090/healthz
TIMEOUT=5

STATUS=$(curl -so /dev/null -w "%{http_code}" --max-time $TIMEOUT $HEALTH_URL)

if [ "$STATUS" != "200" ]; then
    echo "[$(date)] FastAPI health check failed (status: $STATUS), restarting" >> /var/log/bubbles-watchdog.log
    systemctl restart fastapi-sidecar
fi
```

Cron entry:

```cron
* * * * * /usr/local/bin/bubbles-watchdog.sh
```

For most cases, systemd's `Restart=always` is sufficient. The watchdog adds protection against "running but unresponsive" scenarios (hangs, deadlocks).

### 14.4 The Graceful Shutdown Configuration

Prevent 5xx during deployment by handling SIGTERM gracefully in FastAPI:

```python
import signal
from contextlib import asynccontextmanager
from fastapi import FastAPI

shutdown_event = asyncio.Event()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Setup
    yield
    # Cleanup on shutdown
    # Finish in flight requests, close database connections
    await db_pool.close()

app = FastAPI(lifespan=lifespan)
```

systemd will send SIGTERM, wait `TimeoutStopSec` (default 90 seconds), then SIGKILL. During the grace period, in flight requests complete, then the service shuts down cleanly. Next request gets routed to whatever started up next (or experiences brief 502 before systemd restarts the service).

---

## 15. HOW MAJOR CRAWLERS REACT TO EACH 5XX CODE

Quick reference for auditing.

| Code | Googlebot | Bingbot | ClaudeBot | GPTBot | PerplexityBot |
|---|---|---|---|---|---|
| 500 | Retries; sustained leads to index removal | Similar | Skip; cached representation retained | Skip | Skip |
| 502 | Retries; sustained leads to index removal | Similar | Skip | Skip | Skip |
| 503 with Retry-After | Honors Retry-After; URL stays indexed | Honors | Honors | Honors | Honors |
| 503 without Retry-After | Treats like transient error | Similar | Skip after retries | Skip | Skip |
| 504 | Retries; sustained leads to index removal | Similar | Skip | Skip | Skip |

**The 2 day rule (Google):** sustained 503 or 429 over 48 hours leads to URL removal from index. Applies regardless of Retry-After value.

**The implicit rule (other crawlers):** all major crawlers eventually treat persistent 5xx as URL removal. Timing varies but the outcome is similar.

For Bubbles operators: keep 5xx incidents under 24 hours where possible; never exceed 48 hours.

---

## 16. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 16.1 Standard maintenance with 503

```nginx
# Replace the default location / with this during maintenance
location / {
    return 503;
}

error_page 503 /maintenance.html;
location = /maintenance.html {
    root /var/www/sites/example.com;
    internal;
    add_header Retry-After "3600" always;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
}
```

### 16.2 Crawler split maintenance (200 to crawlers, 503 to humans)

```nginx
map $http_user_agent $maintenance_response {
    default                  "503";
    ~*googlebot              "200";
    ~*bingbot                "200";
    ~*claudebot              "200";
    ~*gptbot                 "200";
    ~*oai-searchbot          "200";
    ~*perplexitybot          "200";
}

server {
    location / {
        if ($maintenance_response = "503") {
            return 503;
        }
        root /var/www/sites/example.com;
        try_files /maintenance-200.html =503;
    }

    error_page 503 /maintenance.html;

    location = /maintenance.html {
        internal;
        add_header Retry-After "3600" always;
        add_header Cache-Control "no-store" always;
        root /var/www/sites/example.com;
    }
}
```

### 16.3 Standard FastAPI proxy with timeouts

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_connect_timeout 5s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

### 16.4 Long running endpoint with extended timeout

```nginx
location /api/reports/generate {
    proxy_pass http://127.0.0.1:9090;
    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 300s;  # 5 minutes for slow reports
}
```

### 16.5 Streaming endpoint (no timeout)

```nginx
location /api/stream {
    proxy_pass http://127.0.0.1:9090;
    proxy_buffering off;
    proxy_read_timeout 86400s;  # 24 hours
}
```

### 16.6 Custom 5xx error page

```nginx
error_page 500 502 503 504 /5xx.html;

location = /5xx.html {
    root /var/www/sites/example.com;
    internal;
    add_header Cache-Control "no-store" always;
    add_header X-Robots-Tag "noindex" always;
}
```

```html
<!-- /var/www/sites/example.com/5xx.html -->
<!doctype html>
<html lang="en">
<head>
<title>Server Error</title>
<meta name="robots" content="noindex">
</head>
<body>
<h1>Something went wrong on our end</h1>
<p>We're investigating. Please try again in a few minutes, or contact admin@thatdeveloperguy.com if the problem persists.</p>
</body>
</html>
```

### 16.7 FastAPI global exception handler

```python
import logging
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()
logger = logging.getLogger(__name__)

@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    logger.exception(f"Unhandled exception on {request.method} {request.url.path}")
    return JSONResponse(
        status_code=500,
        content={
            "error": "internal_server_error",
            "request_id": request.headers.get("x-request-id", "unknown"),
        },
        headers={
            "Cache-Control": "no-store",
            "X-Robots-Tag": "noindex",
        }
    )
```

### 16.8 FastAPI 503 with Retry-After for dependency failure

```python
from fastapi import FastAPI, HTTPException

@app.get("/api/data")
async def get_data():
    try:
        return await db.fetch_one("SELECT ...")
    except DatabaseConnectionError as e:
        logger.error(f"Database connection failed: {e}")
        raise HTTPException(
            status_code=503,
            detail="database temporarily unavailable",
            headers={"Retry-After": "30"}
        )
```

### 16.9 nginx upstream with retry on 5xx

```nginx
upstream fastapi_backend {
    server 127.0.0.1:9090 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

location /api/ {
    proxy_pass http://fastapi_backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";

    proxy_next_upstream error timeout http_502 http_504;
    proxy_next_upstream_tries 2;
}
```

### 16.10 Health check endpoint

```python
@app.get("/healthz")
async def healthz():
    # Simple liveness check
    return {"status": "ok"}

@app.get("/readyz")
async def readyz():
    # Readiness check (verify dependencies)
    try:
        await db.fetch_one("SELECT 1")
    except Exception:
        raise HTTPException(status_code=503, detail="database not ready")
    return {"status": "ready"}
```

```nginx
location = /healthz {
    access_log off;
    proxy_pass http://127.0.0.1:9090;
}
```

### 16.11 The Bubbles standard timeout block (template)

```nginx
# Default timeouts for /api/ endpoints
proxy_connect_timeout 5s;
proxy_send_timeout 30s;
proxy_read_timeout 30s;

# For long running endpoints, override per location:
# location /api/long-endpoint {
#     proxy_read_timeout 300s;
# }

# For streaming endpoints:
# location /api/stream {
#     proxy_buffering off;
#     proxy_read_timeout 86400s;
# }
```

### 16.12 Systemd service for FastAPI (production grade)

See Section 14.1 for the complete service file.

---

## 17. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete 5xx aware configuration for a Bubbles client site.

```nginx
# /etc/nginx/sites-available/example.com

upstream fastapi_backend {
    server 127.0.0.1:9090 max_fails=3 fail_timeout=30s;
    keepalive 32;
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

    root /var/www/sites/example.com;
    index index.html;

    # ============= STATIC =============
    location ~* \.(css|js|woff2|jpg|jpeg|png|webp|avif|svg)$ {
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        add_header Accept-Ranges "bytes" always;
    }

    # ============= HEALTH CHECKS =============
    location = /healthz {
        access_log off;
        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_read_timeout 2s;
    }

    # ============= API =============
    location /api/ {
        # OPTIONS preflight (per CORS framework)
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            return 204;
        }

        add_header X-Robots-Tag "noindex, nofollow" always;

        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Standard timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        # Auto retry on transient upstream failure
        proxy_next_upstream error timeout http_502 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;
    }

    # ============= LONG RUNNING ENDPOINTS =============
    location /api/reports/generate {
        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_read_timeout 300s;  # 5 minutes for report generation
    }

    # ============= ROOT =============
    location / {
        try_files $uri $uri/ $uri.html =404;
    }

    # ============= 5xx ERROR PAGES =============
    error_page 500 502 504 /5xx.html;
    error_page 503 /maintenance.html;

    location = /5xx.html {
        internal;
        root /var/www/sites/example.com;
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
    }

    location = /maintenance.html {
        internal;
        root /var/www/sites/example.com;
        add_header Retry-After "3600" always;
        add_header Cache-Control "no-store" always;
        add_header X-Robots-Tag "noindex" always;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify:

```bash
# Health check (should be 200)
curl -sI https://example.com/healthz | head -1
# Expected: HTTP/2 200

# Test that timeouts are configured
nginx -T 2>/dev/null | grep -E "proxy_(connect|send|read)_timeout"
# Expected: proxy_connect_timeout 5s; proxy_send_timeout 30s; proxy_read_timeout 30s;

# Run the diagnostic script
sudo bubbles-health-check.sh
```

---

## 18. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### Systemd self healing

1. [ ] FastAPI sidecar systemd unit file exists.
2. [ ] `Restart=always` is set.
3. [ ] `RestartSec=5` (or similar) prevents thrashing.
4. [ ] `StartLimitInterval` and `StartLimitBurst` configured.
5. [ ] `MemoryMax` and `CPUQuota` limits set.
6. [ ] Service enabled (`systemctl enable fastapi-sidecar`).
7. [ ] Service starts on boot (verified with reboot test).

### Nginx upstream configuration

8. [ ] `upstream` block defined with `max_fails` and `fail_timeout`.
9. [ ] `keepalive 32` (or similar) for connection pooling.
10. [ ] `proxy_next_upstream` configured for retry.
11. [ ] `proxy_connect_timeout 5s` (or aggressive default).
12. [ ] `proxy_send_timeout 30s` for typical endpoints.
13. [ ] `proxy_read_timeout 30s` for typical endpoints.
14. [ ] Long running endpoints have higher `proxy_read_timeout`.

### 500 Internal Server Error

15. [ ] FastAPI has top level exception handler returning 500 cleanly.
16. [ ] Exception handler logs the traceback.
17. [ ] 500 responses have `Cache-Control: no-store`.
18. [ ] 500 responses have `X-Robots-Tag: noindex`.
19. [ ] 500 rate monitored and alerts on baseline deviation.

### 502 Bad Gateway

20. [ ] Bubbles diagnostic script `/usr/local/bin/bubbles-health-check.sh` deployed.
21. [ ] systemd ensures FastAPI runs even after crashes.
22. [ ] nginx upstream definition includes `max_fails` and `fail_timeout`.
23. [ ] 502 rate alerts on sustained presence.
24. [ ] 502 error page provides useful information.

### 503 Service Unavailable

25. [ ] **Retry-After header on every 503 response.**
26. [ ] 503 has `Cache-Control: no-store`.
27. [ ] 503 has `X-Robots-Tag: noindex`.
28. [ ] Maintenance pattern documented and tested.
29. [ ] Crawler split pattern (Section 7.6) ready for use if maintenance exceeds 24 hours.
30. [ ] **Maintenance windows planned for under 48 hours (Google 2 day rule).**
31. [ ] limit_req returns 429 not 503 (per framework-http-rate-control-headers.md).

### 504 Gateway Timeout

32. [ ] Standard `proxy_read_timeout` matches typical endpoint duration.
33. [ ] Long running endpoints have explicit higher timeout OR use 202 async pattern.
34. [ ] Streaming endpoints have `proxy_buffering off` and very high timeout.
35. [ ] 504 rate monitored; sustained 504 triggers investigation.

### Monitoring and alerting

36. [ ] 5xx rate alert configured (e.g., >1% of requests over 5 minutes).
37. [ ] Disk space alert at 80% (prevents 507).
38. [ ] FastAPI process memory alert (prevents OOM kill leading to 502).
39. [ ] Database connection pool exhaustion alert.
40. [ ] CPU saturation alert (prevents 504 from slow processing).

### Operational hygiene

41. [ ] Maintenance config exists at `/etc/nginx/sites-available/<site>.maintenance`.
42. [ ] Maintenance HTML page substantive (not soft 404 risk).
43. [ ] Health check endpoint (`/healthz`) is fast and dependency free.
44. [ ] Readiness endpoint (`/readyz`) checks dependencies and returns 503 if not ready.
45. [ ] Custom 5xx page provides "contact admin@thatdeveloperguy.com" info.

### Cross cutting

46. [ ] `nginx -t` passes without warnings.
47. [ ] `nginx -T | grep error_page` shows expected error page definitions.
48. [ ] `bubbles-health-check.sh` shows all green during normal operation.
49. [ ] Recovery procedures documented (`/var/bubbles/runbooks/5xx.md`).
50. [ ] Quarterly drill: simulate FastAPI crash, verify automatic recovery.

A site that passes all 50 has correctly configured 5xx handling for SEO preservation, operational resilience, and crawler friendly maintenance.

---

## 19. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Maintenance with 503 but no Retry-After.**
Symptom: site returns to normal after maintenance but crawlers slow to re crawl.
Why it breaks: without Retry-After, crawlers guess; some treat it as transient, others as permanent.
Fix: always include Retry-After matching the planned duration.

**Pitfall 2: Maintenance window exceeds 48 hours.**
Symptom: URLs drop from Google index during prolonged maintenance.
Why it breaks: Google's 2 day rule.
Fix: keep maintenance under 48 hours, OR use crawler split pattern (Section 7.6), OR serve 200 with substantive maintenance content.

**Pitfall 3: FastAPI service has no `Restart=always`.**
Symptom: single crash takes the site offline for hours until manual intervention.
Why it breaks: systemd does not restart by default.
Fix: add `Restart=always` and `RestartSec=5` to the unit file.

**Pitfall 4: 503 cached by browsers and intermediaries.**
Symptom: maintenance ends but users still see 503 for hours.
Why it breaks: missing `Cache-Control: no-store`.
Fix: always include `Cache-Control: no-store, no-cache, must-revalidate` on 503 responses.

**Pitfall 5: nginx returns 503 for rate limiting (default behavior).**
Symptom: monitoring alerts on "service unavailable" but service is fine.
Why it breaks: nginx `limit_req` default is 503.
Fix: `limit_req_status 429;` (per framework-http-rate-control-headers.md).

**Pitfall 6: Long running endpoint hits 504 timeout.**
Symptom: report generation endpoint returns timeout error after 60 seconds.
Why it breaks: nginx default `proxy_read_timeout` is 60 seconds.
Fix: increase timeout for that specific location OR redesign with 202 Accepted + status URL.

**Pitfall 7: 500 errors silently ignored.**
Symptom: site appears to work; some specific actions intermittently fail.
Why it breaks: 5xx rate not monitored; users not reporting.
Fix: alert on 5xx rate; log every 5xx with request context.

**Pitfall 8: 502 from FastAPI crash during deploy.**
Symptom: brief 502 during deployment as old process stops and new starts.
Why it breaks: no graceful shutdown handling, no overlap during deploy.
Fix: graceful SIGTERM handling in FastAPI; for zero downtime, run two instances behind nginx and roll one at a time.

**Pitfall 9: 504 because of database deadlock.**
Symptom: specific endpoint occasionally takes 60+ seconds.
Why it breaks: long held database locks; concurrent transactions blocking each other.
Fix: query optimization, transaction isolation review, possibly redesign the operation.

**Pitfall 10: 5xx error page returns 200.**
Symptom: 500/502/503/504 errors served, but Google indexes the error page as a real page.
Why it breaks: nginx `error_page` directive may serve the page with the original error code, but some configs convert to 200.
Fix: verify with curl: status line should match the original error, not 200.

**Pitfall 11: nginx error_page directive misconfigured.**
Symptom: error pages return generic nginx defaults instead of custom pages.
Why it breaks: `error_page` not defined or `internal;` missing on the location.
Fix: ensure `error_page` directive in server block AND `internal;` on the location.

**Pitfall 12: FastAPI workers exhausted, no 5xx returned but slow.**
Symptom: site slow but no error code; users hang on requests.
Why it breaks: all uvicorn workers busy with long requests.
Fix: increase worker count (`--workers 4` or higher), or add async patterns to prevent worker exhaustion.

**Pitfall 13: Cloudflare 521 troubleshooting on a non Cloudflare site.**
Symptom: looking for Cloudflare specific 5xx codes that don't apply to Bubbles.
Why it breaks: confusion between Cloudflare codes and origin codes.
Fix: for Bubbles, 521/522/524 do not exist. Look for 502/504 from nginx instead.

**Pitfall 14: Health check endpoint depends on database, returns 503 when DB is slow.**
Symptom: monitoring shows site unhealthy when database is just busy, not down.
Why it breaks: confusing liveness with readiness.
Fix: split `/healthz` (liveness, just returns 200) from `/readyz` (readiness, checks dependencies).

**Pitfall 15: No log rotation, disk fills, all 5xx.**
Symptom: gradual increase in errors over weeks; eventually site unavailable.
Why it breaks: nginx and application logs fill disk; system unable to write.
Fix: ensure logrotate configured for `/var/log/nginx/*` and `/var/log/bubbles/*`. Monitor disk usage.

---

## 20. DIAGNOSTIC COMMANDS

Reference of every command useful for 5xx investigation.

### Quick health check

```bash
# All in one diagnostic
sudo bubbles-health-check.sh

# Or manually:
systemctl is-active nginx
systemctl is-active fastapi-sidecar
sudo ss -tlnp | grep :9090
curl -sI http://127.0.0.1:9090/healthz | head -1
curl -sI https://example.com/healthz | head -1
```

### Status code distribution from logs

```bash
# All 5xx in last hour
sudo awk -v since="$(date -d '1 hour ago' '+%d/%b/%Y:%H')" '
    $4 ~ since && $9 ~ /^5/ {count[$9]++}
    END {for (s in count) print s, count[s]}
' /var/log/nginx/access.log

# 5xx URLs (which endpoints are failing)
sudo awk '$9 ~ /^5/ {print $9, $7}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -20

# 5xx by status code over time (per minute)
sudo awk '$9 ~ /^5/ {
    gsub(/\[/, "", $4)
    print substr($4, 1, 17), $9
}' /var/log/nginx/access.log | sort | uniq -c | tail -30
```

### Locate FastAPI errors

```bash
# Recent errors only
sudo journalctl -u fastapi-sidecar --since "10 min ago" -p err

# Recent everything (more verbose)
sudo journalctl -u fastapi-sidecar --since "10 min ago"

# Search for specific exception types
sudo journalctl -u fastapi-sidecar | grep -iE "traceback|exception|error"

# Most recent crash
sudo journalctl -u fastapi-sidecar -p crit -n 10
```

### Investigate 502

```bash
# Is FastAPI running?
systemctl status fastapi-sidecar

# When was it last restarted?
sudo systemctl show fastapi-sidecar --property=ActiveEnterTimestamp

# Port listening?
sudo ss -tlnp | grep :9090

# Can nginx reach it?
sudo -u nginx curl -sI http://127.0.0.1:9090/healthz

# Recent nginx errors mentioning upstream
sudo tail -100 /var/log/nginx/error.log | grep upstream | tail -20
```

### Investigate 504

```bash
# Find slow endpoints
sudo awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Recent upstream timeouts in error log
sudo grep "upstream timed out" /var/log/nginx/error.log | tail -20

# Current uvicorn process usage
top -p $(pgrep -f uvicorn) -b -n 1

# Database slow queries (Postgres example)
sudo -u postgres psql -d bubbles -c "SELECT query, mean_exec_time, calls FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"
```

### Verify 503 maintenance config

```bash
# Confirm 503 with Retry-After
curl -sI https://example.com/ | grep -iE "^(HTTP|Retry-After|Cache-Control|X-Robots-Tag)"
# Expected:
# HTTP/2 503
# retry-after: 3600
# cache-control: no-store
# x-robots-tag: noindex

# Confirm the maintenance page is substantive
curl -s https://example.com/ -o /tmp/maintenance.html
WORDS=$(wc -w < /tmp/maintenance.html)
echo "Maintenance page words: $WORDS"
# Expected: 50+ words (avoid soft 404 risk)
```

### Restart sequence

```bash
# Standard restart sequence
sudo systemctl restart fastapi-sidecar
sleep 3
systemctl is-active fastapi-sidecar

# Reload nginx (if needed)
sudo nginx -t && sudo systemctl reload nginx

# Full restart (if needed)
sudo systemctl restart nginx

# Verify
curl -sI https://example.com/healthz | head -1
```

### Server resource investigation

```bash
# Disk space (prevent 507)
df -h /var/www /var/log

# Memory
free -h

# Load average
uptime

# Process count
ps aux | wc -l

# Open file descriptors (nginx limit)
sudo cat /proc/$(pgrep -f "nginx: master")/limits | grep "open files"

# Active connections
sudo ss -tan state established | wc -l
```

### Apply changes

```bash
# Test and reload nginx
sudo nginx -t && sudo systemctl reload nginx

# Reload systemd if unit file changed
sudo systemctl daemon-reload
sudo systemctl restart fastapi-sidecar
```

---

## 21. CROSS-REFERENCES

* [framework-http-2xx-status-codes.md](framework-http-2xx-status-codes.md): 200 with `X-Robots-Tag: noindex` is the alternative to 503 for long maintenance windows.
* [framework-http-3xx-status-codes.md](framework-http-3xx-status-codes.md): 301 to a temporary mirror is the alternative to 503 for extended maintenance.
* [framework-http-4xx-status-codes.md](framework-http-4xx-status-codes.md): 503 vs 429 distinction (covered there); custom error pages pattern.
* [framework-http-caching-headers.md](framework-http-caching-headers.md): `Cache-Control: no-store` on 5xx responses prevents cached errors.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): `X-Robots-Tag: noindex` on 5xx error pages prevents accidental indexing.
* [framework-http-rate-control-headers.md](framework-http-rate-control-headers.md): the full 503+Retry-After+crawler whitelist pattern. The 2 day rule for crawlers is documented at length here as well.
* [framework-http-request-headers.md](framework-http-request-headers.md): User-Agent based crawler whitelist for the crawler split maintenance pattern.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. The 2 day rule and sustained 5xx index removal are covered there.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including maintenance window planning.
* Google HTTP status codes documentation: https://developers.google.com/search/docs/crawling-indexing/http-network-errors
* Google maintenance handling: https://developers.google.com/search/docs/crawling-indexing/pause-online-business
* MDN 500: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/500
* MDN 502: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/502
* MDN 503: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/503
* MDN 504: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/504
* RFC 9110 (HTTP Semantics): https://www.rfc-editor.org/rfc/rfc9110
* nginx error_page directive: https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page
* nginx proxy_next_upstream: https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream
* systemd Restart configuration: https://www.freedesktop.org/software/systemd/man/systemd.service.html#Restart=

---

## 22. THE COMPLETE 12 FILE BUBBLES HTTP WIRE REFERENCE (SERIES SUMMARY)

This framework completes the series. The full reference is now 12 files covering every HTTP header and status code Bubbles can produce or receive.

### 22.1 The Complete File Set

**Track 1: HTTP Headers (response and request side)**

| File | Headers covered |
|---|---|
| framework-http-caching-headers.md | Cache-Control, ETag, Last-Modified, Expires, Vary, Age |
| framework-http-content-headers.md | Content-Type, Content-Language, Content-Encoding, Content-Length, Content-Disposition |
| framework-http-seo-headers.md | X-Robots-Tag, Link, Location |
| framework-http-security-headers.md | HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, CORP, COEP, COOP |
| framework-http-performance-headers.md | Server-Timing, Timing-Allow-Origin, Server, Alt-Svc |
| framework-http-cors-headers.md | Access-Control-Allow-Origin/Methods/Headers/Expose-Headers/Max-Age/Allow-Credentials |
| framework-http-rate-control-headers.md | Retry-After, X-RateLimit-*, RateLimit-* |
| framework-http-request-headers.md | User-Agent, Accept, Accept-Language, Accept-Encoding, Accept-Charset, From, Host, If-Modified-Since, If-None-Match, Referer, Connection |

**Track 2: HTTP Status Codes**

| File | Codes covered |
|---|---|
| framework-http-2xx-status-codes.md | 200, 201, 202, 204, 206 |
| framework-http-3xx-status-codes.md | 301, 302, 303, 304, 307, 308 |
| framework-http-4xx-status-codes.md | 400, 401, 403, 404, 410, 422, 429, 451 (plus 405, 408, 411, 413, 414, 415 briefly) |
| framework-http-5xx-status-codes.md (this) | 500, 502, 503, 504 (plus 505, 507, 508, 511 briefly, plus Cloudflare 520-527 briefly) |

### 22.2 The Cumulative Totals

* **Files**: 12
* **HTTP headers covered**: 46 (response and request side)
* **HTTP status codes covered**: 23 (primary) plus 12 briefly
* **Combined audit checklist items**: approximately 670
* **Common pitfalls documented**: approximately 195
* **Combined size**: approximately 1 MB of markdown reference

### 22.3 The Cross Cutting Patterns

Several patterns appear across multiple frameworks. The framework where each is first defined as the canonical reference:

* **The crawler taxonomy and verification** (Googlebot, ClaudeBot, GPTBot et al with current 2026 UA strings and DNS verification): framework-http-request-headers.md Sections 5 and 6.
* **The 2 day Google deindex rule** for sustained 5xx/429: framework-http-rate-control-headers.md Section 5.3, deeply cross referenced from framework-http-4xx-status-codes.md and this framework.
* **The Bubbles thin content cleanup (Joseph's 5,800 page work)**: framework-http-4xx-status-codes.md Section 17 (the 410 pattern) plus framework-http-2xx-status-codes.md Section 10 (the soft 404 trap context).
* **The 14 paying client subdomain redirects**: framework-http-3xx-status-codes.md Section 16 (the verbatim nginx config from the recent infrastructure overhaul).
* **The default_server returning 444**: framework-http-request-headers.md Section 12 (Host header routing).
* **The FastAPI sidecar architecture**: this framework Section 13 (the canonical diagnostic pattern).

### 22.4 The Operating Conventions

All 12 frameworks consistently follow Joseph's standing rules:

* Zero em dashes, en dashes, smart quotes, or Unicode arrows.
* Nginx focused for Bubbles (Debian, no Cloudflare).
* `/var/www/sites/[domain]/` paths.
* Every nginx change ends with `nginx -t && systemctl reload nginx`.
* FastAPI sidecar on port 9090.
* 50+ item audit checklist per framework.
* 15 common pitfalls per framework.
* Diagnostic commands section per framework.
* Cross references to siblings and authoritative external docs (RFCs, MDN, Google).

### 22.5 The Usage Pattern

For any new Bubbles client deployment, the workflow:

1. Provision the server, install nginx and the FastAPI sidecar.
2. Apply the standard configurations from the relevant framework's "Bubbles Nginx Reference Block" section.
3. Run the audit checklists across all 12 frameworks (the combined ~670 items).
4. Verify each test from the diagnostic commands sections.
5. Document deviations and gaps for the specific client's needs.

For any incident:

1. Identify the affected status code (which framework covers it).
2. Use that framework's diagnostic commands section.
3. Apply the recovery pattern from the relevant recipe section.
4. Confirm via the verification commands.

### 22.6 The Living Document Note

HTTP is a stable protocol. RFCs change rarely. Crawler behaviors change occasionally (Google deprecates Claude-Web in favor of ClaudeBot, OpenAI splits training from search). The frameworks are designed to be updated incrementally as the ecosystem evolves; the underlying patterns (the 2 day rule, the soft 404 trap, the 401 vs 403 distinction) are stable across decades.

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Status code decision tree

```
Server side failure. Which one?
   |
   |---> Application threw exception ........................ 500 Internal Server Error
   |
   |---> Application not running (port not bound) ........... 502 Bad Gateway
   |
   |---> Application returned garbage to nginx ............... 502 Bad Gateway
   |
   |---> Application accepted request but didn't respond .... 504 Gateway Timeout
   |
   |---> Operator intentionally signaling unavailability .... 503 Service Unavailable (+ Retry-After)
```

### The 2 day rule

Sustained 503 or 429 for more than 48 hours = URLs drop from Google index.

* Under 24 hours: 503 with Retry-After is safe.
* 24 to 48 hours: 503 with Retry-After is risky; use crawler split pattern.
* Over 48 hours: do NOT use 503. Serve 200 with substantive maintenance content.

### Bubbles 5xx most likely causes

| Symptom | Most likely cause | Fix |
|---|---|---|
| 502 immediately after deploy | FastAPI failed to start | `journalctl -u fastapi-sidecar` |
| 502 intermittent | FastAPI crashes, no restart | systemd Restart=always |
| 504 specific endpoint | Slow query, slow API call | Profile, optimize, or async |
| 500 specific endpoint | Application bug | Fix the bug |
| 503 unexpected | limit_req default | `limit_req_status 429` |
| All 5xx | Disk full, OOM, etc | Resource check |

### Five rules to memorize

1. 503 needs Retry-After. Always.
2. Maintenance windows under 48 hours. Always.
3. systemd Restart=always for FastAPI. Always.
4. Health check (`/healthz`) independent of dependencies.
5. nginx `proxy_next_upstream` for transient failures.

### Five commands every operator should know

```bash
# 1. Run the health check
sudo bubbles-health-check.sh

# 2. Find current 5xx rate
sudo awk '$9 ~ /^5/ {c++} END {print c+0}' /var/log/nginx/access.log

# 3. Restart FastAPI
sudo systemctl restart fastapi-sidecar

# 4. Check upstream timeouts
sudo grep "upstream timed out" /var/log/nginx/error.log | tail -10

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Health check returns 200
[ $(curl -so /dev/null -w "%{http_code}" https://example.com/healthz) = "200" ] && \
    echo "OK: healthz" || echo "FAIL: healthz"

# 2. Maintenance 503 has Retry-After when active
# (Only run during planned maintenance)
RA=$(curl -sI https://example.com/ | grep -i retry-after | wc -l)
[ $RA = "1" ] && echo "OK: Retry-After present" || echo "Verify maintenance config"

# 3. systemd self healing works
sudo systemctl stop fastapi-sidecar
sleep 6
[ "$(systemctl is-active fastapi-sidecar)" = "active" ] && \
    echo "OK: self healed" || echo "FAIL: did not restart"
```

If all three pass AND no sustained 5xx in the last 24 hours, the 5xx layer is correctly wired.

---

End of framework-http-5xx-status-codes.md.

End of the HTTP status codes series.

End of the 12 file Bubbles HTTP wire reference.
