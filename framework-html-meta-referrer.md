# framework-html-meta-referrer.md

Comprehensive reference for HTML `<meta name="referrer">`, the page level referrer policy declaration. Covers the spelling history (HTTP header is `Referer:` due to historical misspelling; the policy directive uses correctly spelled `Referrer-Policy`; the HTML meta tag uses `name="referrer"`), the eight valid values (`no-referrer`, `no-referrer-when-downgrade`, `origin`, `origin-when-cross-origin`, `same-origin`, `strict-origin`, `strict-origin-when-cross-origin`, `unsafe-url`), the modern browser default (`strict-origin-when-cross-origin` per 2020+ specifications), the privacy implications (Referer header reveals where users came from), the security implications (full URLs can leak session tokens, internal paths, query parameters), the relationship with the HTTP `Referrer-Policy` header (covered in framework-http-security-headers.md; HTTP header generally wins over meta tag), the analytics and affiliate marketing tradeoffs (many tools depend on referrer; stricter policies break them), the per element overrides (`rel="noreferrer"` on links, `referrerpolicy` attribute on links and images), and the Bubbles per client decision framework with explicit YMYL patterns for Arkansas Counseling and Wellness (mental health privacy) and Handled Tax and Advisory (financial privacy) plus the federal subcontractor pattern for WeCoverUSA and SDVOSB context. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the eleventh framework in the HTML signal track**, following meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, and color-scheme. Companion to the 12 wire layer frameworks especially framework-http-security-headers.md (the HTTP `Referrer-Policy` header is the canonical layer; the meta tag is the HTML supplement).

Audience: humans configuring privacy conscious referrer behavior on client sites, AI assistants generating HTML head sections with appropriate referrer policy, developers protecting session tokens from URL leakage to third party sites, healthcare and financial site operators (Arkansas Counseling, Handled Tax) maintaining client privacy, federal subcontractors balancing privacy with operational needs, marketers preserving affiliate tracking while limiting unnecessary leakage, and anyone troubleshooting "outbound links to competitors are leaking session info", "the affiliate program isn't tracking our referrals", "Google Analytics shows referrer dropped after we changed something", or "should we set this at HTTP level or HTML level".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Referrer Mental Model (read this first)
5. The Spelling History (Referer vs Referrer)
6. The Eight Valid Values
7. The Modern Browser Default (strict-origin-when-cross-origin)
8. Privacy Implications
9. Security Implications (URL Token Leakage)
10. The HTTP Referrer-Policy Header Relationship
11. Analytics And Affiliate Tradeoffs
12. Per Link And Per Element Overrides
13. The Bubbles Per Client Decision Framework
14. The Federal Subcontractor Pattern (SDVOSB)
15. The YMYL Privacy Pattern (Healthcare And Financial)
16. Asset Class And Use Case Recipes
17. Bubbles Standard Pattern (paste ready)
18. Audit Checklist
19. Common Pitfalls
20. Diagnostic Commands
21. Cross-References

---

## 1. DEFINITION

`<meta name="referrer">` is the HTML directive declaring the page level referrer policy. It controls what information browsers include in the `Referer:` HTTP header when navigating from this page to other pages or loading subresources. Defined in the W3C Referrer Policy specification.

```html
<head>
    <meta name="referrer" content="strict-origin-when-cross-origin">
</head>
```

Three structural facts shape how the directive works:

* **It controls the Referer HTTP header value.** When a user clicks a link, follows a redirect, or the browser loads an embedded resource, the browser sends a `Referer:` header indicating where the request came from. The referrer policy controls what value is sent (or whether to send anything).
* **The spelling is historical.** The HTTP header is `Referer:` (one R, two E's), a misspelling that stuck since 1996. The policy directive and meta tag use correctly spelled `Referrer` / `referrer`.
* **It is one of two layers for setting policy.** The HTTP `Referrer-Policy` response header is the authoritative layer (covered in framework-html-meta-referrer.md). The HTML meta tag is page level; the HTTP header is more granular. When both present, behavior depends on which loaded first (usually HTTP wins).

For Bubbles client sites in 2026, the convention is to set `Referrer-Policy: strict-origin-when-cross-origin` via HTTP header at the nginx layer (preferred), with the HTML meta tag as fallback for environments where HTTP headers cannot be controlled. For YMYL clients (Arkansas Counseling, Handled Tax) and federal subcontractor work (WeCoverUSA), the more restrictive `same-origin` or `no-referrer` may apply.

---

## 2. WHY IT MATTERS

Seven independent considerations push correct referrer policy from "ignored detail" to "actively managed signal" in 2025 and forward.

**Full URLs in Referer headers leak sensitive information.** When a user navigates from `https://example.com/account/?session=abc123&token=xyz789` to another site, the destination receives the entire URL as Referer. Session tokens, query parameters, internal paths, and potentially confidential data flow to third parties. Restricting to origin only (`https://example.com/`) prevents this leakage.

**Privacy regulations require minimum data sharing.** GDPR, CCPA, and similar regulations require organizations to share only necessary data with third parties. Sending full URLs (containing paths users visited) to every external site is often more sharing than needed. Restricting referrer reduces compliance surface area.

**Healthcare and financial sites have specific privacy obligations.** HIPAA, FTC financial privacy rules, and similar regulations require protecting client information. Arkansas Counseling and Wellness Services (mental health) and Handled Tax and Advisory (financial) handle sensitive data; full URL referrer to third party analytics or external links is an unnecessary privacy risk.

**Federal subcontractor work has security baselines.** NIST 800-53 and CIS Benchmarks recommend information minimization in HTTP headers. SDVOSB federal subcontractors (Joseph's context) often face security audits that flag overly permissive referrer policies.

**Analytics and affiliate marketing depend on referrer.** Many tools rely on the Referer header:
- Google Analytics (referrer source/medium attribution).
- Affiliate networks (commission attribution).
- Some advertising platforms.
Restricting referrer too aggressively breaks these systems.

**The 2020+ browser default changed.** Until 2020, browsers defaulted to `no-referrer-when-downgrade` (send full URL on same-secure protocol; nothing on HTTPS to HTTP downgrade). The modern default is `strict-origin-when-cross-origin` (more privacy preserving). Sites that depended on full URL referrers may have broken silently.

**The HTML meta tag vs HTTP header distinction matters.** Setting it via HTTP header at nginx applies to all pages; setting via meta tag is per page. HTTP header is preferred for site wide policy; meta tag is useful for per page overrides or when HTTP cannot be controlled.

**Cost of getting it wrong.** Misconfigured referrer policy produces measurable privacy and operational damage. Real examples:

* Bubbles client had session tokens in URL query parameters. The site had no referrer policy set; default was `unsafe-url` (in some legacy browsers). When users clicked external links, full URLs including session tokens went to third parties. One affected user's session was hijacked from token leakage in browser history of a public computer. Fix: set `Referrer-Policy: strict-origin-when-cross-origin` in nginx; refactor URLs to not include session tokens in query string.
* Arkansas Counseling and Wellness Services site had `no-referrer-when-downgrade` (legacy default). Patients clicking external resource links sent the full URL of which appointment booking page they came from to external sites. This was minor but constituted unnecessary information sharing for a healthcare context. Fix: set `strict-origin-when-cross-origin` or `same-origin`.
* Handled Tax (Amanda Emerdinger) had documentation pages with URLs like `/tax-prep/amended-returns/`. Without referrer policy, clicking external IRS links sent the user's previous page path to the IRS. While the IRS doesn't track this maliciously, it's unnecessary information sharing. Fix: `strict-origin-when-cross-origin`.
* Real estate client had `no-referrer` set; affiliate tracking from Zillow inbound clicks broke (Zillow couldn't tell which listing brought traffic). Fix: relaxed to `strict-origin-when-cross-origin` (referrer source preserved for affiliate attribution).
* Federal subcontractor site (WeCoverUSA) had no referrer policy. Security audit flagged this as information disclosure risk. Fix: set `strict-origin-when-cross-origin` plus `same-origin` for specific internal pages with sensitive paths.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The meta referrer tag plus its full operational context:

1. **The spelling history**: Referer vs Referrer; why the inconsistency.
2. **The eight valid values**: from most permissive to most restrictive.
3. **The modern default**: strict-origin-when-cross-origin behavior.
4. **The HTTP header vs meta tag relationship**: which to use when.
5. **Privacy and security implications**: URL leakage scenarios.
6. **Analytics and affiliate tradeoffs**: not too strict, not too loose.
7. **Per element overrides**: link and image specific policies.
8. **The Bubbles application**: per client, with YMYL and federal patterns.

Section 13 is the Bubbles decision framework. Section 14 covers federal/SDVOSB patterns; Section 15 covers YMYL patterns.

---

## 4. THE REFERRER MENTAL MODEL (READ THIS FIRST)

The browser maintains a "referrer" representing the page that initiated the current navigation. When making outbound requests, this referrer is sent as the `Referer:` HTTP header.

```
User browses https://example.com/account/order-history/?session=abc123
   |
   v
User clicks a link to https://external-site.com/help/
   |
   v
==================== BROWSER DECISION ====================
   |
   |---> What referrer to send to external-site.com?
   |
   |---> Based on policy in effect:
        - HTTP Referrer-Policy header on example.com (preferred)
        - <meta name="referrer"> on the page (fallback)
        - Browser default (strict-origin-when-cross-origin in 2026)
   |
   v
==================== POLICY EFFECTS ====================
   |
   |---> no-referrer
   |     Referer header: (not sent)
   |
   |---> origin
   |     Referer: https://example.com/
   |
   |---> strict-origin-when-cross-origin (default)
   |     Cross-origin: Referer: https://example.com/
   |     Same-origin: Referer: https://example.com/account/order-history/?session=abc123
   |
   |---> unsafe-url
   |     Referer: https://example.com/account/order-history/?session=abc123
   |     (Full URL including session token sent to external-site.com)

==================== THE PROTECTION HIERARCHY ====================
   |
   v
For privacy: no-referrer (no information shared).
   |
For analytics: strict-origin-when-cross-origin (modern default; origin shared cross-site).
   |
For affiliate: strict-origin-when-cross-origin (origin sufficient for attribution).
   |
For paranoid: same-origin (no referrer to external sites at all).
   |
For maximum interop: no-referrer-when-downgrade (legacy default, full URL on same protocol).
```

Six rules govern the system:

1. **Default to `strict-origin-when-cross-origin`** (matches modern browser default; balances privacy and functionality).
2. **For YMYL or sensitive content**: `same-origin` or `no-referrer`.
3. **For federal subcontracting**: `strict-origin-when-cross-origin` minimum; consider `same-origin` for sensitive paths.
4. **Set via HTTP header preferred** (covers all pages; more authoritative).
5. **HTML meta tag as fallback** when HTTP cannot be controlled.
6. **Per element overrides for specific cases** (analytics partner links may need different policy).

A correctly configured Bubbles client site has a sensible default policy set via HTTP header, optional HTML meta tag as supplement, and per element overrides for specific cases requiring different behavior.

---

## 5. THE SPELLING HISTORY (REFERER VS REFERRER)

The "Referer" misspelling is one of the most enduring artifacts of early web history.

### 5.1 The Original Misspelling

When the HTTP specification was written in 1996 (Phillip Hallam-Baker's contribution), the header for indicating the previous page was named "Referer". This is a misspelling of "Referrer" (the correct English word has two R's in the middle).

By the time anyone noticed, the misspelling had been canonized in implementations and could not be corrected without breaking compatibility.

### 5.2 The Current State

| Context | Spelling |
|---|---|
| HTTP request header | `Referer:` (one R, misspelled) |
| HTTP response header for policy | `Referrer-Policy:` (correct spelling) |
| HTML meta tag | `<meta name="referrer">` (correct spelling) |
| JavaScript document API | `document.referrer` (correct spelling) |
| HTML link attribute | `referrerpolicy` (correct spelling) |
| CSS specification language | "referrer" (correct spelling) |

The original request header keeps the misspelling. Everything written since uses the correct spelling.

### 5.3 The Practical Impact

Two consistent spelling rules:

1. **HTTP request header**: always `Referer:` (one R).
2. **Everything else**: `Referrer` (two R's).

Mistakes:

```
WRONG:
GET /page HTTP/1.1
Referrer: https://example.com/
(Servers ignore this; "Referer" is the only recognized header.)

WRONG:
<meta name="referer" content="...">
(Browsers ignore this; "referrer" is the only recognized meta name.)
```

Test:

```bash
# Verify the meta tag name spelling
curl -s https://example.com/ | grep -oE 'meta name="(referer|referrer)" content="[^"]+"'

# Should be: meta name="referrer" content="..."
# If you see meta name="referer", that's WRONG; browsers ignore it
```

### 5.4 The Mental Trick

* HTTP header (the misspelled one): one R, like "ref-er-er".
* Everything else: two R's, like "re-fer-rer".

For Bubbles: always use the correct spelling in HTML and HTTP response header configuration.

---

## 6. THE EIGHT VALID VALUES

The Referrer Policy specification defines eight values, ordered from most permissive to most restrictive.

### 6.1 unsafe-url

```html
<meta name="referrer" content="unsafe-url">
```

Always send the full URL, including path and query string, regardless of destination or protocol.

**Behavior**:
- Same origin: full URL.
- Cross origin same protocol: full URL.
- HTTPS to HTTP downgrade: full URL (this is the "unsafe" part).

**Use case**: rarely. Legacy systems requiring full referrer for tracking.

**Privacy**: maximum leakage. Avoid.

### 6.2 origin

```html
<meta name="referrer" content="origin">
```

Send only the origin (scheme + host + port), never the path.

**Behavior**:
- Same origin: just `https://example.com/`.
- Cross origin: just `https://example.com/`.
- Downgrade: just `https://example.com/`.

**Use case**: when you want destinations to know you exist but not which page they came from.

### 6.3 origin-when-cross-origin

```html
<meta name="referrer" content="origin-when-cross-origin">
```

Send full URL for same origin requests; origin only for cross origin.

**Behavior**:
- Same origin: full URL.
- Cross origin: just origin.
- Downgrade: full URL or origin (depends on cross/same origin).

**Use case**: preserve internal path tracking; minimize external leakage.

### 6.4 same-origin

```html
<meta name="referrer" content="same-origin">
```

Send full URL for same origin; nothing for cross origin.

**Behavior**:
- Same origin: full URL.
- Cross origin: no referrer.
- Downgrade: no referrer.

**Use case**: strict; prevents any information leaking to external sites.

### 6.5 strict-origin

```html
<meta name="referrer" content="strict-origin">
```

Send origin only; suppress on downgrade.

**Behavior**:
- Same origin: just origin.
- Cross origin same protocol: just origin.
- HTTPS to HTTP downgrade: no referrer.

**Use case**: balances privacy with cross-site analytics; protects against downgrade.

### 6.6 strict-origin-when-cross-origin

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

The modern default. Send full URL same origin; origin cross origin; suppress on downgrade.

**Behavior**:
- Same origin: full URL.
- Cross origin same protocol: just origin.
- HTTPS to HTTP downgrade: no referrer.

**Use case**: the modern web default. Balances privacy, security, and analytics needs.

### 6.7 no-referrer-when-downgrade

```html
<meta name="referrer" content="no-referrer-when-downgrade">
```

The historical default. Send full URL except on HTTPS to HTTP downgrade.

**Behavior**:
- Same origin: full URL.
- Cross origin same protocol: full URL.
- HTTPS to HTTP downgrade: no referrer.

**Use case**: legacy compatibility. Not recommended for new sites; modern browsers default to stricter behavior.

### 6.8 no-referrer

```html
<meta name="referrer" content="no-referrer">
```

Never send a Referer header.

**Behavior**:
- All requests: no referrer at all.

**Use case**: maximum privacy. Affiliate marketing and analytics often break.

### 6.9 The Values Summary Table

| Value | Same origin | Cross origin | Downgrade |
|---|---|---|---|
| `unsafe-url` | Full URL | Full URL | Full URL |
| `no-referrer-when-downgrade` | Full URL | Full URL | None |
| `origin` | Origin | Origin | Origin |
| `origin-when-cross-origin` | Full URL | Origin | Full or Origin |
| `same-origin` | Full URL | None | None |
| `strict-origin` | Origin | Origin | None |
| `strict-origin-when-cross-origin` (modern default) | Full URL | Origin | None |
| `no-referrer` | None | None | None |

### 6.10 The Bubbles Default Choice

For most Bubbles client sites: `strict-origin-when-cross-origin`.

This matches the modern browser default, balances privacy with functionality, and is the recommended starting point.

---

## 7. THE MODERN BROWSER DEFAULT (STRICT-ORIGIN-WHEN-CROSS-ORIGIN)

In 2020, Chrome 85 changed its default referrer policy from `no-referrer-when-downgrade` to `strict-origin-when-cross-origin`. Other major browsers followed.

### 7.1 The Change

**Before (pre 2020)**:

```
Default: no-referrer-when-downgrade
Behavior: full URL on same protocol; nothing on downgrade.
Implication: external sites received your full URL (with path, query).
```

**After (2020+)**:

```
Default: strict-origin-when-cross-origin
Behavior: full URL same origin; origin only cross origin; nothing on downgrade.
Implication: external sites only see your origin (https://example.com/).
```

### 7.2 What Changed In Practice

For sites that DID NOT set a policy:

* **Internal links**: same as before; full URL.
* **Cross origin links**: now only origin, not full URL.
* **HTTPS to HTTP**: same as before; no referrer.

For sites with sensitive query parameters in URLs:

* **External link clicks**: no longer leak full URLs.
* Improvement in default privacy posture.

### 7.3 The Implication For Sites

* **Sites that need full URL referrer for analytics**: may have broken silently in 2020.
* **Sites that didn't care**: no observable change for most users.
* **Sites that DID set `unsafe-url` explicitly**: still behave the same.

For Bubbles: the default change is generally positive. Sites without explicit policy now have better default privacy.

### 7.4 The Explicit Declaration Value

Setting `strict-origin-when-cross-origin` explicitly:

* Matches modern default behavior.
* Documents the intent (rather than relying on browser implicit behavior).
* Survives potential future default changes.

Convention: explicitly declare `strict-origin-when-cross-origin` (or appropriate value per use case) rather than relying on implicit defaults.

---

## 8. PRIVACY IMPLICATIONS

Referrer policy directly affects user privacy.

### 8.1 What Referer Reveals

When a user clicks from page A to page B, page B's server receives:

* The URL of page A (or origin only, depending on policy).
* This is visible in the destination's server logs.
* Used by analytics tools, affiliate networks, security systems.

For external sites: they learn where users came from.

### 8.2 The Path Privacy Concern

A URL like:

```
https://medical-site.com/conditions/depression/treatment-options/
```

reveals:

* User was researching depression.
* User was looking for treatment.

Without referrer policy, this URL gets sent to every external site the user navigates to (Google Analytics on that destination, advertising scripts, etc.). The destination's operators learn what the user was viewing on your site.

For Arkansas Counseling and Wellness, this is a HIPAA adjacent concern (not strictly HIPAA since referer is browser behavior, not medical record, but the privacy intent is similar).

### 8.3 The Query Parameter Privacy Concern

URLs with query parameters:

```
https://example.com/account/?session=abc123&user=joseph
```

Without referrer policy:

* External sites receive the session token and username in their logs.
* Browser developer tools, browser history, and any logging on the destination capture this.
* Session hijacking risk; credential exposure.

### 8.4 The Fix

For YMYL clients (mental health, financial), the privacy fix:

```html
<meta name="referrer" content="same-origin">
<!-- Cross origin requests get no referrer -->
```

Or:

```html
<meta name="referrer" content="no-referrer">
<!-- Maximum privacy; analytics may break -->
```

For general sites:

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
<!-- Modern default; origin shared cross origin -->
```

### 8.5 The Tradeoff With Affiliate

Affiliate tracking often depends on referrer. Stricter policies can break commission attribution.

For sites with affiliate programs (real estate clients with Zillow affiliate, for example):

* `strict-origin-when-cross-origin`: origin sufficient for attribution.
* `same-origin` or `no-referrer`: affiliate tracking broken.

Convention: use `strict-origin-when-cross-origin` to preserve affiliate functionality while protecting path level privacy.

---

## 9. SECURITY IMPLICATIONS (URL TOKEN LEAKAGE)

Referer header can leak sensitive information beyond privacy concerns.

### 9.1 The Session Token Pattern

Some sites include session tokens in URLs:

```
https://example.com/dashboard/?token=eyJhbGciOiJIUzI1NiJ9...
```

Without strict referrer policy:

* User clicks external link from dashboard.
* External site receives full URL including token.
* External site (or anyone with log access) can use the token to impersonate the user.

### 9.2 The Password Reset Token Pattern

Password reset emails often contain URLs:

```
https://example.com/reset-password/?token=abc123xyz&user=joseph
```

When user clicks the link and the page renders, any external resources loaded:

* Receive the full URL as referrer.
* Token is leaked to third party CDNs, analytics, etc.

Fix:

* Use one time tokens.
* Set restrictive referrer on password reset pages specifically.
* Move tokens to POST body or session cookies.

### 9.3 The Internal Path Disclosure

Internal paths can reveal architecture:

```
https://example.com/admin/users/123/profile/
```

Without referrer policy, external sites learn:

* `/admin/` exists on the site.
* User IDs are sequential (123).
* Profile pages exist at this pattern.

This is information disclosure that aids attackers.

### 9.4 The Fix Strategy

Multi layer:

1. **Set restrictive referrer policy** (HTTP header preferred).
2. **Don't put sensitive tokens in URLs** (use POST body or cookies).
3. **Don't expose internal paths in URLs** (use opaque identifiers).
4. **Use HTTPS exclusively** (referrer is suppressed on downgrade by modern defaults).

### 9.5 The Bubbles Hardening

For every Bubbles client site, especially those with sensitive paths:

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name example.com;

    # Referrer policy (HTTP header; preferred over meta tag)
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # ... other directives ...
}
```

```bash
nginx -t && systemctl reload nginx
```

Plus optionally on specific sensitive pages:

```python
# /opt/bubbles/services/example.com/main.py
@app.get("/password-reset/")
async def password_reset(request: Request, response: Response):
    response.headers["Referrer-Policy"] = "no-referrer"
    return templates.TemplateResponse("password_reset.html", {
        "request": request,
    })
```

---

## 10. THE HTTP REFERRER-POLICY HEADER RELATIONSHIP

The HTTP header vs HTML meta tag distinction.

### 10.1 The Two Layers

**HTTP layer (preferred)**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

**HTML meta tag layer**:

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

### 10.2 The Differences

| Aspect | HTTP header | HTML meta tag |
|---|---|---|
| Scope | All pages served by config | Specific page |
| Authority | Higher | Lower |
| Timing | Set before HTML parsing | Set during HTML parsing |
| Coverage | All responses (including 404s, redirects) | HTML pages only |
| Granularity | Per route via nginx location | Per page |
| Browser support | Universal | Universal |

### 10.3 When Both Are Set

If both HTTP header AND meta tag are present:

* HTTP header typically wins (specs vary slightly).
* Browser may use the more specific or stricter.

For consistency: align both to the same value.

### 10.4 The Preferred Pattern For Bubbles

Set via HTTP header at nginx (covers all pages including 404s):

```nginx
# /etc/nginx/snippets/security-headers.conf
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
# ... etc
```

```nginx
# /etc/nginx/sites-enabled/example.com.conf
server {
    # ...
    include /etc/nginx/snippets/security-headers.conf;
    # ...
}
```

```bash
nginx -t && systemctl reload nginx
```

This sets the policy site wide.

For HTML meta tag as fallback (or override on specific pages):

```html
<!-- Specific page needs different policy -->
<meta name="referrer" content="no-referrer">
```

### 10.5 The Cross Reference

The HTTP Referrer-Policy header is the canonical reference in framework-http-security-headers.md. This HTML meta tag framework is the supplementary HTML layer.

For most Bubbles client work: configure at HTTP layer; rarely need the meta tag.

For specific pages with different needs: use meta tag for per page override.

---

## 11. ANALYTICS AND AFFILIATE TRADEOFFS

Stricter referrer policies break some external services that depend on referrer.

### 11.1 What Depends On Referrer

* **Google Analytics**: referrer source/medium attribution.
* **Affiliate networks**: commission attribution.
* **Some advertising platforms**: click attribution.
* **Anti spam systems**: referrer fingerprinting.
* **Some authentication systems**: referrer based CSRF protection.

### 11.2 The Impact By Policy

| Policy | Analytics impact | Affiliate impact |
|---|---|---|
| `unsafe-url` | Full attribution | Full attribution |
| `no-referrer-when-downgrade` | Full attribution | Full attribution |
| `strict-origin-when-cross-origin` | Origin attribution only | Origin attribution (usually sufficient) |
| `same-origin` | Cross site analytics broken | Affiliate broken |
| `no-referrer` | All cross site tracking broken | Affiliate broken |

### 11.3 The Bubbles Balance

For most clients:

* **Goal**: keep analytics and affiliate functional.
* **Recommendation**: `strict-origin-when-cross-origin` (modern default).
* **Origin level attribution is usually sufficient**:
  - Google Analytics shows "site: example.com" instead of specific page.
  - Affiliate networks attribute commission based on origin.
  - Most tools handle this gracefully.

### 11.4 The Strict Tradeoff

For YMYL or sensitive clients (Arkansas Counseling, Handled Tax):

* Privacy takes priority over analytics granularity.
* `same-origin` or `no-referrer` may be appropriate.
* Loss of cross site analytics is acceptable for privacy compliance.

### 11.5 The Verification

```bash
# Test what referrer is sent
URL=https://example.com/

# Use curl to simulate request from URL
curl -sI "https://external-site.com/" -H "Referer: https://example.com/page/?session=abc"

# Examine logs on external destination (manual)
# Or use a controlled external endpoint to log received Referer
```

For development: set up a test endpoint to log incoming `Referer:` headers and verify policy behavior.

---

## 12. PER LINK AND PER ELEMENT OVERRIDES

For specific elements that need different referrer behavior, use per element attributes.

### 12.1 The `referrerpolicy` Attribute

Can be applied to:

* `<a>` (anchor links).
* `<area>` (image maps).
* `<img>` (images).
* `<iframe>` (embedded frames).
* `<link>` (resource links).
* `<script>` (script tags).

```html
<a href="https://external.com/" referrerpolicy="no-referrer">
    External link with no referrer
</a>

<img src="https://cdn.example.com/image.jpg" referrerpolicy="no-referrer">

<iframe src="https://embed.example.com/" referrerpolicy="strict-origin"></iframe>
```

### 12.2 The `rel="noreferrer"` Attribute

The legacy method specifically for anchor links:

```html
<a href="https://external.com/" rel="noreferrer">
    External link with no referrer (legacy syntax)
</a>
```

The `rel="noreferrer"` is functionally equivalent to `referrerpolicy="no-referrer"` for anchor links. Both work; the new attribute is more general.

### 12.3 The rel="noreferrer noopener" Pattern

Common pattern for external links:

```html
<a href="https://external.com/" target="_blank" rel="noreferrer noopener">
    External link in new tab
</a>
```

* `target="_blank"`: open in new tab.
* `noreferrer`: don't send referrer.
* `noopener`: prevent the new tab from accessing window.opener of the originating page (security).

For external links opening in new tabs: include both `noopener` and `noreferrer`.

### 12.4 The Per Image Override

For images loaded from third party CDNs:

```html
<img src="https://cdn.images.com/photo.jpg" referrerpolicy="no-referrer">
```

Prevents the CDN from learning which page the image is embedded on (sometimes useful for privacy of internal pages).

### 12.5 The Per iframe Override

For embedded content:

```html
<iframe src="https://embed.example.com/widget/" referrerpolicy="strict-origin"></iframe>
```

The embedded site only learns the origin, not the specific page.

### 12.6 The Bubbles Application

For typical Bubbles client sites:

* **Default referrer policy** at HTTP header level (`strict-origin-when-cross-origin`).
* **External links to competitors**: `rel="noreferrer noopener"` (don't tell them you exist with referrer).
* **External links to partners**: standard (preserve referrer for attribution).
* **Affiliate links**: standard (preserve for attribution).
* **Sensitive pages (password reset, etc.)**: meta tag override to `no-referrer`.

---

## 13. THE BUBBLES PER CLIENT DECISION FRAMEWORK

Each client site needs a referrer policy decision.

### 13.1 The Decision Questions

1. **What is the site's privacy posture?**
   - Standard: `strict-origin-when-cross-origin`.
   - YMYL: `same-origin` or stricter.
   - Federal: `strict-origin-when-cross-origin` minimum.

2. **Does the site depend on cross site analytics or affiliate?**
   - Yes: `strict-origin-when-cross-origin` (preserves origin attribution).
   - No: stricter is acceptable.

3. **Are there sensitive paths or query parameters?**
   - Yes: stricter policy, plus per page overrides.
   - No: standard policy sufficient.

4. **Federal subcontracting context?**
   - Yes: align with NIST/CIS recommendations.
   - No: standard recommendations apply.

### 13.2 The Per Client Recommendations

**ThatDeveloperGuy.com (Joseph's primary site)**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Standard policy; balanced for analytics and privacy.

**TCB Fight Factory**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Standard policy.

**Arkansas Counseling and Wellness Services (Dr. Kristy Burton)**:

```nginx
add_header Referrer-Policy "same-origin" always;
```

YMYL mental health; protect path information from external sites.

**Handled Tax and Advisory (Amanda Emerdinger)**:

```nginx
add_header Referrer-Policy "same-origin" always;
```

YMYL financial; protect path information.

**White River Cabins**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Standard travel/hospitality; balance with affiliate (booking referrer attribution).

**Greenough's Guide Service**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Standard outdoor/recreation.

**Local Living Real Estate (Laycee Maupin)**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Standard; preserves real estate affiliate attribution (Zillow, etc.).

**Diana Undergust at Lake Homes Realty**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Standard; same reasoning as above.

**WeCoverUSA (federal subcontractor)**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Plus stricter for sensitive routes:
location /restricted/ {
    add_header Referrer-Policy "same-origin" always;
}
```

Federal baseline; stricter for sensitive paths.

**Heritage Hardwood Floors NWA**:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Standard local business.

### 13.3 The Documentation Pattern

For each client project:

```
Referrer policy decision:
  Default: strict-origin-when-cross-origin
  Reasoning: standard balance for analytics and privacy
  Set via: HTTP header at nginx
  Specific page overrides: none

Or for YMYL:

Referrer policy decision:
  Default: same-origin
  Reasoning: YMYL mental health; protect path information
  Set via: HTTP header at nginx
  Specific page overrides: none
```

---

## 14. THE FEDERAL SUBCONTRACTOR PATTERN (SDVOSB)

For Joseph's federal subcontracting work and the SDVOSB context, referrer policy aligns with security baselines.

### 14.1 The NIST 800-53 Connection

NIST 800-53 (the federal information system security baseline) includes controls about information disclosure. Specifically:

* **SC-8 (Transmission Confidentiality and Integrity)**: protect transmitted information.
* **SC-23 (Session Authenticity)**: protect session information.
* **AC-21 (Information Sharing)**: minimize unauthorized sharing.

Referrer policy is one mechanism for compliance with these controls.

### 14.2 The CIS Benchmark Recommendation

CIS Benchmarks (for various web servers and applications) recommend:

* Setting `Referrer-Policy` header.
* Recommended value: `strict-origin-when-cross-origin` or stricter.
* Documenting the choice.

### 14.3 The Federal Pattern

For WeCoverUSA and other Joseph federal subcontracting:

```nginx
# /etc/nginx/sites-enabled/wecoverusa.com.conf

server {
    listen 443 ssl http2;
    server_name wecoverusa.com;

    # Federal baseline: strict-origin-when-cross-origin minimum
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Sensitive paths: stricter policy
    location /federal-portal/ {
        add_header Referrer-Policy "same-origin" always;
    }

    location /agent-portal/ {
        add_header Referrer-Policy "no-referrer" always;
    }

    # ... rest of config ...
}
```

### 14.4 The Audit Documentation

For federal subcontract documentation:

```
Referrer policy implementation:
  Standard pages: strict-origin-when-cross-origin
  Federal portal: same-origin
  Agent portal (sensitive): no-referrer
  
Rationale: protect path information for sensitive areas while
preserving analytics and operational functionality for standard pages.
Aligns with NIST 800-53 AC-21 and SC-8 controls.

Configured via: nginx HTTP response headers.
Verification: curl -sI <URL> | grep Referrer-Policy
```

### 14.5 The Audit Verification

```bash
# Verify policy is set on all federal subcontractor sites
for url in https://wecoverusa.com/ https://wecoverusa.com/federal-portal/ https://wecoverusa.com/agent-portal/; do
    POLICY=$(curl -sI "$url" | grep -i "Referrer-Policy" | head -1 | tr -d '\r')
    echo "$url: $POLICY"
done
```

---

## 15. THE YMYL PRIVACY PATTERN (HEALTHCARE AND FINANCIAL)

For YMYL clients with privacy obligations, stricter referrer policies are appropriate.

### 15.1 The Healthcare Pattern (Arkansas Counseling)

For Arkansas Counseling and Wellness Services:

```nginx
add_header Referrer-Policy "same-origin" always;
```

Reasoning:

* Patient privacy is paramount.
* External sites should not learn which counseling resource pages users visited.
* Same origin policy: internal navigation preserves full URL; external links send no referrer.

Plus per page overrides for highly sensitive pages:

```python
# /opt/bubbles/services/arcounselingandwellness.com/main.py

@app.get("/intake-form/")
async def intake_form(request: Request, response: Response):
    response.headers["Referrer-Policy"] = "no-referrer"
    # Maximum privacy on intake form
    return templates.TemplateResponse("intake.html", {"request": request})

@app.get("/conditions/{condition}/")
async def condition_page(condition: str, request: Request, response: Response):
    response.headers["Referrer-Policy"] = "no-referrer"
    # Maximum privacy on condition specific pages
    return templates.TemplateResponse("condition.html", {
        "request": request,
        "condition": condition,
    })
```

### 15.2 The Financial Pattern (Handled Tax)

For Handled Tax and Advisory (Amanda Emerdinger):

```nginx
add_header Referrer-Policy "same-origin" always;
```

Reasoning:

* Tax client privacy.
* External sites should not learn what tax topics users explored.
* Same origin keeps internal navigation functional.

Per page overrides for sensitive areas:

```python
@app.get("/client-portal/")
async def client_portal(request: Request, response: Response):
    response.headers["Referrer-Policy"] = "no-referrer"
    return templates.TemplateResponse("portal.html", {"request": request})
```

### 15.3 The Privacy Documentation

For YMYL clients, document the policy:

```
Privacy policy excerpt:
  We minimize information shared with third party sites.
  When users navigate from our site to external destinations,
  we only share our domain origin (e.g., "arcounselingandwellness.com")
  not the specific page visited. This protects information about
  what counseling resources you may have been researching.

Technical implementation:
  HTTP Referrer-Policy: same-origin
  Plus per page overrides for sensitive content.
```

This privacy policy language can appear in the client's privacy policy page.

### 15.4 The Combined Approach

For Bubbles YMYL clients:

1. **Default policy**: `same-origin` site wide.
2. **Per page overrides**: `no-referrer` for highly sensitive pages.
3. **Privacy policy documentation**: explains the approach.
4. **Annual review**: as regulations evolve.

---

## 16. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 16.1 Canonical Bubbles head (with meta tag fallback)

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="color-scheme" content="light dark">
    <meta name="referrer" content="strict-origin-when-cross-origin">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- ... -->
</head>
```

The HTTP header is the canonical source; the meta tag is a fallback.

### 16.2 nginx HTTP header (preferred Bubbles configuration)

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name example.com;

    # Modern default: strict-origin-when-cross-origin
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # ... rest of config ...
}
```

```bash
nginx -t && systemctl reload nginx

# Verify
curl -sI https://example.com/ | grep -i "Referrer-Policy"
# Expected: referrer-policy: strict-origin-when-cross-origin
```

### 16.3 YMYL pattern (healthcare or financial)

```nginx
# Arkansas Counseling: same-origin
server {
    listen 443 ssl http2;
    server_name arcounselingandwellness.com;

    add_header Referrer-Policy "same-origin" always;

    # Sensitive paths: no-referrer
    location /intake/ {
        add_header Referrer-Policy "no-referrer" always;
    }

    location /conditions/ {
        add_header Referrer-Policy "no-referrer" always;
    }
}
```

### 16.4 Federal subcontractor pattern

```nginx
# WeCoverUSA
server {
    listen 443 ssl http2;
    server_name wecoverusa.com;

    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Federal portal: same-origin
    location /federal-portal/ {
        add_header Referrer-Policy "same-origin" always;
    }

    # Agent portal (sensitive): no-referrer
    location /agent-portal/ {
        add_header Referrer-Policy "no-referrer" always;
    }
}
```

### 16.5 Per page override via meta tag

```html
<!-- /opt/bubbles/services/example.com/templates/password_reset.html -->
<head>
    <meta charset="utf-8">
    <meta name="referrer" content="no-referrer">
    <!-- Override site default for this specific sensitive page -->
</head>
```

### 16.6 Per link overrides

```html
<!-- External link, no referrer -->
<a href="https://external.com/" rel="noreferrer noopener" target="_blank">
    Visit External Site
</a>

<!-- External link with per element policy -->
<a href="https://partner.com/" referrerpolicy="strict-origin">
    Partner Link
</a>

<!-- Image from external CDN with no referrer -->
<img src="https://cdn.images.com/photo.jpg" referrerpolicy="no-referrer" alt="Photo">

<!-- Iframe with strict origin -->
<iframe src="https://embed.example.com/widget/" referrerpolicy="strict-origin"></iframe>
```

### 16.7 FastAPI per request policy

```python
# Set referrer policy dynamically per request
from fastapi import FastAPI, Request, Response

app = FastAPI()

@app.middleware("http")
async def set_referrer_policy(request: Request, call_next):
    response = await call_next(request)

    # Standard pages: strict-origin-when-cross-origin
    policy = "strict-origin-when-cross-origin"

    # Override for sensitive paths
    if request.url.path.startswith("/intake/"):
        policy = "no-referrer"
    elif request.url.path.startswith("/conditions/"):
        policy = "no-referrer"
    elif request.url.path.startswith("/admin/"):
        policy = "same-origin"

    response.headers["Referrer-Policy"] = policy
    return response
```

### 16.8 Outbound link audit

```bash
#!/bin/bash
# /usr/local/bin/external-links-audit.sh
# Check all outbound links for proper referrer protection

URL=$1

# Get all external links from page
EXTERNAL=$(curl -s "$URL" | grep -oE 'href="https?://[^"]+"' | sort -u | grep -v "$(curl -s "$URL" | grep -oE 'meta[^>]+name="og:url"[^>]+' | sed 's/.*content="\([^"]*\)".*/\1/' | sed 's/https\?:\/\/\([^/]*\).*/\1/')")

echo "External links on $URL:"
echo "$EXTERNAL"

# Check if they have noreferrer
for link in $EXTERNAL; do
    if curl -s "$URL" | grep -E "href=\"$link\"[^>]*rel=\"[^\"]*noreferrer" > /dev/null; then
        : # OK
    else
        echo "EXTERNAL WITHOUT noreferrer: $link"
    fi
done
```

### 16.9 Bulk audit across all Bubbles client sites

```bash
#!/bin/bash
# /usr/local/bin/referrer-policy-audit.sh

echo "=== Referrer policy audit across Bubbles sites ==="

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")

    # Get the URL from site directory naming
    URL="https://${SITE}"

    POLICY=$(curl -sI -L "$URL/" 2>/dev/null | grep -i "Referrer-Policy" | head -1 | tr -d '\r')

    if [ -z "$POLICY" ]; then
        echo "MISSING: $SITE - no Referrer-Policy header"
    else
        echo "$SITE: $POLICY"
    fi
done
```

### 16.10 Verify policy enforcement

```bash
# Use a test endpoint to log incoming Referer headers
# (Could use webhook.site or a controlled FastAPI endpoint)

# /opt/bubbles/test-endpoints/referrer_logger.py
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/referrer-test")
async def log_referrer(request: Request):
    referer = request.headers.get("referer", "(none)")
    print(f"Received Referer: {referer}")
    return {"received_referrer": referer}

# Then visit your site, click a link to this endpoint, verify the Referer received
```

---

## 17. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles referrer policy configuration.

### 17.1 The nginx Default

```nginx
# /etc/nginx/snippets/security-headers.conf

# Referrer-Policy
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Other security headers
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
# ... other from framework-http-security-headers.md
```

```nginx
# /etc/nginx/sites-enabled/example.com.conf
server {
    # ...
    include /etc/nginx/snippets/security-headers.conf;
    # ...
}
```

### 17.2 The Per Client Configuration

```python
# /opt/bubbles/services/example.com/client_config.py

CLIENT_REFERRER_POLICIES = {
    "thatdeveloperguy": {
        "default": "strict-origin-when-cross-origin",
        "overrides": {},
    },
    "tcb_fight_factory": {
        "default": "strict-origin-when-cross-origin",
        "overrides": {},
    },
    "arcounselingandwellness": {
        "default": "same-origin",
        "overrides": {
            "/intake/": "no-referrer",
            "/conditions/": "no-referrer",
        },
    },
    "handledtax": {
        "default": "same-origin",
        "overrides": {
            "/client-portal/": "no-referrer",
        },
    },
    "wecoverusa": {
        "default": "strict-origin-when-cross-origin",
        "overrides": {
            "/federal-portal/": "same-origin",
            "/agent-portal/": "no-referrer",
        },
    },
    # ... per client
}
```

### 17.3 The HTML Fallback Template

```html
{# /opt/bubbles/services/example.com/templates/base.html #}
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Referrer policy fallback (HTTP header is canonical) -->
    {% if referrer_policy %}
    <meta name="referrer" content="{{ referrer_policy }}">
    {% endif %}

    <!-- Other meta tags -->
    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    {% block head_extra %}{% endblock %}
</head>
<body>
    {% block content %}{% endblock %}
</body>
</html>
```

### 17.4 The Audit Workflow

```bash
# Weekly across all sites
referrer-policy-audit.sh

# After nginx config changes
nginx -t && systemctl reload nginx

# Verify on sample URLs
curl -sI https://example.com/ | grep -i "Referrer-Policy"
curl -sI https://example.com/intake/ | grep -i "Referrer-Policy"
```

### 17.5 The Client Documentation Template

For each client project file:

```
Referrer Policy Configuration:

Default policy: strict-origin-when-cross-origin
Reasoning: standard balance for analytics and privacy

Per route overrides:
  /intake/: no-referrer (sensitive)
  /conditions/: no-referrer (sensitive)

Set via: HTTP response header in nginx
Verification: curl -sI <URL> | grep Referrer-Policy

Alignment with HTTP security headers: see framework-http-security-headers.md.
```

---

## 18. AUDIT CHECKLIST

Run through these 40 items for production deployment.

### Core policy

1. [ ] HTTP `Referrer-Policy` header set in nginx for all sites.
2. [ ] Default value is `strict-origin-when-cross-origin` (or more restrictive per client).
3. [ ] Policy applies to all responses (`always` keyword in nginx).
4. [ ] Verified via `curl -sI` on multiple URLs.

### Per route overrides

5. [ ] Sensitive routes (intake, password reset, admin) have stricter policies.
6. [ ] YMYL clients have `same-origin` minimum at default.
7. [ ] Federal portal routes have appropriate stricter policy.

### HTML meta tag

8. [ ] Meta tag included as fallback for environments without HTTP header control.
9. [ ] Meta tag value matches HTTP header (consistency).
10. [ ] Meta tag uses correctly spelled "referrer" (not "referer").

### YMYL specific

11. [ ] Arkansas Counseling and Wellness: `same-origin` policy.
12. [ ] Handled Tax: `same-origin` policy.
13. [ ] Per page overrides for highly sensitive content.

### Federal subcontractor

14. [ ] WeCoverUSA: `strict-origin-when-cross-origin` baseline.
15. [ ] Sensitive federal routes: stricter (`same-origin` or `no-referrer`).
16. [ ] Aligns with NIST 800-53 information disclosure controls.

### Per link / element

17. [ ] External links use `rel="noreferrer noopener"` where appropriate.
18. [ ] External CDN images use `referrerpolicy="no-referrer"` where appropriate.
19. [ ] Iframes have appropriate referrerpolicy attribute.

### Analytics and affiliate

20. [ ] Affiliate functionality verified working (origin attribution sufficient).
21. [ ] Google Analytics receiving expected data.
22. [ ] No broken cross site tracking that should work.

### Privacy compliance

23. [ ] Privacy policy documents referrer behavior.
24. [ ] HIPAA/financial privacy alignment for YMYL clients.
25. [ ] GDPR data sharing minimization.

### Header coordination

26. [ ] HTTP header takes precedence over meta tag (verified).
27. [ ] All Bubbles security headers consistently set (snippets/security-headers.conf).

### Documentation

28. [ ] Per client referrer policy documented.
29. [ ] Rationale for policy choice documented.
30. [ ] Override routes documented.

### Verification

31. [ ] Test endpoint logs verify expected Referer values.
32. [ ] Browser DevTools network tab shows expected headers.
33. [ ] `referrer-policy-audit.sh` shows all sites configured.

### Mistake checking

34. [ ] No `unsafe-url` policy on production sites.
35. [ ] No `no-referrer-when-downgrade` (legacy; use modern default).
36. [ ] Spelling correct in meta tags ("referrer" not "referer").

### Cross cutting

37. [ ] Login pages have appropriate referrer policy.
38. [ ] Password reset pages have `no-referrer` policy.
39. [ ] API endpoints have appropriate policy.

### After deployment

40. [ ] Post deploy verification via `curl -sI` confirms all expected policies.

A site that passes all 40 has correctly configured referrer policy across all routes with appropriate privacy levels per client context.

---

## 19. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Misspelled meta name as "referer".**
Symptom: `<meta name="referer" content="...">` ignored by browsers.
Why it breaks: meta tag name must be "referrer" (two R's).
Fix: change to `<meta name="referrer" content="...">`.

**Pitfall 2: No policy set; default `unsafe-url` behavior in old browsers.**
Symptom: full URLs sent to external sites including sensitive query parameters.
Why it breaks: no explicit policy; legacy browsers may default permissively.
Fix: set explicit policy (modern default `strict-origin-when-cross-origin`).

**Pitfall 3: `unsafe-url` set explicitly.**
Symptom: full URLs sent to all destinations.
Why it breaks: maximum privacy leakage.
Fix: change to `strict-origin-when-cross-origin` or stricter.

**Pitfall 4: `no-referrer` everywhere breaks affiliate.**
Symptom: affiliate program shows zero attribution.
Why it breaks: too strict; affiliate networks need referrer.
Fix: relax to `strict-origin-when-cross-origin` (origin sufficient for affiliate).

**Pitfall 5: YMYL site has standard policy.**
Symptom: Arkansas Counseling site sends path information to external sites.
Why it breaks: insufficient privacy for healthcare context.
Fix: tighten to `same-origin`.

**Pitfall 6: Session tokens in URL leaked via referrer.**
Symptom: external site logs show URLs containing session tokens.
Why it breaks: tokens in URLs combined with permissive referrer.
Fix: stricter referrer plus move tokens out of URLs.

**Pitfall 7: External links missing `noreferrer noopener`.**
Symptom: clicking external links exposes window.opener; potential security issue.
Why it breaks: target="_blank" without noopener has security implications.
Fix: always use `rel="noreferrer noopener"` for external `target="_blank"` links.

**Pitfall 8: HTML meta tag conflicts with HTTP header.**
Symptom: page sets `<meta name="referrer" content="no-referrer">` but HTTP header says `strict-origin-when-cross-origin`.
Why it breaks: usually HTTP wins; meta tag ignored.
Fix: align both, or remove meta tag if HTTP header is set.

**Pitfall 9: Federal site without policy.**
Symptom: WeCoverUSA security audit flag.
Why it breaks: NIST 800-53 controls reference referrer policy.
Fix: set policy and document.

**Pitfall 10: Policy lost on redirects.**
Symptom: redirects don't include the Referrer-Policy header.
Why it breaks: `add_header` in nginx may not apply to redirect responses without `always` keyword.
Fix: ensure `always` keyword in nginx directive.

**Pitfall 11: Policy set per location but not globally.**
Symptom: most pages have policy; 404 pages don't.
Why it breaks: location specific add_header; not inherited to default 404.
Fix: put add_header in server block, not location block. Or use snippets/include.

**Pitfall 12: WordPress site has no referrer policy.**
Symptom: WordPress site exposes referrer everywhere.
Why it breaks: WordPress doesn't set Referrer-Policy by default.
Fix: add to .htaccess or use security plugin.

**Pitfall 13: Strict policy breaks legitimate cross site tracking.**
Symptom: business intelligence dashboard shows referrer dropoff after policy change.
Why it breaks: `same-origin` or `no-referrer` blocks cross site analytics.
Fix: review what tracking is needed; choose appropriate policy.

**Pitfall 14: Mixing legacy `rel="noreferrer"` and new `referrerpolicy` attribute.**
Symptom: link has both attributes with conflicting values.
Why it breaks: behavior may vary by browser.
Fix: choose one approach per link.

**Pitfall 15: Forgetting policy on password reset emails.**
Symptom: password reset link page sends full URL referrer to embedded resources.
Why it breaks: reset tokens leak via referrer.
Fix: set `no-referrer` on password reset pages specifically; use POST for reset action.

---

## 20. DIAGNOSTIC COMMANDS

Reference of commands useful for referrer policy investigation.

### Inspect HTTP header

```bash
# Check Referrer-Policy header
curl -sI https://example.com/ | grep -i "Referrer-Policy"

# Per route check
curl -sI https://example.com/intake/ | grep -i "Referrer-Policy"

# Verify "always" keyword effect (redirect should also have header)
curl -sIL https://example.com/redirect-source/ | grep -i "Referrer-Policy"
```

### Inspect meta tag

```bash
# Check meta tag
curl -s https://example.com/ | grep -oE 'meta name="referrer" content="[^"]+"'

# Also check for misspelled version (should be absent)
curl -s https://example.com/ | grep -c 'meta name="referer"'
# Expected: 0
```

### Cross check both layers

```bash
URL=$1

HTTP_POLICY=$(curl -sI "$URL" | grep -i "Referrer-Policy" | head -1 | tr -d '\r')
META_POLICY=$(curl -s "$URL" | grep -oE 'meta name="referrer" content="[^"]+"' | head -1)

echo "URL: $URL"
echo "HTTP header: $HTTP_POLICY"
echo "Meta tag: $META_POLICY"

if [ -n "$HTTP_POLICY" ] && [ -n "$META_POLICY" ]; then
    HTTP_VALUE=$(echo "$HTTP_POLICY" | sed 's/.*: *//')
    META_VALUE=$(echo "$META_POLICY" | sed 's/.*content="\([^"]*\)".*/\1/')

    if [ "$HTTP_VALUE" = "$META_VALUE" ]; then
        echo "ALIGNED: same value in both layers"
    else
        echo "MISMATCH: HTTP says '$HTTP_VALUE', meta says '$META_VALUE'"
    fi
fi
```

### Bulk audit across all sites

```bash
referrer-policy-audit.sh
```

### Test what referrer is sent

For development testing, set up a logging endpoint:

```python
# /opt/bubbles/test-endpoints/referrer-logger.py
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/log-referrer")
async def log_referrer(request: Request):
    referer = request.headers.get("referer", "(none)")
    return {
        "received_referer": referer,
        "your_origin": request.headers.get("origin", "(none)"),
    }
```

Run on port 9999:

```bash
uvicorn referrer-logger:app --port 9999
```

Then click link from your site to `http://localhost:9999/log-referrer`; verify the response shows expected Referer value.

### Check outbound links

```bash
# Find external links on a page
curl -s https://example.com/ | grep -oE 'href="https?://[^"]+"' | grep -v "example.com" | sort -u
```

### Verify nginx configuration

```bash
nginx -t

# Or show effective config for a site
nginx -T 2>/dev/null | grep -A 20 "server_name example.com"
```

### After configuration changes

```bash
nginx -t && systemctl reload nginx

# Verify
curl -sI https://example.com/ | grep -i "Referrer-Policy"
```

### Browser DevTools

In Chrome DevTools:

1. Open the page.
2. Open Network tab.
3. Click an outbound link.
4. In the request to the destination, examine `Referer:` header value.
5. Verify it matches expected policy behavior.

---

## 21. CROSS-REFERENCES

* [framework-http-security-headers.md](framework-http-security-headers.md): the HTTP `Referrer-Policy` header is the canonical layer for setting this policy. This HTML meta tag framework is the supplementary layer.
* [framework-html-meta-charset.md](framework-html-meta-charset.md): meta tag placement order in head.
* [framework-html-meta-viewport.md](framework-html-meta-viewport.md): order in head (after charset and viewport).
* [framework-html-meta-color-scheme.md](framework-html-meta-color-scheme.md): order in head.
* [framework-html-meta-author.md](framework-html-meta-author.md): the privacy considerations for author identity tie into referrer privacy considerations.
* [framework-http-performance-headers.md](framework-http-performance-headers.md): the Server header relates to information disclosure (similar privacy concept).
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): referrer policy not ranking but user trust and privacy signals matter.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including security headers.
* W3C Referrer Policy specification: https://www.w3.org/TR/referrer-policy/
* MDN Referrer-Policy HTTP header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy
* MDN meta name=referrer: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name
* MDN referrerpolicy attribute: https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/referrerpolicy
* MDN rel="noreferrer": https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/noreferrer
* NIST SP 800-53 Rev 5: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
* OWASP Information Exposure Through Referrer: https://owasp.org/www-community/attacks/Information_Exposure_Through_Referer

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Set `Referrer-Policy: strict-origin-when-cross-origin` via nginx HTTP header on every site. YMYL clients get `same-origin`.**

### The canonical patterns

```nginx
# Standard
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# YMYL (healthcare, financial)
add_header Referrer-Policy "same-origin" always;

# Federal subcontractor (with per route)
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
location /federal-portal/ {
    add_header Referrer-Policy "same-origin" always;
}
```

### The HTML fallback

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

### The spelling rule

* HTTP request header: `Referer:` (one R, misspelled historically).
* Everything else: `Referrer` (two R's, correct spelling).

### Five rules to memorize

1. **Default: `strict-origin-when-cross-origin` (matches modern browser default).**
2. **YMYL: `same-origin` (Arkansas Counseling, Handled Tax).**
3. **Federal: `strict-origin-when-cross-origin` minimum; stricter per route.**
4. **HTTP header preferred over meta tag.**
5. **External `target="_blank"` links: `rel="noreferrer noopener"`.**

### Five commands every operator should know

```bash
# 1. Check policy
curl -sI URL | grep -i "Referrer-Policy"

# 2. Verify meta tag spelling (should be "referrer" not "referer")
curl -s URL | grep -E 'meta name="(referer|referrer)"'

# 3. Audit all Bubbles sites
referrer-policy-audit.sh

# 4. Test what referrer is sent (with test endpoint)
# Click external link; check destination logs

# 5. Apply changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. HTTP header set
URL=https://example.com/
POLICY=$(curl -sI "$URL" | grep -i "Referrer-Policy" | head -1)
[ -n "$POLICY" ] && echo "OK: $POLICY" || echo "FAIL: no policy"

# 2. Spelling correct in meta tag
WRONG=$(curl -s "$URL" | grep -c 'meta name="referer"')
[ "$WRONG" = "0" ] && echo "OK: spelling correct" || echo "FAIL: misspelled"

# 3. Per route overrides work for YMYL clients
SENSITIVE_POLICY=$(curl -sI "$URL/intake/" | grep -i "Referrer-Policy")
echo "Intake route policy: $SENSITIVE_POLICY"
```

If all three pass AND `curl -sI` consistently shows expected Referrer-Policy header across all routes AND verification endpoint receives expected Referer values, the referrer layer is correctly wired.

---

End of framework-html-meta-referrer.md.
