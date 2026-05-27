# framework-html-twitter-cards.md

Comprehensive reference for the Twitter Cards (now X Cards) protocol, the metadata system that controls how URLs are represented when shared on X. Covers the four card types (`summary`, `summary_large_image`, `app`, `player`), the brand and author attribution tags (`twitter:site`, `twitter:creator`), the content override tags (`twitter:title`, `twitter:description`, `twitter:image`, `twitter:image:alt`), the URL tag (`twitter:url`), the player card properties (`twitter:player`, `twitter:player:width`, `twitter:player:height`, `twitter:player:stream`), the app card properties (`twitter:app:id:iphone`, `twitter:app:id:googleplay`, etc.), the critical fallback relationship with Open Graph (X uses og:* tags when twitter:* tags are absent, so the minimal Twitter Cards implementation is a single `twitter:card` declaration), the `name=` attribute mechanism (opposite of Open Graph's `property=`), the image specifications (1200x628 for `summary_large_image`, 144x144 minimum for `summary`, 5 MB file size cap, JPG/PNG/WebP/GIF formats), the deprecated official Card Validator at cards-dev.twitter.com (shut down in 2022 with no replacement from X), the third party validation tools that filled the gap, the X cache behavior (approximately 7 days, no manual purge available), and the Bubbles per client decision framework with the minimal `summary_large_image` plus brand `@thatdeveloperguy` attribution pattern as the default. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the fifteenth framework in the HTML signal track**, following the meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, color-scheme, referrer, content-language, refresh, and Open Graph frameworks. Direct companion to framework-html-open-graph.md (Twitter Cards is the X specific layer on top of Open Graph).

Audience: humans implementing X link previews for client sites, AI assistants generating HTML head sections with appropriate Twitter Cards alongside Open Graph, designers preserving brand image consistency across Facebook (og:image) and X (twitter:image), Bubbles operators making the minimal versus full Twitter Cards decision per client, federal subcontractors ensuring polished X previews when SDVOSB context is shared on the platform, and anyone troubleshooting "X is showing the small summary card instead of the large image", "the wrong handle is being credited", "X stripped the image entirely", or "the card preview is from a previous version of the page".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters (And Why It Matters Less In 2026)
3. What This Covers
4. The Mental Model: One Tag Minimum, Full Set For Control
5. The name= Attribute Mechanism (Vs Open Graph's property=)
6. The Four Card Types
7. The Brand And Author Attribution Tags
8. The Content Override Tags
9. The twitter:image And twitter:image:alt
10. The twitter:url Tag
11. The Player Card Properties
12. The App Card Properties
13. The Open Graph Fallback Relationship
14. The Deprecated Validator And Third Party Replacements
15. The X Cache Behavior
16. The Bubbles Per Client Decision Framework
17. The Server Side Rendering Requirement
18. The Implementation Patterns By Stack
19. The Pre Launch Audit Checklist
20. The Common Failures And Fixes
21. The Federal And SDVOSB Appearance Context On X
22. Quick Reference Tag Block
23. References

---

## 1. DEFINITION

Twitter Cards (rebranded as X Cards after the 2023 platform rename, though "Twitter Cards" remains the universal developer term) is a metadata protocol introduced by Twitter in 2012 that uses `<meta name="twitter:*">` tags in the HTML `<head>` to control how a URL is represented when shared on X.

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@thatdeveloperguy">
<meta name="twitter:creator" content="@thatdeveloperguy">
<meta name="twitter:title" content="Page Title Goes Here">
<meta name="twitter:description" content="Brief description of the page.">
<meta name="twitter:image" content="https://example.com/twitter-card.jpg">
<meta name="twitter:image:alt" content="Descriptive alt text for accessibility">
```

The protocol uses the HTML5 `name=` attribute (not RDFa `property=`), with the `twitter:` namespace prefix. The specification has not been meaningfully updated since 2017. X's official Card Validator at cards-dev.twitter.com was deprecated in 2022 and has not been replaced.

When a user posts a URL on X, the X crawler fetches the page, parses the `twitter:*` tags, and renders a preview card. When `twitter:*` tags are missing, X falls back to the corresponding Open Graph (`og:*`) values. This fallback behavior is the most important practical fact about Twitter Cards in 2026: if Open Graph is implemented correctly per framework-html-open-graph.md, the only Twitter Cards tag strictly required is `twitter:card` to declare the desired card type.

---

## 2. WHY IT MATTERS (AND WHY IT MATTERS LESS IN 2026)

Twitter Cards mattered enormously from 2012 to 2022 when X (then Twitter) was a primary distribution channel for content marketing, blog posts, and news. In 2026 the protocol still matters, but with three significant context shifts:

**Shift 1: The X platform changes.** After the 2022 ownership change and 2023 rename, X's reach for unpaid organic content has declined materially. Many of Joseph's NWA and SWMO target demographics use Facebook, Instagram, and Nextdoor far more than X. For local service businesses (Heritage Hardwood Floors NWA, Arkansas Counseling, Greenough's Guide Service, White River Cabins), X is rarely a meaningful distribution channel.

**Shift 2: The validator deprecation.** X shut down the official Card Validator at cards-dev.twitter.com in 2022 with no replacement. Third party tools fill the gap but cannot force X to re crawl a page on demand. Cache management is harder than for Facebook.

**Shift 3: The fallback to Open Graph.** Because X falls back to `og:*` tags when `twitter:*` tags are absent, the minimal correct implementation is a single `twitter:card` declaration. Everything else can ride on the Open Graph implementation already in place.

**Where Twitter Cards still matters:**

* **Federal subcontracting context.** Federal officials, defense industry analysts, government affairs professionals, and SDVOSB community members are concentrated on X. A WeCoverUSA or ThatDeveloperGuy SDVOSB URL shared on X should produce a polished card with the correct attribution handle.
* **Tech industry visibility.** ThatAIGuy (MEGAMIND), academic AI research outreach, and AI Twitter community engagement all happen on X. Tech professionals share URLs on X far more than on Facebook.
* **News and editorial.** Blog posts, articles, case studies still propagate on X among professional audiences. The article subtype card with proper attribution is meaningful.
* **Brand attribution.** `twitter:site` and `twitter:creator` credit the brand and author handles on the preview card. This is the only way to attach handle attribution to a shared link.

**Where Twitter Cards matters little:**

* Local service business websites whose customers are not on X (the majority of Joseph's NWA and SWMO clients).
* Pages that already have complete Open Graph and don't need X specific overrides.

For Bubbles convention: implement the minimal Twitter Cards block (single `twitter:card` plus optional `twitter:site` brand handle) on every client page. Add the full content override block only where X is a relevant distribution channel for that client.

---

## 3. WHAT THIS COVERS

This framework covers:

* The four card types: `summary` (small thumbnail with title and description), `summary_large_image` (large image above title and description, the most useful), `app` (mobile app promotion), `player` (inline video or audio player).
* The brand and author attribution tags: `twitter:site` (the site's X handle) and `twitter:creator` (the author's X handle).
* The content override tags: `twitter:title`, `twitter:description`, `twitter:image`, `twitter:image:alt`, `twitter:url`.
* The player card properties: `twitter:player`, `twitter:player:width`, `twitter:player:height`, `twitter:player:stream`, `twitter:player:stream:content_type`.
* The app card properties: `twitter:app:id:iphone`, `twitter:app:id:ipad`, `twitter:app:id:googleplay`, `twitter:app:name:*`, `twitter:app:url:*`, `twitter:app:country`.
* The image specifications: 1200x628 for `summary_large_image` (1.91:1, 5 MB max), 144x144 minimum for `summary` (1:1, 4096x4096 max).
* The supported file formats: JPG, PNG, WebP, GIF (first frame only).
* The `name="twitter:..."` attribute mechanism (opposite of Open Graph's `property="og:..."`).
* The Open Graph fallback relationship: X uses og:title, og:description, og:image, og:url when their twitter:* counterparts are absent.
* The deprecated validator history: cards-dev.twitter.com shut down in 2022, no X replacement, third party tools fill the gap.
* The X cache behavior: approximately 7 days, no manual purge available, cache busting URL parameter is the only practical refresh method.
* The silent card type downgrade: X downgrades `summary_large_image` to `summary` when the image is below 300x157 pixels.
* The Bubbles per client decision framework with minimal versus full Twitter Cards configuration per client.
* The federal and SDVOSB context: handle attribution matters when government affairs and defense analysts share URLs.

This framework does NOT cover:

* Open Graph protocol — covered in framework-html-open-graph.md (the parent layer).
* Schema.org JSON-LD — covered in framework-html-schema-jsonld.md.
* X advertising or organic posting strategy.
* X API or Tweet embedding (different system from Cards).

---

## 4. THE MENTAL MODEL: ONE TAG MINIMUM, FULL SET FOR CONTROL

Twitter Cards implementation follows a three layer escalation model:

```
LEVEL 1: Minimum (1 tag, relies entirely on Open Graph fallback)
   |
   |---> twitter:card = summary_large_image
         (X uses og:title, og:description, og:image, og:url)

LEVEL 2: Brand attribution (3 tags, adds X handle credit)
   |
   |---> twitter:card        = summary_large_image
   |---> twitter:site        = @client_handle  (the brand)
   |---> twitter:creator     = @author_handle  (the author or owner)

LEVEL 3: Full override (8 tags, complete control over the X preview)
   |
   |---> twitter:card         = summary_large_image
   |---> twitter:site         = @client_handle
   |---> twitter:creator      = @author_handle
   |---> twitter:url          = https://...
   |---> twitter:title        = X specific title
   |---> twitter:description  = X specific description
   |---> twitter:image        = X specific image
   |---> twitter:image:alt    = accessibility text
```

Six rules govern the system:

1. **Always include `twitter:card`** to declare the desired card type. Without it, X may default to the small `summary` card or skip the preview.
2. **Rely on Open Graph fallback** for title, description, image, and URL unless an X specific override is needed.
3. **Use `twitter:site` and `twitter:creator`** when the client and author have active X handles.
4. **Match image specs to card type**: `summary_large_image` needs 1200x628 minimum; `summary` needs 144x144 minimum.
5. **Server side render the tags** (X crawler does not execute JavaScript).
6. **Test via third party validators** since X's official validator is deprecated.

A correctly configured Bubbles client site has at minimum `twitter:card="summary_large_image"` on every page, plus `twitter:site` when the client has an active X handle, and the full override block on pages where X is a meaningful distribution channel.

---

## 5. THE NAME= ATTRIBUTE MECHANISM (VS OPEN GRAPH'S PROPERTY=)

Twitter Cards uses `name=` while Open Graph uses `property=`. This is the single most common mistake when implementing both side by side.

### 5.1 The Distinction

```html
<!-- Twitter Cards: name= -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Page Title">

<!-- Open Graph: property= -->
<meta property="og:title" content="Page Title">
<meta property="og:type" content="website">
```

Both protocols use the `<meta>` element with the `content` attribute. The first attribute differs:

* Twitter Cards: `name="twitter:..."` (standard HTML5 meta name convention).
* Open Graph: `property="og:..."` (RDFa namespace convention).

### 5.2 The Mistake To Avoid

Using `property=` for Twitter Cards or `name=` for Open Graph:

```html
<!-- WRONG: property attribute on Twitter Cards -->
<meta property="twitter:card" content="summary_large_image">

<!-- CORRECT: name attribute on Twitter Cards -->
<meta name="twitter:card" content="summary_large_image">

<!-- WRONG: name attribute on Open Graph -->
<meta name="og:title" content="Page Title">

<!-- CORRECT: property attribute on Open Graph -->
<meta property="og:title" content="Page Title">
```

The X crawler is lenient and accepts `property="twitter:..."` in many cases, but the spec is `name=`. Some validation tools flag the wrong attribute. Use each consistently.

### 5.3 The Mental Trick To Remember

* **og** stands for Open Graph, which uses RDFa **property**. "OG = property."
* **twitter** uses the standard HTML5 **name**. "twitter = name."

Or more simply: **og:** goes with `property=`, everything else goes with `name=`.

---

## 6. THE FOUR CARD TYPES

X supports four card types. The `twitter:card` tag selects which one to use.

### 6.1 summary

```html
<meta name="twitter:card" content="summary">
```

Small square thumbnail (1:1) displayed to the left of the title and description.

* **Image dimensions**: minimum 144x144, maximum 4096x4096, aspect ratio 1:1.
* **File size**: 5 MB maximum.
* **Supported formats**: JPG, PNG, WebP, GIF (first frame).
* **Use case**: blog posts and articles where the brand logo or a portrait thumbnail is more appropriate than a wide hero image.

The `summary` card is the default if `twitter:card` is missing.

**Bubbles usage**: rarely chosen. Most client content benefits from `summary_large_image`.

### 6.2 summary_large_image

```html
<meta name="twitter:card" content="summary_large_image">
```

Large image displayed above the title and description, taking up significant feed real estate.

* **Image dimensions**: 1200x628 recommended (1.91:1 aspect ratio), 300x157 minimum.
* **File size**: 5 MB maximum.
* **Supported formats**: JPG, PNG, WebP, GIF.
* **Critical**: below 300x157, X silently downgrades to `summary` regardless of the `twitter:card` declaration.

**Use case**: marketing pages, landing pages, product pages, blog posts with a brand image, virtually everything where visual impact matters.

**Bubbles default**: `summary_large_image` on every page of every client site.

### 6.3 player

```html
<meta name="twitter:card" content="player">
<meta name="twitter:player" content="https://example.com/embed/video123">
<meta name="twitter:player:width" content="1280">
<meta name="twitter:player:height" content="720">
<meta name="twitter:image" content="https://example.com/video-poster.jpg">
```

Inline video or audio player embedded directly in the X feed.

* **Player URL**: HTTPS only, must serve a valid embeddable player (HTML5 video or iframe).
* **Image**: poster frame, 1200x628 recommended.
* **Approval required**: player cards require X to approve the player domain before they render. Most third party submissions are not approved in 2026.

**Use case**: media platforms with proprietary players (YouTube, Vimeo, Spotify, SoundCloud). Bubbles clients should not attempt to implement player cards directly; instead, link to the YouTube or Vimeo URL where the host platform already has player card approval.

**Bubbles usage**: never. Always link to the third party host (YouTube, Vimeo) and let their player card take over.

### 6.4 app

```html
<meta name="twitter:card" content="app">
<meta name="twitter:app:id:iphone" content="307234931">
<meta name="twitter:app:id:ipad" content="307234931">
<meta name="twitter:app:id:googleplay" content="com.example.app">
<meta name="twitter:app:name:iphone" content="Example App">
<meta name="twitter:app:name:ipad" content="Example App">
<meta name="twitter:app:name:googleplay" content="Example App">
<meta name="twitter:app:url:iphone" content="examplapp://">
<meta name="twitter:app:url:ipad" content="examplapp://">
<meta name="twitter:app:url:googleplay" content="examplapp://">
<meta name="twitter:app:country" content="US">
```

Mobile app promotion card with direct "Install" or "Open" buttons linking to the App Store and Google Play.

**Use case**: app developers promoting iOS or Android apps.

**Bubbles usage**: not applicable to any current client. Skip.

### 6.5 The Selection Decision

For Bubbles client sites, the selection is simple:

* Every page: `summary_large_image`.
* No exceptions in current client portfolio.

For future clients with podcast or video first content: still `summary_large_image` with a strong poster frame. Player cards are not practically obtainable.

---

## 7. THE BRAND AND AUTHOR ATTRIBUTION TAGS

Two tags attach X handle attribution to the preview card. Both are optional but valuable when present.

### 7.1 twitter:site

```html
<meta name="twitter:site" content="@thatdeveloperguy">
```

The X handle of the website or brand. Displayed on the preview card as the source attribution.

* **Format**: `@` followed by the handle (no spaces, no URL).
* **Required for handle attribution**: without it, X shows the domain but no handle credit.
* **Single value**: one handle per site.

**Bubbles convention**: include `twitter:site` on every client that has an active X handle. Skip on clients without an X presence.

### 7.2 twitter:creator

```html
<meta name="twitter:creator" content="@thatdeveloperguy">
```

The X handle of the individual author of the page. Used on article and editorial content to credit the writer separately from the brand.

* **Format**: `@` followed by the handle.
* **Use case**: blog posts where the author has their own X presence distinct from the brand.

**Bubbles convention**: include `twitter:creator` on blog posts and editorial content when the author has an X handle. On marketing pages where there is no individual author, set `twitter:creator` equal to `twitter:site` or omit entirely.

### 7.3 The Handle Discipline

* Always start with `@`.
* No URL form: `@handle`, not `https://twitter.com/handle` or `https://x.com/handle`.
* Lowercase or mixed case is fine; handles are case insensitive on X.
* Verify the handle exists and is currently held by the intended party. Handles that have been recycled to other users create the wrong attribution.

### 7.4 The Per Client Handle Map

For Bubbles client sites with known X handles:

| Client | twitter:site | twitter:creator |
|---|---|---|
| ThatDeveloperGuy.com | `@thatdeveloperguy` (verify) | `@thatdeveloperguy` |
| ThatAIGuy.org | `@thataiguy` (verify) | `@thataiguy` |
| TCB Fight Factory | client handle if active | client handle |
| All other clients | only if client confirms active X handle | only if author confirmed |

Do not invent or guess handles. Omit the tag if the handle is unknown or inactive.

---

## 8. THE CONTENT OVERRIDE TAGS

Four tags override the Open Graph fallback content with X specific values. Use them only when X specific content differs from the OG content.

### 8.1 twitter:title

```html
<meta name="twitter:title" content="X Specific Headline">
```

Overrides og:title on the X card.

* **Length**: 70 characters or less. X truncates harder than Facebook.
* **Use case**: when the X audience expects a different framing than the Facebook or LinkedIn audience.

**Bubbles convention**: omit. Let og:title handle X via fallback. Override only on the rare page where X needs different copy.

### 8.2 twitter:description

```html
<meta name="twitter:description" content="X specific description text.">
```

Overrides og:description on the X card.

* **Length**: 200 characters or less.
* **Use case**: same as twitter:title; X specific framing.

**Bubbles convention**: omit. Let og:description handle X via fallback.

### 8.3 twitter:image

```html
<meta name="twitter:image" content="https://example.com/twitter-card.jpg">
```

Overrides og:image on the X card.

* **Dimensions**: 1200x628 ideal (1.91:1).
* **Format**: HTTPS absolute URL.
* **Use case**: when a different image is needed for X (e.g., the og:image is sized 1200x630 and gets a few pixels cropped on X, and a separate 1200x628 version exists).

**Bubbles convention**: omit. The 1200x630 og:image works cleanly on X. Override only if a content team has authored a separate twitter:image asset.

### 8.4 twitter:image:alt

```html
<meta name="twitter:image:alt" content="Descriptive alt text for the X card image">
```

Accessibility description for the X card image. Used by screen readers and X's own accessibility tools.

* **Length**: 420 characters or less (X's stated limit).
* **Required for federal subcontracting context** (Section 508 accessibility).

**Bubbles convention**: include when `twitter:image` is explicitly set. When relying on og:image fallback, X uses og:image:alt automatically.

---

## 9. THE TWITTER:IMAGE AND TWITTER:IMAGE:ALT

The image is the single most impactful element of the X card. Detailed specs:

### 9.1 The Dimensions By Card Type

| Card Type | Recommended | Minimum | Aspect Ratio |
|---|---|---|---|
| `summary` | 400x400 | 144x144 | 1:1 (square) |
| `summary_large_image` | 1200x628 | 300x157 | 1.91:1 |
| `player` (poster) | 1200x628 | 300x157 | 1.91:1 |

### 9.2 The Silent Downgrade

X silently downgrades `summary_large_image` to `summary` when the supplied image is below 300x157 pixels. There is no error message. The card simply renders smaller than expected.

**For Bubbles convention**: never ship a twitter:image (or og:image used as fallback) smaller than 1200x628.

### 9.3 The File Size And Format

* **Maximum file size**: 5 MB.
* **Supported formats**: JPG, PNG, WebP, GIF (first frame static on X).
* **Target file size**: 100 to 300 KB for fast rendering.

### 9.4 The Aspect Ratio Tolerance

X accepts the standard og:image dimensions of 1200x630 (1.91:1 nominal) without complaint, though strict X spec is 1200x628 (slightly different decimal). The few pixel difference is invisible in practice; reuse the 1200x630 og:image for X.

### 9.5 The HTTPS Requirement

The image URL must be HTTPS. HTTP image URLs are silently dropped from X cards.

### 9.6 The Same Image As og:image Pattern

For 99% of Bubbles client pages, the optimal pattern is:

```html
<!-- og:image is the single source of truth -->
<meta property="og:image" content="https://heritagehardwoodfloorsnwa.com/og-image.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="Heritage Hardwood Floors NWA installation work">

<!-- X uses the og:image via fallback; only twitter:card is needed -->
<meta name="twitter:card" content="summary_large_image">
```

No `twitter:image` tag at all. X uses og:image automatically.

### 9.7 The Separate twitter:image Pattern (Rare)

For the rare case where a client wants different visual treatment on X:

```html
<meta name="twitter:image" content="https://example.com/twitter-card.jpg">
<meta name="twitter:image:alt" content="X specific card image alt text">
```

When using a separate twitter:image, twitter:image:alt should also be supplied.

---

## 10. THE TWITTER:URL TAG

```html
<meta name="twitter:url" content="https://example.com/page/">
```

Overrides og:url for X card identity.

In practice, og:url and twitter:url should always match. There is no useful scenario for distinct X versus general canonical URLs.

**Bubbles convention**: omit twitter:url. Let X use og:url via fallback.

---

## 11. THE PLAYER CARD PROPERTIES

For completeness, the player card properties:

```html
<meta name="twitter:card" content="player">
<meta name="twitter:site" content="@thatdeveloperguy">
<meta name="twitter:title" content="Video Title">
<meta name="twitter:description" content="Video description.">
<meta name="twitter:image" content="https://example.com/poster.jpg">
<meta name="twitter:player" content="https://example.com/embed/video123">
<meta name="twitter:player:width" content="1280">
<meta name="twitter:player:height" content="720">
<meta name="twitter:player:stream" content="https://example.com/stream.mp4">
<meta name="twitter:player:stream:content_type" content="video/mp4">
```

X requires manual approval for the player domain before player cards render. Approval is rarely granted in 2026 outside of established media platforms.

**For Bubbles convention**: do not implement player cards directly. Link to the YouTube, Vimeo, or other host platform URL and let their already approved player card render.

---

## 12. THE APP CARD PROPERTIES

For completeness, the app card properties:

```html
<meta name="twitter:card" content="app">
<meta name="twitter:site" content="@thatdeveloperguy">
<meta name="twitter:description" content="App description.">
<meta name="twitter:app:country" content="US">

<meta name="twitter:app:name:iphone" content="App Name">
<meta name="twitter:app:id:iphone" content="307234931">
<meta name="twitter:app:url:iphone" content="customscheme://">

<meta name="twitter:app:name:ipad" content="App Name">
<meta name="twitter:app:id:ipad" content="307234931">
<meta name="twitter:app:url:ipad" content="customscheme://">

<meta name="twitter:app:name:googleplay" content="App Name">
<meta name="twitter:app:id:googleplay" content="com.example.app">
<meta name="twitter:app:url:googleplay" content="customscheme://">
```

For Bubbles clients: not applicable. Skip.

---

## 13. THE OPEN GRAPH FALLBACK RELATIONSHIP

This is the single most important fact about Twitter Cards in 2026: X falls back to Open Graph when twitter:* tags are absent. The fallback map:

| Twitter Cards tag (when absent) | X uses this Open Graph tag instead |
|---|---|
| `twitter:title` | `og:title` |
| `twitter:description` | `og:description` |
| `twitter:image` | `og:image` |
| `twitter:image:alt` | `og:image:alt` |
| `twitter:url` | `og:url` |

The only Twitter Cards tags with no Open Graph equivalent:

* `twitter:card` — declares card type. No equivalent in Open Graph. **Required for X**.
* `twitter:site` — X handle of the brand. No equivalent in Open Graph.
* `twitter:creator` — X handle of the author. Closest OG equivalent is `article:author` URL, but X does not use it.

### 13.1 The Minimal Pattern (Open Graph Plus Card Type)

When Open Graph is correctly implemented per framework-html-open-graph.md, the entire Twitter Cards implementation reduces to:

```html
<meta name="twitter:card" content="summary_large_image">
```

That's it. One tag. X uses og:title, og:description, og:image, and og:url for the card content.

### 13.2 The Brand Attribution Pattern (Three Tags)

When the client has an X handle to credit:

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@client_handle">
<meta name="twitter:creator" content="@client_handle">
```

### 13.3 The Full Override Pattern (Eight Tags)

When X content needs to differ from Facebook and LinkedIn content:

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@thatdeveloperguy">
<meta name="twitter:creator" content="@thatdeveloperguy">
<meta name="twitter:url" content="https://example.com/page/">
<meta name="twitter:title" content="X specific title">
<meta name="twitter:description" content="X specific description.">
<meta name="twitter:image" content="https://example.com/twitter-card.jpg">
<meta name="twitter:image:alt" content="X card image alt text">
```

### 13.4 The Bubbles Pattern Decision

For each Bubbles client:

* **No X handle, no special X audience**: minimal pattern (1 tag).
* **Has X handle, no special X audience**: brand attribution pattern (3 tags).
* **Has X handle, X is a meaningful channel** (ThatDeveloperGuy, ThatAIGuy, federal context): brand attribution (3 tags), expanding to full override only when a specific page needs distinct X copy.

---

## 14. THE DEPRECATED VALIDATOR AND THIRD PARTY REPLACEMENTS

X's official Card Validator at cards-dev.twitter.com was deprecated in 2022 and has not been replaced.

### 14.1 The Deprecation History

* **2012**: Twitter introduces Cards and the Card Validator at cards-dev.twitter.com.
* **2017**: Last meaningful protocol update.
* **2022**: Validator deprecated as part of broader API and tool cuts under new platform ownership. No official replacement.
* **2023**: Twitter rebranded as X. Validator URL still loads but no longer shows previews.
* **2026**: No official tool. Third party tools fill the gap. Cache management is now manual.

### 14.2 The Third Party Replacements

In 2026, the practical Twitter Cards validation options:

* **opengraph.xyz** — shows the X preview alongside Facebook, LinkedIn, Slack, Discord, iMessage. Free, no login.
* **socialpreviewhub.com** — dedicated X card preview tool.
* **socialrails.com X card validator** — preview generator.
* **env.dev social share debugger** — multi platform preview including X.
* **opentweet.io X card validator** — preview with editable inputs.
* **metatags.io** — preview generator including X.

None of these can force X's cache to refresh. They only show how the card *would* render based on the current meta tags.

### 14.3 The Ground Truth Validation

The only reliable ground truth for X card rendering in 2026:

1. Post the URL to a private X account or in a DM to yourself.
2. Observe the rendered card.
3. If wrong, fix the tags, append a cache busting parameter (`?v=2`) to the URL, and post again.

This is slow and clunky. Plan for it.

### 14.4 The Bubbles Validation Workflow

After deploying Twitter Cards changes to a Bubbles client site:

1. `nginx -t && systemctl reload nginx`.
2. curl the page and verify twitter:* tags are server side rendered (not JS injected).
3. Run opengraph.xyz on the URL and confirm the X tab shows the expected card.
4. (Optional, for high stakes pages) post the URL to a private X account to confirm ground truth.

---

## 15. THE X CACHE BEHAVIOR

X caches card data for approximately 7 days. There is no manual purge available in 2026.

### 15.1 The Cache Duration

* **First crawl**: X fetches the card data the first time the URL is posted.
* **Cache lifetime**: approximately 7 days.
* **Re crawl trigger**: posting the URL again after cache expiry, or posting with a new query parameter.

### 15.2 The Cache Busting URL Pattern

```
https://heritagehardwoodfloorsnwa.com/services/?v=2
```

Appending `?v=2` creates a new URL identity from X's perspective, forcing a fresh crawl. Increment the version number on each refresh attempt (`?v=3`, `?v=4`).

The og:url and twitter:url tags should still point to the clean canonical URL without the query parameter. The cache busting parameter only affects what X crawls; the displayed URL on the card uses og:url / twitter:url.

### 15.3 The Comparison To Facebook And LinkedIn

| Platform | Cache | Manual Refresh |
|---|---|---|
| Facebook | ~30 days | Sharing Debugger "Scrape Again" (instant) |
| LinkedIn | ~7 days | Post Inspector (instant) |
| X | ~7 days | None (cache busting URL only) |
| Slack | 24 to 48 hours | None (cache busting URL only) |

Facebook and LinkedIn have official refresh tools. X and Slack do not.

### 15.4 The Implication For Bubbles Workflow

For Bubbles client launches: get Twitter Cards right *before* the URL is shared on X for the first time. After the first share, fixing the card requires either waiting 7 days or convincing the audience to reshare with a cache busting URL.

---

## 16. THE BUBBLES PER CLIENT DECISION FRAMEWORK

Each client site needs a Twitter Cards decision.

### 16.1 The Decision Questions

1. **Does the client have an active X handle?** If yes, include `twitter:site` and `twitter:creator`. If no, omit both.
2. **Is X a meaningful distribution channel for this client?** If yes, consider the full override block on key pages. If no, the minimal pattern is sufficient.
3. **Is the federal subcontracting context active?** If yes, ensure X cards render polished previews for government affairs and procurement audiences.
4. **Are there blog posts with distinct authors?** If yes, `twitter:creator` per post.

### 16.2 The Per Client Recommendations

**ThatDeveloperGuy.com:**

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@thatdeveloperguy">
<meta name="twitter:creator" content="@thatdeveloperguy">
```

X is a meaningful channel for SDVOSB visibility and tech industry outreach. Brand attribution pattern on every page. Full override on blog posts where X specific framing is desired.

**ThatAIGuy.org:**

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@thataiguy">
<meta name="twitter:creator" content="@thataiguy">
```

X is the primary channel for AI research and MEGAMIND outreach. Brand attribution on every page. Consider full override on MEGAMIND research pages, paper releases, and academic outreach posts.

**TCB Fight Factory:**

```html
<meta name="twitter:card" content="summary_large_image">
<!-- twitter:site only if TCB has an active X handle -->
```

X is a minor channel for combat sports vs Instagram and Facebook. Minimal pattern unless TCB has a verified active X presence.

**Arkansas Counseling and Wellness Services:**

```html
<meta name="twitter:card" content="summary_large_image">
<!-- twitter:site only if Dr. Kristy Burton has an active X handle -->
```

X is not a relevant channel for mental health practice marketing in NWA. Minimal pattern.

**Handled Tax and Advisory:**

```html
<meta name="twitter:card" content="summary_large_image">
<!-- twitter:site only if Amanda has an active X handle -->
```

X is not a meaningful tax preparation channel. Minimal pattern.

**WeCoverUSA:**

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@wecoverusa">  <!-- verify handle -->
```

Federal subcontractor context. Even minimal X presence benefits from polished card rendering when government affairs, defense industry, or franchised dealer audiences share URLs.

**Eureka Bath Works:**

```html
<meta name="twitter:card" content="summary_large_image">
```

Local home services. X is not a relevant channel. Minimal pattern.

**White River Cabins:**

```html
<meta name="twitter:card" content="summary_large_image">
<!-- twitter:site if active X presence -->
```

Travel and hospitality. Minor X relevance; minimal pattern unless an X presence is actively maintained.

**Greenough's Guide Service:**

```html
<meta name="twitter:card" content="summary_large_image">
```

Outdoor recreation. Facebook is the primary channel. Minimal pattern.

**Heritage Hardwood Floors NWA:**

```html
<meta name="twitter:card" content="summary_large_image">
```

Local home services. Minimal pattern.

**Local Living Real Estate (Laycee Maupin):**

```html
<meta name="twitter:card" content="summary_large_image">
```

Local real estate. Minimal pattern. Add `twitter:site` if Laycee actively uses X for real estate content.

**Diana Undergust (Lake Homes Realty):**

```html
<meta name="twitter:card" content="summary_large_image">
```

Local real estate, Grove OK / Grand Lake. Minimal pattern.

**Homes on Grand:**

```html
<meta name="twitter:card" content="summary_large_image">
```

Minimal pattern.

**Marshallese-Voices:**

```html
<!-- English page -->
<meta name="twitter:card" content="summary_large_image">

<!-- Marshallese page -->
<meta name="twitter:card" content="summary_large_image">
```

X is not a primary channel for the Marshallese community. Minimal pattern. Open Graph multi language fallback handles X correctly without twitter:locale (which does not exist).

**Avid Pest Control, RAH Construction NWA, Steele Solutions, Blue Paradise Dairy, Pretty Clean NWA:**

```html
<meta name="twitter:card" content="summary_large_image">
```

Local services across NWA and SWMO. Minimal pattern for all.

### 16.3 The Documentation Pattern

For each client, document the Twitter Cards configuration in the client project notes:

```
Twitter Cards configuration:
  twitter:card:     summary_large_image
  twitter:site:     [@handle or "none"]
  twitter:creator:  [@handle or "none"]
  Override pattern: [minimal / brand / full]
  Validation:       opengraph.xyz [date], live X post [date if performed]
```

---

## 17. THE SERVER SIDE RENDERING REQUIREMENT

Same constraint as Open Graph: twitter:* tags must be in the initial server side rendered HTML response. The X crawler does not execute JavaScript.

### 17.1 The Crawler Behavior

X's crawler fetches the URL and parses the HTML. If twitter:* (and og:*) tags are not present in the initial HTML response, no card is generated. Client side framework hydration (React `useEffect`, Vue mounted hooks) runs too late.

### 17.2 The Verification Method

```bash
curl -s https://example.com/page/ | grep -i 'twitter:'
```

If the curl output includes the twitter:* tags, they are server side rendered. If empty, the tags are JS injected and invisible to X.

### 17.3 The Static HTML, Next.js, SvelteKit, Astro, Headless Shopify Stacks

Same patterns as for Open Graph (framework-html-open-graph.md Section 19 and Section 20). The Twitter Cards tags live alongside the OG tags in the same `<head>` and are rendered by the same server side or static generation mechanism.

---

## 18. THE IMPLEMENTATION PATTERNS BY STACK

### 18.1 Static HTML (Bubbles Default, Minimal Pattern)

```html
<head>
    <!-- Open Graph (primary signal, X falls back to these) -->
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

    <!-- Twitter Cards minimal: one tag -->
    <meta name="twitter:card" content="summary_large_image">
</head>
```

### 18.2 Static HTML (Brand Attribution Pattern)

```html
<head>
    <!-- Open Graph block (as Section 18.1) -->

    <!-- Twitter Cards brand attribution: three tags -->
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:site" content="@thatdeveloperguy">
    <meta name="twitter:creator" content="@thatdeveloperguy">
</head>
```

### 18.3 Static HTML (Full Override Pattern)

```html
<head>
    <!-- Open Graph block (as Section 18.1) -->

    <!-- Twitter Cards full override: eight tags -->
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:site" content="@thatdeveloperguy">
    <meta name="twitter:creator" content="@thatdeveloperguy">
    <meta name="twitter:url" content="https://thatdeveloperguy.com/blog/post-slug/">
    <meta name="twitter:title" content="X specific headline">
    <meta name="twitter:description" content="X specific description.">
    <meta name="twitter:image" content="https://thatdeveloperguy.com/blog/twitter-card-post.jpg">
    <meta name="twitter:image:alt" content="X card image alt text">
</head>
```

### 18.4 Next.js 14 App Router

```typescript
// app/page.tsx
export const metadata = {
  title: 'Heritage Hardwood Floors NWA | Bella Vista AR',
  description: 'Bella Vista hardwood floor refinishing.',
  openGraph: {
    title: 'Heritage Hardwood Floors NWA | Bella Vista AR',
    description: 'Bella Vista hardwood floor refinishing.',
    url: 'https://heritagehardwoodfloorsnwa.com/',
    siteName: 'Heritage Hardwood Floors NWA',
    locale: 'en_US',
    type: 'website',
    images: [{ url: 'https://heritagehardwoodfloorsnwa.com/og-image.jpg', width: 1200, height: 630, alt: 'Heritage Hardwood Floors NWA' }],
  },
  twitter: {
    card: 'summary_large_image',
    // Add site, creator, title, description, images only when overriding
    // site: '@thatdeveloperguy',
    // creator: '@thatdeveloperguy',
  },
};
```

### 18.5 SvelteKit

```svelte
<svelte:head>
    <!-- OG block -->
    <meta property="og:title" content="Heritage Hardwood Floors NWA" />
    <!-- ... rest of OG ... -->

    <!-- Twitter Cards -->
    <meta name="twitter:card" content="summary_large_image" />
    <meta name="twitter:site" content="@thatdeveloperguy" />
</svelte:head>
```

### 18.6 Astro

```astro
---
const card = {
  type: 'summary_large_image',
  site: '@thatdeveloperguy',
  creator: '@thatdeveloperguy',
};
---
<head>
    <!-- OG block -->

    <meta name="twitter:card" content={card.type} />
    <meta name="twitter:site" content={card.site} />
    <meta name="twitter:creator" content={card.creator} />
</head>
```

### 18.7 The Nginx Sub Filter Backfill

For backfilling the minimal Twitter Cards tag across existing Bubbles HTML pages:

```nginx
sub_filter '</head>' '<meta name="twitter:card" content="summary_large_image"></head>';
sub_filter_once on;
sub_filter_types text/html;
```

End every nginx change with:

```bash
nginx -t && systemctl reload nginx
```

For the FastAPI sidecar on port 9090: include the Twitter Cards block in the SSI injection pattern alongside the OG block.

---

## 19. THE PRE LAUNCH AUDIT CHECKLIST

Before a Bubbles client site goes live, verify Twitter Cards.

### 19.1 The Checklist

* [ ] `twitter:card` present on every page (always `summary_large_image` for Bubbles convention).
* [ ] `twitter:site` present when client has an active X handle.
* [ ] `twitter:creator` present on blog posts with an X authored byline.
* [ ] Open Graph block complete (so X fallback works for title, description, image, URL).
* [ ] og:image (or twitter:image if separately set) is ≥1200x628, HTTPS, ≤5 MB.
* [ ] All twitter:* tags use `name=` attribute (not `property=`).
* [ ] Handle format is `@handle` with leading `@`, no URL form.
* [ ] curl on the page returns twitter:* tags in HTML response (not JS injected).
* [ ] opengraph.xyz X tab shows expected card preview.
* [ ] Ground truth test: post URL to private X account and confirm rendering (for high stakes pages).

### 19.2 The Automated Audit

The thatdeveloperguy.com audit pipeline should check:

* `twitter:card` presence (boolean).
* `twitter:card` value is `summary_large_image` (recommended) or other valid type.
* OG fallback completeness (since X depends on it).
* `twitter:site` matches expected client handle (if mapped).
* No `property="twitter:..."` mistakes.

---

## 20. THE COMMON FAILURES AND FIXES

### 20.1 "X is showing the small summary card instead of the large image"

Cause 1: `twitter:card` is missing or set to `summary`.
Fix: Set `twitter:card="summary_large_image"`.

Cause 2: The image is below 300x157 pixels and X silently downgraded.
Fix: Use a 1200x628 or larger image.

Cause 3: The image URL is HTTP, not HTTPS.
Fix: Use HTTPS absolute URL.

### 20.2 "The wrong handle is credited on the card"

Cause: `twitter:site` or `twitter:creator` has wrong handle, missing `@`, or uses URL form.
Fix: Use `@handle` format with correct case insensitive handle.

### 20.3 "X stripped the image entirely from the card"

Cause 1: Image is HTTP, not HTTPS.
Fix: HTTPS only.

Cause 2: Image file size over 5 MB.
Fix: Compress to under 5 MB (target ≤300 KB).

Cause 3: Image format unsupported (SVG, AVIF).
Fix: Use JPG, PNG, WebP, or GIF.

Cause 4: Image URL returns 404 or is behind authentication.
Fix: Verify image is publicly accessible via HTTPS.

### 20.4 "The card preview is from a previous version of the page"

Cause: X cached the previous version. Cache duration is ~7 days.
Fix: Append `?v=2` to the URL when resharing to force a new crawl. Increment for subsequent attempts.

### 20.5 "Card works in opengraph.xyz preview but not when posted on X"

Cause 1: Tags are JS injected; X crawler did not see them.
Fix: Move tags to server side rendered HTML. Verify with curl.

Cause 2: The page has anti scraper protections that block the X crawler.
Fix: Allow the X crawler user agent (`Twitterbot/1.0`) in any bot rules.

Cause 3: X cached an earlier failed crawl.
Fix: Cache busting URL.

### 20.6 "Tags include property=twitter:... by mistake"

Cause: Confused with Open Graph's `property=` convention.
Fix: Use `name="twitter:..."` consistently. Audit existing pages for the wrong attribute.

### 20.7 "The card title is being truncated mid sentence"

Cause: `twitter:title` (or og:title via fallback) is over 70 characters.
Fix: Keep titles under 70 characters for X. If the page's `og:title` is longer for Facebook context, set a shorter `twitter:title` override.

### 20.8 "Player card is not rendering"

Cause: Player domain has not been approved by X.
Fix: For Bubbles convention, don't attempt player cards. Link to YouTube or Vimeo instead.

### 20.9 "App card is not rendering"

Cause 1: Missing required app ID for the platform X detects.
Fix: Include `twitter:app:id:iphone`, `twitter:app:id:ipad`, and `twitter:app:id:googleplay`.

Cause 2: App is not published or app ID is wrong.
Fix: Verify against App Store Connect and Google Play Console.

### 20.10 "Card rendering is inconsistent: works for some shares, not others"

Cause: X's caching and rendering layer has intermittent failures (acknowledged on the X developer forum). No reliable fix.
Workaround: Ensure tags are correct and server side rendered. Accept some intermittent rendering as a platform limitation. Federal and high stakes shares may benefit from posting from a verified account to improve crawl reliability.

---

## 21. THE FEDERAL AND SDVOSB APPEARANCE CONTEXT ON X

For Joseph's SDVOSB federal subcontracting work, X is a more relevant channel than for typical local services.

### 21.1 The Federal X Audience

* Defense industry analysts and journalists.
* Government affairs professionals.
* SDVOSB community members and veteran owned business advocates.
* SAM.gov and federal procurement Twitter accounts.
* Congressional staff and policy professionals.
* Federal contracting officer professional networks.

These audiences are concentrated on X far more than on Facebook. A ThatDeveloperGuy SDVOSB URL shared on X with a clean polished card builds credibility; the same URL with a missing card or wrong image signals an unfinished web presence.

### 21.2 The ThatDeveloperGuy SDVOSB X Pattern

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@thatdeveloperguy">
<meta name="twitter:creator" content="@thatdeveloperguy">
```

Plus the underlying OG block per framework-html-open-graph.md Section 23.2 with the SDVOSB branded og:image.

### 21.3 The WeCoverUSA Federal X Pattern

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@wecoverusa">  <!-- verify handle -->
```

Federal subcontractor appearance. Polished og:image (per OG framework Section 23.4) handles the X card via fallback.

### 21.4 The Documentation Note

For federal context client work, document Twitter Cards alongside Open Graph in the project notes. Procurement officers and defense analysts may screenshot X previews as part of vendor diligence.

### 21.5 The Handle Verification Discipline

Federal handles in particular need verification:

* The handle exists and is currently held by the intended party.
* The handle is verified (blue check) or otherwise credible.
* The handle has not been recycled, dormant, or compromised.

Crediting the wrong handle on a federal context card is a credibility failure.

---

## 22. QUICK REFERENCE TAG BLOCK

For copy paste into any Bubbles client page `<head>`, after the Open Graph block:

### 22.1 Minimal Pattern (One Tag)

```html
<!-- Twitter Cards (X uses og:* as fallback) -->
<meta name="twitter:card" content="summary_large_image">
```

### 22.2 Brand Attribution Pattern (Three Tags)

```html
<!-- Twitter Cards with brand attribution -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@CLIENT_HANDLE">
<meta name="twitter:creator" content="@CLIENT_HANDLE">
```

### 22.3 Full Override Pattern (Eight Tags)

```html
<!-- Twitter Cards with X specific content overrides -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@CLIENT_HANDLE">
<meta name="twitter:creator" content="@AUTHOR_HANDLE">
<meta name="twitter:url" content="https://CLIENT.com/PAGE/">
<meta name="twitter:title" content="X SPECIFIC TITLE">
<meta name="twitter:description" content="X SPECIFIC DESCRIPTION (200 chars max).">
<meta name="twitter:image" content="https://CLIENT.com/twitter-card.jpg">
<meta name="twitter:image:alt" content="DESCRIPTIVE ALT FOR ACCESSIBILITY">
```

### 22.4 Player Card (Not For Bubbles Direct Implementation)

Link to YouTube or Vimeo instead. Do not implement player cards directly without X domain approval.

### 22.5 App Card (Not Applicable To Bubbles Clients)

Skip. None of Joseph's current clients have a mobile app.

---

## 23. REFERENCES

* X Cards documentation (archived Twitter docs): https://developer.twitter.com/en/docs/twitter-for-websites/cards/overview/abouts-cards
* Summary card with large image: https://developer.twitter.com/en/docs/twitter-for-websites/cards/overview/summary-card-with-large-image
* Player card: https://developer.twitter.com/en/docs/twitter-for-websites/cards/overview/player-card
* App card: https://developer.twitter.com/en/docs/twitter-for-websites/cards/overview/app-card
* Open Graph protocol (the fallback signal): https://ogp.me
* opengraph.xyz preview tool: https://www.opengraph.xyz/
* socialpreviewhub.com X validator: https://socialpreviewhub.com/twitter-card-validator
* env.dev social share debugger: https://env.dev/tools/social-share-debugger

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
* framework-html-open-graph.md (direct parent)
* framework-html-schema-jsonld.md (likely next in track)

**Parent reference:** UNIVERSAL-RANKING-FRAMEWORK.md (the 27 section master document).

---

*End of framework-html-twitter-cards.md.*
