# framework-http-seo-headers.md

Comprehensive reference for the three HTTP response headers that control indexing, resource discovery, and URL forwarding: `X-Robots-Tag` for indexing directives on any file type, `Link` for canonicals/hreflang/resource hints expressed at the HTTP layer, and `Location` for redirects. Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to framework-http-caching-headers.md, framework-http-content-headers.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx, AI assistants generating or repairing nginx config, and anyone troubleshooting "PDF appearing in Google when it should not", "preload not picked up", "redirect chain killing crawl budget", or "wrong page is canonical" anomalies on a self hosted stack.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Indexing Authority Mental Model (read this first)
5. X-Robots-Tag (every meta robots directive at the HTTP layer)
6. Link (canonical, alternate, hreflang, preload, prefetch at HTTP)
7. Location (redirect target with status code semantics)
8. How These Headers Interact
9. Asset Class And Use Case Recipes
10. Bubbles Nginx Reference Block (paste ready)
11. Audit Checklist (50+ items)
12. Common Pitfalls
13. Diagnostic Commands (curl, GSC URL Inspection, browser devtools)
14. Cross-References

---

## 1. DEFINITION

SEO and indexing headers tell search engine crawlers what they may or may not do with a resource, where its canonical version lives, what related resources exist, and where a moved URL can now be found. They operate at the HTTP layer, which means they apply to every response regardless of body content type. The three headers split into three concerns:

* **Indexing control**: `X-Robots-Tag`. It answers "may you index this, follow its links, show a snippet, cache it, translate it?"
* **Resource relationships**: `Link`. It answers "what is the canonical URL, what alternates exist, what resources should the browser preload, what is related to this resource?"
* **Forwarding**: `Location`. It answers "this URL has moved; here is where to go next."

Together these three headers shape every aspect of how search engines and AI crawlers understand a site's structure, what they index, and how PageRank or its modern equivalents flow between URLs. Getting any one of them wrong has direct ranking consequences: deindexed pages, lost canonical signals, broken hreflang clusters, redirect chains that bleed crawl budget, or worse, redirects that point to attacker controlled domains.

---

## 2. WHY IT MATTERS

Five independent pressures push correct SEO headers from "nice to have" to "required infrastructure" in 2025 and forward.

**Indexing is selective by default.** Googlebot's crawl budget is finite. ClaudeBot, GPTBot, PerplexityBot, OAI-SearchBot, and other AI crawlers ration their requests even more aggressively. Sites that signal `noindex` or `nofollow` for low value URLs (filtered listings, search result pages, faceted navigation, internal admin) reclaim that budget for pages that actually matter. Sites that fail to signal end up with thousands of useless URLs in the index diluting their topical authority.

**Non HTML files cannot use meta tags.** PDFs, images, videos, CSV exports, ZIP archives, and any other binary file cannot contain `<meta name="robots">`. The only way to control their indexing is `X-Robots-Tag`. A site that publishes PDFs without the header is silently allowing every PDF to be indexed, which causes brand pages and outdated material to outrank current HTML pages for the same query.

**Resource hints are now a Core Web Vitals lever.** Modern browsers pre fetch resources declared in HTTP `Link` headers before parsing the HTML body. This shaves 50 to 300 ms off LCP for the largest contentful paint candidate (typically the hero image or above the fold font). Google's recommended pattern for hero image LCP optimization explicitly uses `Link: <hero.webp>; rel=preload; as=image; fetchpriority=high` as an HTTP header, not just the HTML link element.

**103 Early Hints exists and matters.** Nginx 1.25+ supports HTTP 103 Early Hints, which lets the server send `Link` preload headers before the final response is ready. Chrome and Edge consume Early Hints and begin fetching critical resources during the time the server is still computing the page. On dynamic sites this can shave 100 to 500 ms off LCP at zero application cost.

**Redirect chains are crawl killers and ranking killers.** Each hop in a redirect chain adds latency, consumes one unit of crawl budget, and risks losing equity along the way. Google has stated that all redirect codes (301, 302, 307, 308) are treated equivalently for ranking purposes, but the practical reality is that chains over three hops trigger soft 404 handling and lost consolidation. A single direct redirect is always better than a chain.

**Open redirects are an active attack surface.** A redirect endpoint that accepts an unvalidated target parameter (`/redirect?url=...`) lets attackers craft phishing URLs that appear to come from your domain. Modern phishing kits scan for these endpoints. Every redirect endpoint that accepts user input must validate the target against an allow list.

**Cost of getting it wrong.** Misconfigured SEO headers produce silent ranking failures and loud security failures. Examples from production sites:

* A staging subdomain was indexed because nobody set `X-Robots-Tag: noindex`. Six months later Google merged duplicate content signals from staging and production, halving organic traffic for shared terms.
* A site's brand PDFs from 2018 ranked above the 2024 HTML refresh because the old PDFs were indexed and Google had no signal that they were superseded.
* A new product launch shipped with `Link: <previous-product>; rel=canonical` accidentally pointing at the old product page, blocking the new launch from indexing for three weeks.
* A 301 chain (`/old` → `/new` → `/newer` → `/current`) bled crawl budget for years until someone audited and collapsed it to a single hop.
* An open redirect on `/r?url=...` was used in a phishing campaign against the client's email list, prompting cease and desist orders and emergency cleanup.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the three headers gets the same six part treatment:

1. **What it does**: the canonical RFC and Google specification plus the practical implication.
2. **Syntax and directives**: every legal value, what it means, and when it is wrong.
3. **How to build it on Bubbles**: paste ready nginx config.
4. **How to verify it**: curl commands plus Google Search Console URL Inspection where applicable.
5. **How to troubleshoot**: the four or five failure modes seen in the field and how to recognize each.
6. **How to fix common breakage**: ordered repair steps.

Recipes for the most common asset classes and use cases are collected in Section 9.

---

## 4. THE INDEXING AUTHORITY MENTAL MODEL (READ THIS FIRST)

Every search engine and AI crawler that fetches a URL runs the same dispatch loop. Internalize it and every header choice becomes obvious.

```
Crawler requests URL
        |
        v
Server responds with status code
        |
        v
Status 200, 304? --> proceed to indexing logic
Status 301, 308? --> follow Location, consolidate signals to target
Status 302, 307? --> follow Location, do NOT consolidate (temporary)
Status 4xx?      --> mark URL as gone or soft 404
Status 5xx?      --> defer, retry later, may eventually drop from index
        |
        v
Read X-Robots-Tag header
        |
        v
Directives present?
   |                                   |
  YES                                 NO
   |                                   |
   v                                   v
Apply directives:               Default behavior:
- noindex: do not add to index   - index this URL
- nofollow: do not follow links  - follow all links on it
- nosnippet: no preview text     - show snippet in SERP
- noarchive: no cached copy      - allow cached copy
- max-snippet:N: cap snippet     - default snippet length
        |
        v
Read Link header
        |
        v
- rel="canonical": treat target as primary, consolidate signals
- rel="alternate" hreflang: register language variants
- rel="alternate" media: register mobile/print variants
- rel="prev" / rel="next": pagination (deprecated for SEO)
- rel="preload" / "prefetch": browser hints, ignored by crawlers
        |
        v
Read HTML body (if Content-Type is text/html and not noindex)
        |
        v
Parse <head> for:
- <meta name="robots"> (may override or supplement X-Robots-Tag)
- <link rel="canonical"> (may override or supplement Link header)
- <link rel="alternate" hreflang="...">
- <link rel="amphtml">
        |
        v
Reconcile signals from HTTP and HTML
        |
        v
Update index entry
```

Three rules govern the system:

1. **HTTP signals reach every file type.** Meta tags only work for HTML. PDFs, images, JSON, CSV all rely exclusively on HTTP headers. If you publish any non HTML file you intend to control, you must use the HTTP layer.
2. **The most restrictive directive wins.** Combining `noindex` and `index` (across HTTP and HTML, or across multiple X-Robots-Tag entries) resolves to `noindex`. Combining `nosnippet` and `max-snippet:200` resolves to `nosnippet`. There is no "negotiation" mechanism; restrictive always beats permissive.
3. **Redirects consolidate or do not consolidate based on status code semantics.** Permanent codes (301, 308) tell crawlers to merge signals into the target URL. Temporary codes (302, 307) tell crawlers to keep the original URL indexed and treat the redirect as situational. Google has said the difference is "smaller than people think" for ranking, but the difference is real and matters for index hygiene.

A correct header set produces consistent behavior across Googlebot, Bingbot, ClaudeBot, GPTBot, PerplexityBot, Applebot, OAI-SearchBot, and every other crawler. The same response, the same outcome everywhere.

---

## 5. X-ROBOTS-TAG (EVERY META ROBOTS DIRECTIVE AT THE HTTP LAYER)

### 5.1 What It Does

`X-Robots-Tag` is the HTTP response header that conveys indexing directives to crawlers. Functionally equivalent to `<meta name="robots" content="...">` but applies to any response type, not just HTML. Documented by Google at developers.google.com/search/docs/crawling-indexing/robots-meta-tag.

```
X-Robots-Tag: noindex
X-Robots-Tag: noindex, nofollow
X-Robots-Tag: googlebot: nosnippet, notranslate
X-Robots-Tag: max-snippet:160, max-image-preview:large
X-Robots-Tag: unavailable_after: 31 Dec 2026 23:59:59 GMT
```

This is the only way to control indexing for non HTML resources: PDFs, images (when accessed directly by URL), videos, JSON endpoints, CSV exports, ZIP downloads, any binary file. For HTML pages, both `X-Robots-Tag` and `<meta name="robots">` work and can be combined.

### 5.2 Every Directive Available

The full set of directives Google supports as of 2026. Other major crawlers (Bing, Yandex, Baidu, ClaudeBot, GPTBot) support most of these.

**Indexing directives:**

| Directive | Meaning |
|---|---|
| `all` | Default. Equivalent to "no restrictions". Same as omitting the header |
| `noindex` | Do not include this URL in the search index |
| `nofollow` | Do not follow any links on this page (does not pass equity) |
| `none` | Shorthand for `noindex, nofollow` |
| `noarchive` | Do not show a "Cached" link in search results |
| `nosnippet` | Do not show a text or video snippet in search results. Also blocks use of the content in AI Overviews and similar features |
| `noimageindex` | Do not index any images on this page or use them as canonical sources |
| `notranslate` | Do not offer translation of this page in search results |
| `indexifembedded` | Allow indexing of this content when embedded via iframe in another page, even when `noindex` also applies. Useful for media publishers who want syndication |
| `unavailable_after: <date>` | After the specified date and time, treat this URL as `noindex`. Date format: `Day, DD Mon YYYY HH:MM:SS GMT` |

**Snippet control directives:**

| Directive | Meaning |
|---|---|
| `max-snippet: <number>` | Maximum length in characters for text snippets. Special values: `0` (no snippet, same as `nosnippet`), `-1` (no limit) |
| `max-image-preview: <setting>` | Largest image preview allowed in snippets. Values: `none`, `standard`, `large` |
| `max-video-preview: <number>` | Maximum number of seconds for video previews. Special values: `0` (static image only), `-1` (no limit) |

**Removed or deprecated:**

| Directive | Status |
|---|---|
| `noodp` | Removed. Open Directory Project is defunct |
| `noydir` | Removed. Yahoo Directory is defunct |
| `noyaca` | Yandex specific. Yandex Catalog is defunct |

### 5.3 Bot Specific Syntax

By default, directives apply to every crawler. To target a specific bot, prefix the directives with the bot's user agent token followed by a colon. Multiple `X-Robots-Tag` headers can stack to specify different rules for different bots.

```
X-Robots-Tag: googlebot: nosnippet, notranslate
X-Robots-Tag: bingbot: noarchive
X-Robots-Tag: noindex
```

In this example:

* Googlebot: gets `nosnippet, notranslate` AND inherits the global `noindex` (because the unprefixed line applies to it too).
* Bingbot: gets `noarchive` AND the global `noindex`.
* All other crawlers: get only `noindex`.

**Important nuance about Google specifically:** Google only honors bot prefix syntax for `googlebot` and `googlebot-news`. Other Google crawlers (Googlebot-Image, Googlebot-Video, Storebot-Google, AdsBot-Google) do not respond to bot prefixed `X-Robots-Tag` directives. They are controlled via robots.txt user agent blocks instead.

Common bot tokens worth knowing:

| Bot | Token |
|---|---|
| Googlebot (web) | `googlebot` |
| Googlebot News | `googlebot-news` |
| Bingbot | `bingbot` |
| Applebot | `applebot` |
| ClaudeBot | `claudebot` |
| GPTBot | `gptbot` |
| PerplexityBot | `perplexitybot` |
| OAI-SearchBot | `oai-searchbot` |
| YandexBot | `yandex` |
| Baiduspider | `baiduspider` |

### 5.4 How To Build It On Bubbles

**Block all indexing for a staging or demo site:**

```nginx
server {
    server_name staging.example.com;

    # Block every page from every crawler
    add_header X-Robots-Tag "noindex, nofollow" always;

    # Also block via robots.txt for belt and suspenders
    location = /robots.txt {
        return 200 "User-agent: *\nDisallow: /\n";
    }
}
```

**Block indexing for non HTML files only:**

```nginx
location ~* \.(pdf|doc|docx|xls|xlsx|csv|zip|tar|gz)$ {
    add_header X-Robots-Tag "noindex" always;
    expires 1d;
    add_header Cache-Control "public, max-age=86400" always;
}
```

This is the primary use case for `X-Robots-Tag`: file types where you cannot use meta tags.

**Block image indexing while allowing the HTML page to index:**

```nginx
# Apply at the image location only, not the parent HTML
location ~* \.(jpg|jpeg|png|gif|webp|avif|svg)$ {
    add_header X-Robots-Tag "noindex" always;
}
```

The HTML page itself remains indexable; only direct image URLs are blocked from being indexed as standalone results.

**Set snippet limits site wide:**

```nginx
server {
    add_header X-Robots-Tag "max-snippet:200, max-image-preview:large, max-video-preview:-1" always;
    # Other directives...
}
```

This is the recommended baseline for most sites. It tells Google: allow up to 200 character text snippets, allow large image previews, no limit on video preview length.

**Different rules for different bots:**

```nginx
# In a server block, stack multiple add_header directives
add_header X-Robots-Tag "googlebot: max-snippet:200, max-image-preview:large" always;
add_header X-Robots-Tag "bingbot: max-snippet:300" always;
add_header X-Robots-Tag "noai" always;
```

The order does not matter for parsing. Each header is independent.

**Time limited content (unavailable_after):**

```nginx
location = /promotion/black-friday-2026.html {
    add_header X-Robots-Tag "unavailable_after: 1 Dec 2026 00:00:00 GMT" always;
    expires 1d;
    add_header Cache-Control "public, max-age=86400" always;
}
```

After December 1, 2026, Google will drop this URL from search results without requiring a manual removal request. The page itself can remain accessible.

**The add_header inheritance trap applies here too.** A nested location's `add_header` wipes parent declarations. Use the snippet include pattern:

```nginx
# /etc/nginx/snippets/robots-baseline.conf
add_header X-Robots-Tag "max-snippet:200, max-image-preview:large, max-video-preview:-1" always;

# In every location that needs it
location /blog/ {
    include snippets/robots-baseline.conf;
    add_header Cache-Control "public, max-age=300" always;
}
```

### 5.5 The robots.txt vs X-Robots-Tag Distinction

These two mechanisms control different things. Confusing them is the most common SEO mistake on the planet.

| Mechanism | Controls |
|---|---|
| `robots.txt` | Which URLs the crawler is allowed to fetch |
| `X-Robots-Tag` (or `<meta robots>`) | Which fetched URLs the crawler is allowed to index |

A URL blocked by robots.txt is never fetched, which means the crawler never sees the `X-Robots-Tag` header. If the URL has external links pointing to it, Google may index it anyway as a "URL only" entry (no title, no snippet, just the URL) based on those external signals.

To actually deindex a URL:

1. **Remove the robots.txt block** so the crawler can fetch the URL.
2. **Add `X-Robots-Tag: noindex`** to the response.
3. **Wait for the crawler to re fetch and process the directive.**
4. **Then optionally add the robots.txt block back** to prevent further crawling, but only after Google has processed the noindex.

Get this order wrong and the URL stays indexed forever. Get it right and the URL drops from the index within a crawl cycle (days to weeks).

### 5.6 How To Verify

```bash
# 1. Confirm header is present
curl -sI https://example.com/some-page | grep -i x-robots-tag

# 2. Check for a specific file type
curl -sI https://example.com/downloads/report.pdf | grep -i x-robots-tag

# 3. Check what Googlebot sees
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/page.html | grep -i x-robots-tag

# 4. Test bot specific directives
curl -sI -A "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)" \
     https://example.com/page.html | grep -i x-robots-tag

# 5. Compare HTTP header to HTML meta tag
echo "=== HTTP X-Robots-Tag ==="
curl -sI https://example.com/page.html | grep -i x-robots-tag
echo "=== HTML meta robots ==="
curl -s https://example.com/page.html | grep -i 'name="robots"'

# 6. Verify a site wide block is in effect
for path in / /about /blog /contact /robots.txt; do
    echo "=== https://staging.example.com$path ==="
    curl -sI "https://staging.example.com$path" | grep -i x-robots-tag
done
```

**Use Google Search Console URL Inspection** for the authoritative check on whether Google sees and respects the directive:

1. Open Google Search Console for the property.
2. Paste the URL into the top search bar.
3. Click "Test live URL".
4. Under "Page indexing", look for "Indexing allowed?" and the source of any "noindex" detection.

The Live Test fetches the URL with Googlebot's actual user agent and shows the headers it saw. This is the ground truth for SEO purposes.

### 5.7 Troubleshooting

**Symptom: Page is still in Google index despite `X-Robots-Tag: noindex`.**
Causes ranked by frequency:
1. The URL is blocked by robots.txt, so Googlebot cannot fetch it and never sees the header. Remove the robots.txt block, wait, then optionally re add.
2. The page has not been re crawled since the directive was added. Wait or request indexing via Search Console URL Inspection.
3. The directive has a typo (e.g. `no-index` instead of `noindex`). Verify with curl.
4. A CDN is stripping the header. Not applicable on Bubbles. Verify with `curl --resolve` directly against the origin.
5. The page has strong external links and Google retains a URL only entry. This is rare and resolves over time.

**Symptom: PDF is being indexed and outranking the corresponding HTML page.**
Cause: No `X-Robots-Tag` on the PDF location.
Fix: see Section 5.4. Add `X-Robots-Tag: noindex` to a location matching `\.pdf$`. If you want the PDF indexable but not outranking the HTML, add `<link rel=canonical>` to the HTML and `X-Robots-Tag: noindex, nofollow` to the PDF.

**Symptom: Two `X-Robots-Tag` headers appear and seem to conflict.**
Multiple directives across multiple headers are interpreted as a union. The most restrictive wins. `X-Robots-Tag: index` plus `X-Robots-Tag: noindex` resolves to `noindex`. There is no conflict; this is the spec.

**Symptom: `unavailable_after` date passed but URL is still indexed.**
Causes:
1. Date format is wrong. Must be RFC 850 or RFC 822 format. The Google docs recommend `Day, DD Mon YYYY HH:MM:SS GMT`. Verify with curl.
2. Google has not re crawled since the date passed. Crawl frequency depends on the URL's profile.
3. `unavailable_after` is intentionally not retroactive in a strong sense; it influences future crawls.

**Symptom: AI engines (ChatGPT, Claude, Perplexity) are still using content despite `nosnippet`.**
The `nosnippet` directive applies to Google search results, including AI Overviews. AI engines that crawl independently (ChatGPT search, Claude search, Perplexity) honor their own published opt out signals which often differ:
* OpenAI: respect `User-agent: GPTBot` in robots.txt
* Anthropic: respect `User-agent: ClaudeBot` in robots.txt
* Perplexity: respect `User-agent: PerplexityBot` in robots.txt
* Common Crawl: respect `User-agent: CCBot` in robots.txt
Add explicit robots.txt blocks for each AI agent you want to opt out of, in addition to any `X-Robots-Tag` directives.

**Symptom: Bot prefixed directive ignored.**
Google only honors `googlebot:` and `googlebot-news:` prefixes. Other Google crawlers (Googlebot-Image, Googlebot-Video) are not controllable via this mechanism. Use robots.txt for those.

### 5.8 How To Fix Common Breakage

**Case: Staging site indexed in Google.**
Add to the server block immediately:

```nginx
server {
    server_name staging.example.com;
    add_header X-Robots-Tag "noindex, nofollow" always;
    # ... other config ...
}
```

`nginx -t && systemctl reload nginx`. Then in Search Console, request removal for the entire staging subdomain. The combination of the noindex header (preventing re indexation) and the removal request (immediate effect) resolves the issue within hours.

**Case: All PDFs are appearing in search and bleeding click traffic from HTML pages.**
Block all PDFs from indexing:

```nginx
location ~* \.pdf$ {
    add_header X-Robots-Tag "noindex" always;
    add_header Cache-Control "public, max-age=86400" always;
}
```

The PDFs remain downloadable; only their index entries disappear. Existing index entries drop on next crawl.

**Case: A campaign page needs to be deindexed after the campaign ends.**
Add `unavailable_after` with the campaign end date:

```nginx
location = /campaigns/spring-2026.html {
    add_header X-Robots-Tag "unavailable_after: 30 Jun 2026 23:59:59 GMT" always;
}
```

No manual intervention required after June 30.

**Case: Need to allow ClaudeBot but block GPTBot.**
Use robots.txt (not X-Robots-Tag, since both ClaudeBot and GPTBot have limited bot prefix support):

```
User-agent: GPTBot
Disallow: /

User-agent: ClaudeBot
Allow: /

User-agent: *
Allow: /
```

Serve via:

```nginx
location = /robots.txt {
    expires 1h;
    add_header Cache-Control "public, max-age=3600" always;
}
```

---

## 6. LINK (CANONICAL, ALTERNATE, HREFLANG, PRELOAD, PREFETCH AT HTTP)

### 6.1 What It Does

The `Link` header expresses relationships between the current response and other resources. Defined in RFC 8288. The same semantic content as the HTML `<link>` element, but in HTTP form. Critical for non HTML responses (PDFs cannot contain `<link>` tags but can have HTTP `Link` headers) and for performance optimization (HTTP `Link` is processed before HTML parsing begins).

```
Link: <https://example.com/page>; rel="canonical"
Link: <https://example.com/es/page>; rel="alternate"; hreflang="es-MX"
Link: </css/main.css>; rel="preload"; as="style"
Link: </hero.webp>; rel="preload"; as="image"; fetchpriority="high"
Link: <https://cdn.example.com>; rel="preconnect"
```

Multiple relationships can be comma separated in a single header or split across multiple `Link` headers. Both forms are equivalent.

### 6.2 Syntax

The general form:

```
Link: <URI>; param1=value1; param2="value 2"
Link: <URI1>; rel="..."; param="..." , <URI2>; rel="..."
```

Rules:

* URI is wrapped in angle brackets: `<https://example.com/x>`.
* Parameters are separated from the URI by a semicolon.
* Parameters are separated from each other by semicolons.
* Multiple relationships are separated by commas (use one comma per Link header line; do not nest).
* Quote values with spaces or special characters using double quotes.
* `rel` is required for almost every meaningful Link header.

### 6.3 Relationship Types

The values for `rel` come from the IANA Link Relations registry. The ones that matter on the web:

**SEO and canonicalization:**

| rel | Purpose |
|---|---|
| `canonical` | Declare the canonical URL for this resource. Critical for SEO. The target URL is treated as the authoritative version |
| `alternate` (with `hreflang`) | Declare a language or region variant of the current resource. Used for international SEO |
| `alternate` (with `type="application/rss+xml"`) | Declare an RSS or Atom feed for this page |
| `alternate` (with `media`) | Declare a print stylesheet, mobile variant, or other media variant |
| `amphtml` | Declare the AMP version of this page. Less relevant in 2026 as AMP has been deprecated by Google |
| `prev` | Previous page in a paginated series. Google announced in 2019 they ignore this; still useful for browser navigation |
| `next` | Next page in a paginated series. Same status as prev |
| `up` | Parent resource in a hierarchical structure |

**Performance:**

| rel | Purpose | `as` required |
|---|---|---|
| `preload` | Fetch this resource at high priority for the current page. Discovered before HTML parsing finishes | Yes |
| `prefetch` | Fetch this resource at low priority for likely future navigation | No, but recommended |
| `preconnect` | Open a TCP/TLS connection to this origin in advance | No |
| `dns-prefetch` | Resolve DNS for this origin in advance | No |
| `modulepreload` | Preload a JavaScript ES module and parse it | No |
| `prerender` | Render this URL in a background tab. Largely replaced by Speculation Rules |

**Other:**

| rel | Purpose |
|---|---|
| `stylesheet` | This resource is a CSS stylesheet. Almost never used as HTTP Link header; use HTML link tag |
| `icon` | Favicon. Almost never used as HTTP Link; use HTML link tag |
| `manifest` | PWA manifest URL |
| `help` | Reference to a help document |
| `license` | Reference to license information for the content |
| `author` | Reference to author information |

### 6.4 The `as` Parameter For Preload

`rel="preload"` requires an `as` parameter declaring the resource type. The browser uses this to set the correct request priority, Accept headers, and CSP enforcement. Wrong `as` value causes the browser to discard the preload and re fetch.

| `as` value | For |
|---|---|
| `script` | JavaScript files |
| `style` | CSS files |
| `font` | Web fonts. Requires `crossorigin` parameter |
| `image` | Images (JPEG, PNG, WebP, AVIF, SVG) |
| `fetch` | XHR or fetch() targets (JSON APIs, etc). Requires `crossorigin` if cross origin |
| `document` | iframe document |
| `video` | Video files |
| `audio` | Audio files |
| `track` | WebVTT track files |
| `worker` | Web Worker scripts |

### 6.5 fetchpriority

Modern browsers (Chrome 101+, Edge 101+, Safari 17.2+, Firefox 132+) support the `fetchpriority` parameter on preload and other resource hints:

```
Link: </hero.webp>; rel=preload; as=image; fetchpriority=high
```

Values: `high`, `low`, `auto` (default). Used to bump or demote a specific resource's priority within the browser's fetch queue. Critical for LCP optimization: marking the hero image as `fetchpriority=high` typically moves LCP earlier by 100 to 400 ms.

### 6.6 crossorigin

Required for fonts and for any cross origin preload that uses CORS:

```
Link: </fonts/main.woff2>; rel=preload; as=font; crossorigin
Link: <https://cdn.example.com/api.js>; rel=preload; as=script; crossorigin=anonymous
```

Values: `anonymous` (no credentials sent, default for `crossorigin`) or `use-credentials` (send cookies and auth).

### 6.7 How To Build It On Bubbles

**Canonical for a PDF or other non HTML asset:**

```nginx
location = /downloads/whitepaper.pdf {
    add_header Link '<https://example.com/whitepapers/q3-2026>; rel="canonical"' always;
}
```

The PDF is the asset; the canonical URL is the HTML landing page about the whitepaper. This consolidates index signals onto the HTML.

**Hreflang via HTTP header (for non HTML):**

```nginx
location = /downloads/manual.pdf {
    add_header Link '<https://example.com/downloads/manual.pdf>; rel="alternate"; hreflang="en-US", <https://example.com/es/downloads/manual.pdf>; rel="alternate"; hreflang="es-MX", <https://example.com/downloads/manual.pdf>; rel="alternate"; hreflang="x-default"' always;
}
```

Each PDF in a multi region cluster declares every other language version. The set must include `x-default` and must be reciprocal (every variant lists every other variant including itself).

**Preload critical assets:**

```nginx
location = / {
    # Preload the hero image, critical CSS, and primary font
    add_header Link '</css/critical.css>; rel="preload"; as="style", </fonts/manrope.woff2>; rel="preload"; as="font"; crossorigin, </images/hero.webp>; rel="preload"; as="image"; fetchpriority="high"' always;
    expires 0;
    add_header Cache-Control "public, max-age=0, must-revalidate" always;
}
```

These preloads start fetching as soon as the response headers arrive at the browser, before HTML parsing begins. For a hero image LCP candidate, this typically improves LCP by 100 to 300 ms.

**Preconnect to external origins (rare on Bubbles since we self host everything):**

```nginx
location / {
    add_header Link '<https://www.googletagmanager.com>; rel="preconnect", <https://www.google-analytics.com>; rel="preconnect"' always;
}
```

Useful when third party origins are unavoidable (analytics, tag manager). Establishes the TCP/TLS connection before the page needs the resource.

**RSS feed declaration via HTTP:**

```nginx
location / {
    add_header Link '</rss>; rel="alternate"; type="application/rss+xml"; title="Site Feed"' always;
}
```

Feed readers and browsers that support RSS auto discovery will find the feed through this header.

**Snippet pattern for the add_header inheritance trap:**

```nginx
# /etc/nginx/snippets/preloads.conf
add_header Link '</css/critical.css>; rel="preload"; as="style", </fonts/manrope.woff2>; rel="preload"; as="font"; crossorigin' always;

# In every relevant location
location ~* \.html$ {
    include snippets/preloads.conf;
    add_header Cache-Control "public, max-age=0, must-revalidate" always;
}
```

### 6.8 HTTP 103 Early Hints (the killer feature)

Nginx 1.25+ supports HTTP 103 Early Hints, which lets the server send `Link` preload headers before the final response is ready. The flow:

1. Browser requests `/`.
2. Server returns `103 Early Hints` immediately with `Link` headers.
3. Browser sees the preload directives and begins fetching critical resources.
4. Server takes 200 to 800 ms to compute the page (database queries, template rendering).
5. Server returns `200 OK` with the page body.
6. By the time the page parses, critical resources are already in cache.

Configuration:

```nginx
location / {
    early_hints 103;
    add_header Link '</css/critical.css>; rel="preload"; as="style"' always;
    proxy_pass http://127.0.0.1:9090;
}
```

The `early_hints 103` directive (nginx 1.25+) tells nginx to emit a 103 status with the `Link` headers before the final response. Supported in Chrome 103+, Edge 103+, Safari 17+, and Firefox 120+. Older browsers ignore the 103 and process the final response normally.

This is the single highest impact optimization for dynamic sites with slow first byte times. On a FastAPI sidecar that takes 400 ms to render, Early Hints typically reduces LCP by 200 to 400 ms with no application change.

### 6.9 How To Verify

```bash
# 1. Confirm Link header is present
curl -sI https://example.com/ | grep -i link

# 2. Pretty print multiple Link entries
curl -sI https://example.com/ | grep -i '^link:' | tr ',' '\n' | sed 's/^/    /'

# 3. Verify canonical for a PDF
curl -sI https://example.com/downloads/report.pdf | grep -i link

# 4. Check hreflang variants
curl -sI https://example.com/ | grep -i link | grep -o 'hreflang="[^"]*"' | sort -u

# 5. Test preload is being emitted
curl -sI https://example.com/ | grep -i link | grep -o 'rel="preload"'

# 6. Verify Early Hints (requires nginx 1.25+ and a client that supports 103)
curl -sv --http2 https://example.com/ 2>&1 | grep -i "HTTP/2 103"

# 7. Check that Googlebot sees the same Link header
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/ | grep -i link

# 8. Verify the URLs in Link headers actually resolve
for url in $(curl -sI https://example.com/ | grep -i link | grep -oE '<[^>]+>' | tr -d '<>'); do
    echo "=== $url ==="
    curl -sI "$url" | head -1
done
```

### 6.10 Troubleshooting

**Symptom: Preload appears in header but browser fetches the resource twice.**
Causes:
1. `as` value mismatch. The browser fetched with `as=script` priority but the page actually loaded the resource as a stylesheet. Fix the `as` to match.
2. Cross origin without `crossorigin` parameter. Fonts especially fail this way. Add `crossorigin` to the Link header.
3. Different URL between preload and actual use. The preload was for `/css/main.css` but the HTML loaded `/css/main.v2.css`. The URLs must match byte for byte.

**Symptom: Canonical via HTTP Link header is ignored by Google.**
Causes:
1. Conflicts with an HTML `<link rel=canonical>` declaring a different URL. Decide which is authoritative and remove the other.
2. The Link header URL is relative when it should be absolute. Always use absolute URLs in `rel=canonical`.
3. The canonical target returns 404 or 5xx. Google ignores broken canonicals.
4. The canonical target itself canonicalizes elsewhere (a chain). Resolve to the final canonical.

**Symptom: Early Hints not being sent.**
Causes:
1. Nginx version under 1.25. Check `nginx -v`.
2. `early_hints 103;` directive missing from the location.
3. The client does not support 103 (most older HTTP clients do not). Test with `curl -sv --http2`.
4. The upstream returned the final response too quickly for nginx to emit the 103 first. Acceptable; the browser still uses the final response's Link headers.

**Symptom: Hreflang cluster has missing reciprocity warnings in Search Console.**
Every variant in the cluster must list every other variant including itself. Audit:

```bash
for variant in en-US es-MX en-GB; do
    URL="https://example.com/$variant/page"
    echo "=== $URL ==="
    curl -sI "$URL" | grep -i link | grep -oE 'hreflang="[^"]*"' | sort -u
done
```

Each should list the same set including `x-default`. Add any missing.

**Symptom: Browser console warns "The resource was preloaded but not used within a few seconds".**
The preload is wasted. Either the resource is not actually critical and should be removed from preload, or the page is not loading it correctly. Audit which resources truly contribute to LCP and preload only those.

### 6.11 How To Fix Common Breakage

**Case: PDFs have no canonical, are competing with HTML in search.**
Add canonical headers to the PDF location:

```nginx
location ~ ^/downloads/(.*)\.pdf$ {
    # Compute the canonical HTML URL based on the PDF name
    # Adjust the pattern to match your site structure
    add_header Link '<https://example.com/whitepapers/$1>; rel="canonical"' always;
    add_header X-Robots-Tag "noindex" always;
}
```

The PDF declares the HTML page as canonical and is itself noindex. Index signals consolidate onto the HTML.

**Case: LCP is slow due to hero image discovery latency.**
Preload the hero image with high fetchpriority:

```nginx
location = / {
    add_header Link '</images/hero-2026.webp>; rel="preload"; as="image"; fetchpriority="high"' always;
}
```

If the hero image varies by viewport (responsive images), preload only the largest variant and let the browser pick the right one via srcset. Or, more advanced, use the `media` attribute on the preload:

```
Link: </hero-large.webp>; rel=preload; as=image; media="(min-width: 1024px)", </hero-small.webp>; rel=preload; as=image; media="(max-width: 1023px)"
```

**Case: Web font loads late and causes FOIT (flash of invisible text).**
Preload the font with crossorigin:

```nginx
location ~* \.html$ {
    add_header Link '</fonts/manrope-400.woff2>; rel="preload"; as="font"; crossorigin' always;
}
```

The font fetch starts immediately. Combined with `font-display: swap` in CSS, FOUT and FOIT are largely eliminated.

**Case: Dynamic page from FastAPI sidecar is slow to TTFB.**
Enable Early Hints:

```nginx
location /dashboard {
    early_hints 103;
    add_header Link '</css/dashboard.css>; rel="preload"; as="style", </js/dashboard.js>; rel="preload"; as="script"' always;
    proxy_pass http://127.0.0.1:9090;
}
```

Modern Chrome and Edge users see LCP improve by 200 to 400 ms with no application code change.

---

## 7. LOCATION (REDIRECT TARGET WITH STATUS CODE SEMANTICS)

### 7.1 What It Does

`Location` declares the URL a client should fetch instead of the current one. Required for any 3xx redirect response. Defined in RFC 9110 Section 10.2.2.

```
HTTP/2 301
Location: https://example.com/new-url
```

The status code that accompanies `Location` carries the semantic meaning: is this redirect permanent or temporary, must the HTTP method be preserved, and how should clients cache the redirect.

### 7.2 The Five Redirect Status Codes

| Code | Name | Permanent? | HTTP method preserved on redirect? | Cacheable by browser? |
|---|---|---|---|---|
| 301 | Moved Permanently | Yes | Often changed to GET historically; modern clients preserve | Yes, indefinitely |
| 302 | Found | No (temporary) | Often changed to GET; modern clients vary | Sometimes |
| 303 | See Other | No | Always changed to GET | No |
| 307 | Temporary Redirect | No | Strictly preserved | No |
| 308 | Permanent Redirect | Yes | Strictly preserved | Yes |

**The rules in practice:**

* **301**: use for permanent URL changes where the original URL is being abandoned. Most common redirect code. Browsers cache aggressively, which means changing the destination later requires cache busting. Search engines consolidate signals into the target URL.
* **302**: use for temporary URL changes that may revert. The original URL remains indexed. Common for A/B tests, geographic redirects, maintenance pages.
* **303**: use after a form POST to redirect to a confirmation page (Post/Redirect/Get pattern). Forces GET on the redirect to prevent accidental form resubmission on browser refresh.
* **307**: same semantics as 302 (temporary) but strictly preserves the HTTP method. A POST stays a POST. Use for temporary redirects of API endpoints where method preservation matters.
* **308**: same semantics as 301 (permanent) but strictly preserves the HTTP method. Use for permanent redirects of API endpoints.

**Google's stance** (John Mueller, 2016 and reaffirmed multiple times): all redirect codes are treated equivalently for SEO ranking purposes. The full PageRank passes through any of them. Choose based on technical semantics, not perceived SEO benefit.

**Practical Bubbles defaults:**

* HTTP to HTTPS migration: 301.
* www to non www (or vice versa): 301.
* Old URL to new URL after a site restructure: 301.
* Marketing campaign redirect: 302 (in case the campaign URL needs to change).
* Geographic or device redirect: 302.
* API endpoint version migration: 308 (preserves POST method).
* Login redirect after form submit: 303.

### 7.3 How To Build It On Bubbles

**Simple permanent redirect:**

```nginx
location = /old-page {
    return 301 https://example.com/new-page;
}
```

**Wildcard redirect for an entire path:**

```nginx
location /old-section/ {
    return 301 https://example.com/new-section/$request_uri;
}
```

Caution: `$request_uri` includes the leading `/old-section/`. For a clean rewrite, use a different variable:

```nginx
location ~ ^/old-section/(.*)$ {
    return 301 https://example.com/new-section/$1;
}
```

**HTTP to HTTPS redirect (every Bubbles site has this):**

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

**www to non www canonicalization:**

```nginx
server {
    listen 443 ssl;
    server_name www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;
    # ... actual site config ...
}
```

**Subdomain to canonical domain (for the 14+ paying client subdomains on thatwebhostingguy.com that 301 to client custom domains):**

```nginx
server {
    listen 443 ssl;
    server_name arcounselingandwellness.thatwebhostingguy.com;
    return 301 https://arcounselingandwellness.com$request_uri;
}
```

**Conditional redirect based on URL pattern:**

```nginx
server {
    server_name example.com;

    # Redirect old blog URLs to new structure
    location ~ ^/blog/(\d{4})/(\d{2})/(.+)$ {
        return 301 https://example.com/articles/$1-$2/$3;
    }

    # Redirect old product URLs based on a map
    location /products/ {
        try_files $uri @product_redirects;
    }

    location @product_redirects {
        if ($product_redirect != "") {
            return 301 $product_redirect;
        }
        return 404;
    }
}

# Defined at http level
map $request_uri $product_redirect {
    default                       "";
    /products/old-widget          /products/new-widget;
    /products/old-gadget          /products/new-gadget;
    /products/discontinued        /;
}
```

**Temporary redirect for maintenance page:**

```nginx
server {
    server_name example.com;

    set $maintenance 0;
    # Toggle this to 1 during maintenance
    # if ($maintenance) {
    #     return 302 /maintenance.html;
    # }

    # ... rest of config ...
}
```

**API endpoint version migration (308 to preserve POST):**

```nginx
location /api/v1/ {
    return 308 https://example.com/api/v2/$request_uri;
}
```

A POST request to `/api/v1/something` becomes a POST to `/api/v2/something`. The body is preserved.

### 7.4 The Open Redirect Vulnerability

A redirect endpoint that takes the target URL from user input without validation is a phishing vector. Attackers craft URLs like:

```
https://example.com/redirect?url=https://phishing-site.com/login
```

The link appears to come from your domain, lending credibility. The victim clicks, lands on the phishing site, enters credentials.

**Defenses:**

**1. Never accept full URLs from user input.** Only accept path components and prepend a fixed origin:

```nginx
# Wrong (vulnerable)
location = /r {
    return 302 $arg_url;
}

# Right
location = /r {
    if ($arg_to ~ ^/[a-z0-9/_-]+$) {
        return 302 https://example.com$arg_to;
    }
    return 400;
}
```

**2. Validate against an allow list of full URLs:**

```nginx
map $arg_url $is_allowed_redirect {
    default                          "";
    https://example.com/promo        "1";
    https://example.com/login        "1";
    https://example.com/help         "1";
}

location = /r {
    if ($is_allowed_redirect = "") {
        return 400;
    }
    return 302 $arg_url;
}
```

**3. Require an HMAC signature on the redirect parameter** so attackers cannot forge new targets. Implement at the upstream FastAPI sidecar level for any redirect endpoint that must accept dynamic targets.

**4. Never combine user input with a redirect status code without one of the above defenses.** Treat redirect endpoints with the same care as authentication endpoints.

### 7.5 Redirect Chains And Crawl Budget

A redirect chain is a sequence of redirects: `/a` → `/b` → `/c` → `/d`. Each hop costs:

* One crawler request (against finite crawl budget).
* One TLS handshake plus one TCP round trip for users (latency adds up).
* Potentially lost PageRank, although Google denies this in modern documentation.

Google's official guidance is to keep chains under 5 hops. Practical recommendation: always 1 hop. Audit periodically:

```bash
# Trace any chain
curl -sLI https://example.com/old-url | grep -i "^location:"

# Count hops
curl -sLI https://example.com/old-url | grep -c "^HTTP/"
```

A `1` indicates direct response (no redirect or single redirect). A `2` or higher indicates a chain.

When a chain is detected:

1. Identify the final destination.
2. Update the original redirect to point directly to the final destination.
3. Leave the intermediate redirects in place for a few months in case other consumers have cached the intermediate URLs.
4. Remove the intermediate redirects once they show zero traffic in logs.

### 7.6 How To Verify

```bash
# 1. Confirm redirect status and target
curl -sI https://example.com/old-page | grep -iE "^(HTTP|Location):"

# 2. Follow the chain
curl -sLI https://example.com/old-page | grep -iE "^(HTTP|Location):"

# 3. Verify final destination resolves
curl -sIL -o /dev/null -w "%{url_effective} %{http_code}\n" https://example.com/old-page

# 4. Verify the redirect preserves the path (for wildcard redirects)
for path in "" "/sub/page" "/file.pdf?query=1"; do
    URL="https://example.com/old-section$path"
    echo "=== $URL ==="
    curl -sI "$URL" | grep -i location
done

# 5. Test that POST is preserved (for 307 / 308)
curl -X POST -sI https://example.com/api/v1/endpoint | grep -iE "^(HTTP|Location):"

# 6. Audit: find chains
for url in https://example.com/old-1 https://example.com/old-2 https://example.com/old-3; do
    HOPS=$(curl -sLI "$url" | grep -c "^HTTP/")
    DEST=$(curl -sIL -o /dev/null -w "%{url_effective}" "$url")
    echo "$url -> $DEST ($HOPS hops)"
done

# 7. Verify Googlebot follows correctly
curl -sIL -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/old-page | grep -iE "^(HTTP|Location):"
```

### 7.7 Troubleshooting

**Symptom: Redirect loop (browser shows "ERR_TOO_MANY_REDIRECTS").**
Causes:
1. Two redirects pointing at each other (`/a` → `/b` → `/a`).
2. Server block redirects HTTP to HTTPS, but the HTTPS server block also has an HTTP to HTTPS redirect, causing infinite loops.
3. A wildcard redirect catches its own target (`location /new/ { return 301 /new/page; }` redirects `/new/page` to `/new/page`).
Fix: trace the chain with curl, identify the loop, rewrite one of the redirects to terminate.

**Symptom: Redirect target has trailing slash but original did not (or vice versa) and Google flags duplicates.**
Cause: inconsistent trailing slash policy.
Fix: pick one policy (typically without trailing slash for non directory URLs) and enforce site wide:

```nginx
# Force no trailing slash on file like URLs
rewrite ^/(.*)/$ /$1 permanent;
```

Or the opposite. Document which is the canonical form and stick to it.

**Symptom: 301 cached forever in browsers, even after fixing the redirect target.**
Cause: browsers cache 301 indefinitely per the HTTP spec. The cached redirect bypasses the server entirely.
Fix options:
1. Change the original URL to a new path that has not been cached as a redirect.
2. Wait for the browser cache to expire (could be months).
3. Use a 302 in the future for redirects that might change.

**Symptom: POST request becomes GET after redirect.**
Cause: the redirect status code is 301, 302, or 303, all of which historically changed the method to GET.
Fix: use 307 (temporary) or 308 (permanent) to strictly preserve the method.

**Symptom: Redirect works in browser but Googlebot still indexes the original URL.**
Causes:
1. The redirect status is 302 or 307, which signals "do not consolidate". Change to 301 or 308 for permanent moves.
2. There is a chain. Google may not follow long chains. Collapse to one hop.
3. The original URL has internal links pointing to it. Update internal links to point directly to the new URL.
4. Time. Consolidation takes weeks. Be patient.

**Symptom: Open redirect security alert.**
The redirect endpoint accepts unvalidated user input.
Fix: implement an allow list, signature validation, or both. See Section 7.4.

### 7.8 How To Fix Common Breakage

**Case: Old domain serving 301 to new domain, but new domain still points back to old in some links.**
Audit and fix:

```bash
# Find any link to the old domain in current site
grep -r "old-domain.com" /var/www/sites/new-domain.com/
```

Replace every occurrence with the new domain. Then verify no infinite loop exists.

**Case: Redirect chain longer than 1 hop discovered in Search Console.**
List the chain:

```bash
curl -sLI https://example.com/old-url | grep -iE "^(HTTP|Location):"
```

Update the originating server block to redirect directly to the final destination:

```nginx
location = /old-url {
    # Was: return 301 /intermediate;
    return 301 https://example.com/final-destination;
}
```

`nginx -t && systemctl reload nginx`.

**Case: HTTPS redirect causing certificate warning.**
The redirect target is `https://example.com` but the SSL certificate covers only `https://www.example.com`. Either:

1. Get a certificate covering both names (typical with Let's Encrypt: include both in `-d` flags).
2. Redirect to the name actually covered.

**Case: Need to redirect a thousand old URLs to specific new URLs.**
Use a map for performance:

```nginx
# In http context
map $request_uri $legacy_redirect {
    default                                    "";
    /old/page-1.html                           /new/page-1;
    /old/page-2.html                           /new/page-2;
    /old/page-3.html                           /new/page-3;
    # ... thousands more ...
}

# In server context
if ($legacy_redirect != "") {
    return 301 $legacy_redirect;
}
```

Maps in nginx are O(1) lookups; even thousands of entries add negligible overhead.

For maps with thousands of entries, externalize:

```nginx
map $request_uri $legacy_redirect {
    include /etc/nginx/maps/legacy-redirects.map;
    default "";
}
```

And maintain `legacy-redirects.map` as a simple text file with one mapping per line.

**Case: Need to migrate from www to non www, preserving all paths.**

```nginx
server {
    listen 443 ssl;
    server_name www.example.com;
    return 301 https://example.com$request_uri;
}
```

Ensure your SSL certificate covers `example.com` (the redirect target). Verify with curl.

---

## 8. HOW THESE HEADERS INTERACT

The three headers do not act independently. Several combinations matter.

### 8.1 X-Robots-Tag On Redirects

A 301 or 302 response can carry an `X-Robots-Tag` header, but the directive is largely ignored by Google. The crawler follows the redirect and applies indexing rules to the target URL, not the redirect itself. The exception is `nofollow`, which can prevent the crawler from following the redirect at all, but this is rare and usually unintended.

```nginx
# Almost always unnecessary
location = /old-url {
    add_header X-Robots-Tag "noindex" always;
    return 301 /new-url;
}
```

The `X-Robots-Tag: noindex` on a 301 has no effect: the old URL was already being deindexed by the 301 itself.

### 8.2 Canonical In HTTP vs HTML

When both `Link: <...>; rel=canonical` HTTP header and `<link rel=canonical>` HTML tag are present, Google treats them as duplicate signals. If they agree, fine. If they disagree, Google picks one based on its own canonicalization logic, which may not match either. **Pick one and stick to it.**

For HTML pages: use the HTML `<link rel=canonical>` tag because it is easier to manage with templates and is what most tools expect.

For non HTML (PDF, image, JSON): use the HTTP `Link` header because there is no HTML to put a tag in.

### 8.3 Redirect Plus Canonical Tag On The Target

The redirect target should have a self referencing canonical tag:

```html
<!-- On the new URL: https://example.com/new-page -->
<link rel="canonical" href="https://example.com/new-page">
```

This is correct and expected. It reinforces that this is the canonical version, not a duplicate of some other page.

### 8.4 hreflang Cluster Cross References

Hreflang annotations declared via HTTP `Link` header or HTML `<link rel=alternate hreflang>` must form a reciprocal cluster. Every variant lists every other variant including itself. Missing any reciprocal reference triggers a Search Console warning and may cause hreflang to be ignored entirely.

For a PDF that exists in two languages:

```nginx
# On /en/manual.pdf
add_header Link '<https://example.com/en/manual.pdf>; rel="alternate"; hreflang="en-US", <https://example.com/es/manual.pdf>; rel="alternate"; hreflang="es-MX", <https://example.com/en/manual.pdf>; rel="alternate"; hreflang="x-default"' always;

# On /es/manual.pdf
add_header Link '<https://example.com/en/manual.pdf>; rel="alternate"; hreflang="en-US", <https://example.com/es/manual.pdf>; rel="alternate"; hreflang="es-MX", <https://example.com/en/manual.pdf>; rel="alternate"; hreflang="x-default"' always;
```

Both PDFs declare the same cluster. The cluster includes both files and an `x-default`.

### 8.5 Preload And Cache-Control

Preloaded resources should still have correct `Cache-Control` headers. If the preloaded asset has `Cache-Control: no-store`, the browser fetches it via the preload, but cannot store it for use by the actual page reference. Result: double fetch.

Ensure preloaded resources are cacheable. Combine with framework-http-caching-headers.md guidance: fingerprinted assets should be `Cache-Control: public, max-age=31536000, immutable` and are perfect preload candidates.

### 8.6 Location Plus Vary

A redirect response should typically not have `Vary` headers. The redirect itself does not vary by request headers. However, if the redirect target differs based on `Accept-Language` (geographic or language based redirect), then `Vary: Accept-Language` is required to prevent caches from serving the wrong target to a different audience.

```nginx
location = / {
    # Redirect based on Accept-Language
    if ($http_accept_language ~* "^es") {
        add_header Vary "Accept-Language" always;
        return 302 /es/;
    }
    add_header Vary "Accept-Language" always;
    return 302 /en/;
}
```

Note: language based redirects are a poor pattern for SEO. They confuse crawlers and prevent users from sharing links between languages. Prefer letting the user select their language explicitly and remembering the choice via cookie or path.

---

## 9. ASSET CLASS AND USE CASE RECIPES

Each block is paste ready. Copy the ones relevant to your build.

### 9.1 Production HTML page (full SEO header set)

```nginx
location ~* \.html$ {
    # X-Robots-Tag: allow indexing, set snippet limits
    add_header X-Robots-Tag "max-snippet:200, max-image-preview:large, max-video-preview:-1" always;

    # Link: preload critical resources
    add_header Link '</css/critical.css>; rel="preload"; as="style", </fonts/manrope.woff2>; rel="preload"; as="font"; crossorigin' always;

    # Cache directives from framework-http-caching-headers.md
    add_header Cache-Control "public, max-age=0, must-revalidate" always;
}
```

### 9.2 PDF (block from index, canonical to HTML)

```nginx
location ~ ^/whitepapers/(.*)\.pdf$ {
    add_header X-Robots-Tag "noindex" always;
    add_header Link '<https://example.com/whitepapers/$1>; rel="canonical"' always;
    add_header Cache-Control "public, max-age=86400" always;
    add_header Content-Disposition "attachment" always;
}
```

### 9.3 Image asset (allow indexing for image search)

```nginx
location ~* \.(jpg|jpeg|png|webp|avif)$ {
    # Default: allow image indexing
    # If you want to block: add_header X-Robots-Tag "noindex" always;
    add_header Cache-Control "public, max-age=2592000" always;
}
```

### 9.4 Image asset (block from image search but page itself indexable)

```nginx
location ~* \.(jpg|jpeg|png|webp|avif)$ {
    add_header X-Robots-Tag "noindex" always;
    add_header Cache-Control "public, max-age=2592000" always;
}
```

### 9.5 Staging or development subdomain (full block)

```nginx
server {
    server_name staging.example.com;

    # Universal block
    add_header X-Robots-Tag "noindex, nofollow" always;

    # Plus robots.txt for belt and suspenders
    location = /robots.txt {
        return 200 "User-agent: *\nDisallow: /\n";
        add_header Content-Type "text/plain";
    }

    # Plus HTTP basic auth as a third layer
    auth_basic "Staging";
    auth_basic_user_file /etc/nginx/.staging-htpasswd;
}
```

### 9.6 API endpoint (do not index, do not follow)

```nginx
location /api/ {
    add_header X-Robots-Tag "noindex, nofollow" always;
    proxy_pass http://127.0.0.1:9090;
}
```

### 9.7 Search result page (do not index, do follow)

```nginx
location /search {
    add_header X-Robots-Tag "noindex, follow" always;
    # The follow allows crawlers to discover linked content
}
```

Internal search result pages are duplicate content for each query and should not be indexed. Allow follow so crawlers can discover the linked actual content pages.

### 9.8 Paginated section (allow indexing of all pages, no rel prev/next)

```nginx
location ~ ^/blog/page/([0-9]+)$ {
    # Google ignored prev/next as of 2019; do not bother with Link rel=prev/next
    add_header X-Robots-Tag "max-snippet:160, max-image-preview:standard" always;
}
```

### 9.9 Time limited campaign page

```nginx
location = /promotions/black-friday-2026 {
    add_header X-Robots-Tag "unavailable_after: 1 Dec 2026 00:00:00 GMT" always;
}
```

### 9.10 Permanent www to non www redirect

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    return 301 https://example.com$request_uri;
}
```

### 9.11 HTTP to HTTPS redirect

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}
```

### 9.12 Old blog URL structure to new

```nginx
# Old: /2024/05/post-title
# New: /blog/post-title
location ~ ^/(\d{4})/(\d{2})/(.+)$ {
    return 301 /blog/$3;
}
```

### 9.13 Subdomain to client custom domain (Bubbles wildcard pattern)

```nginx
server {
    listen 443 ssl;
    server_name handledtax.thatwebhostingguy.com;
    ssl_certificate /etc/letsencrypt/live/thatwebhostingguy.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/thatwebhostingguy.com/privkey.pem;
    return 301 https://handledtax.com$request_uri;
}
```

### 9.14 API version migration (preserve POST method)

```nginx
location /api/v1/ {
    return 308 /api/v2/$request_uri;
}
```

### 9.15 Maintenance mode toggle

```nginx
# At server level
set $maintenance 0;

if (-f /var/www/sites/example.com/.maintenance) {
    set $maintenance 1;
}

location / {
    if ($maintenance) {
        return 302 /maintenance.html;
    }
    try_files $uri $uri/ $uri.html =404;
}

location = /maintenance.html {
    # Allow access to this even in maintenance mode
    internal;
}
```

To enable maintenance: `touch /var/www/sites/example.com/.maintenance`. To disable: `rm /var/www/sites/example.com/.maintenance`. No nginx reload required.

---

## 10. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete SEO header stanza, layered with caching (framework-http-caching-headers.md) and content (framework-http-content-headers.md).

```nginx
# /etc/nginx/sites-available/example.com

# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

# www to non www
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    return 301 https://example.com$request_uri;
}

# Main canonical server
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

    # HTTP/3 advertisement
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # ===== HTML PAGES =====
    location ~* \.html$ {
        include snippets/common-security-headers.conf;
        # Baseline indexing directives
        add_header X-Robots-Tag "max-snippet:200, max-image-preview:large, max-video-preview:-1" always;
        # Preload critical resources
        add_header Link '</css/critical.css>; rel="preload"; as="style", </fonts/manrope.woff2>; rel="preload"; as="font"; crossorigin, </images/hero.webp>; rel="preload"; as="image"; fetchpriority="high"' always;
        # Caching from framework-http-caching-headers.md
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        # Language from framework-http-content-headers.md
        add_header Content-Language "en-US" always;
    }

    # ===== PDFs (noindex, canonical to HTML landing) =====
    location ~ ^/whitepapers/(.*)\.pdf$ {
        include snippets/common-security-headers.conf;
        add_header X-Robots-Tag "noindex" always;
        add_header Link '<https://example.com/whitepapers/$1>; rel="canonical"' always;
        add_header Content-Disposition "attachment" always;
        add_header Cache-Control "public, max-age=86400" always;
    }

    # ===== IMAGES (allow image search indexing) =====
    location ~* \.(jpg|jpeg|png|webp|avif|svg|gif)$ {
        include snippets/common-security-headers.conf;
        add_header Cache-Control "public, max-age=2592000" always;
        # No X-Robots-Tag: default behavior allows image indexing
    }

    # ===== FONTS, CSS, JS (no indexing needed; cache forever) =====
    location ~* \.(woff2|woff|ttf|otf|css|js|mjs)$ {
        include snippets/common-security-headers.conf;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }

    # ===== INTERNAL SEARCH (noindex but follow) =====
    location /search {
        include snippets/common-security-headers.conf;
        add_header X-Robots-Tag "noindex, follow" always;
    }

    # ===== API (sidecar, noindex) =====
    location /api/ {
        include snippets/common-security-headers.conf;
        add_header X-Robots-Tag "noindex, nofollow" always;
        # Early Hints for dynamic responses
        early_hints 103;
        add_header Link '</css/dashboard.css>; rel="preload"; as="style"' always;
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ===== ROBOTS.TXT, SITEMAPS, AI/LLM FILES =====
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

    # ===== LEGACY URL MAP REDIRECTS =====
    if ($legacy_redirect != "") {
        return 301 $legacy_redirect;
    }

    # ===== DEFAULT =====
    location / {
        include snippets/common-security-headers.conf;
        add_header X-Robots-Tag "max-snippet:200, max-image-preview:large, max-video-preview:-1" always;
        add_header Content-Language "en-US" always;
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        try_files $uri $uri/ $uri.html =404;
    }
}

# Legacy URL map (defined at http level in nginx.conf or included)
# map $request_uri $legacy_redirect {
#     default                       "";
#     /old/page-1.html              /new/page-1;
#     /old/page-2.html              /new/page-2;
# }
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

## 11. AUDIT CHECKLIST

Run through these 50 items for any production site.

### X-Robots-Tag

1. [ ] Production HTML pages have `X-Robots-Tag: max-snippet:200, max-image-preview:large, max-video-preview:-1` (or appropriate values, NOT `noindex`).
2. [ ] Staging or development subdomains have `X-Robots-Tag: noindex, nofollow`.
3. [ ] PDFs at customer facing URLs have `X-Robots-Tag: noindex` (unless they should be indexed).
4. [ ] CSV exports and other downloadable data files have `X-Robots-Tag: noindex`.
5. [ ] API endpoints have `X-Robots-Tag: noindex, nofollow`.
6. [ ] Internal search result pages have `X-Robots-Tag: noindex, follow`.
7. [ ] No production HTML page has `X-Robots-Tag: noindex` accidentally.
8. [ ] No directive uses deprecated tokens (`noodp`, `noydir`, `noyaca`).
9. [ ] Bot prefixed directives (`googlebot:`, `bingbot:`) use valid bot tokens.
10. [ ] `unavailable_after` dates use RFC 850 / RFC 822 format.
11. [ ] `nosnippet` is used only when intentional (it blocks AI Overviews too).
12. [ ] X-Robots-Tag from HTTP and `<meta name="robots">` from HTML do not conflict on the same page.

### Link header

13. [ ] HTML pages preload critical CSS, fonts, and hero image via `Link` HTTP header.
14. [ ] Font preloads include `crossorigin` parameter.
15. [ ] All preload URLs use absolute or relative paths that match the actual usage in HTML.
16. [ ] Preload `as` values are correct (`style`, `script`, `font`, `image`, etc).
17. [ ] LCP candidate image has `fetchpriority="high"`.
18. [ ] PDFs and other non HTML assets that need canonicalization use `Link: <...>; rel="canonical"`.
19. [ ] Hreflang annotations form a reciprocal cluster (every variant lists every other variant).
20. [ ] Hreflang cluster includes `x-default`.
21. [ ] No `rel=prev` or `rel=next` headers (Google ignored these in 2019).
22. [ ] Early Hints (103) is enabled for dynamic responses where TTFB exceeds 200 ms.
23. [ ] Nginx version is 1.25+ if Early Hints is used.

### Location and redirects

24. [ ] HTTP requests to port 80 always redirect to HTTPS with 301.
25. [ ] www to non www (or vice versa) is consistent and uses 301.
26. [ ] No redirect chain exceeds 1 hop.
27. [ ] All 301 redirect targets return 200 OK (not another redirect or error).
28. [ ] Permanent redirects use 301 or 308 (not 302 or 307).
29. [ ] Temporary redirects use 302 or 307.
30. [ ] API version migration redirects use 308 to preserve POST method.
31. [ ] No redirect endpoint accepts unvalidated URL parameters from user input (no open redirects).
32. [ ] Trailing slash policy is consistent site wide.
33. [ ] Internal links point directly to the canonical URL (no internal links to redirect sources).
34. [ ] Redirect targets are absolute URLs or path absolute (no protocol relative or relative redirects).
35. [ ] SSL certificate covers all redirect target hostnames.

### Cross cutting

36. [ ] `nginx -t` passes without warnings.
37. [ ] `nginx -T` shows all expected SEO directives in effect for each server block.
38. [ ] Search Console URL Inspection shows correct indexing status for sample URLs.
39. [ ] No URL is in Google's index that should be noindex.
40. [ ] No URL is noindex that should be in Google's index.
41. [ ] No URL is blocked by robots.txt that needs to be noindex (the noindex would not be seen).
42. [ ] PDF count in Search Console matches expected count (audits whether unwanted PDFs are indexed).
43. [ ] Hreflang errors in Search Console are at zero.
44. [ ] Coverage report in Search Console shows no unexpected "Excluded by noindex" entries.
45. [ ] Redirect chain audit (run periodically) shows zero chains over 1 hop.
46. [ ] All preload directives have corresponding usage in HTML (no orphan preloads wasting bandwidth).
47. [ ] LCP image is preloaded with fetchpriority=high.
48. [ ] Early Hints (if used) actually arrives before final response (verified via `curl --http2`).
49. [ ] Open redirect scanner reports clean.
50. [ ] Internal link audit shows no broken links to redirected URLs.

A site that passes all 50 has indexing, resource hints, and forwarding correctly configured.

---

## 12. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: noindex behind robots.txt block.**
Symptom: page stays indexed forever.
Why it breaks: robots.txt prevents fetching; crawler never sees the noindex directive.
Fix: remove robots.txt block, let crawler fetch and process noindex, then optionally re add.

**Pitfall 2: Conflicting X-Robots-Tag and meta robots.**
Symptom: page sometimes indexed, sometimes not; behavior inconsistent.
Why it breaks: both signals are read; the most restrictive wins, but team members assume one or the other is authoritative.
Fix: align both. Pick one as the source of truth (typically HTTP header for non HTML, HTML tag for HTML pages) and maintain consistency.

**Pitfall 3: noindex on a redirected URL.**
Symptom: the noindex header is wasted; the 301 itself handles deindexing.
Why it breaks: a 301 response signals the URL has moved; crawlers process the redirect, not the directives on the old URL.
Fix: remove noindex from redirect responses unless you have a specific reason.

**Pitfall 4: Canonical pointing to a redirected URL.**
Symptom: index signals are diluted; Google follows the chain and selects its own canonical.
Why it breaks: canonical should point to the final canonical URL, not to an intermediate that redirects.
Fix: audit all canonicals. Each must resolve to a 200 OK page that self references as canonical.

**Pitfall 5: Preload with wrong `as` value.**
Symptom: browser fetches the resource twice (preload, then real load).
Why it breaks: the browser cannot match the preloaded resource to the actual request because the priority and headers differ.
Fix: ensure `as` matches the resource type exactly. Font preload requires `as=font` AND `crossorigin`. Image is `as=image`. Script is `as=script`. Style is `as=style`.

**Pitfall 6: Preload of a resource the page does not use.**
Symptom: bandwidth wasted; console warning "preloaded but not used".
Why it breaks: a developer added a preload speculatively, then the corresponding HTML reference was removed or changed.
Fix: audit preloads. Remove any not actually used within a few seconds of page load.

**Pitfall 7: Hreflang cluster missing reciprocity.**
Symptom: Search Console hreflang errors; Google falls back to default behavior.
Why it breaks: hreflang requires every variant to list every other variant including itself.
Fix: build hreflang generation programmatically so all variants always list the full set.

**Pitfall 8: Redirect chain longer than 1 hop.**
Symptom: crawl budget waste; PageRank consolidation slow.
Why it breaks: nobody updated the source of a redirect when the destination itself was redirected.
Fix: audit periodically with curl `-sLI`. Collapse to direct redirects.

**Pitfall 9: Open redirect endpoint.**
Symptom: security alert; phishing campaigns abuse the redirector.
Why it breaks: endpoint accepts arbitrary target URL from user input without validation.
Fix: allow list, signature, or only accept path components. See Section 7.4.

**Pitfall 10: Wrong redirect code for HTTP method preservation.**
Symptom: POST becomes GET after redirect; API calls fail.
Why it breaks: 301, 302, 303 historically change POST to GET.
Fix: use 307 (temporary) or 308 (permanent) for redirects that must preserve POST.

**Pitfall 11: 301 cached forever, hard to undo.**
Symptom: changed the redirect target but browsers still go to the old destination.
Why it breaks: per HTTP spec, browsers may cache 301 indefinitely.
Fix prevention: use 302 for redirects you might change in the future. Fix existing: change the source URL or wait for cache expiration.

**Pitfall 12: Self redirecting URL.**
Symptom: redirect loop; ERR_TOO_MANY_REDIRECTS.
Why it breaks: a location block redirects, and the target also matches the same location pattern.
Fix: tighten the regex match, or check the request URI before redirecting.

**Pitfall 13: Redirect target on a different SSL certificate.**
Symptom: browser shows certificate warning after redirect.
Why it breaks: redirecting `www.example.com` to `example.com` requires the certificate to cover `example.com`.
Fix: ensure the cert covers all redirect targets. With Let's Encrypt, include all hostnames in the `-d` flags.

**Pitfall 14: Add_header inheritance trap on X-Robots-Tag.**
Symptom: noindex applied to staging site disappears from specific file types.
Why it breaks: a location block with `add_header` wipes parent declarations.
Fix: snippet include pattern in every location.

**Pitfall 15: Early Hints emitted but contains preload for a non existent resource.**
Symptom: 404 in network panel for the preloaded resource; LCP not improved.
Why it breaks: preload URL is wrong or the resource was renamed.
Fix: audit Early Hints preload URLs. Each must resolve to 200 OK.

---

## 13. DIAGNOSTIC COMMANDS

Reference of every command useful for SEO header investigation.

### Inspect SEO headers

```bash
# Full SEO family at once
curl -sI https://example.com/page.html | grep -iE "^(x-robots-tag|link|location|content-language):"

# As Googlebot
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/page.html

# As Bingbot
curl -sI -A "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)" \
     https://example.com/page.html

# As ClaudeBot
curl -sI -A "Mozilla/5.0 (compatible; ClaudeBot/1.0; +claudebot@anthropic.com)" \
     https://example.com/page.html

# As GPTBot
curl -sI -A "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko); compatible; GPTBot/1.0; +https://openai.com/gptbot" \
     https://example.com/page.html
```

### X-Robots-Tag verification

```bash
# Confirm directive
curl -sI https://example.com/page.html | grep -i x-robots-tag

# Check all non HTML file types
for ext in pdf png jpg svg woff2 json xml csv zip; do
    URL="https://example.com/test.$ext"
    echo "=== $URL ==="
    curl -sI "$URL" 2>/dev/null | grep -i x-robots-tag
done

# Check for accidental noindex on production pages
for url in / /about /services /contact /blog; do
    echo "=== $url ==="
    curl -sI "https://example.com$url" | grep -i x-robots-tag
done

# Compare HTTP and HTML signals
echo "HTTP:"; curl -sI https://example.com/page.html | grep -i x-robots-tag
echo "HTML:"; curl -s https://example.com/page.html | grep -oE '<meta[^>]*name="robots"[^>]*>'
```

### Link header verification

```bash
# Confirm Link header presence and parse
curl -sI https://example.com/ | grep -i '^link:' | sed 's/, /\n    /g'

# Check canonical
curl -sI https://example.com/page.html | grep -oE 'rel="canonical"[^,]*'

# Check hreflang cluster
curl -sI https://example.com/ | grep -oE 'hreflang="[^"]*"' | sort -u

# Verify preload targets resolve
for url in $(curl -sI https://example.com/ | grep -i '^link:' | grep -oE '<[^>]+>' | tr -d '<>'); do
    STATUS=$(curl -sI "$url" | head -1)
    echo "$url -> $STATUS"
done

# Test Early Hints
curl -sv --http2 https://example.com/ 2>&1 | grep -iE "HTTP/2 (103|200)"
```

### Location and redirect verification

```bash
# Single redirect
curl -sI https://example.com/old-url | grep -iE "^(HTTP|Location):"

# Full chain
curl -sLI https://example.com/old-url | grep -iE "^(HTTP|Location):"

# Count hops
HOPS=$(curl -sLI https://example.com/old-url | grep -c "^HTTP/")
echo "Hops: $HOPS"

# Final URL and status
curl -sILo /dev/null -w "Final: %{url_effective} Status: %{http_code} Redirects: %{num_redirects}\n" \
     https://example.com/old-url

# Audit a list of old URLs
while IFS= read -r url; do
    FINAL=$(curl -sILo /dev/null -w "%{url_effective}" "$url")
    HOPS=$(curl -sLI "$url" | grep -c "^HTTP/")
    STATUS=$(curl -sILo /dev/null -w "%{http_code}" "$url")
    echo "$url -> $FINAL ($HOPS hops, final $STATUS)"
done < urls-to-audit.txt

# Verify POST is preserved on 307/308
curl -X POST -sI https://example.com/api/v1/something | grep -iE "^(HTTP|Location):"

# Detect redirect loop (timeout after 5 seconds, max 10 redirects)
curl -sLI --max-redirs 10 --max-time 5 https://example.com/maybe-looping
```

### Server side investigation

```bash
# Find every X-Robots-Tag in the config
grep -rn "X-Robots-Tag" /etc/nginx/

# Find every Link header
grep -rn "Link" /etc/nginx/sites-available/

# Find every redirect (return 301/302/etc, rewrite ... permanent)
grep -rn -E "return (301|302|303|307|308)|rewrite.*(permanent|redirect)" /etc/nginx/sites-available/

# Find any open redirect patterns (variable in return target)
grep -rn -E "return (301|302|303|307|308).*\\\$" /etc/nginx/sites-available/

# Check active config
nginx -T 2>/dev/null | grep -iE "x-robots-tag|^[[:space:]]*link |return 30|early_hints"

# Reload after changes
nginx -t && systemctl reload nginx
```

### Google Search Console verification

The authoritative SEO check. For each URL you care about:

1. Open Search Console for the property.
2. Top search bar: paste the URL.
3. Click Enter.
4. Wait for the report to load. Sections:
   * **URL is on Google**: indexed correctly.
   * **URL is not on Google**: not indexed; reason shown.
5. Click "Test live URL" to fetch with Googlebot now and see headers.
6. Under "Page indexing":
   * "Indexing allowed?" should be "Yes" for pages you want indexed.
   * "User-declared canonical" and "Google-selected canonical" should match.
7. Under "Crawl":
   * "Last crawl" date should be recent.
   * "Crawled as" should be Googlebot Smartphone (mobile first).

This is ground truth. Any disagreement between curl output and Search Console means the URL has not been re crawled yet.

---

## 14. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): Cache-Control, ETag, Last-Modified, Expires, Vary, Age. Preloaded resources should have correct cache directives; redirects often need no caching; X-Robots-Tag has no caching implications.
* [framework-http-content-headers.md](framework-http-content-headers.md): Content-Type, Content-Language, Content-Encoding, Content-Length, Content-Disposition. Content-Language pairs with hreflang declared via Link header. Content-Disposition pairs with X-Robots-Tag for downloadable file handling.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Section 24 (Bubbles Nginx config) inherits SEO header configuration from here.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook.
* [framework-hreflang.md](framework-hreflang.md): full coverage of multi region SEO. Companion to Link header hreflang in Section 6 here.
* [framework-robotstxt-crawlbudget.md](framework-robotstxt-crawlbudget.md): robots.txt syntax and crawl budget management. Pairs with X-Robots-Tag distinction in Section 5.5.
* [framework-canonicalization.md](framework-canonicalization.md): canonical URL strategy. Companion to Link rel=canonical and the redirect/canonical interaction in Section 8.
* [framework-corewebvitals.md](framework-corewebvitals.md): LCP, INP, CLS targets. Preload via Link header and Early Hints (103) directly improve LCP.
* [framework-security-headers.md](framework-security-headers.md): HSTS, CSP, X-Frame-Options. Open redirect security pairs with the broader security header stack.
* Google X-Robots-Tag documentation: https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag
* Google redirects documentation: https://developers.google.com/search/docs/crawling-indexing/301-redirects
* Google international and multi region: https://developers.google.com/search/docs/specialty/international/overview
* MDN Link header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Link
* MDN Location header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Location
* RFC 8288 (Web Linking): https://www.rfc-editor.org/rfc/rfc8288
* RFC 9110 (HTTP Semantics, redirects and Location): https://www.rfc-editor.org/rfc/rfc9110
* HTTP 103 Early Hints: https://www.rfc-editor.org/rfc/rfc8297

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

### X-Robots-Tag

Production HTML default:
```
X-Robots-Tag: max-snippet:200, max-image-preview:large, max-video-preview:-1
```

Block from indexing:
```
X-Robots-Tag: noindex, nofollow
```

Time limited:
```
X-Robots-Tag: unavailable_after: 31 Dec 2026 23:59:59 GMT
```

Bot specific:
```
X-Robots-Tag: googlebot: nosnippet
X-Robots-Tag: bingbot: noarchive
```

### Link

Canonical for non HTML:
```
Link: <https://example.com/canonical>; rel="canonical"
```

Preload hero image (LCP):
```
Link: </images/hero.webp>; rel="preload"; as="image"; fetchpriority="high"
```

Preload font:
```
Link: </fonts/main.woff2>; rel="preload"; as="font"; crossorigin
```

Hreflang (reciprocal cluster, set on every variant):
```
Link: <https://example.com/>; rel="alternate"; hreflang="en-US", <https://example.com/es/>; rel="alternate"; hreflang="es-MX", <https://example.com/>; rel="alternate"; hreflang="x-default"
```

### Location

| Scenario | Code |
|---|---|
| Permanent URL change | 301 |
| Temporary URL change | 302 |
| After form POST (force GET on next) | 303 |
| Temporary, preserve POST | 307 |
| Permanent, preserve POST | 308 |

HTTP to HTTPS:
```nginx
return 301 https://$host$request_uri;
```

www to non www:
```nginx
return 301 https://example.com$request_uri;
```

Old URL pattern to new:
```nginx
location ~ ^/old/(.*)$ { return 301 /new/$1; }
```

Three commands every operator should know:

```bash
# Confirm SEO headers on a URL
curl -sI https://example.com/page.html | grep -iE "^(x-robots-tag|link|location):"

# Trace a redirect chain
curl -sLI https://example.com/old-url | grep -iE "^(HTTP|Location):"

# Apply nginx changes
nginx -t && systemctl reload nginx
```

Two end to end tests:

```bash
# Verify a noindex directive is in effect
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/staging-page | grep -i x-robots-tag
# Should show: x-robots-tag: noindex, nofollow

# Verify a redirect resolves in one hop
curl -sILo /dev/null -w "Hops: %{num_redirects} Final: %{url_effective} Status: %{http_code}\n" \
     https://example.com/old-url
# Should show: Hops: 1 Final: https://example.com/new-url Status: 200
```

If both produce expected output, the SEO header stack is correctly wired.

---

End of framework-http-seo-headers.md.
