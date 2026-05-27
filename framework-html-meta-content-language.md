# framework-html-meta-content-language.md

Comprehensive reference for HTML `<meta http-equiv="content-language">`, the page language declaration via http-equiv mechanism. Covers the distinction from `<meta name="*">` tags (this uses `http-equiv` which simulates an HTTP response header rather than declaring document metadata), the three language signal channels (the `<html lang="">` attribute which is the modern preferred signal; the `<meta http-equiv="content-language">` which is largely deprecated; the HTTP `Content-Language:` response header covered in framework-http-content-headers.md), the Google deprecation in 2010 (Google explicitly stopped using `http-equiv="content-language"` for language detection), the IETF BCP 47 language tag format (`en-US`, `en-GB`, `mh` for Marshallese, etc.), the `hreflang` link element for multi language sites (the modern international SEO signal), the related `lang` attribute on body elements for mixed language content, the Marshallese-Voices case study (a Bubbles client with Marshallese content where language handling matters), the federal/SDVOSB international context for Joseph's subcontracting work, and the Bubbles convention (use `<html lang>` attribute as primary; omit the http-equiv meta; use hreflang for true multi language sites). Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the twelfth framework in the HTML signal track**, following meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, color-scheme, and referrer. Companion to the 12 wire layer frameworks especially framework-http-content-headers.md (the HTTP `Content-Language:` header is the canonical layer for language signaling).

Audience: humans configuring language signals correctly for single language and multi language sites, AI assistants generating HTML head sections with appropriate language declarations, SEO operators improving international targeting via hreflang, developers migrating sites from CMS auto generated `http-equiv="content-language"` to modern alternatives, operators handling unusual language situations (Marshallese-Voices for the Marshallese community in Springdale AR; federal subcontracts requiring multi language support), and anyone troubleshooting "Google is showing my US English content to UK users", "the Marshallese page is being interpreted as English", "should we use `html lang` or `meta content-language` or both", or "my multi language site isn't being recognized as multi language".

---

## TABLE OF CONTENTS

1. Definition
2. Why It (Mostly Doesn't) Matter
3. What This Covers
4. The Mental Model: HTML lang Is Primary
5. The Three Language Signal Channels
6. The Google Deprecation (2010)
7. The IETF BCP 47 Language Tag Format
8. The hreflang Link Element (Modern International SEO)
9. The Multi Language Site Pattern
10. The Marshallese-Voices Case Study
11. The Federal And SDVOSB International Context
12. The Browser And Tool Behavior
13. The Bubbles Decision Framework
14. Asset Class And Use Case Recipes
15. Bubbles Standard Pattern (paste ready)
16. Audit Checklist
17. Common Pitfalls
18. Diagnostic Commands
19. Cross-References

---

## 1. DEFINITION

`<meta http-equiv="content-language">` is the HTML directive that simulates the HTTP `Content-Language:` response header at the document level. The `http-equiv` attribute (HTTP equivalent) means the tag declares an HTTP response header value via HTML rather than via the actual HTTP response.

```html
<head>
    <meta http-equiv="content-language" content="en-US">
</head>
```

Three structural facts shape how the directive works:

* **It uses `http-equiv`, not `name`.** Unlike most meta tags (`name="description"`, `name="viewport"`, etc.), this tag uses `http-equiv` to simulate an HTTP header. This is a different attribute mechanism with different semantics.
* **Google explicitly deprecated it.** In 2010, Google announced they would no longer use `http-equiv="content-language"` for language detection. Instead, they use the `<html lang="">` attribute and `hreflang` link element. Most other search engines followed suit.
* **The `<html lang>` attribute is the modern primary signal.** Per HTML5 best practices and current Google documentation, the `<html lang>` attribute is the canonical place to declare page language. The http-equiv meta tag adds nothing.

For Bubbles client sites in 2026, the convention is to use `<html lang="en-US">` (or appropriate BCP 47 code) on the `<html>` element. Skip the `http-equiv="content-language"` meta tag entirely. Use `hreflang` link element for multi language sites. For sites with mixed language content (Marshallese-Voices, Spanish content in NWA), use `lang` attribute on specific elements containing the alternate language.

---

## 2. WHY IT (MOSTLY DOESN'T) MATTER

Seven independent considerations clarify why http-equiv="content-language" is largely obsolete in 2026.

**Google deprecated it in 2010.** Per the Google Webmaster Central Blog (December 2010): "We do not consider the value of http-equiv 'content-language' anymore. Instead, use the html lang attribute to specify the language of the content." This has been Google's official position for over 15 years.

**The `<html lang>` attribute is the modern primary signal.** Per HTML5 specification and current Google Search Central documentation: `<html lang="en-US">` on the root element is the recommended way to declare document language. Browsers use this for spell check, screen reader pronunciation, and font selection. Search engines use it for language detection.

**The HTTP Content-Language header is the canonical server side signal.** Per framework-http-content-headers.md Section 2, the HTTP `Content-Language:` response header is the canonical server side language signal. Set via nginx, it applies to all responses. The http-equiv meta tag is a less authoritative duplicate.

**hreflang link element is the modern international SEO signal.** For multi language sites, the `<link rel="alternate" hreflang="...">` mechanism tells Google which language version of a page is for which audience. This is the canonical signal for international SEO. The http-equiv tag does not provide hreflang functionality.

**CMS auto generation can still add the tag.** Some legacy WordPress themes, older Drupal versions, and other CMSs auto generate `<meta http-equiv="content-language">`. The tag is harmless but adds no value. Removal is cleanup, not requirement.

**Browser behavior is the html lang attribute, not http-equiv.** Browsers use `<html lang>` for:
- Hyphenation rules.
- Screen reader pronunciation.
- Font selection for CJK characters.
- Spell check language.
- Voice synthesis.
None of these features look at http-equiv content-language.

**For multi language Bubbles sites, the focus is hreflang.** The Marshallese-Voices site has Marshallese content that needs proper international SEO signaling. The mechanism is hreflang, not http-equiv content-language.

**Cost of getting it wrong.** Misconfigured language signals produce measurable damage. Real examples:

* Bubbles client had `<meta http-equiv="content-language" content="en-US">` but no `<html lang>` attribute. Browser screen readers couldn't determine language; pronunciation was generic. Adding `<html lang="en-US">` fixed the screen reader behavior.
* Marshallese-Voices site had English HTML wrapping Marshallese content without language markers. Search engines treated everything as English; Marshallese language pages didn't rank for Marshallese queries. Adding `<html lang="mh">` on Marshallese pages and `lang="mh"` on Marshallese content elements within English wrappers fixed language identification.
* International real estate client (Diana Undergust serving Grand Lake area) had `http-equiv="content-language"` but no hreflang. Site appeared in search results from multiple English speaking countries with inconsistent ranking. Adding hreflang plus removing the deprecated meta tag clarified the targeting.
* Federal subcontractor site had `http-equiv="content-language" content="en"` (just "en") instead of "en-US" or proper BCP 47. Subtle but Google prefers fully qualified language tags. Updated to `<html lang="en-US">` plus removing the meta tag.
* WordPress migration left `<meta http-equiv="content-language" content="en-US">` in the new Bubbles build. Wappalyzer detected it; analysis showed the new site appeared to be a WordPress derivative. Removing the legacy tag during migration cleanup was a missed step.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The meta content-language tag plus its full operational context:

1. **The three language signal channels**: html lang, http-equiv meta, HTTP header.
2. **The Google deprecation**: what changed and why.
3. **The BCP 47 format**: how to write language tags.
4. **The hreflang mechanism**: for multi language sites.
5. **The Marshallese-Voices case study**: real example for Joseph.
6. **The Bubbles convention**: html lang primary; skip http-equiv meta.

Section 10 is the Marshallese-Voices case study (specific to Joseph's client work). Section 13 is the Bubbles decision framework.

---

## 4. THE MENTAL MODEL: HTML LANG IS PRIMARY

The browser fetches your page. Multiple sources potentially declare language:

```
Page contains language signals:
   |
   v
==================== PER SIGNAL READING ORDER ====================
   |
   |---> HTTP Content-Language response header (set at server)
   |     Sent before HTML parsing begins.
   |     Authoritative for server-declared language.
   |
   |---> <html lang="..."> attribute (parsed early in HTML)
   |     Primary signal for browsers and search engines.
   |     Used for hyphenation, screen reader, search ranking.
   |
   |---> <meta http-equiv="content-language" content="...">
   |     LEGACY: ignored by Google since 2010.
   |     Still parsed by some tools and older crawlers.
   |     No value in modern web.
   |
   |---> <lang="..."> on body elements
   |     For mixed language content.
   |     Per element override.
   |
   |---> <link rel="alternate" hreflang="...">
   |     For multi language site targeting.
   |     The international SEO signal.

==================== THE DECISION TREE ====================
   |
   v
   What is the primary page language?
      |
      v
   Set <html lang="en-US"> (or appropriate BCP 47)
      |
      v
   Does any element contain different language?
      |
      |---> YES: add lang="..." to that element
      |
      |---> NO: continue
      |
      v
   Is this a multi language site (translations exist)?
      |
      |---> YES: add hreflang link elements for each translation
      |
      |---> NO: continue
      |
      v
   Should HTTP Content-Language header be set?
      |
      |---> YES: configure in nginx (covered in framework-http-content-headers.md)
      |
      |---> NO: html lang is sufficient
      |
      v
   Should <meta http-equiv="content-language"> be included?
      |
      |---> ALMOST NEVER: deprecated by Google; redundant with html lang
```

Six rules govern the system:

1. **Always set `<html lang="...">` with appropriate BCP 47 code.**
2. **Skip `<meta http-equiv="content-language">` (Google deprecated; redundant).**
3. **Use `lang="..."` on elements with different language content.**
4. **Use `hreflang` link elements for multi language sites.**
5. **Set HTTP `Content-Language` header at nginx for server side signal.**
6. **Use full BCP 47 codes (`en-US` not just `en`).**

A correctly configured Bubbles client site has `<html lang="en-US">` on every page, no `<meta http-equiv="content-language">`, `lang` attributes on specific elements with different language content, and hreflang links if multi language.

---

## 5. THE THREE LANGUAGE SIGNAL CHANNELS

Language can be declared via three mechanisms. Each has different purpose and authority.

### 5.1 Channel 1: HTTP Content-Language Response Header

```nginx
add_header Content-Language "en-US" always;
```

Or in FastAPI:

```python
response.headers["Content-Language"] = "en-US"
```

**Authority**: highest (server side, parsed first).

**Use case**: site wide language declaration. Per framework-http-content-headers.md Section 2.

**Scope**: applies to all responses.

### 5.2 Channel 2: HTML `<html lang="">` Attribute

```html
<html lang="en-US">
```

**Authority**: primary for HTML and browser behavior.

**Use case**: document level language; used by browsers for hyphenation, screen reader, font selection, search engines for ranking.

**Scope**: applies to the document.

### 5.3 Channel 3: HTML `<meta http-equiv="content-language">`

```html
<meta http-equiv="content-language" content="en-US">
```

**Authority**: deprecated by Google in 2010; some tools still parse.

**Use case**: legacy. Largely obsolete.

**Scope**: applies to the document.

### 5.4 The Comparison

| Channel | Authority | Use in 2026 | Bubbles convention |
|---|---|---|---|
| HTTP Content-Language header | Highest | Recommended | Set via nginx (optional) |
| `<html lang>` attribute | Primary | Required | Always set |
| `<meta http-equiv="content-language">` | Deprecated | Not needed | Omit |

### 5.5 The Alignment

When multiple channels are set, they should agree:

```nginx
# HTTP header
add_header Content-Language "en-US" always;
```

```html
<!-- HTML attribute -->
<html lang="en-US">
```

If they disagree, browsers typically use `<html lang>` (closest to content), search engines may use either or both.

For Bubbles: align them. Don't set conflicting values.

### 5.6 The Bubbles Default

* **HTTP header**: optional, depends on whether you want server level signal (see framework-http-content-headers.md).
* **`<html lang>` attribute**: required on every page.
* **`<meta http-equiv="content-language">`**: omit; legacy.

---

## 6. THE GOOGLE DEPRECATION (2010)

In December 2010, Google publicly announced that they no longer use `<meta http-equiv="content-language">` for language detection.

### 6.1 The Announcement

Per the Google Webmaster Central Blog (now Search Central): "We are no longer using the meta content-language tag for language detection. Instead, please use the html lang attribute to specify the language of your content."

The reasoning: the meta tag is unreliable (often set by CMS without site author awareness, easy to misconfigure). The html lang attribute is part of the HTML5 spec, parsed correctly by browsers, and provides multi purpose value (browser behavior plus search engine signal).

### 6.2 The Practical Implication

For Google ranking:

* Setting `<meta http-equiv="content-language" content="en-US">` provides no SEO benefit.
* Setting `<html lang="en-US">` provides the language signal.
* Setting hreflang link elements provides international targeting.

For other search engines:

* Bing: similar policy; uses html lang attribute.
* Yandex: uses html lang attribute.
* Baidu: uses html lang attribute plus content analysis.

For browsers:

* Use html lang attribute, not http-equiv content-language.

For accessibility tools (screen readers):

* Use html lang attribute, not http-equiv content-language.

### 6.3 The Implication For Cleanup

When migrating from CMS to hand coded Bubbles:

```bash
# Find HTML files with meta http-equiv="content-language"
find /var/www/sites/ -name "*.html" -type f -exec \
    grep -l 'meta http-equiv="content-language"' {} \;

# Remove the legacy tag
find /var/www/sites/ -name "*.html" -type f -exec \
    sed -i '/meta http-equiv="content-language"/d' {} \;

# Verify removal
curl -s https://example.com/ | grep -c 'meta http-equiv="content-language"'
# Expected: 0
```

### 6.4 The Why Keep Awareness

Despite the deprecation, awareness matters because:

* Legacy CMS may still auto generate the tag.
* Older tools may show the tag in audit reports.
* Some older crawlers may still parse it.
* The presence of the tag is a "code smell" suggesting CMS legacy.

For Bubbles: omit by default; remove during migrations.

---

## 7. THE IETF BCP 47 LANGUAGE TAG FORMAT

Language tags follow IETF BCP 47 (Best Current Practice 47). The format is hierarchical.

### 7.1 The Basic Format

```
language-region-script-variant
```

Most commonly used: `language-region`.

### 7.2 Common Examples

| Tag | Meaning |
|---|---|
| `en` | English (any region) |
| `en-US` | English, United States |
| `en-GB` | English, United Kingdom |
| `es` | Spanish (any region) |
| `es-MX` | Spanish, Mexico |
| `fr-CA` | French, Canada |
| `zh-CN` | Chinese, Simplified (China) |
| `zh-TW` | Chinese, Traditional (Taiwan) |
| `ja` | Japanese |
| `de-DE` | German, Germany |
| `mh` | Marshallese |
| `kos` | Kosraean |

### 7.3 The Language Subtag

The first subtag is the language code (ISO 639-1 two letter or ISO 639-2/3 three letter).

For Marshallese: `mh` (ISO 639-1) or `mah` (ISO 639-3).

### 7.4 The Region Subtag

Optional. Specifies the regional variant:

* `US`: United States.
* `GB`: United Kingdom.
* `MX`: Mexico.
* `CA`: Canada.
* `MH`: Marshall Islands.

When omitted: language only (no specific region).

For Bubbles US-based clients: `en-US` is the standard choice.

### 7.5 The Script Subtag

Used when a language has multiple scripts:

* `zh-Hans-CN`: Chinese, Simplified script, China.
* `zh-Hant-TW`: Chinese, Traditional script, Taiwan.
* `sr-Cyrl`: Serbian, Cyrillic script.
* `sr-Latn`: Serbian, Latin script.

For most Bubbles work: language-region format is sufficient.

### 7.6 The Marshallese-Voices Application

For the Marshallese-Voices client:

```html
<!-- Marshallese language page -->
<html lang="mh">
```

Or with explicit region:

```html
<!-- Marshallese language, Marshall Islands -->
<html lang="mh-MH">
```

For Bubbles convention: `mh` (language only) is sufficient unless distinguishing Marshallese variants is needed.

### 7.7 The Bubbles BCP 47 Convention

For most clients: `en-US`.

For Joseph's bilingual or multilingual clients: appropriate BCP 47 code per content.

For Marshallese-Voices: `mh` for Marshallese, `en-US` for English content.

---

## 8. THE HREFLANG LINK ELEMENT (MODERN INTERNATIONAL SEO)

For multi language sites, hreflang is the modern Google recommended signal.

### 8.1 The Pattern

```html
<head>
    <!-- English (US) version -->
    <link rel="alternate" hreflang="en-US" href="https://example.com/">

    <!-- Spanish (Mexico) version -->
    <link rel="alternate" hreflang="es-MX" href="https://example.com/es/">

    <!-- Marshallese version -->
    <link rel="alternate" hreflang="mh" href="https://example.com/mh/">

    <!-- Default for unmatched languages -->
    <link rel="alternate" hreflang="x-default" href="https://example.com/">
</head>
```

### 8.2 The Reciprocity Rule

Each language version must include hreflang for all language versions, including itself.

If `https://example.com/` (English) has:
```html
<link rel="alternate" hreflang="es-MX" href="https://example.com/es/">
```

Then `https://example.com/es/` (Spanish) must have:
```html
<link rel="alternate" hreflang="en-US" href="https://example.com/">
<link rel="alternate" hreflang="es-MX" href="https://example.com/es/">
```

Bidirectional confirmation.

### 8.3 The x-default Special Value

```html
<link rel="alternate" hreflang="x-default" href="https://example.com/">
```

Used when none of the listed languages match the user's preference. The default fallback.

For most Bubbles sites with primary English: x-default points to the English version.

### 8.4 The Implementation Options

**Option A: HTML head (preferred for HTML)**:

```html
<head>
    <link rel="alternate" hreflang="en-US" href="...">
    <link rel="alternate" hreflang="es" href="...">
</head>
```

**Option B: HTTP header (alternative)**:

```nginx
location / {
    add_header Link '<https://example.com/>; rel="alternate"; hreflang="en-US"';
    add_header Link '<https://example.com/es/>; rel="alternate"; hreflang="es"';
}
```

**Option C: Sitemap (for many pages)**:

```xml
<url>
    <loc>https://example.com/</loc>
    <xhtml:link rel="alternate" hreflang="en-US" href="https://example.com/"/>
    <xhtml:link rel="alternate" hreflang="es" href="https://example.com/es/"/>
</url>
```

For most Bubbles multi language sites: HTML head approach.

### 8.5 The Common Mistakes

* **Forgetting reciprocity**: only one direction listed; Google requires bidirectional.
* **Wrong BCP 47 codes**: `en_US` (underscore) instead of `en-US` (hyphen).
* **Missing x-default**: users not matched to any listed language get suboptimal experience.
* **Inconsistent canonical URLs**: hreflang URLs and canonical URLs don't match.

### 8.6 The Bubbles Application

For most Bubbles clients: not multi language. hreflang not needed.

For Marshallese-Voices: hreflang for Marshallese and English versions.

For potential federal sites with multi language requirements: hreflang implementation.

---

## 9. THE MULTI LANGUAGE SITE PATTERN

For sites with multiple language versions, the comprehensive language signaling pattern.

### 9.1 The Five Layer Signal

```html
<!-- 1. HTML lang attribute (page primary language) -->
<html lang="en-US">
<head>
    <!-- 2. Meta tag (skip; deprecated) -->

    <!-- 3. hreflang for each language version -->
    <link rel="alternate" hreflang="en-US" href="https://example.com/">
    <link rel="alternate" hreflang="es-MX" href="https://example.com/es/">
    <link rel="alternate" hreflang="x-default" href="https://example.com/">

    <!-- 4. Canonical URL -->
    <link rel="canonical" href="https://example.com/">

    <!-- 5. Open Graph locale -->
    <meta property="og:locale" content="en_US">
    <meta property="og:locale:alternate" content="es_MX">
</head>
<body>
    <!-- Page content in primary language -->

    <!-- 6. lang attribute on mixed content -->
    <p>Welcome to our service.</p>
    <p lang="es-MX">¡Bienvenidos a nuestro servicio!</p>
</body>
```

Plus on the Spanish version (`/es/`):

```html
<html lang="es-MX">
<head>
    <link rel="alternate" hreflang="en-US" href="https://example.com/">
    <link rel="alternate" hreflang="es-MX" href="https://example.com/es/">
    <link rel="alternate" hreflang="x-default" href="https://example.com/">

    <link rel="canonical" href="https://example.com/es/">

    <meta property="og:locale" content="es_MX">
    <meta property="og:locale:alternate" content="en_US">
</head>
```

### 9.2 The HTTP Header Coordination

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name example.com;

    # English version
    location / {
        add_header Content-Language "en-US" always;
        # ...
    }

    # Spanish version
    location /es/ {
        add_header Content-Language "es-MX" always;
        # ...
    }
}
```

### 9.3 The URL Pattern Choices

**Option A: Subdirectory structure (recommended for Bubbles)**:

```
https://example.com/        (English)
https://example.com/es/     (Spanish)
https://example.com/mh/     (Marshallese)
```

Simple, single domain, single SSL certificate. Good for SEO authority concentration.

**Option B: Subdomain structure**:

```
https://example.com         (English)
https://es.example.com      (Spanish)
https://mh.example.com      (Marshallese)
```

Cleaner separation but split SEO authority.

**Option C: Separate domains**:

```
https://example.com         (English)
https://example.es          (Spanish)
```

Best for strong regional targeting; most complex.

For most Bubbles work: Option A (subdirectory).

### 9.4 The Verification

```bash
# Verify hreflang implementation
URL=https://example.com/

# Get all hreflang from page
curl -s "$URL" | grep -oE 'link rel="alternate" hreflang="[^"]+" href="[^"]+"'

# Verify reciprocity (each version listed in others)
for url in https://example.com/ https://example.com/es/ https://example.com/mh/; do
    echo "=== Checking $url ==="
    curl -s "$url" | grep -oE 'link rel="alternate" hreflang="[^"]+" href="[^"]+"' | head -5
    echo ""
done
```

---

## 10. THE MARSHALLESE-VOICES CASE STUDY

The Marshallese-Voices client (community focused site for the Marshallese American community in NWA, particularly Springdale) is a Bubbles client with real language signaling needs.

### 10.1 The Context

* **Community**: Marshallese American community in Northwest Arkansas; significant population in Springdale and surrounding cities.
* **Content**: information, resources, events for Marshallese community.
* **Languages**: English (primary) and Marshallese (secondary).
* **Goals**: respectful language handling; proper SEO for both languages.

### 10.2 The Standing Cultural Respect

Per the standing context, Marshallese American business community is part of the Springdale area. The Marshallese-Voices client supports this community. Language handling must:

* Treat both languages with equal respect.
* Not "default to English" for Marshallese content.
* Properly identify Marshallese content for screen readers and search engines.

### 10.3 The Site Structure

```
/                   (English landing page)
/about/             (English about)
/resources/         (English resources)
/community/         (English community)
/mh/                (Marshallese landing page)
/mh/about/          (Marshallese about)
/mh/resources/      (Marshallese resources)
/mh/community/      (Marshallese community)
```

### 10.4 The HTML Pattern Per Page

**English page** (`/`):

```html
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Marshallese-Voices | Community Resources in NWA</title>
    <meta name="description" content="Resources and community support for the Marshallese American community in Northwest Arkansas. Available in English and Marshallese.">

    <!-- hreflang for both languages -->
    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <!-- Canonical -->
    <link rel="canonical" href="https://marshallese-voices.org/">

    <!-- Open Graph -->
    <meta property="og:locale" content="en_US">
    <meta property="og:locale:alternate" content="mh">
</head>
<body>
    <h1>Marshallese-Voices</h1>
    <p>Community resources for the Marshallese American community in Northwest Arkansas.</p>

    <!-- Link to Marshallese version -->
    <p><a href="/mh/" hreflang="mh" lang="mh">Yokwe! Click here for Marshallese.</a></p>

    <!-- ... content ... -->
</body>
</html>
```

**Marshallese page** (`/mh/`):

```html
<!doctype html>
<html lang="mh">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Marshallese-Voices | Kein Ekkar im Jipan ilo NWA</title>
    <meta name="description" content="Kein ekkar im jipan an Marshallese American ilo Northwest Arkansas. Bok mall ilo Kajin Kajin Majol im English.">

    <!-- hreflang for both languages -->
    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <!-- Canonical -->
    <link rel="canonical" href="https://marshallese-voices.org/mh/">

    <!-- Open Graph -->
    <meta property="og:locale" content="mh">
    <meta property="og:locale:alternate" content="en_US">
</head>
<body>
    <h1>Marshallese-Voices</h1>

    <!-- Marshallese content with proper lang attribute -->
    <!-- ... -->

    <!-- Link back to English version -->
    <p><a href="/" hreflang="en-US" lang="en-US">Click here for English / Bok kin English.</a></p>
</body>
</html>
```

### 10.5 The Mixed Language Handling

For sections with both English and Marshallese in close proximity:

```html
<div class="bilingual-section">
    <h2>Resources Available</h2>
    <p>Translation services are available.</p>

    <h2 lang="mh">Kein Ekkar im Bok</h2>
    <p lang="mh">Kein im jouj jelå ko rejebar.</p>
</div>
```

Each language segment marked with appropriate `lang` attribute.

### 10.6 The HTTP Header (Optional)

```nginx
# /etc/nginx/sites-enabled/marshallese-voices.org.conf

server {
    listen 443 ssl http2;
    server_name marshallese-voices.org;

    # English content
    location / {
        add_header Content-Language "en-US" always;
        # ...
    }

    # Marshallese content
    location /mh/ {
        add_header Content-Language "mh" always;
        # ...
    }
}
```

### 10.7 The UTF-8 Critical Reminder

Marshallese text uses diacritics and special characters (ā, ē, ī, ō, ū, etc.). Per framework-html-meta-charset.md Section 6 (Marshallese-Voices case study), UTF-8 chain must be end to end:

* HTML: `<meta charset="utf-8">`.
* nginx: `charset utf-8;`.
* Form submissions: `accept-charset="utf-8"`.
* Database: UTF-8 collation.

Without proper UTF-8 chain, Marshallese characters render as mojibake.

### 10.8 The Documentation Note

The Marshallese-Voices site is a Bubbles community focused build. The language handling is respect and accessibility focused; not pure SEO optimization. The cultural context (the broader Marshallese American community in Springdale) is referenced respectfully in associated framework documentation (framework-html-meta-description.md Section 17).

---

## 11. THE FEDERAL AND SDVOSB INTERNATIONAL CONTEXT

For Joseph's federal subcontracting work, multi language requirements may apply.

### 11.1 The Federal Multi Language Requirement

Many federal agencies require:

* Federal forms in English and Spanish (and sometimes more).
* Compliance with Plain Language Act of 2010.
* Section 504/508 accessibility (multi language as access).

For federal subcontracting, sites may need multi language support beyond standard US English.

### 11.2 The WeCoverUSA Pattern (Hypothetical Multi Language)

If WeCoverUSA were to serve federal agencies with Spanish language requirements:

```html
<!doctype html>
<html lang="en-US">
<head>
    <link rel="alternate" hreflang="en-US" href="https://wecoverusa.com/">
    <link rel="alternate" hreflang="es" href="https://wecoverusa.com/es/">
    <link rel="alternate" hreflang="x-default" href="https://wecoverusa.com/">

    <link rel="canonical" href="https://wecoverusa.com/">

    <!-- ... -->
</head>
```

Plus the Spanish version at `/es/`.

### 11.3 The Federal hreflang Considerations

For federal contracting context:

* **Strict reciprocity**: each language version must reference all others.
* **x-default for unmatched users**: clear default behavior.
* **No deprecated tags**: avoid `meta http-equiv="content-language"`.
* **Documented language policy**: in client project notes.

### 11.4 The Compliance Documentation

```
Language signaling implementation:
  Default: English (en-US)
  Alternate: Spanish (es)
  
  HTML attribute: <html lang="en-US"> on English; <html lang="es"> on Spanish.
  hreflang: reciprocal between en-US and es.
  HTTP Content-Language: set per route in nginx.
  
  No deprecated meta http-equiv="content-language" used.
  All BCP 47 language tags conform to standard format.
  
  Aligns with: federal multi language access requirements (where applicable).
```

### 11.5 The Audit Verification

```bash
# Verify federal subcontract language signaling
URL=https://wecoverusa.com/

# Get hreflang
curl -s "$URL" | grep -oE 'link rel="alternate" hreflang="[^"]+"'

# Verify each version has reciprocal hreflang
for lang in en-US es; do
    if [ "$lang" = "en-US" ]; then
        VERSION_URL="$URL"
    else
        VERSION_URL="$URL/${lang}/"
    fi

    echo "Checking $VERSION_URL:"
    curl -s "$VERSION_URL" | grep -oE 'link rel="alternate" hreflang="[^"]+"' | head -3
    echo ""
done
```

---

## 12. THE BROWSER AND TOOL BEHAVIOR

How browsers and crawlers handle language signals.

### 12.1 Browser Behavior

Modern browsers use `<html lang>` for:

* **Spell check**: identifying language for spell checker.
* **Hyphenation**: applying correct hyphenation rules.
* **Screen reader pronunciation**: TTS voice selection.
* **Font selection**: appropriate font for CJK characters.
* **Voice synthesis**: speech synthesis language.

Browsers do NOT use `<meta http-equiv="content-language">` for these purposes.

### 12.2 Google Behavior

* Reads `<html lang>` attribute.
* Reads `hreflang` link elements.
* Reads HTTP `Content-Language` header.
* Ignores `<meta http-equiv="content-language">` (since 2010).

### 12.3 Bing Behavior

Similar to Google. Modern signals via html lang.

### 12.4 Screen Reader Behavior

Screen readers (JAWS, NVDA, VoiceOver, TalkBack) use:

* `<html lang>` for primary document language.
* `lang` attribute on elements for inline language switches.

Critical for accessibility: marking Spanish content within English page with `lang="es"` ensures screen readers pronounce Spanish words correctly.

### 12.5 Wappalyzer And Tech Detection Tools

Some tech analysis tools parse `<meta http-equiv="content-language">` for CMS detection. The presence of the tag may signal a legacy WordPress site (since hand coded sites rarely use it).

### 12.6 The Practical Implication

For Bubbles:

* Set `<html lang>` (primary).
* Set `hreflang` for multi language sites.
* Set HTTP `Content-Language` header (optional secondary signal).
* Skip `<meta http-equiv="content-language">` (deprecated; legacy signal).

---

## 13. THE BUBBLES DECISION FRAMEWORK

Each client site needs a language signaling decision.

### 13.1 The Decision Questions

1. **What is the primary language?**
   - English: `en-US` for US based clients.
   - Spanish: `es-MX` or `es` per audience.
   - Marshallese: `mh`.
   - Multiple: design hreflang structure.

2. **Is the site multi language?**
   - No: just `<html lang>` and optional HTTP header.
   - Yes: full hreflang implementation.

3. **Are there mixed language sections?**
   - Yes: `lang` attribute on specific elements.
   - No: just document level.

4. **Is the http-equiv tag from a legacy CMS still present?**
   - Yes: remove during migration cleanup.
   - No: continue.

### 13.2 The Per Client Recommendations

**ThatDeveloperGuy.com**:
```html
<html lang="en-US">
<!-- No meta http-equiv -->
<!-- No hreflang (single language) -->
```

**TCB Fight Factory**:
```html
<html lang="en-US">
<!-- Similar pattern -->
```

**Arkansas Counseling and Wellness**:
```html
<html lang="en-US">
<!-- English only; standard pattern -->
```

**Handled Tax (Amanda Emerdinger)**:
```html
<html lang="en-US">
<!-- English only -->
```

**Marshallese-Voices**:
```html
<!-- English pages -->
<html lang="en-US">
<link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
<link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
<link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

<!-- Marshallese pages -->
<html lang="mh">
<!-- Same hreflang block, with Marshallese version as canonical -->
```

**White River Cabins**:
```html
<html lang="en-US">
<!-- English only -->
```

**Greenough's Guide Service**:
```html
<html lang="en-US">
<!-- English only -->
```

**Local Living Real Estate**:
```html
<html lang="en-US">
```

**Diana Undergust (Lake Homes Realty)**:
```html
<html lang="en-US">
```

**WeCoverUSA (federal)**:
```html
<!-- English primary, Spanish if required -->
<html lang="en-US">
<!-- hreflang if multi language required -->
```

**Heritage Hardwood Floors NWA**:
```html
<html lang="en-US">
```

### 13.3 The Documentation Pattern

For each client project file:

```
Language signaling:
  Primary language: en-US
  Multi language: no
  
  HTML attribute: <html lang="en-US"> on every page
  HTTP header: Content-Language set in nginx (optional)
  hreflang: not applicable (single language)
  http-equiv meta: omitted (deprecated by Google)
```

For Marshallese-Voices specifically:

```
Language signaling:
  Primary language: English (en-US)
  Alternate: Marshallese (mh)
  
  HTML attribute: <html lang="en-US"> on English; <html lang="mh"> on Marshallese
  HTTP header: Content-Language set per route in nginx
  hreflang: reciprocal between en-US and mh; x-default to English
  http-equiv meta: omitted
  
  Mixed content: lang="mh" on Marshallese elements within English pages
  UTF-8: full chain end to end (HTML, nginx, forms, database)
```

---

## 14. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 14.1 Canonical Bubbles head (single language English)

```html
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- NO meta http-equiv="content-language" (deprecated) -->
    <!-- HTML lang attribute above is sufficient -->

    <!-- ... -->
</head>
```

### 14.2 nginx HTTP header (optional addition)

```nginx
# /etc/nginx/sites-enabled/example.com.conf

server {
    listen 443 ssl http2;
    server_name example.com;

    # Content-Language (optional supplement to html lang)
    add_header Content-Language "en-US" always;

    # ... other directives ...
}
```

```bash
nginx -t && systemctl reload nginx

# Verify
curl -sI https://example.com/ | grep -i "content-language"
# Expected: content-language: en-US
```

### 14.3 Marshallese-Voices English page

```html
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Marshallese-Voices | Community Resources</title>
    <meta name="description" content="Resources for the Marshallese American community in Northwest Arkansas. Available in English and Marshallese.">

    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <link rel="canonical" href="https://marshallese-voices.org/">

    <meta property="og:locale" content="en_US">
    <meta property="og:locale:alternate" content="mh">
</head>
<body>
    <h1>Marshallese-Voices</h1>

    <p>Community resources for the Marshallese American community in Northwest Arkansas.</p>

    <p>
        <a href="/mh/" hreflang="mh" lang="mh">Yokwe! Bok kin Kajin Kajin Majol.</a>
        (Click here for Marshallese.)
    </p>

    <!-- ... content ... -->
</body>
</html>
```

### 14.4 Marshallese-Voices Marshallese page

```html
<!doctype html>
<html lang="mh">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Marshallese-Voices | Kein Ekkar im Jipan</title>
    <meta name="description" content="Kein ekkar im jipan an Marshallese American ilo Northwest Arkansas.">

    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">

    <link rel="canonical" href="https://marshallese-voices.org/mh/">

    <meta property="og:locale" content="mh">
    <meta property="og:locale:alternate" content="en_US">
</head>
<body>
    <h1>Marshallese-Voices</h1>

    <!-- Marshallese content -->

    <p>
        <a href="/" hreflang="en-US" lang="en-US">Click here for English / Bok kin English.</a>
    </p>
</body>
</html>
```

### 14.5 nginx multi language

```nginx
# /etc/nginx/sites-enabled/marshallese-voices.org.conf

server {
    listen 443 ssl http2;
    server_name marshallese-voices.org;

    charset utf-8;  # required for Marshallese characters

    # English (default)
    location / {
        add_header Content-Language "en-US" always;
        root /var/www/sites/marshallese-voices.org/en/;
        try_files $uri $uri/ /index.html;
    }

    # Marshallese
    location /mh/ {
        add_header Content-Language "mh" always;
        alias /var/www/sites/marshallese-voices.org/mh/;
        try_files $uri $uri/ /mh/index.html;
    }

    # ... ssl config etc ...
}
```

### 14.6 Mixed language section

```html
<article lang="en-US">
    <h2>Community Translation Services</h2>
    <p>Translation services available between English and Marshallese.</p>

    <blockquote lang="mh">
        <p>Kein im jouj kojrawe ko rejebar.</p>
        <p>(Translation services available.)</p>
    </blockquote>

    <p>Contact us for assistance.</p>
</article>
```

### 14.7 Remove legacy meta http-equiv

```bash
# Find files with the deprecated tag
find /var/www/sites/ -name "*.html" -type f -exec \
    grep -l 'meta http-equiv="content-language"' {} \;

# Remove
find /var/www/sites/ -name "*.html" -type f -exec \
    sed -i '/meta http-equiv="content-language"/d' {} \;

# Verify
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    if find "$site_dir" -name "*.html" -exec grep -l 'meta http-equiv="content-language"' {} \; | head -1 > /dev/null 2>&1; then
        echo "STILL HAS LEGACY: $SITE"
    fi
done
```

### 14.8 Bulk audit Bubbles client sites

```bash
#!/bin/bash
# /usr/local/bin/language-audit.sh

echo "=== Language signaling audit ==="

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"

    if [ -f "$INDEX" ]; then
        # Check for html lang
        HTML_LANG=$(grep -oE '<html[^>]*lang="[^"]+"' "$INDEX" | head -1 | sed 's/.*lang="\([^"]*\)".*/\1/')

        # Check for deprecated http-equiv
        HAS_HTTP_EQUIV=$(grep -c 'meta http-equiv="content-language"' "$INDEX")

        # Check for hreflang
        HAS_HREFLANG=$(grep -c 'rel="alternate" hreflang=' "$INDEX")

        if [ -z "$HTML_LANG" ]; then
            echo "MISSING HTML LANG: $SITE"
        else
            echo "$SITE: html lang=$HTML_LANG"
        fi

        if [ "$HAS_HTTP_EQUIV" -gt "0" ]; then
            echo "  Has deprecated http-equiv content-language ($HAS_HTTP_EQUIV occurrences)"
        fi

        if [ "$HAS_HREFLANG" -gt "0" ]; then
            echo "  Has hreflang ($HAS_HREFLANG occurrences)"
        fi
    fi
done

echo ""
echo "=== End audit ==="
```

### 14.9 FastAPI multi language routing

```python
# /opt/bubbles/services/marshallese-voices.org/main.py

from fastapi import FastAPI, Request, Response
from fastapi.templating import Jinja2Templates

app = FastAPI()
templates = Jinja2Templates(directory="templates")

LANGUAGE_TO_TEMPLATE_DIR = {
    "en": "en",
    "en-US": "en",
    "mh": "mh",
}

@app.get("/")
async def english_homepage(request: Request, response: Response):
    response.headers["Content-Language"] = "en-US"
    return templates.TemplateResponse("en/homepage.html", {
        "request": request,
        "language": "en-US",
    })

@app.get("/mh/")
async def marshallese_homepage(request: Request, response: Response):
    response.headers["Content-Language"] = "mh"
    return templates.TemplateResponse("mh/homepage.html", {
        "request": request,
        "language": "mh",
    })
```

```html
{# templates/en/base.html #}
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <link rel="alternate" hreflang="en-US" href="https://marshallese-voices.org/">
    <link rel="alternate" hreflang="mh" href="https://marshallese-voices.org/mh/">
    <link rel="alternate" hreflang="x-default" href="https://marshallese-voices.org/">
    <link rel="canonical" href="https://marshallese-voices.org/">
    <!-- ... -->
</head>
```

---

## 15. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles language signaling configuration.

### 15.1 The Single Language Template

```html
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- NO meta http-equiv="content-language" (deprecated) -->

    <!-- ... -->
</head>
```

### 15.2 The Multi Language Template

```html
<!doctype html>
<html lang="{{ page_lang }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    {% for alt in alternate_languages %}
    <link rel="alternate" hreflang="{{ alt.lang }}" href="{{ alt.url }}">
    {% endfor %}
    <link rel="alternate" hreflang="x-default" href="{{ default_url }}">

    <link rel="canonical" href="{{ canonical_url }}">

    <meta property="og:locale" content="{{ og_locale }}">
    {% for alt in alternate_languages if alt.lang != page_lang %}
    <meta property="og:locale:alternate" content="{{ alt.og_locale }}">
    {% endfor %}

    <!-- ... -->
</head>
```

### 15.3 The Per Client Configuration

```python
# /opt/bubbles/services/example.com/client_config.py

CLIENT_LANGUAGES = {
    "thatdeveloperguy": {
        "primary": "en-US",
        "alternates": [],
    },
    "tcb_fight_factory": {
        "primary": "en-US",
        "alternates": [],
    },
    "arcounselingandwellness": {
        "primary": "en-US",
        "alternates": [],
    },
    "handledtax": {
        "primary": "en-US",
        "alternates": [],
    },
    "marshallese_voices": {
        "primary": "en-US",
        "alternates": [
            {"lang": "mh", "url_prefix": "/mh/", "og_locale": "mh"},
        ],
    },
    "wecoverusa": {
        "primary": "en-US",
        "alternates": [],  # add Spanish if federal multilingual required
    },
    # ... per client
}
```

### 15.4 The Audit Workflow

```bash
# Weekly across all sites
language-audit.sh

# After nginx config changes
nginx -t && systemctl reload nginx

# Verify
curl -sI https://example.com/ | grep -i "content-language"
curl -s https://example.com/ | grep -oE '<html[^>]+lang="[^"]+"'
```

### 15.5 The Client Documentation

For each client project file:

```
Language signaling:
  Primary language: {{ primary }}
  Multi language: {{ yes/no }}
  Alternates: {{ list }}
  
  HTML attribute: <html lang="{{ primary }}"> on every page
  HTTP header: Content-Language set via nginx (optional)
  hreflang: {{ implemented/not applicable }}
  http-equiv meta: OMITTED (deprecated by Google)
  
  Special considerations: {{ e.g., UTF-8 chain for Marshallese }}
```

---

## 16. AUDIT CHECKLIST

Run through these 35 items for production deployment.

### Core html lang attribute

1. [ ] `<html lang="...">` present on every page.
2. [ ] Language value is valid BCP 47 (e.g., `en-US` not `en_US`).
3. [ ] Language matches actual page content language.
4. [ ] Consistent across all pages of same language.

### Deprecated tag absence

5. [ ] No `<meta http-equiv="content-language">` on any page.
6. [ ] Legacy CMS migrations have removed the deprecated tag.
7. [ ] Pre deploy check rejects new additions of the deprecated tag.

### HTTP header (optional)

8. [ ] If used: `Content-Language` HTTP header set in nginx.
9. [ ] Header value matches `<html lang>` attribute.
10. [ ] Header applied with `always` keyword.

### Multi language sites

11. [ ] Each language version has correct `<html lang>`.
12. [ ] hreflang link elements present for all language versions.
13. [ ] Reciprocity: each version references all others.
14. [ ] x-default fallback included.
15. [ ] Canonical URLs distinct per language version.
16. [ ] Open Graph locale included.

### Mixed language content

17. [ ] `lang` attribute used on elements with different language.
18. [ ] Screen reader correctly identifies language switches.
19. [ ] No untagged language switches in body content.

### Marshallese-Voices specific

20. [ ] English pages: `<html lang="en-US">`.
21. [ ] Marshallese pages: `<html lang="mh">`.
22. [ ] hreflang reciprocity en-US and mh.
23. [ ] x-default points to English.
24. [ ] UTF-8 chain verified end to end (per framework-html-meta-charset.md).
25. [ ] Mixed content sections have `lang="mh"` on Marshallese elements.

### Federal subcontractor

26. [ ] If multi language required: full hreflang implementation.
27. [ ] BCP 47 codes correctly formatted.
28. [ ] Documentation per contract requirements.

### Browser and accessibility

29. [ ] Screen readers correctly identify document language.
30. [ ] Spell check applies appropriate language rules.
31. [ ] Font rendering appropriate for language.

### Tooling and process

32. [ ] `language-audit.sh` script in `/usr/local/bin/`.
33. [ ] Pre deploy check verifies html lang present.
34. [ ] Pre deploy check verifies no deprecated http-equiv.

### Documentation

35. [ ] Per client language decision documented in project notes.

A site that passes all 35 has correctly configured language signaling for browsers, search engines, accessibility tools, and (where applicable) multi language audiences.

---

## 17. COMMON PITFALLS

Thirteen patterns to recognize and avoid.

**Pitfall 1: Missing `<html lang>` attribute.**
Symptom: page renders but no language signal for browsers or search engines.
Why it breaks: html lang is the primary signal.
Fix: add `<html lang="en-US">` (or appropriate BCP 47).

**Pitfall 2: Using deprecated http-equiv instead of html lang.**
Symptom: page has `<meta http-equiv="content-language" content="en-US">` but no html lang.
Why it breaks: Google ignores the meta since 2010.
Fix: replace with `<html lang="en-US">`.

**Pitfall 3: Both http-equiv meta and html lang present.**
Symptom: redundant signals.
Why it breaks: no harm but legacy CMS smell.
Fix: remove the meta tag; keep html lang.

**Pitfall 4: Wrong BCP 47 format.**
Symptom: `en_US` (underscore) instead of `en-US` (hyphen).
Why it breaks: BCP 47 uses hyphens; underscores invalid.
Fix: use hyphens.

**Pitfall 5: Just `en` without region.**
Symptom: `<html lang="en">` without region.
Why it breaks: less specific; may produce regional mismatches.
Fix: use `en-US` for US-based content.

**Pitfall 6: Mismatch between html lang and HTTP Content-Language.**
Symptom: html says `en-US`, HTTP says `es`.
Why it breaks: conflicting signals.
Fix: align both.

**Pitfall 7: Multi language site without hreflang.**
Symptom: English and Spanish versions exist but no hreflang links.
Why it breaks: Google can't determine which version is for which audience.
Fix: add reciprocal hreflang links.

**Pitfall 8: hreflang without reciprocity.**
Symptom: English page lists Spanish; Spanish page doesn't list English.
Why it breaks: Google requires bidirectional confirmation.
Fix: each version must list all others.

**Pitfall 9: Marshallese content untagged within English page.**
Symptom: `<p>Yokwe!</p>` (Marshallese text) inside English page without `lang="mh"`.
Why it breaks: screen reader pronounces with English rules.
Fix: `<p lang="mh">Yokwe!</p>`.

**Pitfall 10: Inconsistent language across same language pages.**
Symptom: homepage `<html lang="en-US">`; about page `<html lang="en">`; services page `<html lang="en-US">`.
Why it breaks: fragmented signal.
Fix: standardize to one canonical code (typically `en-US` for US clients).

**Pitfall 11: WordPress migration left http-equiv tag.**
Symptom: hand coded Bubbles site has WordPress era meta http-equiv.
Why it breaks: stale tag; signals WordPress legacy via Wappalyzer.
Fix: remove during migration cleanup.

**Pitfall 12: hreflang URL doesn't match canonical.**
Symptom: hreflang says `https://example.com/es/` but canonical says `https://example.com/es/page/`.
Why it breaks: Google may use wrong canonical.
Fix: align hreflang URL with canonical URL.

**Pitfall 13: UTF-8 chain broken for Marshallese.**
Symptom: Marshallese characters render as mojibake despite `lang="mh"`.
Why it breaks: lang attribute doesn't fix encoding.
Fix: ensure full UTF-8 chain per framework-html-meta-charset.md.

---

## 18. DIAGNOSTIC COMMANDS

Reference of commands useful for language signaling investigation.

### Inspect a single URL

```bash
# Get html lang attribute
curl -s https://example.com/ | grep -oE '<html[^>]+lang="[^"]+"' | head -1

# Get meta http-equiv content-language (should be absent)
curl -s https://example.com/ | grep -c 'meta http-equiv="content-language"'

# Get hreflang
curl -s https://example.com/ | grep -oE 'link rel="alternate" hreflang="[^"]+"' | head -5

# Get HTTP Content-Language header
curl -sI https://example.com/ | grep -i "content-language"
```

### Verify reciprocity

```bash
# For multi language sites, verify each version lists all others
for url in https://example.com/ https://example.com/es/ https://example.com/mh/; do
    echo "=== $url ==="
    HREFLANGS=$(curl -s "$url" | grep -oE 'hreflang="[^"]+"' | sort -u)
    echo "$HREFLANGS"
    echo ""
done
```

### Find deprecated http-equiv tags

```bash
# Across all client sites
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"
    if [ -f "$INDEX" ]; then
        if grep -q 'meta http-equiv="content-language"' "$INDEX"; then
            echo "DEPRECATED TAG: $SITE"
            grep 'meta http-equiv="content-language"' "$INDEX" | head -1
        fi
    fi
done
```

### Bulk audit

```bash
language-audit.sh
```

### Lighthouse SEO audit

```bash
# Lighthouse checks for html lang attribute
lighthouse https://example.com/ \
    --only-audits=document-has-html-lang,html-has-lang \
    --output=json --output-path=/tmp/lh.json --quiet

python3 -c "
import json
with open('/tmp/lh.json') as f:
    data = json.load(f)
for audit_id, audit in data.get('audits', {}).items():
    if 'lang' in audit_id:
        print(f'{audit_id}: {audit.get(\"score\")} - {audit.get(\"title\")}')
"
```

### Google Search Console verification

For multi language sites, use Search Console > International Targeting report:

* Confirms hreflang implementation.
* Shows any reciprocity errors.
* Lists language/region targeting.

### Browser verification

In Chrome DevTools:

1. Open the page.
2. Console: `document.documentElement.lang`.
3. Should return the html lang value.

For accessibility:

* Use ChromeVox extension or similar.
* Verify screen reader announces page language.
* Verify language switches are detected.

### After language changes

```bash
nginx -t && systemctl reload nginx
systemctl restart fastapi-sidecar

# Verify
language-audit.sh
```

---

## 19. CROSS-REFERENCES

* [framework-http-content-headers.md](framework-http-content-headers.md): the HTTP `Content-Language:` response header is the canonical server side layer for language signaling.
* [framework-html-meta-charset.md](framework-html-meta-charset.md): UTF-8 chain is critical for languages with special characters (Marshallese-Voices case study in both frameworks).
* [framework-html-meta-description.md](framework-html-meta-description.md): description per language; Marshallese-Voices respectful local context.
* [framework-html-meta-viewport.md](framework-html-meta-viewport.md): order in head.
* [framework-html-meta-color-scheme.md](framework-html-meta-color-scheme.md): order in head.
* [framework-html-meta-referrer.md](framework-html-meta-referrer.md): order in head.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): language and international targeting are part of the broader ranking framework.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including language signaling per client.
* W3C HTML5 lang attribute: https://www.w3.org/TR/html52/dom.html#the-lang-and-xml-lang-attributes
* Google deprecation announcement (2010): https://developers.google.com/search/blog/2010/03/working-with-multilingual-websites
* Google Search Central on language targeting: https://developers.google.com/search/docs/specialty/international/managing-multi-regional-sites
* MDN meta http-equiv=content-language: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta
* IETF BCP 47: https://tools.ietf.org/html/bcp47
* IANA Language Subtag Registry: https://www.iana.org/assignments/language-subtag-registry/language-subtag-registry
* W3C Internationalization on language tags: https://www.w3.org/International/articles/language-tags/
* Marshallese language (Wikipedia): https://en.wikipedia.org/wiki/Marshallese_language

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Use `<html lang="en-US">` (or appropriate BCP 47). Skip `<meta http-equiv="content-language">`. Use hreflang for multi language sites.**

### The canonical patterns

```html
<!-- Standard single language (most Bubbles clients) -->
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <!-- ... no http-equiv content-language ... -->
</head>
```

```html
<!-- Multi language (Marshallese-Voices pattern) -->
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <link rel="alternate" hreflang="en-US" href="https://example.com/">
    <link rel="alternate" hreflang="mh" href="https://example.com/mh/">
    <link rel="alternate" hreflang="x-default" href="https://example.com/">
</head>
```

### The forbidden pattern (legacy)

```html
<!-- Deprecated by Google in 2010; redundant with html lang -->
<meta http-equiv="content-language" content="en-US">
```

### Five rules to memorize

1. **`<html lang="en-US">` (or other BCP 47) on every page.**
2. **Skip `<meta http-equiv="content-language">` (Google deprecated 2010).**
3. **Use BCP 47 format with hyphen (`en-US` not `en_US`).**
4. **Multi language: hreflang reciprocity + x-default.**
5. **Mixed language sections: `lang="..."` on the specific elements.**

### Five commands every operator should know

```bash
# 1. Check html lang
curl -s URL | grep -oE '<html[^>]+lang="[^"]+"' | head -1

# 2. Check for deprecated tag (should be 0)
curl -s URL | grep -c 'meta http-equiv="content-language"'

# 3. Check hreflang
curl -s URL | grep -oE 'rel="alternate" hreflang="[^"]+"'

# 4. Audit across all sites
language-audit.sh

# 5. Apply changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. html lang present and valid BCP 47
URL=https://example.com/
LANG=$(curl -s "$URL" | grep -oE '<html[^>]+lang="[^"]+"' | head -1 | sed 's/.*lang="\([^"]*\)".*/\1/')
if [[ "$LANG" =~ ^[a-z]+(-[A-Z]+)?$ ]]; then
    echo "OK: lang=$LANG (valid BCP 47)"
else
    echo "FAIL: lang=$LANG (invalid or missing)"
fi

# 2. No deprecated http-equiv
HAS_LEGACY=$(curl -s "$URL" | grep -c 'meta http-equiv="content-language"')
[ "$HAS_LEGACY" = "0" ] && echo "OK: no deprecated tag" || echo "FAIL: $HAS_LEGACY legacy tags"

# 3. For multi language: hreflang reciprocity verified
# (Manual check; see Section 18)
```

If all three pass AND screen reader correctly identifies language AND search engines correctly classify content by language, the language signaling layer is correctly wired.

---

End of framework-html-meta-content-language.md.
