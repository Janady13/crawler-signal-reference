# framework-html-open-graph.md

Comprehensive reference for the Open Graph protocol, the meta tag system that controls how URLs are represented when shared on Facebook, LinkedIn, Pinterest, Slack, Discord, Telegram, iMessage, WhatsApp, and most other platforms that generate link previews. Covers the four required basic properties (`og:title`, `og:type`, `og:image`, `og:url`), the three recommended properties (`og:description`, `og:site_name`, `og:locale`), the comprehensive image specification (1200x630 standard for highest quality sharing; 600x315 minimum; sub properties `og:image:width`, `og:image:height`, `og:image:alt`, `og:image:type`, `og:image:secure_url`), the `og:type` values (website, article, video.*, music.*, profile, book) and their type specific properties, the article subtype properties (`article:published_time`, `article:author`, `article:section`, `article:tag`, `article:modified_time`, `article:expiration_time`) essential for blog content, the video and audio types (`og:video`, `og:audio` with their sub properties), the profile and book types (less commonly used), the multi language pattern (`og:locale` plus `og:locale:alternate`) with the Marshallese-Voices case study, the relationship with Twitter Cards (which falls back to Open Graph when twitter:* tags absent), the relationship with Schema.org structured data (complementary but distinct), the validation tools (Facebook Sharing Debugger, LinkedIn Post Inspector, opengraph.xyz), and the Bubbles per client decision framework with brand coherent images, professional YMYL patterns for Arkansas Counseling and Handled Tax, and federal subcontractor appearance for WeCoverUSA. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the fourteenth framework in the HTML signal track**, following the meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, color-scheme, referrer, content-language, and refresh frameworks. Companion to the 12 wire layer frameworks and Twitter Cards framework (next).

Audience: humans implementing professional social media link previews for client sites, AI assistants generating HTML head sections with appropriate Open Graph for the page type, designers creating 1200x630 brand coherent OG images, editorial content authors ensuring blog posts have proper article subtype tags, multi language site operators managing og:locale and og:locale:alternate (Marshallese-Voices), federal subcontractors ensuring professional appearance in link previews shared by federal procurement officers, and anyone troubleshooting "Facebook is showing the wrong image when our URL is shared", "LinkedIn preview shows no description", "the share preview is from a different page on our site", or "iMessage shows just the URL without preview".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Mental Model: Required Four, Recommended Three, Type Specific
5. The RDFa property= Mechanism (vs name=)
6. The Four Required Basic Properties
7. The Three Recommended Properties
8. The og:image Specification (Comprehensive)
9. The og:type Values And Type Specific Properties
10. The og:url And Canonical Alignment Rule
11. The Article Subtype (article:*)
12. The Video And Audio Types
13. The Profile And Book Types
14. The Multi Language Pattern And Marshallese-Voices Case Study
15. The Relationship With Twitter Cards
16. The Relationship With Schema.org Structured Data
17. The Validation Tools And Cache Refresh
18. The Bubbles Per Client Decision Framework
19. The Server Side Rendering Requirement
20. The Implementation Patterns By Stack
21. The Pre Launch Audit Checklist
22. The Common Failures And Fixes
23. The Federal And SDVOSB Appearance Context
24. Quick Reference Tag Block
25. References

---

## 1. DEFINITION

The Open Graph protocol is a metadata standard published by Facebook in 2010 that uses `<meta property="og:*">` tags in the HTML `<head>` to describe how a URL should be represented when shared on social platforms and messaging applications.

```html
<meta property="og:title" content="Page Title Goes Here">
<meta property="og:type" content="website">
<meta property="og:image" content="https://example.com/image.jpg">
<meta property="og:url" content="https://example.com/page/">
<meta property="og:description" content="Brief description of the page.">
<meta property="og:site_name" content="Site Name">
<meta property="og:locale" content="en_US">
```

The protocol uses the RDFa `property=` attribute instead of the standard HTML `name=` attribute, with the `og:` namespace prefix. The spec lives at https://ogp.me and has been unchanged since 2014; what has changed is the rendering and caching behavior of consuming platforms.

When a user pastes a URL into Facebook, LinkedIn, Slack, Discord, iMessage, WhatsApp, Telegram, Pinterest, or virtually any modern application that generates link previews, that platform's crawler fetches the page, parses the `og:*` tags, and uses them to render a preview card with image, title, description, and site identity. Without Open Graph, the same paste produces either no preview, a fallback to whatever the platform can scrape from `<title>` and the first `<img>` it finds, or a bare blue hyperlink with no preview at all.

---

## 2. WHY IT MATTERS

Open Graph is the single highest leverage HTML signal for off site distribution.

**The cost is four lines of HTML in the `<head>`. The upside compounds across every channel a link can be shared in.**

For Bubbles clients, this directly impacts:

* **Click through rate on shared links.** A preview card with image, title, and description converts at multiples of a bare hyperlink. Facebook reports CTR lifts of 2x to 7x for posts with proper Open Graph versus posts without.
* **Facebook marketing reach.** Joseph's Facebook marketing model relies on links shared into 20+ NWA and SWMO groups. A link without preview is invisible noise. A link with preview is a small advertisement that earns clicks on its visual merits.
* **LinkedIn professional appearance.** For federal subcontracting context (WeCoverUSA, ThatDeveloperGuy SDVOSB), a link shared on LinkedIn without preview looks unprofessional and unverified. Procurement officers and contracting officers do notice.
* **Messaging app previews.** iMessage, WhatsApp, Telegram, and Signal all render Open Graph previews. A client sharing their own site URL with a friend or referral generates a polished preview, not a naked URL.
* **AI assistant link unfurling.** ChatGPT, Claude, Gemini, and Perplexity all parse Open Graph when users paste URLs. The og:title and og:description become the AI's understanding of the page.
* **Slack and Discord internal sharing.** When clients share their site internally with their team or with referral partners, Open Graph produces a clean unfurl. Without it, the link is a bare URL.

Sites without Open Graph forfeit every channel where a URL can be pasted. Sites with Open Graph turn every share into a small impression.

---

## 3. WHAT THIS COVERS

This framework covers:

* The four required basic properties (`og:title`, `og:type`, `og:image`, `og:url`).
* The three recommended properties (`og:description`, `og:site_name`, `og:locale`).
* The comprehensive `og:image` specification: dimensions (1200x630 standard, 600x315 minimum), aspect ratio (1.91:1), file format (JPG, PNG, WebP), file size limits per platform, and the five sub properties (`og:image:width`, `og:image:height`, `og:image:alt`, `og:image:type`, `og:image:secure_url`).
* All `og:type` values: `website` (default), `article` (blog content), `video.*` (video pages), `music.*` (audio pages), `profile` (people), `book` (book detail pages), and the deprecated `og:type=blog` and `og:type=product`.
* The article subtype properties (`article:published_time`, `article:modified_time`, `article:expiration_time`, `article:author`, `article:section`, `article:tag`).
* The video and audio sub properties (`og:video:*`, `og:audio:*`).
* The profile sub properties (`profile:first_name`, `profile:last_name`, `profile:username`, `profile:gender`).
* The book sub properties (`book:author`, `book:isbn`, `book:release_date`, `book:tag`).
* The multi language pattern (`og:locale` plus `og:locale:alternate`) with the Marshallese-Voices case study referencing framework-html-content-language.md Section 10.
* The relationship with Twitter Cards (twitter:* tags override og:* tags on X, but X falls back to og:* when twitter:* tags absent).
* The relationship with Schema.org structured data (complementary: Open Graph for social preview, Schema.org for search engine understanding).
* The validation tools (Facebook Sharing Debugger at developers.facebook.com/tools/debug/, LinkedIn Post Inspector at linkedin.com/post-inspector/, opengraph.xyz, env.dev meta tag generator).
* The cache refresh behavior per platform (Facebook 30 days, LinkedIn 7 days, Slack 24 to 48 hours, Discord no official tool).
* The server side rendering requirement (most crawlers do not execute JavaScript; `og:*` tags must be in the initial HTML response, not injected by React/Vue/Svelte hydration).
* The Bubbles per client decision framework with brand coherent images for ThatDeveloperGuy, TCB Fight Factory (purple and black only), Arkansas Counseling and Wellness, Handled Tax and Advisory, WeCoverUSA (federal appearance), White River Cabins, Greenough's Guide Service, Heritage Hardwood Floors NWA, Local Living Real Estate, Diana Undergust, Homes on Grand, Marshallese-Voices (multi language), Avid Pest Control, RAH Construction NWA, Steele Solutions, Blue Paradise Dairy, Pretty Clean NWA, and Eureka Bath Works.

This framework does NOT cover:

* Twitter Cards specifically (`twitter:card`, `twitter:site`, `twitter:creator`, `twitter:image:alt`) — covered in framework-html-twitter-cards.md (next framework in track).
* Schema.org JSON-LD structured data — covered in framework-html-schema-jsonld.md.
* `<link rel="canonical">` — covered in framework-html-canonical.md.
* `<link rel="alternate">` with hreflang — covered in framework-html-hreflang.md (alongside content-language).
* OpenGraph 2.0 proposals or the deprecated music subtypes that never reached wide adoption.

---

## 4. THE MENTAL MODEL: REQUIRED FOUR, RECOMMENDED THREE, TYPE SPECIFIC

Open Graph implementation follows a three layer model:

```
LAYER 1: Required (4 properties on every page)
   |
   |---> og:title       (the social preview headline)
   |---> og:type        (website, article, video.*, music.*, profile, book)
   |---> og:image       (the 1200x630 preview image)
   |---> og:url         (the canonical URL of the page)

LAYER 2: Recommended (3 properties on every page)
   |
   |---> og:description (the social preview body text)
   |---> og:site_name   (the site identity displayed in the card)
   |---> og:locale      (the language and region, e.g. en_US)

LAYER 3: Image sub properties (4 to 5 properties on the og:image)
   |
   |---> og:image:width      (pixel width, typically 1200)
   |---> og:image:height     (pixel height, typically 630)
   |---> og:image:alt        (accessibility description)
   |---> og:image:type       (MIME type, e.g. image/jpeg)
   |---> og:image:secure_url (HTTPS URL when og:image is HTTP, rarely used in 2026)

LAYER 4: Type specific (only when og:type matches)
   |
   |---> article:* properties when og:type=article (blog posts)
   |---> video:* properties when og:type=video.*
   |---> music:* properties when og:type=music.*
   |---> profile:* properties when og:type=profile
   |---> book:* properties when og:type=book

LAYER 5: Multi language (when site serves multiple languages)
   |
   |---> og:locale          (primary locale)
   |---> og:locale:alternate (each additional locale, repeatable)
```

Seven rules govern the system:

1. **Always include the four required properties** on every indexable page.
2. **Use og:type appropriately**: `website` for marketing pages, `article` for blog and editorial.
3. **og:image must be 1200x630** (or close to that ratio) with width, height, alt, and type sub properties declared.
4. **og:url must match the canonical URL** declared in `<link rel="canonical">`.
5. **For multi language sites**, use og:locale with og:locale:alternate (Marshallese-Voices pattern).
6. **For editorial content**, add article:* subtype properties (published_time at minimum).
7. **Validate via Facebook Sharing Debugger and LinkedIn Post Inspector** before assuming the preview works.

A correctly configured Bubbles client site has comprehensive Open Graph on every page, type appropriate properties matched to the page content, brand coherent imagery in the 1200x630 og:image, and a validated cache state on Facebook and LinkedIn.

---

## 5. THE RDFA PROPERTY= MECHANISM (VS NAME=)

Open Graph uses `property=` instead of `name=`. This is a different attribute mechanism with different semantics.

### 5.1 The Distinction

```html
<!-- Standard meta tag (HTML5 spec) -->
<meta name="description" content="...">

<!-- Open Graph meta tag (RDFa convention) -->
<meta property="og:description" content="...">
```

Both use the `<meta>` element with the `content` attribute. The difference is the first attribute:

* `name="..."` is for standard HTML5 meta tags registered in the WHATWG name extensions registry.
* `property="..."` is for RDFa namespaced tags; Open Graph and its subtypes use this.

### 5.2 The Namespace Prefix

Open Graph tags use the `og:` prefix:

```html
<meta property="og:title" content="...">
<meta property="og:image" content="...">
```

Article subtype uses the `article:` prefix:

```html
<meta property="article:published_time" content="...">
<meta property="article:author" content="...">
```

Each Open Graph type has its own namespace prefix: `video:`, `music:`, `profile:`, `book:`.

### 5.3 The Namespace Declaration (Optional)

Strictly per RDFa, the namespace should be declared on the `<html>` element:

```html
<html prefix="og: https://ogp.me/ns# article: https://ogp.me/ns/article#">
```

In practice, every major platform (Facebook, LinkedIn, Slack, Discord, X, iMessage, WhatsApp, Telegram, Pinterest) detects `og:*` and subtype tags without the namespace declaration. The declaration is technically correct but functionally optional in 2026.

For Bubbles convention: omit the namespace declaration. Cleaner HTML, no rendering impact.

### 5.4 The Validator Behavior

Some strict RDFa validators warn about missing namespace declaration. The Facebook Sharing Debugger does not. The LinkedIn Post Inspector does not. Browsers and platform crawlers parse correctly without it.

### 5.5 The Mistake To Avoid

Using `name="og:..."` instead of `property="og:..."`:

```html
<!-- WRONG: name attribute -->
<meta name="og:title" content="...">

<!-- CORRECT: property attribute -->
<meta property="og:title" content="...">
```

Some platforms accept the wrong form leniently; many do not. Facebook in particular has historically been strict. Use `property=` consistently for every `og:`, `article:`, `video:`, `music:`, `profile:`, and `book:` tag.

Twitter Cards uses the opposite convention (`name="twitter:..."`), which is a frequent source of confusion. Covered in framework-html-twitter-cards.md.

---

## 6. THE FOUR REQUIRED BASIC PROPERTIES

These four are the minimum for a working social preview. Missing any one of them produces a degraded preview on most platforms.

### 6.1 og:title

The headline displayed in the preview card.

```html
<meta property="og:title" content="Heritage Hardwood Floors NWA | Bella Vista Refinishing & Installation">
```

**Length guidance**: 60 to 90 characters. Facebook truncates at approximately 88 characters on desktop, fewer on mobile. LinkedIn truncates around 120 characters but shows a strong visual break at 70.

**Distinction from `<title>`**: og:title can differ from the document `<title>`. Often `<title>` is optimized for search ("Hardwood Floor Refinishing Bella Vista AR | Heritage Hardwood Floors NWA"), while og:title is optimized for social skim ("Heritage Hardwood Floors NWA | Bella Vista Refinishing"). When in doubt, mirror the `<title>` minus the trailing brand fragment.

**For Bubbles client default**: og:title mirrors `<title>` unless a separate social headline is explicitly authored.

### 6.2 og:type

Declares the kind of object this page represents.

```html
<meta property="og:type" content="website">
```

The six modern values used in production:

| og:type | When to use |
|---|---|
| `website` | Marketing pages, home page, service pages, location pages, contact pages. The default. |
| `article` | Blog posts, news articles, editorial content with an author and publish date. |
| `video.movie`, `video.episode`, `video.tv_show`, `video.other` | Pages whose primary content is a video. |
| `music.song`, `music.album`, `music.playlist`, `music.radio_station` | Pages whose primary content is audio. |
| `profile` | Pages representing a single person. |
| `book` | Pages representing a single book. |

Two deprecated values to avoid: `blog` (use `article` per blog post page; use `website` on the blog index page) and `product` (use Schema.org Product JSON-LD instead; covered in framework-html-schema-jsonld.md).

**For Bubbles client default**: `website` on all marketing pages, `article` on every blog post page.

### 6.3 og:image

The preview image. Covered exhaustively in Section 8. The minimum required tag:

```html
<meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/og-image.jpg">
```

**Absolute URL required.** Relative paths are silently ignored by most crawlers.

**HTTPS required.** Mixed content (HTTPS page with HTTP image) is rejected or downgraded on every modern platform.

### 6.4 og:url

The canonical URL of the page. Must match `<link rel="canonical">`.

```html
<meta property="og:url" content="https://heritagehardwoodfloorsnwa.com/services/refinishing/">
```

**Why it exists**: when a URL is shared with tracking parameters or session fragments (`?utm_source=facebook&fbclid=abc`), Facebook uses og:url as the canonical identity. Shares of `https://example.com/page/?utm=x` and `https://example.com/page/?utm=y` are deduplicated to the same object if both pages declare the same og:url.

**Alignment rule**: og:url, `<link rel="canonical">`, the HTTP `Link: rel="canonical"` header (if used), and the URL in the XML sitemap should all match. Mismatches cause Facebook to split the share counter across two object identities and Search Console to flag canonical confusion.

---

## 7. THE THREE RECOMMENDED PROPERTIES

These three are strongly recommended on every page. Their absence produces a usable preview but degraded quality.

### 7.1 og:description

The body text displayed below the title in the preview card.

```html
<meta property="og:description" content="Bella Vista hardwood floor refinishing and installation. Family owned, NWA based, 20+ years experience. Free in home estimates within 48 hours.">
```

**Length guidance**: 110 to 200 characters. Facebook truncates around 200 on desktop, around 110 on mobile. LinkedIn truncates around 200. Slack and Discord show roughly 250.

**Distinction from `<meta name="description">`**: identical content is acceptable and common. If the page has been optimized for SERP CTR with a specific meta description (per framework-html-meta-description.md), reuse it for og:description.

**For Bubbles client default**: og:description mirrors the meta description.

### 7.2 og:site_name

The identity of the site, displayed in the preview card alongside the URL.

```html
<meta property="og:site_name" content="Heritage Hardwood Floors NWA">
```

This is the human readable site identity, not the URL. Facebook displays it in the gray text above the title. LinkedIn shows it as part of the source attribution.

**Stable across the entire site.** Same value on every page. Changing it page to page confuses platforms and harms the consolidated share count.

**For Bubbles client default**: the client's primary business name as it appears in the site logo and footer.

### 7.3 og:locale

The language and region of the page content, in IETF BCP 47 format but with an underscore separator (not hyphen).

```html
<meta property="og:locale" content="en_US">
```

Note: Open Graph uses `en_US` (underscore) while HTTP Content-Language and html lang use `en-US` (hyphen). This is a Facebook spec quirk that has been frozen since 2010 and applies to Open Graph only.

Common values for Bubbles clients:

* `en_US` — US English (default for every NWA and SWMO client).
* `en_GB` — UK English.
* `es_ES` — Spain Spanish.
* `es_US` — US Spanish (for Hispanic NWA market segments).
* `mh_MH` — Marshallese (Marshallese-Voices; see Section 14).

**For Bubbles client default**: `en_US` on every English language client site.

---

## 8. THE OG:IMAGE SPECIFICATION (COMPREHENSIVE)

The og:image is the highest impact element of the preview card. A polished, brand coherent image at the correct dimensions converts; a missing, cropped, or generic image does not.

### 8.1 The Universal Recommended Dimensions: 1200x630

```html
<meta property="og:image" content="https://example.com/og-image.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:type" content="image/jpeg">
<meta property="og:image:alt" content="Heritage Hardwood Floors NWA team installing oak flooring in a Bella Vista home">
```

1200x630 pixels with a 1.91:1 aspect ratio is the universal standard. It renders correctly on Facebook, LinkedIn (which technically prefers 1200x627 but accepts 1200x630 cleanly), X large card, Slack, Discord, iMessage, WhatsApp, and Telegram.

**If you only make one OG image per page, make it 1200x630.**

### 8.2 The Per Platform Dimensions (2026)

| Platform | Optimal | Notes |
|---|---|---|
| Facebook | 1200x630 (1.91:1) | Up to 8 MB. Falls back to small thumbnail below 600x315. |
| LinkedIn | 1200x627 (1.91:1) | Up to 5 MB. Below 401px wide shows as small thumbnail. |
| X (Twitter) summary_large_image | 1200x600 (2:1) | Will crop top and bottom of a 1200x630 image by a few pixels. |
| X (Twitter) summary | 1200x1200 (1:1) | Square, used only when twitter:card=summary. |
| Slack | 1200x630 (1.91:1) | No documented file size cap. |
| Discord | 1200x630 (1.91:1) | Up to 8 MB. Animates GIF first frame. |
| iMessage | 1200x630 (1.91:1) | Sticky cache; rerendering can take days. |
| WhatsApp | 1200x630 (1.91:1) | Target file size ≤ 300 KB for reliable rendering. |
| Pinterest | 1000x1500 (2:3 vertical) | Pinterest prefers vertical; if Pinterest is a major channel, supply a second og:image. |

### 8.3 The Minimum And Fallback Behavior

* Below 600x315: Facebook renders as small thumbnail card (not large).
* Below 401px wide: LinkedIn renders as small thumbnail.
* Below 200x200: most platforms ignore the image entirely.

**For Bubbles convention: never ship an og:image smaller than 1200x630.**

### 8.4 The File Format Decision

| Format | Compatibility | Recommendation |
|---|---|---|
| JPG | Universal | **Default for photographic content.** |
| PNG | Universal | **Default for graphic content with text and flat color.** |
| WebP | Facebook, X, LinkedIn, Slack, Discord; uneven on older crawlers | Safe as primary if JPG fallback exists; risky as the only format. |
| AVIF | Facebook only | **Do not use as og:image in 2026.** |
| GIF | Most platforms show first frame as static; Discord animates | Use only when animation is the point. |
| SVG | Inconsistent rendering | **Do not use as og:image.** |

**For Bubbles convention: JPG for photo based og:image, PNG for design based og:image.** Skip WebP for OG until 2027 or later; the savings are not worth the risk.

### 8.5 The File Size Limits

* Facebook: 8 MB.
* LinkedIn: 5 MB.
* Discord: 8 MB.
* WhatsApp: practical reliability cap around 300 KB.

**Target ≤ 300 KB** to render reliably on every platform including WhatsApp.

### 8.6 The Safe Zone For Text And Logos

Different platforms crop the image slightly differently. Keep critical content (headline text, logo, faces) inside the centered 1080x600 safe zone of a 1200x630 image. The outer 60 pixels on each side may be cropped by X or by mobile rendering.

### 8.7 The og:image:width And og:image:height

```html
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
```

**Strongly recommended.** Platforms render the preview faster when dimensions are declared upfront. LinkedIn in particular will sometimes skip the image entirely on first crawl if dimensions are missing and the image takes too long to download for size inspection.

### 8.8 The og:image:alt

```html
<meta property="og:image:alt" content="Heritage Hardwood Floors NWA team installing oak flooring in a Bella Vista home">
```

Accessibility description used by screen readers when the preview card is rendered in an assistive context. Also used by some AI link unfurling tools.

**Required for federal subcontracting context** (Section 508 accessibility). Best practice everywhere.

### 8.9 The og:image:type

```html
<meta property="og:image:type" content="image/jpeg">
```

MIME type of the image. Optional but helps crawlers process the file without sniffing.

Valid values: `image/jpeg`, `image/png`, `image/webp`, `image/gif`.

### 8.10 The og:image:secure_url

```html
<meta property="og:image:secure_url" content="https://example.com/og-image.jpg">
```

Used historically when og:image contained an HTTP URL and a separate HTTPS URL needed to be declared. In 2026, og:image should always be HTTPS, making og:image:secure_url redundant. Omit it.

### 8.11 The Multiple og:image Pattern

Open Graph allows multiple og:image tags. Platforms typically use the first one but may rotate or let the user choose.

```html
<meta property="og:image" content="https://example.com/og-primary.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">

<meta property="og:image" content="https://example.com/og-secondary.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
```

**For Bubbles convention: one og:image per page.** Multiple is occasionally useful for ecommerce product galleries; rarely worth the complexity for typical client sites.

### 8.12 The Brand Coherent Image Convention

For Bubbles clients, the og:image is a brand asset and follows the same rules as the rest of the site visual identity:

* **ThatDeveloperGuy**: purple and green palette, hand coded aesthetic, agency logo. Footer credit "Crafted by ThatDeveloperGuy.com" is NOT in the OG image (it lives in the site footer only).
* **TCB Fight Factory**: purple and black only; no pure black backgrounds; fight imagery or logo on brand purple field.
* **Arkansas Counseling and Wellness Services**: warm professional palette, Dr. Kristy Burton brand colors, no clinical sterility.
* **Handled Tax and Advisory**: professional financial palette, Amanda's brand, conservative and trustworthy aesthetic; em dashes in body copy do not apply to OG image text.
* **WeCoverUSA**: federal subcontractor appearance, polished professional aesthetic for procurement officer LinkedIn views.

### 8.13 The Page Specific OG Image Pattern

For Bubbles clients with larger content libraries (blog posts, location pages, service pages, cabin pages), generate page specific og:image rather than reusing the home page og:image everywhere:

```html
<!-- Home page -->
<meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/og-image.jpg">

<!-- Services > Refinishing page -->
<meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/services/refinishing/og-image.jpg">

<!-- Location > Bella Vista page -->
<meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/locations/bella-vista/og-image.jpg">
```

Page specific og:image lifts CTR on shared deep links. The home page og:image is a generic fallback only.

---

## 9. THE OG:TYPE VALUES AND TYPE SPECIFIC PROPERTIES

The `og:type` value declares what kind of object this page represents and unlocks type specific sub properties.

### 9.1 og:type=website (Default)

```html
<meta property="og:type" content="website">
```

The default for marketing pages, home page, service pages, contact pages, location pages, gallery pages, and any non editorial page.

No type specific sub properties required. The four required and three recommended properties cover everything.

**For Bubbles convention: every non blog page uses `og:type="website"`.**

### 9.2 og:type=article (Editorial Content)

```html
<meta property="og:type" content="article">
```

Used on blog posts, news articles, case studies, and editorial content with an author and publish date.

Unlocks the `article:*` sub properties (Section 11).

**For Bubbles convention: every blog post page uses `og:type="article"` with at minimum `article:published_time`.**

### 9.3 og:type=video.* (Video Content)

Four subtypes:

* `video.movie` — feature length film.
* `video.episode` — single episode of a TV series.
* `video.tv_show` — a TV series.
* `video.other` — anything else.

Unlocks the `video:*` sub properties (Section 12).

**For Bubbles convention: rarely applicable.** Client video content is almost always hosted on YouTube, Vimeo, or Facebook, where the host platform's Open Graph handles preview generation. Skip on Bubbles client sites.

### 9.4 og:type=music.* (Audio Content)

Four subtypes:

* `music.song` — a single song.
* `music.album` — an album.
* `music.playlist` — a playlist.
* `music.radio_station` — a radio station.

Unlocks the `music:*` sub properties.

**For Bubbles convention: not applicable.** Skip.

### 9.5 og:type=profile (Person)

```html
<meta property="og:type" content="profile">
<meta property="profile:first_name" content="Joseph">
<meta property="profile:last_name" content="Anady">
<meta property="profile:username" content="thatdeveloperguy">
```

Used on bio pages or about pages representing a single person. Adds `profile:first_name`, `profile:last_name`, `profile:username`, and `profile:gender` sub properties.

**For Bubbles convention: rarely useful.** A practitioner about page (Dr. Kristy Burton, Amanda Emerdinger, Joseph) can use `og:type="profile"` if desired, but `og:type="website"` works fine. Use Person JSON-LD instead for richer signal to search engines.

### 9.6 og:type=book

```html
<meta property="og:type" content="book">
<meta property="book:author" content="https://example.com/authors/jane-doe/">
<meta property="book:isbn" content="978-3-16-148410-0">
<meta property="book:release_date" content="2026-03-15">
```

For pages representing a single book. Not applicable to any current Bubbles client.

### 9.7 The Deprecated Values

* `og:type="blog"` — never use. Use `website` on the blog index, `article` on each post.
* `og:type="product"` — never use. Facebook supports it but parsing is unreliable; use Schema.org Product JSON-LD instead.
* `og:type="restaurant.*"` — Facebook proprietary; not standard. Use Schema.org Restaurant.
* `og:type="business.business"` — Facebook proprietary; use Schema.org LocalBusiness.

---

## 10. THE OG:URL AND CANONICAL ALIGNMENT RULE

The og:url and the `<link rel="canonical">` URL must match. Mismatch causes platform confusion and split share counts.

### 10.1 The Pattern

```html
<head>
    <link rel="canonical" href="https://heritagehardwoodfloorsnwa.com/services/refinishing/">

    <meta property="og:url" content="https://heritagehardwoodfloorsnwa.com/services/refinishing/">
</head>
```

Identical URL in both places.

### 10.2 The Trailing Slash Policy

Per Joseph's Bubbles trailing slash policy, all client URLs end in `/` (except files with extensions). Both `<link rel="canonical">` and `og:url` reflect this.

```html
<!-- CORRECT (trailing slash matches Bubbles policy) -->
<link rel="canonical" href="https://heritagehardwoodfloorsnwa.com/services/">
<meta property="og:url" content="https://heritagehardwoodfloorsnwa.com/services/">

<!-- WRONG (mismatched trailing slash) -->
<link rel="canonical" href="https://heritagehardwoodfloorsnwa.com/services/">
<meta property="og:url" content="https://heritagehardwoodfloorsnwa.com/services">
```

### 10.3 The HTTPS And Host Consistency

og:url must be HTTPS. og:url must be the canonical host (no www if site canonicalizes to bare, no bare if site canonicalizes to www).

### 10.4 The Query Parameter Stripping

og:url should never contain tracking parameters (utm_*, fbclid, gclid). Strip them.

Facebook uses og:url to deduplicate shares; if `?utm_source=facebook` is in og:url, every UTM tagged share creates a new object identity.

### 10.5 The Pagination Pattern

For paginated content (`/blog/`, `/blog/page/2/`, `/blog/page/3/`):

* Each page declares its own canonical URL.
* og:url on `/blog/page/2/` is `https://example.com/blog/page/2/`, not `/blog/`.

Don't collapse pagination into a single og:url; treat each page as a distinct object.

---

## 11. THE ARTICLE SUBTYPE (article:*)

When `og:type="article"`, the `article:*` sub properties provide editorial metadata that improves rendering on Facebook, LinkedIn, and AI assistant unfurling.

### 11.1 article:published_time

ISO 8601 datetime when the article was first published.

```html
<meta property="article:published_time" content="2026-05-15T08:00:00-05:00">
```

Include timezone offset (Central US is `-05:00` during DST, `-06:00` outside DST). Some platforms display "Published 3 days ago" using this value.

**Required for any og:type=article page.**

### 11.2 article:modified_time

ISO 8601 datetime of the most recent substantive update.

```html
<meta property="article:modified_time" content="2026-05-20T14:30:00-05:00">
```

Update when the article content materially changes. Don't update for trivial typo fixes.

**Recommended for any updated article. Aligns with Schema.org `dateModified`.**

### 11.3 article:expiration_time

ISO 8601 datetime after which the article is no longer accurate or relevant.

```html
<meta property="article:expiration_time" content="2027-05-15T00:00:00-05:00">
```

Used for time sensitive content (event announcements, limited time offers, year specific guides). Most clients don't need it.

### 11.4 article:author

URL pointing to the author's profile page on the site (or a Facebook profile URL in legacy implementations).

```html
<meta property="article:author" content="https://thatdeveloperguy.com/about/">
```

For multi author sites (rare in Bubbles clients), repeat the tag for each author:

```html
<meta property="article:author" content="https://example.com/authors/joseph/">
<meta property="article:author" content="https://example.com/authors/sarah/">
```

**Recommended for every article.** Improves E-E-A-T signal for editorial credibility (referenced in framework-html-meta-author.md).

### 11.5 article:section

The primary editorial category or section.

```html
<meta property="article:section" content="Web Development">
```

Single value (not repeatable). Use the site's primary content category for the post.

Common Bubbles client sections:

* ThatDeveloperGuy: "Web Development", "SEO", "Engine Optimization", "Northwest Arkansas Business", "Case Studies".
* Arkansas Counseling and Wellness: "Mental Health", "Couples Therapy", "Anxiety", "Depression", "Family Therapy".
* Handled Tax and Advisory: "Tax Filing", "Small Business", "Self Employed", "Multi State Returns".
* White River Cabins: "Cabin Stays", "Things To Do", "Local Guides", "Trip Planning".

### 11.6 article:tag

Editorial tags. Repeatable for each tag.

```html
<meta property="article:tag" content="hand coded">
<meta property="article:tag" content="SEO">
<meta property="article:tag" content="Northwest Arkansas">
<meta property="article:tag" content="small business">
```

3 to 7 tags is typical. Aligns with the post's keyword targets and the on page tag display.

### 11.7 The Complete Article Pattern

```html
<head>
    <link rel="canonical" href="https://thatdeveloperguy.com/blog/hand-coded-vs-platforms/">

    <!-- Required four -->
    <meta property="og:title" content="Why Hand Coded Sites Outperform Platforms">
    <meta property="og:type" content="article">
    <meta property="og:image" content="https://thatdeveloperguy.com/blog/og-hand-coded-vs-platforms.jpg">
    <meta property="og:url" content="https://thatdeveloperguy.com/blog/hand-coded-vs-platforms/">

    <!-- Recommended three -->
    <meta property="og:description" content="Why hand coded websites outperform Wix and Squarespace for small business SEO. 5 case studies from Northwest Arkansas.">
    <meta property="og:site_name" content="ThatDeveloperGuy">
    <meta property="og:locale" content="en_US">

    <!-- Image sub properties -->
    <meta property="og:image:width" content="1200">
    <meta property="og:image:height" content="630">
    <meta property="og:image:type" content="image/jpeg">
    <meta property="og:image:alt" content="Hand coded versus platforms comparison illustration with ThatDeveloperGuy branding">

    <!-- Article subtype -->
    <meta property="article:published_time" content="2026-05-15T08:00:00-05:00">
    <meta property="article:modified_time" content="2026-05-15T08:00:00-05:00">
    <meta property="article:author" content="https://thatdeveloperguy.com/about/">
    <meta property="article:section" content="Web Development">
    <meta property="article:tag" content="hand coded">
    <meta property="article:tag" content="SEO">
    <meta property="article:tag" content="Northwest Arkansas">
</head>
```

Comprehensive article level Open Graph.

### 11.8 The Bubbles Article Convention

For every editorial blog post on Bubbles client sites:

* `og:type="article"`.
* `article:published_time` required.
* `article:modified_time` updated on substantive edits.
* `article:author` URL pointing to the author about page.
* `article:section` from the editorial taxonomy.
* `article:tag` for 3 to 7 primary keywords.

---

## 12. THE VIDEO AND AUDIO TYPES

For pages whose primary content is a video or audio file, the video and audio types unlock inline player embedding on some platforms.

### 12.1 og:video

```html
<meta property="og:type" content="video.other">
<meta property="og:video" content="https://example.com/video.mp4">
<meta property="og:video:secure_url" content="https://example.com/video.mp4">
<meta property="og:video:type" content="video/mp4">
<meta property="og:video:width" content="1280">
<meta property="og:video:height" content="720">
```

When set, Facebook may embed an inline video player in the preview card instead of a static image. og:image is still required as a poster frame fallback.

**For Bubbles convention: rarely applicable.** Most client video content is hosted on YouTube, Vimeo, or Facebook directly; the host platform's Open Graph handles preview generation when those URLs are shared.

If a Bubbles client wants to self host a video and have it embed in Facebook previews (rare; the TCB Fight Factory sizzle reel is a candidate), implement these tags on the page that displays the video.

### 12.2 og:audio

```html
<meta property="og:type" content="music.song">
<meta property="og:audio" content="https://example.com/audio.mp3">
<meta property="og:audio:secure_url" content="https://example.com/audio.mp3">
<meta property="og:audio:type" content="audio/mpeg">
```

For podcast episodes and music pages. Rarely applicable for Bubbles client work.

### 12.3 The Bubbles Application

For most Bubbles clients: skip video and audio og:* tags entirely.

For potential future podcast or self hosted video clients: implement when the use case justifies it.

---

## 13. THE PROFILE AND BOOK TYPES

Two less commonly used types.

### 13.1 og:type=profile

```html
<meta property="og:type" content="profile">
<meta property="profile:first_name" content="Joseph">
<meta property="profile:last_name" content="Anady">
<meta property="profile:username" content="thatdeveloperguy">
```

For pages representing a single person. Optional sub properties: `profile:gender` (`male`, `female`, or omit).

**For Bubbles convention: not necessary.** Use `og:type="website"` on practitioner about pages and add Schema.org Person JSON-LD for richer entity signal.

### 13.2 og:type=book

```html
<meta property="og:type" content="book">
<meta property="book:author" content="https://example.com/authors/jane/">
<meta property="book:isbn" content="978-3-16-148410-0">
<meta property="book:release_date" content="2026-03-15">
<meta property="book:tag" content="memoir">
```

For pages representing a single book. Not applicable to any current Bubbles client.

---

## 14. THE MULTI LANGUAGE PATTERN AND MARSHALLESE-VOICES CASE STUDY

Open Graph supports multi language sites through `og:locale` (primary) and `og:locale:alternate` (each additional language).

### 14.1 The Pattern

```html
<!-- On the English page -->
<meta property="og:locale" content="en_US">
<meta property="og:locale:alternate" content="mh_MH">
<meta property="og:locale:alternate" content="es_US">

<!-- On the Marshallese page -->
<meta property="og:locale" content="mh_MH">
<meta property="og:locale:alternate" content="en_US">
<meta property="og:locale:alternate" content="es_US">
```

Each language version declares its own primary locale and lists the others as alternates. Facebook uses this to serve users the preview in their preferred language when available.

### 14.2 The Underscore Notation

Open Graph locale uses underscore (`en_US`), not hyphen (`en-US`). This is the opposite of HTTP Content-Language and html lang (which use hyphen per BCP 47).

**Consistent mistake to avoid:**

```html
<!-- WRONG: hyphen (BCP 47 format, not OG format) -->
<meta property="og:locale" content="en-US">

<!-- CORRECT: underscore (OG format) -->
<meta property="og:locale" content="en_US">
```

### 14.3 The Marshallese-Voices Case Study

Per framework-html-content-language.md Section 10, marshallese-voices.org serves English and Marshallese content at separate URL paths (`/` for English, `/mh/` for Marshallese).

**On the English home page at `marshallese-voices.org/`:**

```html
<head>
    <html lang="en-US">

    <link rel="canonical" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <meta property="og:title" content="Marshallese Voices | Cultural Awareness and Community">
    <meta property="og:type" content="website">
    <meta property="og:image" content="https://marshallese-voices.org/og-image-en.jpg">
    <meta property="og:url" content="https://marshallese-voices.org/">
    <meta property="og:description" content="Building bridges between cultures, sharing stories, and raising awareness about the Marshallese community.">
    <meta property="og:site_name" content="Marshallese Voices">
    <meta property="og:locale" content="en_US">
    <meta property="og:locale:alternate" content="mh_MH">

    <meta property="og:image:width" content="1200">
    <meta property="og:image:height" content="630">
    <meta property="og:image:type" content="image/jpeg">
    <meta property="og:image:alt" content="Marshallese Voices community stories and cultural awareness platform">
</head>
```

**On the Marshallese home page at `marshallese-voices.org/mh/`:**

```html
<head>
    <html lang="mh">

    <link rel="canonical" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <meta property="og:title" content="Ainikien Ri-Majol | Kallapljok Im Jaap Eddo">
    <meta property="og:type" content="website">
    <meta property="og:image" content="https://marshallese-voices.org/og-image-mh.jpg">
    <meta property="og:url" content="https://marshallese-voices.org/mh/">
    <meta property="og:description" content="Kōṃṃan ialin kōtaan ro im aolep kein katak ko im kallapljok kōn ri-Majol jukjuk in pād.">
    <meta property="og:site_name" content="Marshallese Voices">
    <meta property="og:locale" content="mh_MH">
    <meta property="og:locale:alternate" content="en_US">

    <meta property="og:image:width" content="1200">
    <meta property="og:image:height" content="630">
    <meta property="og:image:type" content="image/jpeg">
    <meta property="og:image:alt" content="Ainikien Ri-Majol kōnnaan im kōkajoor jukjuk in pād">
</head>
```

### 14.4 The Separate og:image Per Language

Note that the English page uses `og-image-en.jpg` and the Marshallese page uses `og-image-mh.jpg`. Each language version has a separately authored image with language appropriate text on the card. Reusing the English og:image for the Marshallese page misses the entire point.

### 14.5 The UTF-8 Critical Reminder

Marshallese text in og:title, og:description, and og:image:alt uses diacritics (ā, ē, ī, ō, ū, ṃ, ḷ, ṇ, ṃ̌). Per framework-html-meta-charset.md Section 6, the UTF-8 chain must be end to end: HTML charset, nginx charset, form charset, database collation. Without it, Marshallese OG content renders as mojibake (Ã¤ instead of ä).

### 14.6 The Bubbles Multi Language Convention

For any future Bubbles client requiring multi language support:

1. Each language version declares `og:locale` for itself.
2. Each language version declares `og:locale:alternate` for every other language version.
3. Each language version has a separately authored og:image.
4. og:title, og:description, og:image:alt are all in the target language.
5. UTF-8 chain is verified end to end (framework-html-meta-charset.md).
6. hreflang alternates align with the OG locale set (framework-html-content-language.md).

---

## 15. THE RELATIONSHIP WITH TWITTER CARDS

Twitter (now X) uses its own meta tag protocol called Twitter Cards (`twitter:*`), but the X crawler falls back to Open Graph when Twitter Card tags are absent.

### 15.1 The Fallback Behavior

If a page has `og:title`, `og:description`, `og:image`, and `og:url` but no `twitter:*` tags, X will render a small summary card using the og:* values.

If a page also adds:

```html
<meta name="twitter:card" content="summary_large_image">
```

X renders the large image card using the og:image (1200x600 ideal; 1200x630 acceptable).

### 15.2 The Override Behavior

When both og:* and twitter:* tags are present, twitter:* tags take precedence on X. og:* tags continue to control every other platform.

```html
<!-- OG values used by Facebook, LinkedIn, Slack, etc. -->
<meta property="og:title" content="Heritage Hardwood Floors NWA">
<meta property="og:description" content="Bella Vista hardwood floor refinishing.">
<meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/og-image.jpg">

<!-- Twitter-specific overrides used by X only -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Heritage Hardwood Floors NWA | Bella Vista AR">
<meta name="twitter:description" content="Hardwood refinishing and installation across NWA.">
<meta name="twitter:image" content="https://heritagehardwoodfloorsnwa.com/twitter-card.jpg">
```

### 15.3 The Minimal Twitter Cards Pattern

For Bubbles clients, the minimal pattern is:

```html
<meta name="twitter:card" content="summary_large_image">
```

Just one tag. X uses og:title, og:description, and og:image for the card; the only Twitter specific signal needed is the card type.

For richer Twitter signal (twitter:site, twitter:creator, twitter:image:alt), see framework-html-twitter-cards.md (next framework in track).

### 15.4 The property= vs name= Inconsistency

Open Graph uses `property=`. Twitter Cards uses `name=`. This is a frequent source of confusion. Use each consistently:

```html
<!-- Open Graph -->
<meta property="og:image" content="...">

<!-- Twitter Cards -->
<meta name="twitter:image" content="...">
```

---

## 16. THE RELATIONSHIP WITH SCHEMA.ORG STRUCTURED DATA

Open Graph and Schema.org JSON-LD are complementary, not competing.

### 16.1 The Distinct Purposes

| Open Graph | Schema.org JSON-LD |
|---|---|
| Social preview rendering | Search engine understanding |
| Facebook, LinkedIn, Slack, iMessage | Google, Bing, Yandex, Baidu |
| Image, title, description, locale | Entities, relationships, types, properties |
| Lives in `<meta>` tags | Lives in `<script type="application/ld+json">` |
| Audience: social platforms and humans | Audience: search engines and AI |

Both should be present. They serve different consumers.

### 16.2 The Overlap

Some signals appear in both:

* `og:title` and Schema.org `name` (often identical).
* `og:description` and Schema.org `description` (often identical).
* `og:image` and Schema.org `image` (can be identical or different).
* `article:published_time` and Schema.org `datePublished` (must be identical).
* `article:modified_time` and Schema.org `dateModified` (must be identical).
* `article:author` and Schema.org `author.url` (must be identical).

When dates appear in both Open Graph and Schema.org, they must match. Mismatch confuses Google.

### 16.3 The Bubbles Pattern

For every Bubbles client page:

* Open Graph in `<meta>` tags for social preview.
* Schema.org JSON-LD in `<script>` for search engine understanding (LocalBusiness on the home page and contact page, Service on service pages, Article on blog posts, BreadcrumbList on interior pages).

Both are required. Neither replaces the other.

Covered in framework-html-schema-jsonld.md and the UNIVERSAL-RANKING-FRAMEWORK.md schema library.

---

## 17. THE VALIDATION TOOLS AND CACHE REFRESH

Open Graph implementation requires validation. The default validation tools are free and authoritative.

### 17.1 The Facebook Sharing Debugger

URL: https://developers.facebook.com/tools/debug/

Paste the URL → click "Debug" → review parsed Open Graph data and the rendered preview.

Two critical functions:

* **Scrape**: forces Facebook to re crawl the URL immediately, parsing the current Open Graph state.
* **Show Existing Scrape Information**: displays whatever Facebook has currently cached.

Click "Scrape Again" after any Open Graph change to force Facebook to update its cached version.

Facebook caches Open Graph data for approximately 30 days. Without a debugger scrape, an Open Graph fix can take a month to propagate to actual shares.

### 17.2 The LinkedIn Post Inspector

URL: https://www.linkedin.com/post-inspector/

Paste the URL → click "Inspect" → LinkedIn re crawls and displays the rendered preview.

No LinkedIn login required.

LinkedIn caches Open Graph data for approximately 7 days. The Post Inspector is the only consistently reliable way to force a refresh.

### 17.3 The X (Twitter) Card Validator

URL historically at https://cards-dev.twitter.com/validator was deprecated after Twitter became X. In 2026, the practical validation method is to paste the URL into a private direct message to oneself and observe the preview that renders.

### 17.4 The Third Party Tools

* **opengraph.xyz** — free preview generator showing how the URL renders on Facebook, LinkedIn, X, Slack, Discord, and iMessage in a single view.
* **env.dev social share debugger** — multi platform preview with copyable HTML snippet generation.
* **metatags.io** — preview generator with editable inputs.
* **Slack message preview** — paste the URL into the #scratch channel and observe the unfurl.
* **Discord message preview** — same approach in a private Discord server.

### 17.5 The Cache Refresh Timing By Platform

| Platform | Cache Duration | Refresh Method |
|---|---|---|
| Facebook | ~30 days | Sharing Debugger "Scrape Again" |
| LinkedIn | ~7 days | Post Inspector |
| Slack | 24 to 48 hours | Wait, or append `?v=2` to force new URL |
| Discord | Variable | No official tool; append cache busting query |
| iMessage | Very sticky | Can take days to refresh; cache busting URL is most reliable |
| X | ~7 days | No public tool; re share with cache busting URL |
| WhatsApp | Variable | No official tool; cache busting URL |
| Telegram | Variable | t.me/IamRobot bot, or cache busting URL |

### 17.6 The Cache Busting URL Pattern

When a platform won't refresh:

```
https://heritagehardwoodfloorsnwa.com/services/refinishing/?v=2
```

The query parameter creates a new URL identity from the platform's perspective, forcing a fresh crawl. The og:url tag should still point to the canonical clean URL.

### 17.7 The Bubbles Validation Workflow

After deploying Open Graph changes to a Bubbles client site:

1. `nginx -t && systemctl reload nginx`.
2. curl the page and verify the og:* tags are in the served HTML (server side rendered, not JS injected).
3. Run the Facebook Sharing Debugger and click "Scrape Again".
4. Run the LinkedIn Post Inspector.
5. Share the URL in a private Slack or Discord channel and visually confirm the preview.
6. Document the validation in the client project notes.

---

## 18. THE BUBBLES PER CLIENT DECISION FRAMEWORK

Each client site needs Open Graph decisions.

### 18.1 The Decision Questions

1. **What og:image represents the brand?** Brand colors, logo, type set. Page specific for major pages, generic fallback for the rest.
2. **What og:site_name appears as the site identity?** The client's primary business name as displayed in the logo and footer.
3. **What is the primary og:locale?** `en_US` for every current Bubbles client except Marshallese-Voices.
4. **Is multi language needed?** Marshallese-Voices yes; everyone else no.
5. **For editorial content, what article:* configuration?** Author URL, section taxonomy, tag set.

### 18.2 The Per Client Recommendations

**ThatDeveloperGuy.com:**

```html
<meta property="og:site_name" content="ThatDeveloperGuy">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://thatdeveloperguy.com/og-image.jpg">
<!-- Brand purple and green gradient with agency logo -->
```

For blog posts (og:type=article):

```html
<meta property="article:author" content="https://thatdeveloperguy.com/about/">
<meta property="article:section" content="Web Development">
<!-- Plus article:published_time, article:modified_time, article:tag per post -->
```

**ThatAIGuy.org:**

```html
<meta property="og:site_name" content="ThatAIGuy">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://thataiguy.org/og-image.jpg">
<!-- MEGAMIND aesthetic; AI and SaaS studio brand -->
```

**TCB Fight Factory:**

```html
<meta property="og:site_name" content="TCB Fight Factory">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://tcbfightfactory.com/og-image.jpg">
<!-- Brand purple and black only; no pure black backgrounds -->
```

**Arkansas Counseling and Wellness Services:**

```html
<meta property="og:site_name" content="Arkansas Counseling and Wellness Services">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://arcounselingandwellness.com/og-image.jpg">
<!-- Professional warm brand image; Dr. Kristy Burton's palette -->
```

For YMYL blog posts:

```html
<meta property="article:author" content="https://arcounselingandwellness.com/about/">
<meta property="article:section" content="Mental Health">
<!-- Dr. Kristy Burton credentialed authorship -->
```

**Handled Tax and Advisory (Amanda Emerdinger):**

```html
<meta property="og:site_name" content="Handled Tax & Advisory">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://handledtax.com/og-image.jpg">
<!-- Professional financial palette; conservative trustworthy aesthetic -->
```

For tax content articles:

```html
<meta property="article:author" content="https://handledtax.com/about/">
<meta property="article:section" content="Tax Filing">
<!-- PTIN credentialed; no CPA/EA claims per Amanda's hard guardrail -->
```

The em dash allowance for Amanda's body copy does NOT extend to OG title, OG description, OG image alt, or article:* values. Those follow the standard Bubbles no dashes rule.

**WeCoverUSA:**

```html
<meta property="og:site_name" content="WeCoverUSA">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://wecoverusa.com/og-image.jpg">
<!-- Federal subcontractor appearance; professional polished -->
```

**Eureka Bath Works:**

```html
<meta property="og:site_name" content="Eureka Bath Works">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://eurekabathworks.com/og-image.jpg">
<!-- Brand image showcasing finished bath installations -->
```

**White River Cabins:**

```html
<meta property="og:site_name" content="White River Cabins">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://whiterivercabins.com/og-image.jpg">
<!-- Scenic cabin and river hero image; warm and inviting -->
```

Per cabin page: page specific og:image showing the cabin exterior or signature interior.

**Greenough's Guide Service:**

```html
<meta property="og:site_name" content="Greenough's Guide Service">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://greenoughsguideservice.com/og-image.jpg">
<!-- Trophy fish, guide on the water, or signature client experience -->
```

**Heritage Hardwood Floors NWA (Jessica):**

```html
<meta property="og:site_name" content="Heritage Hardwood Floors NWA">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/og-image.jpg">
<!-- Finished hardwood floor work; warm wood tones -->
```

**Local Living Real Estate (Laycee Maupin):**

```html
<meta property="og:site_name" content="Local Living Real Estate">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://locallivingrealtynwa.com/og-image.jpg">
<!-- NWA home or Laycee professional headshot with brand -->
```

**Diana Undergust (Lake Homes Realty Grand Lake):**

```html
<meta property="og:site_name" content="Diana Undergust | Lake Homes Realty">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://[domain]/og-image.jpg">
<!-- Grand Lake / Grove OK lakefront imagery -->
```

**Homes on Grand:**

```html
<meta property="og:site_name" content="Homes on Grand">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://homesongrand.thatwebhostingguy.com/og-image.jpg">
<!-- Until promoted off the wildcard subdomain; then update og:image and og:url -->
```

**Marshallese-Voices:**

```html
<!-- English page -->
<meta property="og:site_name" content="Marshallese Voices">
<meta property="og:locale" content="en_US">
<meta property="og:locale:alternate" content="mh_MH">
<meta property="og:image" content="https://marshallese-voices.org/og-image-en.jpg">

<!-- Marshallese page -->
<meta property="og:site_name" content="Marshallese Voices">
<meta property="og:locale" content="mh_MH">
<meta property="og:locale:alternate" content="en_US">
<meta property="og:image" content="https://marshallese-voices.org/og-image-mh.jpg">
```

Full multi language pattern per Section 14.

**Avid Pest Control (Rachel Rice):**

```html
<meta property="og:site_name" content="Avid Pest Control">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://avidpests.com/og-image.jpg">
```

**RAH Construction NWA (Steven Hulsey):**

```html
<meta property="og:site_name" content="RAH Construction NWA">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://[domain]/og-image.jpg">
```

**Steele Solutions (Jim and Kim Steele):**

```html
<meta property="og:site_name" content="Steele Solutions">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://steelesolutions.thatwebhostingguy.com/og-image.jpg">
```

**Blue Paradise Dairy (Sara White):**

```html
<meta property="og:site_name" content="Blue Paradise Dairy">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://blueparadisedairy.com/og-image.jpg">
<!-- Product pages: page specific og:image for each product -->
```

**Pretty Clean NWA (Shay Anady):**

```html
<meta property="og:site_name" content="Pretty Clean NWA">
<meta property="og:locale" content="en_US">
<meta property="og:image" content="https://[domain]/og-image.jpg">
```

### 18.3 The Documentation Pattern

For each client, document the OG configuration in the client project notes:

```
Open Graph configuration:
  og:site_name:    [client business name]
  og:locale:       en_US
  og:type default: website
  og:type blog:    article
  og:image:        /og-image.jpg (1200x630 JPG)
  Per page OG:     [yes/no per page type]

  Validation:
    Facebook Sharing Debugger: scraped [date]
    LinkedIn Post Inspector: scraped [date]
    Slack preview: confirmed [date]
```

---

## 19. THE SERVER SIDE RENDERING REQUIREMENT

Open Graph tags must be in the initial HTML response. Crawlers do not execute JavaScript.

### 19.1 The Crawler Behavior

Facebook, LinkedIn, X, Slack, Discord, iMessage, WhatsApp, Telegram, and Pinterest all use server side HTML parsing. None execute JavaScript.

If `<meta property="og:title">` is injected by React, Vue, Svelte, or any client side framework after page load, the crawler sees an empty `<head>` and no preview is generated.

### 19.2 The Static HTML Stack (Bubbles Default)

For Bubbles static HTML clients (the majority): og:* tags live in the `<head>` of every HTML file directly. No JavaScript dependency. This is the simplest and most reliable stack for Open Graph.

### 19.3 The Next.js / React Stack

For Bubbles clients on Next.js (thatdevpro.com, future React projects): og:* tags must be rendered server side via the Next.js Metadata API or via `<Head>` from `next/head` in pages that use server rendering or static generation. Client side `useEffect` Head manipulation will NOT work for OG.

```jsx
// Next.js 14 App Router metadata.ts
export const metadata = {
  title: 'Heritage Hardwood Floors NWA',
  openGraph: {
    title: 'Heritage Hardwood Floors NWA',
    description: 'Bella Vista hardwood floor refinishing.',
    images: [{ url: 'https://example.com/og-image.jpg', width: 1200, height: 630 }],
    type: 'website',
    url: 'https://example.com/',
    siteName: 'Heritage Hardwood Floors NWA',
    locale: 'en_US',
  },
};
```

### 19.4 The SvelteKit / Astro Stacks

Both support server side rendering by default. Place og:* tags in the page level `<svelte:head>` or Astro `<Head>` component; the framework renders them server side.

### 19.5 The Headless Shopify Stack (Blue Paradise Dairy)

The headless frontend on Bubbles must render og:* server side. Hydrogen or a custom Node SSR layer is required; pure client side React (Vite SPA) will not work for Open Graph.

### 19.6 The Verification Method

```bash
curl -s https://example.com/page/ | grep -i 'og:'
```

If the curl output includes the og:* tags, they are server side rendered. If the curl output is empty for og:* (despite the page rendering correctly in a browser), the tags are JS injected and will be invisible to crawlers.

---

## 20. THE IMPLEMENTATION PATTERNS BY STACK

### 20.1 Static HTML (Bubbles Default)

```html
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <title>Heritage Hardwood Floors NWA | Bella Vista AR</title>
    <meta name="description" content="Bella Vista hardwood floor refinishing.">
    <link rel="canonical" href="https://heritagehardwoodfloorsnwa.com/">

    <meta property="og:title" content="Heritage Hardwood Floors NWA | Bella Vista AR">
    <meta property="og:type" content="website">
    <meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/og-image.jpg">
    <meta property="og:url" content="https://heritagehardwoodfloorsnwa.com/">
    <meta property="og:description" content="Bella Vista hardwood floor refinishing.">
    <meta property="og:site_name" content="Heritage Hardwood Floors NWA">
    <meta property="og:locale" content="en_US">
    <meta property="og:image:width" content="1200">
    <meta property="og:image:height" content="630">
    <meta property="og:image:type" content="image/jpeg">
    <meta property="og:image:alt" content="Heritage Hardwood Floors NWA installation in Bella Vista">

    <meta name="twitter:card" content="summary_large_image">
</head>
```

### 20.2 Next.js 14 App Router

```typescript
// app/page.tsx
export const metadata = {
  title: 'Heritage Hardwood Floors NWA | Bella Vista AR',
  description: 'Bella Vista hardwood floor refinishing.',
  alternates: {
    canonical: 'https://heritagehardwoodfloorsnwa.com/',
  },
  openGraph: {
    title: 'Heritage Hardwood Floors NWA | Bella Vista AR',
    description: 'Bella Vista hardwood floor refinishing.',
    url: 'https://heritagehardwoodfloorsnwa.com/',
    siteName: 'Heritage Hardwood Floors NWA',
    locale: 'en_US',
    type: 'website',
    images: [
      {
        url: 'https://heritagehardwoodfloorsnwa.com/og-image.jpg',
        width: 1200,
        height: 630,
        alt: 'Heritage Hardwood Floors NWA installation in Bella Vista',
        type: 'image/jpeg',
      },
    ],
  },
  twitter: {
    card: 'summary_large_image',
  },
};
```

### 20.3 SvelteKit

```svelte
<!-- src/routes/+page.svelte -->
<svelte:head>
    <title>Heritage Hardwood Floors NWA | Bella Vista AR</title>
    <meta name="description" content="Bella Vista hardwood floor refinishing." />
    <link rel="canonical" href="https://heritagehardwoodfloorsnwa.com/" />

    <meta property="og:title" content="Heritage Hardwood Floors NWA | Bella Vista AR" />
    <meta property="og:type" content="website" />
    <meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/og-image.jpg" />
    <meta property="og:url" content="https://heritagehardwoodfloorsnwa.com/" />
    <meta property="og:description" content="Bella Vista hardwood floor refinishing." />
    <meta property="og:site_name" content="Heritage Hardwood Floors NWA" />
    <meta property="og:locale" content="en_US" />
    <meta property="og:image:width" content="1200" />
    <meta property="og:image:height" content="630" />
    <meta property="og:image:type" content="image/jpeg" />
    <meta property="og:image:alt" content="Heritage Hardwood Floors NWA installation in Bella Vista" />

    <meta name="twitter:card" content="summary_large_image" />
</svelte:head>
```

### 20.4 Astro

```astro
---
// src/pages/index.astro
const og = {
  title: 'Heritage Hardwood Floors NWA | Bella Vista AR',
  description: 'Bella Vista hardwood floor refinishing.',
  url: 'https://heritagehardwoodfloorsnwa.com/',
  image: 'https://heritagehardwoodfloorsnwa.com/og-image.jpg',
};
---
<head>
    <title>{og.title}</title>
    <meta name="description" content={og.description} />
    <link rel="canonical" href={og.url} />

    <meta property="og:title" content={og.title} />
    <meta property="og:type" content="website" />
    <meta property="og:image" content={og.image} />
    <meta property="og:url" content={og.url} />
    <meta property="og:description" content={og.description} />
    <meta property="og:site_name" content="Heritage Hardwood Floors NWA" />
    <meta property="og:locale" content="en_US" />
    <meta property="og:image:width" content="1200" />
    <meta property="og:image:height" content="630" />
    <meta property="og:image:type" content="image/jpeg" />
    <meta property="og:image:alt" content="Heritage Hardwood Floors NWA installation in Bella Vista" />
</head>
```

### 20.5 The Nginx Sub Filter Injection (Bulk Backfill)

For backfilling Open Graph across a large number of existing static HTML pages on Bubbles, the FastAPI sidecar on port 9090 with nginx SSI is the right approach (per UNIVERSAL-RANKING-FRAMEWORK.md). Inline injection via `sub_filter` is possible for a single tag, but for the full OG block, FastAPI SSI is cleaner.

For the AdSense pattern already in use on demo and wildcard subdomains, the same nginx sub_filter approach can inject a default og:image:

```nginx
sub_filter '</head>' '<meta property="og:image" content="https://example.com/og-image.jpg"></head>';
sub_filter_once on;
sub_filter_types text/html;
```

End every nginx change with:

```bash
nginx -t && systemctl reload nginx
```

---

## 21. THE PRE LAUNCH AUDIT CHECKLIST

Before a Bubbles client site goes live, verify Open Graph completeness.

### 21.1 The Checklist

* [ ] `og:title` present on every page.
* [ ] `og:type` present on every page (website or article).
* [ ] `og:image` present on every page (1200x630, JPG or PNG, ≤300 KB target).
* [ ] `og:url` present on every page (matching `<link rel="canonical">`).
* [ ] `og:description` present on every page (110 to 200 characters).
* [ ] `og:site_name` present on every page (consistent across the site).
* [ ] `og:locale` present on every page (`en_US` for English clients).
* [ ] `og:image:width` and `og:image:height` declared (`1200` and `630`).
* [ ] `og:image:type` declared (`image/jpeg` or `image/png`).
* [ ] `og:image:alt` declared with descriptive accessibility text.
* [ ] On blog posts: `article:published_time`, `article:author`, `article:section`, `article:tag` declared.
* [ ] `<meta name="twitter:card" content="summary_large_image">` declared.
* [ ] og:url matches `<link rel="canonical">` exactly.
* [ ] og:image is HTTPS, absolute URL, ≥1200x630, ≤8 MB (≤300 KB target).
* [ ] curl on the page returns og:* tags in HTML response (not JS injected).
* [ ] Facebook Sharing Debugger scrape returns clean preview with no warnings.
* [ ] LinkedIn Post Inspector returns clean preview.
* [ ] Slack DM unfurl confirmed visually.
* [ ] Discord DM unfurl confirmed visually (if relevant client channel).
* [ ] iMessage preview confirmed (paste link in self DM).
* [ ] No `name="og:..."` mistakes anywhere (always `property="og:..."`).
* [ ] No relative URLs in `og:image` or `og:url`.
* [ ] No `http://` URLs in `og:image` (always HTTPS).
* [ ] Locale uses underscore (`en_US`), not hyphen.

### 21.2 The Automated Audit

The thatdeveloperguy.com audit pipeline already checks for OG presence. The check should be extended to verify:

* OG presence (boolean).
* OG completeness (all 4 required + 3 recommended + image sub properties).
* og:url matches canonical (boolean).
* og:image dimensions ≥ 1200x630 (HEAD request to image URL + image size check).
* og:image file size ≤ 8 MB.

For sites with 100+ pages, automated auditing is the only viable approach.

---

## 22. THE COMMON FAILURES AND FIXES

### 22.1 "Facebook is showing the wrong image"

Cause 1: Facebook cached an old version of the page before og:image was added.
Fix: Run the Facebook Sharing Debugger and click "Scrape Again".

Cause 2: Another image on the page is larger than og:image and Facebook picked it instead.
Fix: Make og:image the largest image on the page, or hide the competing image from crawlers.

Cause 3: og:image URL is relative or HTTP.
Fix: Use absolute HTTPS URL.

### 22.2 "LinkedIn preview shows no description"

Cause 1: og:description is missing.
Fix: Add `<meta property="og:description" content="...">`.

Cause 2: LinkedIn cached the page before og:description was added.
Fix: Run the LinkedIn Post Inspector.

### 22.3 "iMessage shows just the URL with no preview"

Cause 1: og:image is missing, broken, or too small.
Fix: Verify og:image is present, HTTPS, 1200x630, accessible.

Cause 2: iMessage cached an old failed crawl.
Fix: Append `?v=2` to the URL when sharing to force a new crawl.

### 22.4 "The preview shows the wrong page on our site"

Cause: og:url is missing or points to a different page.
Fix: Ensure og:url on every page points to its own canonical URL.

### 22.5 "Open Graph works on Chrome devtools but not on Facebook"

Cause: og:* tags are injected by client side JavaScript.
Fix: Move OG tags to server side rendered HTML. Verify with `curl -s [url] | grep og:`.

### 22.6 "Some pages show preview, others don't"

Cause: Inconsistent OG implementation across page templates.
Fix: Audit every template type (home, service, location, blog, contact) and ensure OG is in each.

### 22.7 "Facebook share count is split across two values"

Cause: og:url and the actual shared URL differ (trailing slash, www vs non www, http vs https).
Fix: Align og:url with the canonical host and trailing slash policy.

### 22.8 "WhatsApp shows broken image"

Cause: og:image file is too large (over ~300 KB).
Fix: Compress the og:image. Target 100 to 250 KB.

### 22.9 "Article preview shows generic 'just now' instead of publish date"

Cause: `article:published_time` is missing or in wrong format.
Fix: Add ISO 8601 datetime with timezone offset (`2026-05-15T08:00:00-05:00`).

### 22.10 "Cloudflare proxy is breaking OG preview"

Not applicable to Bubbles (no Cloudflare or third party CDN in front). Per Joseph's standing rule, Bubbles is direct origin. This is mentioned only because the failure pattern is common in the broader industry.

---

## 23. THE FEDERAL AND SDVOSB APPEARANCE CONTEXT

For Joseph's SDVOSB federal subcontracting work, Open Graph carries professional appearance weight beyond the consumer marketing use case.

### 23.1 The Procurement Officer Use Case

Federal procurement officers and contracting officers regularly share vendor URLs on LinkedIn (internally and to peers) and in agency messaging systems. A vendor URL that produces a polished preview signals operational maturity. A vendor URL that produces a bare hyperlink or a broken preview signals an unfinished web presence.

### 23.2 The ThatDeveloperGuy SDVOSB OG Pattern

```html
<meta property="og:site_name" content="ThatDeveloperGuy | SDVOSB Certified">
<meta property="og:image" content="https://thatdeveloperguy.com/og-image-sdvosb.jpg">
<meta property="og:image:alt" content="ThatDeveloperGuy SDVOSB certified web development agency, Cassville Missouri">
```

The og:image for SDVOSB context should include:

* Agency name and logo.
* SDVOSB certification mark or text.
* Professional palette consistent with brand guidelines.
* No frivolous or playful imagery.

### 23.3 The SAM.gov Profile Consistency

The og:site_name and og:description should be consistent with the SAM.gov registered business name and description. Procurement officers cross reference.

### 23.4 The WeCoverUSA Federal Appearance Pattern

```html
<meta property="og:site_name" content="WeCoverUSA">
<meta property="og:image" content="https://wecoverusa.com/og-image.jpg">
<meta property="og:image:alt" content="WeCoverUSA F&I vertical solutions for franchised dealers nationwide">
```

The og:image should signal:

* National scope (not regional).
* Polished commercial brand.
* No personal photography.
* Logo and tagline visible.

### 23.5 The Documentation Note

For federal context client work, document the OG decision in the project notes alongside the SAM.gov and SDVOSB compliance notes. Procurement officers may screenshot LinkedIn previews as part of vendor diligence files.

---

## 24. QUICK REFERENCE TAG BLOCK

For copy paste into any Bubbles client page `<head>`:

### 24.1 Marketing Page (og:type=website)

```html
<!-- Open Graph -->
<meta property="og:title" content="PAGE TITLE">
<meta property="og:type" content="website">
<meta property="og:image" content="https://CLIENT.com/og-image.jpg">
<meta property="og:url" content="https://CLIENT.com/PAGE/">
<meta property="og:description" content="PAGE DESCRIPTION (110 to 200 chars).">
<meta property="og:site_name" content="CLIENT BUSINESS NAME">
<meta property="og:locale" content="en_US">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:type" content="image/jpeg">
<meta property="og:image:alt" content="DESCRIPTIVE ALT TEXT FOR ACCESSIBILITY">

<!-- Twitter Cards (X uses og:* as fallback) -->
<meta name="twitter:card" content="summary_large_image">
```

### 24.2 Blog Post (og:type=article)

```html
<!-- Open Graph -->
<meta property="og:title" content="ARTICLE TITLE">
<meta property="og:type" content="article">
<meta property="og:image" content="https://CLIENT.com/blog/og-ARTICLE-SLUG.jpg">
<meta property="og:url" content="https://CLIENT.com/blog/ARTICLE-SLUG/">
<meta property="og:description" content="ARTICLE DESCRIPTION (110 to 200 chars).">
<meta property="og:site_name" content="CLIENT BUSINESS NAME">
<meta property="og:locale" content="en_US">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:type" content="image/jpeg">
<meta property="og:image:alt" content="DESCRIPTIVE ALT TEXT FOR ARTICLE OG IMAGE">

<!-- Article subtype -->
<meta property="article:published_time" content="2026-MM-DDTHH:MM:SS-05:00">
<meta property="article:modified_time" content="2026-MM-DDTHH:MM:SS-05:00">
<meta property="article:author" content="https://CLIENT.com/about/">
<meta property="article:section" content="EDITORIAL SECTION">
<meta property="article:tag" content="TAG ONE">
<meta property="article:tag" content="TAG TWO">
<meta property="article:tag" content="TAG THREE">

<!-- Twitter Cards -->
<meta name="twitter:card" content="summary_large_image">
```

### 24.3 Multi Language Page (Marshallese-Voices Pattern)

```html
<!-- On English page -->
<meta property="og:locale" content="en_US">
<meta property="og:locale:alternate" content="mh_MH">

<!-- On Marshallese page -->
<meta property="og:locale" content="mh_MH">
<meta property="og:locale:alternate" content="en_US">
```

---

## 25. REFERENCES

* The Open Graph protocol specification: https://ogp.me
* The article subtype properties: https://ogp.me/#type_article
* The video subtype properties: https://ogp.me/#type_video
* The music subtype properties: https://ogp.me/#type_music
* The profile subtype properties: https://ogp.me/#type_profile
* The book subtype properties: https://ogp.me/#type_book
* Facebook Sharing Debugger: https://developers.facebook.com/tools/debug/
* Facebook Sharing Best Practices: https://developers.facebook.com/docs/sharing/best-practices/
* LinkedIn Post Inspector: https://www.linkedin.com/post-inspector/
* LinkedIn Making Content Shareable: https://www.linkedin.com/help/linkedin/answer/a521928
* Slack Unfurling Links in Messages: https://api.slack.com/reference/messaging/link-unfurling
* opengraph.xyz preview tool: https://www.opengraph.xyz/
* env.dev OG guide: https://env.dev/guides/opengraph

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
* framework-html-content-language.md
* framework-html-meta-refresh.md
* framework-html-twitter-cards.md (next)

**Parent reference:** UNIVERSAL-RANKING-FRAMEWORK.md (the 27 section master document).

---

*End of framework-html-open-graph.md.*
