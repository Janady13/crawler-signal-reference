# framework-http-content-headers.md

Comprehensive reference for the five HTTP response headers that describe what the response body is, how it is encoded on the wire, how big it is, and what the client should do with it. Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to framework-http-caching-headers.md, UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

Audience: humans configuring nginx, AI assistants generating or repairing nginx config, and anyone troubleshooting "wrong content type", "garbled compressed response", "browser displaying instead of downloading", or "language metadata not picked up" anomalies on a self hosted stack.

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Content Description Mental Model (read this first)
5. Content-Type (the most important header on any response)
6. Content-Language (the i18n breadcrumb Bing and Baidu actually use)
7. Content-Encoding (compression on the wire)
8. Content-Length (the framing contract)
9. Content-Disposition (inline vs attachment, filenames, security)
10. How These Headers Interact
11. Asset Class Recipes (HTML, JSON, CSS/JS, images, fonts, PDF, fonts CORS, downloads)
12. Bubbles Nginx Reference Block (paste ready)
13. Audit Checklist (50+ items)
14. Common Pitfalls
15. Diagnostic Commands (curl, file, mime detection, browser devtools)
16. Cross-References

---

## 1. DEFINITION

Content description headers tell the client exactly what payload it just received: the media type, the natural language, the compression applied for transport, the byte length, and whether to display inline or save to disk. The five headers split into three concerns:

* **Identity**: `Content-Type`, `Content-Language`. They answer "what is this and which language is it in?"
* **Transport encoding**: `Content-Encoding`. It answers "how were the bytes transformed for the wire?"
* **Framing and handling**: `Content-Length`, `Content-Disposition`. They answer "how many bytes is the body?" and "should the browser render it or prompt to save it?"

Together these headers tell the client everything it needs to render, parse, decompress, or download the response correctly. Getting any one of them wrong produces visible failures: garbled characters, mis indexed pages, browsers refusing to display content, downloads with the wrong filename, security warnings, or in the worst case, an XSS vector.

---

## 2. WHY IT MATTERS

Five independent pressures push correct content headers from "polish" to "required infrastructure" in 2025 and forward.

**Mime type sniffing is dead.** Modern browsers (Chrome, Firefox, Safari, Edge) all honor `X-Content-Type-Options: nosniff` and refuse to execute or render a response whose `Content-Type` does not match what the markup or script tag expects. A CSS file served as `text/plain` will not load. A JavaScript module served as `text/html` will be refused. A font served without `font/woff2` or proper MIME will not render. The forgiveness of the 2010s is gone.

**Compression is the single biggest performance lever.** A typical HTML response compresses 70 to 85 percent with gzip, 80 to 90 percent with brotli, and 85 to 92 percent with zstd. Pages that ship uncompressed bytes are paying 5x to 10x in TTFB and LCP for no reason. Getting `Content-Encoding` right (and the negotiation chain that produces it) is the difference between a 1.2 second LCP and a 4.5 second LCP on the same content.

**International SEO without `Content-Language` is incomplete.** Google reads `hreflang` and largely ignores `Content-Language`, but Bing, Baidu, and Yandex use `Content-Language` as a primary signal. A site serving multiple regions or languages that omits `Content-Language` cedes ranking in those engines. Bing serves the default search experience for 30+ percent of US desktop traffic when Copilot, ChatGPT search, and Edge defaults are aggregated. AI engines that ingest Bing's index inherit the same dependency.

**Content-Disposition is a security surface.** Misconfigured `Content-Disposition` is a documented attack vector for reflected file download (RFD) exploits, cross site scripting via uploaded files, and filename based traversal. The header must be set correctly on every endpoint that returns user uploaded content or generated downloads.

**Content-Length and Transfer-Encoding desync is exploit material.** HTTP request smuggling, the class of attack that compromised major CDNs throughout 2019 to 2023, relies on disagreements between `Content-Length` and `Transfer-Encoding: chunked` at different layers. While modern nginx handles this correctly by default, any custom upstream that emits both can reopen the vulnerability.

**Cost of getting it wrong.** Misconfigured content headers produce silent and loud failures. Loud examples from production sites:

* A CSS file is served as `application/octet-stream` because a custom mime type map is missing the entry. Pages render unstyled. Lighthouse score collapses.
* A JSON API endpoint sends `Content-Type: text/html`. Every client error handler that does `response.json()` throws on parse. Frontend breaks silently.
* A site is served with `Content-Type: text/html` but no charset. Browser guesses Windows 1252 and displays mojibake instead of the intended UTF-8. Search snippets show the broken characters in SERPs.
* A PDF download endpoint omits `Content-Disposition: attachment`. Browser tries to render the PDF inline, which can trigger plugin prompts on older clients and looks unprofessional on all of them.
* A gzipped response goes out without `Content-Encoding: gzip`. Browser receives binary garbage, fails to render, throws console errors.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the five headers gets the same six part treatment:

1. **What it does**: the canonical RFC 9110 or RFC 6266 definition plus the practical implication.
2. **Syntax and values**: every legal value, what it means, and when it is wrong.
3. **How to build it on Bubbles**: paste ready nginx config.
4. **How to verify it**: curl commands that prove the header is set and behaving.
5. **How to troubleshoot**: the four or five failure modes seen in the field and how to recognize each.
6. **How to fix common breakage**: ordered repair steps.

Recipes for the most common asset classes are collected in Section 11 so a builder can copy a block per file type without rederiving from first principles.

---

## 4. THE CONTENT DESCRIPTION MENTAL MODEL (READ THIS FIRST)

Every HTTP client that receives a response runs the same dispatch loop. Internalize it and every header choice becomes obvious.

```
Response arrives
        |
        v
Read Content-Encoding header
        |
        v
Is the body compressed?
   |                            |
  YES                          NO
   |                            |
   v                            v
Decompress using algorithm    Skip decompression
in Content-Encoding
   (gzip / br / zstd / deflate)
        |
        v
Read Content-Type header
        |
        v
What is this?
   |
   v
text/html  -> parse as HTML, render in document context
text/css   -> parse as stylesheet, only if <link rel=stylesheet>
application/javascript -> parse and execute as script
application/json -> parse as JSON
image/*    -> decode and render
application/pdf -> render inline (PDF viewer) OR prompt to save
                   (decided by Content-Disposition)
        |
        v
Read Content-Disposition
        |
        v
inline?    -> render in page context
attachment? -> prompt download with given filename
        |
        v
Read Content-Language (if set)
        |
        v
Pass to language tools, accessibility, search indexers
```

Three rules govern the entire system:

1. **Content-Encoding is reversed before everything else.** It is purely a transport optimization. The client must decompress before doing anything else with the body.
2. **Content-Type is authoritative.** With `X-Content-Type-Options: nosniff` set (which is the Bubbles default), the browser will not second guess the declared MIME type. If you say `text/plain`, your JavaScript will not execute. Period.
3. **Content-Disposition decides the UX.** `inline` says "show it"; `attachment` says "save it". The default when the header is absent depends on the MIME type and browser, which is unpredictable. Always set explicitly for non standard delivery.

A correct header set must produce sensible behavior across every browser, every search engine crawler, every AI crawler, every accessibility tool, and every command line client (curl, wget, AI scrapers using Python requests, automation pipelines). The same response, the same headers, the same outcome everywhere.

---

## 5. CONTENT-TYPE (THE MOST IMPORTANT HEADER ON ANY RESPONSE)

### 5.1 What It Does

`Content-Type` declares the MIME type of the response body and, for text types, the character encoding (charset). Defined in RFC 9110 Section 8.3.

```
Content-Type: text/html; charset=utf-8
Content-Type: application/json
Content-Type: image/webp
Content-Type: font/woff2
Content-Type: application/pdf
```

The MIME type tells the browser how to dispatch the response: render it, execute it, decode it, or download it. The charset tells the browser how to decode the byte stream into characters. Without an accurate `Content-Type`, the browser falls back to MIME sniffing or to defaults that are almost always wrong.

Modern Bubbles sites set `X-Content-Type-Options: nosniff` globally, which **disables** MIME sniffing entirely. With nosniff active, an incorrect `Content-Type` does not get rescued by the browser's guess; the resource simply fails to load.

### 5.2 Syntax And Common Values

Format: `<type>/<subtype>` optionally followed by `; <parameter>=<value>` pairs.

```
Content-Type: text/html; charset=utf-8
Content-Type: application/javascript; charset=utf-8
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
```

**Reference table of MIME types you will actually use on a static or near static site:**

| File extension | Correct Content-Type | Notes |
|---|---|---|
| `.html`, `.htm` | `text/html; charset=utf-8` | Always include charset |
| `.css` | `text/css; charset=utf-8` | Charset matters for non ASCII content (rare but possible) |
| `.js`, `.mjs` | `application/javascript; charset=utf-8` or `text/javascript; charset=utf-8` | Both are valid; nginx uses `application/javascript` by default |
| `.json` | `application/json; charset=utf-8` | RFC 8259 says JSON is always UTF-8; charset is technically redundant but harmless |
| `.xml` | `application/xml; charset=utf-8` or `text/xml; charset=utf-8` | Use `application/xml` for machine readable, `text/xml` for human readable. Sitemaps use `application/xml` |
| `.txt` | `text/plain; charset=utf-8` | robots.txt, humans.txt, ai.txt, llms.txt all use this |
| `.csv` | `text/csv; charset=utf-8` | |
| `.svg` | `image/svg+xml` | SVG is XML; renders as image |
| `.png` | `image/png` | |
| `.jpg`, `.jpeg` | `image/jpeg` | Note: `jpeg` not `jpg` |
| `.gif` | `image/gif` | |
| `.webp` | `image/webp` | |
| `.avif` | `image/avif` | Newer; some older mime.types files lack it |
| `.ico` | `image/vnd.microsoft.icon` or `image/x-icon` | Browsers accept both |
| `.woff2` | `font/woff2` | Modern format. Older mime.types may say `application/font-woff2`, which works but is deprecated |
| `.woff` | `font/woff` | |
| `.ttf` | `font/ttf` | |
| `.otf` | `font/otf` | |
| `.pdf` | `application/pdf` | |
| `.zip` | `application/zip` | |
| `.tar` | `application/x-tar` | |
| `.gz` | `application/gzip` | The file itself; do not confuse with Content-Encoding: gzip |
| `.mp4` | `video/mp4` | |
| `.webm` | `video/webm` | |
| `.mp3` | `audio/mpeg` | Note: `mpeg` not `mp3` |
| `.ogg` | `audio/ogg` | |
| `.wasm` | `application/wasm` | Required for WebAssembly streaming compilation |
| `.manifest`, `.webmanifest` | `application/manifest+json` | PWA manifest |
| `.map` | `application/json` | Source maps are JSON |
| `.glb` | `model/gltf-binary` | 3D model |
| `.gltf` | `model/gltf+json` | 3D model JSON |

**The `application/octet-stream` value** is the universal "I do not know what this is, treat it as raw bytes". Browsers will refuse to render or execute it and will prompt for download. Seeing this on a CSS, JS, or image response means your mime type map is missing the extension.

### 5.3 Charset: Always UTF-8

For any text type, always declare `charset=utf-8`. The exceptions are vanishingly rare (legacy systems serving Shift-JIS or Big5, which Bubbles does not host).

Why charset matters: a UTF-8 byte sequence interpreted as Windows-1252 produces mojibake. The em dash byte sequence `\xE2\x80\x94` shows up as `â€"`. Apostrophes become `â€™`. The site looks broken in Latin extended scripts (French, Spanish, Portuguese, German with umlauts) and is illegible in Cyrillic, Greek, Arabic, or CJK.

Three places to declare UTF-8, in order of authority:

1. HTTP `Content-Type` header (most authoritative): `Content-Type: text/html; charset=utf-8`
2. HTML `<meta charset="utf-8">` (must be in the first 1024 bytes of the body): `<meta charset="utf-8">`
3. HTML `<meta http-equiv="Content-Type">` (legacy, redundant if either of the above is set): `<meta http-equiv="Content-Type" content="text/html; charset=utf-8">`

The HTTP header wins if it disagrees with the meta tag. Set both for belt and suspenders. The meta tag is what crawlers without HTTP context (saved HTML, archive snapshots) will use.

### 5.4 How Nginx Determines Content-Type

Nginx looks up the file extension in the mime type map, which is loaded from `/etc/nginx/mime.types` (or wherever `include mime.types;` points). The default Debian mime.types file is comprehensive but occasionally outdated. Recent additions like `.avif`, `.webmanifest`, and `.wasm` may need to be added manually.

The lookup logic:

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    # ... server blocks ...
}
```

If no mime type is found for the extension, nginx falls back to `default_type`. The recommended default is `application/octet-stream`, which forces a download prompt and prevents the wrong type from being assumed. The dangerous alternative `text/html` (a frequent default in old configs) would cause unknown files to be parsed as HTML, which is an XSS surface for any uploaded content.

To add or override mime types without editing the system mime.types file, use a `types` block in your server or location:

```nginx
http {
    include mime.types;
    default_type application/octet-stream;

    # Add modern types missing from old mime.types
    types {
        image/avif                       avif;
        font/woff2                       woff2;
        application/wasm                 wasm;
        application/manifest+json        webmanifest;
        model/gltf-binary                glb;
        model/gltf+json                  gltf;
        application/ld+json              jsonld;
    }
}
```

### 5.5 Adding Charset Automatically

Nginx adds the charset to `Content-Type` for matching types when `charset` is set:

```nginx
http {
    charset utf-8;
    charset_types
        text/html
        text/css
        text/plain
        text/xml
        text/csv
        application/javascript
        application/json
        application/xml
        application/atom+xml
        application/rss+xml
        application/ld+json
        image/svg+xml;
}
```

Output:

```
Content-Type: text/html; charset=utf-8
Content-Type: application/javascript; charset=utf-8
Content-Type: application/json; charset=utf-8
```

The `charset_types` directive controls which MIME types get the charset appended. Image types and binary types should never get a charset (it would be meaningless and might confuse strict parsers).

### 5.6 How To Build It On Bubbles

The complete content type stanza for a Bubbles `nginx.conf` http block:

```nginx
http {
    # Load the system mime types map
    include /etc/nginx/mime.types;

    # Safe default: force download for unknown types
    default_type application/octet-stream;

    # Add modern types missing from /etc/nginx/mime.types on older systems
    types {
        image/avif                       avif;
        font/woff2                       woff2;
        application/wasm                 wasm;
        application/manifest+json        webmanifest manifest;
        application/ld+json              jsonld;
        model/gltf-binary                glb;
        model/gltf+json                  gltf;
        text/markdown                    md;
    }

    # Automatic charset on text types
    charset utf-8;
    charset_types
        text/html
        text/css
        text/plain
        text/xml
        text/csv
        application/javascript
        application/json
        application/xml
        application/atom+xml
        application/rss+xml
        application/ld+json
        image/svg+xml;

    # Refuse MIME sniffing site wide
    add_header X-Content-Type-Options "nosniff" always;
}
```

For an upstream FastAPI sidecar that returns content with its own Content-Type, no nginx configuration is needed; the upstream's `Content-Type` is passed through automatically. To force or override:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_hide_header Content-Type;
    add_header Content-Type "application/json; charset=utf-8" always;
}
```

This is rarely the right approach. Fix the upstream to emit the correct Content-Type instead.

### 5.7 How To Verify

```bash
# 1. Confirm Content-Type and charset for HTML
curl -sI https://example.com/ | grep -i content-type
# Expected: content-type: text/html; charset=utf-8

# 2. Check every common asset type
for ext in html css js json png jpg svg woff2 pdf xml; do
    URL="https://example.com/test.$ext"
    echo "=== $URL ==="
    curl -sI "$URL" 2>/dev/null | grep -i content-type
done

# 3. Verify the body actually matches the declared type
curl -s https://example.com/sitemap.xml | head -1
# Expected: <?xml version="1.0" ...

# 4. Detect MIME from body bytes (compare to declared)
curl -s -o /tmp/asset https://example.com/path/to/asset
file --mime-type /tmp/asset
# Compare the file command's detection to the curl -sI declaration

# 5. Check nosniff is active
curl -sI https://example.com/ | grep -i x-content-type-options
# Expected: x-content-type-options: nosniff

# 6. Verify a font loads correctly (browsers reject incorrect font types)
curl -sI https://example.com/fonts/main.woff2 | grep -i content-type
# Expected: content-type: font/woff2 (NOT application/octet-stream)
```

### 5.8 Troubleshooting

**Symptom: CSS, JS, or font files refuse to load. Browser console says "MIME type mismatch" or "was blocked due to MIME type".**
Cause: `X-Content-Type-Options: nosniff` is active (correct) and the served `Content-Type` does not match what the link/script/font expects.
Fix:
1. Identify the failing asset's URL.
2. Run `curl -sI <url> | grep -i content-type`.
3. If the type is `application/octet-stream`, the extension is missing from nginx's mime type map. Add it via the `types {}` block in Section 5.6.
4. If the type is `text/plain` or wrong, search for an `add_header Content-Type` or `default_type` that is overriding the correct value.
5. Reload nginx: `nginx -t && systemctl reload nginx`.

**Symptom: Mojibake appears in the rendered page (â€" instead of em dash, etc).**
Cause: Charset is missing or wrong.
Fix:
1. Run `curl -sI https://example.com/page.html | grep -i content-type`.
2. If charset is missing, add the `charset utf-8;` and `charset_types` directives from Section 5.6.
3. Ensure the HTML body itself contains `<meta charset="utf-8">` in the first 1024 bytes.
4. Verify the source file is actually saved as UTF-8: `file -bi /var/www/sites/example.com/index.html` should report `text/html; charset=utf-8`.

**Symptom: Browser downloads a file instead of rendering it (or vice versa).**
Cause: Either the `Content-Type` is wrong (browser cannot render that type) or `Content-Disposition: attachment` is set when it should not be.
Fix:
1. Check the Content-Type with curl. If it is `application/octet-stream`, fix the mime type map.
2. Check for `Content-Disposition: attachment` and remove if not desired. See Section 9.

**Symptom: Lighthouse or PageSpeed warns about MIME type errors.**
Cause: One or more assets are served with `application/octet-stream` or `text/plain` instead of their correct type.
Fix: list failing assets, audit mime types, add missing types to the `types {}` block.

**Symptom: JSON API endpoint returns `Content-Type: text/html` and breaks the frontend.**
Cause: Upstream is not setting Content-Type, or nginx is rewriting it.
Fix: ensure the FastAPI sidecar (or whatever upstream) emits `Content-Type: application/json` explicitly:

```python
from fastapi.responses import JSONResponse
return JSONResponse(content=data, media_type="application/json")
```

### 5.9 How To Fix Common Breakage

**Case: `.avif` images served as `application/octet-stream` and browsers refuse them.**
Old mime.types file. Add the type:

```nginx
http {
    types {
        image/avif avif;
    }
}
```

`nginx -t && systemctl reload nginx`.

**Case: `.woff2` font served as `application/octet-stream` and the browser falls back to a system font.**
Same root cause. Add `font/woff2 woff2;` to the `types {}` block.

**Case: `.wasm` module fails to instantiate with `Module compilation` error.**
WebAssembly streaming compilation requires `Content-Type: application/wasm`. Add `application/wasm wasm;`.

**Case: `.webmanifest` for PWA returns 404 mime warning.**
Add `application/manifest+json webmanifest;`. iOS Safari is strict about this; Chrome is more forgiving but warnings appear in DevTools.

**Case: Sitemap returns `text/plain` instead of `application/xml`.**
The file extension is probably `.xml` but a location block is overriding the type:

```nginx
# Bad
location = /sitemap.xml {
    add_header Content-Type "text/plain" always;
}

# Good
location = /sitemap.xml {
    # Let nginx assign application/xml from mime.types
    expires 10m;
}
```

---

## 6. CONTENT-LANGUAGE (THE I18N BREADCRUMB BING AND BAIDU ACTUALLY USE)

### 6.1 What It Does

`Content-Language` declares the natural language(s) the response body is written in, using IANA language tags (BCP 47). Defined in RFC 9110 Section 8.5.

```
Content-Language: en
Content-Language: en-US
Content-Language: es-MX
Content-Language: zh-Hans-CN
Content-Language: en, fr
```

**Reality check on search engine behavior in 2026:**

* **Google**: largely ignores `Content-Language` for ranking and indexing decisions. Uses `hreflang` annotations (in HTML head, sitemap, or HTTP `Link` header) as the authoritative i18n signal. The HTML `<html lang="...">` attribute is also consulted.
* **Bing and Microsoft Copilot**: use `Content-Language` HTTP header AND `<meta http-equiv="content-language">` AND `<html lang>` AND any hreflang annotations. The HTTP header is the highest priority.
* **Baidu**: relies primarily on `Content-Language` and `<meta http-equiv>`. No hreflang support.
* **Yandex**: supports hreflang and reads `Content-Language` as a secondary signal.
* **AI engines** (Perplexity, ChatGPT search, Claude search, Bing-derived agents): inherit Bing's index for many regions, so `Content-Language` flows through.

For a Bubbles site that wants to rank well across all engines, set `Content-Language` AND `<html lang="...">` AND `hreflang` annotations. They are not redundant; they serve different audiences.

### 6.2 Syntax: BCP 47 Language Tags

The value is a comma separated list of BCP 47 tags. A tag has up to four parts joined by hyphens:

```
language [- script ] [- region ] [- variant ]
```

* **language** (required): two letter ISO 639-1 code (`en`, `es`, `fr`, `de`, `zh`, `ja`, `ar`).
* **script** (optional): four letter ISO 15924 code with first letter capitalized (`Hans`, `Hant`, `Latn`, `Cyrl`). Use when the script is ambiguous (Chinese simplified vs traditional, Serbian Cyrillic vs Latin).
* **region** (optional): two letter ISO 3166-1 country code, uppercase (`US`, `GB`, `MX`, `CN`, `BR`).
* **variant** (rare): registered variant codes.

Common Bubbles serving region tags:

| Region | Tag | Notes |
|---|---|---|
| US English (Northwest Arkansas, Southwest Missouri) | `en-US` | Default for almost every Bubbles client site |
| UK English | `en-GB` | |
| Generic English (no region preference) | `en` | Use when you do not want to commit to a region |
| Spanish (Mexico) | `es-MX` | Latin American Spanish, used in NWA Marshallese, Hispanic outreach |
| Spanish (Spain) | `es-ES` | European Spanish |
| Marshallese | `mh` | For Marshallese Voices and similar community sites |
| Simplified Chinese (mainland) | `zh-Hans-CN` or `zh-CN` | |
| Traditional Chinese (Taiwan) | `zh-Hant-TW` or `zh-TW` | |

**Common errors to avoid:**

* `en_US` (underscore): wrong. BCP 47 uses hyphens. Underscores are POSIX locale syntax, not language tags.
* `EN-US` (all caps language): wrong. Language must be lowercase per convention. Region must be uppercase.
* `english`: wrong. Use the two letter code.
* `en-USA`: wrong. Region is two letters, not three.

### 6.3 Single Language Per Page (Almost Always)

A page should have one primary language. Multilingual pages (a Spanish translation block embedded in an English page) should use the `lang` attribute on the surrounding element, not multiple values in `Content-Language`.

```html
<html lang="en-US">
  <body>
    <p>This is English.</p>
    <p lang="es">Esto es español.</p>
  </body>
</html>
```

The HTTP header for this page is `Content-Language: en-US`. The Spanish paragraph is signaled via the inline `lang` attribute. Crawlers and screen readers respect this distinction.

If a page truly is presented bilingually as a single document (rare), `Content-Language: en, es` is valid, but the SEO outcome will be unpredictable; most engines will pick the first tag.

### 6.4 How To Build It On Bubbles

Per server block:

```nginx
server {
    server_name example.com;

    # Apply to the entire site
    add_header Content-Language "en-US" always;
}
```

Per location (when subsections target different regions):

```nginx
server {
    server_name example.com;

    add_header Content-Language "en-US" always;

    location /es/ {
        add_header Content-Language "es-MX" always;
    }

    location /es/es/ {
        add_header Content-Language "es-ES" always;
    }
}
```

Remember the `add_header` inheritance trap: when you redeclare `add_header` inside a location, **all** parent `add_header` directives are dropped. Use a snippet pattern. From framework-http-caching-headers.md:

```nginx
# /etc/nginx/snippets/common-security-headers.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Then in each location:

```nginx
location /es/ {
    include snippets/common-security-headers.conf;
    add_header Content-Language "es-MX" always;
}
```

### 6.5 The HTML Layer (Required In Parallel)

Setting only the HTTP header is not enough. Every page must also have:

```html
<!DOCTYPE html>
<html lang="en-US">
```

And in `<head>`:

```html
<meta http-equiv="content-language" content="en-US">
```

The `meta http-equiv` is technically redundant if the HTTP header is correctly set, but Bing has historically prioritized it over the HTTP header in some indexing pipelines, so include it.

For sites with multiple language versions, add hreflang in `<head>`:

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/">
<link rel="alternate" hreflang="es-MX" href="https://example.com/es/">
<link rel="alternate" hreflang="x-default" href="https://example.com/">
```

`x-default` tells Google which version to show when no language match is determined (typically the global English version).

### 6.6 How To Verify

```bash
# 1. Confirm the header is present
curl -sI https://example.com/ | grep -i content-language
# Expected: content-language: en-US

# 2. Check a localized subpath
curl -sI https://example.com/es/ | grep -i content-language
# Expected: content-language: es-MX

# 3. Confirm the html lang attribute matches
curl -s https://example.com/ | grep -o '<html[^>]*lang="[^"]*"'
# Expected: <html lang="en-US"

# 4. Check that hreflang is present
curl -s https://example.com/ | grep -o 'hreflang="[^"]*"' | sort -u

# 5. Verify Bing can see it
curl -sI -A "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)" \
     https://example.com/ | grep -i content-language
```

### 6.7 Troubleshooting

**Symptom: Bing indexes English page as Spanish or vice versa.**
The `Content-Language` HTTP header, `<meta http-equiv>` tag, and `<html lang>` attribute disagree.
Fix: align all three. The values must be byte identical including casing of the region code.

**Symptom: Google ignores the language signal entirely.**
Expected. Google uses hreflang, not `Content-Language`. Implement hreflang annotations in `<head>` or as a sitemap addition.

**Symptom: Header is set to `en_US` and the screen reader pronounces words wrong.**
Underscore is wrong syntax. Change to `en-US`.

**Symptom: Different locations return the same Content-Language.**
The `add_header` inheritance trap fired. The location block dropped the parent declaration and did not redeclare. See Section 6.4.

**Symptom: Header is present but empty (`Content-Language: `).**
A variable expansion failed (e.g. `add_header Content-Language $lang;` where `$lang` resolved to empty). Fix the upstream variable or hardcode the value.

### 6.8 How To Fix Common Breakage

**Case: Site serves en-US content but Content-Language is missing entirely.**
Add to the server block:

```nginx
add_header Content-Language "en-US" always;
```

And in the HTML `<html>` element, set `lang="en-US"`. Add `<meta http-equiv="content-language" content="en-US">` to `<head>`. Reload nginx.

**Case: Multi region site uses Content-Language but hreflang is missing.**
Google needs hreflang regardless of Content-Language. Add hreflang annotations to `<head>` for every page, mapping the equivalent versions:

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/page">
<link rel="alternate" hreflang="es-MX" href="https://example.com/es/page">
<link rel="alternate" hreflang="x-default" href="https://example.com/page">
```

The hreflang annotations are reciprocal: every page in the set must annotate every other version including itself.

**Case: Region code is wrong (CAN instead of CA, USA instead of US).**
ISO 3166-1 alpha-2 only. Fix the value.

---

## 7. CONTENT-ENCODING (COMPRESSION ON THE WIRE)

### 7.1 What It Does

`Content-Encoding` declares the compression algorithm applied to the response body for transport. Defined in RFC 9110 Section 8.4.

```
Content-Encoding: gzip
Content-Encoding: br
Content-Encoding: zstd
Content-Encoding: deflate
```

The client must reverse the encoding (decompress) before doing anything with the body. The decompressed body is what `Content-Type` describes. The `Content-Length` header (when present) reports the **compressed** byte count, not the original.

`Content-Encoding` is distinct from `Transfer-Encoding`. The latter (`Transfer-Encoding: chunked`) is hop by hop framing and is handled invisibly by HTTP libraries. `Content-Encoding` is end to end and the client application sees its effect (must decompress).

### 7.2 Syntax And Algorithms

A single token per response, optionally a comma separated list if multiple encodings are stacked (rare and discouraged).

| Algorithm | Token | Compression ratio (text) | Speed | Browser support |
|---|---|---|---|---|
| Gzip | `gzip` | 70 to 85 percent | Fast | Universal, since the late 1990s |
| Brotli | `br` | 80 to 90 percent | Slower than gzip to compress, comparable to decompress | Chrome 50+, Firefox 44+, Safari 11+, Edge 15+. Universal in modern browsers since 2017 |
| Zstandard | `zstd` | 85 to 92 percent | Faster than gzip to compress AND decompress | Chrome 123+ (March 2024), Edge 123+, Firefox 126+ (May 2024). Safari does not support zstd as of 2026 |
| Deflate | `deflate` | Similar to gzip | Similar to gzip | Universal but quirky implementations; gzip is always a better choice |
| Identity | `identity` | None | n/a | The literal "no encoding". Almost never appears explicitly; absence of `Content-Encoding` implies identity |

**Practical guidance for Bubbles in 2026:**

* Always serve `gzip` (universal fallback, supported by every client including older crawlers).
* Serve `brotli` when available (best compression for the most users; modern browser default preference).
* Serve `zstd` when the module is installed (slight win over brotli for clients that prefer it).
* Never use `deflate`. It has implementation inconsistencies that occasionally produce decode failures. Gzip serves the same use case more reliably.

### 7.3 The Negotiation Chain

The client sends `Accept-Encoding: gzip, br, zstd` in the request. The server picks one of the offered algorithms (preferring whichever the server prefers, weighted by client `q` values if present) and applies it. The server then sends:

```
Content-Encoding: br
Vary: Accept-Encoding
```

The `Vary: Accept-Encoding` is critical. It tells every cache between the server and client that responses for the same URL differ based on `Accept-Encoding`. Without `Vary: Accept-Encoding`, a cache could serve a brotli response to a client that did not request brotli, producing binary garbage in the browser.

This is the same Vary requirement covered in framework-http-caching-headers.md Section 9. The two frameworks are coupled here.

### 7.4 How To Build It On Bubbles

**Gzip (built in to nginx, always available):**

```nginx
http {
    gzip on;
    gzip_vary on;
    gzip_min_length 1100;
    gzip_comp_level 6;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/xml
        text/csv
        application/javascript
        application/json
        application/xml
        application/atom+xml
        application/rss+xml
        application/ld+json
        application/manifest+json
        application/wasm
        image/svg+xml
        font/ttf
        font/otf;
    gzip_disable "msie6";
}
```

* `gzip on`: enable.
* `gzip_vary on`: emit `Vary: Accept-Encoding` automatically when compressed. Always set.
* `gzip_min_length 1100`: do not compress responses smaller than 1100 bytes (overhead exceeds savings for tiny responses).
* `gzip_comp_level 6`: balance between CPU and ratio. Range 1 to 9. Level 6 is the sweet spot for HTTP.
* `gzip_proxied any`: compress responses going through proxy headers, not just direct requests.
* `gzip_types`: which Content-Type values are eligible. **`text/html` is always compressed by nginx regardless and does not need to be listed.** List everything else that is compressible.
* `gzip_disable "msie6"`: skip compression for ancient IE6. Harmless.

**Brotli (requires `ngx_brotli` module):**

```nginx
# Verify the module is loaded (one of these)
load_module modules/ngx_http_brotli_filter_module.so;
load_module modules/ngx_http_brotli_static_module.so;

http {
    brotli on;
    brotli_comp_level 6;
    brotli_min_length 1100;
    brotli_types
        text/plain
        text/css
        text/xml
        text/csv
        application/javascript
        application/json
        application/xml
        application/atom+xml
        application/rss+xml
        application/ld+json
        application/manifest+json
        application/wasm
        image/svg+xml
        font/ttf
        font/otf;
}
```

* Brotli emits `Vary: Accept-Encoding` automatically when used (no `brotli_vary` directive exists; gzip_vary handles it for both).
* Brotli compression level 6 is a good HTTP default. Level 11 is best ratio but very slow (only acceptable for static pre compressed files).

**Zstd (requires `zstd-nginx-module`):**

```nginx
load_module modules/ngx_http_zstd_filter_module.so;
load_module modules/ngx_http_zstd_static_module.so;

http {
    zstd on;
    zstd_comp_level 6;
    zstd_min_length 1100;
    zstd_types
        text/plain
        text/css
        text/xml
        application/javascript
        application/json
        application/xml
        application/ld+json
        image/svg+xml;
}
```

When all three are enabled, nginx will offer all three to clients. The client's `Accept-Encoding` list and preference order determine which is sent. Chrome, Edge, and Firefox typically prefer zstd then brotli then gzip. Safari prefers brotli then gzip.

**Precompressed static files:**

For static assets, the highest performance option is to precompress at build time and let nginx serve the precompressed file directly without CPU work:

```nginx
http {
    gzip_static on;     # serve file.css.gz if it exists and client accepts gzip
    brotli_static on;   # serve file.css.br if it exists and client accepts brotli
    zstd_static on;     # serve file.css.zst if it exists and client accepts zstd
}
```

Then in your build pipeline:

```bash
# In your deploy script, after building static assets
cd /var/www/sites/example.com
for f in $(find . -type f \( -name "*.css" -o -name "*.js" -o -name "*.html" -o -name "*.svg" -o -name "*.json" -o -name "*.xml" \)); do
    gzip -9 -k -f "$f"        # produces $f.gz
    brotli -9 -k -f "$f"      # produces $f.br (requires brotli CLI tool)
    zstd -19 -k -f "$f"       # produces $f.zst (requires zstd CLI tool)
done
```

Result: zero CPU cost per request for the most aggressive compression level (gzip 9, brotli 11, zstd 22 are all viable for precompressed files since they are computed once).

### 7.5 How To Verify

```bash
# 1. Check that compression actually applies
curl -sI -H "Accept-Encoding: gzip" https://example.com/page.html | grep -i content-encoding
# Expected: content-encoding: gzip

curl -sI -H "Accept-Encoding: br" https://example.com/page.html | grep -i content-encoding
# Expected: content-encoding: br

curl -sI -H "Accept-Encoding: zstd" https://example.com/page.html | grep -i content-encoding
# Expected: content-encoding: zstd (if module installed)

# 2. Confirm Vary is set
curl -sI -H "Accept-Encoding: gzip" https://example.com/page.html | grep -i vary
# Expected: vary: Accept-Encoding

# 3. Compare sizes
RAW=$(curl -s -H "Accept-Encoding: identity" https://example.com/page.html | wc -c)
GZIP=$(curl -s -H "Accept-Encoding: gzip" --output - https://example.com/page.html | wc -c)
BROTLI=$(curl -s -H "Accept-Encoding: br" --output - https://example.com/page.html | wc -c)
echo "Raw: $RAW bytes"
echo "Gzip: $GZIP bytes ($(echo "scale=1; ($GZIP * 100) / $RAW" | bc) percent)"
echo "Brotli: $BROTLI bytes ($(echo "scale=1; ($BROTLI * 100) / $RAW" | bc) percent)"

# 4. Decompress and verify content matches
curl -s --compressed https://example.com/page.html > /tmp/decompressed.html
curl -s -H "Accept-Encoding: identity" https://example.com/page.html > /tmp/raw.html
diff /tmp/decompressed.html /tmp/raw.html
# Expected: no output (files match)

# 5. Check that small responses are not compressed (gzip_min_length working)
curl -sI -H "Accept-Encoding: gzip" https://example.com/tiny.html | grep -i content-encoding
# If the file is under 1100 bytes, no content-encoding header should appear

# 6. Verify Googlebot gets compressed responses
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     -H "Accept-Encoding: gzip, deflate, br" \
     https://example.com/ | grep -i content-encoding
```

### 7.6 Troubleshooting

**Symptom: Browser receives binary garbage instead of HTML.**
A compressed response was sent to a client that did not request compression, or a cache served a compressed copy to such a client.
Causes:
1. `Vary: Accept-Encoding` is missing. The cache cannot distinguish compressed and uncompressed variants.
2. Compression is applied unconditionally (no `Accept-Encoding` check).
3. A reverse proxy or intermediate stripped the `Content-Encoding` header but not the compressed body.
Fix: enable `gzip_vary on`. Verify with curl as shown above.

**Symptom: Content-Encoding header missing on responses that should be compressed.**
Causes:
1. The response Content-Type is not in `gzip_types` (or `brotli_types`/`zstd_types`).
2. The response is smaller than `gzip_min_length`.
3. `gzip off` is set in the location or server block.
4. The upstream is returning a `Content-Encoding` already (in which case nginx will not double compress) but the encoding value is unexpected.
5. nginx version too old (gzip was always available but module loading varies).
Fix: confirm the Content-Type matches a listed type, check sizes, search config for `gzip off`.

**Symptom: Compression is on but ratios are terrible (only 10 to 20 percent savings).**
Causes:
1. Content is already compressed (images, video, fonts in .woff2 which is brotli compressed internally). Compressing again costs CPU and saves nothing.
2. Comp level is too low (level 1 produces poor ratios).
Fix: do not compress already compressed types (do not list `image/jpeg`, `image/png`, `image/webp`, `video/mp4`, `font/woff2` in your `*_types` directives). Use level 6 for dynamic, level 9 or 11 for precompressed.

**Symptom: ETag changes when compression is applied.**
Expected. See framework-http-caching-headers.md Section 6.4. Nginx weakens strong ETags when applying gzip on the fly. This is correct behavior and does not break caching.

**Symptom: Safari users see uncompressed responses while Chrome users see brotli.**
Expected. Safari does not support zstd as of 2026 and prefers brotli. Chrome may prefer zstd. The negotiation chain is working correctly. Each user gets their best supported encoding.

**Symptom: `Accept-Encoding: gzip, deflate, br` but server returns `Content-Encoding: identity`.**
Server is choosing to not compress for some reason. Most likely the Content-Type is not in the compress types list, or the response is small.

### 7.7 How To Fix Common Breakage

**Case: A new content type is being served uncompressed.**
Example: a `application/ld+json` schema endpoint is shipping 50 KB uncompressed.
Fix: add `application/ld+json` to `gzip_types` and `brotli_types`. Reload nginx.

**Case: Brotli module installed but never used.**
The `load_module` directive is missing from the top of `nginx.conf`. Add:

```nginx
load_module modules/ngx_http_brotli_filter_module.so;
load_module modules/ngx_http_brotli_static_module.so;
```

These must be at the very top of `nginx.conf`, before the `events` block.

**Case: Precompressed files are not being served.**
Verify the `.gz`, `.br`, or `.zst` files exist next to the originals:

```bash
ls -la /var/www/sites/example.com/main.css*
# Expected:
# main.css
# main.css.gz
# main.css.br
# main.css.zst
```

Verify `gzip_static on`, `brotli_static on`, `zstd_static on` are set. Verify file permissions allow nginx to read them.

**Case: All responses are sending Content-Encoding: gzip even for binary files.**
The `gzip_types` list includes binary types or the default catches them. Check the list, remove image and video types.

**Case: PageSpeed Insights reports "Enable text compression" warning.**
Some asset is not being compressed. Run:

```bash
for asset in /css/main.css /js/app.js /api/data.json; do
    echo "=== $asset ==="
    curl -sI -H "Accept-Encoding: gzip" "https://example.com$asset" | grep -iE "content-(type|encoding|length)"
done
```

Find the missing types, add to `gzip_types`, reload.

---

## 8. CONTENT-LENGTH (THE FRAMING CONTRACT)

### 8.1 What It Does

`Content-Length` reports the number of bytes in the response body, as transmitted on the wire (after Content-Encoding, before TLS encoding). Defined in RFC 9110 Section 8.6.

```
Content-Length: 12847
```

It serves several functions:

1. **Framing**: tells the client how much body to read. Without it, the client must rely on `Transfer-Encoding: chunked` or connection close to know when the response ends.
2. **Progress display**: lets browsers and downloaders show download progress bars.
3. **Range request validation**: required for `HTTP 206 Partial Content` responses.
4. **Keep-alive efficiency**: allows the connection to be reused without waiting for close.

`Content-Length` is the **compressed** byte count when `Content-Encoding` is applied. Not the original byte count.

### 8.2 When Content-Length Is And Is Not Present

| Scenario | Content-Length present? |
|---|---|
| Static file served by nginx, no compression | Yes, always |
| Static file served with `gzip_static on` (precompressed) | Yes, the .gz file's byte count |
| Static file with on the fly gzip | **No.** Nginx switches to `Transfer-Encoding: chunked` because the compressed size is not known up front |
| Dynamic response from upstream (FastAPI sidecar) | Depends on upstream. FastAPI usually sets it for non streaming responses |
| Server Sent Events (`text/event-stream`) | Never. Uses chunked transfer indefinitely |
| HTTP/2 and HTTP/3 responses | Optional. The protocol has its own framing; Content-Length is informational |
| Responses with `Transfer-Encoding: chunked` | Must not be present (RFC violation if both appear) |

The absence of `Content-Length` is not an error. Modern clients handle both cases. The absence does mean progress bars cannot be shown until the download completes.

### 8.3 The Critical Rule: Never Both

A response must contain `Content-Length` OR `Transfer-Encoding`, but never both. The combination is the basis for HTTP request smuggling attacks. From the spec:

> A sender MUST NOT send a message that contains both a `Content-Length` header field and a non identity `Transfer-Encoding` header field, as this might confuse downstream recipients.

Nginx enforces this correctly. The only way to violate it is with a buggy upstream that emits both. Defense:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    # If upstream emits both, strip Content-Length and let chunked dominate
    # This is a defensive measure; the right fix is at the upstream
    # proxy_hide_header Content-Length;
}
```

The commented line is a last resort. The correct fix is to find the upstream code that emits both headers and fix it.

### 8.4 How To Build It On Bubbles

`Content-Length` is set automatically by nginx for static files. No configuration needed.

For upstream responses, the upstream sets it. Nginx passes it through unmodified. To force chunked transfer (no Content-Length):

```nginx
location /stream/ {
    chunked_transfer_encoding on;
    proxy_pass http://127.0.0.1:9090;
    proxy_buffering off;     # disable buffering so chunks flush immediately
}
```

To force the opposite (buffer the upstream and emit Content-Length):

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_buffering on;
    proxy_buffer_size 8k;
    proxy_buffers 8 8k;
}
```

With `proxy_buffering on`, nginx buffers the full upstream response, measures it, and emits `Content-Length`. Good for small responses. Bad for large streaming responses (delays first byte).

### 8.5 How To Verify

```bash
# 1. Check Content-Length is present for static
curl -sI https://example.com/style.css | grep -i content-length
# Expected: content-length: 12847

# 2. Verify the value is accurate
DECLARED=$(curl -sI -H "Accept-Encoding: identity" https://example.com/style.css | grep -i content-length | awk '{print $2}' | tr -d '\r')
ACTUAL=$(curl -s -H "Accept-Encoding: identity" https://example.com/style.css | wc -c)
echo "Declared: $DECLARED, Actual: $ACTUAL"
# Should be equal

# 3. Check what happens with on the fly gzip
curl -sI -H "Accept-Encoding: gzip" https://example.com/style.css | grep -iE "content-length|transfer-encoding"
# Expected: transfer-encoding: chunked (no content-length)

# 4. Check precompressed file behavior
ls -la /var/www/sites/example.com/style.css*
# If style.css.gz exists:
curl -sI -H "Accept-Encoding: gzip" https://example.com/style.css | grep -i content-length
# Expected: content-length: <gz file size>

# 5. Ensure no conflict
curl -sI https://example.com/page.html | grep -iE "content-length|transfer-encoding"
# Exactly one of the two should appear, never both
```

### 8.6 Troubleshooting

**Symptom: Browser shows no progress bar on download.**
Cause: `Content-Length` is missing because the response is chunked. This is usually because of on the fly compression.
Fix: precompress the file (`gzip_static on` plus build pipeline produces .gz files). Once nginx serves the precompressed file directly, it knows the size and emits Content-Length.

**Symptom: Download stops at the wrong byte count.**
Cause: Declared `Content-Length` does not match actual body size. Almost always an upstream bug.
Fix: identify the upstream, find the code that computes Content-Length, ensure it matches the byte count of what is actually written.

**Symptom: Browser warns about "incomplete response" or "transfer interrupted".**
Cause: Client received fewer bytes than `Content-Length` promised. Either the connection dropped or the server's computation was wrong.
Fix: check upstream logs for errors. Check network for intermittent connectivity. Verify `Content-Length` computation.

**Symptom: HTTP smuggling vulnerability scanner reports issue.**
Cause: Both `Content-Length` and `Transfer-Encoding: chunked` are appearing on the same response, or there is disagreement between the front nginx and an upstream.
Fix: find the upstream emitting both. Strip one. Confirm with curl that only one appears.

**Symptom: Range requests (HTTP 206) fail for compressed content.**
Cause: Range requests require `Content-Length` and a strong ETag. On the fly compression breaks both.
Fix: precompress the resource, or disable compression for that location, or accept that ranges work only for the uncompressed variant.

### 8.7 How To Fix Common Breakage

**Case: PDF download is missing Content-Length and progress bar.**
The PDF is being on the fly compressed. PDFs are already compressed internally; further compression is wasted CPU. Exclude PDF from compression:

```nginx
# Do NOT add application/pdf to gzip_types
# If a global gzip rule somehow catches it, exclude per location:
location ~* \.pdf$ {
    gzip off;
    expires 1d;
}
```

After this, the static PDF serves directly with accurate Content-Length.

**Case: Streaming response from FastAPI sidecar gets buffered and delayed.**
nginx is computing Content-Length by buffering the upstream. Disable buffering for the location:

```nginx
location /stream/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_buffering off;
    proxy_request_buffering off;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

This produces chunked transfer with no Content-Length, suitable for SSE or streaming JSON.

**Case: Upstream emits both Content-Length and Transfer-Encoding.**
Find the upstream code. In Python with FastAPI, this can happen if you set `Content-Length` manually on a `StreamingResponse`. Remove the manual header; the framework will handle framing correctly.

---

## 9. CONTENT-DISPOSITION (INLINE VS ATTACHMENT, FILENAMES, SECURITY)

### 9.1 What It Does

`Content-Disposition` tells the browser whether to render the response inline (in the page or a viewer) or to prompt the user to save it as a file. When prompting to save, it also suggests a filename. Defined in RFC 6266.

```
Content-Disposition: inline
Content-Disposition: attachment
Content-Disposition: attachment; filename="report.pdf"
Content-Disposition: attachment; filename="report.pdf"; filename*=UTF-8''report-%E2%82%AC.pdf
```

The two main values:

* **`inline`** (default for most types): render the content in the page context. Images appear in `<img>` tags. PDFs render in the browser's PDF viewer. HTML renders as a page.
* **`attachment`**: prompt the user with a Save As dialog. The browser does not render the content; it downloads it.

The `filename` parameter suggests a filename for the Save As dialog. The `filename*` parameter (RFC 5987) allows non ASCII characters via percent encoding plus charset prefix.

### 9.2 Syntax

```
Content-Disposition: <disposition-type>
Content-Disposition: <disposition-type>; <parameter>=<value>
```

Disposition types:

* `inline`: render in page (default if omitted).
* `attachment`: download.

Parameters:

* `filename="..."`: ASCII safe filename suggestion. Wrap in double quotes if it contains spaces or special characters.
* `filename*=charset''percent-encoded`: RFC 5987 encoded filename for non ASCII. Example: `filename*=UTF-8''r%C3%A9sum%C3%A9.pdf` for `résumé.pdf`.

When both `filename` and `filename*` are present, the client uses `filename*` if it understands it (every modern browser does), falling back to `filename` for compatibility. **Best practice: always provide both.**

### 9.3 How To Build It On Bubbles

**Force download for an entire location:**

```nginx
location /downloads/ {
    add_header Content-Disposition "attachment" always;
    # The browser will use the URL's last path segment as the default filename
}
```

**Force download with a specific filename:**

```nginx
location = /downloads/quarterly-report.pdf {
    add_header Content-Disposition 'attachment; filename="quarterly-report.pdf"' always;
}
```

**Force download with internationalized filename:**

```nginx
location = /downloads/resume.pdf {
    add_header Content-Disposition 'attachment; filename="resume.pdf"; filename*=UTF-8''r%C3%A9sum%C3%A9.pdf' always;
}
```

**Force inline rendering (override browser default):**

```nginx
location ~* \.pdf$ {
    add_header Content-Disposition "inline" always;
}
```

Useful when serving PDFs that should appear in the browser PDF viewer rather than downloading.

**Dynamic filename from upstream (FastAPI sidecar):**

```python
from fastapi.responses import FileResponse
from urllib.parse import quote

filename = "Q3 Report (Final).pdf"
ascii_safe = "Q3-Report-Final.pdf"
url_encoded = quote(filename.encode('utf-8'))

return FileResponse(
    path="/var/data/q3-report.pdf",
    media_type="application/pdf",
    headers={
        "Content-Disposition": f'attachment; filename="{ascii_safe}"; filename*=UTF-8\'\'{url_encoded}'
    }
)
```

### 9.4 Security: Filename Sanitization Is Mandatory

`Content-Disposition` is a documented attack vector. When the filename is constructed from user input (uploaded filename, user provided field, URL parameter), unsanitized values enable:

* **Reflected file download (RFD)**: attacker tricks browser into saving a malicious script as `.bat`, `.cmd`, `.sh`, or similar executable. User double clicks the download. Code executes.
* **Header injection**: filename containing `\r\n` characters injects new headers, potentially overwriting Content-Type or adding `Set-Cookie`.
* **Directory traversal**: filename containing `../` or `..\` confuses the client into writing to unexpected paths (defense is at the client, but server should never emit these).
* **Filename truncation**: very long filenames cause some clients to truncate at unexpected points, potentially exposing internal data.

**Mandatory sanitization rules** for any user influenced filename:

1. Strip control characters: `\r`, `\n`, `\t`, anything below ASCII 32.
2. Strip or replace path separators: `/`, `\`, `:`.
3. Strip quotes: `"`, `'`, backtick.
4. Limit length to 255 characters (filesystem limit on most systems).
5. Enforce an allow list of file extensions (`.pdf`, `.zip`, `.csv`, etc), reject anything else.
6. Never trust the client supplied filename for any server side action; only use it as the suggested download name.

Python sanitization example:

```python
import re

def sanitize_filename(filename: str) -> str:
    # Strip path components
    filename = filename.replace('/', '').replace('\\', '')
    # Strip control characters
    filename = re.sub(r'[\x00-\x1f\x7f]', '', filename)
    # Strip quotes
    filename = filename.replace('"', '').replace("'", '').replace('`', '')
    # Limit length
    filename = filename[:200]
    # Disallow empty result
    if not filename or filename in ('.', '..'):
        filename = 'download'
    return filename
```

### 9.5 The Inline HTML Trap

If you serve user uploaded HTML files with `Content-Disposition: inline` (or no Content-Disposition at all), and the `Content-Type` is `text/html`, the file renders as a page in your origin's security context. This is a stored XSS.

Defenses ranked by quality:

1. **Never serve user uploaded content from the main origin.** Use a separate sandbox domain (`user-uploads.example.com`) so uploaded HTML cannot access your site's cookies or local storage. This is the gold standard and is how Google Drive, Dropbox, and similar services handle uploads.
2. **Always force `Content-Disposition: attachment` for user uploads.** The browser downloads instead of rendering. No script execution.
3. **Force a safe Content-Type.** Serve all user uploads as `application/octet-stream` (forces download regardless of file extension).
4. **Always set `X-Content-Type-Options: nosniff`.** Already a Bubbles default. Prevents the browser from sniffing and deciding the file is HTML when it should be text.

The Bubbles convention for any user upload endpoint:

```nginx
location /user-uploads/ {
    add_header Content-Type "application/octet-stream" always;
    add_header Content-Disposition "attachment" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Content-Security-Policy "default-src 'none'" always;
}
```

This combination makes user uploads inert regardless of their content.

### 9.6 How To Verify

```bash
# 1. Confirm Content-Disposition is set for a download endpoint
curl -sI https://example.com/downloads/report.pdf | grep -i content-disposition
# Expected: content-disposition: attachment; filename="report.pdf"

# 2. Verify the response prompts download (look for the header in browser too)
curl -sI https://example.com/downloads/report.pdf
# Manually check headers

# 3. Test filename with special characters
curl -sI https://example.com/downloads/resume.pdf | grep -i content-disposition
# Expected: contains both filename= and filename*= parameters

# 4. Verify no Content-Disposition on pages that should render inline
curl -sI https://example.com/about.html | grep -i content-disposition
# Expected: no content-disposition header

# 5. Check that user uploads are forced to download
curl -sI https://example.com/user-uploads/photo.jpg | grep -iE "content-disposition|content-type"
# Expected: content-type: application/octet-stream
# Expected: content-disposition: attachment
```

### 9.7 Troubleshooting

**Symptom: Browser tries to render PDF instead of downloading.**
Cause: No `Content-Disposition: attachment` header.
Fix: add to the location for PDF downloads:

```nginx
location ~* \.pdf$ {
    add_header Content-Disposition "attachment" always;
}
```

**Symptom: Download has the wrong filename.**
Cause: filename parameter is missing or wrong.
Fix: set the filename explicitly:

```nginx
location = /downloads/report.pdf {
    add_header Content-Disposition 'attachment; filename="quarterly-report-q3-2026.pdf"' always;
}
```

**Symptom: Non ASCII characters in filename appear as `?` or get stripped.**
Cause: Using only the `filename=` parameter without RFC 5987 encoding.
Fix: add `filename*=UTF-8''percent-encoded-name`:

```
Content-Disposition: attachment; filename="resume.pdf"; filename*=UTF-8''r%C3%A9sum%C3%A9.pdf
```

**Symptom: Security scanner flags Content-Disposition as a vulnerability.**
Cause: filename is constructed from unsanitized user input.
Fix: implement the sanitization function in Section 9.4. Never pass raw user input into the header value.

**Symptom: User uploaded HTML file renders and runs scripts in your origin.**
Critical XSS. Fix immediately per Section 9.5: force `attachment` disposition and `application/octet-stream` content type on all user uploads. Long term: move uploads to a sandbox subdomain.

### 9.8 How To Fix Common Breakage

**Case: All PDFs render inline but client wants downloads.**
Add the attachment header for PDFs:

```nginx
location ~* \.pdf$ {
    add_header Content-Disposition "attachment" always;
}
```

If you want some PDFs to render inline (preview) and others to download, use different paths:

```nginx
location /preview/ {
    add_header Content-Disposition "inline" always;
}
location /download/ {
    add_header Content-Disposition "attachment" always;
}
```

**Case: Download dialog shows `Untitled` or random garbage filename.**
The filename is missing from Content-Disposition. Set it explicitly.

**Case: Filename with spaces gets cut off at the first space.**
Filename is not wrapped in quotes. Wrap it:

```
attachment; filename="My Document.pdf"
```

Not:

```
attachment; filename=My Document.pdf
```

**Case: Need to serve a generated filename like `export-2026-05-24.csv`.**
Generate it in the upstream and emit the Content-Disposition header from there. Nginx can't easily build dynamic filenames at the static layer.

```python
# FastAPI example
from datetime import date
filename = f"export-{date.today().isoformat()}.csv"
return Response(
    content=csv_bytes,
    media_type="text/csv",
    headers={"Content-Disposition": f'attachment; filename="{filename}"'}
)
```

---

## 10. HOW THESE HEADERS INTERACT

The five headers do not act independently. Several combinations matter.

### 10.1 Content-Type and Content-Encoding

`Content-Type` describes the decompressed body. `Content-Encoding` describes the transformation applied. The client decompresses first, then parses according to Content-Type.

```
Content-Type: application/json; charset=utf-8
Content-Encoding: br
Content-Length: 1247
```

Means: the wire bytes are brotli compressed. Decompress them. The result is UTF-8 encoded JSON. The wire length is 1247 bytes.

A trap: serving a file that is already compressed (`.gz` file) with `Content-Encoding: gzip` set causes double compression. Browser tries to gunzip the already gzipped bytes, fails. The fix:

```nginx
# Serving .gz files as their own resource (rare)
location ~ \.gz$ {
    # The file is application/gzip (a downloadable archive)
    # Do NOT set Content-Encoding
    types {
        application/gzip gz;
    }
    add_header Content-Disposition "attachment" always;
}

# Serving precompressed assets (most cases)
location ~ \.(css|js|html)$ {
    gzip_static on;   # nginx adds Content-Encoding: gzip when it serves the .gz
}
```

`gzip_static on` is the correct path. nginx sees the `.gz` next to `main.css`, serves it, AND sets `Content-Encoding: gzip` AND keeps `Content-Type: text/css`. All three are coherent.

### 10.2 Content-Type and Content-Disposition

The browser's default disposition depends on Content-Type:

| Content-Type | Default browser behavior |
|---|---|
| `text/html` | Render inline as page |
| `text/css`, `application/javascript` | Not directly accessed by user; loaded by parent page |
| `image/*` | Render inline if loaded by `<img>`, prompt to save if accessed directly |
| `application/pdf` | Render in PDF viewer (Chrome, Edge, Firefox) or prompt to save (older clients) |
| `application/zip`, `application/x-tar` | Always prompt to save |
| `application/octet-stream` | Always prompt to save |
| Unknown types | Prompt to save |

`Content-Disposition: attachment` overrides every default. `Content-Disposition: inline` overrides defaults that would otherwise prompt to save.

### 10.3 Content-Length and Content-Encoding

`Content-Length` is the compressed byte count when `Content-Encoding` is applied. The decompressed byte count is not reported anywhere in standard HTTP headers.

If you need to know the uncompressed size at the client, the application protocol must convey it (e.g. a custom header like `X-Decoded-Length` or include it in JSON metadata).

### 10.4 Content-Language and the HTML lang Attribute

`Content-Language` HTTP header, `<meta http-equiv="content-language">`, and `<html lang="...">` should all agree. They serve different consumers:

* HTTP header: Bing, Baidu, intermediate caches that vary on language.
* meta http-equiv: archived copies that lost HTTP context, some legacy crawlers.
* html lang: every modern crawler, screen readers, browser spell checkers, automatic translation tools.

For Google specifically, `<html lang>` plus hreflang annotations are the canonical signal. The HTTP header is mostly invisible to Google.

### 10.5 Content-Length and Transfer-Encoding

Mutually exclusive. Never both on the same response. nginx enforces this. Custom upstreams must follow the rule.

### 10.6 Content-Type Charset and Body Encoding

The charset parameter in `Content-Type` describes how the bytes encode characters. The bytes themselves must actually be in that encoding. Setting `charset=utf-8` on a Windows-1252 byte stream produces mojibake just as surely as omitting the charset.

To verify your source files are UTF-8:

```bash
file -bi /var/www/sites/example.com/index.html
# Expected: text/html; charset=utf-8

# If it reports text/html; charset=iso-8859-1 or similar:
iconv -f iso-8859-1 -t utf-8 /var/www/sites/example.com/index.html > /tmp/converted.html
mv /tmp/converted.html /var/www/sites/example.com/index.html
```

---

## 11. ASSET CLASS RECIPES

Each block is a paste ready location stanza for the five content headers. Copy the ones relevant to your build.

### 11.1 HTML (UTF-8 always, no Content-Disposition)

```nginx
location ~* \.html$ {
    # Content-Type: text/html; charset=utf-8 (auto from mime.types + charset utf-8)
    # Content-Language set at server level: en-US
    # Content-Encoding: gzip or br (auto from gzip/brotli on)
    # Content-Length: auto for static, chunked when on the fly compression active
    # Content-Disposition: none (default inline)
    expires 0;
    add_header Cache-Control "public, max-age=0, must-revalidate" always;
}
```

### 11.2 CSS

```nginx
location ~* \.css$ {
    # Content-Type: text/css; charset=utf-8 (auto)
    # Content-Encoding: gzip/br/zstd (auto, large savings)
    # Content-Length: auto for precompressed, chunked otherwise
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}
```

### 11.3 JavaScript

```nginx
location ~* \.(js|mjs)$ {
    # Content-Type: application/javascript; charset=utf-8 (auto)
    # Content-Encoding: gzip/br/zstd (auto)
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}
```

### 11.4 JSON

```nginx
location ~* \.json$ {
    # Content-Type: application/json; charset=utf-8 (auto)
    # Content-Encoding: gzip/br/zstd (auto)
    expires 1h;
    add_header Cache-Control "public, max-age=3600, must-revalidate" always;
}

# For schema and structured data
location ~* \.jsonld$ {
    types { application/ld+json jsonld; }
    expires 1h;
    add_header Cache-Control "public, max-age=3600" always;
}
```

### 11.5 XML (sitemaps, feeds)

```nginx
location ~* \.xml$ {
    # Content-Type: application/xml; charset=utf-8 (auto)
    # Content-Encoding: gzip/br/zstd (auto)
    expires 10m;
    add_header Cache-Control "public, max-age=600" always;
}
```

### 11.6 SVG

```nginx
location ~* \.svg$ {
    # Content-Type: image/svg+xml; charset=utf-8 (auto)
    # Content-Encoding: gzip/br/zstd (auto; SVG is text, compresses ~70 percent)
    expires 30d;
    add_header Cache-Control "public, max-age=2592000" always;
}
```

### 11.7 Raster images (PNG, JPEG, WebP, AVIF)

```nginx
location ~* \.(png|jpe?g|gif|webp|avif|ico)$ {
    # Content-Type: image/png, image/jpeg, etc (auto)
    # Content-Encoding: none (already compressed internally)
    # Make sure these are NOT in gzip_types or brotli_types
    expires 30d;
    add_header Cache-Control "public, max-age=2592000" always;
}
```

### 11.8 Fonts (woff2, woff, ttf, otf)

```nginx
location ~* \.(woff2|woff|ttf|otf|eot)$ {
    # Content-Type: font/woff2, font/ttf, etc (auto)
    # Content-Encoding: NONE for woff2 (already brotli compressed)
    #                   gzip/br OK for ttf, otf (uncompressed formats)
    # CORS: required if loaded from a different origin
    add_header Access-Control-Allow-Origin "*" always;
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}
```

### 11.9 PDF (with explicit attachment)

```nginx
location ~* \.pdf$ {
    # Content-Type: application/pdf (auto)
    # Content-Encoding: NONE (PDF already compressed internally)
    expires 1d;
    add_header Cache-Control "public, max-age=86400" always;
    add_header Content-Disposition "attachment" always;
    add_header X-Content-Type-Options "nosniff" always;
}

# Or for inline preview
location /preview/ {
    add_header Content-Disposition "inline" always;
}
```

### 11.10 Downloadable archives (zip, tar, gz)

```nginx
location ~* \.(zip|tar|gz|bz2|7z|rar)$ {
    # Content-Type: application/zip, application/x-tar, etc (auto)
    # Content-Encoding: NONE (these ARE the archive; do not double compress)
    expires 1d;
    add_header Cache-Control "public, max-age=86400" always;
    add_header Content-Disposition "attachment" always;
}
```

### 11.11 User uploaded content (security hardened)

```nginx
location /user-uploads/ {
    # Force download, force binary type, prevent XSS via uploaded HTML
    add_header Content-Type "application/octet-stream" always;
    add_header Content-Disposition "attachment" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Content-Security-Policy "default-src 'none'" always;
    expires 1d;
}
```

### 11.12 robots.txt, ai.txt, llms.txt, security.txt

```nginx
location ~* /(robots\.txt|ai\.txt|llms\.txt|humans\.txt|\.well-known/security\.txt)$ {
    # Content-Type: text/plain; charset=utf-8 (auto from .txt)
    # Content-Encoding: gzip (auto; saves bandwidth on long robots files)
    expires 1h;
    add_header Cache-Control "public, max-age=3600" always;
}
```

### 11.13 WebAssembly modules

```nginx
location ~* \.wasm$ {
    # Content-Type: application/wasm (must add to types {} if missing from mime.types)
    types { application/wasm wasm; }
    # Content-Encoding: gzip/br/zstd (auto; .wasm benefits from compression)
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
    add_header Cross-Origin-Embedder-Policy "require-corp" always;
    add_header Cross-Origin-Opener-Policy "same-origin" always;
}
```

### 11.14 PWA manifest

```nginx
location ~* \.webmanifest$ {
    types { application/manifest+json webmanifest; }
    # Charset: utf-8 (auto via charset_types)
    expires 1d;
    add_header Cache-Control "public, max-age=86400" always;
}
```

### 11.15 Generated CSV exports (from FastAPI sidecar)

```nginx
location /export/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_pass_header Content-Type;
    proxy_pass_header Content-Disposition;
    # FastAPI must emit:
    #   Content-Type: text/csv; charset=utf-8
    #   Content-Disposition: attachment; filename="export-2026-05-24.csv"
}
```

---

## 12. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete content header stanza, layered with the caching stanza from framework-http-caching-headers.md.

```nginx
# /etc/nginx/nginx.conf (or http context)
http {
    # ===== CONTENT TYPE =====
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    types {
        image/avif                       avif;
        font/woff2                       woff2;
        application/wasm                 wasm;
        application/manifest+json        webmanifest manifest;
        application/ld+json              jsonld;
        model/gltf-binary                glb;
        model/gltf+json                  gltf;
        text/markdown                    md;
    }

    # ===== CHARSET =====
    charset utf-8;
    charset_types
        text/html
        text/css
        text/plain
        text/xml
        text/csv
        application/javascript
        application/json
        application/xml
        application/atom+xml
        application/rss+xml
        application/ld+json
        image/svg+xml;

    # ===== COMPRESSION: GZIP =====
    gzip on;
    gzip_vary on;
    gzip_min_length 1100;
    gzip_comp_level 6;
    gzip_proxied any;
    gzip_static on;
    gzip_types
        text/plain
        text/css
        text/xml
        text/csv
        application/javascript
        application/json
        application/xml
        application/atom+xml
        application/rss+xml
        application/ld+json
        application/manifest+json
        application/wasm
        image/svg+xml
        font/ttf
        font/otf;
    gzip_disable "msie6";

    # ===== COMPRESSION: BROTLI (if module installed) =====
    brotli on;
    brotli_static on;
    brotli_comp_level 6;
    brotli_min_length 1100;
    brotli_types
        text/plain
        text/css
        text/xml
        text/csv
        application/javascript
        application/json
        application/xml
        application/atom+xml
        application/rss+xml
        application/ld+json
        application/manifest+json
        application/wasm
        image/svg+xml
        font/ttf
        font/otf;

    # ===== COMPRESSION: ZSTD (if module installed) =====
    zstd on;
    zstd_static on;
    zstd_comp_level 6;
    zstd_min_length 1100;
    zstd_types
        text/plain
        text/css
        text/xml
        application/javascript
        application/json
        application/xml
        application/ld+json
        image/svg+xml;
}

# Per server block
server {
    listen 443 ssl;
    http2 on;
    server_name example.com www.example.com;
    root /var/www/sites/example.com;
    index index.html;

    # ===== CONTENT-LANGUAGE =====
    # Default language for the site
    # NOTE: snippets pattern is required so location blocks do not drop this
    set $site_language "en-US";

    # ===== SECURITY FAMILY (no MIME sniffing, etc) =====
    include snippets/common-security-headers.conf;
    add_header Content-Language $site_language always;

    # ===== ASSET CLASS BLOCKS =====
    # (See Section 11 for all asset class recipes; combine with caching recipes from framework-http-caching-headers.md)

    location ~* \.html$ {
        include snippets/common-security-headers.conf;
        add_header Content-Language $site_language always;
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
    }

    location ~* \.(css|js|mjs)$ {
        include snippets/common-security-headers.conf;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }

    location ~* \.(woff2|woff|ttf|otf|eot)$ {
        include snippets/common-security-headers.conf;
        add_header Access-Control-Allow-Origin "*" always;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }

    location ~* \.(png|jpe?g|gif|webp|avif|svg|ico)$ {
        include snippets/common-security-headers.conf;
        add_header Cache-Control "public, max-age=2592000" always;
    }

    location ~* \.pdf$ {
        include snippets/common-security-headers.conf;
        add_header Content-Disposition "attachment" always;
        add_header Cache-Control "public, max-age=86400" always;
    }

    location /user-uploads/ {
        include snippets/common-security-headers.conf;
        add_header Content-Type "application/octet-stream" always;
        add_header Content-Disposition "attachment" always;
        add_header Content-Security-Policy "default-src 'none'" always;
        add_header Cache-Control "public, max-age=86400" always;
    }

    location /api/ {
        include snippets/common-security-headers.conf;
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass_header Content-Type;
        proxy_pass_header Content-Disposition;
        proxy_pass_header Content-Language;
        add_header Vary "Accept-Encoding" always;
    }

    location / {
        include snippets/common-security-headers.conf;
        add_header Content-Language $site_language always;
        add_header Cache-Control "public, max-age=0, must-revalidate" always;
        try_files $uri $uri/ $uri.html =404;
    }
}
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

## 13. AUDIT CHECKLIST

Run through these 50 items for any production site. Each item is a single curl command or visual inspection. Pass means the listed expected value appears.

### Content-Type

1. [ ] HTML response has `Content-Type: text/html; charset=utf-8`.
2. [ ] CSS response has `Content-Type: text/css; charset=utf-8`.
3. [ ] JavaScript response has `Content-Type: application/javascript; charset=utf-8` (or `text/javascript`).
4. [ ] JSON response has `Content-Type: application/json` (charset optional).
5. [ ] XML sitemap has `Content-Type: application/xml`.
6. [ ] SVG response has `Content-Type: image/svg+xml`.
7. [ ] AVIF image has `Content-Type: image/avif`.
8. [ ] WebP image has `Content-Type: image/webp`.
9. [ ] WOFF2 font has `Content-Type: font/woff2` (NOT `application/octet-stream`).
10. [ ] WASM module has `Content-Type: application/wasm`.
11. [ ] PDF has `Content-Type: application/pdf`.
12. [ ] `X-Content-Type-Options: nosniff` is present on every response.
13. [ ] No response has `Content-Type: application/octet-stream` for known file extensions.
14. [ ] No response has `Content-Type: text/html` for non HTML files.
15. [ ] `default_type` in nginx config is `application/octet-stream` (not `text/html`).

### Content-Language

16. [ ] HTML response has `Content-Language: en-US` (or the correct site language).
17. [ ] Language code uses hyphen, not underscore (`en-US` not `en_US`).
18. [ ] Region code is uppercase two letter ISO 3166-1 (`US` not `usa`).
19. [ ] HTML `<html>` element has matching `lang` attribute.
20. [ ] HTML `<head>` has matching `<meta http-equiv="content-language" content="...">`.
21. [ ] Multi region sites have hreflang annotations in `<head>` for every language version.
22. [ ] hreflang includes `x-default` for the global default.
23. [ ] Localized paths (e.g. `/es/`) emit the correct `Content-Language` for that locale.

### Content-Encoding

24. [ ] HTML response with `Accept-Encoding: gzip` returns `Content-Encoding: gzip`.
25. [ ] HTML response with `Accept-Encoding: br` returns `Content-Encoding: br` (if brotli installed).
26. [ ] HTML response with `Accept-Encoding: zstd` returns `Content-Encoding: zstd` (if zstd installed).
27. [ ] CSS, JS, JSON, XML, SVG all compress.
28. [ ] Already compressed types (JPEG, PNG, WebP, MP4, WOFF2) do NOT have `Content-Encoding`.
29. [ ] Compressed responses have `Vary: Accept-Encoding`.
30. [ ] Request without `Accept-Encoding` returns uncompressed body.
31. [ ] No use of `Content-Encoding: deflate`.
32. [ ] Brotli module loaded via `load_module` if used.
33. [ ] Zstd module loaded via `load_module` if used.
34. [ ] Precompressed `.gz`, `.br`, `.zst` files exist for static assets in production.

### Content-Length

35. [ ] Static file response has `Content-Length` (when not on the fly compressed).
36. [ ] On the fly compressed response uses `Transfer-Encoding: chunked`.
37. [ ] No response has both `Content-Length` and `Transfer-Encoding`.
38. [ ] Declared `Content-Length` matches actual body byte count.
39. [ ] Large precompressed files have `Content-Length` showing the compressed size.

### Content-Disposition

40. [ ] PDF download endpoints have `Content-Disposition: attachment`.
41. [ ] Zip and archive endpoints have `Content-Disposition: attachment`.
42. [ ] User upload serving endpoints have `Content-Disposition: attachment` AND `Content-Type: application/octet-stream`.
43. [ ] Non download responses (HTML pages, CSS, JS, images) do NOT have `Content-Disposition: attachment`.
44. [ ] Filename parameters are wrapped in double quotes.
45. [ ] Non ASCII filenames include both `filename=` (ASCII fallback) and `filename*=` (RFC 5987 UTF-8).
46. [ ] User input is never directly concatenated into Content-Disposition values without sanitization.

### Cross-cutting

47. [ ] `nginx -t` passes without warnings.
48. [ ] `nginx -T` (effective config) shows all expected directives.
49. [ ] Lighthouse "Properly size images" and "Enable text compression" warnings are clear.
50. [ ] Bing Webmaster Tools shows correct language detection (if Bing account is connected).

A site that passes all 50 is ready for production traffic and crawling.

---

## 14. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Default type set to `text/html`.**
Symptom: unknown file extensions render as broken HTML in the browser.
Why it breaks: `default_type text/html` causes nginx to label every unknown extension as HTML, opening an XSS surface for any user controlled path.
Fix: `default_type application/octet-stream`. Add specific types via `types {}` block for any extension you actually serve.

**Pitfall 2: Charset missing from text Content-Type.**
Symptom: pages with non ASCII characters (curly quotes, em dashes, accented characters) display as mojibake.
Why it breaks: browser falls back to Windows-1252 or another locale default.
Fix: enable `charset utf-8;` and `charset_types` directive (Section 5.5).

**Pitfall 3: Mime type missing for new file extensions.**
Symptom: AVIF images not loading, WOFF2 fonts falling back to system fonts, WASM modules failing to instantiate.
Why it breaks: `/etc/nginx/mime.types` on older Debian releases predates these formats. nginx falls back to `default_type`.
Fix: add types in the http block (Section 5.6).

**Pitfall 4: Compressing already compressed content.**
Symptom: high CPU on nginx, no measurable savings.
Why it breaks: JPEGs, PNGs, MP4s, WOFF2 fonts are already compressed. Gzipping them costs CPU and may even increase size by a few bytes.
Fix: do not list `image/jpeg`, `image/png`, `image/webp`, `image/avif`, `video/*`, `audio/*`, `font/woff2`, `application/zip` in `gzip_types`, `brotli_types`, or `zstd_types`.

**Pitfall 5: Vary: Accept-Encoding missing.**
Symptom: random clients receive binary garbage; cache hit rates from CDNs or corporate proxies break.
Why it breaks: a cache serves a gzipped body to a client that did not request gzip.
Fix: `gzip_vary on;` (Section 7.4 and framework-http-caching-headers.md Section 9).

**Pitfall 6: Content-Disposition: attachment on every response.**
Symptom: every page tries to download instead of render.
Why it breaks: a server wide `add_header Content-Disposition "attachment"` was added somewhere.
Fix: remove the global declaration. Only apply `attachment` to specific download locations.

**Pitfall 7: Unsanitized filename in Content-Disposition.**
Symptom: security scanner reports vulnerability; user uploads can inject CRLF and forge headers.
Why it breaks: unsanitized user input flowing directly into a header value.
Fix: sanitize per Section 9.4. Never trust client supplied filenames.

**Pitfall 8: User uploaded HTML rendered inline.**
Symptom: stored XSS in user uploads.
Why it breaks: file served with `Content-Type: text/html` and no `Content-Disposition: attachment`.
Fix: see Section 9.5. Force `attachment` plus `application/octet-stream` on every user upload endpoint.

**Pitfall 9: Content-Language using underscores or wrong case.**
Symptom: language detection fails in screen readers, translation tools, Bing indexing.
Why it breaks: `en_US` is POSIX locale syntax, not BCP 47. `EN-US` violates the conventional casing.
Fix: `en-US` exactly. Lowercase language, uppercase region, hyphen between.

**Pitfall 10: Setting Content-Language without matching `<html lang>`.**
Symptom: inconsistent language signals; some crawlers ignore the page, some misindex.
Why it breaks: the HTTP header says `es-MX`, the HTML says `<html lang="en">`. Crawlers do not know which to trust.
Fix: align all three layers: HTTP header, meta tag, html attribute.

**Pitfall 11: Both Content-Length and Transfer-Encoding present.**
Symptom: HTTP request smuggling vulnerability scanner alert.
Why it breaks: per the RFC, this combination must not occur. Different intermediaries may interpret framing differently, enabling smuggling.
Fix: find the upstream emitting both. Remove the manual Content-Length and let chunked dominate.

**Pitfall 12: Content-Length wrong for an upstream response.**
Symptom: clients report "incomplete response" or truncated content.
Why it breaks: upstream computed Content-Length before adding (or after removing) some bytes.
Fix: find the upstream code path. Either trust the framework to set Content-Length automatically, or compute it after the body is finalized.

**Pitfall 13: Double compression (serving a .gz file with Content-Encoding: gzip).**
Symptom: browser fails to decompress.
Why it breaks: a `.tar.gz` archive is application data; serving it as `Content-Encoding: gzip` tells the browser to decompress the wire bytes, but then the result is still a `.tar` that the browser cannot show.
Fix: serve `.gz` files (when intended as downloads) with `Content-Type: application/gzip` and NO `Content-Encoding`. Use `gzip_static on` for precompressed CSS/JS, which sets the headers correctly.

**Pitfall 14: Add_header inheritance trap on Content-Type.**
Symptom: nosniff or other security headers missing on certain file types after a location block override.
Why it breaks: any `add_header` in a location wipes out all parent add_headers (see framework-http-caching-headers.md Section 5.5).
Fix: snippet include pattern in every location.

**Pitfall 15: Hardcoded Content-Type overriding nginx mime detection.**
Symptom: PNG returns `Content-Type: text/html`, JS returns `Content-Type: text/plain`, etc.
Why it breaks: someone added `add_header Content-Type "..."` in the wrong scope.
Fix: search the config: `grep -rn "Content-Type" /etc/nginx/`. Remove inappropriate manual declarations. Let mime.types handle it.

---

## 15. DIAGNOSTIC COMMANDS

Reference of every command useful for content header investigation.

### Inspect content headers

```bash
# Full header set
curl -sI https://example.com/page.html

# Content family only
curl -sI https://example.com/page.html | grep -iE "content-(type|language|encoding|length|disposition)"

# As Googlebot
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
     https://example.com/page.html

# As Bingbot (matters for Content-Language)
curl -sI -A "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)" \
     https://example.com/page.html

# As ClaudeBot
curl -sI -A "Mozilla/5.0 (compatible; ClaudeBot/1.0; +claudebot@anthropic.com)" \
     https://example.com/page.html
```

### Verify Content-Type

```bash
# Confirm declared type
curl -sI https://example.com/script.js | grep -i content-type

# Verify actual file type
curl -s -o /tmp/asset https://example.com/script.js
file --mime-type /tmp/asset
# Compare to declared

# Detect mojibake risk: dump first 200 bytes
curl -s https://example.com/ | head -c 200 | xxd

# Verify HTML meta charset is in first 1024 bytes
curl -s https://example.com/ | head -c 1024 | grep -i charset
```

### Verify Content-Language

```bash
# Header check
curl -sI https://example.com/ | grep -i content-language
curl -sI https://example.com/es/ | grep -i content-language

# HTML lang attribute
curl -s https://example.com/ | grep -oE '<html[^>]*lang="[^"]*"'

# Meta http-equiv
curl -s https://example.com/ | grep -oE '<meta[^>]*http-equiv="content-language"[^>]*>'

# All hreflang annotations
curl -s https://example.com/ | grep -oE 'hreflang="[^"]*"' | sort -u
```

### Verify Content-Encoding

```bash
# Compression negotiation
for enc in gzip br zstd identity; do
    SIZE=$(curl -s -H "Accept-Encoding: $enc" --output - https://example.com/page.html | wc -c)
    HDR=$(curl -sI -H "Accept-Encoding: $enc" https://example.com/page.html | grep -i content-encoding | awk '{print $2}' | tr -d '\r')
    echo "$enc: declared=$HDR size=$SIZE"
done

# Verify compressed payload actually decompresses to expected content
curl -s --compressed https://example.com/page.html > /tmp/decompressed.html
curl -s -H "Accept-Encoding: identity" https://example.com/page.html > /tmp/raw.html
diff /tmp/decompressed.html /tmp/raw.html && echo "Match"

# Check that brotli module is loaded
nginx -V 2>&1 | grep -o brotli
ls /etc/nginx/modules/ | grep brotli

# Check that zstd module is loaded
nginx -V 2>&1 | grep -o zstd
ls /etc/nginx/modules/ | grep zstd
```

### Verify Content-Length

```bash
# Declared vs actual (identity, no compression)
DECLARED=$(curl -sI -H "Accept-Encoding: identity" https://example.com/style.css | grep -i content-length | awk '{print $2}' | tr -d '\r')
ACTUAL=$(curl -s -H "Accept-Encoding: identity" https://example.com/style.css | wc -c)
echo "Declared: $DECLARED, Actual: $ACTUAL"
[ "$DECLARED" = "$ACTUAL" ] && echo "Match" || echo "MISMATCH"

# Confirm chunked when compressed
curl -sI -H "Accept-Encoding: gzip" https://example.com/page.html | grep -iE "content-length|transfer-encoding"

# Look for the dangerous both-headers combo
curl -sI https://example.com/api/data | awk 'tolower($0) ~ /^(content-length|transfer-encoding):/ {print}'
```

### Verify Content-Disposition

```bash
# Confirm attachment for downloads
curl -sI https://example.com/downloads/report.pdf | grep -i content-disposition

# Confirm inline (or absent) for renderable content
curl -sI https://example.com/page.html | grep -i content-disposition

# Test that user uploads are forced to download
curl -sI https://example.com/user-uploads/file.html | grep -iE "content-(type|disposition)"
# Both must be present and restrictive

# Check filename quoting
curl -sI https://example.com/downloads/report.pdf | grep -i content-disposition
# Filename should be wrapped in double quotes
```

### Server side investigation

```bash
# Check nginx mime.types
cat /etc/nginx/mime.types | grep -i avif
cat /etc/nginx/mime.types | grep -i woff2
cat /etc/nginx/mime.types | grep -i wasm

# Show all add_header in active config
nginx -T 2>/dev/null | grep -i add_header

# Show all compression config in effect
nginx -T 2>/dev/null | grep -iE "gzip|brotli|zstd"

# Show all charset directives
nginx -T 2>/dev/null | grep -i charset

# Confirm modules loaded
nginx -V 2>&1 | tr ' ' '\n' | grep -iE "brotli|zstd"

# Inspect source file encoding
file -bi /var/www/sites/example.com/index.html
# Expected: text/html; charset=utf-8

# Find files that are not UTF-8
find /var/www/sites/example.com -name "*.html" -exec file -bi {} \; | grep -v utf-8

# After config changes
nginx -t && systemctl reload nginx
```

### Browser DevTools quick reference

In Chrome DevTools Network panel, click any request, then:

* **Headers** tab: shows all response headers including the five in this framework.
* **Response** tab: shows the decompressed body. If the body looks like binary garbage here, Content-Encoding is wrong.
* **Size column**: shows wire size (compressed). Hover for "encoded size" and "actual size" (decompressed).
* The "Type" column shows the browser's interpretation based on Content-Type (Document, Stylesheet, Script, XHR, Image, Font, etc).

To force a download when the server returned inline:

* Right click on a link, "Save Link As".
* Or add `?download=true` and have the upstream check for the query and add `Content-Disposition: attachment`.

---

## 16. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): Cache-Control, ETag, Last-Modified, Expires, Vary, Age. Vary: Accept-Encoding is shared between this framework and that one; gzip weakens ETags is shared knowledge.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Section 24 (Bubbles Nginx config) inherits content header configuration from here.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook. References this framework for any content header decision.
* [framework-security-headers.md](framework-security-headers.md): HSTS, CSP, X-Frame-Options, X-Content-Type-Options. The nosniff header is critical for the Content-Type discipline described here.
* [framework-corewebvitals.md](framework-corewebvitals.md): LCP, INP, CLS targets. Content-Encoding directly affects LCP (smaller payloads transmit faster).
* [framework-hreflang.md](framework-hreflang.md): international SEO. Companion to Content-Language Section 6 here.
* [framework-compression.md](framework-compression.md): deep dive on gzip, brotli, zstd configuration, performance tuning, and migration. This file references it for advanced compression scenarios.
* Google official documentation on Content-Type and indexing: https://developers.google.com/search/docs/crawling-indexing/indexable-content
* Google international and multilingual sites: https://developers.google.com/search/docs/specialty/international/overview
* MDN Content-Type: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Type
* MDN Content-Disposition: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
* RFC 9110 (HTTP Semantics): https://www.rfc-editor.org/rfc/rfc9110
* RFC 6266 (Content-Disposition in HTTP): https://www.rfc-editor.org/rfc/rfc6266
* RFC 5987 (Character set and language encoding for HTTP header parameters): https://www.rfc-editor.org/rfc/rfc5987

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Content-Type table

| Asset type | Content-Type |
|---|---|
| HTML | `text/html; charset=utf-8` |
| CSS | `text/css; charset=utf-8` |
| JavaScript | `application/javascript; charset=utf-8` |
| JSON | `application/json` |
| JSON-LD | `application/ld+json` |
| XML sitemap | `application/xml` |
| SVG | `image/svg+xml` |
| PNG | `image/png` |
| JPEG | `image/jpeg` |
| WebP | `image/webp` |
| AVIF | `image/avif` |
| WOFF2 | `font/woff2` |
| PDF | `application/pdf` |
| ZIP | `application/zip` |
| WASM | `application/wasm` |
| Web manifest | `application/manifest+json` |
| Plain text | `text/plain; charset=utf-8` |
| Unknown (safe default) | `application/octet-stream` |

### Content-Language

* Always `en-US` (or correct BCP 47) on every HTML response.
* Match `<html lang="en-US">`.
* Match `<meta http-equiv="content-language" content="en-US">`.
* Add hreflang for multi region sites.

### Content-Encoding

* Compress: HTML, CSS, JS, JSON, XML, SVG, plain text, fonts (ttf/otf only).
* Do NOT compress: JPEG, PNG, WebP, AVIF, MP4, WOFF2 (already compressed).
* Always pair with `Vary: Accept-Encoding`.
* Prefer precompressed (`gzip_static`, `brotli_static`, `zstd_static`) for static assets.

### Content-Length

* Auto for static files.
* Auto for precompressed files (the compressed size).
* Replaced with `Transfer-Encoding: chunked` when on the fly compression is active.
* Never appears with `Transfer-Encoding` together.

### Content-Disposition

* `attachment` for downloads (PDF, ZIP, generated CSVs).
* Omitted (default `inline`) for HTML, CSS, JS, images that should render.
* User uploads: always `attachment` plus `application/octet-stream`.
* Filenames wrapped in double quotes, both `filename` and `filename*` for non ASCII.

Three commands every operator should know:

```bash
# Confirm content type is correct
curl -sI https://example.com/script.js | grep -i content-type

# Confirm compression is working
curl -sI -H "Accept-Encoding: br" https://example.com/page.html | grep -iE "content-encoding|vary"

# Apply changes
nginx -t && systemctl reload nginx
```

And two end to end test invocations:

```bash
# Verify a JSON API returns the right type and decompresses correctly
curl -s --compressed -H "Accept-Encoding: br, gzip" https://example.com/api/data | head -c 200
curl -sI --compressed https://example.com/api/data | grep -iE "content-(type|encoding|length)"

# Verify a PDF download has the right disposition
curl -sI https://example.com/downloads/report.pdf | grep -iE "content-(type|disposition|length)"
```

If both produce expected output, the content header stack is correctly wired.

---

End of framework-http-content-headers.md.
