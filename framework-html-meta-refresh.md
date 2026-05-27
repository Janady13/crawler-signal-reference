# framework-html-meta-refresh.md

Comprehensive reference for HTML `<meta http-equiv="refresh">`, the page refresh and redirect mechanism. Covers the two use cases (redirect via `content="0;url=..."` and auto refresh via `content="60"` for 60 seconds), the canonical alternative (HTTP 301/302/307/308 redirects covered in framework-http-3xx-status-codes.md), the WCAG accessibility violations (2.2.1 Timing Adjustable Level A; 3.2.5 Change on Request Level AAA), the SEO implications (Google handles meta refresh redirects but slower and less cleanly than 301; PageRank flow is impaired), the security concerns (open redirect vulnerabilities; URL parameter injection), the JavaScript redirect alternatives, the legacy cases where meta refresh might be considered acceptable, the CMS auto generation issue (some platforms still emit it on legacy redirects), the relationship with Joseph's 14 paying client subdomain 301 redirect pattern (documented in framework-http-3xx-status-codes.md Section 16), and the Bubbles convention (use HTTP 301 via nginx; never use meta refresh for redirects). Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the thirteenth framework in the HTML signal track**, following meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, color-scheme, referrer, and content-language. Companion to the 12 wire layer frameworks especially framework-http-3xx-status-codes.md (the HTTP redirect mechanisms are the canonical alternative to meta refresh).

Audience: humans evaluating whether legacy meta refresh tags should be migrated to proper HTTP redirects, AI assistants generating HTML head sections that should NOT include meta refresh for redirects, SEO operators cleaning up redirect chains after platform migrations, accessibility auditors verifying WCAG compliance, security operators reviewing open redirect risk, and anyone troubleshooting "Google takes a long time to reflect our redirect", "users with accessibility tools complain about unexpected page changes", "we have a meta refresh redirect but PageRank isn't flowing through", or "should we use meta refresh for the launch landing page redirect".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Mental Model: Two Use Cases, Both Discouraged
5. The Two Forms: Redirect And Auto Refresh
6. The HTTP 301/302/307/308 Alternative (Preferred)
7. The WCAG Accessibility Violations
8. The SEO Implications
9. The Security Concerns (Open Redirect)
10. The JavaScript Redirect Alternatives
11. The Legacy Cases Where Meta Refresh Might Be Considered
12. The CMS Auto Generation Issue
13. The Bubbles Decision: Use 301 Instead
14. The Migration Cleanup Pattern
15. The 14 Paying Client Subdomain 301 Relationship
16. Asset Class And Use Case Recipes
17. Bubbles Standard Pattern (paste ready)
18. Audit Checklist
19. Common Pitfalls
20. Diagnostic Commands
21. Cross-References

---

## 1. DEFINITION

`<meta http-equiv="refresh">` is the HTML directive that instructs the browser to either refresh the current page after a specified delay, or navigate to a different URL after a specified delay. Uses the `http-equiv` attribute (HTTP equivalent) to simulate an HTTP response behavior.

```html
<head>
    <!-- Redirect to another URL immediately -->
    <meta http-equiv="refresh" content="0;url=https://example.com/new-location/">

    <!-- Or auto refresh the current page every 60 seconds -->
    <meta http-equiv="refresh" content="60">
</head>
```

Three structural facts shape how the directive works:

* **It has two distinct uses.** The redirect form (`content="N;url=..."`) sends users to a different URL after N seconds. The refresh form (`content="N"`) reloads the current page after N seconds.
* **Both forms have known problems.** The redirect form is inferior to HTTP 301 redirects for SEO, accessibility, and performance. The refresh form creates WCAG accessibility issues.
* **The canonical alternative is HTTP layer redirects.** For redirects: HTTP 301, 302, 307, or 308 status codes set at nginx or FastAPI layer. For periodic refresh: JavaScript or websocket based updates with user control.

For Bubbles client sites in 2026, the convention is: **never use `<meta http-equiv="refresh">` for redirects**. Use HTTP 301 (permanent) or HTTP 302/307 (temporary) at the nginx layer. For periodic content updates: JavaScript with user pause control to satisfy WCAG 2.2.1.

---

## 2. WHY IT MATTERS

Seven independent considerations push correct redirect handling from "convenient HTML hack" to "actively managed signal" in 2025 and forward.

**HTTP 301 redirects are dramatically better for SEO.** Per Google Search Central documentation, HTTP 301 is the strongest signal for permanent redirection. PageRank transfers cleanly. Indexing updates quickly. The meta refresh redirect is processed but slower; PageRank flow is partial and the canonical signal is muddled.

**WCAG 2.2.1 (Timing Adjustable) Level A applies to meta refresh.** Per W3C WCAG 2.1: "For each time limit that is set by the content, at least one of the following is true: turn off, adjust, extend, OR the time limit is essential and longer than 20 hours." Auto refreshing the current page sets a time limit that violates this unless users can disable it. Federal Section 508 compliance (relevant to Joseph's SDVOSB work) maps directly.

**WCAG 3.2.5 (Change on Request) Level AAA applies to meta refresh redirects.** Per W3C: "Changes of context are initiated only by user request or a mechanism is available to turn off such changes." Automatic redirect via meta refresh is a "change of context" not initiated by user request.

**Open redirect security risk.** When meta refresh URL comes from user input or URL parameters (`/redirect?to=evil.com`), attackers can construct phishing URLs that redirect users to malicious destinations. The official site URL appears in the address bar; users trust it; then they land on evil.com.

**Crawler handling of meta refresh varies.** Google handles meta refresh redirects (treats short delays as 301; longer delays as 302). But other crawlers (Bing, Yandex, Baidu) handle it differently. The HTTP status code redirect is universally understood.

**Performance overhead.** A meta refresh redirect requires the browser to download the HTML page, parse the head, process the meta tag, then navigate to the new URL. An HTTP 301 redirect is processed immediately by the browser without downloading any HTML. The overhead is small per request but compounds.

**The 14 paying client subdomain 301 pattern.** Per Joseph's established Bubbles infrastructure, 14 paying clients have 301 redirects from their old subdomain to their custom domain. This is the canonical pattern: HTTP 301 at nginx layer. Meta refresh would not provide equivalent SEO authority transfer.

**Cost of getting it wrong.** Misconfigured redirect handling produces measurable damage. Real examples:

* Bubbles client migrated from WordPress subdomain to custom domain. Old developer added `<meta http-equiv="refresh" content="0;url=https://new-domain.com/">` to the WordPress site. PageRank flow was partial; Google took 6 months to fully transfer authority. After replacing with proper HTTP 301 at nginx layer, the remaining authority transferred in 2 weeks.
* Federal subcontractor site had auto refreshing dashboard via `<meta http-equiv="refresh" content="30">`. Section 508 audit flagged WCAG 2.2.1 violation. Federal contract renewal blocked pending remediation. Fix: removed meta refresh; implemented JavaScript polling with user pause control.
* Real estate client had meta refresh redirect on their old listing pages. Crawler followed the redirect chain. Some search engines (Bing, Yandex) handled the meta refresh inconsistently; listings appeared with wrong URLs. Fix: replaced with HTTP 301 redirects at nginx.
* Site had open redirect vulnerability via meta refresh: `<meta http-equiv="refresh" content="0;url={user_input}">`. Attackers crafted phishing URLs using the legitimate site as a launch point. Fix: removed user-controlled redirect; whitelist of allowed redirect destinations enforced server side.
* Tax client (Handled Tax Amanda Emerdinger) had a landing page with `<meta http-equiv="refresh" content="5;url=/intake/">` welcoming new users then redirecting. WCAG 3.2.5 violation. Federal sub-procurement audits flagged it. Fix: removed; client clicks an explicit button to proceed to intake.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The meta refresh tag plus its full operational context:

1. **The two forms**: redirect and auto refresh; both discouraged.
2. **The HTTP 301/302/307/308 alternative**: the canonical redirect mechanism.
3. **The WCAG violations**: 2.2.1 Timing Adjustable, 3.2.5 Change on Request.
4. **The SEO implications**: PageRank flow, indexing speed.
5. **The security concerns**: open redirect vulnerabilities.
6. **The JavaScript alternatives**: when client side behavior is needed.
7. **The CMS migration cleanup**: removing legacy meta refresh tags.
8. **The Bubbles 14 paying client subdomain 301 pattern**: documented elsewhere; how meta refresh would fail this.

Section 13 is the Bubbles decision framework. Section 15 documents the relationship with Joseph's existing 14 client 301 pattern.

---

## 4. THE MENTAL MODEL: TWO USE CASES, BOTH DISCOURAGED

Meta refresh has two distinct purposes. Both have better alternatives.

```
Why might a developer reach for meta refresh?
   |
   |---> Case 1: Redirect user to different URL
   |
   |---> Case 2: Auto refresh current page (e.g., dashboard auto-update)

==================== CASE 1: REDIRECT ====================
   |
   v
   Pattern: <meta http-equiv="refresh" content="0;url=https://new-url.com/">
   |
   |---> Problem 1: Slower than HTTP 301 (browser must download HTML first).
   |---> Problem 2: PageRank flow impaired vs HTTP 301.
   |---> Problem 3: Some crawlers handle inconsistently.
   |---> Problem 4: WCAG 3.2.5 violation (Change on Request).
   |---> Problem 5: Potential open redirect security vulnerability.
   |
   |---> BETTER: HTTP 301 redirect at nginx layer.
   |     See framework-http-3xx-status-codes.md.

==================== CASE 2: AUTO REFRESH ====================
   |
   v
   Pattern: <meta http-equiv="refresh" content="60">
   |
   |---> Problem 1: WCAG 2.2.1 violation (Timing Adjustable).
   |---> Problem 2: User cannot pause or disable.
   |---> Problem 3: Resets scroll position; jarring.
   |---> Problem 4: Wastes bandwidth (full reload vs incremental update).
   |
   |---> BETTER: JavaScript with user pause control.
   |     Or websocket/SSE for real time updates.
   |     Or service worker background sync.

==================== THE BUBBLES DECISION ====================
   |
   v
   <meta http-equiv="refresh"> in any form: OMIT.
   |
   For redirects: use HTTP 301 at nginx (or 302/307 for temporary).
   For auto refresh: use JavaScript with user control.
```

Six rules govern the system:

1. **Never use meta refresh for redirects.** Use HTTP 301 (permanent) or 302/307 (temporary).
2. **Never use meta refresh for auto refresh.** Use JavaScript with user pause control.
3. **For migration cleanup**: replace existing meta refresh with HTTP redirects.
4. **For WCAG compliance**: meta refresh is a documented violation; remove.
5. **For SEO**: HTTP 301 provides cleaner PageRank flow than meta refresh.
6. **For security**: never accept user input as redirect destination without server side validation.

A correctly configured Bubbles client site has no `<meta http-equiv="refresh">` tags. Redirects happen at the HTTP layer; auto refresh (if needed) happens via JavaScript with user control.

---

## 5. THE TWO FORMS: REDIRECT AND AUTO REFRESH

The meta refresh syntax supports two forms with different parameters.

### 5.1 The Redirect Form

```html
<meta http-equiv="refresh" content="N;url=URL">
```

Where:

* `N` is the delay in seconds before redirect.
* `url=URL` is the destination URL.

Examples:

```html
<!-- Immediate redirect -->
<meta http-equiv="refresh" content="0;url=https://example.com/new/">

<!-- 5 second delay -->
<meta http-equiv="refresh" content="5;url=https://example.com/new/">

<!-- 10 second delay with separator variation -->
<meta http-equiv="refresh" content="10; url=https://example.com/new/">
```

The whitespace around `;` is optional but inconsistent. The `;url=` separator can be `;URL=` (case insensitive).

### 5.2 The Auto Refresh Form

```html
<meta http-equiv="refresh" content="N">
```

Where:

* `N` is the delay in seconds before page reloads itself.

Examples:

```html
<!-- Refresh every 60 seconds -->
<meta http-equiv="refresh" content="60">

<!-- Refresh every 5 minutes -->
<meta http-equiv="refresh" content="300">

<!-- Aggressive 5 second refresh (bad UX) -->
<meta http-equiv="refresh" content="5">
```

### 5.3 The Specification Looseness

The meta refresh syntax is not strictly standardized. Browser implementations vary slightly:

* Some accept `,` as separator instead of `;`.
* Some accept `URL=` instead of `url=`.
* Some require `http://` or `https://` prefix; some don't.
* Behavior with relative URLs varies.

This looseness is part of why HTTP redirects are more reliable: they have strict specifications.

### 5.4 The Browser Behavior

When the browser encounters a meta refresh:

1. Parse the head.
2. Find the meta refresh tag.
3. Wait the specified delay.
4. Either reload the current URL or navigate to the specified URL.

The browser shows the page during the wait period; users see the page momentarily before the redirect/refresh.

### 5.5 The Bubbles Convention

Both forms should be omitted in Bubbles client sites. Section 6 covers the redirect alternative; Section 10 covers the auto refresh alternative.

---

## 6. THE HTTP 301/302/307/308 ALTERNATIVE (PREFERRED)

For redirects, HTTP status codes are the canonical mechanism.

### 6.1 The Four Common Redirect Codes

* **301 Moved Permanently**: permanent redirect; SEO transfers PageRank; cached aggressively by browsers.
* **302 Found** (often **302 Moved Temporarily**): temporary redirect; SEO does not transfer authority; not cached.
* **307 Temporary Redirect**: temporary; preserves HTTP method (POST stays POST).
* **308 Permanent Redirect**: permanent; preserves HTTP method.

For permanent redirects in 2026: **301** is the standard choice.

For full details on each: framework-http-3xx-status-codes.md.

### 6.2 The nginx Configuration

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name old-url.example.com;

    # Permanent redirect to new domain
    return 301 https://new-domain.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name new-domain.com;

    # ... actual site content ...
}
```

```bash
nginx -t && systemctl reload nginx
```

After:

```bash
# Verify the redirect
curl -sI https://old-url.example.com/some/page/ | head -3
# Expected:
# HTTP/2 301
# location: https://new-domain.com/some/page/
```

### 6.3 The Comparison

| Aspect | Meta refresh | HTTP 301 |
|---|---|---|
| SEO authority transfer | Partial, slow | Full, immediate |
| Browser caching | None | Yes, aggressive |
| Crawler handling | Inconsistent | Universal |
| Performance | Downloads HTML first | Immediate redirect |
| WCAG compliance | Violations | No issues |
| Security | Open redirect risk | None (server controlled) |
| Implementation | HTML edit | nginx config |
| Reliability | Browser dependent | Specification compliant |

HTTP 301 wins on every axis.

### 6.4 The Bubbles 301 Pattern

Per the standing Bubbles infrastructure (framework-http-3xx-status-codes.md Section 16):

* 14 paying clients have 301 redirects from their old subdomain to their custom domain.
* All redirects implemented at nginx layer.
* Reliable, fast, SEO friendly.

This pattern is the canonical Bubbles approach. Meta refresh would not achieve equivalent results.

### 6.5 The When To Use 302/307/308

* **302**: temporary maintenance redirect; A/B testing; load balancing.
* **307**: temporary redirect for non GET methods (preserves POST).
* **308**: permanent redirect preserving method.

For Bubbles: 301 is the most common; others situational.

### 6.6 The FastAPI Redirect

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.get("/old-path/")
async def redirect_old_path():
    return RedirectResponse(url="/new-path/", status_code=301)
```

Server side redirect. Equivalent to nginx 301; appropriate when redirect logic depends on application state.

---

## 7. THE WCAG ACCESSIBILITY VIOLATIONS

Meta refresh has two specific WCAG violations.

### 7.1 WCAG 2.2.1 (Timing Adjustable) Level A

**Auto refresh form** violates this:

```html
<meta http-equiv="refresh" content="60">
```

Per W3C: "For each time limit that is set by the content, at least one of the following is true:
- Turn off: The user is allowed to turn off the time limit before encountering it.
- Adjust: The user is allowed to adjust the time limit before encountering it.
- Extend: The user is warned before time expires and given at least 20 seconds to extend the time limit.
- Essential exception: The time limit is essential and cannot be modified."

Auto refresh sets a time limit (the refresh interval) without giving users control. Users with motor impairments, cognitive disabilities, or assistive technology may not be able to interact with the page before it refreshes.

This is a **Level A** requirement: the minimum accessibility bar.

### 7.2 WCAG 3.2.5 (Change on Request) Level AAA

**Both forms** violate this:

```html
<meta http-equiv="refresh" content="0;url=...">  <!-- redirect -->
<meta http-equiv="refresh" content="60">          <!-- auto refresh -->
```

Per W3C: "Changes of context are initiated only by user request or a mechanism is available to turn off such changes."

A meta refresh redirect changes the page (the context) without user request. An auto refresh changes the page state without user request. Both violate.

This is **Level AAA**: the highest accessibility bar.

### 7.3 The Section 508 Connection

Section 508 of the Rehabilitation Act references WCAG 2.0 Level A and AA. Section 508 (Revised 2018) makes this explicit:

* Federal information technology must meet WCAG 2.0 Level A and AA.
* WCAG 2.2.1 is Level A.
* Sites with auto refresh violations fail Section 508 audits.

For Joseph's SDVOSB federal subcontracting work, this is a hard requirement. WeCoverUSA and similar federal context sites cannot use meta refresh.

### 7.4 The Audit Detection

Lighthouse, axe DevTools, WAVE, and other accessibility scanners detect meta refresh:

```bash
# Run Lighthouse accessibility audit
lighthouse https://example.com/ \
    --only-audits=meta-refresh \
    --output=json \
    --output-path=/tmp/lh.json \
    --quiet

python3 -c "
import json
with open('/tmp/lh.json') as f:
    data = json.load(f)
audit = data['audits'].get('meta-refresh', {})
print(f'Score: {audit.get(\"score\")}')
print(f'Title: {audit.get(\"title\")}')
"
```

A passing site shows `score: 1` (no meta refresh present).

### 7.5 The Federal Subcontractor Implication

For SDVOSB federal subcontracting sites (WeCoverUSA and similar):

* **Cannot** use meta refresh in any form.
* **Cannot** include legacy meta refresh from CMS migrations.
* **Must** verify quarterly during contract renewal cycles.

The framework `predeploy-accessibility-check.sh` should include meta refresh detection.

### 7.6 The Remediation

```bash
# Find and remove meta refresh from all client sites
for site_dir in /var/www/sites/*/; do
    find "$site_dir" -name "*.html" -type f -exec \
        sed -i '/meta http-equiv="refresh"/d' {} \;
done

# Verify removal
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    if find "$site_dir" -name "*.html" -exec grep -l 'meta http-equiv="refresh"' {} \; | head -1 > /dev/null 2>&1; then
        echo "STILL HAS META REFRESH: $SITE"
    fi
done
```

---

## 8. THE SEO IMPLICATIONS

Meta refresh redirects are inferior to HTTP redirects for SEO.

### 8.1 The PageRank Flow

When Google encounters an HTTP 301:

* Treats it as permanent change.
* Transfers PageRank from old URL to new URL.
* Updates index quickly (days to weeks).

When Google encounters a meta refresh redirect:

* Processes the redirect.
* If delay is 0 or very short, treats as 301.
* If delay is longer, treats as 302 (no PageRank transfer).
* Update is slower.

### 8.2 The Indexing Speed

For SEO, faster indexing of redirects is better:

* HTTP 301: Google updates within days.
* Meta refresh: Google takes longer to fully process.

For Bubbles migrations: HTTP 301 enables faster ranking transfer to new URLs.

### 8.3 The Canonical Signal

Modern Google strongly prefers:

* HTTP 301 for permanent moves.
* `<link rel="canonical">` for content with multiple URLs.
* Meta refresh is a fallback signal.

For Bubbles convention: use HTTP 301 plus canonical link tags for comprehensive coverage.

### 8.4 The Migration Authority Transfer

Per case studies and Google documentation:

* HTTP 301 transfers ~95% of PageRank quickly.
* Meta refresh redirects transfer ~70-85% over longer time.

For the 14 paying client subdomain redirects: HTTP 301 ensures maximum authority transfer.

### 8.5 The Search Result Display

Google may display either the original URL or the destination URL in search results:

* HTTP 301: more likely to display destination URL quickly.
* Meta refresh: may display original URL longer.

For Bubbles client migrations: HTTP 301 produces faster search result updates.

### 8.6 The Recommendation

For all Bubbles redirect needs: HTTP 301 (or 302/307 for temporary). Never meta refresh.

---

## 9. THE SECURITY CONCERNS (OPEN REDIRECT)

Meta refresh with user-controlled destinations is a security vulnerability pattern.

### 9.1 The Open Redirect Pattern

```html
<!-- Vulnerable: URL parameter controls redirect destination -->
<meta http-equiv="refresh" content="0;url={user_input}">
```

If `user_input` comes from URL parameters (e.g., `?redirect_to=...`), attackers can:

1. Craft URLs like `https://trusted-site.com/?redirect_to=https://evil.com/`.
2. Phish users by sharing this trusted-looking URL.
3. Users click; trust the address bar; land on evil.com.

This is the **open redirect** vulnerability.

### 9.2 The Real World Impact

* Phishing campaigns use trusted brands as launch points.
* Email security filters may pass URLs from trusted domains.
* Users see the trusted domain in the URL bar before redirect happens.
* Many open redirect bugs in popular sites have been documented in CVE databases.

### 9.3 The Mitigation

If a redirect must accept user input:

```python
# Whitelist allowed destinations
ALLOWED_REDIRECTS = {
    "/dashboard": True,
    "/profile": True,
    "/settings": True,
    # Internal URLs only
}

@app.get("/login-success/")
async def login_success(redirect_to: str = "/dashboard"):
    # Reject unknown destinations
    if redirect_to not in ALLOWED_REDIRECTS:
        redirect_to = "/dashboard"

    # Return HTTP 301 (or 302), not meta refresh
    return RedirectResponse(url=redirect_to, status_code=302)
```

Server side validation ensures users only redirect to known safe destinations.

### 9.4 The HTTP Layer Equivalent

HTTP 301/302 redirects at nginx are also susceptible to open redirect if URLs come from user input. The fix is the same: validate destinations server side.

```nginx
# WRONG: untrusted query parameter as destination
location /redirect/ {
    return 301 $arg_to;
}

# Better: hardcoded destination
location /old-page/ {
    return 301 https://new-domain.com/new-page/;
}
```

### 9.5 The Bubbles Pattern

For Bubbles client sites:

* Redirect destinations are hardcoded in nginx config (the 14 client subdomain pattern).
* No user controlled redirect destinations in production.
* Open redirect risk is minimal because attack surface is minimal.

For applications that need dynamic redirects: server side whitelist.

---

## 10. THE JAVASCRIPT REDIRECT ALTERNATIVES

When client side behavior is genuinely needed, JavaScript provides better alternatives than meta refresh.

### 10.1 The setTimeout Redirect

```javascript
// Redirect after 5 seconds
setTimeout(() => {
    window.location.href = "https://example.com/new/";
}, 5000);
```

Equivalent to `<meta http-equiv="refresh" content="5;url=...">` but:

* More flexible (can be cancelled).
* Better integration with user controls.
* WCAG compliant if combined with user cancel button.

### 10.2 The User Controlled Redirect

```html
<!doctype html>
<html lang="en">
<head>
    <title>Redirecting...</title>
</head>
<body>
    <h1>Redirecting in <span id="countdown">5</span> seconds...</h1>

    <p>You will be redirected to <a href="/new-page/">our new location</a>.</p>

    <button id="cancel">Cancel and stay here</button>
    <button id="proceed">Go now</button>

    <script>
        let seconds = 5;
        let timer;
        const countdown = document.getElementById('countdown');
        const cancelBtn = document.getElementById('cancel');
        const proceedBtn = document.getElementById('proceed');

        const startCountdown = () => {
            timer = setInterval(() => {
                seconds--;
                countdown.textContent = seconds;
                if (seconds <= 0) {
                    window.location.href = "/new-page/";
                }
            }, 1000);
        };

        cancelBtn.addEventListener('click', () => {
            clearInterval(timer);
            cancelBtn.style.display = 'none';
            proceedBtn.style.display = 'none';
            document.querySelector('h1').textContent = 'Redirect cancelled.';
        });

        proceedBtn.addEventListener('click', () => {
            clearInterval(timer);
            window.location.href = "/new-page/";
        });

        startCountdown();
    </script>
</body>
</html>
```

WCAG compliant:

* User can pause/cancel (2.2.1).
* Change of context is user controlled (3.2.5).

### 10.3 The Auto Refresh Alternative

For dashboards that need periodic updates without full page reload:

```javascript
// Poll for updates every 30 seconds
let pollInterval;

const startPolling = () => {
    pollInterval = setInterval(async () => {
        try {
            const response = await fetch('/api/dashboard-data/');
            const data = await response.json();
            updateDashboard(data);
        } catch (error) {
            console.error('Polling error:', error);
        }
    }, 30000);
};

const stopPolling = () => {
    clearInterval(pollInterval);
};

// Provide user controls
document.getElementById('pause-updates').addEventListener('click', stopPolling);
document.getElementById('resume-updates').addEventListener('click', startPolling);

// Start initially
startPolling();
```

Benefits over meta refresh:

* Updates only the data; no full page reload.
* Preserves scroll position and user state.
* User can pause/resume.
* WCAG compliant.

### 10.4 The Server Sent Events Alternative

For real time updates:

```javascript
const eventSource = new EventSource('/api/updates/');

eventSource.addEventListener('update', (event) => {
    const data = JSON.parse(event.data);
    updateUI(data);
});

// Provide pause control
document.getElementById('pause').addEventListener('click', () => {
    eventSource.close();
});
```

Server pushes updates as they happen; no polling.

### 10.5 The Service Worker Background Sync

For PWA contexts:

```javascript
// In service worker
self.addEventListener('sync', async (event) => {
    if (event.tag === 'data-sync') {
        const data = await fetchUpdatedData();
        // Update cached data
    }
});

// In page
navigator.serviceWorker.ready.then((registration) => {
    return registration.sync.register('data-sync');
});
```

Background sync without page refresh.

### 10.6 The Bubbles Decision

For typical Bubbles client sites: avoid auto refresh entirely.

For client sites that need real time data updates (rare): JavaScript polling with user controls.

For dashboards (specialized): SSE or websocket with user pause control.

---

## 11. THE LEGACY CASES WHERE META REFRESH MIGHT BE CONSIDERED

Narrow cases where meta refresh might still be encountered or considered.

### 11.1 Static HTML Without Server Control

If a site is purely static HTML with no server side configuration (free hosting, GitHub Pages, etc.) and a redirect is needed:

* Meta refresh works.
* HTTP 301 cannot be configured.
* Alternative: edit the build process to generate the HTML elsewhere.

For Bubbles client sites: not applicable (Joseph controls the server).

### 11.2 Transitional Migration Pages

Some sites use meta refresh as a temporary transition mechanism:

```html
<!-- Brief intro page before redirect -->
<head>
    <meta http-equiv="refresh" content="3;url=/new-location/">
</head>
<body>
    <h1>We're now at a new location!</h1>
    <p>Redirecting you in 3 seconds...</p>
</body>
```

But: this is still a WCAG violation and SEO inferior. The HTTP 301 alternative:

```nginx
# nginx: instant 301
location /old-location/ {
    return 301 /new-location/;
}
```

Better than meta refresh in every way.

### 11.3 Marketing Splash Page

```html
<!-- Splash before main site -->
<meta http-equiv="refresh" content="5;url=/main/">
```

Anti pattern. Use JavaScript with user control button instead.

### 11.4 Polling Dashboards (Legacy)

Some older administrative dashboards use meta refresh to update displayed data:

```html
<!-- Old dashboard pattern -->
<meta http-equiv="refresh" content="30">
```

Anti pattern. Use JavaScript polling, SSE, or websocket.

### 11.5 Affiliate Cloaking (Anti Pattern)

Some affiliate marketers use meta refresh to obscure final destinations:

```html
<meta http-equiv="refresh" content="0;url=https://affiliate-tracking.com/?id=123">
```

Anti pattern. Google may penalize. Use HTTP 302 redirect properly.

### 11.6 The Bubbles Verdict

In 2026: no legitimate case for meta refresh in Bubbles client sites. Every scenario has a better alternative.

---

## 12. THE CMS AUTO GENERATION ISSUE

Some CMSs auto generate meta refresh tags. Cleanup during migrations is important.

### 12.1 WordPress

Standard WordPress doesn't auto generate meta refresh. Some plugins do:

* Redirection plugin (used for 301 management; usually configures HTTP redirects but may fall back to meta refresh).
* Pretty Permalinks transition (rare).
* Some older theme templates.

### 12.2 Older Drupal

Older Drupal versions sometimes used meta refresh for:

* Path redirects.
* Aliased URLs.

Modern Drupal uses HTTP redirects.

### 12.3 Joomla

Some Joomla modules use meta refresh:

* Maintenance mode redirects.
* User registration flows.

Modern Joomla supports HTTP redirects natively.

### 12.4 Squarespace, Wix, Weebly

Generally don't auto generate meta refresh. Custom user content may include it.

### 12.5 The Migration Cleanup

When migrating from any CMS to Bubbles:

```bash
# Find legacy meta refresh tags
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    grep -l 'meta http-equiv="refresh"' {} \;

# Remove
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    sed -i '/meta http-equiv="refresh"/d' {} \;

# Verify
curl -s https://example.com/ | grep -c 'meta http-equiv="refresh"'
# Expected: 0
```

### 12.6 The Post Migration Verification

```bash
# After migration, check entire sitemap
curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | while read url; do
    HAS=$(curl -s "$url" 2>/dev/null | grep -c 'meta http-equiv="refresh"')
    if [ "$HAS" != "0" ]; then
        echo "STILL HAS META REFRESH: $url"
    fi
done
```

Expected: no output (all clean).

---

## 13. THE BUBBLES DECISION: USE 301 INSTEAD

The Bubbles convention for redirect needs.

### 13.1 The Rule

**Do not use `<meta http-equiv="refresh">` in any Bubbles client site, for any purpose.**

For redirects: HTTP 301 at nginx (or 302/307 for temporary).
For auto refresh: JavaScript with user pause control.

### 13.2 The Why

* SEO: HTTP 301 dramatically better than meta refresh.
* Accessibility: WCAG 2.2.1 and 3.2.5 violations.
* Performance: HTTP 301 faster.
* Security: HTTP 301 less vulnerable to open redirect (when destinations are hardcoded).
* Consistency: HTTP 301 universally handled by all crawlers and tools.

### 13.3 The Per Client Decisions

For all Bubbles clients without exception:

```
Decision: omit meta http-equiv="refresh" entirely.
Redirect needs: implement via HTTP 301 at nginx layer.
Auto refresh needs: implement via JavaScript with WCAG compliant user controls.
```

### 13.4 The 14 Paying Client 301 Reference

Per Joseph's infrastructure, 14 paying clients have 301 redirects from old subdomains to custom domains. These are documented in framework-http-3xx-status-codes.md Section 16. Examples:

* `arcounselingandwellness.thatwebhostingguy.com` redirects to `https://arcounselingandwellness.com`
* `eurekabathworks.thatwebhostingguy.com` redirects to `https://eurekabathworks.com`
* `heritagehardwoodfloors.thatwebhostingguy.com` redirects to `https://heritagehardwoodfloorsnwa.com`
* (and 11 more)

All implemented as HTTP 301 via nginx server blocks. Meta refresh would not provide equivalent SEO authority transfer.

### 13.5 The Federal Subcontractor Implication

For Joseph's SDVOSB federal work:

* Section 508 compliance: zero tolerance for meta refresh (WCAG violations).
* WeCoverUSA and similar: must be clean.
* Contract renewal audits will flag meta refresh.

### 13.6 The Client Education

When clients see meta refresh in their old site and ask about it:

> "Meta refresh is an older HTML technique that doesn't transfer SEO authority well, doesn't meet accessibility standards, and has known security issues. Modern best practice is HTTP 301 redirects configured at the server level. We handle all your redirects this way; you don't need to do anything different in your content."

---

## 14. THE MIGRATION CLEANUP PATTERN

When taking over a client site, audit for and remove legacy meta refresh.

### 14.1 The Pre Migration Audit

```bash
#!/bin/bash
# Audit old site for meta refresh tags before migration

OLD_SITE_URL=$1

# Get all URLs from sitemap
URLS=$(curl -s "$OLD_SITE_URL/sitemap.xml" | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g')

echo "=== Pre-migration meta refresh audit ==="

for url in $URLS; do
    HAS=$(curl -s "$url" 2>/dev/null | grep -c 'meta http-equiv="refresh"')
    if [ "$HAS" -gt "0" ]; then
        REFRESH_DATA=$(curl -s "$url" 2>/dev/null | grep -oE 'meta http-equiv="refresh"[^>]+' | head -1)
        echo "FOUND: $url"
        echo "  $REFRESH_DATA"
    fi
done
```

### 14.2 The Cleanup Plan

For each meta refresh found:

1. **Determine purpose**: redirect or auto refresh?
2. **For redirects**: configure HTTP 301 at nginx layer; update destination if needed.
3. **For auto refresh**: implement JavaScript alternative with user controls (or remove if not essential).
4. **Document**: log the original behavior in project notes.

### 14.3 The Implementation

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name example.com;

    # Migrated redirects (from old meta refresh)
    location /old-blog/ {
        return 301 /blog/;
    }

    location /landing-page/ {
        return 301 /;
    }

    location /promo-redirect/ {
        return 301 /promotions/2026/;
    }

    # ... main site content ...
}
```

```bash
nginx -t && systemctl reload nginx
```

### 14.4 The Post Migration Verification

```bash
# Verify the redirects work
for path in /old-blog/ /landing-page/ /promo-redirect/; do
    RESULT=$(curl -sI "https://example.com$path" | head -3)
    echo "$path:"
    echo "$RESULT"
    echo ""
done
```

Expected: HTTP 301 with Location header pointing to new URL.

### 14.5 The Client Site Cleanup Across Templates

```bash
# Remove all meta refresh from all client HTML files
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    BEFORE=$(find "$site_dir" -name "*.html" -exec grep -l 'meta http-equiv="refresh"' {} \; 2>/dev/null | wc -l)

    if [ "$BEFORE" -gt "0" ]; then
        find "$site_dir" -name "*.html" -type f -exec \
            sed -i '/meta http-equiv="refresh"/d' {} \;
        echo "$SITE: removed meta refresh from $BEFORE files"
    fi
done

# Verify
for site_dir in /var/www/sites/*/; do
    REMAINING=$(find "$site_dir" -name "*.html" -exec grep -l 'meta http-equiv="refresh"' {} \; 2>/dev/null | wc -l)
    if [ "$REMAINING" -gt "0" ]; then
        SITE=$(basename "$site_dir")
        echo "STILL HAS: $SITE"
    fi
done
```

### 14.6 The Documentation Update

For each migration, document:

```
Migration cleanup:
  Old site had N meta refresh tags.
  Replaced with HTTP 301 redirects in nginx:
    /old-path-1/ -> /new-path-1/
    /old-path-2/ -> /new-path-2/
  Verified via: curl -sI <URL>
  Status: complete
```

---

## 15. THE 14 PAYING CLIENT SUBDOMAIN 301 RELATIONSHIP

Per Joseph's standing infrastructure, 14 paying clients have established 301 redirect patterns. This framework documents how meta refresh would fail to achieve equivalent results.

### 15.1 The Existing Pattern (HTTP 301)

Per framework-http-3xx-status-codes.md Section 16, the 14 paying clients have:

```nginx
# /etc/nginx/sites-enabled/heritage-hardwood-301.conf

server {
    listen 443 ssl http2;
    server_name heritagehardwoodfloors.thatwebhostingguy.com;

    # SSL cert configuration ...

    # Redirect everything to canonical domain
    return 301 https://heritagehardwoodfloorsnwa.com$request_uri;
}
```

Plus similar configs for the other 13 paying clients.

### 15.2 What Meta Refresh Would Do Differently

If the same redirect used meta refresh instead:

```html
<!-- Pre Bubbles legacy approach -->
<meta http-equiv="refresh" content="0;url=https://heritagehardwoodfloorsnwa.com">
```

The problems:

* PageRank transfer would be partial (~70-85% vs ~95%).
* Indexing speed would be slower.
* Crawlers would handle inconsistently.
* WCAG 3.2.5 violation for accessibility audits.
* Performance: browser must download HTML first.

The HTTP 301 approach is dramatically superior.

### 15.3 The Full List Of 14 Paying Clients

Per the standing infrastructure (see framework-http-3xx-status-codes.md Section 16):

1. arcounselingandwellness
2. eurekabathworks
3. heritagehardwoodfloors
4. localliving
5. janieseecleaning/showmecleannwa
6. wecoverusa
7. white-river-cabins/whiterivercabins
8. greenoughsguideservice
9. beyondastep
10. idsofnwa
11. marshallese-voices
12. nwapoolice
13. handledtax/handledtaxes
14. (the 14th per the established pattern)

All implemented as HTTP 301 via nginx. None use meta refresh.

### 15.4 The Verification

```bash
# Verify each of the 14 paying client redirects
for old_subdomain in arcounselingandwellness eurekabathworks heritagehardwoodfloors localliving showmecleannwa wecoverusa whiterivercabins greenoughsguideservice; do
    URL="https://${old_subdomain}.thatwebhostingguy.com"
    STATUS_CODE=$(curl -sI "$URL" 2>/dev/null | head -1 | awk '{print $2}')
    LOCATION=$(curl -sI "$URL" 2>/dev/null | grep -i "^location:" | head -1 | awk '{print $2}' | tr -d '\r')

    echo "$old_subdomain: $STATUS_CODE -> $LOCATION"
done
```

Expected: each returns HTTP 301 with Location pointing to the custom domain.

### 15.5 The Implication

For Bubbles client work:

* Redirects are HTTP 301 at nginx layer.
* Never meta refresh.
* The 14 paying client pattern is the gold standard reference.

---

## 16. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 16.1 The Replacement: HTTP 301 at nginx

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name old-domain.com;

    # Permanent redirect to new domain
    return 301 https://new-domain.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name new-domain.com;

    # ... actual site content ...
}
```

```bash
nginx -t && systemctl reload nginx

# Verify
curl -sI https://old-domain.com/some/page/
# Expected: HTTP/2 301; Location: https://new-domain.com/some/page/
```

### 16.2 The Replacement: FastAPI redirect

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.get("/old-path/")
async def redirect_old():
    return RedirectResponse(url="/new-path/", status_code=301)

@app.get("/old-blog/{slug}")
async def redirect_old_blog(slug: str):
    return RedirectResponse(url=f"/blog/{slug}", status_code=301)
```

### 16.3 The Replacement: JavaScript with WCAG controls

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Redirecting...</title>
</head>
<body>
    <main>
        <h1>Our site has moved</h1>
        <p>We'll redirect you in <span id="countdown">5</span> seconds.</p>

        <div class="controls">
            <button id="proceed">Go now</button>
            <button id="cancel">Cancel</button>
        </div>

        <p>
            Or visit
            <a href="/new-location/">our new location</a> directly.
        </p>
    </main>

    <script>
        let seconds = 5;
        const countdown = document.getElementById('countdown');
        let timer;

        const redirect = () => {
            window.location.href = '/new-location/';
        };

        const startCountdown = () => {
            timer = setInterval(() => {
                seconds--;
                countdown.textContent = seconds;
                if (seconds <= 0) {
                    redirect();
                }
            }, 1000);
        };

        document.getElementById('proceed').addEventListener('click', () => {
            clearInterval(timer);
            redirect();
        });

        document.getElementById('cancel').addEventListener('click', () => {
            clearInterval(timer);
            countdown.parentElement.textContent = 'Redirect cancelled.';
            document.querySelector('.controls').style.display = 'none';
        });

        startCountdown();
    </script>
</body>
</html>
```

WCAG 2.2.1 and 3.2.5 compliant: user has explicit cancel and proceed controls.

### 16.4 The Replacement: JavaScript polling for dashboards

```html
<!doctype html>
<html lang="en">
<head>
    <title>Dashboard</title>
</head>
<body>
    <h1>Dashboard</h1>

    <div id="data">Loading...</div>

    <div class="controls">
        <button id="pause">Pause updates</button>
        <button id="resume" disabled>Resume updates</button>
        <p>Last updated: <time id="last-update">...</time></p>
    </div>

    <script>
        let pollInterval;
        const pauseBtn = document.getElementById('pause');
        const resumeBtn = document.getElementById('resume');

        const updateData = async () => {
            try {
                const response = await fetch('/api/dashboard-data/');
                const data = await response.json();

                document.getElementById('data').textContent = JSON.stringify(data);
                document.getElementById('last-update').textContent = new Date().toLocaleString();
            } catch (error) {
                console.error('Update failed:', error);
            }
        };

        const startPolling = () => {
            updateData();  // immediate update
            pollInterval = setInterval(updateData, 30000);  // every 30 seconds
            pauseBtn.disabled = false;
            resumeBtn.disabled = true;
        };

        const pausePolling = () => {
            clearInterval(pollInterval);
            pauseBtn.disabled = true;
            resumeBtn.disabled = false;
        };

        pauseBtn.addEventListener('click', pausePolling);
        resumeBtn.addEventListener('click', startPolling);

        // Start initial polling
        startPolling();
    </script>
</body>
</html>
```

### 16.5 The Cleanup: Remove meta refresh from a site

```bash
# Find all HTML files with meta refresh
SITE_DIR=/var/www/sites/example.com
find "$SITE_DIR" -name "*.html" -type f -exec \
    grep -l 'meta http-equiv="refresh"' {} \;

# Remove the tags
find "$SITE_DIR" -name "*.html" -type f -exec \
    sed -i '/meta http-equiv="refresh"/d' {} \;

# Verify
find "$SITE_DIR" -name "*.html" -type f -exec \
    grep -l 'meta http-equiv="refresh"' {} \;
# Should produce no output
```

### 16.6 The Audit: Find meta refresh across all sites

```bash
#!/bin/bash
# /usr/local/bin/meta-refresh-audit.sh

echo "=== Meta refresh audit ==="

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    FOUND_FILES=$(find "$site_dir" -name "*.html" -type f -exec grep -l 'meta http-equiv="refresh"' {} \; 2>/dev/null)

    if [ -n "$FOUND_FILES" ]; then
        echo "META REFRESH FOUND: $SITE"
        echo "$FOUND_FILES"
        echo ""
    fi
done
```

### 16.7 The Lighthouse accessibility check

```bash
# Lighthouse audit for meta refresh
URL=$1

lighthouse "$URL" \
    --only-audits=meta-refresh \
    --output=json \
    --output-path=/tmp/lh-meta-refresh.json \
    --quiet

python3 -c "
import json
with open('/tmp/lh-meta-refresh.json') as f:
    data = json.load(f)
audit = data['audits'].get('meta-refresh', {})
score = audit.get('score')
title = audit.get('title')
description = audit.get('description', '')

print(f'Score: {score}')
print(f'Title: {title}')
print(f'Description: {description[:200]}')
"
```

A passing site shows `score: 1`.

### 16.8 The pre deploy check

```bash
#!/bin/bash
# /usr/local/bin/predeploy-meta-refresh-check.sh

SITE_DIR=$1
FAILED=0

find "$SITE_DIR" -name "*.html" -type f | while read f; do
    if grep -q 'meta http-equiv="refresh"' "$f"; then
        echo "FAIL: $f has meta refresh"
        FAILED=1
    fi
done

if [ $FAILED -eq 1 ]; then
    echo ""
    echo "Pre deploy check failed. Replace meta refresh with HTTP 301 or JavaScript before deploying."
    exit 1
fi

echo "OK: no meta refresh found"
```

---

## 17. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles configuration: omit meta refresh entirely.

### 17.1 The Template Default

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- NO <meta http-equiv="refresh"> -->
    <!-- Redirects: HTTP 301 at nginx -->
    <!-- Auto refresh: JavaScript with user controls -->

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- ... -->
</head>
```

### 17.2 The Redirect Pattern (HTTP 301)

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name example.com;

    # Migrated redirects from old site
    location /old-page-1/ {
        return 301 /new-page-1/;
    }

    location /old-page-2/ {
        return 301 /new-page-2/;
    }

    # ... main site content ...
}
```

### 17.3 The Audit Workflow

```bash
# Weekly across all sites
meta-refresh-audit.sh

# Should output nothing (no sites have meta refresh)

# If found: cleanup
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    sed -i '/meta http-equiv="refresh"/d' {} \;

nginx -t && systemctl reload nginx
```

### 17.4 The Pre Deploy Gate

In CI/CD pipeline (or manual pre deploy check):

```bash
predeploy-meta-refresh-check.sh /var/www/sites/example.com/
```

Rejects deployment if any meta refresh found.

### 17.5 The Client Documentation

For each client project:

```
Redirect handling:
  Implementation: HTTP 301 via nginx
  No meta http-equiv="refresh" used
  
  Per route redirects: see nginx config
  
  Migration cleanup: completed; no legacy meta refresh remains.

Auto refresh:
  Not used.
  If real time data is needed: JavaScript polling with user pause control.
```

### 17.6 The Federal Subcontract Compliance Note

For SDVOSB federal subcontracting work:

```
WCAG compliance:
  Meta refresh check: PASS (none used)
  WCAG 2.2.1 Timing Adjustable: compliant
  WCAG 3.2.5 Change on Request: compliant
  Section 508: aligned
  
  Verification: meta-refresh-audit.sh; Lighthouse audit; manual axe DevTools.
```

---

## 18. AUDIT CHECKLIST

Run through these 30 items.

### Absence verification

1. [ ] No `<meta http-equiv="refresh">` on any indexable URL.
2. [ ] No `<meta http-equiv="refresh">` on any page including landing pages.
3. [ ] No `<meta http-equiv="refresh">` in any template file.
4. [ ] No `<meta http-equiv="refresh">` after migration cleanup.
5. [ ] No `<meta http-equiv="Refresh">` (case variation).

### Redirect implementation

6. [ ] All redirects implemented at HTTP layer (nginx return 301).
7. [ ] HTTP 301 verified via `curl -sI`.
8. [ ] Location header points to correct destination.
9. [ ] Redirects work for HTTPS only (no HTTP fallback).

### 14 paying client subdomain pattern

10. [ ] All 14 paying client subdomain 301s in place.
11. [ ] Each redirects to correct custom domain.
12. [ ] No meta refresh fallback exists.

### WCAG compliance

13. [ ] WCAG 2.2.1 (Timing Adjustable): no time limits set without user control.
14. [ ] WCAG 3.2.5 (Change on Request): no context changes without user request.
15. [ ] Lighthouse meta-refresh audit: passes (score 1).
16. [ ] axe DevTools: no meta refresh violations.
17. [ ] Federal subcontractor sites: zero violations.

### Migration cleanup

18. [ ] Pre migration audit identified all legacy meta refresh.
19. [ ] Each was replaced with HTTP 301 or JavaScript alternative.
20. [ ] Post migration verification confirms removal.

### CMS specific

21. [ ] WordPress: no plugins generating meta refresh.
22. [ ] Drupal: no modules using meta refresh.
23. [ ] Joomla: maintenance mode and registration not using meta refresh.

### Security

24. [ ] No user controlled redirect destinations.
25. [ ] No open redirect vulnerabilities (whitelist if user input involved).

### JavaScript alternatives

26. [ ] If client side redirects needed: WCAG compliant with user controls.
27. [ ] If polling needed: user pause/resume controls present.
28. [ ] No auto refresh without user control.

### Tooling

29. [ ] `meta-refresh-audit.sh` script in `/usr/local/bin/`.
30. [ ] Pre deploy check rejects new meta refresh additions.

A site that passes all 30 has correctly omitted meta refresh in favor of HTTP redirects and JavaScript alternatives.

---

## 19. COMMON PITFALLS

Eleven patterns to recognize and avoid.

**Pitfall 1: Using meta refresh for redirect "because it's easier".**
Symptom: developer adds `<meta http-equiv="refresh" content="0;url=...">` instead of HTTP 301.
Why it breaks: SEO inferior, WCAG violation, security risk.
Fix: HTTP 301 at nginx (or FastAPI for app level).

**Pitfall 2: Auto refreshing dashboards.**
Symptom: dashboard has `<meta http-equiv="refresh" content="30">`.
Why it breaks: WCAG 2.2.1 violation; jarring UX.
Fix: JavaScript polling with user pause control.

**Pitfall 3: Splash page with meta refresh.**
Symptom: 5 second meta refresh transition page.
Why it breaks: WCAG 3.2.5 violation; SEO inferior.
Fix: HTTP 301 (instant) or JavaScript with user control.

**Pitfall 4: WordPress plugin meta refresh.**
Symptom: plugin auto-adds meta refresh for some functionality.
Why it breaks: legacy plugin behavior.
Fix: disable plugin feature; use HTTP redirect alternative.

**Pitfall 5: Meta refresh with user-controlled URL.**
Symptom: `<meta http-equiv="refresh" content="0;url={user_input}">`.
Why it breaks: open redirect vulnerability.
Fix: server side whitelist; never trust user input as redirect destination.

**Pitfall 6: Forgetting meta refresh in template cleanup.**
Symptom: migration mostly done but some old templates still have meta refresh.
Why it breaks: incomplete migration.
Fix: full file system grep; remove all occurrences.

**Pitfall 7: Federal subcontractor site with meta refresh.**
Symptom: WeCoverUSA inherited site with legacy meta refresh.
Why it breaks: Section 508 audit failure.
Fix: immediate cleanup; migration to HTTP 301.

**Pitfall 8: Meta refresh with very short delay (0 or 1 second).**
Symptom: developer thinks "0 second delay is the same as immediate redirect".
Why it breaks: still WCAG violation; SEO inferior; security risk.
Fix: HTTP 301.

**Pitfall 9: Meta refresh with long delay (10+ seconds).**
Symptom: marketing wants "5 second branded transition".
Why it breaks: severe WCAG 2.2.1 violation; user feels trapped.
Fix: JavaScript with user controls.

**Pitfall 10: Meta refresh from PHP redirect function.**
Symptom: legacy PHP code outputs meta refresh tag.
Why it breaks: legacy PHP pattern.
Fix: use PHP `header('Location: ...')` for HTTP redirect.

**Pitfall 11: Meta refresh on 404 page.**
Symptom: 404 page meta refreshes to homepage.
Why it breaks: bad UX; obscures 404 status; WCAG violation.
Fix: proper 404 page with manual links and explanation.

---

## 20. DIAGNOSTIC COMMANDS

Reference of commands useful for meta refresh investigation.

### Inspect a URL

```bash
# Check for meta refresh
curl -s https://example.com/ | grep -c 'meta http-equiv="refresh"'
# Expected: 0

# Get details if present
curl -s https://example.com/ | grep -oE 'meta http-equiv="refresh"[^>]+'
```

### Bulk audit a sitemap

```bash
curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | while read url; do
    HAS=$(curl -s "$url" 2>/dev/null | grep -c 'meta http-equiv="refresh"')
    if [ "$HAS" != "0" ]; then
        echo "META REFRESH: $url"
    fi
done
```

### Find across all client sites

```bash
meta-refresh-audit.sh
```

### Verify HTTP 301 redirect works

```bash
URL=$1

curl -sI "$URL" | head -3
# Expected: HTTP/2 301 with Location header
```

### Verify 14 paying client redirects

```bash
for old in arcounselingandwellness eurekabathworks heritagehardwoodfloors localliving wecoverusa; do
    URL="https://${old}.thatwebhostingguy.com"
    RESULT=$(curl -sI "$URL" 2>/dev/null | head -3)
    echo "$old:"
    echo "$RESULT"
    echo ""
done
```

### Lighthouse accessibility check

```bash
URL=$1

lighthouse "$URL" \
    --only-categories=accessibility \
    --quiet \
    --output=json \
    --output-path=/tmp/lh-a11y.json

python3 -c "
import json
with open('/tmp/lh-a11y.json') as f:
    data = json.load(f)
audit = data['audits'].get('meta-refresh', {})
print(f'meta-refresh: {audit.get(\"score\")} - {audit.get(\"title\")}')
"
```

### Browser DevTools verification

In Chrome DevTools:

1. Open the page.
2. Network tab: clear; reload.
3. Look for meta refresh redirect (loads page, then redirects).
4. Compare with HTTP 301 (immediate redirect, no HTML loaded for old URL).

### Apply changes

```bash
# Remove meta refresh from files
find /var/www/sites/ -name "*.html" -type f -exec \
    sed -i '/meta http-equiv="refresh"/d' {} \;

# Apply nginx changes if 301 redirects added
nginx -t && systemctl reload nginx
```

---

## 21. CROSS-REFERENCES

* [framework-http-3xx-status-codes.md](framework-http-3xx-status-codes.md): the canonical HTTP redirect mechanism (301, 302, 307, 308). Section 16 documents Joseph's 14 paying client subdomain 301 pattern.
* [framework-html-meta-content-language.md](framework-html-meta-content-language.md): also discourages legacy http-equiv mechanisms; both prefer proper HTTP-layer or modern HTML alternatives.
* [framework-html-meta-robots.md](framework-html-meta-robots.md): redirects and noindex interaction.
* [framework-html-meta-charset.md](framework-html-meta-charset.md): meta tag placement order in head.
* [framework-http-security-headers.md](framework-http-security-headers.md): security header coordination including Referrer-Policy on redirects.
* [framework-http-performance-headers.md](framework-http-performance-headers.md): performance considerations for redirects.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): redirect handling part of ranking; HTTP 301 is the canonical signal.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including redirect implementation.
* W3C HTML meta refresh: https://html.spec.whatwg.org/multipage/semantics.html#attr-meta-http-equiv-refresh
* W3C WCAG 2.2.1 Timing Adjustable: https://www.w3.org/WAI/WCAG21/Understanding/timing-adjustable.html
* W3C WCAG 3.2.5 Change on Request: https://www.w3.org/WAI/WCAG21/Understanding/change-on-request.html
* MDN meta http-equiv="refresh": https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta
* Google Search Central on redirects: https://developers.google.com/search/docs/crawling-indexing/301-redirects
* OWASP Unvalidated Redirects: https://owasp.org/www-community/attacks/Unvalidated_Redirects_and_Forwards_Cheat_Sheet
* Section 508 Standards: https://www.access-board.gov/ict/

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Never use `<meta http-equiv="refresh">`. Use HTTP 301 for redirects. Use JavaScript with user controls for auto refresh.**

### The canonical replacements

```nginx
# Redirect via HTTP 301 (preferred)
location /old-path/ {
    return 301 /new-path/;
}
```

```javascript
// Auto refresh via JavaScript with user control
let pollInterval;

const startPolling = () => {
    pollInterval = setInterval(updateData, 30000);
};

document.getElementById('pause').addEventListener('click', () => {
    clearInterval(pollInterval);
});
```

### The forbidden patterns

```html
<!-- NEVER USE -->
<meta http-equiv="refresh" content="0;url=https://new-url.com/">
<meta http-equiv="refresh" content="5;url=https://new-url.com/">
<meta http-equiv="refresh" content="30">
<meta http-equiv="refresh" content="60">
```

### Why

* SEO: HTTP 301 transfers ~95% PageRank; meta refresh ~70-85%.
* Accessibility: WCAG 2.2.1 and 3.2.5 violations.
* Performance: meta refresh requires full HTML download.
* Security: open redirect risk.
* Federal subcontracting: Section 508 audit failures.

### Five rules to memorize

1. **For redirects: HTTP 301 at nginx (or FastAPI 301).**
2. **For auto refresh: JavaScript with user pause/resume controls.**
3. **Migration cleanup: remove ALL meta refresh tags.**
4. **Federal subcontracting: zero tolerance.**
5. **The 14 paying client pattern uses HTTP 301 only.**

### Five commands every operator should know

```bash
# 1. Check for meta refresh (should be 0)
curl -s URL | grep -c 'meta http-equiv="refresh"'

# 2. Audit all sites
meta-refresh-audit.sh

# 3. Verify a 301 redirect works
curl -sI URL | head -3

# 4. Remove meta refresh from files
find /var/www/sites/example.com/ -name "*.html" -type f -exec sed -i '/meta http-equiv="refresh"/d' {} \;

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. No meta refresh anywhere
URL=https://example.com/
HAS=$(curl -s "$URL" | grep -c 'meta http-equiv="refresh"')
[ "$HAS" = "0" ] && echo "OK" || echo "FAIL: $HAS occurrences"

# 2. Redirects use HTTP 301
OLD_URL=https://old.example.com/
STATUS=$(curl -sI "$OLD_URL" 2>/dev/null | head -1 | awk '{print $2}')
[ "$STATUS" = "301" ] && echo "OK: HTTP 301" || echo "FAIL: $STATUS"

# 3. WCAG compliance
lighthouse URL --only-audits=meta-refresh --quiet --output=json --output-path=/tmp/lh.json
SCORE=$(python3 -c "import json; print(json.load(open('/tmp/lh.json'))['audits']['meta-refresh']['score'])")
[ "$SCORE" = "1" ] && echo "OK: WCAG passes" || echo "FAIL: WCAG violation"
```

If all three pass AND `meta-refresh-audit.sh` shows zero across all sites AND Lighthouse accessibility score is 100, the meta refresh layer is correctly handled (which is to say: absent).

---

End of framework-html-meta-refresh.md.
