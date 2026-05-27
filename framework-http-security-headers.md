# framework-http-security-headers.md

Comprehensive reference for the nine HTTP response headers that establish security guarantees and trust boundaries for a web origin: `Strict-Transport-Security` (HSTS) for transport, `Content-Security-Policy` (CSP) for resource and script control, `X-Frame-Options` for clickjacking defense, `X-Content-Type-Options` for MIME sniffing prevention, `Referrer-Policy` for outbound URL privacy, `Permissions-Policy` for browser capability control, and the cross origin isolation trio `Cross-Origin-Resource-Policy` (CORP), `Cross-Origin-Embedder-Policy` (COEP), and `Cross-Origin-Opener-Policy` (COOP). Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to framework-http-caching-headers.md, framework-http-content-headers.md, framework-http-seo-headers.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx, AI assistants generating or repairing nginx config, security auditors, and anyone troubleshooting "CSP blocking my GTM scripts", "site permanently broken after HSTS preload", "clickjacking warning from scanner", or "SharedArrayBuffer not available" anomalies on a self hosted stack.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Defense In Depth Mental Model (read this first)
5. Strict-Transport-Security (HSTS, transport security)
6. Content-Security-Policy (the biggest lever, script and resource control)
7. X-Frame-Options (clickjacking defense, mostly superseded by CSP frame-ancestors)
8. X-Content-Type-Options (MIME sniffing prevention)
9. Referrer-Policy (outbound URL privacy)
10. Permissions-Policy (browser capability control)
11. Cross-Origin-Resource-Policy (CORP, who can load this resource)
12. Cross-Origin-Embedder-Policy (COEP, what this document can embed)
13. Cross-Origin-Opener-Policy (COOP, what this document can share a window with)
14. The Cross Origin Isolation Pattern (COEP plus COOP plus CORP)
15. How These Headers Interact
16. Asset Class And Use Case Recipes
17. Bubbles Nginx Reference Block (paste ready)
18. Audit Checklist (60+ items)
19. Common Pitfalls
20. Diagnostic Commands (curl, observatory.mozilla.org, securityheaders.com, browser devtools)
21. Cross-References

---

## 1. DEFINITION

Security and trust signal headers tell the browser what it may safely do with a response, what other origins may interact with it, what scripts may execute on it, what capabilities the page may use, and how transport must be secured. They operate as defense in depth: each one closes off a class of attack, and missing any one creates a gap that scanners and attackers will find. The nine headers split into four concerns:

* **Transport**: `Strict-Transport-Security`. It answers "must this origin always be accessed over HTTPS?"
* **Content execution and embedding**: `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`. They answer "what scripts may run, what frames may embed this page, may the browser sniff a different type than declared?"
* **Privacy and capability**: `Referrer-Policy`, `Permissions-Policy`. They answer "how much referrer information leaks, which device APIs may this page use?"
* **Cross origin isolation**: `Cross-Origin-Resource-Policy`, `Cross-Origin-Embedder-Policy`, `Cross-Origin-Opener-Policy`. They answer "who may load this resource, what may this document embed, what may share a window with this document?"

Together these nine headers determine the security posture of an origin. Getting any one wrong opens a class of attack: clickjacking, XSS, MIME confusion, referrer leaks, capability abuse, Spectre style side channel attacks, or downgrade to HTTP. Getting them all right earns an A or A+ rating on observatory.mozilla.org and securityheaders.com, which several enterprise procurement processes now require for vendor approval.

---

## 2. WHY IT MATTERS

Six independent pressures push correct security headers from "polish" to "required infrastructure" in 2025 and forward.

**Enterprise procurement now scans for security headers.** Mid to large enterprises evaluating vendor websites routinely check securityheaders.com and observatory.mozilla.org as part of vendor onboarding. A grade of D or F is a fast path to elimination. An A or A+ is table stakes for credibility in any B2B context. SOC 2 audits, ISO 27001 reviews, and HIPAA risk assessments all include security header checks.

**Search engines and AI crawlers prefer secure origins.** Google has confirmed HTTPS as a ranking signal since 2014 and progressively penalized mixed content. Bing, Perplexity, ClaudeBot, and GPTBot all crawl HTTPS preferentially and treat HTTP origins with suspicion. An origin without HSTS, especially without HSTS preload, signals operational immaturity to ranking systems.

**Browser features are gated on security headers.** Modern web APIs (SharedArrayBuffer, high precision timers, WebAssembly threading, OPFS write access) require cross origin isolation, which requires COEP plus COOP. Sites that need these features (WebAssembly games, video editors, scientific computing apps) cannot function without the right headers.

**XSS is still the number one web vulnerability.** Every OWASP Top 10 list since 2007 has included XSS or its variants. CSP is the single most effective mitigation against the entire class. A site without CSP is one stored XSS away from credential theft, session hijacking, or arbitrary code execution in every visitor's browser.

**Clickjacking is trivial without X-Frame-Options or CSP frame-ancestors.** Any site that can be embedded in an iframe can be clickjacked. The attack overlays the target site in an invisible frame and tricks users into clicking elements they cannot see. Defense is a single header. Failure to set it has caused real money losses (cryptocurrency wallet drains, account takeovers).

**Spectre and similar side channel attacks broke the assumption that cross origin data is opaque.** Before 2018, browsers assumed that an attacker on `evil.com` could not read the pixel data of an image loaded from `bank.com`. Spectre invalidated that assumption. The fix is the cross origin isolation pattern (COEP plus COOP plus CORP), which separates processes so a malicious page cannot reach a victim page's memory via timing attacks.

**Cost of getting it wrong.** Misconfigured security headers produce loud security failures and silent ranking failures. Real examples:

* HSTS preload submitted without testing all subdomains. The internal `admin.example.com` subdomain only served HTTP. After preload, all employee admins lost access for the six months it took to remove the domain from the preload list.
* CSP rolled out in enforce mode without Report-Only testing. Google Tag Manager started failing silently. Analytics data lost for three weeks before anyone noticed.
* No X-Frame-Options or CSP frame-ancestors. Attacker built an invisible iframe over the customer portal's withdraw button. Users clicking what looked like a free game button were actually authorizing transfers.
* No `X-Content-Type-Options: nosniff`. User uploaded a .txt file that contained HTML and JavaScript. Browser sniffed it as HTML, rendered it, ran the script, stole every other user's session cookie.
* Referrer-Policy unset, default behavior leaks full URL with query string (often containing session tokens, password reset codes, or authentication tokens) to every external image, script, and link. Customer data exposed in third party analytics logs.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the nine headers gets the same six part treatment:

1. **What it does**: the canonical RFC or W3C spec plus the practical implication.
2. **Syntax and directives**: every legal value, what it means, and when it is wrong.
3. **How to build it on Bubbles**: paste ready nginx config.
4. **How to verify it**: curl commands plus external scanner checks where applicable.
5. **How to troubleshoot**: the four or five failure modes seen in the field and how to recognize each.
6. **How to fix common breakage**: ordered repair steps.

The cross origin isolation pattern (combining COEP, COOP, and CORP) gets its own dedicated section because the three headers must be tuned together. Asset class recipes are collected in Section 16.

---

## 4. THE DEFENSE IN DEPTH MENTAL MODEL (READ THIS FIRST)

Security headers form layers. No single header is sufficient. Each layer addresses a different attack class, and defenses are stacked so that bypassing one still leaves the attacker facing the next. Internalize the layer model and every header decision becomes obvious.

```
Visitor's browser requests https://example.com/
        |
        v
==================== TRANSPORT LAYER ====================
        |
        v
Strict-Transport-Security (HSTS)
   Force HTTPS only, prevent downgrade attacks
   Prevent man in the middle attacks on public Wi-Fi
        |
        v
==================== CONTENT EXECUTION LAYER ====================
        |
        v
Content-Security-Policy
   Control which scripts may execute
   Control which origins may be loaded as scripts, styles, images, frames
   Block inline scripts unless explicitly allowed (via nonce or hash)
        |
        v
X-Content-Type-Options: nosniff
   Force the declared Content-Type to be authoritative
   Block MIME confusion attacks
        |
        v
X-Frame-Options (or CSP frame-ancestors)
   Control whether this page may be embedded in an iframe
   Block clickjacking
        |
        v
==================== PRIVACY AND CAPABILITY LAYER ====================
        |
        v
Referrer-Policy
   Control how much referrer URL is sent to other origins
   Prevent token and PII leakage via outbound requests
        |
        v
Permissions-Policy
   Control which browser APIs (camera, mic, geolocation, etc) may be used
   Prevent third party scripts from abusing capabilities
        |
        v
==================== CROSS ORIGIN ISOLATION LAYER ====================
        |
        v
Cross-Origin-Resource-Policy (on resources)
   Control who may load this resource
Cross-Origin-Embedder-Policy (on documents)
   Require resources to opt in via CORP or CORS
Cross-Origin-Opener-Policy (on documents)
   Isolate browsing context from cross origin windows
        |
        v
        ==> Cross origin isolated state
        ==> Unlocks SharedArrayBuffer, high precision timers, WebAssembly threading
        |
        v
==================== APPLICATION ====================
        |
        v
Render page, execute scripts within all the constraints above
```

Five rules govern the system:

1. **Stack the layers.** Even if your CSP is bulletproof, set HSTS so the connection cannot be downgraded. Even if HSTS is set, set CSP so XSS cannot execute. The point is overlap.
2. **Restrict by default, allow by exception.** Every header should start with the most restrictive policy that works. Expand only when a specific feature requires it.
3. **Test in Report-Only first.** CSP and CORP/COEP/COOP can break sites in subtle ways. Roll out in report only mode, monitor violations for at least a week, then enforce.
4. **HSTS preload is one way.** Submitting to the preload list is reversible but takes months. Test thoroughly before submitting.
5. **The `add_header` inheritance trap applies to every security header.** Any `add_header` in a location block wipes parent declarations. Use the snippet include pattern from the start.

A correctly configured security header stack produces an A+ rating on observatory.mozilla.org and securityheaders.com, blocks XSS even if attackers find an injection vector, prevents clickjacking, prevents transport downgrade, controls capability abuse, and isolates the origin from cross origin attacks. The same response, the same protection, every visitor.

---

## 5. STRICT-TRANSPORT-SECURITY (HSTS, TRANSPORT SECURITY)

### 5.1 What It Does

`Strict-Transport-Security` (HSTS) instructs the browser to access this origin only over HTTPS for a specified duration, ignoring any user attempt to use HTTP and refusing to bypass certificate errors. Defined in RFC 6797.

```
Strict-Transport-Security: max-age=31536000
Strict-Transport-Security: max-age=31536000; includeSubDomains
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

The first time a browser receives the header over HTTPS, it remembers the origin and the max-age. Every subsequent attempt to visit `http://example.com/...` is automatically upgraded to `https://` before the request leaves the browser. Certificate validation errors cannot be clicked through. This prevents downgrade attacks (active man in the middle stripping HTTPS) and stops accidental HTTP requests from leaking anything sensitive.

HSTS only applies to subsequent visits. The very first visit to a new origin is still vulnerable to a downgrade. The **preload list** solves this: browsers ship with a hardcoded list of origins that have HSTS pre activated, so even the first visit is HTTPS only.

### 5.2 Directives

| Directive | Meaning |
|---|---|
| `max-age=<seconds>` | How long the browser remembers HSTS for this origin. Required. Common values: `0` (disable), `86400` (1 day, for testing), `2592000` (30 days), `31536000` (1 year), `63072000` (2 years, recommended for preload) |
| `includeSubDomains` | HSTS applies to every subdomain. Required for preload list. Once enabled with a long max-age, every subdomain must serve HTTPS or it becomes inaccessible |
| `preload` | Signals consent to be included in the browser preload list. Required for preload list submission. Without preload directive, the preload list submission is rejected |

### 5.3 The Preload List (Permanent Consequences Warning)

The HSTS preload list is maintained by the Chromium project and used by Chrome, Edge, Firefox, Safari, Opera, and Brave. To submit a domain:

1. Serve HSTS on the apex domain over HTTPS with `max-age >= 31536000`, `includeSubDomains`, and `preload`.
2. Every HTTP request must redirect to HTTPS via 301.
3. Every subdomain (now and in the future, including `mail.`, `admin.`, `dev.`, `staging.`) must serve HTTPS with a valid certificate.
4. Submit at https://hstspreload.org/.
5. Wait weeks to months for inclusion in the next browser release.

**Permanent consequences:**

* Removal takes months after request and is not guaranteed within that timeframe.
* If any subdomain serves only HTTP after preload, that subdomain is **permanently inaccessible** to every browser that has the preload list cached.
* Adding a new subdomain that only does HTTP requires a TLS certificate from day one.
* Internal subdomains used by employees (admin tools, dev environments, monitoring) become inaccessible without HTTPS.

**Bubbles policy:** preload list submission is appropriate for established client production domains. It is **not appropriate** for `thatwebhostingguy.com` (the wildcard subdomain platform) because that platform spins up new subdomains constantly and not all may be production ready. For `thatwebhostingguy.com` use long max-age and includeSubDomains but skip preload.

### 5.4 The Ramp Up Strategy

Do not deploy long max-age HSTS on day one. If something breaks, every browser that visited during that window is locked into the broken state for the full max-age duration. Ramp up:

```
# Week 1: testing only
Strict-Transport-Security: max-age=86400

# Week 2 to 4: short commitment
Strict-Transport-Security: max-age=2592000; includeSubDomains

# Month 2: medium commitment
Strict-Transport-Security: max-age=15768000; includeSubDomains

# Month 3+: full commitment, ready for preload
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

At each stage, verify every subdomain works over HTTPS, every internal tool still loads, and there are no certificate issues. Move to the next stage only after the current max-age has fully elapsed.

### 5.5 How To Build It On Bubbles

For an established production site already on HTTPS:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # HSTS: 2 years, all subdomains, preload list eligible
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}

# HTTP server redirects to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

**Critical:** HSTS must be sent ONLY on HTTPS responses. Sending HSTS on an HTTP response is per the spec ignored, but it indicates the server is confused. The HTTP server block above does not include `add_header Strict-Transport-Security`.

For a new site rolling out HSTS for the first time:

```nginx
# Week 1: short max-age, no preload
add_header Strict-Transport-Security "max-age=86400" always;
```

After verifying everything works, progress through the stages in Section 5.4.

For `thatwebhostingguy.com` wildcard platform (no preload):

```nginx
server {
    listen 443 ssl;
    server_name *.thatwebhostingguy.com thatwebhostingguy.com;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    # No preload directive
}
```

### 5.6 How To Verify

```bash
# 1. Confirm HSTS is sent on HTTPS
curl -sI https://example.com/ | grep -i strict-transport-security
# Expected: strict-transport-security: max-age=63072000; includeSubDomains; preload

# 2. Confirm HSTS is NOT sent on HTTP (should be a 301 instead)
curl -sI http://example.com/ | grep -iE "strict-transport-security|^HTTP|^Location"
# Expected: HTTP/1.1 301 Moved Permanently
# Expected: Location: https://example.com/
# Expected: NO strict-transport-security header

# 3. Check max-age is at least 1 year (for preload eligibility)
MAX_AGE=$(curl -sI https://example.com/ | grep -i strict-transport-security | grep -oE "max-age=[0-9]+" | cut -d= -f2)
echo "max-age = $MAX_AGE seconds = $(echo "$MAX_AGE / 86400" | bc) days"

# 4. Check preload status
curl -s "https://hstspreload.org/api/v2/status?domain=example.com" | python3 -m json.tool

# 5. Verify every subdomain serves HTTPS (required before preload)
for sub in www mail admin dev staging api blog; do
    echo "=== $sub.example.com ==="
    curl -sI -m 5 "https://$sub.example.com/" 2>&1 | head -1 || echo "FAILED to connect"
done

# 6. Browser check: visit chrome://net-internals/#hsts and query example.com
```

### 5.7 Troubleshooting

**Symptom: HSTS preload submission rejected.**
Causes:
1. Missing `preload` directive in the header. Add it.
2. Missing `includeSubDomains`. Add it.
3. max-age under 31536000. Increase to at least 31536000.
4. HTTP does not redirect to HTTPS, or redirects with 302 instead of 301. Fix the HTTP server block.
5. A subdomain (including one you forgot about) serves only HTTP or has an invalid certificate. Audit all subdomains.

**Symptom: A subdomain became inaccessible after enabling HSTS with includeSubDomains.**
That subdomain does not serve HTTPS. Either:
1. Provision a TLS certificate for it immediately.
2. Disable HSTS includeSubDomains (browsers will pick up the change on their next visit, but only after the current max-age expires).
3. If you preloaded already, you must remove from the preload list (months long process) or stand up HTTPS for the broken subdomain.

The right answer is almost always option 1: get the certificate.

**Symptom: Old browsers refuse to accept new certificate after expiry.**
This is HSTS working as designed. With HSTS active, expired or invalid certificates cannot be clicked through. The user must wait until the certificate is renewed or visit a different browser.

For Bubbles: ensure `certbot.timer` is active and tested:

```bash
systemctl status certbot.timer
sudo certbot renew --dry-run
```

**Symptom: HSTS header appears on HTTP responses.**
Wrong but harmless per the spec. Browsers ignore HSTS over HTTP. Fix by removing the `add_header` from the HTTP server block. The HTTP server block should only redirect, not set headers.

**Symptom: Removed includeSubDomains but old browsers still enforce it.**
Browsers honor the cached value for the original max-age duration. The fix is to wait, or to issue a new header with `max-age=0` for a few days, which clears the HSTS state, then re enable with the desired (more permissive) configuration.

### 5.8 How To Fix Common Breakage

**Case: Need to roll back HSTS quickly.**
Set max-age to 0:

```nginx
add_header Strict-Transport-Security "max-age=0" always;
```

Browsers that revisit pick up the new value and clear their stored HSTS state. Browsers that do not revisit retain the old value until it expires naturally.

If preloaded, max-age=0 alone is insufficient. You must also remove from the preload list, which takes months.

**Case: Subdomain forgotten during preload, now inaccessible.**
Add HTTPS to the subdomain immediately:

```bash
sudo certbot --nginx -d forgotten.example.com
```

This is faster than removing from the preload list.

**Case: Want to test HSTS without committing.**
Use the `Strict-Transport-Security-Report-Only` header. Note: this is a draft proposal, not widely supported. Better strategy: use a short max-age (86400 = 1 day) during testing and observe via browser devtools.

---

## 6. CONTENT-SECURITY-POLICY (THE BIGGEST LEVER, SCRIPT AND RESOURCE CONTROL)

### 6.1 What It Does

`Content-Security-Policy` (CSP) is a declarative allow list of where the browser may load resources from. Defined in W3C CSP Level 3. It controls which scripts may execute, which styles may load, which images and fonts may be fetched, which origins may be embedded in frames, and many other resource boundaries. CSP is the single most effective defense against XSS, the most common web vulnerability class.

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-aBc123Xyz' 'strict-dynamic'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
```

A response with this CSP would: load resources only from same origin by default; allow scripts from same origin AND any inline script with the matching nonce; allow trust to propagate through dynamic script loading; allow same origin styles plus inline styles; allow images from same origin, data URLs, and any HTTPS origin; refuse to be embedded in any frame; restrict `<base>` to same origin; restrict form submission targets to same origin.

CSP is the most powerful and most complex header in this framework. The configuration is per site and almost never trivially copyable between sites. Each origin needs a CSP tuned to its actual resource graph.

### 6.2 Directives

The full directive reference. Each directive controls a resource type or a security behavior.

**Fetch directives (control resource loading by type):**

| Directive | Controls |
|---|---|
| `default-src` | Fallback for any fetch directive not explicitly set. Default for all others |
| `script-src` | JavaScript files and inline scripts |
| `script-src-elem` | JavaScript loaded via `<script>` elements (more specific than script-src) |
| `script-src-attr` | Inline event handlers like `onclick=`. Should always be `'none'` |
| `style-src` | CSS files and inline styles |
| `style-src-elem` | CSS via `<link>` and `<style>` |
| `style-src-attr` | Inline `style=` attributes |
| `img-src` | Images |
| `font-src` | Web fonts |
| `connect-src` | XHR, fetch(), WebSocket, EventSource, sendBeacon targets |
| `media-src` | `<audio>` and `<video>` sources |
| `object-src` | `<object>`, `<embed>`, `<applet>`. Should always be `'none'` |
| `frame-src` | `<iframe>` and `<frame>` sources |
| `child-src` | Worker and frame sources (deprecated; use frame-src and worker-src) |
| `worker-src` | Web Workers, Shared Workers, Service Workers |
| `manifest-src` | PWA manifest URL |
| `prefetch-src` | Prefetched resources (deprecated, removed from spec) |

**Document directives (control page properties):**

| Directive | Controls |
|---|---|
| `base-uri` | What `<base href="...">` may set as the base URL. Should always be `'self'` or `'none'` |
| `sandbox` | Apply iframe sandbox restrictions to the document itself |

**Navigation directives:**

| Directive | Controls |
|---|---|
| `form-action` | URLs that forms may submit to |
| `frame-ancestors` | What origins may embed this page in a frame (replaces X-Frame-Options) |

**Reporting:**

| Directive | Controls |
|---|---|
| `report-uri <url>` | URL to send violation reports (legacy) |
| `report-to <group>` | Reporting API group to send violation reports (modern) |

**Trusted Types:**

| Directive | Controls |
|---|---|
| `require-trusted-types-for 'script'` | Requires Trusted Types for DOM XSS sinks |
| `trusted-types <policy-name>` | Allow listed Trusted Types policies |

### 6.3 Source Values

What can appear inside a directive's value list:

| Value | Meaning |
|---|---|
| `'self'` | Same origin (scheme, host, port match) |
| `'none'` | Nothing allowed |
| `'unsafe-inline'` | Allow inline scripts and styles. Bypasses most XSS protection. **Avoid** |
| `'unsafe-eval'` | Allow `eval()`, `setTimeout(string)`, `new Function()`. **Avoid** |
| `'unsafe-hashes'` | Allow inline event handlers if they match a hash. Niche |
| `'strict-dynamic'` | Trust propagates from nonce'd/hash'd scripts to scripts they load. The modern best practice |
| `'nonce-<base64>'` | Allow inline scripts/styles that have this nonce attribute |
| `'sha256-<base64>'` | Allow inline scripts/styles whose content hashes to this value |
| `https:` | Any HTTPS origin |
| `http:` | Any HTTP origin |
| `data:` | Data URLs (`data:image/png;base64,...`). For images and fonts; **never** for scripts |
| `blob:` | Blob URLs |
| `filesystem:` | Filesystem URLs |
| `mediastream:` | MediaStream URLs |
| `https://example.com` | Specific origin |
| `https://*.example.com` | Wildcard host |
| `https://example.com:8443` | Specific port |
| `*` | Any origin (use sparingly) |

### 6.4 The Nonce Plus strict-dynamic Pattern (Modern Best Practice)

The single most important CSP pattern in 2026 is `nonce + 'strict-dynamic'` for script-src. It solves the impossible problem of allowing legitimate inline scripts (GTM bootstrap, analytics, custom inline data) without allowing arbitrary attacker injected scripts.

How it works:

1. Server generates a fresh random nonce per response (e.g. 16 random bytes base64 encoded).
2. Server includes the nonce in the CSP header: `script-src 'nonce-aBc123XyZ' 'strict-dynamic'`.
3. Server emits inline scripts with the matching nonce attribute: `<script nonce="aBc123XyZ">...</script>`.
4. Browser allows these scripts. Attacker injected scripts (which do not know the nonce) are blocked.
5. `'strict-dynamic'` means: any script loaded by a nonce'd script is also trusted, recursively. So GTM (nonce'd) can inject its tags without you having to list every Google domain.

Implementation requires generating the nonce server side. Nginx alone cannot do this; you need a sidecar (FastAPI, Lua via OpenResty, or similar) to inject a fresh nonce into both the header and the HTML body.

FastAPI sidecar example:

```python
import secrets
import base64
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse

app = FastAPI()

@app.get("/")
async def index(request: Request):
    nonce = base64.b64encode(secrets.token_bytes(16)).decode()
    html = f'''
    <!DOCTYPE html>
    <html lang="en-US">
    <head>
        <meta charset="utf-8">
        <title>Example</title>
        <script nonce="{nonce}">
            // GTM bootstrap with nonce
            (function(w,d,s,l,i){{w[l]=w[l]||[];w[l].push({{'gtm.start':
            new Date().getTime(),event:'gtm.js'}});var f=d.getElementsByTagName(s)[0],
            j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
            'https://www.googletagmanager.com/gtm.js?id='+i+dl;j.setAttribute('nonce','{nonce}');
            f.parentNode.insertBefore(j,f);}})(window,document,'script','dataLayer','GTM-PM56NF52');
        </script>
    </head>
    <body>...</body>
    </html>
    '''

    csp = (
        f"default-src 'self'; "
        f"script-src 'nonce-{nonce}' 'strict-dynamic' https: 'unsafe-inline'; "
        f"style-src 'self' 'unsafe-inline'; "
        f"img-src 'self' data: https:; "
        f"font-src 'self'; "
        f"connect-src 'self' https://*.google-analytics.com https://*.googletagmanager.com; "
        f"frame-ancestors 'none'; "
        f"base-uri 'self'; "
        f"form-action 'self'; "
        f"object-src 'none'; "
        f"upgrade-insecure-requests"
    )

    return HTMLResponse(
        content=html,
        headers={"Content-Security-Policy": csp}
    )
```

The `'unsafe-inline'` plus `https:` after the nonce is a backward compatibility fallback: older browsers that do not understand `'strict-dynamic'` (none in 2026, but defense in depth) fall back to those values. Browsers that understand `'strict-dynamic'` ignore them. This is the documented pattern from web.dev/strict-csp.

### 6.5 Static Sites Without Server Side Nonce Generation

Pure static sites (no FastAPI sidecar) cannot generate nonces. Three alternatives:

**1. Hash based CSP for known inline scripts.** Compute the SHA256 hash of each inline script at build time, embed the hashes in the CSP header. Only works if scripts never change. GTM does not work this way because its content changes.

```nginx
add_header Content-Security-Policy "script-src 'self' 'sha256-LjyKKKsXNcXyTtUH3GcRbE2QEm2X3MfFs6ZkPHfZQiM=' https://www.googletagmanager.com" always;
```

**2. Origin allow list for everything.** Less secure but simpler to maintain.

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://www.google-analytics.com" always;
```

This is what most Bubbles static sites use. `'unsafe-inline'` is not ideal but acceptable given the controlled content of the site.

**3. Move dynamic responses to the FastAPI sidecar.** Pages that need nonce based CSP (logged in areas, dashboards) go through the sidecar; static marketing pages use origin allow list CSP.

### 6.6 Report-Only Mode (Mandatory For First Deploy)

The `Content-Security-Policy-Report-Only` header has identical syntax to `Content-Security-Policy` but does not enforce. Violations are reported to the configured endpoint but resources still load. This is mandatory for the first deploy of any new CSP: it surfaces every violation without breaking the site.

```nginx
# Report-only during testing
add_header Content-Security-Policy-Report-Only "default-src 'self'; script-src 'self' 'unsafe-inline'; report-uri /csp-report" always;

location = /csp-report {
    proxy_pass http://127.0.0.1:9090;
}
```

The FastAPI sidecar receives reports as `application/csp-report` content type:

```python
@app.post("/csp-report")
async def csp_report(request: Request):
    report = await request.json()
    logging.warning(f"CSP violation: {report}")
    return Response(status_code=204)
```

After at least a week of monitoring with zero unexpected violations, switch from `Content-Security-Policy-Report-Only` to `Content-Security-Policy` (the enforcing variant) and the policy goes live.

### 6.7 The frame-ancestors Directive (Replaces X-Frame-Options)

`frame-ancestors` controls which origins may embed this page in an iframe. It supersedes `X-Frame-Options` and is more flexible (supports multiple origins, wildcards, etc).

```
Content-Security-Policy: frame-ancestors 'none'
Content-Security-Policy: frame-ancestors 'self'
Content-Security-Policy: frame-ancestors https://partner.example.com
Content-Security-Policy: frame-ancestors 'self' https://*.example.com
```

| Value | Meaning |
|---|---|
| `'none'` | No embedding allowed (equivalent to X-Frame-Options: DENY) |
| `'self'` | Only same origin embedding (equivalent to X-Frame-Options: SAMEORIGIN) |
| `https://example.com` | Only this specific origin |
| `https://*.example.com` | Any subdomain of example.com |

Modern browsers respect `frame-ancestors` over `X-Frame-Options` when both are present and conflict. Best practice: set both for belt and suspenders.

### 6.8 The upgrade-insecure-requests Directive

```
Content-Security-Policy: upgrade-insecure-requests
```

Tells the browser to automatically upgrade any `http://` resource references in the page to `https://`. Useful for fixing mixed content during a slow HTTPS migration. Once all content is on HTTPS, this directive is harmless and can be left in place.

### 6.9 How To Build It On Bubbles

**Minimal CSP for static Bubbles site without third party scripts:**

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests" always;
```

This is locked down. No third party resources. Only inline styles are allowed (which most templates need for theme injection). No inline scripts.

**Standard Bubbles CSP with GTM, GA4, and common third parties:**

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://www.google-analytics.com https://www.googleadservices.com https://googleads.g.doubleclick.net https://connect.facebook.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: https: blob:; font-src 'self' data: https://fonts.gstatic.com; connect-src 'self' https://*.google-analytics.com https://*.analytics.google.com https://*.googletagmanager.com https://www.facebook.com; frame-src 'self' https://www.googletagmanager.com https://www.google.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests" always;
```

This is the typical Bubbles production CSP. Allows all the common analytics and tag manager domains. Still blocks `object-src` and `frame-ancestors`.

**Strict CSP with nonce (requires FastAPI sidecar):**

```nginx
location / {
    proxy_pass http://127.0.0.1:9090;
    # CSP header is generated by the sidecar per request with a fresh nonce
    # Nginx does not override
}
```

The sidecar emits the CSP header with the nonce, as shown in Section 6.4.

**Report-Only deployment:**

```nginx
# First week: report only, see what breaks
add_header Content-Security-Policy-Report-Only "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com; report-uri /csp-report" always;

# After validating: enforce
# add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com; report-uri /csp-report" always;
```

### 6.10 How To Verify

```bash
# 1. Confirm CSP is sent
curl -sI https://example.com/ | grep -i content-security-policy

# 2. Pretty print the directive list
curl -sI https://example.com/ | grep -i content-security-policy | tr ';' '\n' | sed 's/^ */    /'

# 3. Test in Chrome DevTools:
#    Open the site, open DevTools (F12), Console tab
#    Any CSP violations appear in red
#    Network tab: blocked resources show "blocked:csp" in Status

# 4. Use external scanner
echo "Visit: https://csp-evaluator.withgoogle.com/?csp=https://example.com/"

# 5. Use observatory.mozilla.org
echo "Visit: https://observatory.mozilla.org/analyze/example.com"

# 6. Verify that 'strict-dynamic' is correctly understood (if used)
# Look for 'strict-dynamic' in the CSP value
curl -sI https://example.com/ | grep -i content-security-policy | grep -o "'strict-dynamic'"

# 7. Test that a CSP violation is reported (for Report-Only)
curl -sX POST https://example.com/csp-report \
     -H "Content-Type: application/csp-report" \
     -d '{"csp-report":{"document-uri":"https://example.com/","violated-directive":"script-src","blocked-uri":"https://evil.com/script.js"}}'
# Should return 204 No Content
```

### 6.11 Troubleshooting

**Symptom: GTM not loading; console shows "Refused to execute inline script".**
Cause: CSP `script-src` does not allow GTM's inline bootstrap or its loaded scripts.
Fix: add `https://www.googletagmanager.com https://www.google-analytics.com` to script-src. If using strict CSP, ensure the GTM snippet has the matching nonce attribute. See Section 6.4.

**Symptom: GA4 events not firing; no console errors but no data in GA.**
Cause: CSP `connect-src` does not allow the GA endpoint.
Fix: add `https://*.google-analytics.com https://*.analytics.google.com` to connect-src. Verify with Network tab in DevTools (look for blocked requests to `region1.google-analytics.com` or similar).

**Symptom: External fonts not loading; falls back to system fonts.**
Cause: CSP `font-src` does not allow the font origin.
Fix: add the font origin (`https://fonts.gstatic.com` for Google Fonts) to font-src.

**Symptom: Inline event handlers (`<button onclick="...">`) not working.**
Cause: CSP blocks inline event handlers unless `'unsafe-inline'` or matching hash is allowed for script-src-attr. The modern recommendation is to move these to addEventListener calls in JavaScript.
Fix: refactor inline handlers to use proper event listeners. Or add `script-src-attr 'unsafe-inline'` (less secure).

**Symptom: Images displaying as broken links; console shows CSP block.**
Cause: img-src does not allow the image origin.
Fix: list the origin in img-src. For dynamic images from various CDNs, `img-src 'self' data: https:` is a common permissive value.

**Symptom: Page works in development but breaks in production.**
Cause: production CSP is stricter than development. Test environments often use permissive CSP for debugging.
Fix: deploy in Report-Only mode first to surface every violation, fix them, then enforce.

**Symptom: CSP report endpoint receiving millions of reports.**
Causes:
1. Browser extensions injecting content trigger spurious violations. Filter reports by `source-file` to exclude extension origins.
2. A real ongoing XSS attack. Investigate immediately.
3. A legitimate resource was missed in the policy. Add it.

**Symptom: Header value exceeds size limit.**
Large CSP with many origins can exceed default header size limits (8 KB nginx default). Either increase the limit:

```nginx
proxy_buffer_size 16k;
proxy_buffers 8 16k;
large_client_header_buffers 4 16k;
```

Or simplify the CSP by using wildcards (`https://*.googleapis.com` instead of listing each subdomain).

### 6.12 How To Fix Common Breakage

**Case: GTM works in browser but CSP reports show violations from server side rendering.**
The SSR pipeline does not include the nonce in script tags. Audit the template engine to ensure nonce attribute is injected on every inline script.

**Case: Need to allow a third party script that loads other scripts dynamically.**
Use `'strict-dynamic'` pattern (Section 6.4). The first script gets a nonce; any scripts it loads are trusted automatically.

**Case: Site has hundreds of inline event handlers (legacy code).**
Short term: add `'unsafe-inline'` to script-src-attr. Long term: refactor to addEventListener.

**Case: CSP blocking legitimate AdSense scripts.**
AdSense requires permissive CSP. The minimum directives needed:

```
script-src 'self' 'unsafe-inline' https://pagead2.googlesyndication.com https://*.googleadservices.com https://*.googlesyndication.com https://*.doubleclick.net;
img-src 'self' data: https:;
frame-src https://googleads.g.doubleclick.net https://www.google.com;
connect-src 'self' https://pagead2.googlesyndication.com https://*.googleadservices.com;
```

AdSense's full domain list changes occasionally; monitor reports.

**Case: Want to use SharedArrayBuffer but it is undefined.**
Cross origin isolation required. Implement COEP plus COOP (Section 14). CSP alone does not enable SharedArrayBuffer.

---

## 7. X-FRAME-OPTIONS (CLICKJACKING DEFENSE, MOSTLY SUPERSEDED BY CSP FRAME-ANCESTORS)

### 7.1 What It Does

`X-Frame-Options` controls whether the browser may render this page inside a `<frame>`, `<iframe>`, `<embed>`, or `<object>`. Defined in RFC 7034. Superseded by CSP `frame-ancestors` for new development, but still widely deployed and respected.

```
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
```

The clickjacking attack: attacker creates a page with an invisible iframe pointing at your site, overlaid with decoy elements. User clicks what they think is a button on the attacker's page, actually clicks an authorized action on your site (transfer money, change settings, accept terms). `X-Frame-Options: DENY` prevents the iframe from rendering at all.

### 7.2 Values

| Value | Meaning |
|---|---|
| `DENY` | No site (including same origin) may embed this in a frame |
| `SAMEORIGIN` | Only same origin pages may embed |
| `ALLOW-FROM <uri>` | Deprecated. Only this specific origin may embed. Not supported in Chrome or Safari; use CSP frame-ancestors instead |

### 7.3 Relationship to CSP frame-ancestors

`frame-ancestors` (in CSP) is more capable. It supports:

* Multiple origins (X-Frame-Options can list only one).
* Wildcards.
* Granular path control.
* Modern browsers respect it over X-Frame-Options when both are present and conflict.

**Bubbles policy:** set both. The redundancy costs nothing and protects against older browsers or misconfigured intermediaries.

### 7.4 How To Build It On Bubbles

For a site that should never be embedded:

```nginx
add_header X-Frame-Options "DENY" always;
# CSP equivalent (set both)
add_header Content-Security-Policy "frame-ancestors 'none'" always;
```

For a site that may be embedded by same origin pages (most cases):

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Content-Security-Policy "frame-ancestors 'self'" always;
```

For a site that may be embedded by a specific partner:

```nginx
# X-Frame-Options cannot do this safely in modern browsers
# Use CSP frame-ancestors only
add_header Content-Security-Policy "frame-ancestors 'self' https://partner.example.com" always;
```

The Bubbles default in `snippets/common-security-headers.conf`:

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

The CSP equivalent is set in the per site CSP.

### 7.5 How To Verify

```bash
# 1. Confirm X-Frame-Options is sent
curl -sI https://example.com/ | grep -i x-frame-options
# Expected: x-frame-options: SAMEORIGIN

# 2. Test actual frame blocking
# Create a test HTML file locally:
cat > /tmp/clickjack-test.html << 'EOF'
<html><body>
  <h1>Clickjack test</h1>
  <iframe src="https://example.com/" width="800" height="600"></iframe>
</body></html>
EOF

# Open the file in a browser; the iframe should be blocked or empty

# 3. Verify in DevTools Network tab: the framed request shows "X-Frame-Options block" in Console

# 4. Confirm CSP frame-ancestors agrees
curl -sI https://example.com/ | grep -i content-security-policy | grep -o "frame-ancestors [^;]*"
```

### 7.6 Troubleshooting

**Symptom: Legitimate iframe of own site is blocked.**
X-Frame-Options is DENY. Change to SAMEORIGIN.

**Symptom: Embedded partner integration blocked.**
ALLOW-FROM is not supported in Chrome or Safari. Use CSP frame-ancestors instead and remove ALLOW-FROM.

**Symptom: Scanner reports missing X-Frame-Options.**
Header is not set. Add to common-security-headers.conf and reload.

**Symptom: X-Frame-Options and frame-ancestors disagree.**
Modern browsers honor frame-ancestors. Older browsers honor X-Frame-Options. Set both to consistent values to avoid version specific behavior.

### 7.7 How To Fix Common Breakage

**Case: Need to allow embedding from a partner site after launch.**
Update CSP frame-ancestors (X-Frame-Options cannot list specific allowed origins):

```nginx
add_header Content-Security-Policy "... frame-ancestors 'self' https://partner.example.com; ..." always;
# Keep X-Frame-Options for older browsers:
add_header X-Frame-Options "SAMEORIGIN" always;
```

The X-Frame-Options says "same origin or nothing" which is more restrictive than frame-ancestors allows, but modern browsers prefer frame-ancestors so the partner can embed.

---

## 8. X-CONTENT-TYPE-OPTIONS (MIME SNIFFING PREVENTION)

### 8.1 What It Does

`X-Content-Type-Options: nosniff` tells the browser to treat the declared `Content-Type` header as authoritative and not attempt to guess (sniff) the content type from the body bytes. The only valid value is `nosniff`. Defined in WHATWG Fetch specification.

```
X-Content-Type-Options: nosniff
```

Without nosniff, the browser may sniff content. A response declared `text/plain` containing HTML and JavaScript may be sniffed as HTML and rendered, executing the JavaScript. This is the basis of the user upload XSS attack class discussed in framework-http-content-headers.md Section 9.5.

With nosniff, declared types are honored. A file declared `text/plain` is displayed as text even if the body contains executable HTML.

### 8.2 Why It Is Always Set To nosniff

The header has effectively one purpose: enable nosniff. There is no situation in 2026 where you want MIME sniffing. Always set:

```
X-Content-Type-Options: nosniff
```

### 8.3 How To Build It On Bubbles

In the shared security header snippet:

```nginx
# /etc/nginx/snippets/common-security-headers.conf
add_header X-Content-Type-Options "nosniff" always;
```

Included in every location across the Bubbles fleet.

### 8.4 How To Verify

```bash
# Confirm header is present on every response
curl -sI https://example.com/ | grep -i x-content-type-options
# Expected: x-content-type-options: nosniff

# Check across asset types
for path in / /css/main.css /js/app.js /images/logo.png /downloads/file.pdf; do
    HEADER=$(curl -sI "https://example.com$path" 2>/dev/null | grep -i x-content-type-options)
    echo "$path: $HEADER"
done

# Verify behavior with a deliberately misdeclared file
# (Test only on a non production site)
# Create a .txt file containing HTML
echo '<script>alert("test")</script>' > /var/www/sites/test.example.com/test.txt
curl -sI https://test.example.com/test.txt | grep -iE "content-type|x-content-type-options"
# Should show: content-type: text/plain
# Should show: x-content-type-options: nosniff
# Browser should display the text, NOT execute the script
```

### 8.5 Troubleshooting

**Symptom: Stored XSS via uploaded HTML file.**
nosniff is missing OR Content-Type is wrong (declared as `text/html` when it should be `application/octet-stream` for downloads).
Fix: ensure nosniff is set site wide, ensure user uploads are served with `Content-Type: application/octet-stream` and `Content-Disposition: attachment` (see framework-http-content-headers.md Section 9.5).

**Symptom: Old browsers ignoring nosniff.**
Internet Explorer and very old browsers had inconsistent nosniff support. Not a concern in 2026.

**Symptom: Header missing on some file types.**
The `add_header` inheritance trap. A location block redeclared headers without including the snippet. Audit:

```bash
nginx -T 2>/dev/null | grep -B5 "add_header" | grep -B1 "x-content-type-options" | head -50
```

### 8.6 How To Fix Common Breakage

**Case: Pen tester reports missing nosniff on some endpoints.**
Add `include snippets/common-security-headers.conf` to every location that redeclares any add_header.

**Case: A specific file needs sniffing for legacy reasons.**
This is an antipattern. Fix the underlying Content-Type instead. nosniff should never be removed.

---

## 9. REFERRER-POLICY (OUTBOUND URL PRIVACY)

### 9.1 What It Does

`Referrer-Policy` controls how much of the current URL (the referrer) is included in the `Referer` HTTP header on outbound requests (links, images, scripts, fetch). Defined in W3C Referrer Policy spec. Critical for privacy because URLs often contain session tokens, password reset codes, search queries, and other sensitive data.

```
Referrer-Policy: strict-origin-when-cross-origin
Referrer-Policy: no-referrer
Referrer-Policy: same-origin
```

The default browser behavior (when no header is set) varies by browser, but most modern browsers default to `strict-origin-when-cross-origin` since 2020. Setting the header explicitly removes the ambiguity.

### 9.2 The Eight Values

| Value | Meaning |
|---|---|
| `no-referrer` | Never send Referer. Maximum privacy. Some analytics break |
| `no-referrer-when-downgrade` | Send full URL except when going from HTTPS to HTTP. Legacy default |
| `origin` | Send only the origin (`https://example.com`), no path or query |
| `origin-when-cross-origin` | Send full URL same origin, origin only cross origin |
| `same-origin` | Send full URL same origin, nothing cross origin |
| `strict-origin` | Send origin only, except when HTTPS to HTTP (nothing) |
| `strict-origin-when-cross-origin` | Full URL same origin, origin cross origin, nothing HTTPS to HTTP. **Modern default and recommendation** |
| `unsafe-url` | Always send full URL. Maximum leakage. Never use |

### 9.3 Why strict-origin-when-cross-origin Is The Right Default

The default value chosen by modern browsers and recommended by web.dev:

* Within your own origin, full URL is sent (useful for analytics and internal tracking).
* To other HTTPS origins, only your origin is sent (no path, no query, no token leakage).
* To HTTP origins, nothing is sent (no transport downgrade leakage).

For most sites this is correct. For sites that want even more privacy (e.g. anything healthcare related, anything with strong PII), use `same-origin` (nothing cross origin) or `no-referrer` (nothing ever).

### 9.4 How To Build It On Bubbles

In the shared security header snippet:

```nginx
# /etc/nginx/snippets/common-security-headers.conf
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

For a healthcare or HIPAA aware site, increase to `same-origin`:

```nginx
add_header Referrer-Policy "same-origin" always;
```

For specific high sensitivity pages (password reset, account settings):

```nginx
location ~ ^/(account|password-reset|verify) {
    include snippets/common-security-headers.conf;
    add_header Referrer-Policy "no-referrer" always;
}
```

### 9.5 How To Verify

```bash
# 1. Confirm header is present
curl -sI https://example.com/ | grep -i referrer-policy
# Expected: referrer-policy: strict-origin-when-cross-origin

# 2. Test actual referrer behavior
# Visit your site, click an external link, check the destination's logs
# Or use https://httpbin.org/headers as the click target
echo "Visit your site, click a link to https://httpbin.org/headers"
echo "Check the 'Referer' header in the response"

# 3. Verify per page override works
curl -sI https://example.com/account | grep -i referrer-policy
```

### 9.6 Troubleshooting

**Symptom: Analytics platform shows zero referrer data.**
Referrer-Policy is set to `no-referrer` or `same-origin`, which is correctly preventing referrer from leaving the origin.
Fix: this is by design. If analytics needs cross origin referrer, change to `strict-origin-when-cross-origin` (origin only) or accept the privacy tradeoff.

**Symptom: Password reset URL with token appearing in third party logs.**
Referrer-Policy not set, or set to `unsafe-url` or `no-referrer-when-downgrade`. The full URL including the token is being leaked.
Fix: set to `strict-origin-when-cross-origin` site wide, or to `no-referrer` on the password reset endpoint specifically.

**Symptom: Affiliate tracking does not record source.**
Affiliate tracking systems rely on the Referer header. If your site sets `no-referrer`, the affiliate link's tracking pixel cannot identify the source.
Fix: discuss tradeoffs. The affiliate cannot get full URL data; they can get origin. `strict-origin-when-cross-origin` is the compromise.

### 9.7 How To Fix Common Breakage

**Case: Need to allow third party affiliates to know which page sent traffic.**
Use `origin` or `strict-origin-when-cross-origin`. The affiliate sees your domain but not the specific page.

**Case: Want to leak nothing across origins for compliance reasons.**
Use `same-origin`. Combine with explicit click handling for affiliate tracking (passing source data in URL parameters rather than relying on Referer).

---

## 10. PERMISSIONS-POLICY (BROWSER CAPABILITY CONTROL)

### 10.1 What It Does

`Permissions-Policy` controls which browser APIs and features may be used on this page, including by third party iframes and scripts. Replaces the older `Feature-Policy` header (same purpose, different syntax). Defined in W3C Permissions Policy spec.

```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
Permissions-Policy: geolocation=(self), camera=(self "https://video.example.com")
```

The header tells the browser: "even if a script on this page tries to access the camera, refuse unless explicitly allowed". This protects users from third party scripts (analytics, ads, social widgets) silently activating sensitive APIs.

### 10.2 Syntax

The header value is a comma separated list of directives. Each directive has the form `feature=(<allowlist>)`. The allowlist is a space separated list of origins or special values inside parentheses.

```
Permissions-Policy: camera=(), microphone=(self), geolocation=(self "https://maps.example.com")
```

Allowlist values:

| Value | Meaning |
|---|---|
| `()` (empty) | Feature disabled everywhere |
| `(self)` | Feature allowed only in this document and same origin iframes |
| `(*)` | Feature allowed everywhere including cross origin iframes |
| `("https://example.com")` | Feature allowed in this origin |
| `(self "https://example.com")` | Same origin plus the specified origin |

### 10.3 Common Directives

The full list is large and growing. Most commonly configured:

**Privacy sensitive APIs (lock down by default):**

| Directive | Controls |
|---|---|
| `camera` | Camera access (getUserMedia) |
| `microphone` | Microphone access (getUserMedia) |
| `geolocation` | Geolocation API |
| `payment` | Payment Request API |
| `usb` | WebUSB API |
| `bluetooth` | Web Bluetooth API |
| `serial` | Web Serial API |
| `hid` | WebHID API |
| `midi` | Web MIDI API |
| `magnetometer` | Magnetometer sensor |
| `gyroscope` | Gyroscope sensor |
| `accelerometer` | Accelerometer sensor |
| `ambient-light-sensor` | Ambient light sensor |

**Browser features (selectively enable):**

| Directive | Controls |
|---|---|
| `fullscreen` | requestFullscreen() |
| `picture-in-picture` | Picture in Picture API |
| `autoplay` | Video and audio autoplay |
| `clipboard-read` | Clipboard read API |
| `clipboard-write` | Clipboard write API |
| `notifications` | Web Notifications |
| `push` | Push API |
| `screen-wake-lock` | Screen Wake Lock API |

**Privacy and tracking (block these):**

| Directive | Controls |
|---|---|
| `interest-cohort` | FLoC (deprecated). Set to `()` to opt out historically |
| `browsing-topics` | Topics API (FLoC's replacement). Set to `()` to opt out of tracking cohort calculation |
| `attribution-reporting` | Attribution Reporting API |
| `join-ad-interest-group` | Protected Audience API (FLEDGE) |
| `run-ad-auction` | Protected Audience API (FLEDGE) |
| `private-aggregation` | Private Aggregation API |

**Performance:**

| Directive | Controls |
|---|---|
| `unload` | Allow `unload` event handlers. Setting `()` blocks them, which improves bfcache eligibility and Core Web Vitals |
| `sync-xhr` | Synchronous XMLHttpRequest. Setting `()` blocks (recommended) |
| `cross-origin-isolated` | Whether the document may be cross origin isolated |

### 10.4 The Bubbles Baseline

For a typical Bubbles site that does not need camera, mic, or other sensitive APIs:

```nginx
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), serial=(), hid=(), midi=(), magnetometer=(), gyroscope=(), accelerometer=(), ambient-light-sensor=(), browsing-topics=(), interest-cohort=()" always;
```

This:

* Disables every sensitive device API.
* Opts out of FLoC and Topics tracking cohorts.
* Leaves general browser features (fullscreen, autoplay, notifications) at default permissive.

For a site that uses some features (e.g. video conferencing):

```nginx
add_header Permissions-Policy "camera=(self), microphone=(self), display-capture=(self), geolocation=(), payment=(), usb=(), bluetooth=()" always;
```

For maximum lockdown plus performance optimization:

```nginx
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), browsing-topics=(), interest-cohort=(), unload=(), sync-xhr=()" always;
```

### 10.5 How To Build It On Bubbles

In the shared security header snippet:

```nginx
# /etc/nginx/snippets/common-security-headers.conf
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), browsing-topics=(), interest-cohort=()" always;
```

For specific sites with different needs (medical practice using video conferencing, etc), override at the server level:

```nginx
server {
    server_name telehealth.example.com;

    # Standard headers minus permissions, since we override
    include snippets/common-security-headers-no-permissions.conf;
    add_header Permissions-Policy "camera=(self), microphone=(self), display-capture=(self), geolocation=(), payment=(), usb=(), bluetooth=()" always;
}
```

### 10.6 How To Verify

```bash
# 1. Confirm header is present
curl -sI https://example.com/ | grep -i permissions-policy

# 2. Pretty print
curl -sI https://example.com/ | grep -i permissions-policy | tr ',' '\n' | sed 's/^ */    /'

# 3. Test a blocked API in browser console (DevTools):
# navigator.geolocation.getCurrentPosition(p => console.log(p), e => console.error(e))
# Should show error like "Geolocation is blocked by permissions policy"

# 4. Check via observatory.mozilla.org or securityheaders.com
echo "Visit: https://securityheaders.com/?q=https://example.com/"
```

### 10.7 Troubleshooting

**Symptom: Feature blocked unexpectedly.**
Cause: Permissions-Policy disables it.
Fix: add `(self)` or appropriate allowlist to the directive.

**Symptom: Console warning "Error with Permissions-Policy header: Origin trial controlled feature not enabled: 'interest-cohort'".**
Cause: browser does not recognize `interest-cohort` (it was deprecated). The warning is harmless; the directive is ignored by browsers that do not implement it.
Fix: optionally remove `interest-cohort=()` from the header (Topics API replaced FLoC; use `browsing-topics=()`).

**Symptom: Browser does not honor a third party iframe's request for camera.**
Cause: parent document's Permissions-Policy does not allow camera in cross origin iframes.
Fix: explicitly allow: `camera=(self "https://embedded.example.com")`.

**Symptom: Header has wrong syntax (old Feature-Policy syntax).**
Old syntax: `camera 'none'; microphone 'self'`. New syntax: `camera=(), microphone=(self)`.
Fix: convert to new syntax. The two headers can coexist for older browsers but the new syntax is canonical.

### 10.8 How To Fix Common Breakage

**Case: Browser console warning about unknown Permissions-Policy feature.**
Browser does not recognize the feature name (maybe it is too new or too old). Unknown features are ignored, so the warning is harmless. To silence, remove the unknown feature.

**Case: Need to allow geolocation on a specific page only.**
Override at the location:

```nginx
location = /find-store {
    include snippets/common-security-headers-base.conf;
    add_header Permissions-Policy "geolocation=(self), camera=(), microphone=(), payment=(), usb=(), bluetooth=()" always;
}
```

**Case: Site uses Google Maps which requests geolocation.**
Allow geolocation for the Google Maps origin:

```nginx
add_header Permissions-Policy 'geolocation=(self "https://maps.googleapis.com"), camera=(), microphone=(), payment=()' always;
```

---

## 11. CROSS-ORIGIN-RESOURCE-POLICY (CORP, WHO CAN LOAD THIS RESOURCE)

### 11.1 What It Does

`Cross-Origin-Resource-Policy` (CORP) is set on a resource (image, script, stylesheet, font, JSON) to tell the browser which origins are allowed to load it. Defined in the WHATWG Fetch spec. Part of the cross origin isolation pattern.

```
Cross-Origin-Resource-Policy: same-origin
Cross-Origin-Resource-Policy: same-site
Cross-Origin-Resource-Policy: cross-origin
```

CORP differs from CORS:

* **CORS** allows a foreign origin to load your resource (you opt them in via response headers).
* **CORP** restricts who may load your resource (you opt out cross origin embedding by default).

A resource with `Cross-Origin-Resource-Policy: same-origin` cannot be loaded by any other origin, even via `<img src>` or `<script src>`. This blocks Spectre style side channel attacks and resource theft.

### 11.2 Values

| Value | Meaning |
|---|---|
| `same-origin` | Only the exact same origin (scheme, host, port) may load this |
| `same-site` | Only the same site may load this (different subdomains OK, different sites not) |
| `cross-origin` | Any origin may load this (default behavior) |

### 11.3 When To Set Each

| Resource type | Recommended CORP |
|---|---|
| Private user content (logged in user's profile photo) | `same-origin` |
| Site brand assets used only on the main site | `same-origin` or `same-site` |
| Resources you want partner sites to embed (logos, badges) | `cross-origin` |
| Public CDN assets used by many sites (fonts, libraries) | `cross-origin` |
| Resources that must be loadable when the embedding site has COEP | `cross-origin` (or absent, but absent means the embedder might block via COEP) |

### 11.4 How To Build It On Bubbles

For private user content:

```nginx
location /user-content/ {
    add_header Cross-Origin-Resource-Policy "same-origin" always;
}
```

For brand assets restricted to the main site:

```nginx
location ~* \.(jpg|jpeg|png|webp|svg|woff2)$ {
    add_header Cross-Origin-Resource-Policy "same-site" always;
}
```

For publicly embeddable resources (badges, logos that partners may embed):

```nginx
location /embed/ {
    add_header Cross-Origin-Resource-Policy "cross-origin" always;
}
```

The Bubbles default for a client site: `same-site` for all assets. This protects against hot linking and Spectre style attacks while allowing legitimate same site usage.

### 11.5 How To Verify

```bash
# 1. Confirm CORP is set
curl -sI https://example.com/images/logo.png | grep -i cross-origin-resource-policy

# 2. Test that a cross origin embed fails
# Create a test page on a different origin attempting to load the asset
# Browser console will show: NotSameOriginAfterDefaultedToSameOriginByCoep
# Or: blocked:NotSameOrigin

# 3. Verify for all asset classes
for path in /css/main.css /js/app.js /images/logo.png /fonts/main.woff2; do
    HEADER=$(curl -sI "https://example.com$path" 2>/dev/null | grep -i cross-origin-resource-policy)
    echo "$path: $HEADER"
done
```

### 11.6 Troubleshooting

**Symptom: External site cannot embed our logo.**
CORP is `same-origin` or `same-site`. Either:
1. Change to `cross-origin` for that specific path if external embedding is desired.
2. Refuse the embedding by maintaining the current value.

**Symptom: Our own embedded resource on a different subdomain blocked.**
CORP is `same-origin`. Change to `same-site` to allow subdomains.

**Symptom: COEP is enabled, all images blocked.**
COEP requires every cross origin resource to opt in via CORP. The blocked images are missing CORP. Either:
1. Add CORP to the resources.
2. Use CORS instead.
3. Switch COEP to `credentialless` mode.

### 11.7 How To Fix Common Breakage

**Case: CDN fonts blocked after enabling COEP.**
Google Fonts (when self hosted from googleapis.com) does not send CORP. Either:
1. Self host the fonts on Bubbles (recommended for performance anyway).
2. Use COEP `credentialless` mode.

**Case: Want to allow only specific partners to embed.**
CORP cannot do origin specific allowlisting. Use CORS instead: `Access-Control-Allow-Origin: https://partner.example.com`.

---

## 12. CROSS-ORIGIN-EMBEDDER-POLICY (COEP, WHAT THIS DOCUMENT CAN EMBED)

### 12.1 What It Does

`Cross-Origin-Embedder-Policy` (COEP) is set on a document and controls how that document may load cross origin resources. Required for cross origin isolation, which unlocks SharedArrayBuffer, high precision timers, and WebAssembly threading. Defined in the WHATWG HTML spec.

```
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Embedder-Policy: credentialless
Cross-Origin-Embedder-Policy: unsafe-none
```

When COEP is enabled, every cross origin resource the document loads must explicitly opt in via:

* CORP (`Cross-Origin-Resource-Policy: cross-origin` or compatible).
* CORS (`Access-Control-Allow-Origin: <embedder origin>`).

Without opt in, the browser refuses to load the resource. This prevents the embedder from accidentally pulling cross origin data into its process memory.

### 12.2 Values

| Value | Meaning |
|---|---|
| `unsafe-none` | Default. No restriction. Cross origin resources load freely |
| `require-corp` | Strict. Cross origin resources must opt in via CORP or CORS |
| `credentialless` | Lenient. Cross origin requests sent without credentials (no cookies, no client certificates). Easier to adopt than require-corp |

### 12.3 require-corp vs credentialless

**require-corp** is the original mode. Every cross origin resource (image, script, font, iframe) must send a CORP header or be CORS enabled. This is the most secure but hardest to deploy because every third party (CDN, font provider, image host, ad network) must cooperate.

**credentialless** is the newer alternative (Chrome 96+, Firefox 119+). Cross origin resources are loaded without credentials (no cookies). Because the request is anonymous, the resource cannot leak the user's identity, and the browser does not need an explicit opt in from the server. Easier to adopt.

**Bubbles recommendation:** start with `credentialless` if cross origin isolation is needed. Move to `require-corp` only if a specific third party requirement (analytics endpoint that needs cookies, etc) demands it.

### 12.4 How To Build It On Bubbles

For sites that need SharedArrayBuffer or other isolated APIs (rare; mostly WebAssembly applications):

```nginx
location / {
    add_header Cross-Origin-Embedder-Policy "credentialless" always;
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    # CORP on every served resource
}
```

For typical sites that do not need isolated APIs:

```nginx
# Do not set COEP. Default unsafe-none is correct.
```

Most Bubbles sites have no need for cross origin isolation. Do not enable COEP unless a specific feature requires it.

### 12.5 How To Verify

```bash
# 1. Check COEP is set
curl -sI https://example.com/ | grep -i cross-origin-embedder-policy

# 2. Verify cross origin isolation in browser console:
# window.crossOriginIsolated
# Expected: true (when COEP + COOP set correctly)

# 3. Test SharedArrayBuffer availability:
# typeof SharedArrayBuffer
# Expected: "function" when isolated, "undefined" when not

# 4. Check that resources still load (no COEP block errors in console)
```

### 12.6 Troubleshooting

**Symptom: After enabling COEP, half the images on the page fail to load.**
Those images are cross origin without CORP. Either:
1. Self host the images.
2. Add CORP to them (requires control of the source server).
3. Switch to `credentialless` COEP.

**Symptom: window.crossOriginIsolated is false despite setting COEP.**
COEP requires COOP to be `same-origin` AND for the document not to have been opened by a cross origin opener. Verify both headers are set.

**Symptom: Third party scripts fail with NetworkError.**
Those scripts' origins do not send CORP, and the script needs credentials so credentialless does not work either. Move the script to your own origin (proxy it), or accept that you cannot have cross origin isolation while using that script.

### 12.7 How To Fix Common Breakage

**Case: WebAssembly app needs SharedArrayBuffer but it is undefined.**
Enable COEP plus COOP:

```nginx
location /webapp/ {
    add_header Cross-Origin-Embedder-Policy "credentialless" always;
    add_header Cross-Origin-Opener-Policy "same-origin" always;
}
```

Verify in console: `crossOriginIsolated === true`.

**Case: COEP blocks Google Fonts.**
Self host fonts:

```bash
# Download woff2 files from Google Fonts to /var/www/sites/example.com/fonts/
# Reference locally in CSS
```

Or switch to `credentialless`.

---

## 13. CROSS-ORIGIN-OPENER-POLICY (COOP, WHAT THIS DOCUMENT CAN SHARE A WINDOW WITH)

### 13.1 What It Does

`Cross-Origin-Opener-Policy` (COOP) controls whether this document shares a browsing context group with cross origin documents opened via `window.open` or that have opened this document. Required for cross origin isolation. Defined in the WHATWG HTML spec.

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin-allow-popups
Cross-Origin-Opener-Policy: unsafe-none
```

When COOP is `same-origin`:

* If this document opens a cross origin popup via `window.open`, the popup gets a different browsing context group. They cannot communicate via `window.opener`.
* If this document was opened by a cross origin window, the opener relationship is severed: `window.opener` is null.
* This prevents tab nabbing and reduces side channel attack surface.

### 13.2 Values

| Value | Meaning |
|---|---|
| `unsafe-none` | Default. No isolation. Cross origin windows may communicate via window.opener |
| `same-origin-allow-popups` | This document is isolated from cross origin openers, but popups it opens can still communicate back |
| `same-origin` | Strict isolation. Required for cross origin isolation. No cross origin window communication |

### 13.3 How To Build It On Bubbles

For typical Bubbles sites (defense against tab nabbing):

```nginx
add_header Cross-Origin-Opener-Policy "same-origin" always;
```

This is included in the standard `snippets/common-security-headers.conf`.

For sites that need to open cross origin popups (OAuth flows, payment processors):

```nginx
add_header Cross-Origin-Opener-Policy "same-origin-allow-popups" always;
```

The popup can still communicate back via `window.opener.postMessage`, which is what OAuth flows need.

### 13.4 How To Verify

```bash
# 1. Confirm header
curl -sI https://example.com/ | grep -i cross-origin-opener-policy

# 2. Test in browser console:
# Open a popup: window.open("https://example.com", "_blank")
# In the popup: window.opener
# Expected: null when COOP is same-origin

# 3. Verify cross origin isolation status (combined with COEP):
# window.crossOriginIsolated
# Expected: true when COOP same-origin AND COEP require-corp/credentialless
```

### 13.5 Troubleshooting

**Symptom: OAuth popup completes auth but cannot communicate back.**
COOP is `same-origin`. The popup cannot reach `window.opener`.
Fix: change to `same-origin-allow-popups`.

**Symptom: window.crossOriginIsolated is false despite COEP plus COOP.**
Possible causes:
1. COOP is `unsafe-none` or `same-origin-allow-popups` instead of `same-origin`.
2. The document was opened by a cross origin window (COOP cannot retroactively isolate).
3. An iframe in the document is cross origin without proper headers.

**Symptom: window.opener is null in a popup that used to work.**
COOP enforcement is in place. The popup must be from same origin or use postMessage with explicit origin checks.

### 13.6 How To Fix Common Breakage

**Case: OAuth or social login broken after COOP enforcement.**
Use `same-origin-allow-popups`:

```nginx
add_header Cross-Origin-Opener-Policy "same-origin-allow-popups" always;
```

The OAuth popup can communicate back via postMessage.

**Case: Need full cross origin isolation but OAuth must work.**
Move OAuth flow to a separate domain that does not have strict COOP, then transfer the auth result via secure cookie back to the isolated domain.

---

## 14. THE CROSS ORIGIN ISOLATION PATTERN (COEP PLUS COOP PLUS CORP)

### 14.1 What It Achieves

Cross origin isolation unlocks browser APIs that were disabled after Spectre:

* `SharedArrayBuffer`: shared memory between threads.
* `performance.now()` with high precision (nanoseconds instead of millisecond resolution).
* `performance.measureUserAgentSpecificMemory()`.
* WebAssembly threading.

These features are required for sophisticated WebAssembly applications, scientific computing, video editing, and gaming in the browser.

### 14.2 The Three Header Requirement

To achieve cross origin isolation, all three must be true:

1. **COOP: same-origin** on the document.
2. **COEP: require-corp** OR **COEP: credentialless** on the document.
3. **Every cross origin resource has CORP or CORS** allowing this document.

Verify with `window.crossOriginIsolated` in the console.

### 14.3 Full Configuration

```nginx
server {
    listen 443 ssl;
    server_name webapp.example.com;

    # ===== Cross origin isolation =====
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    add_header Cross-Origin-Embedder-Policy "credentialless" always;

    # ===== CORP on all served resources =====
    location ~* \.(jpg|jpeg|png|webp|svg|woff2|css|js|wasm)$ {
        add_header Cross-Origin-Resource-Policy "same-origin" always;
        # ... caching etc ...
    }

    # ===== Permissions-Policy must NOT block cross-origin-isolated =====
    # Default is allow; just ensure we do not block it
    # add_header Permissions-Policy "cross-origin-isolated=(self), camera=(), ..." always;
}
```

### 14.4 How To Verify Cross Origin Isolation

```bash
# 1. Confirm all three headers
curl -sI https://webapp.example.com/ | grep -iE "cross-origin-(opener|embedder|resource)-policy"

# 2. In browser DevTools console on the document:
# window.crossOriginIsolated
# Expected: true

# 3. Verify SharedArrayBuffer is available:
# typeof SharedArrayBuffer
# Expected: "function"

# 4. Verify high precision performance.now:
# performance.now()
# Expected: many decimal places of precision (not capped at millisecond resolution)
```

### 14.5 Rollout Strategy

Cross origin isolation is invasive. Rolling it out site wide breaks anything that loads cross origin resources without CORP.

**Recommended approach:**

1. Scope cross origin isolation to a specific subdomain (e.g. `webapp.example.com`) rather than the whole site.
2. Self host every resource that subdomain needs.
3. Verify in DevTools that no resources are blocked.
4. Confirm `window.crossOriginIsolated === true`.
5. Only then enable the isolated APIs in code.

Most Bubbles client sites do not need cross origin isolation and should not enable COEP/COOP. Apply only to sites that explicitly require SharedArrayBuffer or related APIs.

---

## 15. HOW THESE HEADERS INTERACT

The nine headers form a layered system. Several specific interactions matter.

### 15.1 X-Frame-Options vs CSP frame-ancestors

Both control framing. Modern browsers honor frame-ancestors when both are present and conflict. Older browsers honor X-Frame-Options. Best practice: set both with consistent values.

### 15.2 CSP and Permissions-Policy

CSP controls resource loading. Permissions-Policy controls capability access. They are orthogonal. A page can have a permissive CSP but restrictive Permissions-Policy (or vice versa). Set both to maximize defense.

### 15.3 HSTS and CSP upgrade-insecure-requests

HSTS upgrades the connection. `upgrade-insecure-requests` (CSP) upgrades subresource references in the HTML. They work together: HSTS upgrades the top level request, CSP upgrade-insecure-requests upgrades references within the page.

### 15.4 CORP, COEP, and COOP

These three work together to provide cross origin isolation. None of them alone is sufficient. See Section 14 for the combined pattern.

### 15.5 Referrer-Policy and CSP report-uri

When CSP sends a violation report, the report includes the URL where the violation occurred (document-uri). If Referrer-Policy is `no-referrer`, the destination of the report does not see a Referer header, but the report body itself still contains URLs. The two are independent.

### 15.6 X-Content-Type-Options and CSP

nosniff is an absolute. CSP can layer additional restrictions but nosniff is non negotiable. Always set nosniff.

### 15.7 Permissions-Policy and Cross-Origin-Embedder-Policy

The `cross-origin-isolated` directive in Permissions-Policy can block isolation even when COOP and COEP are set. If you want isolation, do not block it: `Permissions-Policy: cross-origin-isolated=(self)` or just omit the directive.

---

## 16. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per use case.

### 16.1 Standard Bubbles client site (production, public marketing site)

```nginx
# /etc/nginx/snippets/common-security-headers.conf
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), browsing-topics=(), interest-cohort=()" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Resource-Policy "same-site" always;
```

CSP is per site (set separately, not in the shared snippet, because each site's CSP is unique).

### 16.2 Standard CSP for a marketing site with GTM, GA4, AdSense

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://www.google-analytics.com https://www.googleadservices.com https://googleads.g.doubleclick.net https://pagead2.googlesyndication.com https://*.googlesyndication.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: blob: https:; font-src 'self' data: https://fonts.gstatic.com; connect-src 'self' https://*.google-analytics.com https://*.analytics.google.com https://*.googletagmanager.com https://*.doubleclick.net; frame-src 'self' https://www.googletagmanager.com https://www.google.com https://googleads.g.doubleclick.net; frame-ancestors 'self'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests" always;
```

### 16.3 Strict CSP with nonce for FastAPI sidecar dynamic pages

Generated per request in the upstream. nginx passes through:

```nginx
location /dashboard {
    proxy_pass http://127.0.0.1:9090;
    # Sidecar emits CSP with nonce; nginx does not override
}
```

See Section 6.4 for the FastAPI implementation.

### 16.4 Staging or development environment

```nginx
server {
    server_name staging.example.com;

    include snippets/common-security-headers.conf;

    # Block all indexing
    add_header X-Robots-Tag "noindex, nofollow" always;

    # Permissive CSP for debugging
    add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval' https:; report-uri /csp-report" always;

    # Basic auth as additional layer
    auth_basic "Staging";
    auth_basic_user_file /etc/nginx/.staging-htpasswd;
}
```

### 16.5 OAuth or social login callback page

Needs `same-origin-allow-popups` to communicate back from the OAuth popup:

```nginx
location ~ ^/auth/(login|callback) {
    include snippets/common-security-headers-no-coop.conf;
    add_header Cross-Origin-Opener-Policy "same-origin-allow-popups" always;
}
```

### 16.6 WebAssembly application requiring cross origin isolation

```nginx
server {
    server_name webapp.example.com;

    include snippets/common-security-headers-base.conf;

    add_header Cross-Origin-Opener-Policy "same-origin" always;
    add_header Cross-Origin-Embedder-Policy "credentialless" always;

    location / {
        add_header Cross-Origin-Resource-Policy "same-origin" always;
        try_files $uri $uri/ /index.html;
    }
}
```

### 16.7 Public embeddable resources (logos, badges)

```nginx
location /embed/ {
    add_header Cross-Origin-Resource-Policy "cross-origin" always;
    add_header Access-Control-Allow-Origin "*" always;
    # CORS for fonts and CSS specifically
}
```

### 16.8 Healthcare or HIPAA aware site

```nginx
server {
    server_name patient-portal.example.com;

    include snippets/common-security-headers.conf;

    # Stricter referrer policy
    add_header Referrer-Policy "no-referrer" always;

    # Stricter CSP
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests" always;

    # Block all third party tracking
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), browsing-topics=(), interest-cohort=(), attribution-reporting=(), join-ad-interest-group=(), run-ad-auction=(), private-aggregation=()" always;
}
```

### 16.9 User upload serving endpoint (max security)

```nginx
location /user-uploads/ {
    include snippets/common-security-headers.conf;
    # Force download, prevent any execution
    add_header Content-Type "application/octet-stream" always;
    add_header Content-Disposition "attachment" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Content-Security-Policy "default-src 'none'" always;
    add_header Cross-Origin-Resource-Policy "same-origin" always;
}
```

### 16.10 thatwebhostingguy.com wildcard (no preload, isolated demos)

```nginx
server {
    server_name *.thatwebhostingguy.com;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    # NO preload (different demos may not all support HTTPS)

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Permissive Permissions-Policy (demos may need various features)
    add_header Permissions-Policy "camera=(self), microphone=(self), geolocation=(self), interest-cohort=(), browsing-topics=()" always;

    # Wildcard demos: AdSense permitted
    add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval' https:; img-src 'self' data: blob: https:; font-src 'self' data: https:; connect-src 'self' https:; frame-src 'self' https:; frame-ancestors 'self'; base-uri 'self'; object-src 'none'" always;
}
```

---

## 17. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete security header stanza, layered with caching (framework-http-caching-headers.md), content (framework-http-content-headers.md), and SEO (framework-http-seo-headers.md).

```nginx
# /etc/nginx/snippets/common-security-headers.conf
# Shared by every Bubbles client production site

# Transport
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# Content
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Capabilities
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), browsing-topics=(), interest-cohort=()" always;

# Cross-origin (no isolation; just baseline hardening)
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Resource-Policy "same-site" always;
```

```nginx
# /etc/nginx/sites-available/example.com

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    listen 443 quic reuseport;
    listen [::]:443 quic reuseport;
    server_name www.example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    listen 443 quic reuseport;
    listen [::]:443 quic reuseport;
    server_name example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/sites/example.com;
    index index.html;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # ===== HTML PAGES =====
    location ~* \.html$ {
        include snippets/common-security-headers.conf;
        # Per site CSP
        add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://www.google-analytics.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://*.google-analytics.com https://*.googletagmanager.com; frame-ancestors 'self'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests" always;
        # Plus caching, content, SEO headers from other frameworks
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        add_header Content-Language "en-US" always;
        add_header X-Robots-Tag "max-snippet:200, max-image-preview:large, max-video-preview:-1" always;
    }

    # ===== STATIC ASSETS =====
    location ~* \.(css|js|woff2|woff|ttf|otf)$ {
        include snippets/common-security-headers.conf;
        add_header Cross-Origin-Resource-Policy "same-site" always;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }

    # ===== IMAGES =====
    location ~* \.(jpg|jpeg|png|gif|webp|avif|svg|ico)$ {
        include snippets/common-security-headers.conf;
        add_header Cross-Origin-Resource-Policy "same-site" always;
        add_header Cache-Control "public, max-age=2592000" always;
    }

    # ===== PDF DOWNLOADS =====
    location ~* \.pdf$ {
        include snippets/common-security-headers.conf;
        add_header Content-Disposition "attachment" always;
        add_header X-Robots-Tag "noindex" always;
        add_header Cache-Control "public, max-age=86400" always;
    }

    # ===== USER UPLOADS (max security) =====
    location /user-uploads/ {
        include snippets/common-security-headers.conf;
        add_header Content-Type "application/octet-stream" always;
        add_header Content-Disposition "attachment" always;
        add_header Content-Security-Policy "default-src 'none'" always;
        add_header Cross-Origin-Resource-Policy "same-origin" always;
        add_header Cache-Control "public, max-age=86400" always;
    }

    # ===== API ENDPOINTS (FastAPI sidecar) =====
    location /api/ {
        include snippets/common-security-headers.conf;
        add_header X-Robots-Tag "noindex, nofollow" always;
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # CSP and other headers may be overridden by upstream
    }

    # ===== ROOT =====
    location / {
        include snippets/common-security-headers.conf;
        add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://www.google-analytics.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://*.google-analytics.com https://*.googletagmanager.com; frame-ancestors 'self'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests" always;
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        try_files $uri $uri/ $uri.html =404;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

External validation:

```bash
echo "https://observatory.mozilla.org/analyze/example.com"
echo "https://securityheaders.com/?q=https://example.com/"
echo "https://csp-evaluator.withgoogle.com/?csp=https://example.com/"
```

Target: A or A+ on both observatory.mozilla.org and securityheaders.com.

---

## 18. AUDIT CHECKLIST

Run through these 60 items for any production site.

### Strict-Transport-Security

1. [ ] HTTPS responses include `Strict-Transport-Security` header.
2. [ ] HTTP responses do NOT include `Strict-Transport-Security` header.
3. [ ] max-age is at least 31536000 (1 year) for established sites.
4. [ ] `includeSubDomains` is present (after verifying all subdomains support HTTPS).
5. [ ] `preload` is present for sites submitted to or eligible for the preload list.
6. [ ] HTTP requests redirect to HTTPS with 301 (not 302).
7. [ ] Every subdomain serves HTTPS with a valid certificate.
8. [ ] certbot.timer is active and tested.

### Content-Security-Policy

9. [ ] CSP header is present on every HTML response.
10. [ ] `default-src` is set (fallback for unset directives).
11. [ ] `script-src` does not use `'unsafe-inline'` without justification.
12. [ ] `script-src` does not use `'unsafe-eval'` without justification.
13. [ ] If `'strict-dynamic'` is used, a fresh nonce is included on every response.
14. [ ] `object-src 'none'` is set (blocks Flash, deprecated plugins).
15. [ ] `base-uri 'self'` or `'none'` is set.
16. [ ] `form-action` is restricted appropriately.
17. [ ] `frame-ancestors` is set (replaces X-Frame-Options behavior).
18. [ ] `upgrade-insecure-requests` is set (during HTTPS migration).
19. [ ] CSP includes `report-uri` or `report-to` for violation monitoring.
20. [ ] CSP violations are monitored (not just emitted to /dev/null).
21. [ ] CSP rolled out in Report-Only mode first, then enforced.
22. [ ] CSP evaluator (csp-evaluator.withgoogle.com) shows no high severity issues.

### X-Frame-Options

23. [ ] X-Frame-Options is set on every HTML response.
24. [ ] Value is DENY or SAMEORIGIN (not ALLOW-FROM).
25. [ ] CSP frame-ancestors agrees with X-Frame-Options value.

### X-Content-Type-Options

26. [ ] `X-Content-Type-Options: nosniff` is set on every response.
27. [ ] All Content-Type values are accurate (see framework-http-content-headers.md).
28. [ ] User upload endpoints use `application/octet-stream` plus `attachment` disposition.

### Referrer-Policy

29. [ ] Referrer-Policy is set on every response.
30. [ ] Value is `strict-origin-when-cross-origin` or stricter.
31. [ ] Password reset and account pages use `no-referrer` or `same-origin`.

### Permissions-Policy

32. [ ] Permissions-Policy is set on every response.
33. [ ] Sensitive APIs (camera, microphone, geolocation, payment, USB, Bluetooth) are disabled unless needed.
34. [ ] `browsing-topics=()` is set (opt out of Topics tracking).
35. [ ] `interest-cohort=()` is set (historical FLoC opt out; harmless).

### Cross-Origin-Resource-Policy

36. [ ] CORP is set on assets, value appropriate to the use case.
37. [ ] Private user content uses `same-origin`.
38. [ ] Brand assets use `same-site`.
39. [ ] Publicly embeddable assets use `cross-origin`.

### Cross-Origin-Embedder-Policy

40. [ ] COEP is NOT set unless cross origin isolation is required.
41. [ ] If COEP is set, COOP is also set to `same-origin`.
42. [ ] If COEP is set, every cross origin resource has CORP or CORS.

### Cross-Origin-Opener-Policy

43. [ ] COOP is set on HTML responses.
44. [ ] Value is `same-origin` for typical sites.
45. [ ] OAuth flows use `same-origin-allow-popups`.

### Cross cutting

46. [ ] `nginx -t` passes without warnings.
47. [ ] `nginx -T` (effective config) shows all expected directives.
48. [ ] observatory.mozilla.org grade is A or A+.
49. [ ] securityheaders.com grade is A or A+.
50. [ ] CSP evaluator (csp-evaluator.withgoogle.com) shows no high severity issues.
51. [ ] Browser DevTools console shows no CSP violations on production.
52. [ ] All headers are sent with `always` (apply to error responses too).
53. [ ] Snippet include pattern is used (no add_header inheritance trap).
54. [ ] Headers consistent across all server blocks for the domain.
55. [ ] Headers consistent across HTTP/2 and HTTP/3.
56. [ ] Headers consistent across direct requests and FastAPI sidecar passthrough.
57. [ ] No header value contains user input without sanitization.
58. [ ] No header value contains protocol relative URLs.
59. [ ] No deprecated headers present (Public-Key-Pins, Expect-CT, Feature-Policy).
60. [ ] Site responds correctly to security scanner without producing false errors.

A site that passes all 60 has correctly configured security and trust signal headers and will earn A or A+ grades from external scanners.

---

## 19. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: HSTS preload submitted before verifying subdomains.**
Symptom: critical internal subdomain inaccessible permanently.
Why it breaks: preload list takes months to remove from. Any subdomain serving only HTTP after preload is locked out.
Fix prevention: audit every subdomain over HTTPS before submitting. Fix any HTTP only subdomain first.

**Pitfall 2: CSP enforced without Report-Only test phase.**
Symptom: GTM scripts blocked, analytics offline, third party widgets broken.
Why it breaks: real world resource graphs always include unexpected origins.
Fix prevention: Report-Only for at least a week, fix every violation, then enforce.

**Pitfall 3: 'unsafe-inline' in script-src.**
Symptom: XSS injection executes despite CSP.
Why it breaks: `'unsafe-inline'` allows any inline script, defeating the primary protection.
Fix: replace with nonce + strict-dynamic pattern. If pure static site, accept the lower security level.

**Pitfall 4: X-Frame-Options without CSP frame-ancestors.**
Symptom: scanner reports framing protection incomplete; some browsers do not enforce X-Frame-Options.
Why it breaks: X-Frame-Options is being deprecated; CSP frame-ancestors is the authoritative replacement.
Fix: set both for belt and suspenders.

**Pitfall 5: nosniff missing on user upload serving endpoint.**
Symptom: stored XSS via uploaded HTML file.
Why it breaks: browser sniffs uploaded file as HTML, renders it, runs embedded scripts.
Fix: set nosniff site wide AND serve user uploads with `application/octet-stream` plus `attachment` disposition.

**Pitfall 6: Referrer-Policy unset, default leaks tokens.**
Symptom: password reset tokens or session IDs appear in third party analytics logs.
Why it breaks: default Referer header includes full URL with query string.
Fix: set `strict-origin-when-cross-origin` site wide; `no-referrer` on sensitive endpoints.

**Pitfall 7: Permissions-Policy blocks legitimate features.**
Symptom: geolocation, camera, or other API silently refused.
Why it breaks: directive value is `()` (block) instead of `(self)` or appropriate allowlist.
Fix: set appropriate allowlist for features the site actually uses.

**Pitfall 8: COEP enabled, all cross origin images blocked.**
Symptom: layout broken, half the images missing.
Why it breaks: third party CDN images do not have CORP or CORS.
Fix: switch COEP to `credentialless`, OR self host the resources, OR add CORP at source.

**Pitfall 9: COOP same-origin breaks OAuth flow.**
Symptom: OAuth popup completes auth but cannot return data.
Why it breaks: same-origin breaks window.opener.
Fix: use `same-origin-allow-popups` instead.

**Pitfall 10: HSTS on HTTP server block.**
Symptom: nothing breaks but scanner notes anomaly.
Why it breaks: HSTS over HTTP is ignored by the spec but indicates configuration confusion.
Fix: remove HSTS from HTTP server blocks. HTTP server should only redirect.

**Pitfall 11: CSP with no fallback (no default-src).**
Symptom: some resource types unrestricted because their directive was not set.
Why it breaks: directives not explicitly set inherit from default-src; without default-src, they are unrestricted.
Fix: always set `default-src 'self'` as a baseline.

**Pitfall 12: add_header inheritance trap on security headers.**
Symptom: nosniff disappears on specific file types after a location block override.
Why it breaks: nested `add_header` wipes parent declarations.
Fix: snippet include pattern in every location.

**Pitfall 13: Header value too long for nginx default buffer.**
Symptom: nginx error or response truncated.
Why it breaks: CSP with many origins exceeds default 8 KB header buffer.
Fix: increase `large_client_header_buffers`, or simplify CSP using wildcards.

**Pitfall 14: Permissions-Policy uses old Feature-Policy syntax.**
Symptom: browser warning, header value ignored.
Why it breaks: Permissions-Policy uses `feature=(allowlist)` not `feature 'self'`.
Fix: convert to current syntax.

**Pitfall 15: Mixed HTTP and HTTPS resources block on COEP.**
Symptom: COEP blocks HTTP loaded images, fonts, scripts.
Why it breaks: HTTP resources cannot send CORP or pass CORS checks.
Fix: upgrade all resources to HTTPS; use CSP `upgrade-insecure-requests`.

---

## 20. DIAGNOSTIC COMMANDS

Reference of every command useful for security header investigation.

### Inspect all security headers at once

```bash
# Full security family
curl -sI https://example.com/ | grep -iE "^(strict-transport-security|content-security-policy|x-frame-options|x-content-type-options|referrer-policy|permissions-policy|cross-origin-(opener|embedder|resource)-policy):"

# Pretty print
curl -sI https://example.com/ | grep -iE "^(strict-|content-security|x-|referrer|permissions|cross-origin):" | while read line; do
    KEY=$(echo "$line" | cut -d: -f1)
    VAL=$(echo "$line" | cut -d: -f2- | sed 's/^ //')
    echo ""
    echo "=== $KEY ==="
    echo "$VAL" | tr ';,' '\n' | sed 's/^ */    /'
done
```

### HSTS verification

```bash
# Header presence and content
curl -sI https://example.com/ | grep -i strict-transport-security

# HTTP redirect check
curl -sI http://example.com/ | head -3

# Preload eligibility
curl -s "https://hstspreload.org/api/v2/status?domain=example.com" | python3 -m json.tool

# All subdomains over HTTPS
for sub in $(dig +short ANY example.com | grep -oE "^[a-z0-9-]+\.example\.com\."); do
    sub=${sub%.}
    echo -n "$sub: "
    curl -sI -m 5 "https://$sub/" 2>&1 | head -1 || echo "FAILED"
done
```

### CSP verification

```bash
# Header presence
curl -sI https://example.com/ | grep -i content-security-policy

# Evaluator (Google's checker)
echo "Visit: https://csp-evaluator.withgoogle.com/?csp=https%3A%2F%2Fexample.com%2F"

# Report endpoint test
curl -X POST https://example.com/csp-report \
     -H "Content-Type: application/csp-report" \
     -d '{"csp-report":{"document-uri":"https://example.com/","violated-directive":"script-src","blocked-uri":"https://evil.com/script.js"}}'

# Check for unsafe directives
curl -sI https://example.com/ | grep -i content-security-policy | grep -oE "unsafe-(inline|eval)"
```

### Cross origin verification

```bash
# All three cross origin headers
curl -sI https://example.com/ | grep -iE "^cross-origin-"

# Verify isolation in browser:
# In DevTools console: window.crossOriginIsolated
# Expected: true (if attempting isolation)

# Verify SharedArrayBuffer:
# typeof SharedArrayBuffer === "function"
```

### External scanners

```bash
# Mozilla Observatory
echo "https://observatory.mozilla.org/analyze/example.com"

# Security Headers
echo "https://securityheaders.com/?q=https://example.com/"

# Hardenize (comprehensive)
echo "https://www.hardenize.com/report/example.com"

# Internet.nl
echo "https://internet.nl/site/example.com/"

# Qualys SSL Labs (for TLS, not headers)
echo "https://www.ssllabs.com/ssltest/analyze.html?d=example.com"
```

### Server side investigation

```bash
# All add_header directives in effect
nginx -T 2>/dev/null | grep -iE "add_header.*(strict-transport|content-security|x-frame|x-content-type|referrer|permissions|cross-origin)"

# Confirm includes work
nginx -T 2>/dev/null | grep -i "include.*security"

# Find any conflict (multiple values for same header)
nginx -T 2>/dev/null | grep -i "add_header" | awk '{print $2}' | sort | uniq -c | sort -rn

# Apply changes
nginx -t && systemctl reload nginx
```

### Browser DevTools quick reference

1. **Application or Storage panel**: shows HSTS state (`chrome://net-internals/#hsts`).
2. **Network panel**: shows headers for each request. Click any request, "Headers" tab.
3. **Console panel**: shows CSP violations in red.
4. **Issues panel** (Chrome): shows security issues with explanations.
5. **Security panel**: shows certificate, connection, and origin info.

Useful console commands:

```javascript
// Cross origin isolation status
window.crossOriginIsolated

// Available APIs that require isolation
typeof SharedArrayBuffer
typeof performance.measureUserAgentSpecificMemory

// Trigger a CSP report (for testing report endpoint)
eval("test")   // Will trigger script-src violation if unsafe-eval not allowed
```

---

## 21. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): Cache-Control, ETag, Last-Modified, Expires, Vary, Age. Security headers should be cached with the response; do not cache security headers separately or stripped.
* [framework-http-content-headers.md](framework-http-content-headers.md): Content-Type, Content-Language, Content-Encoding, Content-Length, Content-Disposition. nosniff pairs with strict Content-Type discipline. Content-Disposition is part of the user upload security pattern (Section 16.9 here).
* [framework-http-seo-headers.md](framework-http-seo-headers.md): X-Robots-Tag, Link, Location. Open redirect security in framework-http-seo-headers.md Section 7.4 is part of the overall security posture.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Security headers contribute to E-E-A-T trust signals.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook.
* [framework-csp-deployment.md](framework-csp-deployment.md): deeper dive into CSP rollout strategy, nonce architecture, GTM integration.
* [framework-tls-config.md](framework-tls-config.md): TLS cipher selection, certificate management, OCSP stapling. Pairs with HSTS for transport security.
* Google CSP documentation: https://developers.google.com/tag-platform/security/guides/csp
* web.dev strict CSP guide: https://web.dev/articles/strict-csp
* HSTS preload list: https://hstspreload.org/
* MDN security headers reference: https://developer.mozilla.org/en-US/docs/Web/Security
* Mozilla Observatory: https://observatory.mozilla.org/
* securityheaders.com: https://securityheaders.com/
* CSP Evaluator: https://csp-evaluator.withgoogle.com/
* RFC 6797 (HSTS): https://www.rfc-editor.org/rfc/rfc6797
* RFC 7034 (X-Frame-Options): https://www.rfc-editor.org/rfc/rfc7034
* W3C CSP Level 3: https://www.w3.org/TR/CSP3/
* W3C Referrer Policy: https://www.w3.org/TR/referrer-policy/
* W3C Permissions Policy: https://www.w3.org/TR/permissions-policy/

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Bubbles default security header stack

```nginx
# /etc/nginx/snippets/common-security-headers.conf
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), browsing-topics=(), interest-cohort=()" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Resource-Policy "same-site" always;
```

### Standard CSP for Bubbles client site with GTM and GA4

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://www.google-analytics.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://*.google-analytics.com https://*.googletagmanager.com; frame-ancestors 'self'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests
```

### Header purpose table

| Header | One line purpose |
|---|---|
| Strict-Transport-Security | Force HTTPS forever |
| Content-Security-Policy | Allow list of where scripts and resources may come from |
| X-Frame-Options | Block clickjacking (legacy) |
| X-Content-Type-Options | Block MIME sniffing |
| Referrer-Policy | Control what URL info leaks to other origins |
| Permissions-Policy | Disable sensitive browser APIs |
| Cross-Origin-Resource-Policy | Who may load this resource |
| Cross-Origin-Embedder-Policy | What this document may embed |
| Cross-Origin-Opener-Policy | What this document may share a window with |

### CSP value cheat sheet

| Want | Use |
|---|---|
| Only same origin scripts | `script-src 'self'` |
| Plus inline scripts safely | `script-src 'self' 'nonce-RANDOM' 'strict-dynamic'` |
| Plus GTM | `script-src 'self' 'unsafe-inline' https://www.googletagmanager.com` |
| Block all framing | `frame-ancestors 'none'` |
| Same site framing | `frame-ancestors 'self'` |
| Specific partner | `frame-ancestors 'self' https://partner.example.com` |
| No plugins ever | `object-src 'none'` |
| Auto upgrade HTTP refs | `upgrade-insecure-requests` |

### Five commands every operator should know

```bash
# 1. View all security headers
curl -sI https://example.com/ | grep -iE "^(strict|content-security|x-|referrer|permissions|cross-origin):"

# 2. Check observatory score
echo "Visit: https://observatory.mozilla.org/analyze/example.com"

# 3. Check securityheaders.com score
echo "Visit: https://securityheaders.com/?q=https://example.com/"

# 4. Evaluate CSP
echo "Visit: https://csp-evaluator.withgoogle.com/?csp=https://example.com/"

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Verify HSTS is enforced
curl -sI https://example.com/ | grep -i strict-transport-security
# Expected: strict-transport-security: max-age=63072000; includeSubDomains; preload

# 2. Verify no CSP unsafe directives (or that they are justified)
curl -sI https://example.com/ | grep -i content-security-policy | grep -oE "unsafe-(inline|eval)"
# If output appears, document why or remove

# 3. Verify nosniff and X-Frame-Options site wide
for path in / /about /services /blog /contact; do
    HEADERS=$(curl -sI "https://example.com$path" | grep -iE "x-content-type-options|x-frame-options")
    echo "=== $path ==="
    echo "$HEADERS"
done
# Both headers should appear on every page
```

If all three produce expected output AND observatory.mozilla.org returns A or A+, the security header stack is correctly wired.

---

End of framework-http-security-headers.md.
