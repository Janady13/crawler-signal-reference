# framework-html-meta-robots.md

Comprehensive reference for HTML `<meta name="robots">` and crawler specific variants (`googlebot`, `bingbot`, `googlebot-news`). Covers every directive: `index`, `noindex`, `follow`, `nofollow`, `none`, `all`, `noarchive`, `nosnippet`, `max-snippet:N`, `max-image-preview:none|standard|large`, `max-video-preview:N`, `notranslate`, `noimageindex`, `unavailable_after:DATE`, `indexifembedded`, plus the related `data-nosnippet` HTML attribute and the non standard AI directives (`noai`, `noimageai`). Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the first framework beyond the HTTP wire layer.** The previous 12 frameworks (8 headers + 4 status codes) documented what nginx and FastAPI send and receive at the protocol level. This framework documents the HTML signals embedded in page content that crawlers parse after the HTTP response has been received. Companion to the 12 wire layer frameworks, especially framework-http-seo-headers.md which covers the X-Robots-Tag HTTP header (carrying the same directives at the protocol level for non HTML resources).

Audience: humans configuring per page indexing controls, AI assistants generating HTML head sections that crawlers interpret correctly, SEO operators auditing the directives present on every URL, developers building CMS templates that emit correct meta robots tags, and anyone troubleshooting "page not indexed despite no noindex", "site search results showing in Google despite blocked in robots.txt", "thin pages not deindexing after adding noindex".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Meta Robots Mental Model (read this first)
5. Meta Robots vs X-Robots-Tag (HTML vs HTTP decision)
6. The Five Meta Tag Names (robots, googlebot, bingbot, googlebot-news, plus AI variants)
7. The Complete Directive List
8. The data-nosnippet HTML Attribute (partial page control)
9. User Agent Precedence And Conflict Resolution
10. The robots.txt Plus noindex Paradox
11. AI Specific Directives (noai, noimageai, non standard)
12. The Bubbles Standard Patterns (per page type)
13. The Bubbles Thin Content Cleanup Pattern (Joseph's 5,800 page work, HTML side)
14. How Major Crawlers React To Each Directive
15. Asset Class And Use Case Recipes
16. Bubbles HTML Reference Block (paste ready)
17. Audit Checklist (50+ items)
18. Common Pitfalls
19. Diagnostic Commands
20. Cross-References

---

## 1. DEFINITION

The HTML `<meta name="robots">` tag is a page level directive embedded in the `<head>` of an HTML document. Crawlers parse it after fetching the page and apply the directives to determine indexing behavior, snippet generation, link following, archiving, and other behaviors. Defined informally across search engine documentation (Google, Bing, Yandex, Apple) with no single RFC owning the entire feature set.

```html
<head>
    <meta name="robots" content="noindex, follow">
    <meta name="googlebot" content="noimageindex">
    <meta name="bingbot" content="noarchive">
</head>
```

Three structural facts shape how the tag works:

* **One name per crawler scope.** `robots` applies to all crawlers; `googlebot` applies only to Google's search bot; `bingbot` applies only to Bing's; `googlebot-news` applies only to Google News. Multiple meta tags with different name attributes can coexist.
* **Comma separated directives in the content attribute.** Both case insensitive. `<meta name="robots" content="noindex, follow">` is equivalent to `<meta name="ROBOTS" content="NOINDEX, FOLLOW">`.
* **Most restrictive wins.** When multiple directives conflict (same page has `index` and `noindex`, or different crawler scopes give different signals), the most restrictive directive applies.

For Bubbles client sites, the meta robots tag is the primary mechanism for declaring "do not index this page" on HTML responses. The X-Robots-Tag HTTP header (from framework-http-seo-headers.md) is the equivalent for non HTML resources (PDFs, images, JSON APIs) and for cases where editing HTML is impractical (CDN edge rules, etc).

---

## 2. WHY IT MATTERS

Eight independent pressures push correct meta robots usage from "default behavior" to "actively managed signal" in 2025 and forward.

**Indexing the wrong pages drags down site quality.** Soft 404 pages, thin content templates, search results pages, filter parameter URLs, login walls: all are URLs that should not appear in Google's index. Without meta robots controls, they get indexed and dilute the site's overall quality signal. Pages that should rank for valuable terms rank worse because the site is "diluted" with low quality URLs.

**The robots.txt plus noindex paradox is widely misunderstood.** Per Google's documentation: if robots.txt blocks crawling of a URL, Google never fetches the page, never sees the noindex meta tag, and may still index the URL based on external signals (links pointing to it). The correct sequence is "allow crawling, then noindex"; the wrong sequence is "block in robots.txt and noindex"; the latter actually keeps the URL in the index.

**`<meta name="robots" content="index, follow">` is pointless.** John Mueller (Google) confirmed Googlebot ignores it; default behavior is index/follow. Sites that emit this on every page are wasting bytes and signaling no useful information. The framework documents when to emit directives (when changing default) and when to omit them (always allowed by default).

**Snippet length controls protect editorial content.** `max-snippet:0` removes snippets entirely (preventing Google from showing description text). `max-snippet:160` limits to 160 characters. Editorial sites that want full snippets use `max-snippet:-1` (unlimited). Without explicit controls, Google chooses snippet length algorithmically; for high traffic news sites, this matters for ad revenue and click through.

**AI Overviews opt out requires the right directives.** As of 2026, `max-snippet:0` plus `nosnippet` are the documented controls for opting out of AI generated summaries in Google Search. Sites that want to remain searchable but not appear in AI Overviews use these directives selectively.

**Time limited content needs `unavailable_after`.** Event pages, sale pages, expired offers: setting `unavailable_after:` with an RFC 850 date causes Google to drop the URL from the index roughly 24 hours after the date passes. Better than leaving expired content indexed and ranking for outdated information.

**`indexifembedded` solves the embedded content paradox.** Podcast hosts, video platforms, and content syndicators want their host page not indexed but the embedded version (in third party iframes) to be indexable. Without `indexifembedded`, `noindex` on the host page prevents both. With it, the host page stays out of the index but the embedded version is included where the iframe parent is indexable.

**Bing has different syntax requirements.** Bingbot is generally compliant with Googlebot syntax but has minor differences (some directives Google supports, Bing does not, and vice versa). Sites that care about Bing visibility document the differences.

**Cost of getting it wrong.** Misconfigured meta robots produces silent SEO damage. Real examples:

* Bubbles client site had `<meta name="robots" content="noindex">` accidentally left on the homepage after a development push. Site dropped from Google in 5 days. Discovered after 3 weeks of declining traffic. Removed the tag; recovery took 4 weeks.
* Search results pages `/search?q=*` returned 200 with thin content and no noindex. Indexed for thousands of random query strings. GSC showed 12,000 "soft 404" entries before action was taken.
* Robots.txt blocked `/admin/*` AND admin pages had `<meta name="robots" content="noindex">` in the HTML. Google had previously indexed several admin URLs from external links. The noindex never took effect because Google could not crawl to see it. URLs persisted in index for 18 months.
* News site set `max-snippet:0` site wide to "protect content". Lost click through because Google search results showed no preview text. Conversions dropped 30%.
* Site set `<meta name="googlebot" content="noindex">` for staging environment. Forgot to remove it before production launch. Production live but invisible to Google for 11 days.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The directives Joseph listed plus essential context get the full treatment:

1. **What it does**: the canonical Google/Bing definition plus practical implication.
2. **When to use it**: which scenarios warrant this specific directive.
3. **How to emit it in HTML**: paste ready snippets.
4. **How crawlers react**: indexing pipeline behavior per major crawler.
5. **How to verify**: curl plus grep patterns, browser DevTools.
6. **Common misuse and how to fix**: typical wrong choices and replacements.

Sections 9 to 13 are deep dives on cross cutting concerns: user agent precedence rules, the robots.txt plus noindex paradox, AI directive landscape, the Bubbles standard patterns per page type, and the thin content cleanup HTML side (complement to framework-http-4xx-status-codes.md Section 17 which covers the 410 server response).

---

## 4. THE META ROBOTS MENTAL MODEL (READ THIS FIRST)

A crawler fetches a URL. The HTTP response is 200 with HTML. The crawler parses the HTML and looks for `<meta name="robots">` and related tags in the `<head>`. The crawler then decides:

```
Crawler fetched URL, got 200 with HTML.
        |
        v
==================== STEP 1: PARSE META ROBOTS ====================
        |
        v
Find all <meta name="robots|googlebot|bingbot|...">
   |
   |---> Generic <meta name="robots"> applies to all crawlers
   |
   |---> Crawler specific (e.g., <meta name="googlebot">) overrides generic FOR THAT CRAWLER
   |
   |---> Multiple tags merge; most restrictive directive wins
        |
        v
==================== STEP 2: APPLY DIRECTIVES ====================
        |
        v
Does noindex apply?
   YES to URL excluded from index (but body still parsed if needed for links)
   NO  to URL considered for indexing
        |
        v
Does nofollow apply?
   YES to Links on the page not followed for crawl discovery
   NO  to Links followed normally
        |
        v
Snippet generation rules:
   nosnippet              to No snippet shown
   max-snippet:N          to Snippet capped at N characters
   max-image-preview:X    to Image preview size limited
   max-video-preview:N    to Video preview capped at N seconds
        |
        v
Other behaviors:
   noarchive              to No "cached" link in results
   notranslate            to No Google Translate offer
   noimageindex           to Images on page not indexed (but page still indexed)
   unavailable_after:DATE to Drop from index after this date
   indexifembedded        to Index when embedded even if noindex
        |
        v
==================== STEP 3: RESULT ====================
        |
        v
URL processed per the combined directives.
```

Six rules govern the system:

1. **Default is `index, follow`.** Emit directives only when you want to CHANGE behavior. `<meta name="robots" content="index, follow">` is noise.
2. **Most restrictive wins.** When directives conflict, the more restrictive choice applies.
3. **`noindex` requires crawling.** A URL blocked in robots.txt cannot be deindexed via noindex (the crawler never sees the tag).
4. **Meta robots is for HTML only.** For PDFs, images, JSON APIs use X-Robots-Tag header.
5. **User agent specific tags override generic.** `<meta name="googlebot" content="noindex">` overrides `<meta name="robots" content="index">` for Googlebot.
6. **Case insensitive but be consistent.** Lowercase is conventional and matches most documentation examples.

A correctly configured site emits meta robots only where defaults need to change, uses crawler specific tags only when the directive should differ per crawler, never combines robots.txt block with noindex, and emits the tags via server side rendering (not client side JavaScript).

---

## 5. META ROBOTS VS X-ROBOTS-TAG (HTML VS HTTP DECISION)

The same directives can be sent two ways:

* **HTML `<meta name="robots">`**: embedded in the page `<head>`. Crawler sees after fetching the page.
* **HTTP `X-Robots-Tag` response header**: sent in the HTTP response (covered in framework-http-seo-headers.md). Crawler sees before parsing the body.

When to use each:

| Scenario | Use this |
|---|---|
| HTML page where you can edit the template | `<meta name="robots">` (cleaner, version controlled with content) |
| Non HTML resource (PDF, image, video, JSON) | `X-Robots-Tag` (can't add meta tags to non HTML) |
| Need to set directive at the proxy layer (nginx) | `X-Robots-Tag` (HTML edit not required) |
| Need to set directive per file extension or path pattern | `X-Robots-Tag` (set via nginx config) |
| Per page logic in the application | `<meta name="robots">` (render per request) |
| Cross cutting policy (all admin pages noindex) | `X-Robots-Tag` in nginx location block |

**Combination behavior**: when both `<meta>` and `X-Robots-Tag` are present, they combine (most restrictive wins). Setting `index` in HTML cannot override `noindex` from the HTTP header.

For Bubbles client sites: the convention is `<meta name="robots">` for individual page level controls and `X-Robots-Tag` for path level policy (e.g., `/api/*` gets `X-Robots-Tag: noindex` at the nginx layer regardless of what the upstream returns).

---

## 6. THE FIVE META TAG NAMES (ROBOTS, GOOGLEBOT, BINGBOT, GOOGLEBOT-NEWS, PLUS AI VARIANTS)

### 6.1 `<meta name="robots">` (Generic, All Crawlers)

The default. Applies to every crawler that recognizes the robots meta convention (Googlebot, Bingbot, DuckDuckBot, Applebot, AmazonBot, etc).

```html
<meta name="robots" content="noindex, follow">
```

When you do not need crawler specific behavior, use this. Most Bubbles client sites have only this generic form.

### 6.2 `<meta name="googlebot">` (Googlebot Only)

Applies only to Googlebot (and inherits to Googlebot-Image, Googlebot-Video, Googlebot-News by default). Overrides the generic `robots` tag for Googlebot.

```html
<meta name="robots" content="index, follow">
<meta name="googlebot" content="noindex">
```

In this example, all crawlers except Googlebot see "index, follow". Googlebot sees "noindex". Useful for "I want Bing to index but not Google" scenarios (rare in practice).

### 6.3 `<meta name="bingbot">` (Bingbot Only)

The Bing equivalent. Bingbot supports most Google directives plus a few unique ones (less commonly used).

```html
<meta name="bingbot" content="noarchive">
```

Bing supports `nocache` as a synonym for `noarchive`. Google does not.

### 6.4 `<meta name="googlebot-news">` (Google News Only)

Applies specifically to Googlebot when crawling for Google News inclusion. Useful for content publishers who want a page to be in regular Google Search but NOT in Google News (or vice versa).

```html
<meta name="robots" content="index, follow">
<meta name="googlebot-news" content="noindex">
```

The page indexes normally in Google Search but is excluded from Google News.

### 6.5 AI Crawler Specific Meta Tags (Non Standard)

Various proposals exist for AI crawler specific directives:

```html
<meta name="claude" content="noindex">
<meta name="chatgpt" content="noindex">
<meta name="perplexitybot" content="noindex">
```

**These are not officially documented by Anthropic, OpenAI, or Perplexity.** Some sites emit them as a best effort signal but their behavior is undefined. The standardized approach is the `noai` and `noimageai` directives via robots.txt and the User-Agent based blocking covered in framework-http-request-headers.md.

For Bubbles: do not rely on AI specific meta tags. Use robots.txt for crawler level opt out and HTTP level UA detection for actual access control.

---

## 7. THE COMPLETE DIRECTIVE LIST

### 7.1 `index` / `noindex`

**`index`** (default): the URL may appear in search results.

**`noindex`**: the URL should not appear in search results.

```html
<meta name="robots" content="noindex">
```

Per Google: when the crawler sees `noindex`, it processes the page (extracts links, etc) but does not add it to the searchable index. Existing index entries are removed on subsequent crawl.

**`index` is the default.** Emitting it does nothing useful. Per John Mueller (Google): `<meta name="robots" content="index, follow">` is "a waste of HTML space and is ignored by Googlebot."

**When to use `noindex`:**

* Login walls, account pages.
* Search results pages.
* Filter parameter URLs.
* Print views.
* Thin content that should remain accessible.
* Staging and demo subdomains.
* Internal tools.

**When NOT to use `noindex`:**

* Pages with substantive content you want ranked.
* Pages you want to remove permanently (use 410 Gone instead, see framework-http-4xx-status-codes.md).

### 7.2 `follow` / `nofollow`

**`follow`** (default): crawlers follow links on the page for further discovery.

**`nofollow`**: crawlers do not follow links on the page. Treated as "do not pass authority to linked pages".

```html
<meta name="robots" content="noindex, nofollow">
```

**`follow` is the default.** Like `index`, emitting it is unnecessary.

**When to use `nofollow`:**

* User generated content where you cannot vet outbound links.
* Paid content where outbound links should not pass authority.
* Internal admin pages where outbound links to user data should not be discovered.

**The historical evolution:** `rel="nofollow"` on individual `<a>` tags is more common than page level `nofollow` in modern SEO. Page level nofollow is the nuclear option.

### 7.3 `none` And `all`

**`none`**: equivalent to `noindex, nofollow`. Shorthand for "do nothing with this page".

**`all`**: equivalent to `index, follow`. Default behavior; not useful to emit.

```html
<meta name="robots" content="none">

<!-- Equivalent to: -->
<meta name="robots" content="noindex, nofollow">
```

Use `none` when you want both behaviors but prefer the shorter form.

### 7.4 `noarchive`

Prevents Google from showing a "cached" link in search results.

```html
<meta name="robots" content="noarchive">
```

Historically critical for content that changed frequently (news, stock prices). In 2024 Google removed the cached link feature from standard search results, making `noarchive` largely irrelevant. The directive is still honored where applicable.

Bing supports `nocache` as a synonym; Google does not.

### 7.5 `nosnippet`

Prevents Google from showing a text or video preview snippet in search results.

```html
<meta name="robots" content="nosnippet">
```

The page appears in results but with no preview text underneath the title. Users see only the title and URL.

**Important 2026 implication:** `nosnippet` is one of the controls for opting out of Google AI Overviews. Pages with `nosnippet` are not used to generate AI generated summary content. For sites that want to remain searchable but not appear in AI generated answers, this directive is the standard signal.

### 7.6 `max-snippet:N`

Limits the snippet to N characters.

```html
<meta name="robots" content="max-snippet:160">
<meta name="robots" content="max-snippet:-1">     <!-- unlimited -->
<meta name="robots" content="max-snippet:0">      <!-- no snippet -->
```

| Value | Effect |
|---|---|
| `max-snippet:-1` | Unlimited snippet length (recommended for editorial content) |
| `max-snippet:0` | No snippet shown (equivalent to `nosnippet`) |
| `max-snippet:160` | Cap at 160 characters |
| Any positive integer | Cap at that character count |

**Bubbles recommended pattern** for editorial sites: `max-snippet:-1, max-image-preview:large, max-video-preview:-1` (the "Show My Content" pattern).

### 7.7 `max-image-preview:none|standard|large`

Controls the size of image previews in search results.

```html
<meta name="robots" content="max-image-preview:large">
```

| Value | Effect |
|---|---|
| `max-image-preview:none` | No image preview |
| `max-image-preview:standard` | Default size (small thumbnail) |
| `max-image-preview:large` | Large image preview (more prominent in results) |

For visual content sites (image galleries, real estate, products), `max-image-preview:large` is the recommended default to maximize visual prominence in search results.

### 7.8 `max-video-preview:N`

Controls the maximum video preview duration in seconds.

```html
<meta name="robots" content="max-video-preview:30">
<meta name="robots" content="max-video-preview:-1">    <!-- unlimited -->
<meta name="robots" content="max-video-preview:0">     <!-- no preview -->
```

| Value | Effect |
|---|---|
| `max-video-preview:-1` | Unlimited preview duration |
| `max-video-preview:0` | No video preview |
| `max-video-preview:N` | Preview capped at N seconds |

For video content sites: use `max-video-preview:-1` to allow Google maximum preview discretion.

### 7.9 `notranslate`

Prevents Google from offering translation of the page in search results.

```html
<meta name="robots" content="notranslate">
```

Useful for content that is:

* Already in a target language (translation would be lossy).
* Brand sensitive (translation could damage brand voice).
* Code, technical content, or anything with specialized vocabulary.

### 7.10 `noimageindex`

Prevents images on this page from being indexed in Google Image Search. The page itself still indexes normally.

```html
<meta name="robots" content="noimageindex">
```

Different from `max-image-preview:none` which controls preview display in regular search results.

**The distinction:**

* `noimageindex`: images do not appear in Google Image Search at all.
* `max-image-preview:none`: images may still be in Image Search but no preview shown in regular search results.

For Bubbles client sites with proprietary product images: `noimageindex` prevents competitors from finding and using your product photography via Image Search.

### 7.11 `unavailable_after:DATE`

Tells Google to drop the URL from the index after a specific date. Useful for time limited content.

```html
<meta name="robots" content="unavailable_after: 25 May 2026 12:00:00 PST">
```

The date format is RFC 850 (similar to HTTP date but with explicit timezone).

```
unavailable_after: 25 May 2026 12:00:00 PST
unavailable_after: Sunday, 25-May-26 12:00:00 GMT
unavailable_after: 2026-05-25T12:00:00-08:00
```

After the date passes, Google stops showing the URL in search results within approximately 24 hours.

**Use cases:**

* Event pages: `unavailable_after: <day after event>`.
* Sale pages: `unavailable_after: <sale end date>`.
* Job postings: `unavailable_after: <position filled date>`.
* News articles with time bound relevance.

**Note:** the URL itself does not 404 after the date; it just drops from the index. The content remains accessible to direct visitors. For full removal, combine with 410 Gone after the date.

### 7.12 `indexifembedded`

Google specific (since 2022). Tells Google that even though this page has `noindex`, the content should be indexable when embedded in third party pages via iframe or similar.

```html
<meta name="googlebot" content="noindex, indexifembedded">
```

**The use case:** podcast hosts and video platforms have URLs like:

* `https://podcast.host/play?episode=12345` (the host's own player page)
* `https://news-site.com/article` (a third party news site embedding the podcast player)

The host wants the news site to be able to embed the podcast (and have that content discoverable via the news site URL) but does not want the standalone player page indexed independently.

Without `indexifembedded`: `noindex` on the host page prevents both standalone indexing AND iframe content indexing.

With `indexifembedded`: the host page stays out of the index, but when the news site embeds the player, the embedded content can be indexed as part of the news site URL.

**Google specific.** Bing does not support `indexifembedded`.

### 7.13 Less Common Directives

* **`nositelinkssearchbox`**: prevents Google from showing the sitelinks search box in results. Rare; only relevant for major brands.
* **`indexifembedded`**: covered above.
* **`unavailable_after`**: covered above.

---

## 8. THE DATA-NOSNIPPET HTML ATTRIBUTE (PARTIAL PAGE CONTROL)

Beyond meta tags, Google supports the `data-nosnippet` HTML attribute on individual elements. This controls what parts of a page can appear in snippets and AI generated content, with sub page granularity.

```html
<p>This content can appear in search snippets.</p>

<div data-nosnippet>
This content will NOT appear in search snippets or AI Overviews.
</div>

<p>This content can appear in snippets again.</p>
```

The `data-nosnippet` attribute can be applied to any HTML element. Content within marked elements is excluded from:

* Search result snippet text.
* AI Overview generated answers.
* Google Discover preview text.

### 8.1 When To Use data-nosnippet

* **Brand voice protection**: keep marketing copy out of snippets but allow factual content.
* **Premium content gating**: mark gated paragraphs as `data-nosnippet` so they cannot be revealed via search.
* **Pricing or terms protection**: prevent specific pricing details from appearing in AI generated summaries.
* **Editorial sensitive content**: parts of an article that should not be quoted out of context.

### 8.2 The Combination With max-snippet

`data-nosnippet` element level control plus `max-snippet:N` page level control work together. The combination:

```html
<meta name="robots" content="max-snippet:-1, max-image-preview:large, max-video-preview:-1">
...
<body>
    <h1>Article title</h1>
    <p>First paragraph, summary appropriate.</p>

    <div data-nosnippet>
    Sensitive details that should not appear in snippets or AI summaries.
    </div>

    <p>Rest of the article continues...</p>
</body>
```

Google is given freedom to generate large snippets (page level), but specific elements are excluded from any snippet (element level).

### 8.3 Verification

```bash
# Check that data-nosnippet attributes are present where intended
curl -s https://example.com/some-page | grep -o 'data-nosnippet[^>]*' | head -10
```

---

## 9. USER AGENT PRECEDENCE AND CONFLICT RESOLUTION

When a page has multiple meta robots tags or conflicting directives, the resolution follows specific rules.

### 9.1 The Generic vs Specific Rule

Crawler specific tags override generic for that crawler:

```html
<meta name="robots" content="index, follow">
<meta name="googlebot" content="noindex">
```

* Bingbot, Applebot, DuckDuckBot: see "index, follow" (use the generic tag).
* Googlebot: sees "noindex" (uses the specific tag, ignores generic).

### 9.2 The Most Restrictive Wins Rule

When multiple directives conflict, the most restrictive applies:

```html
<meta name="robots" content="index">
<meta name="robots" content="noindex">
```

Result: `noindex` wins (more restrictive than index).

```html
<meta name="robots" content="max-snippet:160">
<meta name="robots" content="nosnippet">
```

Result: `nosnippet` wins (more restrictive: no snippet at all overrides "snippet up to 160").

### 9.3 The Combined Directives Rule

When directives are not conflicting, they combine:

```html
<meta name="robots" content="noindex">
<meta name="robots" content="nosnippet">
```

Result: both apply. URL is not indexed AND if it somehow appears, no snippet.

### 9.4 The Header Plus Meta Combination

`X-Robots-Tag` HTTP header and `<meta name="robots">` HTML tag combine the same way: most restrictive wins.

```
HTTP/2 200 OK
X-Robots-Tag: index

<html>
<head>
<meta name="robots" content="noindex">
</head>
```

Result: `noindex` wins.

### 9.5 The Bubbles Precedence Pattern

For Bubbles client sites, the canonical pattern:

* **HTTP header (nginx)**: path level policy. `/api/*` location adds `X-Robots-Tag: noindex` regardless of upstream.
* **HTML meta (FastAPI/template)**: page level overrides for specific URLs.

This means the HTTP header is a baseline floor (always at least this restrictive); the HTML meta can add more restrictions but cannot remove the header's restrictions.

---

## 10. THE ROBOTS.TXT PLUS NOINDEX PARADOX

The single most misunderstood interaction in technical SEO. Per Google's explicit documentation:

> "If your page is blocked with a robots.txt file, its URL can still appear in search results, but the search result will not have a description... To properly exclude your page from search results, unblock your page in robots.txt and use noindex."

### 10.1 The Mechanism

The sequence:

1. Site adds `Disallow: /admin/*` to robots.txt.
2. Site adds `<meta name="robots" content="noindex">` to admin pages.
3. Operator believes admin pages are deindexed.

**Reality:** robots.txt blocks crawling. Googlebot never fetches the admin pages. Therefore Google never sees the noindex meta tag. If anyone (even one external link) points to an admin URL, Google indexes the URL (without content, but indexed).

### 10.2 The Correct Sequence

To deindex a URL via noindex:

1. **Allow crawling in robots.txt** (remove Disallow or use Allow override).
2. **Add `<meta name="robots" content="noindex">` to the page** (or X-Robots-Tag).
3. **Wait** for Google to recrawl. Recrawl happens on Google's schedule; typically days to weeks.
4. **Verify** with Google Search Console URL Inspection tool.
5. **After deindexing**: you may then add to robots.txt to prevent future crawl waste (but the noindex must remain visible to crawlers that recrawl).

### 10.3 The Decision Matrix

| Goal | robots.txt | meta robots |
|---|---|---|
| Allow crawling and indexing | (no rule needed) | (no tag needed, default) |
| Allow crawling but prevent indexing | Allow | `noindex` |
| Prevent crawling entirely (URL may still show in results) | `Disallow` | (irrelevant) |
| **WRONG: try to deindex via robots.txt** | `Disallow` | doesn't work |

### 10.4 The Cleanup Pattern For Currently Indexed But Should Not Be

If URLs are currently in the index but should be removed:

```bash
# Step 1: REMOVE the robots.txt disallow temporarily
# Edit /var/www/sites/example.com/robots.txt
# Remove or comment out: Disallow: /admin/*

# Step 2: Verify noindex is in place on the affected pages
curl -s https://example.com/admin/ | grep -i 'meta name="robots"'

# Step 3: Wait for Google to recrawl (or accelerate via GSC URL Removals tool)

# Step 4: After deindexing verified in GSC, re-add Disallow if desired
# Edit /var/www/sites/example.com/robots.txt
# Re-add: Disallow: /admin/*
```

This is the painful but correct dance.

---

## 11. AI SPECIFIC DIRECTIVES (NOAI, NOIMAGEAI, NON STANDARD)

The AI crawler landscape (covered in framework-http-request-headers.md Section 5.2) has produced various proposals for opting content out of AI training and retrieval. Most are non standard.

### 11.1 The Officially Documented Approaches

For Google's AI products (Gemini, AI Overviews):

* **Google-Extended in robots.txt** (NOT a meta tag): `User-agent: Google-Extended\nDisallow: /` opts out of Gemini training while keeping Googlebot search access.
* **`nosnippet` meta tag**: pages with nosnippet are excluded from AI Overviews.
* **`max-snippet:0` meta tag**: same effect.

For OpenAI:

* **GPTBot user agent in robots.txt**: `User-agent: GPTBot\nDisallow: /` blocks training crawler.
* **OAI-SearchBot user agent**: `User-agent: OAI-SearchBot\nDisallow: /` blocks search retrieval.
* **ChatGPT-User user agent**: `User-agent: ChatGPT-User\nDisallow: /` blocks per query fetches.

For Anthropic:

* **ClaudeBot user agent in robots.txt**: `User-agent: ClaudeBot\nDisallow: /` blocks training crawler.
* **Claude-SearchBot, Claude-User**: per crawler blocks via robots.txt.

### 11.2 The Non Standard Proposals

Various meta tag proposals have circulated:

```html
<!-- All non standard; honored by some, ignored by most -->
<meta name="robots" content="noai">
<meta name="robots" content="noimageai">
<meta name="claudebot" content="noindex">
<meta name="gptbot" content="noindex">
```

**These directives are not standardized.** Anthropic, OpenAI, and Perplexity do not officially document them. Some sites emit them as a best effort signal; their effect is undefined.

### 11.3 The Bubbles Approach

For Bubbles client sites that need AI opt out:

1. **Use robots.txt for crawler level blocks.** The standard, documented mechanism.
2. **Do not rely on non standard meta tags.** Their effect is unpredictable.
3. **For partial opt out (snippets but not training)**: combine `Google-Extended` Disallow with `Googlebot` Allow.
4. **For aggressive opt out**: combine robots.txt blocks with HTTP layer UA blocking (per framework-http-request-headers.md Section 6).

---

## 12. THE BUBBLES STANDARD PATTERNS (PER PAGE TYPE)

The canonical meta robots configuration for each page type Joseph encounters.

### 12.1 Public Marketing Pages (Index Normally)

```html
<head>
    <!-- No meta robots needed; default is index, follow -->
    <!-- Optionally include for snippet control: -->
    <meta name="robots" content="max-snippet:-1, max-image-preview:large, max-video-preview:-1">
</head>
```

Per John Mueller, `index, follow` is unnecessary. The snippet control directives are the only useful additions for editorial content.

### 12.2 Login And Account Pages

```html
<head>
    <meta name="robots" content="noindex, follow">
</head>
```

`noindex` prevents login pages from cluttering search results. `follow` allows authority to flow through any internal links on the page (still useful for the site graph).

### 12.3 Search Results Pages

```html
<head>
    <meta name="robots" content="noindex, follow">
</head>
```

Search results pages should never be indexed (they're soft 404 risk and crawl budget waste). `follow` preserves the link graph through them.

### 12.4 Filter And Sort Parameter URLs

```html
<head>
    <meta name="robots" content="noindex, follow">
    <link rel="canonical" href="https://example.com/products">
</head>
```

The canonical tag points to the unfiltered base URL. `noindex` prevents the parameter URL from being indexed.

### 12.5 Pagination Beyond First Page

```html
<!-- On /blog/page/2, /blog/page/3, etc -->
<head>
    <meta name="robots" content="noindex, follow">
    <link rel="canonical" href="https://example.com/blog/page/2">
</head>
```

Modern SEO advice on pagination evolved. Older approach: rel="prev/next". Current approach: noindex deeper pages, keep first page indexed. The canonical points to the page itself (not the first page) because that is the actual URL.

### 12.6 Thank You / Confirmation Pages

```html
<head>
    <meta name="robots" content="noindex, nofollow">
</head>
```

Thank you pages have no value in search. `nofollow` prevents authority from flowing out through any links on the page (e.g., "share on social" links).

### 12.7 Admin And Internal Tools

```html
<head>
    <meta name="robots" content="noindex, nofollow">
</head>
```

Admin pages should never be indexed and should not pass authority. The HTTP layer also adds `X-Robots-Tag: noindex, nofollow` for defense in depth.

### 12.8 Staging And Demo Subdomains

```html
<head>
    <meta name="robots" content="noindex, nofollow">
</head>
```

Plus robots.txt level block:

```
# /var/www/sites/staging.example.com/robots.txt
User-agent: *
Disallow: /
```

Belt and suspenders: meta robots AND robots.txt AND optionally HTTP basic auth.

### 12.9 Editorial Content (Maximize Discoverability)

```html
<head>
    <meta name="robots" content="max-snippet:-1, max-image-preview:large, max-video-preview:-1">
</head>
```

Allows Google maximum discretion on snippet and preview generation. Recommended for news, blogs, articles.

### 12.10 Time Limited Content

```html
<head>
    <meta name="robots" content="unavailable_after: 25 May 2026 12:00:00 PST">
</head>
```

Indexes normally before the date, drops from index after. Useful for event pages, sales, time bound offers.

### 12.11 AI Snippet Opt Out

```html
<head>
    <meta name="robots" content="nosnippet">
</head>
```

Prevents Google AI Overviews from using the page content. Loses snippet visibility in regular search but maintains indexing.

### 12.12 Embed Compatible Content (Podcast/Video Hosts)

```html
<head>
    <meta name="googlebot" content="noindex, indexifembedded">
</head>
```

Host player page does not index standalone, but the player content indexes when embedded in third party pages.

### 12.13 Internal Search Reference Pages

```html
<head>
    <meta name="robots" content="noimageindex">
</head>
```

Page indexes normally; product images are not indexed in Google Image Search (prevents image scraping for competitors).

---

## 13. THE BUBBLES THIN CONTENT CLEANUP PATTERN (JOSEPH'S 5,800 PAGE WORK, HTML SIDE)

The complement to framework-http-4xx-status-codes.md Section 17 (which covers the 410 server response). This section covers the HTML side: pages that should remain accessible to users but should not be indexed.

### 13.1 The Three Bucket Reminder

From the 4xx framework Section 17:

* **Bucket 1: Keep + improve.** Substantive content added. Pages remain indexable.
* **Bucket 2: noindex, follow.** Pages remain accessible to users; deindexed from Google.
* **Bucket 3: 410 Gone.** Pages removed entirely.

This section covers Bucket 2 implementation in HTML.

### 13.2 The Template Pattern

For dynamic location pages (the Bubbles services template):

```html
<!-- /templates/services/{city}/index.html -->
<head>
    <title>Web Development Services in {{ city }} | ThatDeveloperGuy</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    {% if city in NOINDEX_CITIES %}
    <meta name="robots" content="noindex, follow">
    {% endif %}

    <link rel="canonical" href="https://thatdeveloperguy.com/services/{{ city }}/">
</head>
```

Where `NOINDEX_CITIES` is the list of cities that should remain accessible but not indexed.

### 13.3 The FastAPI Sidecar Pattern

For server side rendered pages:

```python
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse

NOINDEX_CITIES = {
    # Far flung cities outside the NWA/SWMO core market
    "detroit", "chicago", "denver", "portland", "seattle",
    # ... full list from the original cleanup ...
}

@app.get("/services/{city}/")
async def services_page(city: str, request: Request):
    is_noindex = city.lower() in NOINDEX_CITIES

    html = render_template(
        "services_city.html",
        city=city,
        is_noindex=is_noindex,
    )

    headers = {}
    if is_noindex:
        # Belt and suspenders: HTTP header AND HTML meta
        headers["X-Robots-Tag"] = "noindex, follow"

    return HTMLResponse(content=html, headers=headers)
```

### 13.4 The Verification

```bash
# Verify noindex pages emit the meta tag
for city in detroit chicago denver; do
    HAS_NOINDEX=$(curl -s "https://thatdeveloperguy.com/services/$city/" \
                  | grep -c 'meta name="robots" content="noindex')
    echo "$city: noindex meta tag count = $HAS_NOINDEX"
done
# Expected: 1 for each

# Verify HTTP header too
for city in detroit chicago denver; do
    HEADER=$(curl -sI "https://thatdeveloperguy.com/services/$city/" \
             | grep -i x-robots-tag)
    echo "$city: $HEADER"
done
# Expected: x-robots-tag: noindex, follow

# Verify core cities are NOT noindexed
for city in cassville fayetteville bentonville; do
    HAS_NOINDEX=$(curl -s "https://thatdeveloperguy.com/services/$city/" \
                  | grep -c 'meta name="robots" content="noindex')
    echo "$city: noindex meta tag count = $HAS_NOINDEX"
done
# Expected: 0 for each (core cities stay indexable)
```

### 13.5 The Recovery Timeline (Same As 4xx Framework)

Per framework-http-4xx-status-codes.md Section 17.5:

* **Week 1**: deployment. Googlebot starts encountering noindex.
* **Month 1 to 2**: noindex URLs gradually deindex.
* **Month 3**: most URLs in this tier are out of the index.
* **Month 4 to 6**: ranking improvement on the kept (substantive) pages.

The HTML side and HTTP side work together; either alone would work but the combination is fastest.

---

## 14. HOW MAJOR CRAWLERS REACT TO EACH DIRECTIVE

Quick reference table.

| Directive | Googlebot | Bingbot | ClaudeBot | GPTBot | PerplexityBot |
|---|---|---|---|---|---|
| `noindex` | Honored (removes from index) | Honored | Honored (caches negative) | Honored | Honored |
| `nofollow` | Honored (skips link discovery) | Honored | Likely honored | Likely honored | Likely honored |
| `noarchive` | Honored (cached link removed) | Honored | N/A | N/A | N/A |
| `nosnippet` | Honored; also blocks AI Overviews | Honored | Likely honored for snippet display | Likely honored | Likely honored |
| `max-snippet:N` | Honored | Honored | Likely ignored | Likely ignored | Likely ignored |
| `max-image-preview` | Honored | Honored | N/A | N/A | N/A |
| `max-video-preview:N` | Honored | Honored | N/A | N/A | N/A |
| `notranslate` | Honored | Honored | N/A | N/A | N/A |
| `noimageindex` | Honored | Honored | N/A | N/A | N/A |
| `unavailable_after` | Honored | Likely ignored | Likely ignored | Likely ignored | Likely ignored |
| `indexifembedded` | Honored (Google only) | Not supported | Not documented | Not documented | Not documented |
| `data-nosnippet` attribute | Honored; blocks AI Overviews | Likely honored | Behavior undocumented | Likely honored | Likely honored |

**Key implications for Bubbles:**

* For SEO baseline: `noindex` is universally honored.
* For AI opt out: `nosnippet` and `max-snippet:0` are the strongest documented controls.
* For element level control: `data-nosnippet` works for Google; behavior elsewhere varies.

---

## 15. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 15.1 Standard editorial page

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="robots" content="max-snippet:-1, max-image-preview:large, max-video-preview:-1">

    <title>Article Title | Site Name</title>
    <meta name="description" content="...">
    <link rel="canonical" href="https://example.com/articles/article-slug/">
</head>
```

### 15.2 Login page

```html
<head>
    <meta name="robots" content="noindex, follow">
    <title>Sign In | Site Name</title>
</head>
```

### 15.3 Search results page

```html
<head>
    <meta name="robots" content="noindex, follow">
    <link rel="canonical" href="https://example.com/search">
</head>
```

The canonical points to the base search URL without query parameters.

### 15.4 Filter or sort URL

```html
<head>
    <meta name="robots" content="noindex, follow">
    <link rel="canonical" href="https://example.com/products">
</head>
```

### 15.5 Pagination (page 2+)

```html
<head>
    <meta name="robots" content="noindex, follow">
    <link rel="canonical" href="https://example.com/blog/page/2/">
</head>
```

### 15.6 Thank you / confirmation page

```html
<head>
    <meta name="robots" content="noindex, nofollow">
</head>
```

### 15.7 Admin and internal tools

```html
<head>
    <meta name="robots" content="noindex, nofollow">
</head>
```

Plus at the nginx layer (per framework-http-seo-headers.md):

```nginx
location /admin/ {
    add_header X-Robots-Tag "noindex, nofollow" always;
    # ...
}
```

### 15.8 Time limited event page

```html
<head>
    <meta name="robots" content="unavailable_after: 31 Dec 2026 23:59:59 PST, max-snippet:-1, max-image-preview:large">
    <title>NWA Tech Conference 2026 | December 1-3</title>
</head>
```

After Dec 31, Google drops the URL from the index. Until then, full snippets and large images are allowed.

### 15.9 AI opt out (keep searchable but no AI Overviews)

```html
<head>
    <meta name="robots" content="nosnippet">
    <title>Article Title | Site Name</title>
</head>
```

Plus consider data-nosnippet on specific paragraphs:

```html
<p>Editorial content that can be summarized.</p>

<div data-nosnippet>
Specific competitive intelligence or copyrighted material that should not be in AI summaries.
</div>
```

### 15.10 Embed compatible content (podcast host)

```html
<head>
    <meta name="googlebot" content="noindex, indexifembedded">
    <meta name="robots" content="noindex">
</head>
```

Standalone player page is not indexed; embedded player content is indexable when embedded in third party iframes.

### 15.11 Staging site (full block)

```html
<head>
    <meta name="robots" content="noindex, nofollow">
</head>
```

Plus robots.txt:

```
# /var/www/sites/staging.example.com/robots.txt
User-agent: *
Disallow: /
```

Plus HTTP Basic Auth at nginx layer:

```nginx
location / {
    auth_basic "Staging";
    auth_basic_user_file /etc/nginx/.htpasswd-staging;
    # ...
}
```

Triple defense.

### 15.12 The Bubbles thin content noindex tier (Joseph's pattern)

```html
{% if city in NOINDEX_CITIES %}
<meta name="robots" content="noindex, follow">
{% else %}
<meta name="robots" content="max-snippet:-1, max-image-preview:large, max-video-preview:-1">
{% endif %}
```

Pages outside the core market: noindex but still followable. Core market pages: maximum discoverability.

### 15.13 Print version

```html
<head>
    <meta name="robots" content="noindex">
    <link rel="canonical" href="https://example.com/article/normal-version/">
</head>
```

Print versions duplicate the main article. noindex prevents duplicate content; canonical points to the primary.

### 15.14 Bingbot specific differing behavior

```html
<head>
    <meta name="robots" content="index, follow">
    <meta name="bingbot" content="noarchive">
</head>
```

All crawlers can index and follow. Bing specifically does not show a cached link.

### 15.15 Google News opt out (regular search OK)

```html
<head>
    <meta name="googlebot-news" content="noindex">
    <!-- No other robots tag: defaults to index, follow for regular search -->
</head>
```

Indexes in regular Google Search but excluded from Google News.

---

## 16. BUBBLES HTML REFERENCE BLOCK (PASTE READY)

The canonical Bubbles meta robots configuration patterns for a FastAPI + Jinja2 template.

```python
# /opt/bubbles/templates/_meta_robots.html
# Reusable Jinja2 macro for emitting meta robots tags

{% macro meta_robots(directive_set='default') %}
    {% if directive_set == 'default' %}
        {# Editorial pages: maximum discoverability, no need to emit index/follow #}
        <meta name="robots" content="max-snippet:-1, max-image-preview:large, max-video-preview:-1">
    {% elif directive_set == 'noindex_follow' %}
        {# Search results, filter URLs, pagination beyond first, demo subdomains #}
        <meta name="robots" content="noindex, follow">
    {% elif directive_set == 'noindex_nofollow' %}
        {# Admin, thank-you, internal tools #}
        <meta name="robots" content="noindex, nofollow">
    {% elif directive_set == 'login' %}
        {# Login pages, account pages #}
        <meta name="robots" content="noindex, follow">
    {% elif directive_set == 'noimageindex' %}
        {# Pages where images are proprietary and should not be in Image Search #}
        <meta name="robots" content="max-snippet:-1, max-image-preview:standard, noimageindex">
    {% elif directive_set == 'embed_only' %}
        {# Podcast/video player pages #}
        <meta name="googlebot" content="noindex, indexifembedded">
        <meta name="robots" content="noindex">
    {% elif directive_set == 'no_ai' %}
        {# Searchable but not in AI Overviews #}
        <meta name="robots" content="nosnippet">
    {% endif %}
{% endmacro %}
```

Usage in templates:

```html
{# /opt/bubbles/templates/services_city.html #}
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    {% from '_meta_robots.html' import meta_robots %}

    {% if city in NOINDEX_CITIES %}
        {{ meta_robots('noindex_follow') }}
    {% else %}
        {{ meta_robots('default') }}
    {% endif %}

    <title>Web Development Services in {{ city }} | ThatDeveloperGuy</title>
    <link rel="canonical" href="https://thatdeveloperguy.com/services/{{ city|lower }}/">
</head>
<body>
    <!-- ... content ... -->
</body>
</html>
```

Site wide audit verification:

```bash
# Verify expected meta robots presence per URL pattern
for path in / /blog/ /services/cassville/ /services/detroit/ /admin/ /login /search; do
    echo "=== $path ==="
    curl -s "https://thatdeveloperguy.com$path" 2>/dev/null \
        | grep -oE 'meta name="(robots|googlebot)"[^>]+' \
        | head -3
done
```

After any template change:

```bash
nginx -t && systemctl reload nginx  # if static files
# OR
systemctl restart fastapi-sidecar    # if FastAPI templates
```

---

## 17. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### Generic meta robots

1. [ ] No unnecessary `<meta name="robots" content="index, follow">` (default behavior, do not emit).
2. [ ] Meta tags are server side rendered (not added via client side JavaScript).
3. [ ] All meta robots tags are in `<head>` (not in `<body>`).
4. [ ] Meta robots case is consistent (lowercase convention).

### Page type coverage

5. [ ] Login pages: `noindex, follow`.
6. [ ] Account pages: `noindex, follow`.
7. [ ] Admin pages: `noindex, nofollow`.
8. [ ] Search results pages: `noindex, follow`.
9. [ ] Filter and sort parameter URLs: `noindex, follow` plus canonical.
10. [ ] Pagination beyond first page: `noindex, follow`.
11. [ ] Thank you / confirmation pages: `noindex, nofollow`.
12. [ ] Print views: `noindex` plus canonical.
13. [ ] Staging and demo subdomains: `noindex, nofollow` plus robots.txt block.
14. [ ] Editorial content: `max-snippet:-1, max-image-preview:large, max-video-preview:-1`.

### Snippet controls

15. [ ] No accidental `max-snippet:0` site wide (would remove all snippets).
16. [ ] `data-nosnippet` attribute used on sensitive elements where appropriate.
17. [ ] AI opt out pages have `nosnippet` (if intentional).

### Time bound content

18. [ ] Event pages have `unavailable_after: <date>` set to post event.
19. [ ] Sale pages have `unavailable_after: <date>` set to sale end.
20. [ ] Time bound directives use correct date format (RFC 850 compatible).

### User agent specific

21. [ ] No conflicting `<meta name="googlebot">` and `<meta name="robots">` directives unless intentional.
22. [ ] Google News opt outs use `<meta name="googlebot-news">`.
23. [ ] `indexifembedded` used only on actual embed source pages.

### The robots.txt paradox

24. [ ] No URLs are in both robots.txt Disallow AND have noindex meta tag.
25. [ ] URLs intended to be deindexed are NOT blocked in robots.txt.
26. [ ] After URLs are deindexed (per GSC verification), robots.txt rules can be tightened.

### Bubbles thin content cleanup (Joseph's pattern)

27. [ ] `NOINDEX_CITIES` (or similar) list defined and applied in template.
28. [ ] Far flung cities: `noindex, follow` plus X-Robots-Tag header.
29. [ ] Core cities (NWA/SWMO): default discoverability directives.
30. [ ] GSC Page Indexing report monitored for deindexing progress.

### HTML and HTTP combination

31. [ ] `/api/*` location has `X-Robots-Tag: noindex` at nginx level (covered in framework-http-seo-headers.md).
32. [ ] Static PDFs have appropriate X-Robots-Tag at nginx level.
33. [ ] HTML pages have meta robots as primary, X-Robots-Tag as backstop.

### Verification

34. [ ] Critical URLs verified via curl + grep.
35. [ ] GSC URL Inspection tool verifies index status matches intent.
36. [ ] Lighthouse audit shows no unexpected indexing issues.
37. [ ] Screaming Frog or similar crawl audit done quarterly.

### CMS or framework specific

38. [ ] CMS plugin settings checked (WordPress Yoast, similar) to ensure they emit consistent tags.
39. [ ] No client side JavaScript framework removing or modifying meta tags after page load.

### Cross cutting

40. [ ] `_meta_robots.html` (or equivalent) macro/helper in place for consistency.
41. [ ] All page types covered in the macro/helper.
42. [ ] Templates use the macro/helper, not raw meta tags scattered through templates.
43. [ ] Code review process catches accidental directive changes.
44. [ ] Pre deploy check verifies no unintended noindex on production.

### Documentation

45. [ ] Page type to directive mapping documented.
46. [ ] Process for adding new page types to the noindex tier documented.
47. [ ] Process for removing pages from noindex tier (when content improves) documented.

### Monitoring

48. [ ] GSC Page Indexing alerts configured.
49. [ ] Weekly review of "Not indexed" reasons.
50. [ ] Quarterly audit of which URLs have which directives.

A site that passes all 50 has correctly configured meta robots usage for SEO control, AI opt out where desired, and accurate signal to crawlers.

---

## 18. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Robots.txt Disallow plus meta noindex on the same URL.**
Symptom: URLs that should be deindexed remain in Google's index.
Why it breaks: robots.txt block prevents Googlebot from crawling; without crawling, the noindex meta tag is never seen.
Fix: remove the Disallow from robots.txt; allow crawling so Google can see the noindex.

**Pitfall 2: Emitting `<meta name="robots" content="index, follow">` everywhere.**
Symptom: bytes wasted; no information conveyed.
Why it breaks: index/follow is the default. Per John Mueller, Google ignores this.
Fix: omit the tag entirely on pages that should be indexed normally.

**Pitfall 3: `noindex` accidentally left on production after testing.**
Symptom: site disappears from Google after deploy.
Why it breaks: development environment used noindex; tag left in template during merge.
Fix: pre deploy check that production templates do not have unconditional noindex.

**Pitfall 4: Adding `noindex` via client side JavaScript.**
Symptom: Google indexes the page despite noindex appearing in the rendered DOM.
Why it breaks: Googlebot's initial crawl is HTML only; the JS modifies DOM later.
Fix: server side render the meta tag.

**Pitfall 5: `max-snippet:0` site wide thinking it improves SEO.**
Symptom: search results show no snippet text; click through drops.
Why it breaks: `max-snippet:0` removes the description preview. Users see only title and URL.
Fix: use `max-snippet:-1` for editorial content (unlimited length, Google decides).

**Pitfall 6: `noindex` on pagination first page.**
Symptom: pagination structure loses its first page from the index.
Why it breaks: noindex applied to all pagination pages including page 1.
Fix: only noindex pages 2+. First page should be indexable.

**Pitfall 7: Search results page with `index` (no noindex).**
Symptom: thousands of "soft 404" search URLs indexed for random queries.
Why it breaks: search results pages are thin content. Indexing them dilutes site quality.
Fix: `noindex, follow` on all search results pages.

**Pitfall 8: Filter URLs without canonical.**
Symptom: `/products?color=red`, `/products?color=blue` etc all indexed as duplicates.
Why it breaks: missing canonical tag.
Fix: `noindex, follow` plus `<link rel="canonical" href="https://example.com/products">`.

**Pitfall 9: AI opt out with non standard meta tags.**
Symptom: emitting `<meta name="noai" content="1">` and expecting AI training opt out.
Why it breaks: non standard; not honored by major AI crawlers.
Fix: use robots.txt Disallow with GPTBot, ClaudeBot, etc; or use `nosnippet` for Google AI Overviews.

**Pitfall 10: `unavailable_after` with wrong date format.**
Symptom: directive ignored; page stays indexed past expiration.
Why it breaks: date format does not match expected formats.
Fix: use RFC 850 format: `unavailable_after: 25 May 2026 12:00:00 PST`.

**Pitfall 11: `indexifembedded` without companion `noindex`.**
Symptom: directive has no effect.
Why it breaks: `indexifembedded` only modifies behavior when paired with noindex.
Fix: combine: `<meta name="googlebot" content="noindex, indexifembedded">`.

**Pitfall 12: HTTP X-Robots-Tag conflicts with HTML meta.**
Symptom: page indexed despite meta noindex (or vice versa).
Why it breaks: most restrictive wins; the HTTP layer may add `noindex` while the HTML says index.
Fix: align the two layers; document which is the source of truth per URL pattern.

**Pitfall 13: `nofollow` on all pages (page level).**
Symptom: internal link graph loses all authority flow.
Why it breaks: page level nofollow prevents authority distribution through internal links.
Fix: only use page level nofollow when truly needed; use `rel="nofollow"` on individual external links instead.

**Pitfall 14: `googlebot-news` directive without intent.**
Symptom: site mysteriously excluded from Google News despite being a news site.
Why it breaks: directive misapplied or left from a copy paste from a different site.
Fix: only set `googlebot-news` directives if you intentionally want news specific behavior.

**Pitfall 15: Adding noindex to a page that has external links pointing to it.**
Symptom: page deindexes but the external links continue pointing to a non indexable URL.
Why it breaks: external authority "wasted" on a deindexed page.
Fix: consider whether to instead redirect (301) to a relevant indexable URL, or improve the noindex page so it becomes indexable.

---

## 19. DIAGNOSTIC COMMANDS

Reference of every command useful for meta robots investigation.

### Inspect meta robots on a single URL

```bash
# Extract all meta robots related tags
curl -s https://example.com/some-page | grep -oE 'meta name="[^"]*robot[^"]*"[^>]+' | head -10

# Extract just the directives
curl -s https://example.com/some-page | grep -oE 'meta name="robots"[^>]+' | sed 's/.*content="\([^"]*\)".*/\1/'

# Also check googlebot specific
curl -s https://example.com/some-page | grep -oE 'meta name="googlebot"[^>]+'

# Check data-nosnippet usage
curl -s https://example.com/some-page | grep -c 'data-nosnippet'
```

### Audit a sitemap for unexpected noindex

```bash
# Pages in sitemap should NOT be noindex
curl -s https://example.com/sitemap.xml | \
    grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | \
    while read url; do
        NOINDEX=$(curl -s "$url" 2>/dev/null | grep -ic 'content="noindex')
        if [ "$NOINDEX" != "0" ]; then
            echo "PROBLEM: $url has noindex"
        fi
    done
```

### Verify noindex on a specific set of URLs (should be noindexed)

```bash
# URLs expected to be noindex
EXPECTED_NOINDEX=(
    "/search?q=test"
    "/login"
    "/admin/"
    "/account/profile"
)

for path in "${EXPECTED_NOINDEX[@]}"; do
    NOINDEX_COUNT=$(curl -s "https://example.com$path" 2>/dev/null | grep -ic 'content="noindex')
    if [ "$NOINDEX_COUNT" = "0" ]; then
        echo "MISSING NOINDEX: $path"
    fi
done
```

### Cross check HTTP header AND meta tag alignment

```bash
URL="https://example.com/admin/"

# HTTP header
HEADER=$(curl -sI "$URL" | grep -i "x-robots-tag" | tr -d '\r')

# HTML meta
META=$(curl -s "$URL" | grep -oE 'meta name="robots"[^>]+' | head -1)

echo "URL: $URL"
echo "HTTP: $HEADER"
echo "META: $META"

# If both should say noindex, both should be present
```

### Diff meta robots between two pages

```bash
URL1="https://example.com/services/cassville/"  # expected indexable
URL2="https://example.com/services/detroit/"    # expected noindex

diff \
    <(curl -s "$URL1" | grep -oE 'meta name="robots"[^>]+') \
    <(curl -s "$URL2" | grep -oE 'meta name="robots"[^>]+')
```

### Find pages in GSC with unexpected status

In Google Search Console:

1. Page Indexing report to "Indexed, though blocked by robots.txt".
2. Page Indexing report to "Excluded by 'noindex' tag".
3. Compare counts to expected (matches your noindex_follow tier).

Manual but critical step; cannot be automated via standard CLI.

### Browser DevTools verification

In Chrome DevTools:

1. Open the page.
2. View Source (Cmd/Ctrl + U) to see initial HTML.
3. Find `<meta name="robots">` in the source.
4. Inspect Element (Cmd/Ctrl + Shift + I) to Network tab to check headers for `X-Robots-Tag`.
5. Console: `document.head.querySelector('meta[name="robots"]')?.content` shows the current state.

### Bulk audit script

```bash
#!/bin/bash
# /usr/local/bin/audit-meta-robots.sh
# Run weekly to verify meta robots state across critical URLs

BASE=https://example.com

# Pages that SHOULD be indexed
INDEXABLE=(
    "/"
    "/about"
    "/services"
    "/blog"
    "/contact"
)

# Pages that SHOULD have noindex
NOINDEX=(
    "/login"
    "/search?q=test"
    "/admin"
)

echo "=== Indexable pages ==="
for path in "${INDEXABLE[@]}"; do
    NOINDEX_COUNT=$(curl -s "$BASE$path" 2>/dev/null | grep -ic 'content="noindex')
    if [ "$NOINDEX_COUNT" != "0" ]; then
        echo "PROBLEM: $path has noindex but should not"
    else
        echo "OK: $path"
    fi
done

echo ""
echo "=== Should-be-noindexed pages ==="
for path in "${NOINDEX[@]}"; do
    NOINDEX_COUNT=$(curl -s "$BASE$path" 2>/dev/null | grep -ic 'content="noindex')
    if [ "$NOINDEX_COUNT" = "0" ]; then
        echo "PROBLEM: $path missing noindex"
    else
        echo "OK: $path"
    fi
done
```

### After template changes

```bash
# Clear any cached templates
systemctl restart fastapi-sidecar

# Verify the change is live
curl -s https://example.com/test-page | grep -oE 'meta name="robots"[^>]+'

# Then GSC URL Inspection for the changed pages to accelerate recrawl
```

---

## 20. CROSS-REFERENCES

* [framework-http-seo-headers.md](framework-http-seo-headers.md): the X-Robots-Tag HTTP header is the equivalent for non HTML resources and path level policy at the nginx layer. The directives are the same; the delivery mechanism differs.
* [framework-http-request-headers.md](framework-http-request-headers.md) Section 5: the AI crawler taxonomy (current UA strings for ClaudeBot, GPTBot, OAI-SearchBot, etc).
* [framework-http-2xx-status-codes.md](framework-http-2xx-status-codes.md) Section 10: the soft 404 trap; pages with `noindex` plus thin content are different from `200 with thin content` (which is the soft 404).
* [framework-http-3xx-status-codes.md](framework-http-3xx-status-codes.md) Section 16: the Bubbles 14 paying client subdomain redirect pattern (the 301 layer; this framework covers the noindex meta tier).
* [framework-http-4xx-status-codes.md](framework-http-4xx-status-codes.md) Section 17: the Bubbles thin content cleanup HTTP side (410 Gone for bucket 3); this framework covers the HTML side (noindex for bucket 2).
* [framework-http-caching-headers.md](framework-http-caching-headers.md): meta robots tags are content; cache headers control how the response containing them is cached.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Meta robots is one of the levers for controlling what gets indexed.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including meta robots placement in templates.
* Google robots meta tag documentation: https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag
* Google indexifembedded blog post: https://developers.google.com/search/blog/2022/01/robots-meta-tag-indexifembedded
* Google data-nosnippet documentation: https://developers.google.com/search/docs/appearance/snippet
* Bing robots meta tag documentation: https://www.bing.com/webmasters/help/which-robots-metatags-does-bing-support-5198d240
* MDN meta name="robots": https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Meta robots directive decision matrix

```
What is this page?
   |
   |---> Substantive editorial content ............ (no tag needed; default is index, follow)
   |                                                 Optional: max-snippet:-1, max-image-preview:large
   |
   |---> Search results, filters, pagination 2+ ... noindex, follow
   |
   |---> Login, account, registration .............. noindex, follow
   |
   |---> Admin, internal tools ..................... noindex, nofollow
   |
   |---> Thank you, confirmation ................... noindex, nofollow
   |
   |---> Staging, demo, development ................ noindex, nofollow + robots.txt block
   |
   |---> Time limited (event, sale) ................ unavailable_after: DATE
   |
   |---> Want searchable but NOT in AI Overviews ... nosnippet
   |
   |---> Podcast or video host page ................ noindex, indexifembedded (Google only)
   |
   |---> Print version ............................. noindex + canonical to main version
   |
   |---> Page with proprietary images .............. noimageindex
```

### Tag scope

| Name | Applies to |
|---|---|
| `robots` | All crawlers (default) |
| `googlebot` | Googlebot (overrides robots for Google) |
| `bingbot` | Bingbot (overrides robots for Bing) |
| `googlebot-news` | Google News specifically |

### Five rules to memorize

1. **Default is index, follow.** Do not emit it; it is noise.
2. **noindex requires crawlability.** Do not combine with robots.txt Disallow.
3. **Most restrictive wins.** When directives conflict.
4. **Server side render the tag.** Client side JS modifications may not be seen.
5. **Sitemap should never list noindex URLs.** They should not be there.

### Five commands every operator should know

```bash
# 1. Check meta robots for a URL
curl -s https://example.com/page | grep -oE 'meta name="robots"[^>]+'

# 2. Find unexpected noindex in sitemap
curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | head -20 | while read url; do
    NOINDEX=$(curl -s "$url" | grep -ic 'content="noindex')
    [ "$NOINDEX" != "0" ] && echo "PROBLEM: $url"
done

# 3. Verify expected noindex pages
for p in /login /admin/ /search; do
    HAS=$(curl -s "https://example.com$p" | grep -ic 'noindex')
    [ "$HAS" = "0" ] && echo "MISSING NOINDEX: $p" || echo "OK: $p"
done

# 4. Cross check HTTP header AND meta tag
URL=https://example.com/admin/
curl -sI "$URL" | grep -i x-robots-tag
curl -s "$URL" | grep -oE 'meta name="robots"[^>]+'

# 5. After template change
systemctl restart fastapi-sidecar
curl -s https://example.com/changed-page | grep -oE 'meta name="robots"[^>]+'
```

### Three end to end tests

```bash
# 1. Homepage is indexable
HAS_NOINDEX=$(curl -s https://example.com/ | grep -ic 'content="noindex')
[ "$HAS_NOINDEX" = "0" ] && echo "OK: homepage indexable" || echo "FAIL: homepage has noindex!"

# 2. Admin pages have noindex
HAS_NOINDEX=$(curl -s https://example.com/admin/ | grep -ic 'content="noindex')
[ "$HAS_NOINDEX" != "0" ] && echo "OK: admin noindexed" || echo "FAIL: admin missing noindex"

# 3. Sitemap URLs are all indexable
PROBLEMS=$(curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | head -10 | while read url; do
    HAS=$(curl -s "$url" | grep -ic 'content="noindex')
    [ "$HAS" != "0" ] && echo "$url"
done | wc -l)
[ "$PROBLEMS" = "0" ] && echo "OK: no noindex URLs in sitemap" || echo "FAIL: $PROBLEMS sitemap URLs have noindex"
```

If all three pass AND GSC Page Indexing report shows expected "Excluded by noindex" count matching your intent, the meta robots layer is correctly wired.

---

End of framework-html-meta-robots.md.

End of the first framework in the HTML signal track beyond the HTTP wire layer.
