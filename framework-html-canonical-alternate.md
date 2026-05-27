# framework-html-canonical-alternate.md

Comprehensive reference for the `<link rel="canonical">` and `<link rel="alternate">` link element family, the HTML head signals that control URL deduplication (canonical), international language and regional targeting (hreflang alternates), content syndication (RSS, Atom, and JSON feed alternates), and the now mostly deprecated Accelerated Mobile Pages signal (`rel="amphtml"`). Covers the canonical signal in depth (the basic pattern, self referencing canonical as the default, cross domain canonical for content republishing, the pagination question, the parameter and tracking handling, the mobile and desktop variant case, the HTML head versus HTTP `Link:` header placement methods, and the canonical alignment with `og:url` and the XML sitemap per framework-html-open-graph.md Section 10), the hreflang alternate signal in depth (the three rules that govern every working implementation, IETF BCP 47 language code format, the x-default fallback pattern, the three placement methods, and the Marshallese-Voices multi language deep dive case study), the RSS, Atom, and JSON feed alternate patterns (alternate content type representations for syndication), the AMP signal status in 2026 (Google removed AMP requirement for Top Stories in May 2021; approximately 40% of formerly AMP sites have migrated off; recommended migration pattern is remove `rel="amphtml"`, 301 redirect AMP URLs to canonical, update sitemap), and the Bubbles per client decision framework with the standard self referencing canonical pattern on every page as default plus the full multi language hreflang stack for Marshallese-Voices plus feed alternates for clients with blogs or news content. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, BIND9 nameservers, self hosted origin at 169.155.162.118, trailing slash policy enforced site wide, no Cloudflare or third party CDN in front).

**This is the eighteenth framework in the HTML signal track**, following the meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, color-scheme, referrer, content-language, refresh, Open Graph, Twitter Cards, search engine verification, and App and PWA frameworks. Companion to framework-html-open-graph.md (canonical alignment with og:url), framework-html-content-language.md (the language signal layer that hreflang sits on top of), and framework-http-3xx-status-codes.md (the 301 redirect mechanism that interacts with canonical).

Audience: humans implementing canonical and alternate signals on client sites, AI assistants generating HTML head sections with proper canonical and hreflang alongside Open Graph and Twitter Cards, Bubbles operators enforcing the trailing slash policy in canonical URLs across the 130+ site fleet, multi language site operators managing the Marshallese-Voices English plus Marshallese cluster, editorial content authors setting up blog RSS feed alternates for syndication, federal subcontractors maintaining clean canonical signals for SDVOSB visibility, and anyone troubleshooting "Google Search Console says we have duplicate content without a canonical", "Search Console reports hreflang return tag errors", "Google is showing the wrong language version in international SERPs", "Feedly does not discover our RSS feed", or "AMP versions are still being shown in Google search after we migrated off".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Mental Model: One Canonical, Many Alternates
5. The Canonical Link Element Fundamentals
6. The Canonical Edge Cases
7. The Canonical Implementation Methods
8. The Trailing Slash Policy And Canonical (Bubbles)
9. The Alternate Link Element And Its Five Roles
10. The Hreflang Fundamentals
11. The Three Rules Of Working Hreflang
12. The X-default Pattern
13. The BCP 47 Language Code Reference
14. The Marshallese-Voices Multi Language Case Study
15. The Three Hreflang Placement Methods
16. The RSS Feed Alternate
17. The Atom Feed Alternate
18. The JSON Feed Alternate
19. The Mobile And Desktop Variant Alternate (Legacy)
20. The Amphtml Signal And Its 2026 Status
21. The AMP Migration Off Pattern
22. The Bubbles Per Client Decision Framework
23. The Server Side Rendering Requirement
24. Implementation Patterns By Stack
25. The Pre Launch Audit Checklist
26. The Common Failures And Fixes
27. The Federal And SDVOSB Context
28. Quick Reference Tag Blocks
29. References

---

## 1. DEFINITION

The `<link rel="canonical">` and `<link rel="alternate">` link elements in the HTML `<head>` declare two complementary signals about URL identity and relationships:

* **`rel="canonical"`** declares "this URL is the canonical or preferred representation of this content". Used to deduplicate near identical pages and consolidate ranking signals to one URL.
* **`rel="alternate"`** declares "this other URL is an alternate representation of the same content for a different language, region, content type, format, or platform". Used for international targeting (hreflang), content syndication (RSS, Atom, JSON feeds), and historical mobile or desktop variants.

```html
<head>
    <!-- Canonical: this is the preferred URL for this content -->
    <link rel="canonical" href="https://example.com/page/">

    <!-- Hreflang alternates: same content, other languages -->
    <link rel="alternate" hreflang="en-US" href="https://example.com/page/">
    <link rel="alternate" hreflang="es-US" href="https://example.com/es/page/">
    <link rel="alternate" hreflang="x-default" href="https://example.com/page/">

    <!-- Feed alternates: same content, different formats -->
    <link rel="alternate" type="application/rss+xml" href="https://example.com/feed.xml" title="Site RSS Feed">
    <link rel="alternate" type="application/atom+xml" href="https://example.com/atom.xml" title="Site Atom Feed">
    <link rel="alternate" type="application/json" href="https://example.com/feed.json" title="Site JSON Feed">
</head>
```

The canonical signal is the single most important URL deduplication mechanism in modern SEO. The hreflang alternate signal is the canonical mechanism for international and multilingual targeting on Google and Yandex (Bing ignores hreflang and reads `content-language` and server signals instead, per framework-html-content-language.md Section 6.1).

The `rel="amphtml"` link element historically pointed from a canonical page to its AMP version. In 2026 it is mostly deprecated: Google removed the AMP requirement for Top Stories in May 2021, removed the AMP badge from search results, and approximately 40% of formerly AMP enabled sites have migrated off the format.

---

## 2. WHY IT MATTERS

The canonical signal directly determines:

* **Which URL appears in Google search results.** When multiple URLs serve substantially similar content (trailing slash variants, query parameter variants, www versus non www, http versus https, capitalization variants, session ID variants), Google picks one canonical URL to display. The site can either let Google guess (often wrong) or declare the preferred URL via `rel="canonical"` (almost always followed).
* **Where ranking signals consolidate.** Backlinks, internal links, and engagement signals to non canonical variants of a URL all flow to the canonical version when the signal is correctly declared. Without canonical, signals fragment across duplicate URLs and ranking weakens.
* **How crawl budget is spent.** Google's crawler revisits the canonical URL more often than its duplicates. Sites with proper canonical signals get cleaner index coverage reports and fewer "duplicate without user selected canonical" errors in Search Console.
* **The Open Graph share count consolidation.** Per framework-html-open-graph.md Section 10, og:url should match the canonical URL. Mismatch fragments Facebook share counts across two object identities.

The hreflang alternate signal directly determines:

* **Which language version Google serves in international SERPs.** A US searcher querying in English gets `en-US`; a Marshallese speaker in Springdale gets `mh`. Without hreflang, Google may serve the wrong language version or merge them as duplicates.
* **How multi language ranking signals consolidate.** Properly clustered hreflang variants accumulate ranking signal as a single entity rather than competing as duplicates.

The RSS, Atom, and JSON feed alternates directly enable:

* **Content syndication.** Feed readers (Feedly, Inoreader, NetNewsWire, Reeder), AI assistants, news aggregators, and email subscription services discover the feed via the `<link rel="alternate">` declaration in the homepage `<head>`.
* **Email automation.** Services like Mailchimp and Substack can ingest the feed and generate newsletter content automatically.

For Bubbles client sites specifically, canonical and hreflang directly impact:

* **The trailing slash policy enforcement** (per Joseph's standing rule, all Bubbles client URLs end in `/` for paths). The canonical URL must reflect this.
* **The www versus non www decision** per client (some clients canonicalize to bare domain, some to www; verify per project notes).
* **The 14 paying client 301 redirect pattern** documented in the Bubbles infrastructure overhaul. 301 redirects from non canonical URLs to canonical URLs are the foundation that the `rel="canonical"` signal sits on top of.
* **The Marshallese-Voices multi language implementation** at `/` for English and `/mh/` for Marshallese, with full hreflang reciprocity, x-default to English, and self referencing canonical on each language variant.

---

## 3. WHAT THIS COVERS

This framework covers:

* `<link rel="canonical">` fundamentals: the basic pattern, the self referencing canonical default, when to use cross domain canonical, the pagination handling (`rel="next"` and `rel="prev"` deprecation status), parameter and tracking URL handling.
* The canonical implementation methods: HTML head (the default), HTTP `Link:` header (for non HTML content like PDFs), and the XML sitemap `<loc>` value.
* The Bubbles trailing slash policy and how it shapes canonical URLs.
* `<link rel="alternate">` and its five distinct roles: hreflang (international), feed type (RSS/Atom/JSON), mobile/desktop variant (legacy), AMP (mostly deprecated), and media (print stylesheets and similar; not covered in depth).
* The hreflang signal in depth: the three rules of working hreflang (reciprocity, self reference, canonical alignment), the IETF BCP 47 language code format with common mistakes (`en-UK` invalid, `en-GB` valid), the x-default fallback pattern, and the three placement methods (HTML head, XML sitemap `xhtml:link`, HTTP `Link:` header).
* The Marshallese-Voices multi language case study referencing framework-html-content-language.md Section 10: English at `/`, Marshallese at `/mh/`, full reciprocal hreflang cluster, x-default to English, self referencing canonical on each variant, UTF-8 chain end to end.
* The RSS feed alternate (`application/rss+xml`) for blogs, news, and editorial content.
* The Atom feed alternate (`application/atom+xml`) and its differences from RSS.
* The JSON feed alternate (`application/json`) and the JSON Feed 1.1 specification.
* The mobile and desktop variant alternate pattern (`m.example.com` paired with `media="only screen..."`), now legacy and discouraged in favor of responsive design.
* The `rel="amphtml"` signal status in 2026: mostly deprecated, with the migration off pattern (remove the link, 301 redirect AMP URLs to canonical, update sitemap, monitor for traffic fluctuation).
* The Bubbles per client decision framework with the standard self referencing canonical pattern on every page as default, the full Marshallese-Voices hreflang stack, and feed alternates for blog enabled clients.
* The federal and SDVOSB context with canonical discipline for procurement officer link sharing.

This framework does NOT cover:

* `<link rel="manifest">` (Web App Manifest reference) — covered in framework-html-app-pwa-meta.md.
* `<link rel="apple-touch-icon">` and the icon family — covered in framework-html-favicon-icons.md.
* `<link rel="preconnect">`, `<link rel="dns-prefetch">`, `<link rel="preload">` (performance hints) — covered in framework-html-resource-hints.md.
* `<link rel="stylesheet">` and `<link rel="modulepreload">` — out of scope.
* 301 redirect mechanics — covered in framework-http-3xx-status-codes.md.

---

## 4. THE MENTAL MODEL: ONE CANONICAL, MANY ALTERNATES

```
LAYER 1: The single canonical (one per page)
   |
   |---> <link rel="canonical" href="https://example.com/page/">
         (declares the preferred URL for this content)

LAYER 2: The hreflang alternate cluster (when multi language)
   |
   |---> <link rel="alternate" hreflang="en-US" href="https://example.com/page/">    (self reference)
   |---> <link rel="alternate" hreflang="mh"    href="https://example.com/mh/page/">  (Marshallese)
   |---> <link rel="alternate" hreflang="x-default" href="https://example.com/page/"> (fallback)

LAYER 3: The feed alternates (when site has a feed)
   |
   |---> <link rel="alternate" type="application/rss+xml"  href="/feed.xml"   title="Blog RSS">
   |---> <link rel="alternate" type="application/atom+xml" href="/atom.xml"   title="Blog Atom">
   |---> <link rel="alternate" type="application/json"     href="/feed.json"  title="Blog JSON">

LAYER 4: The legacy alternates (avoid in new builds)
   |
   |---> <link rel="amphtml"   href="...">  (mostly deprecated; migrate off)
   |---> <link rel="alternate" media="only screen and (max-width: 640px)" href="...">  (mobile variant; use responsive design)
```

Seven rules govern the system:

1. **Every page declares its canonical URL.** Self referencing canonical is the default.
2. **The canonical URL is absolute** (full `https://` prefix), not relative.
3. **The canonical URL follows the Bubbles trailing slash policy** (paths end in `/`).
4. **The canonical URL matches og:url** (per framework-html-open-graph.md Section 10) and the XML sitemap entry.
5. **Hreflang clusters require reciprocity**: every variant in the cluster references every other variant and itself.
6. **Hreflang variants must self canonicalize**: the canonical on the French page points to the French page, not the English page.
7. **The AMP link element is mostly deprecated**: migrate off; do not implement on new builds.

A correctly configured Bubbles client site has a self referencing canonical on every page, full reciprocal hreflang cluster on every page of every multi language client, feed alternates declared on every page of every blog enabled client, and zero `rel="amphtml"` references.

---

## 5. THE CANONICAL LINK ELEMENT FUNDAMENTALS

### 5.1 The Basic Pattern

```html
<link rel="canonical" href="https://example.com/page/">
```

Three requirements:

* **Absolute URL**: full `https://` prefix.
* **HTTPS protocol**: HTTP canonical is treated as a downgrade signal.
* **Trailing slash matches Bubbles policy**: paths end in `/`.

### 5.2 The Self Referencing Canonical Default

Every page declares its own URL as canonical:

```html
<!-- On https://example.com/services/refinishing/ -->
<link rel="canonical" href="https://example.com/services/refinishing/">
```

This is the default for almost every page on a Bubbles client site. The self referencing canonical:

* Confirms to Google "this is the canonical URL for this content".
* Prevents accidental canonical conflicts with hreflang (per Section 11).
* Survives URL parameter scenarios where Google might otherwise pick the wrong variant.

### 5.3 The Cross Domain Canonical

When the same content is published on multiple domains (syndication, republishing partners, content licensing), the canonical points to the original publisher:

```html
<!-- On https://syndicate.com/article/ -->
<link rel="canonical" href="https://originalpublisher.com/article/">
```

The syndicating site declares the original publisher as canonical, consolidating ranking signal to the original.

**For Bubbles clients**: rarely applicable. If a Bubbles client republishes Joseph's blog content on their site (and Joseph approves), the cross domain canonical pattern protects against duplicate content treatment.

### 5.4 The Parameter Handling

Tracking parameters (`?utm_source=facebook`, `?fbclid=abc`, `?gclid=xyz`, `?msclkid=def`) create URL variants that serve the same content. The canonical URL strips them:

```html
<!-- User visits https://example.com/page/?utm_source=facebook -->
<!-- Canonical strips the tracking parameter -->
<link rel="canonical" href="https://example.com/page/">
```

Google's canonical processing strips most tracking parameters automatically, but explicit declaration prevents edge cases.

### 5.5 The Pagination Handling

For paginated content (`/blog/`, `/blog/page/2/`, `/blog/page/3/`):

* Each page declares its own canonical (self referencing).
* `/blog/page/2/` canonicalizes to `/blog/page/2/`, NOT to `/blog/`.
* Google deprecated `rel="next"` and `rel="prev"` for pagination in 2019; do not implement them.

```html
<!-- On /blog/page/2/ -->
<link rel="canonical" href="https://example.com/blog/page/2/">
```

This pattern preserves each page's ability to surface independently in search results.

### 5.6 The Mobile And Desktop Variant Canonical (Legacy)

When a separate mobile site exists at `m.example.com` (legacy pattern; modern Bubbles convention is responsive design with a single URL), the canonical from the mobile page points to the desktop:

```html
<!-- On https://m.example.com/page/ -->
<link rel="canonical" href="https://example.com/page/">

<!-- On https://example.com/page/ -->
<link rel="canonical" href="https://example.com/page/">
<link rel="alternate" media="only screen and (max-width: 640px)" href="https://m.example.com/page/">
```

**For Bubbles convention**: avoid this pattern. Use responsive design with a single URL.

### 5.7 The HTTPS And Host Consistency

The canonical URL uses the canonical host. If the site canonicalizes to bare domain, the canonical never includes www. If the site canonicalizes to www, the canonical always includes www.

```html
<!-- If site canonicalizes to bare domain -->
<link rel="canonical" href="https://example.com/page/">

<!-- If site canonicalizes to www -->
<link rel="canonical" href="https://www.example.com/page/">
```

Mismatch between the canonical declaration and the actual canonical host (enforced via 301 redirect) is a frequent source of crawl issues.

### 5.8 The Bubbles Pattern Summary

For every Bubbles client page:

```html
<link rel="canonical" href="https://[client-domain.com]/[path/]">
```

Self referencing, absolute, HTTPS, trailing slash, canonical host.

---

## 6. THE CANONICAL EDGE CASES

### 6.1 The Filter And Sort Parameters On Ecommerce And Listing Pages

For product or listing pages with filter and sort parameters (`/products/?color=red&sort=price_asc`):

* Each filter combination is technically a distinct URL.
* Google would treat them as duplicates without canonical guidance.
* Canonical typically points to the unfiltered base URL.

```html
<!-- On /products/?color=red&sort=price_asc -->
<link rel="canonical" href="https://example.com/products/">
```

Exception: when a specific filter combination is intentionally indexed as a landing page (e.g., `/products/red/` as a curated red product landing page), that landing page has its own self referencing canonical.

### 6.2 The Print Friendly Page

For `/page/?print=1` or `/page/print/` variants designed for print:

```html
<!-- On the print variant -->
<link rel="canonical" href="https://example.com/page/">
```

Print variants canonicalize to the standard view. Or skip print URLs entirely and use `@media print` CSS instead (Bubbles convention).

### 6.3 The Session ID Parameter

For sites that append session IDs to URLs (legacy ecommerce, some PHP applications):

```html
<!-- On /page/?sid=abc123 -->
<link rel="canonical" href="https://example.com/page/">
```

Better: configure the application not to append session IDs at all (use cookies). Canonical is a workaround.

### 6.4 The Capitalization Variants

`/Page/`, `/page/`, `/PAGE/` are technically distinct URLs. Pick one (lowercase by Bubbles convention) and canonicalize:

```html
<!-- On any case variant -->
<link rel="canonical" href="https://example.com/page/">
```

Plus 301 redirects to enforce the canonical case at the nginx layer.

### 6.5 The Trailing Slash Mismatch

This is the most common Bubbles edge case. Per Joseph's policy, all paths end in `/`:

```html
<!-- /page (no trailing slash) 301 redirects to /page/ -->
<!-- Canonical on /page/ is self referencing -->
<link rel="canonical" href="https://example.com/page/">
```

Files with extensions (`/page.html`, `/sitemap.xml`) do not have trailing slashes.

### 6.6 The Canonical To 404

A canonical URL must return HTTP 200. If the canonical target returns 404, the canonical signal is broken.

```html
<!-- WRONG: canonical points to a deleted page -->
<link rel="canonical" href="https://example.com/old-page/">
```

When migrating content, update canonicals to point to the new URL, and 301 redirect old URLs.

### 6.7 The Canonical To Redirect

A canonical URL should return HTTP 200, not 3xx. If the canonical target redirects, Google follows the redirect but the signal is weaker.

```html
<!-- WRONG: canonical points to a URL that 301 redirects -->
<link rel="canonical" href="https://www.example.com/page/">  <!-- redirects to https://example.com/page/ -->

<!-- CORRECT: canonical points to the final 200 URL -->
<link rel="canonical" href="https://example.com/page/">
```

### 6.8 The Canonical To Noindex

A canonical URL must be indexable. If the canonical target has `<meta name="robots" content="noindex">`, the canonical signal is ignored.

This pattern occurs accidentally when a staging URL is left as canonical. Verify the canonical target's robots meta during pre launch audit.

---

## 7. THE CANONICAL IMPLEMENTATION METHODS

Three placement methods exist for the canonical signal.

### 7.1 The HTML Head (Default)

```html
<head>
    <link rel="canonical" href="https://example.com/page/">
</head>
```

The standard placement. Used by every HTML page.

### 7.2 The HTTP Link Header (For Non HTML Content)

For PDFs, images, video files, and other non HTML resources that need a canonical signal:

```
Link: <https://example.com/whitepaper.pdf>; rel="canonical"
```

Configured in nginx:

```nginx
location ~ \.pdf$ {
    add_header Link "<https://example.com/$uri>; rel=\"canonical\"";
}
```

Less common but useful for whitepaper PDFs (ThatDeveloperGuy SDVOSB capability statements, Handled Tax advisory PDFs, WeCoverUSA F&I whitepapers).

### 7.3 The XML Sitemap Loc Value

Every URL in the XML sitemap is implicitly canonical:

```xml
<url>
    <loc>https://example.com/page/</loc>
    <lastmod>2026-05-25</lastmod>
</url>
```

The sitemap `<loc>` URL must match the page's `rel="canonical"` URL.

### 7.4 The Method Consistency Requirement

If the same content is referenced via multiple methods, they must all agree:

* `rel="canonical"` in HTML head.
* `og:url` meta tag.
* XML sitemap `<loc>`.
* HTTP `Link:` header (if used).

Inconsistency confuses Google. Pick a single canonical URL and use it everywhere.

---

## 8. THE TRAILING SLASH POLICY AND CANONICAL (BUBBLES)

Joseph's standing rule: all Bubbles client URLs end in `/` for paths (files with extensions excluded).

### 8.1 The nginx Enforcement Layer

```nginx
# Add trailing slash to paths without extension
location ~ ^([^.]*[^/])$ {
    return 301 $scheme://$host$1/;
}
```

301 redirect enforces the trailing slash at the HTTP layer.

### 8.2 The Canonical Declaration Layer

```html
<!-- On https://example.com/services/ -->
<link rel="canonical" href="https://example.com/services/">
```

Canonical URL includes the trailing slash.

### 8.3 The Alignment Across All Signals

| Signal | Format |
|---|---|
| `<link rel="canonical">` | `https://example.com/services/` |
| `<meta property="og:url">` | `https://example.com/services/` |
| `<meta property="twitter:url">` (rare) | `https://example.com/services/` |
| XML sitemap `<loc>` | `https://example.com/services/` |
| Internal links from other pages | `/services/` |
| nginx 301 enforcement | `/services` → `/services/` |

Every signal points to the same URL. No drift.

### 8.4 The File Exception

Files with extensions do not have trailing slashes:

```html
<!-- Whitepaper PDF -->
<a href="/whitepaper.pdf">Download the whitepaper</a>

<!-- Sitemap (XML file) -->
<!-- /sitemap.xml -->
```

The trailing slash policy applies to directory style paths only.

---

## 9. THE ALTERNATE LINK ELEMENT AND ITS FIVE ROLES

`<link rel="alternate">` is a single attribute (`rel="alternate"`) that serves five distinct purposes depending on other attributes:

### 9.1 Hreflang (International)

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/page/">
```

Covered in Sections 10 through 15.

### 9.2 Feed Type (Syndication)

```html
<link rel="alternate" type="application/rss+xml" href="/feed.xml" title="Blog RSS Feed">
```

Covered in Sections 16, 17, 18.

### 9.3 Media (Mobile And Desktop Variants, Legacy)

```html
<link rel="alternate" media="only screen and (max-width: 640px)" href="https://m.example.com/page/">
```

Covered in Section 19.

### 9.4 AMP (Mostly Deprecated)

```html
<link rel="amphtml" href="https://example.com/amp/page/">
```

Note: `rel="amphtml"` not `rel="alternate"`. Covered in Sections 20, 21.

### 9.5 Media Type Variants (Print Stylesheets, Etc.)

Rarely used in practice. Not covered in depth.

The remainder of the framework treats these five roles separately. The hreflang role is by far the most operationally important for Bubbles.

---

## 10. THE HREFLANG FUNDAMENTALS

`<link rel="alternate" hreflang="...">` declares language and regional alternate versions of the same content.

### 10.1 The Basic Pattern

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/">
<link rel="alternate" hreflang="es-US" href="https://example.com/es/">
<link rel="alternate" hreflang="x-default" href="https://example.com/">
```

Each language variant gets its own `<link>` element. The `hreflang` attribute uses the IETF BCP 47 language tag format (see Section 13).

### 10.2 The Search Engine Behavior

* **Google**: uses hreflang as the primary signal for international targeting. Documented and supported since 2011.
* **Yandex**: uses hreflang similarly to Google.
* **Bing**: ignores hreflang. Uses HTTP `Content-Language` header, `<html lang>` attribute, and server signals. Per framework-html-content-language.md.
* **Baidu, Naver, Seznam**: do not consistently support hreflang.

### 10.3 The Use Case

Three scenarios warrant hreflang:

1. **Same content, different languages** (English and Marshallese versions of Marshallese-Voices).
2. **Same content, different regional variants of the same language** (US English versus UK English for a federal contractor serving both markets).
3. **Mixed**: different languages AND different regions (US English, UK English, Spanish for US Hispanic market, Spanish for Spain market).

### 10.4 The When To Skip Hreflang

For a single language site (most Bubbles clients): skip hreflang entirely. Hreflang on a monolingual site is noise.

For sites targeting multiple English speaking countries with identical content: hreflang may help disambiguate. But often the simpler approach is one English version with content optimized for the primary market.

---

## 11. THE THREE RULES OF WORKING HREFLANG

Three rules govern every working hreflang implementation. Per Google Search Central guidance and confirmed in 2026 industry audits, approximately 65% of enterprise multilingual sites have at least one rule violation that silently invalidates the cluster.

### 11.1 Rule 1: Reciprocity

Every variant in the cluster must reference every other variant. The cluster is bidirectional.

```html
<!-- On the English page -->
<link rel="alternate" hreflang="en-US" href="https://example.com/">
<link rel="alternate" hreflang="mh" href="https://example.com/mh/">

<!-- On the Marshallese page (MUST also include the English reference) -->
<link rel="alternate" hreflang="en-US" href="https://example.com/">
<link rel="alternate" hreflang="mh" href="https://example.com/mh/">
```

If page A references page B but page B does not reference page A, Google Search Console reports the "no return tags" error and may discard the entire cluster.

### 11.2 Rule 2: Self Reference

Every variant must include a hreflang entry pointing to itself.

```html
<!-- On the English page: includes en-US pointing to the English page itself -->
<link rel="alternate" hreflang="en-US" href="https://example.com/">

<!-- On the Marshallese page: includes mh pointing to the Marshallese page itself -->
<link rel="alternate" hreflang="mh" href="https://example.com/mh/">
```

Without self reference, Google does not recognize the page as a valid member of its own cluster.

### 11.3 Rule 3: Canonical Alignment

Every variant must self canonicalize. The canonical on the French page points to the French page; never to the English version.

```html
<!-- On the Marshallese page -->
<link rel="canonical" href="https://example.com/mh/">  <!-- self referencing, NOT the English URL -->

<link rel="alternate" hreflang="en-US" href="https://example.com/">
<link rel="alternate" hreflang="mh" href="https://example.com/mh/">
```

If a variant's canonical points to a different language version, Google treats the variant as a duplicate of the canonical target and discards the entire hreflang cluster. This is the most common silent hreflang failure.

### 11.4 The Three Rules Combined

```
For each page P in the cluster:
  ✓ P has a self referencing <link rel="canonical">
  ✓ P has <link rel="alternate" hreflang="X"> for every variant X in the cluster, including P itself
  ✓ Every other page in the cluster also has <link rel="alternate" hreflang="P"> pointing to P
```

If all three rules hold for every page in the cluster, the cluster is valid.

### 11.5 The Common Failure Modes

* Page A in `en-US`, page B in `es-US`. Canonical on B points to A. Google ignores hreflang.
* Page A references B via hreflang; B does not reference A. Google reports "no return tags".
* Page A uses `en-UK` (invalid; should be `en-GB`). Google silently ignores.
* Page A references a URL that returns 404 or 301 redirects. Cluster weakens or fails.
* Page A's hreflang target has `<meta name="robots" content="noindex">`. Cluster fails for that variant.

---

## 12. THE X-DEFAULT PATTERN

The `x-default` hreflang value declares the fallback URL when no other variant matches the user's language or region.

### 12.1 The Pattern

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/">
<link rel="alternate" hreflang="mh" href="https://example.com/mh/">
<link rel="alternate" hreflang="x-default" href="https://example.com/">
```

When a user's language and region do not match any other declared variant, Google serves the `x-default` URL.

### 12.2 The Common Choice

For most sites: point `x-default` to the English version (or whichever language is the broadest fallback).

For Marshallese-Voices: x-default points to the English version (broader audience).

### 12.3 The Alternative: Language Selector Page

For sites with many language variants and no clear "primary" language, x-default can point to a language selector page:

```html
<link rel="alternate" hreflang="x-default" href="https://example.com/select-language/">
```

The selector page lets the user choose. Rare for Bubbles clients; not applicable in current portfolio.

### 12.4 The Requirement

x-default is strongly recommended but not strictly required. Its absence weakens international targeting for unmatched users.

For Bubbles convention: always include x-default in any hreflang cluster.

---

## 13. THE BCP 47 LANGUAGE CODE REFERENCE

Hreflang values use the IETF BCP 47 language tag format (per framework-html-content-language.md Section 7).

### 13.1 The Format

`[language]` or `[language]-[region]`:

* `en` — English (no region).
* `en-US` — US English.
* `en-GB` — UK English (note: GB, not UK).
* `es` — Spanish (no region).
* `es-US` — US Hispanic Spanish.
* `es-ES` — Spain Spanish.
* `es-MX` — Mexico Spanish.
* `mh` — Marshallese (no region; no widely used regional code).
* `mh-MH` — Marshallese, Marshall Islands (used rarely).

### 13.2 The Hyphen Format

BCP 47 uses hyphens: `en-US`, `es-MX`, `mh`. (Open Graph `og:locale` uses underscores: `en_US`, `es_MX`, `mh_MH`. This is the most common source of confusion across signals.)

### 13.3 The Common Invalid Codes

| Wrong | Correct | Why |
|---|---|---|
| `en-UK` | `en-GB` | UK is not an ISO 3166 country code; GB is. |
| `pt-BZ` | `pt-BR` | BZ is Belize; BR is Brazil. |
| `en_US` | `en-US` | Hyphen, not underscore. |
| `EN-us` | `en-US` | Case matters: language lowercase, region uppercase. |
| `es-419` | `es-419` | 419 is the UN M.49 code for Latin America; valid but uncommon. |

Google silently ignores invalid codes. Pages with invalid hreflang appear to be "set up correctly" but produce no international targeting effect.

### 13.4 The Language Only vs Language Region Choice

* `hreflang="en"` targets all English speakers regardless of region.
* `hreflang="en-US"` targets US English speakers specifically.

Use the most specific code that matches the content. For Marshallese-Voices English content (US specific NWA community focus), `en-US` is more specific than `en`. For content equally applicable to all English speakers, `en` is appropriate.

---

## 14. THE MARSHALLESE-VOICES MULTI LANGUAGE CASE STUDY

Per framework-html-content-language.md Section 10 and framework-html-open-graph.md Section 14, marshallese-voices.org serves English content at `/` and Marshallese content at `/mh/`. The full hreflang and canonical implementation:

### 14.1 The Site Structure

```
https://marshallese-voices.org/              English home
https://marshallese-voices.org/mission/       English mission page
https://marshallese-voices.org/blog/          English blog index
https://marshallese-voices.org/blog/[post]/  English blog post

https://marshallese-voices.org/mh/                  Marshallese home
https://marshallese-voices.org/mh/mission/         Marshallese mission page
https://marshallese-voices.org/mh/blog/             Marshallese blog index
https://marshallese-voices.org/mh/blog/[post]/    Marshallese blog post
```

### 14.2 The English Home Page Pattern

```html
<head>
    <html lang="en-US">

    <link rel="canonical" href="https://marshallese-voices.org/">

    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <meta property="og:url" content="https://marshallese-voices.org/">
    <meta property="og:locale" content="en_US">
    <meta property="og:locale:alternate" content="mh_MH">

    <meta http-equiv="content-language" content="en-US">
    <!-- (deprecated meta tag; the html lang and HTTP Content-Language are primary) -->
</head>
```

### 14.3 The Marshallese Home Page Pattern

```html
<head>
    <html lang="mh">

    <link rel="canonical" href="https://marshallese-voices.org/mh/">

    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <meta property="og:url" content="https://marshallese-voices.org/mh/">
    <meta property="og:locale" content="mh_MH">
    <meta property="og:locale:alternate" content="en_US">
</head>
```

Critical: the canonical on the Marshallese page is `/mh/`, NOT `/`. Pointing the Marshallese canonical to the English URL would discard the entire hreflang cluster.

### 14.4 The Three Rules Verification

* **Rule 1 (reciprocity)**: English page references Marshallese via hreflang, AND Marshallese page references English via hreflang. ✓
* **Rule 2 (self reference)**: English page has `hreflang="en-US"` pointing to itself, AND Marshallese page has `hreflang="mh"` pointing to itself. ✓
* **Rule 3 (canonical alignment)**: English page canonical is `/`, Marshallese page canonical is `/mh/`. Both self referencing. ✓

Plus x-default declared on both pages, pointing to the English version (broader audience fallback). ✓

### 14.5 The Mission Page Variant (Same Pattern)

```html
<!-- English mission at /mission/ -->
<link rel="canonical" href="https://marshallese-voices.org/mission/">
<link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/mission/">
<link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/mission/">
<link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/mission/">

<!-- Marshallese mission at /mh/mission/ -->
<link rel="canonical" href="https://marshallese-voices.org/mh/mission/">
<link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/mission/">
<link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/mission/">
<link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/mission/">
```

Every page pair across the site follows this pattern. The cluster scales by content area, not by language.

### 14.6 The UTF-8 Critical Reminder

Marshallese URLs do not contain diacritics in the path, but the page content does (ā, ē, ī, ō, ū, ṃ, ḷ, ṇ). Per framework-html-meta-charset.md Section 6, the UTF-8 chain must be end to end: HTML charset, nginx charset, form charset, database collation. The canonical and hreflang URLs themselves are ASCII; the issue is the content rendered at those URLs.

### 14.7 The Documentation Note

The Marshallese-Voices site is a Bubbles community focused build. The language handling is respect and accessibility focused; cultural context is referenced respectfully in framework-html-meta-description.md Section 17.

---

## 15. THE THREE HREFLANG PLACEMENT METHODS

Hreflang can be declared in three places. Pick one per site; mixing methods causes conflicts.

### 15.1 Method 1: HTML Head (Default For Bubbles)

```html
<head>
    <link rel="alternate" hreflang="en-US" href="https://example.com/">
    <link rel="alternate" hreflang="mh" href="https://example.com/mh/">
    <link rel="alternate" hreflang="x-default" href="https://example.com/">
</head>
```

The simplest method. Every page in the cluster includes the full hreflang set in its own `<head>`.

**Pros**: easy to implement, easy to audit via View Source.
**Cons**: every page must be updated when a new language variant is added.
**Best for**: Bubbles static HTML sites with manageable number of pages and language variants (Marshallese-Voices fits).

### 15.2 Method 2: XML Sitemap (xhtml:link)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
    <url>
        <loc>https://example.com/</loc>
        <xhtml:link rel="alternate" hreflang="en-US" href="https://example.com/"/>
        <xhtml:link rel="alternate" hreflang="mh" href="https://example.com/mh/"/>
        <xhtml:link rel="alternate" hreflang="x-default" href="https://example.com/"/>
    </url>
    <url>
        <loc>https://example.com/mh/</loc>
        <xhtml:link rel="alternate" hreflang="en-US" href="https://example.com/"/>
        <xhtml:link rel="alternate" hreflang="mh" href="https://example.com/mh/"/>
        <xhtml:link rel="alternate" hreflang="x-default" href="https://example.com/"/>
    </url>
</urlset>
```

**Pros**: centralized; one file controls the entire cluster.
**Cons**: requires sitemap regeneration on every change; not visible to humans browsing HTML.
**Best for**: large multilingual sites with many language variants and frequent additions.

### 15.3 Method 3: HTTP Link Header

```
Link: <https://example.com/>; rel="alternate"; hreflang="en-US",
      <https://example.com/mh/>; rel="alternate"; hreflang="mh",
      <https://example.com/>; rel="alternate"; hreflang="x-default"
```

Configured in nginx:

```nginx
location / {
    add_header Link '<https://example.com/>; rel="alternate"; hreflang="en-US", <https://example.com/mh/>; rel="alternate"; hreflang="mh", <https://example.com/>; rel="alternate"; hreflang="x-default"';
}
```

**Pros**: works for non HTML content (PDFs); centralized in nginx config.
**Cons**: not visible in HTML source; harder to debug.
**Best for**: PDF or non HTML resources with language variants.

### 15.4 The Bubbles Convention

For Marshallese-Voices and any future Bubbles multi language client: HTML head method (Method 1). Simplest, most transparent.

For Bubbles clients with PDFs in multiple languages (rare): HTTP Link header method on those specific files.

---

## 16. THE RSS FEED ALTERNATE

The `<link rel="alternate" type="application/rss+xml">` declares an RSS 2.0 feed for the site.

### 16.1 The Pattern

```html
<link rel="alternate" type="application/rss+xml" href="/feed.xml" title="ThatDeveloperGuy Blog RSS">
```

Three attributes:

* `rel="alternate"`: declares an alternate representation.
* `type="application/rss+xml"`: the MIME type signals RSS.
* `href`: the feed URL.
* `title`: human readable feed name (optional but recommended).

### 16.2 What It Enables

* **Feed readers** (Feedly, Inoreader, NetNewsWire, Reeder, NewsBlur) auto discover the feed when the user pastes the homepage URL.
* **Email automation** services (Mailchimp, Substack) ingest the feed for newsletter generation.
* **AI assistants** (Claude, ChatGPT, Perplexity, Gemini) can be pointed at the feed for content monitoring.
* **Aggregators** and link sharing tools discover the feed for cross posting.

### 16.3 The RSS Feed XML

A minimal RSS feed:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
    <channel>
        <title>ThatDeveloperGuy Blog</title>
        <link>https://thatdeveloperguy.com/blog/</link>
        <description>Web development and engine optimization from Northwest Arkansas.</description>
        <language>en-US</language>
        <atom:link href="https://thatdeveloperguy.com/feed.xml" rel="self" type="application/rss+xml" />
        
        <item>
            <title>Post Title</title>
            <link>https://thatdeveloperguy.com/blog/post-slug/</link>
            <description>Post excerpt or full content.</description>
            <pubDate>Sun, 25 May 2026 08:00:00 -0500</pubDate>
            <guid>https://thatdeveloperguy.com/blog/post-slug/</guid>
        </item>
        <!-- ... more items ... -->
    </channel>
</rss>
```

Note the `<atom:link rel="self">` reference inside the channel; this is a strongly recommended convention for RSS 2.0 feeds.

### 16.4 The MIME Type On The Server

Serve `/feed.xml` with MIME type `application/rss+xml`:

```nginx
location = /feed.xml {
    types { } default_type "application/rss+xml; charset=utf-8";
}
```

End every nginx change with:

```bash
nginx -t && systemctl reload nginx
```

### 16.5 The Bubbles Pattern

For every Bubbles client with a blog:

```html
<link rel="alternate" type="application/rss+xml" href="/feed.xml" title="[CLIENT] Blog RSS">
```

Generate the feed.xml from the blog content. For Bubbles static HTML clients, a build step generates the feed from the blog post Markdown or HTML files.

---

## 17. THE ATOM FEED ALTERNATE

The `<link rel="alternate" type="application/atom+xml">` declares an Atom feed for the site.

### 17.1 The Pattern

```html
<link rel="alternate" type="application/atom+xml" href="/atom.xml" title="ThatDeveloperGuy Blog Atom">
```

Same structure as RSS but uses Atom 1.0 format (RFC 4287).

### 17.2 The Differences From RSS

* Atom 1.0 is a stricter, more modern specification than RSS 2.0.
* Atom requires more explicit metadata (author, updated timestamp, unique ID per entry).
* Atom is preferred by some technical audiences (developer blogs, academic feeds).
* Most feed readers accept both formats interchangeably.

### 17.3 The Atom Feed XML

A minimal Atom feed:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
    <title>ThatDeveloperGuy Blog</title>
    <link href="https://thatdeveloperguy.com/blog/" rel="alternate" type="text/html"/>
    <link href="https://thatdeveloperguy.com/atom.xml" rel="self" type="application/atom+xml"/>
    <id>https://thatdeveloperguy.com/</id>
    <updated>2026-05-25T08:00:00-05:00</updated>
    <author>
        <name>Joseph W. Anady</name>
        <uri>https://thatdeveloperguy.com/about/</uri>
    </author>
    
    <entry>
        <title>Post Title</title>
        <link href="https://thatdeveloperguy.com/blog/post-slug/"/>
        <id>https://thatdeveloperguy.com/blog/post-slug/</id>
        <published>2026-05-25T08:00:00-05:00</published>
        <updated>2026-05-25T08:00:00-05:00</updated>
        <summary>Post excerpt.</summary>
    </entry>
</feed>
```

### 17.4 The Bubbles Pattern

For every Bubbles client blog, providing both RSS and Atom is the comprehensive convention:

```html
<link rel="alternate" type="application/rss+xml" href="/feed.xml" title="[CLIENT] Blog RSS">
<link rel="alternate" type="application/atom+xml" href="/atom.xml" title="[CLIENT] Blog Atom">
```

Two files, generated alongside each other. The audience overlap is partial; some readers prefer one over the other.

---

## 18. THE JSON FEED ALTERNATE

The `<link rel="alternate" type="application/json">` declares a JSON Feed (per the JSON Feed 1.1 specification at https://www.jsonfeed.org).

### 18.1 The Pattern

```html
<link rel="alternate" type="application/json" href="/feed.json" title="ThatDeveloperGuy Blog JSON Feed">
```

### 18.2 The JSON Feed Format

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "ThatDeveloperGuy Blog",
  "home_page_url": "https://thatdeveloperguy.com/blog/",
  "feed_url": "https://thatdeveloperguy.com/feed.json",
  "language": "en-US",
  "authors": [
    {
      "name": "Joseph W. Anady",
      "url": "https://thatdeveloperguy.com/about/"
    }
  ],
  "items": [
    {
      "id": "https://thatdeveloperguy.com/blog/post-slug/",
      "url": "https://thatdeveloperguy.com/blog/post-slug/",
      "title": "Post Title",
      "content_html": "<p>Post content as HTML.</p>",
      "date_published": "2026-05-25T08:00:00-05:00",
      "tags": ["seo", "web development"]
    }
  ]
}
```

### 18.3 The Adoption Status

JSON Feed has growing adoption but is still smaller than RSS and Atom. Major feed readers (Feedly, Inoreader, NetNewsWire) support JSON Feed.

JSON Feed is particularly useful for:

* Programmatic consumption (easier to parse than XML in JavaScript and Python).
* AI assistant integration (LLMs parse JSON more reliably than XML).
* Modern developer workflows.

### 18.4 The Bubbles Pattern

For ThatDeveloperGuy, ThatAIGuy, and any Bubbles client with a developer or AI focused audience:

```html
<link rel="alternate" type="application/json" href="/feed.json" title="JSON Feed">
```

For local service clients: skip JSON Feed; RSS and Atom are sufficient.

### 18.5 The Three Feed Stack

For ThatDeveloperGuy and similar:

```html
<link rel="alternate" type="application/rss+xml" href="/feed.xml" title="Blog RSS">
<link rel="alternate" type="application/atom+xml" href="/atom.xml" title="Blog Atom">
<link rel="alternate" type="application/json" href="/feed.json" title="Blog JSON Feed">
```

Three files, generated together by the build step.

---

## 19. THE MOBILE AND DESKTOP VARIANT ALTERNATE (LEGACY)

```html
<!-- On desktop page -->
<link rel="alternate" media="only screen and (max-width: 640px)" href="https://m.example.com/page/">

<!-- On mobile page (m.example.com) -->
<link rel="canonical" href="https://example.com/page/">
```

The historical pattern for sites with separate `m.example.com` mobile sites.

### 19.1 The 2026 Status

This pattern is legacy. Modern Bubbles convention is responsive design with a single URL.

Google still supports the m. variant pattern but documents responsive design as the recommended approach.

### 19.2 The Bubbles Convention

Do not implement the m. variant pattern on new Bubbles client builds. Use responsive CSS to serve a single URL with mobile optimized rendering.

For inherited client sites with existing m. subdomain: plan migration to responsive design. The migration is operationally similar to the AMP migration off pattern (Section 21).

---

## 20. THE AMPHTML SIGNAL AND ITS 2026 STATUS

`<link rel="amphtml" href="...">` historically pointed from a canonical page to its AMP (Accelerated Mobile Pages) version.

### 20.1 The Historical Pattern

```html
<!-- On the canonical page -->
<link rel="amphtml" href="https://example.com/amp/page/">

<!-- On the AMP page -->
<link rel="canonical" href="https://example.com/page/">
```

The AMP version got cached on Google's servers, served from Google's CDN, and appeared with a lightning bolt badge in mobile search results.

### 20.2 The 2026 Status

* **May 2020**: Google removed the AMP badge from search results.
* **May 2021**: Google removed AMP as a requirement for Top Stories carousel inclusion. Non AMP pages compete equally.
* **2026**: Approximately 40% of formerly AMP enabled sites have migrated off. AMP provides no ranking advantage. Major publishers (Washington Post, CNBC, Vox Media, Bustle Digital Group) have removed AMP entirely.

### 20.3 The Recommendation For New Bubbles Builds

**Do not implement AMP.** Do not include `<link rel="amphtml">` on any new Bubbles client site. AMP adds maintenance overhead with zero ranking benefit in 2026.

Modern Core Web Vitals optimization on standard HTML beats AMP on every metric that matters.

### 20.4 The Recommendation For Existing Bubbles Sites

If any Bubbles client site currently has AMP versions and `<link rel="amphtml">` references, plan migration off (Section 21).

### 20.5 The Niche Surviving AMP Use Cases

* **AMP Stories (Web Stories)**: short form vertical content for Google Discover. Niche.
* **Sites with limited development resources** where AMP's restricted template is operationally easier than building fast responsive HTML. Rare; not applicable to Bubbles (hand coded responsive HTML is the default).

For Bubbles convention: skip AMP entirely. The skill and discipline that produced ThatDeveloperGuy and the client portfolio already produces faster sites than AMP would.

---

## 21. THE AMP MIGRATION OFF PATTERN

If a Bubbles client site has AMP versions, migrate off using this pattern.

### 21.1 The Steps

1. **Remove `<link rel="amphtml">`** from all canonical pages. Google stops looking for AMP versions on next crawl.
2. **301 redirect AMP URLs to canonical URLs** at the nginx layer:
    ```nginx
    location ~ ^/amp/(.*)$ {
        return 301 https://example.com/$1;
    }
    ```
3. **Remove all AMP URLs from the XML sitemap**.
4. **Submit the updated sitemap to Google Search Console** to accelerate re crawl.
5. **Test Core Web Vitals on the canonical (non AMP) version** before fully decommissioning. Ensure the canonical performs well.
6. **Monitor traffic for 30 to 60 days** during the transition. Expect a small temporary fluctuation (industry reports indicate ~12% dip recovering within weeks).
7. **Delete AMP files from the server** once the migration is confirmed stable.

### 21.2 The Schema And Tag Cleanup

After removal:

* Remove any AMP specific JSON-LD from canonical pages.
* Remove any AMP specific `<meta>` tags.
* Update internal links that point to `/amp/` URLs (they'll 301 redirect, but cleaner is better).

### 21.3 The Verification

```bash
# Confirm AMP URLs return 301
curl -I https://example.com/amp/page/

# Confirm canonical pages no longer reference amphtml
curl -s https://example.com/page/ | grep amphtml
```

The first should show `301 Moved Permanently`. The second should return nothing.

### 21.4 The Bubbles Application

For Bubbles clients without AMP (every current client in the portfolio): nothing to do. The migration off pattern is documented for completeness and for any future inherited site that arrives with AMP.

---

## 22. THE BUBBLES PER CLIENT DECISION FRAMEWORK

Each client needs canonical and alternate decisions.

### 22.1 The Decision Questions

1. **What is the canonical host?** (Bare domain or www; document per client.)
2. **Is the trailing slash policy enforced at the nginx layer?** (Always yes for Bubbles.)
3. **Are there multiple language variants?** (Only Marshallese-Voices currently.)
4. **Does the client have a blog or editorial content?** (RSS + Atom + optional JSON Feed.)
5. **Are there any inherited AMP versions?** (Migrate off if yes.)
6. **Are there any non HTML resources needing canonical?** (PDFs for SDVOSB capability statements, F&I whitepapers, tax guides.)

### 22.2 The Standard Pattern (Every Client)

```html
<!-- Canonical: self referencing on every page -->
<link rel="canonical" href="https://[CLIENT].com/[path/]">
```

That's it. One line per page. No hreflang (unless multi language). No AMP. No m. variant.

### 22.3 The Per Client Recommendations

**ThatDeveloperGuy.com:**

```html
<link rel="canonical" href="https://thatdeveloperguy.com/[path/]">

<!-- On blog pages -->
<link rel="alternate" type="application/rss+xml" href="https://thatdeveloperguy.com/feed.xml" title="ThatDeveloperGuy Blog RSS">
<link rel="alternate" type="application/atom+xml" href="https://thatdeveloperguy.com/atom.xml" title="ThatDeveloperGuy Blog Atom">
<link rel="alternate" type="application/json" href="https://thatdeveloperguy.com/feed.json" title="ThatDeveloperGuy Blog JSON Feed">
```

Three feed formats for developer audience.

**ThatAIGuy.org:**

Same three feed stack. AI focused audience benefits from JSON Feed for programmatic consumption.

**Marshallese-Voices:**

Full hreflang cluster (Section 14) plus RSS and Atom for the blog. No JSON Feed (community audience, not developer focused).

**TCB Fight Factory:**

Canonical only. No blog feeds (event focused content).

**Arkansas Counseling and Wellness Services:**

Canonical only. Blog (if active) gets RSS + Atom.

**Handled Tax and Advisory:**

Canonical only. Blog (if active) gets RSS + Atom. PDF capability statements get HTTP Link canonical header.

**WeCoverUSA:**

Canonical only on marketing pages. Blog (if active) gets RSS + Atom. F&I whitepaper PDFs get HTTP Link canonical header.

**Heritage Hardwood Floors NWA:**

Canonical only. Blog (if active) gets RSS.

**Eureka Bath Works, White River Cabins, Greenough's Guide Service:**

Canonical only. Blog feeds added per content strategy.

**Local Living Real Estate, Diana Undergust, Homes on Grand:**

Canonical only. Listing feeds (if implemented in future) would use a custom format, not RSS.

**Avid Pest Control, RAH Construction NWA, Steele Solutions, Blue Paradise Dairy, Pretty Clean NWA:**

Canonical only.

### 22.4 The Documentation Pattern

For each client, document the canonical and alternate configuration:

```
Canonical and alternate configuration:
  Canonical host:       [bare domain or www]
  Trailing slash:       enforced (nginx 301)
  
  Hreflang cluster:     [none / English-Marshallese / English-Spanish / etc.]
  
  Feed alternates:
    RSS:    [yes/no] at /feed.xml
    Atom:   [yes/no] at /atom.xml
    JSON:   [yes/no] at /feed.json
  
  AMP:                  none (do not implement)
  m. variant:           none (responsive design)
```

---

## 23. THE SERVER SIDE RENDERING REQUIREMENT

Canonical and alternate signals must be in the initial server side HTML response. Crawlers do not execute JavaScript.

### 23.1 The Crawler Behavior

Google's crawler executes JavaScript in some contexts (rendered HTML phase) but the canonical signal is consumed during the initial crawl, before JavaScript executes.

If `<link rel="canonical">` is injected by React, Vue, Svelte, or any client side framework, the canonical may be missed on the first crawl and only picked up on the rendered HTML phase (which happens later and inconsistently).

### 23.2 The Risk

Client side injected canonical can cause:

* Initial crawl indexes the wrong URL.
* Inconsistent canonical assignment in Search Console.
* Hreflang clusters that "look right" in the rendered HTML but fail in Google's processing because the crawl pipeline reads pre rendered HTML.

### 23.3 The Verification

```bash
curl -s https://example.com/page/ | grep -E '<link rel="(canonical|alternate)"'
```

If output includes the canonical and alternate links, server side rendered. If empty, client side injected; fix before launch.

### 23.4 The Stack Patterns

* **Static HTML (Bubbles default)**: trivially server side. No issue.
* **Next.js App Router**: use the `metadata` API or `<Head>` from `next/head` in server components. Avoid client side `useEffect` Head manipulation.
* **SvelteKit**: `<svelte:head>` in page level Svelte renders server side.
* **Astro**: `<head>` content in `.astro` files renders server side.
* **Headless Shopify (Blue Paradise Dairy)**: requires Node SSR layer; pure client side React (Vite SPA) would fail.

---

## 24. IMPLEMENTATION PATTERNS BY STACK

### 24.1 Static HTML (Bubbles Default)

Direct in the `<head>`:

```html
<head>
    <meta charset="utf-8">
    <title>Page Title</title>

    <!-- Canonical -->
    <link rel="canonical" href="https://example.com/page/">

    <!-- Hreflang (Marshallese-Voices only) -->
    <link rel="alternate" hreflang="en-US" href="https://example.com/page/">
    <link rel="alternate" hreflang="mh" href="https://example.com/mh/page/">
    <link rel="alternate" hreflang="x-default" href="https://example.com/page/">

    <!-- Feed alternates (blog enabled clients) -->
    <link rel="alternate" type="application/rss+xml" href="/feed.xml" title="Blog RSS">
    <link rel="alternate" type="application/atom+xml" href="/atom.xml" title="Blog Atom">
    <link rel="alternate" type="application/json" href="/feed.json" title="Blog JSON Feed">
</head>
```

### 24.2 Next.js 14 App Router

```typescript
// app/page.tsx
export const metadata = {
  alternates: {
    canonical: 'https://example.com/',
    languages: {
      'en-US': 'https://example.com/',
      'mh': 'https://example.com/mh/',
      'x-default': 'https://example.com/',
    },
    types: {
      'application/rss+xml': 'https://example.com/feed.xml',
      'application/atom+xml': 'https://example.com/atom.xml',
      'application/json': 'https://example.com/feed.json',
    },
  },
};
```

Next.js 14 renders these into proper `<link>` elements server side.

### 24.3 SvelteKit

```svelte
<!-- src/routes/+page.svelte -->
<svelte:head>
    <link rel="canonical" href="https://example.com/" />
    <link rel="alternate" hreflang="en-US" href="https://example.com/" />
    <link rel="alternate" hreflang="mh" href="https://example.com/mh/" />
    <link rel="alternate" hreflang="x-default" href="https://example.com/" />
    <link rel="alternate" type="application/rss+xml" href="/feed.xml" title="Blog RSS" />
</svelte:head>
```

### 24.4 Astro

```astro
---
const canonical = `https://example.com${Astro.url.pathname}`;
---
<head>
    <link rel="canonical" href={canonical} />
    <link rel="alternate" type="application/rss+xml" href="/feed.xml" title="Blog RSS" />
</head>
```

### 24.5 The nginx HTTP Link Header Pattern (For PDFs)

```nginx
location ~ \.pdf$ {
    add_header Link "<https://example.com$uri>; rel=\"canonical\"";
}
```

Reload nginx:

```bash
nginx -t && systemctl reload nginx
```

### 24.6 The Feed Generation Build Step

For Bubbles static HTML clients with blogs, generate the feed files at build time. A bash example:

```bash
# Generate /feed.xml, /atom.xml, /feed.json from /blog/*.html files
# (build script details depend on the client setup)
```

Place the generated feed files at the web root.

---

## 25. THE PRE LAUNCH AUDIT CHECKLIST

Before a Bubbles client site goes live.

### 25.1 The Canonical Checklist

* [ ] `<link rel="canonical">` present in `<head>` on every page.
* [ ] Canonical URL is absolute and HTTPS.
* [ ] Canonical URL matches the trailing slash policy.
* [ ] Canonical URL matches `og:url` (per framework-html-open-graph.md).
* [ ] Canonical URL matches XML sitemap `<loc>`.
* [ ] Canonical URL returns HTTP 200 (not 3xx, not 4xx, not noindex).
* [ ] No canonical loops (page A canonicalizes to B which canonicalizes to A).
* [ ] No canonical to staging URLs (verify the final canonical points to production).
* [ ] Self referencing canonical on every regular page.
* [ ] Cross domain canonical only where intentional.

### 25.2 The Hreflang Checklist (Multi Language Clients Only)

* [ ] Reciprocity rule: every variant references every other variant.
* [ ] Self reference rule: each variant references itself.
* [ ] Canonical alignment rule: each variant has a self referencing canonical.
* [ ] x-default declared.
* [ ] All hreflang URLs return HTTP 200 (no 404, no redirects).
* [ ] All hreflang URLs are indexable (no noindex).
* [ ] BCP 47 language codes are valid (use `en-GB` not `en-UK`).
* [ ] Hyphen format used (`en-US` not `en_US`).
* [ ] One placement method used consistently (HTML head or sitemap, not mixed).

### 25.3 The Feed Alternate Checklist (Blog Enabled Clients Only)

* [ ] RSS feed at `/feed.xml` exists and validates.
* [ ] Atom feed at `/atom.xml` exists and validates.
* [ ] JSON Feed at `/feed.json` (if implemented) exists and validates.
* [ ] Feed MIME types served correctly by nginx.
* [ ] `<link rel="alternate" type="...">` references each feed on every page (or at minimum the homepage and blog index).
* [ ] Feed `title` attribute is descriptive.

### 25.4 The AMP Checklist

* [ ] No `<link rel="amphtml">` references anywhere on the site.
* [ ] No `/amp/` URLs returning HTTP 200 (any legacy AMP URLs should 301 redirect).
* [ ] No AMP entries in the XML sitemap.

### 25.5 The curl Verification

```bash
# Canonical
curl -s https://example.com/page/ | grep -i 'rel="canonical"'

# All alternates
curl -s https://example.com/page/ | grep -E '<link rel="alternate"'

# AMP (should return nothing)
curl -s https://example.com/page/ | grep amphtml
```

---

## 26. THE COMMON FAILURES AND FIXES

### 26.1 "Google Search Console reports duplicate without user selected canonical"

Cause 1: `<link rel="canonical">` is missing.
Fix: Add self referencing canonical to every page.

Cause 2: Canonical is JS injected; Google's first crawl missed it.
Fix: Move to server side rendered HTML.

Cause 3: Multiple competing URLs (trailing slash variant, parameter variants) exist.
Fix: Enforce canonical with 301 redirects at nginx layer plus declare canonical.

### 26.2 "Google is indexing the wrong URL variant"

Cause: Canonical points to one URL but internal links and sitemap point elsewhere.
Fix: Align all signals (canonical, sitemap, og:url, internal links) to a single URL.

### 26.3 "Hreflang return tag error in Search Console"

Cause: Reciprocity broken. Page A references B but B does not reference A.
Fix: Add the return reference on B. Audit every page in the cluster.

### 26.4 "Hreflang is being ignored entirely"

Cause: Canonical alignment broken. A French page canonicalizes to the English version.
Fix: Each variant must self canonicalize.

### 26.5 "Marshallese page is appearing in English search results"

Cause: hreflang cluster not validating; Google falls back to merging the two variants.
Fix: Run through the three rules (reciprocity, self reference, canonical alignment); fix the broken one.

### 26.6 "RSS feed is not appearing in Feedly when users paste the URL"

Cause 1: `<link rel="alternate" type="application/rss+xml">` is missing.
Fix: Add the link element in the `<head>`.

Cause 2: Feed file returns wrong MIME type.
Fix: Configure nginx to serve `/feed.xml` with `application/rss+xml`.

Cause 3: Feed XML has syntax errors.
Fix: Validate with `xmllint --noout /feed.xml`.

### 26.7 "AMP versions are still appearing in Google search after migration off"

Cause: Google has not yet re crawled. AMP URLs in the cache still match.
Fix: Wait for re crawl; ensure 301 redirects are in place to accelerate.

### 26.8 "Canonical points to a 301 redirect"

Cause: Canonical URL itself is a redirect target.
Fix: Update canonical to point to the final 200 URL.

### 26.9 "Canonical and og:url do not match"

Cause: Inconsistent URL declaration across signals.
Fix: Align canonical, og:url, twitter:url (if set), and XML sitemap entry.

### 26.10 "Page returns 404 but is listed in hreflang cluster"

Cause: A linked URL no longer exists.
Fix: Update or remove the hreflang reference. Maintain cluster integrity.

---

## 27. THE FEDERAL AND SDVOSB CONTEXT

For Joseph's SDVOSB federal subcontracting work, canonical discipline matters operationally.

### 27.1 The Procurement Officer Link Sharing Pattern

Federal procurement officers and contracting officers share vendor URLs via email, agency messaging systems, and LinkedIn. The shared URL becomes the canonical reference in agency documentation.

If the same content is reachable via multiple URLs (www and non www, http and https, trailing slash variants), agency systems may bookmark inconsistent URLs. The canonical signal ensures the agency settles on one URL as the authoritative reference.

### 27.2 The PDF Capability Statement Canonical

For SDVOSB capability statements distributed as PDFs:

```nginx
location = /capability-statement.pdf {
    add_header Link "<https://thatdeveloperguy.com/capability-statement.pdf>; rel=\"canonical\"";
}
```

The HTTP Link header ensures Google indexes the PDF at the canonical URL even if it's distributed via direct email or document sharing.

### 27.3 The SAM.gov URL Consistency

The URL registered on SAM.gov should match the canonical URL declared on the vendor site. Procurement officers cross reference; inconsistency raises diligence concerns.

For ThatDeveloperGuy.com SDVOSB registration: ensure the SAM.gov URL and the site canonical agree exactly (https, no www if bare is canonical, trailing slash if present).

### 27.4 The WeCoverUSA Federal Context

Same pattern. The canonical URL declared on the WeCoverUSA site must match the URL used in:

* SAM.gov vendor profile.
* F&I dealer outreach materials.
* Capability statement PDFs.
* Email signatures used by Richard and the team.

Consistency signals operational maturity.

---

## 28. QUICK REFERENCE TAG BLOCKS

For copy paste into any Bubbles client page `<head>`:

### 28.1 Standard Pattern (Every Bubbles Client)

```html
<!-- Canonical -->
<link rel="canonical" href="https://[CLIENT].com/[PATH/]">
```

One line. Self referencing. Trailing slash. HTTPS.

### 28.2 Multi Language Pattern (Marshallese-Voices Only)

```html
<!-- On English page -->
<link rel="canonical" href="https://marshallese-voices.org/[path/]">
<link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/[path/]">
<link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/[path/]">
<link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/[path/]">

<!-- On Marshallese page -->
<link rel="canonical" href="https://marshallese-voices.org/mh/[path/]">
<link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/[path/]">
<link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/[path/]">
<link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/[path/]">
```

### 28.3 Blog Enabled Pattern (Clients With Blogs)

```html
<!-- RSS, Atom, JSON Feed -->
<link rel="alternate" type="application/rss+xml" href="/feed.xml" title="[CLIENT] Blog RSS">
<link rel="alternate" type="application/atom+xml" href="/atom.xml" title="[CLIENT] Blog Atom">
<link rel="alternate" type="application/json" href="/feed.json" title="[CLIENT] Blog JSON Feed">
```

For developer audience clients (ThatDeveloperGuy, ThatAIGuy): all three.

For general audience clients with blogs: RSS and Atom; skip JSON.

### 28.4 AMP Pattern (Do Not Implement)

```html
<!-- DO NOT INCLUDE rel="amphtml" on new Bubbles builds -->
<!-- AMP is mostly deprecated in 2026 -->
```

### 28.5 The PDF Canonical Header (nginx)

```nginx
location ~ \.pdf$ {
    add_header Link "<https://example.com$uri>; rel=\"canonical\"";
}
```

End every nginx change with:

```bash
nginx -t && systemctl reload nginx
```

---

## 29. REFERENCES

* Google Search Central canonical URLs: https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls
* Google Search Central hreflang: https://developers.google.com/search/docs/specialty/international/localized-versions
* RSS 2.0 specification: https://www.rssboard.org/rss-specification
* Atom 1.0 specification (RFC 4287): https://datatracker.ietf.org/doc/html/rfc4287
* JSON Feed 1.1: https://www.jsonfeed.org/version/1.1/
* IETF BCP 47 language tags: https://datatracker.ietf.org/doc/html/rfc5646
* Google AMP deprecation status (Page Experience update, May 2021): https://developers.google.com/search/blog/2021/04/more-details-page-experience

**Companion frameworks in the HTML signal track:**

* framework-html-meta-robots.md
* framework-html-meta-charset.md
* framework-html-meta-viewport.md
* framework-html-meta-description.md
* framework-html-meta-keywords.md
* framework-html-meta-author.md
* framework-html-meta-generator.md
* framework-html-meta-copyright.md
* framework-html-meta-theme-color.md
* framework-html-meta-color-scheme.md
* framework-html-meta-referrer.md
* framework-html-content-language.md (direct companion for hreflang)
* framework-html-meta-refresh.md
* framework-html-open-graph.md (direct companion for canonical/og:url alignment)
* framework-html-twitter-cards.md
* framework-html-search-engine-verification.md
* framework-html-app-pwa-meta.md

**Companion frameworks in adjacent tracks:**

* framework-http-3xx-status-codes.md (the 301 redirect mechanism that interacts with canonical)
* framework-http-dns-records.md (the DNS layer)
* framework-html-xml-sitemap.md (the sitemap layer that mirrors canonical)
* framework-html-schema-jsonld.md (Schema.org structured data)

**Parent reference:** UNIVERSAL-RANKING-FRAMEWORK.md (the 27 section master document).

---

*End of framework-html-canonical-alternate.md.*
