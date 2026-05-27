# framework-http-performance-headers.md

Comprehensive reference for the four HTTP response headers that expose, control, or advertise performance characteristics of the server and its protocols: `Server-Timing` for backend timing telemetry, `Timing-Allow-Origin` for cross origin access to Resource Timing data, `Server` for stack disclosure (and how to suppress it), and `Alt-Svc` for HTTP/3 advertisement. Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to framework-http-caching-headers.md, framework-http-content-headers.md, framework-http-seo-headers.md, framework-http-security-headers.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx, AI assistants generating or repairing nginx config, performance engineers building RUM dashboards, security auditors checking for fingerprint leakage, and anyone troubleshooting "Resource Timing returning zeros for our own CDN", "HTTP/3 not being negotiated despite being enabled", or "Server header leaking nginx version to scanners" anomalies on a self hosted stack.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Performance Observability Mental Model (read this first)
5. Server-Timing (backend timing exposed to the browser)
6. Timing-Allow-Origin (cross origin permission for Resource Timing API)
7. Server (stack disclosure and how to suppress it)
8. Alt-Svc (HTTP/3 advertisement)
9. How These Headers Interact
10. Asset Class And Use Case Recipes
11. Bubbles Nginx Reference Block (paste ready)
12. Audit Checklist (50+ items)
13. Common Pitfalls
14. Diagnostic Commands (curl, browser PerformanceObserver, chrome://net-internals)
15. Cross-References

---

## 1. DEFINITION

Performance and protocol headers expose, restrict, or advertise capabilities and timing data that browsers and monitoring tools use to measure and optimize page load. They differ from the caching, content, SEO, and security families in that they do not directly affect what the user sees; they affect what the operator can measure and what protocols the browser negotiates on subsequent connections. The four headers split into three concerns:

* **Telemetry**: `Server-Timing`, `Timing-Allow-Origin`. They answer "how long did the backend spend, and may my monitoring code read detailed timing for cross origin resources?"
* **Stack disclosure**: `Server`. It answers "what software is serving this response?" The practical question is whether to reveal that information or suppress it.
* **Protocol advertisement**: `Alt-Svc`. It answers "what alternative protocols are available at this origin?" In 2026 the primary use is advertising HTTP/3 over QUIC.

Together these four headers determine how visible the server's performance is to operators, monitoring tools, and visitors' browsers, and what protocols the browser uses for subsequent connections. Getting them right closes a real visibility gap in RUM dashboards, unlocks HTTP/3 negotiation for measurable LCP gains, and reduces the attack surface from server fingerprinting.

---

## 2. WHY IT MATTERS

Five independent pressures push correct performance headers from "nice to have" to "required infrastructure" in 2025 and forward.

**Real User Monitoring needs server side context.** Browser Performance API exposes TTFB, LCP, CLS, INP, and detailed network timings, but cannot see inside the server. A 400 ms TTFB tells you the response was slow; it does not tell you whether 350 ms was database query and 50 ms was template render, or vice versa. `Server-Timing` exposes that breakdown to the same RUM beacon that captures Core Web Vitals, making backend performance correlatable with frontend metrics without separate APM instrumentation.

**Cross origin resources are blind by default.** Browsers zero out detailed Resource Timing API data for cross origin resources to prevent timing based fingerprinting. Without `Timing-Allow-Origin`, your monitoring code sees `connectStart=0, responseStart=0, transferSize=0` for every CSS, JS, font, and image loaded from a different origin (a CDN, a font provider, a static asset subdomain). The RUM dashboard shows phantom data with zero values. Solving this is one header per resource.

**Server fingerprinting accelerates exploitation.** Server: nginx/1.24.0 tells an attacker exactly which CVE list to consult. A scanner can identify your stack within seconds of touching a single URL. Suppressing version disclosure (or the entire Server header) is one of the cheapest defenses in depth measures available. Every security scanner in the procurement audit suite flags this and grades it.

**HTTP/3 is real and measurable.** Chrome, Edge, Firefox, and Safari all support HTTP/3 over QUIC in 2026. Sites running HTTP/3 see 50 to 200 ms TTFB improvement on mobile and lossy networks compared to HTTP/2, primarily from 1-RTT handshakes and head of line blocking elimination. But the browser only uses HTTP/3 after seeing `Alt-Svc: h3=":443"`. Sites that have HTTP/3 enabled in nginx but never set Alt-Svc never get HTTP/3 traffic.

**Cost of getting it wrong.** Misconfigured performance headers produce silent visibility gaps and missed optimizations. Real examples:

* RUM dashboard shows 30 percent of requests with `responseEnd=0, transferSize=0`. Investigation reveals every asset on the cdn subdomain is cross origin without Timing-Allow-Origin. Three quarters of the timeline is missing.
* TTFB tracking flat at 400 ms despite a database query optimization that should have cut it in half. No Server-Timing exposed; operator cannot tell the optimization actually deployed without server log spelunking.
* Security scanner flags Server header disclosing nginx/1.24.0. Procurement review flags the site as "uses outdated nginx" (1.24.0 was current at audit time) and recommends vendor switch.
* Site claims to support HTTP/3 in marketing materials but `curl --http3` fails. Investigation reveals nginx http3 directives are present but `Alt-Svc` header was forgotten. Browsers never opt in to HTTP/3.
* Server-Timing exposes database query strings as `desc` values. Penetration test extracts schema details by reading the headers. CVE filed for information disclosure.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the four headers gets the same six part treatment:

1. **What it does**: the canonical W3C or IETF spec plus the practical implication.
2. **Syntax and parameters**: every legal value, what it means, and when it is wrong.
3. **How to build it on Bubbles**: paste ready nginx config, plus FastAPI sidecar code where the upstream is the natural emitter.
4. **How to verify it**: curl commands plus browser PerformanceObserver checks where applicable.
5. **How to troubleshoot**: the four or five failure modes seen in the field and how to recognize each.
6. **How to fix common breakage**: ordered repair steps.

Recipes for the most common use cases are collected in Section 10.

---

## 4. THE PERFORMANCE OBSERVABILITY MENTAL MODEL (READ THIS FIRST)

Every visit to a Bubbles site emits a stream of timing data the operator may collect. Where that data comes from and what controls its visibility is governed by these four headers. Internalize the data flow and every header decision becomes obvious.

```
Visitor browser requests https://example.com/page
        |
        v
==================== PROTOCOL NEGOTIATION ====================
        |
        v
First visit: TCP+TLS+HTTP/2 (or HTTP/1.1)
        |
        v
Server returns response with Alt-Svc: h3=":443"; ma=86400
        |
        v
Browser remembers: HTTP/3 available at this origin for 86400 seconds
        |
        v
Next visit (within ma seconds): browser tries HTTP/3 first via QUIC over UDP
        |
        v
==================== BACKEND TIMING ====================
        |
        v
Upstream (FastAPI sidecar on port 9090) processes request
        |
        v
Sidecar measures: db query took 12 ms, template render took 8 ms, cache miss
        |
        v
Sidecar emits: Server-Timing: db;dur=12, render;dur=8, cache;desc=miss
        |
        v
Nginx passes Server-Timing through to client
        |
        v
==================== STACK DISCLOSURE ====================
        |
        v
Nginx considers whether to emit Server: nginx/1.26.0
        |
        v
server_tokens off => emits Server: nginx (no version)
server_tokens build => emits Server: nginx (built with custom name)
headers-more module + more_set_headers "Server: " => suppresses entirely
        |
        v
==================== BROWSER OBSERVATION ====================
        |
        v
Browser receives response, parses, fires PerformanceObserver
        |
        v
For same origin resources: full timing available always
For cross origin resources: timing zeroed unless Timing-Allow-Origin allows
        |
        v
RUM library reads PerformanceResourceTiming, PerformanceServerTiming
        |
        v
RUM library beacons data back to monitoring endpoint
        |
        v
Operator dashboard shows full performance breakdown including backend
```

Four rules govern the system:

1. **Tell the browser what is available, only the once.** Alt-Svc is set on any response; the browser caches the protocol info and uses it for the configured duration. A single response is sufficient to upgrade subsequent connections.
2. **Expose what you want measured, suppress what you do not.** Server-Timing exposes backend phases to the browser and any script running there. Decide which phases are useful for RUM and which would leak too much detail.
3. **Allow timing access deliberately.** Timing-Allow-Origin is opt in. Without it, cross origin resources are timing opaque. With it, the listed origins (or all origins with `*`) see detailed timings.
4. **Suppress fingerprintable defaults.** The default Server header value (nginx/1.26.0) is more information than is needed. `server_tokens off` reduces it to `nginx`; the headers-more module can remove it entirely.

A correctly configured performance header stack produces measurable HTTP/3 traffic share in access logs, populated cross origin timings in RUM dashboards, observable backend phase breakdowns alongside browser timings, and no version disclosure to scanners.

---

## 5. SERVER-TIMING (BACKEND TIMING EXPOSED TO THE BROWSER)

### 5.1 What It Does

`Server-Timing` is a response header that communicates one or more metrics from the server side processing of a request. Defined in the W3C Server Timing specification. The metrics are made available to JavaScript on the page via the `PerformanceServerTiming` interface, which means RUM libraries can collect backend timing alongside browser timing in a single beacon.

```
Server-Timing: db;dur=12.4
Server-Timing: db;dur=12.4, render;dur=8.2, cache;desc=hit
Server-Timing: total;dur=187;desc="full request"
```

A typical RUM workflow: the backend records timestamps at meaningful phase boundaries (request start, database query end, template render end, cache lookup result), computes durations, and emits them as Server-Timing entries. The browser receives them. A RUM script on the page reads them via `performance.getEntriesByType("navigation")[0].serverTiming` and beacons them to the monitoring endpoint. The dashboard now shows a single view of backend and frontend performance.

Critical: `Server-Timing` exposes its values to any script running on the page, including third party scripts. Decide carefully which metrics to expose. Database query identifiers, file paths, internal service names, and similar implementation details should never appear in Server-Timing values that ship to public pages.

### 5.2 Syntax

The header value is a comma separated list of metrics. Each metric has a name and optional parameters.

```
Server-Timing: <name>[;dur=<duration>][;desc=<description>][, ...]
```

| Field | Purpose | Format |
|---|---|---|
| `<name>` | Identifier for the metric. Required | Token (no spaces, no special characters) |
| `dur=<duration>` | Time taken in milliseconds. Optional | Number, may include decimals |
| `desc=<description>` | Human readable description. Optional | Token or quoted string |

Examples:

```
Server-Timing: db;dur=12
Server-Timing: db;dur=12;desc="user lookup"
Server-Timing: db;dur=12, cpu;dur=45
Server-Timing: cache;desc=miss
Server-Timing: total;dur=187
```

Multiple Server-Timing headers may appear on the same response; they are processed as if joined by commas.

### 5.3 Metric Design Principles

The single most important design decision: what to measure and what to expose.

**Useful metrics to expose:**

* `total` or `app`: total backend processing time. Lets RUM correlate against TTFB.
* `db`: database query time as an aggregate.
* `render`: template or response body generation time.
* `cache`: cache lookup result (`desc=hit` or `desc=miss`). Even just a `desc` with no `dur` is valuable.
* `cdn` or `origin`: when the response was served from cache vs computed at origin.
* `auth`: authentication or authorization time. Useful when slow auth dominates TTFB.
* `external`: time spent waiting on external API calls.

**Metrics to avoid exposing:**

* Specific database query identifiers (`db.users.lookup-by-id-12847`).
* File paths (`render.template=/var/www/.../views/profile.html`).
* Internal service names (`api.auth-service.v3-staging`).
* Sensitive timing that could reveal authentication state (a slow vs fast auth path may reveal whether a user exists).

**Naming convention:** short, lowercase, hyphen separated. Match the convention to keep dashboards readable.

### 5.4 How To Build It On Bubbles

`Server-Timing` is generated by the upstream application, not by nginx. The natural place for it is the FastAPI sidecar on port 9090. Nginx passes it through.

**FastAPI sidecar emitting Server-Timing:**

```python
import time
from contextlib import contextmanager
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse

app = FastAPI()

class Timer:
    def __init__(self):
        self.metrics = []
        self._start_total = time.perf_counter()

    @contextmanager
    def phase(self, name, desc=None):
        start = time.perf_counter()
        yield
        dur_ms = (time.perf_counter() - start) * 1000
        entry = f"{name};dur={dur_ms:.1f}"
        if desc:
            entry += f';desc="{desc}"'
        self.metrics.append(entry)

    def mark(self, name, desc):
        self.metrics.append(f'{name};desc="{desc}"')

    def header_value(self):
        total_ms = (time.perf_counter() - self._start_total) * 1000
        self.metrics.append(f"total;dur={total_ms:.1f}")
        return ", ".join(self.metrics)


@app.get("/page")
async def render_page(request: Request):
    timer = Timer()

    with timer.phase("db", "user lookup"):
        user = await fetch_user(request)

    with timer.phase("render"):
        html = render_template("page.html", user=user)

    timer.mark("cache", "miss")

    return HTMLResponse(
        content=html,
        headers={"Server-Timing": timer.header_value()}
    )
```

Output header (example):

```
Server-Timing: db;dur=12.4;desc="user lookup", render;dur=8.2, cache;desc=miss, total;dur=24.1
```

**Nginx pass through:**

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_pass_header Server-Timing;
    # Default nginx pass through of response headers includes Server-Timing
    # The proxy_pass_header line is explicit; usually not needed
}
```

**Nginx adding its own Server-Timing metrics:**

Nginx Plus and OpenResty can add Server-Timing with internal nginx variables. On vanilla nginx, the cleanest approach is to inject via the upstream. If absolutely needed, use the headers-more module:

```nginx
location / {
    more_set_headers "Server-Timing: nginx;dur=$request_time";
}
```

`$request_time` is the time nginx spent processing the request including upstream wait. For pure static files served by nginx directly, this is the total request handling time. For proxied responses, it includes upstream latency.

### 5.5 RUM Integration

A minimal RUM snippet that reads Server-Timing and includes it in a beacon:

```html
<script>
  (function() {
    function reportTiming() {
      const nav = performance.getEntriesByType("navigation")[0];
      if (!nav) return;

      const serverTimings = nav.serverTiming || [];
      const data = {
        url: location.href,
        ttfb: nav.responseStart - nav.requestStart,
        lcp: null,  // populated by PerformanceObserver
        server: {}
      };

      for (const t of serverTimings) {
        data.server[t.name] = {
          duration: t.duration,
          description: t.description
        };
      }

      navigator.sendBeacon("/rum", JSON.stringify(data));
    }

    window.addEventListener("load", () => setTimeout(reportTiming, 0));
  })();
</script>
```

The `/rum` endpoint on the FastAPI sidecar receives the beacon and writes to whatever monitoring backend you use (Prometheus, ClickHouse, file logs, Datadog, etc).

### 5.6 How To Verify

```bash
# 1. Confirm header is present
curl -sI https://example.com/page | grep -i server-timing
# Expected: server-timing: db;dur=12.4, render;dur=8.2, total;dur=24.1

# 2. Verify multiple metrics
curl -sI https://example.com/page | grep -i server-timing | tr ',' '\n' | sed 's/^ */    /'

# 3. Test in browser DevTools:
# Open DevTools, Network tab, click a request, Timing tab
# Look for "Server Timing" section showing each metric
# Or in Console: performance.getEntriesByType("navigation")[0].serverTiming
# Expected: array of PerformanceServerTiming objects

# 4. Verify nginx passes through correctly
curl -sI -A "test" https://example.com/api/data | grep -i server-timing

# 5. Test in static file location (should be absent for pure static)
curl -sI https://example.com/static/main.css | grep -i server-timing
# Expected: empty (no Server-Timing on pure static unless explicitly added)

# 6. Sanity check: total dur should match observed request time
DUR=$(curl -sI https://example.com/page | grep -i server-timing | grep -oE "total;dur=[0-9.]+" | cut -d= -f2)
echo "Server reports total = $DUR ms"
curl -so /dev/null -w "Curl observed = %{time_starttransfer} s\n" https://example.com/page
```

### 5.7 Troubleshooting

**Symptom: Server-Timing not appearing in response.**
Causes ranked by frequency:
1. Upstream is not emitting it. Verify by curling the upstream directly: `curl -sI http://127.0.0.1:9090/page | grep -i server-timing`.
2. Nginx is stripping it. Check for `proxy_hide_header Server-Timing` in the config (rare).
3. The location is serving static files, not the upstream. Static files have no application timing to expose.

**Symptom: Header appears in curl but not in browser PerformanceServerTiming.**
1. The request was cross origin without Timing-Allow-Origin. See Section 6.
2. The browser is reading from cache (304 response). Cached responses may or may not include the original Server-Timing.
3. RUM library is reading wrong API. Use `performance.getEntriesByType("navigation")[0].serverTiming` for the page itself, and `performance.getEntriesByType("resource")` for subresources.

**Symptom: Total dur is much higher than TTFB.**
The total measured server side does not match what the browser observes because of TCP queue time, TLS handshake, body streaming, or other transport overhead. Server-Timing captures the application processing time; TTFB captures the browser perspective. The two are correlated but not identical.

**Symptom: Server-Timing values include sensitive data (file paths, queries).**
Audit the upstream code. Anything in `desc=` is publicly visible. Restrict to safe identifiers only.

**Symptom: Header is very long, exceeds nginx buffer limit.**
Reduce the number of metrics or shorten descriptions. Most useful information fits in 200 to 500 bytes.

### 5.8 How To Fix Common Breakage

**Case: Want backend timing visible in RUM but FastAPI is not emitting Server-Timing.**
Add the Timer class shown in Section 5.4 to the FastAPI codebase. Wrap each meaningful phase. Emit the header on the response.

**Case: Server-Timing exposes the database name, which should not be public.**
Fix at the upstream. Change `db;desc="postgres-prod-replica"` to just `db;dur=N` with no desc, or use a generic identifier.

**Case: Need to add Server-Timing on responses served directly by nginx without a sidecar.**
For static files there is no meaningful application timing. The closest useful metric is `$request_time` exposed via headers-more:

```nginx
location ~* \.html$ {
    more_set_headers "Server-Timing: nginx;dur=$request_time";
}
```

This tells the visitor's RUM how long nginx took. For pure static, that is almost zero (single digit milliseconds typically), so the value is informational rather than actionable.

---

## 6. TIMING-ALLOW-ORIGIN (CROSS ORIGIN PERMISSION FOR RESOURCE TIMING API)

### 6.1 What It Does

`Timing-Allow-Origin` is the opt in mechanism that grants other origins permission to read detailed timing data for a resource via the Resource Timing API. Defined in the W3C Resource Timing specification. Without this header, cross origin resources expose only `startTime`, `duration`, and `responseEnd`; everything else (DNS lookup, TCP connect, TLS handshake, request start, response start, transfer size, encoded body size, decoded body size, next hop protocol) is zeroed out.

```
Timing-Allow-Origin: *
Timing-Allow-Origin: https://example.com
Timing-Allow-Origin: https://example.com, https://www.example.com
```

The header has nothing to do with whether the resource may be loaded (that is CORS via `Access-Control-Allow-Origin`). It controls only whether timing data may be read after the resource has loaded.

### 6.2 Syntax

```
Timing-Allow-Origin: <origin> [, <origin>]*
Timing-Allow-Origin: *
Timing-Allow-Origin: null
```

| Value | Meaning |
|---|---|
| `*` | Any origin may read full timing |
| `<origin>` | Only the specified origin may read full timing |
| `<origin>, <origin>` | Multiple specific origins |
| `null` | The origin "null" (rare; sandboxed iframes and data URLs) |

The header may appear once with multiple origins, or multiple times. Both forms are equivalent.

### 6.3 What Is Hidden Without Timing-Allow-Origin

For cross origin resources, the following attributes are set to 0 (numbers) or empty string (strings):

* `redirectStart`, `redirectEnd`
* `domainLookupStart`, `domainLookupEnd` (DNS)
* `connectStart`, `connectEnd` (TCP)
* `secureConnectionStart` (TLS)
* `requestStart` (request emission)
* `responseStart` (first byte of response)
* `transferSize` (wire bytes)
* `encodedBodySize`
* `decodedBodySize`
* `firstInterimResponseStart`
* `finalResponseHeadersStart`
* `nextHopProtocol` (becomes empty string)
* `contentType`

The only attributes available without Timing-Allow-Origin: `name`, `entryType`, `startTime`, `duration`, `responseEnd`, `initiatorType`, `renderBlockingStatus`.

For most RUM purposes, this is unusable. The dashboard cannot tell whether a slow cross origin resource was slow because of DNS, TCP, TLS, or transfer time. The whole point of Resource Timing is the breakdown.

### 6.4 Bubbles Pattern: Self Hosted Origin Means Mostly Same Origin

On Bubbles, every paying client site serves its own assets from the same domain. There is no separate CDN origin. As a result, **same origin Resource Timing is fully available without any Timing-Allow-Origin header**. The header is needed only for the few cases of legitimate cross origin assets:

* Third party fonts (if not self hosted).
* External analytics scripts (GTM, GA4, Facebook Pixel, etc).
* Third party widgets (chat, social embeds).
* Subdomain assets if some sites use one (rare on Bubbles).

For the third party assets: you cannot set Timing-Allow-Origin on Google's servers. Either accept the blind spot or self host those assets.

**Bubbles policy:** self host every static asset on the main domain. The asset Resource Timing is automatically same origin and fully visible. Where third party scripts are unavoidable (GTM, analytics), accept the timing blind spot and note it in monitoring.

### 6.5 How To Build It On Bubbles

For a static asset CDN subdomain (rare in the Bubbles fleet):

```nginx
server {
    server_name cdn.example.com;

    location / {
        add_header Timing-Allow-Origin "https://example.com, https://www.example.com" always;
        # Also typically needs CORS for cross origin script and font loading
        add_header Access-Control-Allow-Origin "https://example.com" always;
    }
}
```

For an embeddable widget served to partners:

```nginx
server {
    server_name widget.example.com;

    location / {
        add_header Timing-Allow-Origin "*" always;
        add_header Access-Control-Allow-Origin "*" always;
    }
}
```

For typical Bubbles sites: **do not set Timing-Allow-Origin** because all assets are same origin and the header is unnecessary.

### 6.6 How To Verify

```bash
# 1. Check if header is set (for cross origin asset endpoints)
curl -sI https://cdn.example.com/main.css | grep -i timing-allow-origin

# 2. Verify Resource Timing works in browser DevTools
# Console:
# const entries = performance.getEntriesByType("resource");
# entries.forEach(e => console.log(e.name, e.transferSize, e.responseStart));
# Same origin: real numbers
# Cross origin without TAO: zeros
# Cross origin with TAO=*: real numbers

# 3. Test from a different origin
# On a different domain, load the resource and check timing visibility

# 4. For the Bubbles same origin pattern:
curl -sI https://example.com/css/main.css | grep -i timing-allow-origin
# Expected: no header (not needed for same origin)
curl -sI https://example.com/css/main.css | grep -i content-type
# Verify the asset itself is served correctly (Timing-Allow-Origin is irrelevant here)
```

### 6.7 Troubleshooting

**Symptom: RUM dashboard shows zero transferSize and zero responseStart for many resources.**
Cause: those resources are cross origin and lack Timing-Allow-Origin.
Fix:
1. If they are on your own subdomain: add the header at that subdomain.
2. If they are third party (Google Fonts, GTM, analytics): you cannot fix it. Self host or accept the blind spot.

**Symptom: Some attributes available, others zero.**
Some browsers selectively zero only the sensitive attributes (transferSize, encodedBodySize, decodedBodySize) even when Timing-Allow-Origin is `*`. This is per the spec: TAO grants access to timing, but body size attributes have additional restrictions. The spec is evolving; in 2026 this affects only specific edge cases.

**Symptom: Same origin resources also show zero timing.**
Not a TAO issue. Possible causes:
1. The resource was served from `disk cache` or `memory cache` (browser cache). Resource Timing entries for cached resources have different shapes.
2. The fetch was a `no-cors` request (cross origin without CORS). Different attribute restrictions apply.
3. A service worker is intercepting; Resource Timing exposes service worker phases.

**Symptom: Header set to `*` but security scanner flags it.**
`*` allows any origin to read timing. For a public CDN, this is intended. For an internal asset that should not leak timing, restrict to specific origins.

### 6.8 How To Fix Common Breakage

**Case: Move third party fonts to self hosted to fix Resource Timing.**
Download woff2 files from Google Fonts (the URLs are stable; right click the network tab and save):

```bash
mkdir -p /var/www/sites/example.com/fonts
cd /var/www/sites/example.com/fonts
# Download font files referenced in CSS
curl -O https://fonts.gstatic.com/s/manrope/v15/...woff2
# Update CSS @font-face to reference local URLs
```

After self hosting, the fonts are same origin and Resource Timing works without any header.

**Case: Partner integration needs to see resource timing for embedded asset.**
Add their origin to TAO:

```nginx
location /widget.js {
    add_header Timing-Allow-Origin "https://partner.example.com" always;
    add_header Access-Control-Allow-Origin "https://partner.example.com" always;
}
```

Both headers; CORS for loading, TAO for timing visibility.

---

## 7. SERVER (STACK DISCLOSURE AND HOW TO SUPPRESS IT)

### 7.1 What It Does

`Server` is the standard HTTP response header that identifies the server software. Set automatically by nginx (and by most other servers) with the software name and often the version. Defined in RFC 9110.

```
Server: nginx
Server: nginx/1.26.0
Server: Apache/2.4.62 (Debian)
Server: Microsoft-IIS/10.0
```

The header is informational. It serves no functional purpose for clients. Its only practical effect is to tell external observers (scanners, crawlers, attackers, monitoring tools, browser DevTools) what server software is running.

### 7.2 Why Suppress It

The Server header is a fingerprinting signal. An attacker who knows you run `nginx/1.26.0` can immediately consult the CVE list for that version and target known vulnerabilities. An automated mass exploit scanner picks the lowest hanging fruit first; anything that makes you a more interesting or easier target is undesirable.

In 2026 this is "security through obscurity" in the strict sense: a determined attacker can fingerprint your stack through other means (TLS handshake fingerprints via JA3, response timing characteristics, error page formats, header order). But obscurity raises the bar for automated scanners and casual reconnaissance, which is real value at zero cost.

Compliance frameworks routinely flag version disclosure:

* PCI DSS: requires removal of "unnecessary information" from headers.
* OWASP Top 10: server fingerprinting falls under A05 Security Misconfiguration.
* Most pen test reports include "version disclosure" as a low severity finding that gets fixed before approval.

### 7.3 The Three Levels Of Suppression

| Level | What Server header looks like | How to achieve |
|---|---|---|
| Default | `Server: nginx/1.26.0` | Default nginx behavior |
| Suppress version | `Server: nginx` | `server_tokens off` |
| Suppress entirely | (no Server header) | `headers-more` module with `more_clear_headers Server` |

**Suppress version (`server_tokens off`):** the simplest and most common. Achieved in vanilla nginx, no extra modules. The Server header still appears but only as `nginx`, with no version. This satisfies most scanners and audit requirements.

**Suppress entirely:** requires the `headers-more-nginx-module`. Removes the Server header from all responses. Useful for maximum opacity. Some intermediate proxies and load balancers may add their own Server header back; verify after deployment.

**Suppress and replace:** also requires `headers-more`. Replaces the Server header with a custom string (e.g. `Server: Bubbles`). Provides a vanity branding option while not leaking the real stack.

### 7.4 server_tokens Directive

The full directive:

```nginx
server_tokens on | off | build | string;
```

| Value | Behavior |
|---|---|
| `on` (default) | Emit nginx and version: `nginx/1.26.0` |
| `off` | Emit nginx only: `nginx` |
| `build` | Emit nginx, version, AND build name (rare) |
| `string` | Custom string (requires nginx Plus, commercial subscription) |

For Bubbles (open source nginx on Debian): `off` is the recommended setting. It is set at the http level and inherited by all servers and locations.

### 7.5 Removing The Header Entirely

If the `headers-more-nginx-module` is installed (available as a Debian package `libnginx-mod-http-headers-more-filter`):

```nginx
http {
    server_tokens off;
    more_clear_headers Server;
}
```

`more_clear_headers Server` removes the Server header entirely from all responses. Verify after deployment:

```bash
curl -sI https://example.com/ | grep -i "^server:"
# Expected: no output
```

### 7.6 How To Build It On Bubbles

Standard Bubbles config (suppress version only):

```nginx
# /etc/nginx/nginx.conf, in the http block
http {
    server_tokens off;
    # ... other config ...
}
```

Apply globally; no per server override needed.

Maximum opacity (suppress entirely):

```bash
# Install headers-more module
sudo apt install libnginx-mod-http-headers-more-filter
```

```nginx
# /etc/nginx/nginx.conf
http {
    server_tokens off;
    more_clear_headers Server;
}
```

Custom replacement (vanity branding):

```nginx
http {
    server_tokens off;
    more_set_headers "Server: Bubbles";
}
```

Now every response says `Server: Bubbles`. Does not reveal nginx and avoids version disclosure entirely.

### 7.7 How To Verify

```bash
# 1. Check current value
curl -sI https://example.com/ | grep -i "^server:"
# Expected (after server_tokens off): server: nginx
# Expected (after more_clear_headers): no output
# Expected (after custom): server: Bubbles

# 2. Verify across all responses including errors
for path in / /404-test /admin /robots.txt; do
    HEADER=$(curl -sI "https://example.com$path" 2>/dev/null | grep -i "^server:")
    echo "$path: $HEADER"
done

# 3. Check that nginx version is not leaked elsewhere (error pages)
curl -s https://example.com/this-does-not-exist.html | grep -i nginx
# Expected: no output (or your custom 404 page without nginx mention)

# 4. Use security scanner
echo "Visit: https://securityheaders.com/?q=https://example.com/"
# Should NOT flag "Server header version disclosure"

# 5. Look at full server fingerprint surface
curl -sIv https://example.com/ 2>&1 | grep -iE "^< (server|x-powered-by|via):"
```

### 7.8 Troubleshooting

**Symptom: server_tokens off applied but Server header still shows version.**
1. The directive is in the wrong context. It must be at http, server, or location level.
2. nginx was not reloaded. `nginx -t && systemctl reload nginx`.
3. An intermediate (load balancer, reverse proxy) is replacing the header. Not applicable on Bubbles.
4. Custom build of nginx that ignores `server_tokens`. Rare.

**Symptom: more_clear_headers Server does not work.**
1. The `headers-more` module is not installed or not loaded. Install via `apt install libnginx-mod-http-headers-more-filter`. Verify with `nginx -V 2>&1 | grep headers-more`.
2. The directive is in the wrong scope. It works in http, server, location.

**Symptom: Error pages still show nginx version.**
nginx default error pages (`/usr/share/nginx/html/50x.html` etc) contain the version in HTML. Override with custom error pages:

```nginx
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;
```

Where `/404.html` and `/50x.html` are your own custom error pages without any nginx mention.

**Symptom: X-Powered-By or other stack headers still appearing.**
The Server header is not the only fingerprint. Audit and suppress:

```nginx
# In addition to Server suppression
more_clear_headers "X-Powered-By";
more_clear_headers "X-AspNet-Version";
more_clear_headers "X-AspNetMvc-Version";
```

The FastAPI sidecar should also be configured to not emit `X-Powered-By` or similar.

**Symptom: Vanity Server header value is too long or contains illegal characters.**
The Server header value must be a token list. Avoid spaces (unless quoted), commas, control characters.

### 7.9 How To Fix Common Breakage

**Case: Scanner flags nginx version disclosure.**
Apply `server_tokens off`:

```nginx
http {
    server_tokens off;
}
```

`nginx -t && systemctl reload nginx`. Verify and re scan.

**Case: Need maximum security opacity for high value site.**
Install headers-more and remove the Server header entirely:

```bash
sudo apt install libnginx-mod-http-headers-more-filter
```

```nginx
http {
    server_tokens off;
    more_clear_headers Server;
    more_clear_headers "X-Powered-By";
}
```

**Case: Upstream FastAPI sidecar adds its own Server header.**
By default uvicorn/FastAPI emits `Server: uvicorn`. Disable at the application level:

```python
import uvicorn

if __name__ == "__main__":
    uvicorn.run("main:app", host="127.0.0.1", port=9090, server_header=False)
```

Or strip in nginx:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_hide_header Server;
}
```

---

## 8. ALT-SVC (HTTP/3 ADVERTISEMENT)

### 8.1 What It Does

`Alt-Svc` (Alternative Services) advertises that the origin is reachable via a different protocol, port, or host than the one currently in use. Defined in RFC 7838. In 2026 the dominant use case is advertising HTTP/3 over QUIC: the server tells the browser "next time, you can reach me at the same authority using HTTP/3 over UDP".

```
Alt-Svc: h3=":443"; ma=86400
Alt-Svc: h3=":443"; ma=86400, h3-29=":443"; ma=86400
Alt-Svc: clear
```

The browser remembers the alternative for the duration of `ma` (max age, in seconds). On subsequent requests, it attempts the alternative first. If HTTP/3 negotiation succeeds, the entire session uses HTTP/3. If it fails (network blocks UDP, server down, etc), the browser falls back to HTTP/2 and may temporarily distrust the alternative.

### 8.2 Syntax

```
Alt-Svc: <protocol>=<authority>; ma=<seconds> [; persist=1]
Alt-Svc: clear
```

| Field | Meaning |
|---|---|
| `<protocol>` | ALPN protocol identifier. `h3` for HTTP/3, `h2` for HTTP/2 over TLS, `h3-29` for HTTP/3 draft 29 (legacy) |
| `<authority>` | Authority where the alternative is reachable, in the form `[host]:port`. Empty host means "same host as current" |
| `ma=<seconds>` | Max age. How long the browser remembers this alternative |
| `persist=1` | Optional. The browser should retain the alternative across network changes (default is to clear on network change) |
| `clear` | Clears all previously announced alternatives for this origin |

Common patterns:

```
# Advertise HTTP/3 on the same port for 1 day
Alt-Svc: h3=":443"; ma=86400

# Same as above, persistent across network changes
Alt-Svc: h3=":443"; ma=86400; persist=1

# Multiple HTTP/3 versions for compatibility
Alt-Svc: h3=":443"; ma=86400, h3-29=":443"; ma=86400

# Clear all alternatives (useful when disabling HTTP/3)
Alt-Svc: clear
```

### 8.3 The Two Part HTTP/3 Setup

`Alt-Svc` alone does nothing. HTTP/3 requires both:

1. **Nginx configured to actually serve HTTP/3.** Requires nginx 1.25+ built with QUIC support, `listen 443 quic` directive, `http3 on`, and UDP/443 open in firewall.
2. **Alt-Svc header advertising HTTP/3.** Tells browsers it is available.

Without the header, browsers never attempt HTTP/3 even when it is technically available. Without the listen directive, the header advertises something that does not exist and browsers will fall back to HTTP/2 after the first failure.

### 8.4 How To Build It On Bubbles

**Step 1: Ensure nginx supports HTTP/3.**

```bash
nginx -V 2>&1 | grep -o "with-http_v3_module"
# Should output: with-http_v3_module
```

Debian 13 (Bookworm) ships nginx 1.22, which does NOT have HTTP/3. Upgrade options:

```bash
# Use nginx official APT repository for the latest stable
echo "deb https://nginx.org/packages/debian $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
sudo apt update
sudo apt install nginx
```

Verify after upgrade:

```bash
nginx -v
# Expected: nginx version: nginx/1.26.0 or later
nginx -V 2>&1 | grep -o "with-http_v3_module"
# Expected: with-http_v3_module
```

**Step 2: Open UDP/443 in firewall.**

```bash
sudo ufw allow 443/udp
# Or via iptables
sudo iptables -A INPUT -p udp --dport 443 -j ACCEPT
```

**Step 3: Configure nginx for HTTP/3.**

```nginx
server {
    # HTTP/2 over TCP (existing)
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    # HTTP/3 over QUIC (UDP, new)
    listen 443 quic reuseport;
    listen [::]:443 quic reuseport;

    http3 on;

    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # HTTP/3 requires TLS 1.3
    ssl_protocols TLSv1.2 TLSv1.3;

    # 0-RTT for fast reconnects (optional but recommended)
    ssl_early_data on;

    # Advertise HTTP/3 availability
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # ... rest of config ...
}
```

**Critical:** the `listen 443 quic reuseport` directive must appear in **exactly one server block** for that address. If multiple server blocks share the same address with quic, only the first wins; the others fail. The `reuseport` flag is what allows multiple worker processes to share the QUIC socket.

If you have multiple sites on the same nginx instance (which Bubbles does, 27+ custom domains), the `reuseport` flag goes on **one default server** and the rest just `listen 443 quic`:

```nginx
# Default server (gets reuseport)
server {
    listen 443 quic reuseport default_server;
    listen [::]:443 quic reuseport default_server;
    # ... default config ...
}

# All other sites
server {
    listen 443 quic;
    listen [::]:443 quic;
    server_name example.com;
    # ... site config ...
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

**Step 4: Reload and verify.**

```bash
nginx -t && systemctl reload nginx
```

### 8.5 Considerations For The Bubbles Wildcard

`thatwebhostingguy.com` serves 119+ wildcard subdomains from a single server block. HTTP/3 with `reuseport` works for the entire instance; the Alt-Svc header should be added to the wildcard server too:

```nginx
server {
    listen 443 ssl;
    listen 443 quic reuseport;
    http2 on;
    http3 on;
    server_name *.thatwebhostingguy.com thatwebhostingguy.com;

    # ... SSL config, AdSense injection, etc ...

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

All 119 wildcard demos get HTTP/3 advertised. The browser learns it per origin (each subdomain is its own origin for HTTP/3 purposes), but the underlying nginx serves them all.

### 8.6 How To Verify

```bash
# 1. Confirm Alt-Svc header is sent
curl -sI https://example.com/ | grep -i alt-svc
# Expected: alt-svc: h3=":443"; ma=86400

# 2. Verify nginx is configured for HTTP/3
nginx -T 2>/dev/null | grep -E "http3 on|listen.*quic"

# 3. Test actual HTTP/3 connection
curl --http3 -sI https://example.com/
# Expected: HTTP/3 200
# If curl was built without HTTP/3 support: error

# 4. Check UDP/443 is open and listening
sudo ss -lunpt | grep ":443"
# Expected: line showing UDP port 443 bound to nginx

# 5. Browser test
# Open Chrome DevTools, Network panel, click any request
# Look at "Protocol" column
# Expected: "h3" (or "h3-29" for older draft)
# If "h2" or "h2+quic-99": HTTP/3 not negotiating

# 6. Chrome internal HTTP/3 status
# Navigate to: chrome://net-internals/#alt-svc
# Search for example.com
# Should show: alternative h3 with positive remaining time

# 7. Use external tester
echo "Visit: https://http3check.net/?host=example.com"
# Or: https://www.cdn77.com/http3-test
```

### 8.7 Troubleshooting

**Symptom: Alt-Svc header sent but browser still uses HTTP/2.**
Causes ranked by frequency:
1. UDP/443 is blocked somewhere between server and client. Verify firewall on the server. Some corporate networks block all UDP.
2. nginx is not actually serving HTTP/3 despite the listen directive. Run `nginx -T | grep "quic"` and ensure `listen 443 quic` appears.
3. HTTP/3 connection failed once and the browser is in fallback. Clear browser cache and Alt-Svc state, or test from a fresh browser profile.
4. The header value is malformed. Verify exact spelling, no extra spaces.

**Symptom: HTTP/3 works in curl but not in Chrome.**
1. Browser HTTP/3 might be disabled by flag. Check `chrome://flags/#enable-quic`.
2. Browser is using HTTP/3 but Network tab labels are confusing. Verify by `chrome://net-internals/#alt-svc`.
3. Browser cached a previous failed HTTP/3 attempt. Clear browsing data including network state.

**Symptom: nginx fails to start after adding quic listen.**
1. nginx version is too old. Requires 1.25+.
2. nginx not compiled with HTTP/3 support. Run `nginx -V 2>&1 | grep http_v3`.
3. Conflicting `reuseport` declarations. Only one server should have `reuseport` per address.
4. UDP port already in use. `sudo ss -lunpt | grep ":443"` to identify the process.

**Symptom: HTTP/3 works but performance is worse than HTTP/2.**
1. Server is single CPU, QUIC is more CPU intensive than HTTP/2. Verify load.
2. Connection migration is happening (mobile networks switching). Set `quic_retry on` for stability.
3. Initial connection takes longer than HTTP/2 because of QUIC handshake overhead. Subsequent requests should be faster. Measure across multiple requests, not just one.

**Symptom: Alt-Svc clearing on network change.**
Default behavior. Add `persist=1` if needed:

```
Alt-Svc: h3=":443"; ma=86400; persist=1
```

Mobile users switching between Wi-Fi and cellular benefit from persistence.

### 8.8 How To Fix Common Breakage

**Case: HTTP/3 enabled in nginx but no measurable HTTP/3 traffic.**
Most likely the Alt-Svc header is missing. Add to every server block:

```nginx
add_header Alt-Svc 'h3=":443"; ma=86400' always;
```

Reload nginx. Wait for browsers to discover the alternative on their next request. Within an hour you should see HTTP/3 in access logs.

**Case: Access log does not distinguish HTTP/3 from HTTP/2.**
Add the protocol to the log format:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" '
                '$server_protocol "$http3"';

access_log /var/log/nginx/access.log main;
```

Now you can `grep "h3" /var/log/nginx/access.log` to count HTTP/3 traffic.

**Case: Need to disable HTTP/3 quickly (emergency rollback).**
Three options ranked by speed:

```nginx
# Fastest: clear the Alt-Svc advertisement so new clients do not try
add_header Alt-Svc 'clear' always;

# Or remove the quic listen
# listen 443 quic reuseport;  # commented out

# Or block UDP/443 at firewall
sudo ufw deny 443/udp
```

`Alt-Svc: clear` is fastest because it propagates as browsers next visit. Within a day, all clients will have cleared their HTTP/3 state.

**Case: nginx version too old for HTTP/3.**
Use official nginx APT repository for the latest stable. See Step 1 in Section 8.4.

---

## 9. HOW THESE HEADERS INTERACT

The four headers form a coherent performance picture. Specific interactions matter.

### 9.1 Server-Timing And Timing-Allow-Origin

`Server-Timing` is exposed to JavaScript on the page via `PerformanceServerTiming`. For same origin responses, the data is fully available. For cross origin responses, the same Timing-Allow-Origin rule applies: without TAO, the `serverTiming` attribute on PerformanceResourceTiming returns an empty array.

A CDN that wants its Server-Timing exposed to monitoring on the consuming origin must set BOTH:

```
Server-Timing: cdn-cache;desc=hit
Timing-Allow-Origin: https://example.com
```

For Bubbles same origin case: no TAO needed; Server-Timing flows through.

### 9.2 Server And Security Headers

The Server header is a fingerprint. The other security headers in framework-http-security-headers.md (HSTS, CSP, etc) provide functional security. Suppressing the Server header reduces fingerprinting but does nothing for real attackers; the security headers do the actual work.

Both are needed: suppress Server for procurement audits and pen test reports, set real security headers for real protection.

### 9.3 Alt-Svc And HSTS

HSTS forces HTTPS. Alt-Svc upgrades within HTTPS to HTTP/3. They are independent: a site can have HSTS without HTTP/3, or HTTP/3 without HSTS (rare, since HTTP/3 requires TLS).

Order of negotiation:

1. Browser requests `http://example.com/` (or types `example.com` and browser tries HTTP first).
2. HSTS rule (cached from a previous visit, or in preload list) upgrades to `https://`.
3. TLS handshake. ALPN negotiates HTTP/2.
4. Response includes `Alt-Svc: h3=":443"`. Browser stores for future use.
5. Next visit: browser tries HTTP/3 over QUIC first. Falls back to HTTP/2 on failure.

### 9.4 Alt-Svc And Cache-Control

The Alt-Svc header is processed independently of caching. A 304 Not Modified response does not need to include Alt-Svc because the browser already has it cached. Browsers may also cache the Alt-Svc value beyond the cache lifetime of the response, governed by the `ma` parameter.

### 9.5 Server-Timing And Privacy

Server-Timing values are visible to any script on the page. Third party scripts (analytics, ads, widgets) can read them. Decide what is acceptable to expose.

Bubbles policy: expose only generic phase names (`db`, `render`, `cache`, `total`) without sensitive descriptions. The total request time and rough breakdown is enough for RUM purposes and reveals nothing exploitable.

---

## 10. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks for the most common scenarios.

### 10.1 Standard Bubbles HTTP/3 + Server header suppression

```nginx
# /etc/nginx/nginx.conf
http {
    server_tokens off;
    # Optional: completely remove Server header (requires headers-more module)
    # more_clear_headers Server;
}

# Per server block
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    listen 443 quic reuseport;  # only one server gets reuseport
    listen [::]:443 quic reuseport;
    http3 on;

    server_name example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

### 10.2 Dynamic page from FastAPI sidecar with Server-Timing

Sidecar Python:

```python
@app.get("/dashboard")
async def dashboard(request: Request):
    timer = Timer()  # see Section 5.4

    with timer.phase("auth"):
        user = await authenticate(request)

    with timer.phase("db", "user data"):
        data = await fetch_dashboard_data(user)

    with timer.phase("render"):
        html = render_template("dashboard.html", data=data)

    return HTMLResponse(
        content=html,
        headers={"Server-Timing": timer.header_value()}
    )
```

Nginx:

```nginx
location /dashboard {
    proxy_pass http://127.0.0.1:9090;
    # Server-Timing passes through automatically
}
```

### 10.3 Static asset on a CDN subdomain (rare on Bubbles)

```nginx
server {
    server_name cdn.example.com;

    location / {
        add_header Timing-Allow-Origin "https://example.com, https://www.example.com" always;
        add_header Access-Control-Allow-Origin "https://example.com" always;
        # Plus normal caching, content type, etc
    }
}
```

### 10.4 Public embeddable widget (TAO for any consumer)

```nginx
server {
    server_name widget.example.com;

    location / {
        add_header Timing-Allow-Origin "*" always;
        add_header Access-Control-Allow-Origin "*" always;
    }
}
```

### 10.5 Maximum opacity Server suppression for high value site

```nginx
http {
    server_tokens off;
    more_clear_headers Server;
    more_clear_headers "X-Powered-By";
    more_clear_headers "X-AspNet-Version";
}

server {
    # ... config ...
    # Custom error pages without nginx mention
    error_page 404 /errors/404.html;
    error_page 500 502 503 504 /errors/50x.html;
}
```

### 10.6 Vanity Server header for branding

```nginx
http {
    server_tokens off;
    more_set_headers "Server: Bubbles";
}
```

Every response shows `Server: Bubbles` regardless of underlying nginx version.

### 10.7 thatwebhostingguy.com wildcard with HTTP/3

```nginx
server {
    listen 443 ssl;
    listen 443 quic reuseport default_server;  # reuseport on the wildcard
    listen [::]:443 ssl default_server;
    listen [::]:443 quic reuseport default_server;
    http2 on;
    http3 on;

    server_name *.thatwebhostingguy.com thatwebhostingguy.com;

    ssl_certificate /etc/letsencrypt/live/thatwebhostingguy.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/thatwebhostingguy.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

All 119 wildcards get HTTP/3 advertised. The `default_server` plus `reuseport` flag here means this server block owns the QUIC socket.

### 10.8 Per client custom domain on Bubbles (no reuseport since wildcard has it)

```nginx
server {
    listen 443 ssl;
    listen 443 quic;  # no reuseport; the wildcard server has it
    listen [::]:443 ssl;
    listen [::]:443 quic;
    http2 on;
    http3 on;

    server_name client-domain.com www.client-domain.com;

    ssl_certificate /etc/letsencrypt/live/client-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/client-domain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

### 10.9 RUM beacon endpoint receiving Server-Timing data

FastAPI sidecar:

```python
from fastapi import FastAPI, Request
from fastapi.responses import Response
import json
import time

app = FastAPI()

@app.post("/rum")
async def rum_beacon(request: Request):
    body = await request.body()
    try:
        data = json.loads(body)
        # Write to monitoring backend
        # Example: append to JSONL file
        with open("/var/log/rum.jsonl", "a") as f:
            f.write(json.dumps({
                "timestamp": time.time(),
                "remote": request.client.host,
                **data
            }) + "\n")
    except Exception as e:
        pass  # Never fail a beacon
    return Response(status_code=204)
```

Nginx:

```nginx
location = /rum {
    proxy_pass http://127.0.0.1:9090;
    # Allow large beacons
    client_max_body_size 16k;
    # Do not cache
    add_header Cache-Control "no-store" always;
}
```

### 10.10 Log format including HTTP/3 indicator and request_time

```nginx
log_format perf '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" '
                '$server_protocol "$http3" '
                'rt=$request_time uct=$upstream_connect_time '
                'urt=$upstream_response_time';

access_log /var/log/nginx/access.log perf;
```

`$http3` is empty when HTTP/2 is used and "h3" when HTTP/3 is used. `$request_time` is the full nginx processing time. `$upstream_response_time` is the FastAPI sidecar wait time.

---

## 11. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete performance header stanza, layered with the previous four frameworks.

```nginx
# /etc/nginx/nginx.conf
http {
    # Reduce fingerprinting
    server_tokens off;

    # Optional: full Server header suppression (requires headers-more module)
    # more_clear_headers Server;

    # Performance oriented log format
    log_format perf '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$server_protocol "$http3" '
                    'rt=$request_time uct=$upstream_connect_time '
                    'urt=$upstream_response_time';

    access_log /var/log/nginx/access.log perf;
}
```

```nginx
# /etc/nginx/sites-available/example.com

# HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

# www to non www (note: no quic on this; only the canonical server gets quic)
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    return 301 https://example.com$request_uri;
}

# Canonical
server {
    # HTTP/2 over TCP
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    # HTTP/3 over QUIC
    listen 443 quic;
    listen [::]:443 quic;
    http3 on;

    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    root /var/www/sites/example.com;
    index index.html;

    # Advertise HTTP/3
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # Pull in shared security headers
    include snippets/common-security-headers.conf;

    # ===== HTML PAGES =====
    location ~* \.html$ {
        include snippets/common-security-headers.conf;
        add_header Alt-Svc 'h3=":443"; ma=86400' always;
        # Per site CSP (from framework-http-security-headers.md)
        # Per site Content-Language, Cache-Control, X-Robots-Tag (from other frameworks)
    }

    # ===== STATIC ASSETS (same origin, no TAO needed) =====
    location ~* \.(css|js|woff2|jpg|jpeg|png|webp|avif|svg)$ {
        include snippets/common-security-headers.conf;
        add_header Cross-Origin-Resource-Policy "same-site" always;
        # No Timing-Allow-Origin needed since assets are same origin
    }

    # ===== API ENDPOINTS (FastAPI sidecar emits Server-Timing) =====
    location /api/ {
        include snippets/common-security-headers.conf;
        add_header X-Robots-Tag "noindex, nofollow" always;
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # Server-Timing passes through automatically
    }

    # ===== RUM BEACON =====
    location = /rum {
        proxy_pass http://127.0.0.1:9090;
        client_max_body_size 16k;
        add_header Cache-Control "no-store" always;
    }

    # ===== ROOT =====
    location / {
        include snippets/common-security-headers.conf;
        try_files $uri $uri/ $uri.html =404;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify:

```bash
# Server header reduced
curl -sI https://example.com/ | grep -i "^server:"

# Alt-Svc set
curl -sI https://example.com/ | grep -i alt-svc

# HTTP/3 negotiable
curl --http3 -sI https://example.com/ | head -1
```

---

## 12. AUDIT CHECKLIST

Run through these 50 items for any production site.

### Server-Timing

1. [ ] Dynamic responses from FastAPI sidecar include `Server-Timing` header.
2. [ ] Server-Timing values do not expose sensitive data (file paths, queries, internal service names).
3. [ ] Metric names are short and consistent across endpoints.
4. [ ] `total` metric is exposed for at least one comprehensive duration.
5. [ ] Static assets are NOT decorated with Server-Timing (no application timing applies).
6. [ ] Server-Timing values appear in browser DevTools Network panel timing section.
7. [ ] `performance.getEntriesByType("navigation")[0].serverTiming` returns an array in the console.
8. [ ] RUM beacon captures Server-Timing values and ships to monitoring.

### Timing-Allow-Origin

9. [ ] If assets are on a cross origin (CDN subdomain), TAO is set appropriately.
10. [ ] Same origin assets do NOT have TAO (it is unnecessary noise).
11. [ ] Cross origin partner widgets have TAO matching the consuming origin or `*`.
12. [ ] Resource Timing data in RUM shows non zero transferSize for monitored resources.
13. [ ] If using third party fonts or scripts, blind spots are documented and accepted.

### Server

14. [ ] `server_tokens off` is set in `nginx.conf` at http level.
15. [ ] Server header value is `nginx` (no version) at minimum.
16. [ ] If maximum opacity required, Server header is removed entirely via headers-more.
17. [ ] No `X-Powered-By` or similar disclosure headers.
18. [ ] Custom error pages do not mention nginx version.
19. [ ] Scanner does not flag version disclosure (Invicti, Acunetix, Qualys, etc).
20. [ ] FastAPI sidecar does not emit its own `Server: uvicorn` to the client.

### Alt-Svc

21. [ ] `Alt-Svc: h3=":443"; ma=86400` header sent on every response.
22. [ ] nginx version 1.25+ confirmed (run `nginx -v`).
23. [ ] nginx built with HTTP/3 module (run `nginx -V 2>&1 | grep http_v3`).
24. [ ] `listen 443 quic` directive present in canonical server block.
25. [ ] `http3 on;` directive present.
26. [ ] UDP/443 open in firewall.
27. [ ] `ssl_protocols TLSv1.2 TLSv1.3` (HTTP/3 requires 1.3).
28. [ ] `ssl_early_data on` set (for 0-RTT, optional but recommended).
29. [ ] Exactly one server has `reuseport` on its quic listen (default server).
30. [ ] Other servers have plain `listen 443 quic` (no reuseport).
31. [ ] `curl --http3 https://example.com/` negotiates HTTP/3 successfully.
32. [ ] Browser DevTools Network panel shows `h3` protocol for repeat visits.
33. [ ] Access log includes HTTP/3 indicator (`$http3` variable).
34. [ ] HTTP/3 traffic percentage in logs is non zero (verifies real world adoption).

### Cross cutting

35. [ ] `nginx -t` passes without warnings.
36. [ ] `nginx -T` shows all expected directives in effect.
37. [ ] HTTP/3 works from external networks (not just localhost).
38. [ ] Server-Timing visible in browser PerformanceServerTiming.
39. [ ] Resource Timing populates for same origin assets.
40. [ ] TTFB measurements correlate with Server-Timing total.
41. [ ] Alt-Svc value uses ma=86400 or longer (not the absent default).
42. [ ] No protocol downgrade attempts (HTTP/3 stays sticky once negotiated).
43. [ ] Access log has both HTTP/2 and HTTP/3 entries (mix is healthy).
44. [ ] No errors in nginx error log related to QUIC.
45. [ ] Performance monitoring backend ingests Server-Timing values correctly.
46. [ ] Server header consistent across all server blocks and locations.
47. [ ] Alt-Svc header consistent across all server blocks.
48. [ ] No accidental Server header reintroduction by FastAPI sidecar or other upstream.
49. [ ] Documentation lists which third party scripts are timing blind (no TAO).
50. [ ] Quarterly review confirms HTTP/3 adoption rate and Server-Timing coverage.

A site that passes all 50 has correctly configured performance observability and protocol advertisement.

---

## 13. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: HTTP/3 enabled in nginx but no Alt-Svc header set.**
Symptom: zero HTTP/3 traffic in access logs despite valid nginx config.
Why it breaks: browsers never attempt HTTP/3 without Alt-Svc telling them it is available.
Fix: add `Alt-Svc 'h3=":443"; ma=86400'` to every server block.

**Pitfall 2: Multiple servers with `reuseport` on quic listen.**
Symptom: nginx fails to start with EADDRINUSE on UDP/443.
Why it breaks: only one server block per address may declare `reuseport`.
Fix: one server (typically the default) has `reuseport`; all others have plain `listen 443 quic`.

**Pitfall 3: Server-Timing exposing sensitive details.**
Symptom: pen tester reports information disclosure via Server-Timing.
Why it breaks: `desc="users.lookup_by_email"` reveals database schema.
Fix: restrict `desc=` values to generic identifiers, or omit `desc` entirely.

**Pitfall 4: Timing-Allow-Origin set to `*` on a private internal resource.**
Symptom: scanner reports overly permissive TAO.
Why it breaks: any origin can profile timing of an internal resource.
Fix: restrict to specific origins, or remove TAO if not needed (defaults are restrictive enough).

**Pitfall 5: server_tokens off but Server header still leaking version.**
Symptom: scanner flags version after applying the directive.
Why it breaks: nginx was not reloaded, or directive is in wrong context.
Fix: `nginx -t && systemctl reload nginx`. Verify with curl.

**Pitfall 6: Vanity Server header that looks suspicious.**
Symptom: `Server: hacker-mcgee` triggers WAF rules at downstream proxies.
Why it breaks: some WAFs and security tools flag unusual Server values.
Fix: use a plausible looking value (`Server: web`, `Server: bubbles`, or just remove entirely).

**Pitfall 7: HTTP/3 advertised but UDP blocked.**
Symptom: browsers try HTTP/3, fail, fall back to HTTP/2 every time.
Why it breaks: firewall or upstream router blocks UDP/443.
Fix: open UDP/443 (`sudo ufw allow 443/udp`). Verify with `nc -u -l 443` and external test.

**Pitfall 8: nginx version too old for HTTP/3.**
Symptom: `listen 443 quic` rejected with "unknown directive".
Why it breaks: nginx versions before 1.25 lack HTTP/3 module.
Fix: install from official nginx APT repository to get 1.26+.

**Pitfall 9: Cross origin assets blind in RUM without TAO.**
Symptom: dashboard shows transferSize=0 for half the resources.
Why it breaks: cross origin browser policy zeroes detailed timing without TAO opt in.
Fix: self host the assets (preferred on Bubbles), or coordinate with the third party to add TAO.

**Pitfall 10: Server-Timing not visible in browser.**
Symptom: curl shows the header but browser Network panel does not list "Server Timing" section.
Why it breaks: typically the response is cross origin and Timing-Allow-Origin is missing.
Fix: set TAO on cross origin endpoints that emit Server-Timing.

**Pitfall 11: Alt-Svc with ma=0 or ma not set.**
Symptom: HTTP/3 negotiates once, then browser forgets.
Why it breaks: default ma is unspecified; some browsers treat as 0.
Fix: always include `ma=86400` (1 day) or longer.

**Pitfall 12: HTTP/3 works in curl but Chrome refuses.**
Symptom: `curl --http3 -sI https://example.com/` succeeds; Chrome stays on HTTP/2.
Why it breaks: Chrome had a failed HTTP/3 attempt and is in cooldown, OR Chrome flag disabled HTTP/3.
Fix: clear browsing data including network state. Check `chrome://flags/#enable-quic`.

**Pitfall 13: Server-Timing on every request, including for static assets.**
Symptom: log volume explosion; thousands of empty Server-Timing entries.
Why it breaks: the upstream is generating timing for every request including ones where nothing meaningful happens.
Fix: emit Server-Timing only on dynamic responses. Static asset locations should not invoke the upstream.

**Pitfall 14: Server header reintroduced by FastAPI sidecar.**
Symptom: nginx server_tokens off applied, but client still sees `Server: uvicorn`.
Why it breaks: when nginx proxies, it passes through the upstream's Server header.
Fix: configure uvicorn with `server_header=False`, or strip in nginx with `proxy_hide_header Server`.

**Pitfall 15: Alt-Svc but no nginx HTTP/3 actually configured.**
Symptom: Alt-Svc header set, but HTTP/3 fails on first attempt. Browsers stop trying.
Why it breaks: advertising a non existent alternative.
Fix: either add the nginx http3 listen directive, or remove the Alt-Svc header until ready.

---

## 14. DIAGNOSTIC COMMANDS

Reference of every command useful for performance header investigation.

### Inspect all performance headers

```bash
# Full performance family
curl -sI https://example.com/ | grep -iE "^(server-timing|timing-allow-origin|server|alt-svc):"

# Including via HTTP/3 if available
curl --http3 -sI https://example.com/ | grep -iE "^(server-timing|timing-allow-origin|server|alt-svc):"

# As different user agents (different Server-Timing may be exposed)
for ua in "Googlebot" "Mozilla/5.0 Chrome" "curl"; do
    echo "=== $ua ==="
    curl -sI -A "$ua" https://example.com/ | grep -i server-timing
done
```

### Server-Timing verification

```bash
# Header presence
curl -sI https://example.com/page | grep -i server-timing

# Parse and display
curl -sI https://example.com/page | grep -i server-timing | tr ',' '\n' | sed 's/^ */    /'

# Verify upstream emits it
curl -sI http://127.0.0.1:9090/page | grep -i server-timing

# Browser DevTools console:
# performance.getEntriesByType("navigation")[0].serverTiming
# performance.getEntriesByType("resource")[0].serverTiming

# Sanity check: total dur matches observed
TOTAL=$(curl -sI https://example.com/page | grep -i server-timing | grep -oE "total;dur=[0-9.]+" | cut -d= -f2)
OBSERVED=$(curl -so /dev/null -w "%{time_starttransfer}" https://example.com/page)
echo "Server reports: $TOTAL ms"
echo "Client observed: $(echo "$OBSERVED * 1000" | bc) ms"
```

### Timing-Allow-Origin verification

```bash
# Header presence on cross origin asset
curl -sI https://cdn.example.com/main.css | grep -i timing-allow-origin

# Browser DevTools console:
# const r = performance.getEntriesByName("https://cdn.example.com/main.css")[0];
# console.log({transferSize: r.transferSize, responseStart: r.responseStart});
# Expected: real numbers with TAO; zeros without

# Audit which resources are missing timing
# Open DevTools, run:
# performance.getEntriesByType("resource").filter(r => r.transferSize === 0 && r.duration > 0)
```

### Server header verification

```bash
# Current value
curl -sI https://example.com/ | grep -i "^server:"

# Across all error states
for status in 200 404 500; do
    curl -sIo /dev/null -w "Status: %{http_code} Server: " "https://example.com/test-status-$status"
    curl -sI "https://example.com/test-status-$status" | grep -i "^server:" | cut -d: -f2-
done

# Check error pages
curl -s https://example.com/this-does-not-exist | grep -i nginx

# Confirm server_tokens setting in active config
nginx -T 2>/dev/null | grep -i server_tokens

# Check headers-more is loaded if used
nginx -V 2>&1 | grep -o headers-more

# Sanity check: no Server header from upstream sneaking through
curl -sI http://127.0.0.1:9090/api/anything | grep -i "^server:"
```

### Alt-Svc and HTTP/3 verification

```bash
# Header presence
curl -sI https://example.com/ | grep -i alt-svc

# Test HTTP/3 with curl (requires curl built with HTTP/3 support)
curl --http3 -sI https://example.com/
# Look for: HTTP/3 200

# Verify nginx config
nginx -T 2>/dev/null | grep -E "http3 on|listen.*quic|reuseport"

# Check UDP/443 listening
sudo ss -lunpt | grep ":443"

# Check firewall
sudo ufw status | grep 443
sudo iptables -L INPUT -n | grep 443

# External tester
echo "Visit: https://http3check.net/?host=example.com"

# Browser:
# DevTools > Network > any request > Protocol column should show "h3"
# chrome://net-internals/#alt-svc > search example.com

# Count HTTP/3 traffic in logs (requires $http3 in log_format)
grep "h3" /var/log/nginx/access.log | wc -l
grep -c "\"h3\"" /var/log/nginx/access.log
echo "HTTP/3 vs HTTP/2 ratio:"
awk '{print $NF}' /var/log/nginx/access.log | sort | uniq -c
```

### Server side investigation

```bash
# Active nginx config
nginx -T 2>/dev/null > /tmp/nginx-active.conf

# Find performance directives
grep -E "server_tokens|http3|quic|early_data" /tmp/nginx-active.conf

# Find all add_header directives related to performance
grep -A1 "add_header" /tmp/nginx-active.conf | grep -iE "alt-svc|server-timing|timing-allow"

# Check uvicorn configuration
ps auxww | grep uvicorn | grep -oE "server.header[a-z]*"

# Apply changes
nginx -t && systemctl reload nginx
```

### Browser DevTools quick reference

In Chrome DevTools Network panel:

* **Protocol column**: shows `h3`, `h2`, or `http/1.1`. Add via column header right click if not visible.
* **Timing tab on each request**: shows the timeline including any Server-Timing metrics.
* **Console PerformanceObserver test:**

```javascript
// Show navigation timing including serverTiming
const nav = performance.getEntriesByType("navigation")[0];
console.log({
    ttfb: nav.responseStart - nav.requestStart,
    serverTiming: nav.serverTiming
});

// Show all resources with their timing
performance.getEntriesByType("resource").forEach(r => {
    console.log(r.name, {
        protocol: r.nextHopProtocol,
        transferSize: r.transferSize,
        duration: r.duration
    });
});
```

* **Network panel filter**: type `protocol:h3` to show only HTTP/3 requests.

---

## 15. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): Cache-Control, ETag, Last-Modified, Expires, Vary, Age. Caching directives interact with Server-Timing because cached responses do not pass through the upstream's timing instrumentation.
* [framework-http-content-headers.md](framework-http-content-headers.md): Content-Type, Content-Language, Content-Encoding, Content-Length, Content-Disposition. Content-Encoding (compression) affects Resource Timing transfer size measurements.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): X-Robots-Tag, Link, Location. The Link header preload + Early Hints (103) pairs with HTTP/3 advertisement for the fastest possible TTFB optimization.
* [framework-http-security-headers.md](framework-http-security-headers.md): HSTS, CSP, X-Frame-Options, etc. Server suppression pairs with the broader security header stack for fingerprint reduction. HTTP/3 advertisement (Alt-Svc) layers on top of HSTS for transport security plus performance.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Performance headers contribute to Core Web Vitals.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook.
* [framework-corewebvitals.md](framework-corewebvitals.md): LCP, INP, CLS targets. Server-Timing and HTTP/3 directly improve TTFB which improves LCP.
* [framework-rum.md](framework-rum.md): Real User Monitoring implementation. Companion to Server-Timing and Timing-Allow-Origin.
* [framework-http3-quic.md](framework-http3-quic.md): deeper dive on HTTP/3, QUIC, 0-RTT, connection migration.
* Server Timing W3C spec: https://www.w3.org/TR/server-timing/
* Resource Timing W3C spec: https://www.w3.org/TR/resource-timing/
* MDN Server-Timing: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Server-Timing
* MDN Timing-Allow-Origin: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Timing-Allow-Origin
* MDN Alt-Svc: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Alt-Svc
* Nginx HTTP/3 documentation: https://nginx.org/en/docs/http/ngx_http_v3_module.html
* RFC 9114 (HTTP/3): https://www.rfc-editor.org/rfc/rfc9114
* RFC 7838 (HTTP Alternative Services): https://www.rfc-editor.org/rfc/rfc7838
* RFC 9110 (HTTP Semantics, Server header): https://www.rfc-editor.org/rfc/rfc9110
* HTTP/3 check: https://http3check.net/

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Bubbles default performance configuration

```nginx
# http context
http {
    server_tokens off;
    log_format perf '$remote_addr - [$time_local] "$request" $status $body_bytes_sent $server_protocol "$http3" rt=$request_time urt=$upstream_response_time';
    access_log /var/log/nginx/access.log perf;
}

# Per server block
server {
    listen 443 ssl;
    listen 443 quic;
    http2 on;
    http3 on;

    # On the default server only:
    # listen 443 quic reuseport;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

### Header purpose table

| Header | One line purpose |
|---|---|
| Server-Timing | Expose backend phase durations to browser RUM |
| Timing-Allow-Origin | Grant cross origin access to Resource Timing |
| Server | Identify server software (suppress for fingerprint reduction) |
| Alt-Svc | Advertise HTTP/3 availability |

### Server-Timing format

```
Server-Timing: name;dur=ms[;desc="text"], next;dur=ms
```

### Timing-Allow-Origin values

| Value | Use case |
|---|---|
| `*` | Public embeddable resource |
| `https://example.com` | Specific consuming origin |
| (absent) | Same origin only (default) |

### Server suppression levels

| Level | Method |
|---|---|
| Version hidden | `server_tokens off` |
| Header removed | `more_clear_headers Server` (headers-more) |
| Vanity replacement | `more_set_headers "Server: Bubbles"` |

### Alt-Svc minimum value

```
Alt-Svc: h3=":443"; ma=86400
```

### Five commands every operator should know

```bash
# 1. View all performance headers
curl -sI https://example.com/ | grep -iE "^(server-timing|timing-allow-origin|server|alt-svc):"

# 2. Test HTTP/3 negotiation
curl --http3 -sI https://example.com/

# 3. Check Server-Timing in browser console
# performance.getEntriesByType("navigation")[0].serverTiming

# 4. Count HTTP/3 traffic share
awk '{print $NF}' /var/log/nginx/access.log | sort | uniq -c

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Server header is minimized
curl -sI https://example.com/ | grep -i "^server:"
# Expected: server: nginx (not nginx/1.x.x)

# 2. HTTP/3 is negotiable
curl --http3 -sI https://example.com/ | head -1
# Expected: HTTP/3 200

# 3. Alt-Svc is set
curl -sI https://example.com/ | grep -i alt-svc
# Expected: alt-svc: h3=":443"; ma=86400
```

If all three produce expected output AND access logs show non zero HTTP/3 traffic share AND RUM dashboard shows populated Server-Timing values, the performance header stack is correctly wired.

---

End of framework-http-performance-headers.md.
