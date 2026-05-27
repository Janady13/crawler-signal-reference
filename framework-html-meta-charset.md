# framework-html-meta-charset.md

Comprehensive reference for the HTML `<meta charset="utf-8">` declaration and the complete character encoding story across the five operational layers (HTTP response header, HTML document declaration, file system encoding, database encoding, form submission encoding). Covers the HTML5 syntax, the legacy HTML4 `<meta http-equiv="Content-Type">` form, the 1024 byte placement rule, byte order mark (BOM) handling, the encoding detection cascade browsers perform when declarations conflict, UTF-8 vs other encodings, the mojibake debugging problem, and the security implications of missing charset declarations. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, PostgreSQL with default UTF-8 encoding, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the second framework in the HTML signal track**, following framework-html-meta-robots.md. Companion to the 12 wire layer frameworks, especially framework-http-content-headers.md (which covers the `Content-Type` HTTP header that pairs with the HTML charset declaration).

Audience: humans hand coding HTML who need to get encoding right on the first try, AI assistants generating HTML head sections that survive special character content, operators debugging "the apostrophe shows as â€™ on my site", developers serving Marshallese, Korean, Chinese, Arabic, or other non Latin script content (relevant: the Marshallese-Voices client), and anyone troubleshooting "form submission corrupts characters", "database returns question marks for Unicode", or "Google search result shows broken characters in the title".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Character Encoding Mental Model (read this first)
5. The HTML5 charset Declaration vs The HTML4 Equivalent
6. The Five Layer Alignment Problem
7. The 1024 Byte Rule (placement requirement)
8. The Byte Order Mark (BOM) Problem
9. The Encoding Detection Cascade
10. UTF-8 vs Other Encodings (why UTF-8 wins in 2026)
11. The Content-Type HTTP Header Interaction
12. Form Submission Encoding (accept-charset)
13. Database Encoding (the PostgreSQL pattern)
14. The Mojibake Problem (debugging broken characters)
15. Special Characters: HTML Entities vs Raw UTF-8
16. The Security Implications (UTF-7 sniffing, charset based XSS)
17. The Marshallese-Voices Case Study (why this matters)
18. Asset Class And Use Case Recipes
19. Bubbles Standard Pattern (paste ready)
20. Audit Checklist (50+ items)
21. Common Pitfalls
22. Diagnostic Commands
23. Cross-References

---

## 1. DEFINITION

`<meta charset="utf-8">` is the HTML5 declaration that informs the browser (and any HTML parser, including search engine crawlers) which character encoding the document uses. Defined in the HTML Living Standard. The declaration must appear within the first 1024 bytes of the document for browsers to honor it (the "encoding sniffing" cutoff).

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Page Title</title>
    ...
</head>
```

Three structural facts shape how the declaration works:

* **The browser needs to know the encoding before parsing the page.** Different encodings interpret the same bytes differently. ASCII byte `0xE9` is undefined in pure ASCII, "é" in ISO-8859-1, and the start of a 2 byte sequence in UTF-8. Without a declaration, browsers guess (poorly).
* **The HTML5 form (`<meta charset="utf-8">`) is the modern syntax.** The HTML4 form (`<meta http-equiv="Content-Type" content="text/html; charset=utf-8">`) still works but is verbose and legacy. New code uses the short form.
* **The HTTP Content-Type header takes precedence over the HTML meta tag.** If the server sends `Content-Type: text/html; charset=ISO-8859-1` and the HTML says `<meta charset="utf-8">`, the browser uses ISO-8859-1 (HTTP wins). The HTML declaration is a fallback.

For Bubbles client sites in 2026, the only correct answer is UTF-8 across every layer. UTF-8 covers every character in every language, including emoji, mathematical symbols, and historical scripts. Any non UTF-8 encoding is a legacy artifact that should be migrated away.

---

## 2. WHY IT MATTERS

Eight independent pressures push correct charset declaration from "default behavior" to "actively managed signal" in 2025 and forward.

**UTF-8 is the universal correct answer; everything else is wrong.** Per the WHATWG HTML Living Standard, all new HTML documents should be UTF-8. ASCII is a subset of UTF-8 (every ASCII byte is also a valid UTF-8 byte). ISO-8859-1, Windows-1252, and other legacy encodings are deprecated. Any site using a non UTF-8 encoding in 2026 has a bug waiting to surface the moment any non ASCII character enters the content stream.

**Mojibake is invisible to the author and visible to every visitor.** Wrong encoding produces broken character display: smart quotes become "â€œ" and "â€", em dashes become "â€"", apostrophes become "â€™". For sites with editorial content pulled from Microsoft Word or other source applications (which produce smart quotes by default), encoding mismatches turn every quoted phrase into garbage. The author writing the content rarely sees this because their browser correctly handles the document; the issue appears only to certain visitors with caches or proxies that mishandle the encoding.

**The 1024 byte rule is unforgiving.** Per the HTML5 spec, browsers only honor the meta charset declaration if it appears in the first 1024 bytes of the document. Sites with bloated head sections (excessive comment blocks, large inline scripts before the charset tag, multiple high priority preload tags before charset) can push the declaration past the cutoff. The browser then defaults to its own encoding heuristic, which is often wrong.

**The Content-Type HTTP header overrides the meta tag.** If the server sends a Content-Type with charset that does not match the HTML declaration, the HTTP wins. Sites that say `<meta charset="utf-8">` but the server sends `Content-Type: text/html` (no charset) work correctly. Sites that say `<meta charset="utf-8">` but the server sends `Content-Type: text/html; charset=ISO-8859-1` are broken regardless of what the HTML says.

**Forms submit data using the page's encoding by default.** A form on a UTF-8 page submits form data as UTF-8. A form on an ISO-8859-1 page submits as ISO-8859-1. Server side code that always parses input as UTF-8 will produce mojibake when the input was actually ISO-8859-1. The `accept-charset="utf-8"` attribute on `<form>` elements forces UTF-8 submission regardless of page encoding.

**Database encoding must align.** PostgreSQL defaults to UTF-8 (modern versions; very old versions defaulted to SQL_ASCII). Connections from FastAPI to PostgreSQL must declare UTF-8 in the connection string. Sites with UTF-8 HTML and UTF-8 form submission but a database created with SQL_ASCII encoding produce silent data corruption: special characters are stored as raw bytes that cannot be retrieved correctly.

**Security: charset based XSS attacks.** Without an explicit charset declaration, browsers historically performed "charset sniffing" on the page content. Attackers could inject UTF-7 encoded payloads that survived ASCII filters and were then decoded as JavaScript by the browser. Per OWASP recommendations, always declare an explicit charset. The `X-Content-Type-Options: nosniff` header (covered in framework-http-security-headers.md) further reduces this risk.

**SEO: broken characters in search results.** Google indexes the page using the declared encoding. Wrong encoding means broken characters appear in the search snippet, title tag, and meta description. Users see garbage in search results; click through rate plummets. The fix is correct charset declaration; the indexing pipeline then re processes the page correctly within days to weeks.

**Cost of getting it wrong.** Misconfigured character encoding produces visible content damage and silent data corruption. Real examples:

* Bubbles client editorial site copy pasted articles from a Word document. Smart quotes survived as UTF-8 bytes in the source file. Page declared no charset. Some browsers rendered correctly; others showed "â€œ" and "â€" for every quoted phrase. Reader complaints surfaced 3 weeks after launch. Fix: add `<meta charset="utf-8">` to all templates.
* Marshallese-Voices content with diacritics (ā, ļ, ņ, ō, ū, etc) loaded fine in development. Production database (created years earlier with default SQL_ASCII) stored the characters as raw bytes. Display worked because the bytes happened to round trip through the SQL_ASCII to UTF-8 conversion correctly. Then the team needed to search the database for "Mājeej" and found nothing; the search query was UTF-8 but the database contents were SQL_ASCII interpreted as UTF-8. Three layer encoding mismatch.
* Contact form on a Bubbles client site received submissions with names like "Joseé". The form page was UTF-8 but the response from form processing used Latin-1 encoding internally; the é was double encoded. After fix: `accept-charset="utf-8"` on the form, UTF-8 throughout the pipeline.
* Marketing site declared `<meta charset="utf-8">` correctly, but server sent `Content-Type: text/html; charset=ISO-8859-1` (nginx default for HTML in some old configurations). Browser used ISO-8859-1. Mojibake everywhere. Fix: nginx `charset utf-8;` directive plus ensure `Content-Type` includes charset=utf-8.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The single tag plus its full operational context:

1. **The declaration itself**: HTML5 short form vs HTML4 long form.
2. **The placement rule**: within the first 1024 bytes.
3. **The five layer alignment**: HTTP, HTML, file, database, form.
4. **The encoding detection cascade**: what browsers do when declarations are missing or conflict.
5. **The security implications**: charset sniffing attacks, UTF-7 historical issues.
6. **The debugging methodology**: how to find and fix mojibake.

Section 17 covers the Marshallese-Voices case study (a Bubbles client where UTF-8 actually matters for non Latin script content). Sections 19 to 22 are the standard reference, audit, and diagnostic tooling.

---

## 4. THE CHARACTER ENCODING MENTAL MODEL (READ THIS FIRST)

A character encoding is a mapping from bytes to characters. Different encodings map the same bytes to different characters.

```
Byte sequence: 0xE2 0x80 0x99
   |
   |---> Interpreted as UTF-8 ........... right single quotation mark (')
   |
   |---> Interpreted as ISO-8859-1 ...... three characters: â, €, ™
   |
   |---> Interpreted as Windows-1252 .... three characters: â, €, ™

================ THE ENCODING JOURNEY ================

Author writes content with special characters
   |
   v
File saved with some encoding (file metadata or implicit)
   |
   v
Server reads file, sends to client
   |
   v
HTTP response: Content-Type: text/html; charset=??
   |
   v
HTML content includes <meta charset="..."> declaration
   |
   v
Browser parses encoding declaration, interprets bytes accordingly
   |
   v
Characters render correctly (or as mojibake)

================ WHERE MISMATCHES HAPPEN ================

* File saved as Windows-1252 but server declares UTF-8.
* HTML declares UTF-8 but HTTP header declares ISO-8859-1.
* Form submits as UTF-8 but server parses as Windows-1252.
* Database stores as SQL_ASCII but expected UTF-8.
* JavaScript reads localStorage as Latin-1 but stored as UTF-8.
```

Six rules govern the system:

1. **UTF-8 everywhere.** Every layer of the stack must use UTF-8 in 2026. No exceptions.
2. **Declare charset explicitly.** Never rely on browser sniffing.
3. **HTTP header takes precedence over HTML meta tag.** Both should match (and both should say UTF-8).
4. **Declare charset within the first 1024 bytes of the HTML.** Browsers ignore late declarations.
5. **No byte order mark (BOM) for UTF-8 in HTML.** Browsers handle but the BOM causes other issues.
6. **All five layers must align.** HTTP, HTML, file, database, form.

A correctly configured site declares UTF-8 at every layer, places the `<meta charset>` declaration as the first tag inside `<head>`, never uses a BOM, and verifies alignment via curl checks and database inspection.

---

## 5. THE HTML5 CHARSET DECLARATION VS THE HTML4 EQUIVALENT

### 5.1 The HTML5 Short Form (Modern)

```html
<meta charset="utf-8">
```

The canonical modern syntax. Concise, unambiguous, recommended by the HTML Living Standard. All major browsers since IE9 (2011) honor this form.

### 5.2 The HTML4 Long Form (Legacy)

```html
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
```

The older syntax, intended to mimic an HTTP header within the HTML. Still works in all browsers but is verbose and signals legacy. Sometimes seen on older sites or in CMS templates that have not been updated.

The two forms are functionally equivalent for charset declaration purposes. New code should use the short form.

### 5.3 Combined Form (Defensive)

Some sites emit both for maximum compatibility:

```html
<head>
    <meta charset="utf-8">
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
</head>
```

This is unnecessary in 2026 (IE9+ supports the short form, and all modern browsers prefer the short form when both are present). It is harmless but wasteful.

For Bubbles: use the short form only.

### 5.4 The Case Insensitivity Rule

Both forms are case insensitive:

```html
<meta CHARSET="UTF-8">
<meta charset="UTF-8">
<meta charset="utf-8">
<META CHARSET="utf-8">
```

All identical to the browser. Convention is lowercase: `<meta charset="utf-8">`.

### 5.5 The Quotation Mark Rule

Attribute values may be quoted or unquoted in HTML5:

```html
<meta charset="utf-8">    <!-- Quoted (canonical) -->
<meta charset='utf-8'>    <!-- Single quoted -->
<meta charset=utf-8>      <!-- Unquoted (HTML5 allows for simple values) -->
```

Convention is double quoted for consistency with most HTML examples.

---

## 6. THE FIVE LAYER ALIGNMENT PROBLEM

For UTF-8 to work end to end, all five layers must agree. A mismatch at any layer produces mojibake.

### 6.1 Layer 1: HTTP Response Content-Type Header

The nginx response must include charset=utf-8 in the Content-Type header.

```nginx
# /etc/nginx/sites-available/example.com
server {
    # Method 1: nginx charset directive (recommended)
    charset utf-8;

    # Method 2: explicit Content-Type with charset (alternative)
    # location ~* \.html$ {
    #     add_header Content-Type "text/html; charset=utf-8" always;
    # }
}
```

The `charset utf-8;` directive tells nginx to add `; charset=utf-8` to the Content-Type for text responses.

Verify:

```bash
curl -sI https://example.com/ | grep -i content-type
# Expected: content-type: text/html; charset=utf-8
```

### 6.2 Layer 2: HTML Meta Charset Declaration

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <!-- Other tags follow -->
</head>
```

The declaration MUST be in the first 1024 bytes of the document. Place it as the first child of `<head>`.

Verify:

```bash
# Check that meta charset is in the first 1024 bytes
curl -s https://example.com/ | head -c 1024 | grep -i 'charset="utf-8"'
```

### 6.3 Layer 3: File System Encoding

The .html file on disk must be saved as UTF-8 (without BOM).

```bash
# Check file encoding
file -i /var/www/sites/example.com/index.html
# Expected: index.html: text/html; charset=utf-8

# If incorrect, convert
iconv -f WINDOWS-1252 -t UTF-8 input.html > output.html

# Strip BOM if present
sed -i '1s/^\xEF\xBB\xBF//' /var/www/sites/example.com/index.html
```

For Joseph's hand coded sites: ensure the text editor saves as UTF-8 without BOM (most modern editors default to this; VS Code, vim, nano all default to UTF-8).

### 6.4 Layer 4: Database Encoding (PostgreSQL)

PostgreSQL databases have a server encoding setting. For Bubbles:

```bash
# Check PostgreSQL server encoding
sudo -u postgres psql -c "SHOW server_encoding;"
# Expected: UTF8

# Check database specific encoding
sudo -u postgres psql -d bubbles -c "SELECT datname, pg_encoding_to_char(encoding) FROM pg_database WHERE datname = 'bubbles';"
# Expected: bubbles | UTF8
```

If a database was created with SQL_ASCII (the historical default on some installs), it must be dumped, recreated as UTF-8, and restored:

```bash
# Dump existing database
sudo -u postgres pg_dump bubbles > /tmp/bubbles.dump

# Drop and recreate with UTF-8
sudo -u postgres psql -c "DROP DATABASE bubbles;"
sudo -u postgres psql -c "CREATE DATABASE bubbles WITH ENCODING='UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE=template0;"

# Restore
sudo -u postgres psql bubbles < /tmp/bubbles.dump
```

Connection string from FastAPI should explicitly request UTF-8:

```python
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/bubbles?client_encoding=utf8"
```

### 6.5 Layer 5: Form Submission Encoding

HTML forms submit data using the page's encoding by default. To force UTF-8 regardless:

```html
<form action="/submit" method="post" accept-charset="utf-8">
    <!-- ... -->
</form>
```

The `accept-charset="utf-8"` attribute is belt and suspenders: it forces the browser to submit form data as UTF-8 even if the page encoding is different.

For FastAPI receiving form data, the default encoding is UTF-8 (FastAPI handles this correctly by default). No additional configuration needed.

### 6.6 The Verification Across All Five

```bash
#!/bin/bash
# /usr/local/bin/charset-audit.sh

URL=$1
FILE=$2

echo "=== Charset audit ==="

# Layer 1: HTTP Content-Type
CT=$(curl -sI "$URL" | grep -i content-type)
echo "HTTP Content-Type: $CT"
if [[ "$CT" =~ "charset=utf-8" ]]; then
    echo "  Layer 1 (HTTP): OK"
else
    echo "  Layer 1 (HTTP): MISSING charset=utf-8"
fi

# Layer 2: HTML meta charset
HEAD_BYTES=$(curl -s "$URL" | head -c 1024)
if echo "$HEAD_BYTES" | grep -qi 'charset="utf-8"\|charset=utf-8'; then
    echo "  Layer 2 (HTML): OK (within first 1024 bytes)"
else
    echo "  Layer 2 (HTML): MISSING or beyond 1024 bytes"
fi

# Layer 3: file encoding (if file path provided)
if [ -n "$FILE" ]; then
    FILE_ENC=$(file -i "$FILE")
    echo "File encoding: $FILE_ENC"
    if [[ "$FILE_ENC" =~ "charset=utf-8" ]]; then
        echo "  Layer 3 (file): OK"
    else
        echo "  Layer 3 (file): MISMATCH"
    fi
fi

# Layer 4: database
DB_ENC=$(sudo -u postgres psql -d bubbles -At -c "SHOW server_encoding;" 2>/dev/null)
echo "Database encoding: $DB_ENC"
if [ "$DB_ENC" = "UTF8" ]; then
    echo "  Layer 4 (db): OK"
else
    echo "  Layer 4 (db): NOT UTF8"
fi

# Layer 5: form encoding (look for accept-charset in forms)
FORM_CHARSETS=$(curl -s "$URL" | grep -oE 'accept-charset="[^"]+"' | head -3)
echo "Form accept-charset attributes: $FORM_CHARSETS"

echo "=== End audit ==="
```

---

## 7. THE 1024 BYTE RULE (PLACEMENT REQUIREMENT)

Per the HTML5 spec, browsers check the first 1024 bytes of an HTML document for a charset declaration. If no charset is found in that window, the browser falls back to sniffing or its default encoding.

### 7.1 The Practical Implication

Place `<meta charset="utf-8">` as the FIRST tag inside `<head>`. Before title, before viewport, before any link or script tags.

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">             <!-- FIRST -->
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Page Title</title>
    <!-- everything else follows -->
</head>
```

### 7.2 The Failure Mode

Sites with bloated head sections can accidentally push the charset declaration past the 1024 byte cutoff. Common offenders:

* Excessive comment headers at the top of the document.
* Large inline `<style>` blocks before the charset declaration.
* Preload tags emitted by CMS or framework conventions before the charset.
* Multiple `<meta>` tags with very long content attributes (Open Graph descriptions, schema, etc) before charset.

When this happens:

```html
<!-- WRONG: charset pushed past 1024 bytes -->
<!doctype html>
<html>
<head>
    <!--
    This is a long comment block describing the page.
    It explains the purpose, authors, change history,
    and lots of other context that takes hundreds of bytes.
    ...
    [many lines]
    ...
    -->
    <meta property="og:description" content="A very long Open Graph description that takes another 200 bytes...">
    <meta property="og:title" content="...">
    <!-- ... many more meta tags ... -->
    <meta charset="utf-8">    <!-- TOO LATE; might be past 1024 bytes -->
```

Browsers do not see the charset declaration and fall back to sniffing.

### 7.3 The Verification

```bash
# Check exact byte position of the charset declaration
curl -s https://example.com/ | head -c 2000 | grep -bn 'charset'
# Output format: "byte_offset:line_number:content"
# If byte_offset > 1024, the declaration is too late
```

### 7.4 The Defense

Always place `<meta charset="utf-8">` immediately after the opening `<head>` tag, before anything else:

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <!-- everything else after charset -->
```

This consistently keeps the declaration within the first ~50 bytes of the head section.

---

## 8. THE BYTE ORDER MARK (BOM) PROBLEM

A byte order mark is a Unicode signature at the start of a file. For UTF-8, the BOM is the three byte sequence `0xEF 0xBB 0xBF`.

### 8.1 Why The BOM Exists

The BOM serves two purposes:

1. **Identify the encoding**: UTF-8 vs UTF-16 vs UTF-32.
2. **Indicate byte order**: for UTF-16/32, which byte comes first (little endian vs big endian).

For UTF-8 specifically, byte order is irrelevant (UTF-8 is byte stream oriented). The BOM only serves as a signature.

### 8.2 The Problem In HTML Context

A BOM at the start of an HTML file causes problems:

* **Whitespace appears at the top of the page.** Some browsers render the BOM as a visible character or as additional vertical space.
* **PHP and other server side languages produce "headers already sent" errors** because the BOM bytes are output before the headers can be set.
* **Comparison with `<!doctype html>` fails.** Some validation tools expect the document to start exactly with `<!doctype html>` and the BOM violates this.

### 8.3 The Convention

For HTML files: never use a BOM. Modern editors save UTF-8 without BOM by default. Verify with `file -i`:

```bash
# Check for BOM
file -i /var/www/sites/example.com/index.html

# Without BOM:
# index.html: text/html; charset=utf-8

# With BOM:
# index.html: text/html; charset=utf-8 (with byte order mark)
```

### 8.4 Stripping A BOM

```bash
# Remove BOM from a single file
sed -i '1s/^\xEF\xBB\xBF//' /var/www/sites/example.com/index.html

# Remove BOM from all HTML files in a directory
find /var/www/sites/example.com/ -name "*.html" -exec \
    sed -i '1s/^\xEF\xBB\xBF//' {} \;

# Or using awk
awk 'NR==1 {sub(/^\xef\xbb\xbf/, "")} {print}' input.html > output.html
```

### 8.5 Editor Settings

For hand coded HTML (Joseph's workflow):

* **VS Code**: by default saves UTF-8 without BOM. Verify in status bar (bottom right shows "UTF-8" not "UTF-8 with BOM").
* **vim**: by default does not add BOM. To explicitly disable: `:set nobomb`.
* **nano**: default does not add BOM.
* **Sublime Text**: by default does not add BOM. Check Preferences > Settings: `"show_encoding": true`.
* **Notepad** (Windows): historically adds BOM. Use Notepad++ instead (configurable to "UTF-8 without BOM").

For CSS, JavaScript, and JSON files: the same rule applies. Never use a BOM. JSON specifically forbids the BOM per the RFC.

---

## 9. THE ENCODING DETECTION CASCADE

When a browser fetches an HTML document, it determines the encoding in a specific order. Per the HTML5 spec:

### 9.1 The Cascade Order

1. **User specified override** (browser settings, F12 DevTools).
2. **Byte order mark (BOM)** if present.
3. **HTTP Content-Type header** charset parameter.
4. **`<meta charset>` or `<meta http-equiv>` within the first 1024 bytes**.
5. **Heuristic detection** based on byte patterns (the "sniffer").

The first match wins. Later sources are ignored.

### 9.2 The Implications

* **HTTP wins over HTML.** If both are present, HTTP is authoritative.
* **BOM wins over both.** If a BOM is present, it overrides everything else (which is why removing the BOM is essential when HTTP/HTML say UTF-8 anyway).
* **Heuristic detection is unreliable.** It works for some encoding patterns but fails for short documents or mixed content.

### 9.3 The Browser Sniffer Heuristic

When no declaration is present, browsers attempt to detect the encoding from the byte patterns:

* If bytes look like ASCII only: treat as ASCII (compatible with UTF-8).
* If bytes have UTF-8 multi byte sequences in valid patterns: treat as UTF-8.
* If bytes have ISO-8859-1 single byte patterns: treat as ISO-8859-1.
* If bytes have Windows-1252 patterns: treat as Windows-1252.

The heuristic is reasonable for clear cut cases but fails for:

* Short documents with insufficient bytes to detect.
* Documents with mixed encodings (e.g., UTF-8 with a few bytes from another encoding).
* Documents that look like one encoding but are actually another.

**The lesson:** always declare. Never rely on the heuristic.

### 9.4 The HTML5 Spec Quote

> If the document was served with a Content-Type metadata that included a charset parameter, then the charset parameter is the document's character encoding... If the document was not served with such metadata, then the document's character encoding is determined by an algorithm.

The algorithm is the cascade above. The first item with a valid charset wins.

---

## 10. UTF-8 VS OTHER ENCODINGS (WHY UTF-8 WINS IN 2026)

### 10.1 The Universal Recommendation

Per the WHATWG HTML Living Standard, all new HTML documents should use UTF-8:

> "Authors are encouraged to use UTF-8."

Per Google, Bing, and every major web standards body: UTF-8 is the default and recommended encoding for the web.

### 10.2 Why UTF-8 Wins

| Feature | UTF-8 | ISO-8859-1 | Windows-1252 | Shift_JIS | GB2312 |
|---|---|---|---|---|---|
| Covers all Unicode characters | Yes | No (256 only) | No (256 only) | No (Japanese only) | No (Chinese only) |
| ASCII compatible | Yes | Yes | Yes | No | No |
| Variable width | Yes (1-4 bytes) | No (1 byte) | No (1 byte) | Yes (1-2 bytes) | Yes (1-2 bytes) |
| Used by Google as default | Yes | No | No | No | No |
| Used by modern Linux | Yes | No | No | No | No |
| Compatible with all browsers | Yes | Mostly | Mostly | Older browsers | Older browsers |
| Recommended for the web | Yes | No (deprecated) | No (deprecated) | No (regional only) | No (regional only) |

### 10.3 The Single Encoding Strategy

For Bubbles in 2026:

* **HTML files**: UTF-8.
* **CSS files**: UTF-8.
* **JavaScript files**: UTF-8.
* **JSON files**: UTF-8 (RFC 8259 mandates UTF-8).
* **Database**: UTF-8.
* **Form submissions**: UTF-8.
* **Log files**: UTF-8.
* **Configuration files**: UTF-8.
* **Source code comments**: UTF-8.

No mixing. No exceptions.

### 10.4 The Legacy Migration

If you inherit a site using a legacy encoding:

1. **Identify all files using non UTF-8 encoding.**

```bash
# Find files with non UTF-8 encoding
find /var/www/sites/example.com/ -type f -name "*.html" | while read f; do
    ENC=$(file -i "$f" | grep -oE "charset=[^ ;]+")
    if [ "$ENC" != "charset=utf-8" ] && [ "$ENC" != "charset=us-ascii" ]; then
        echo "$f: $ENC"
    fi
done
```

2. **Convert each file to UTF-8.**

```bash
# Convert Windows-1252 to UTF-8
iconv -f WINDOWS-1252 -t UTF-8 input.html > output.html

# Convert ISO-8859-1 to UTF-8
iconv -f ISO-8859-1 -t UTF-8 input.html > output.html
```

3. **Update all meta charset declarations.**

```bash
# Replace any non UTF-8 charset declarations
find /var/www/sites/example.com/ -name "*.html" -exec \
    sed -i 's/charset="ISO-8859-1"/charset="utf-8"/gI; s/charset="Windows-1252"/charset="utf-8"/gI' {} \;
```

4. **Update nginx config.**

```nginx
charset utf-8;
```

5. **Verify.**

```bash
# All files should now be UTF-8
find /var/www/sites/example.com/ -name "*.html" -exec file -i {} \; | grep -v "charset=utf-8\|charset=us-ascii"
# Should produce no output
```

---

## 11. THE CONTENT-TYPE HTTP HEADER INTERACTION

The HTTP Content-Type header (covered in detail in framework-http-content-headers.md Section 5) takes precedence over the HTML meta tag.

### 11.1 The Precedence Rule

```
HTTP/2 200 OK
Content-Type: text/html; charset=ISO-8859-1

<html>
<head>
<meta charset="utf-8">
```

Result: browser uses **ISO-8859-1** (HTTP header wins).

### 11.2 The Bubbles Nginx Configuration

The recommended nginx configuration:

```nginx
# /etc/nginx/nginx.conf or in http context

http {
    charset utf-8;
    charset_types text/html text/css text/xml text/plain application/json application/javascript application/xml application/atom+xml application/rss+xml;

    # ... other directives
}
```

The `charset utf-8;` directive tells nginx to add `; charset=utf-8` to the Content-Type for text responses. The `charset_types` directive lists which MIME types should have charset added.

### 11.3 The Verification

```bash
# Verify HTTP header
curl -sI https://example.com/ | grep -i content-type
# Expected: content-type: text/html; charset=utf-8

# Test multiple file types
for url in / /style.css /script.js /api/data.json; do
    CT=$(curl -sI "https://example.com$url" | grep -i content-type)
    echo "$url: $CT"
done
```

### 11.4 The FastAPI Default

FastAPI defaults to UTF-8 for all JSON responses:

```
HTTP/2 200 OK
Content-Type: application/json
```

Note: JSON does not need a charset parameter (RFC 8259 mandates UTF-8 for JSON). But for HTML responses from FastAPI:

```python
from fastapi.responses import HTMLResponse

@app.get("/")
async def homepage():
    return HTMLResponse(content="<html>...</html>", media_type="text/html; charset=utf-8")
```

Or rely on FastAPI's default (which adds charset for HTMLResponse).

---

## 12. FORM SUBMISSION ENCODING (ACCEPT-CHARSET)

Forms submit data using the page's encoding by default. The `accept-charset` attribute on `<form>` allows explicit control.

### 12.1 The Default Behavior

```html
<!doctype html>
<html>
<head>
    <meta charset="utf-8">
</head>
<body>
    <form action="/submit" method="post">
        <input name="message">
        <button type="submit">Send</button>
    </form>
</body>
</html>
```

The form submits as UTF-8 (matches the page encoding).

### 12.2 The Explicit Declaration

```html
<form action="/submit" method="post" accept-charset="utf-8">
    <input name="message">
    <button type="submit">Send</button>
</form>
```

The form submits as UTF-8 regardless of page encoding. Useful when:

* The page might be served with conflicting encoding declarations (legacy templates).
* Defense in depth.
* The form is loaded in iframes from other pages with different encodings.

### 12.3 The Server Side Reception

FastAPI receives form data as UTF-8 by default:

```python
from fastapi import FastAPI, Form

app = FastAPI()

@app.post("/submit")
async def submit(message: str = Form(...)):
    # message is a Python str (Unicode), correctly decoded from UTF-8
    return {"received": message}
```

No additional encoding handling needed in FastAPI when the form submits UTF-8.

### 12.4 The Verification

Test with special characters:

```bash
# Submit form with non ASCII character
curl -X POST -d "message=Mājeej" -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    https://example.com/submit

# Expected: server receives "Mājeej" correctly
```

### 12.5 The Common Failure Mode

A form on a UTF-8 page with no explicit `accept-charset` works fine. A form on a page with conflicting encoding declarations (HTTP says ISO-8859-1, HTML says UTF-8) may submit as either depending on browser interpretation. The fix:

1. Align HTTP and HTML to both say UTF-8.
2. Add `accept-charset="utf-8"` to the form for defense in depth.

---

## 13. DATABASE ENCODING (THE POSTGRESQL PATTERN)

PostgreSQL database encoding must align with the application stack's UTF-8 expectations.

### 13.1 The Server Encoding

PostgreSQL has multiple encoding settings:

* **server_encoding**: encoding of the entire server (compile time setting; rarely changed).
* **client_encoding**: encoding the client uses for sending/receiving (per session).
* **database encoding**: encoding for a specific database (set at database creation).

For Bubbles, all three should be UTF-8.

### 13.2 The Verification

```bash
# Server encoding (compile time)
sudo -u postgres psql -c "SHOW server_encoding;"
# Expected: server_encoding | UTF8

# Database specific encoding
sudo -u postgres psql -d bubbles -c "SELECT pg_encoding_to_char(encoding) FROM pg_database WHERE datname = 'bubbles';"
# Expected: UTF8

# Client encoding (current session)
sudo -u postgres psql -c "SHOW client_encoding;"
# Expected: client_encoding | UTF8

# All databases on the server
sudo -u postgres psql -c "SELECT datname, pg_encoding_to_char(encoding) FROM pg_database;"
# Every database should be UTF8
```

### 13.3 The Connection String

FastAPI connection should explicitly declare client encoding:

```python
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/bubbles?client_encoding=utf8"

# Or for psycopg2/sync clients:
DATABASE_URL = "postgresql://user:pass@localhost/bubbles?client_encoding=UTF8"
```

### 13.4 The Migration From SQL_ASCII

If a database was created with the historical SQL_ASCII encoding, all character data is stored as raw bytes (no encoding awareness). Migration to UTF-8 requires:

```bash
# 1. Dump existing database
sudo -u postgres pg_dump -d bubbles --no-owner --no-acl > /tmp/bubbles.sql

# 2. Examine the dump for encoding hints
file -i /tmp/bubbles.sql

# 3. If the dump is mixed encoding, you may need iconv:
iconv -f WINDOWS-1252 -t UTF-8 /tmp/bubbles.sql > /tmp/bubbles-utf8.sql

# 4. Drop and recreate database with UTF-8
sudo -u postgres psql -c "DROP DATABASE bubbles;"
sudo -u postgres psql -c "CREATE DATABASE bubbles WITH ENCODING='UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE=template0;"

# 5. Restore the converted dump
sudo -u postgres psql -d bubbles < /tmp/bubbles-utf8.sql

# 6. Verify
sudo -u postgres psql -d bubbles -c "SELECT pg_encoding_to_char(encoding) FROM pg_database WHERE datname = 'bubbles';"
```

This is destructive and downtime requires. Schedule during a maintenance window.

### 13.5 The Locale Consideration

PostgreSQL uses LC_COLLATE and LC_CTYPE for sorting and case conversion. For UTF-8 content:

```bash
# Check current locale
sudo -u postgres psql -c "SHOW lc_collate;"
sudo -u postgres psql -c "SHOW lc_ctype;"

# Expected:
# lc_collate | en_US.UTF-8
# lc_ctype   | en_US.UTF-8
```

For Marshallese content, English locale collation may produce unexpected sort orders. Specialized collations are beyond the scope of this framework, but `en_US.UTF-8` correctly handles UTF-8 bytes without corruption (even if the sort order is not linguistically perfect for Marshallese).

---

## 14. THE MOJIBAKE PROBLEM (DEBUGGING BROKEN CHARACTERS)

Mojibake is the technical term for character corruption due to encoding mismatch.

### 14.1 The Common Symptoms

| What you wrote | What appears | Likely cause |
|---|---|---|
| `'` (right single quote) | `â€™` | UTF-8 bytes interpreted as Windows-1252 |
| `"` `"` (curly quotes) | `â€œ` `â€` | UTF-8 bytes interpreted as Windows-1252 |
| `é` | `Ã©` | UTF-8 bytes interpreted as ISO-8859-1 |
| `ā` (Marshallese long a) | `Ä?` or `?` | UTF-8 not preserved through some layer |
| `ñ` | `Ã±` | UTF-8 interpreted as ISO-8859-1 |
| `Ā` | `??` | UTF-8 bytes lost or replaced |

### 14.2 The Diagnostic Method

When you see mojibake, ask: "what encoding produces these specific replacement characters?"

The pattern `â€™` is the classic signature of UTF-8 bytes (`0xE2 0x80 0x99` for the right single quote) interpreted as Windows-1252 (`â`, `€`, `™`).

The pattern `Ã©` is the signature of UTF-8 bytes (`0xC3 0xA9` for é) interpreted as ISO-8859-1 (`Ã`, `©`).

The pattern `?` (literal question marks) usually means the character was lost during conversion (not just misinterpreted).

### 14.3 The Layer Investigation

When mojibake appears, work through the five layers:

```bash
# Layer 1: HTTP
curl -sI "$URL" | grep -i content-type

# Layer 2: HTML
curl -s "$URL" | head -c 1024 | grep -i charset

# Layer 3: file
file -i "$FILE"

# Layer 4: database (if dynamic content)
psql -d bubbles -c "SHOW client_encoding; SHOW server_encoding;"
# Then check the actual bytes:
psql -d bubbles -c "SELECT title, octet_length(title), length(title) FROM articles WHERE id = 1;"

# Layer 5: form
# Check accept-charset attribute and Content-Type of form submission
```

Find the mismatch.

### 14.4 The Database Specific Diagnostic

For mojibake in database content:

```sql
-- Check the actual bytes stored
SELECT
    title,
    octet_length(title) AS byte_count,
    length(title) AS char_count,
    encode(convert_to(title, 'UTF8'), 'hex') AS hex_bytes
FROM articles
WHERE id = 1;
```

If byte_count > char_count: characters are multi byte (likely UTF-8 working correctly).
If they're equal: characters are single byte (possibly ASCII, possibly corrupt).
The hex bytes show you exactly what's stored.

### 14.5 The Repair Strategies

**Repair option 1: re convert content from known wrong encoding.**

```python
# If content was stored as UTF-8 bytes but interpreted as Latin-1:
broken_text = "Mojibake â€™ string"
fixed = broken_text.encode('latin-1').decode('utf-8')
# fixed: "Mojibake ' string"
```

**Repair option 2: re fetch from source.**

If the original content can be obtained from a clean source (backup, original document, etc), reload it through a properly configured pipeline.

**Repair option 3: regex replace common mojibake patterns.**

```python
# Map common mojibake patterns to correct characters
MOJIBAKE_FIXES = {
    'â€™': "'",
    'â€œ': '"',
    'â€': '"',
    'â€"': '...',  # Use a literal ellipsis or three dots
    'Ã©': 'é',
    'Ã¨': 'è',
    'Ã±': 'ñ',
}

def fix_mojibake(text):
    for wrong, right in MOJIBAKE_FIXES.items():
        text = text.replace(wrong, right)
    return text
```

Use repair option 1 when possible; it is the cleanest. Use option 3 as a last resort and verify each fix.

---

## 15. SPECIAL CHARACTERS: HTML ENTITIES VS RAW UTF-8

HTML supports two ways to encode special characters: HTML entities (`&amp;`, `&copy;`, etc) and raw UTF-8 bytes.

### 15.1 The Entity Approach (Legacy)

```html
<p>Copyright &copy; 2026 ThatDeveloperGuy. All rights reserved.</p>
<p>Use the &amp; character carefully.</p>
<p>The Greek letter alpha is &alpha;.</p>
```

Works in any encoding (entities are pure ASCII).

### 15.2 The Raw UTF-8 Approach (Modern)

```html
<p>Copyright © 2026 ThatDeveloperGuy. All rights reserved.</p>
<p>Use the & character carefully.</p>
<p>The Greek letter alpha is α.</p>
```

Works when the document is correctly declared as UTF-8.

### 15.3 The Required Entities

Five characters MUST be entities in HTML content (they would otherwise be interpreted as markup):

| Character | Entity | Hex |
|---|---|---|
| `&` | `&amp;` | `&#x26;` |
| `<` | `&lt;` | `&#x3C;` |
| `>` | `&gt;` | `&#x3E;` |
| `"` (in attribute values) | `&quot;` | `&#x22;` |
| `'` (in attribute values) | `&apos;` | `&#x27;` |

For all other characters, raw UTF-8 is preferred for readability.

### 15.4 The Marshallese Example

Marshallese script uses diacritics like macrons (ā, ē, ō, ū) and cedillas (ļ, ņ).

Entity approach (legacy, verbose):

```html
<h1>K&#x16D;ņa&#x101; &#x14D;rraan</h1>
```

Raw UTF-8 approach (modern, readable):

```html
<h1>Kūņaā ōrraan</h1>
```

The raw UTF-8 approach is the standard for modern content. The entity approach is rare in 2026 except for the five reserved characters.

### 15.5 The Verification

Open the rendered page in a browser. View source. The raw UTF-8 characters should appear as themselves (not as entities and not as mojibake).

```bash
# Check that raw UTF-8 characters are preserved in source
curl -s https://marshallese-voices.example/ | grep -oE 'Kūņaā ōrraan'
# Expected: matches the actual UTF-8 bytes
```

---

## 16. THE SECURITY IMPLICATIONS (UTF-7 SNIFFING, CHARSET BASED XSS)

Missing charset declarations create security risks.

### 16.1 The Historical UTF-7 XSS Attack

Pre 2010, attackers could exploit charset sniffing to bypass XSS filters:

1. Server sends HTML without explicit charset.
2. Server sanitizes input, removing `<script>` and similar.
3. Attacker submits content encoded as UTF-7.
4. Browser sniffs the content, detects UTF-7 patterns, decodes as UTF-7.
5. After decoding, the previously safe content contains `<script>` (which the server's UTF-8 based sanitizer did not see).
6. XSS executes.

### 16.2 The Modern Mitigation

Modern browsers (post 2010) have substantially reduced UTF-7 sniffing, but the defense in depth is:

1. **Always declare charset.** No sniffing means no attack.
2. **Set `X-Content-Type-Options: nosniff`** (covered in framework-http-security-headers.md). Prevents content type sniffing more broadly.
3. **Strict input sanitization.** Even with charset declared, sanitize all user input.

### 16.3 The OWASP Recommendation

Per OWASP Cross Site Scripting Prevention Cheat Sheet:

> "Set the page encoding explicitly to UTF-8 with a Content-Type response header AND a meta tag in the page itself."

Belt and suspenders: HTTP header AND HTML meta tag. The HTML meta tag is the fallback if the HTTP header is somehow stripped.

### 16.4 The Bubbles Configuration

In nginx:

```nginx
# Charset declaration
charset utf-8;

# Prevent MIME type sniffing
add_header X-Content-Type-Options "nosniff" always;
```

In HTML:

```html
<meta charset="utf-8">
```

Both layers declare; no sniffing required; attack surface reduced.

---

## 17. THE MARSHALLESE-VOICES CASE STUDY (WHY THIS MATTERS)

The Marshallese-Voices client is the canonical example in the Bubbles portfolio of why UTF-8 alignment matters operationally.

### 17.1 The Linguistic Context

Marshallese (Kajin M̧ajeļ) is the official language of the Marshall Islands. The orthography uses:

* **Macron diacritics**: ā, ē, ī, ō, ū (on the standard A, E, I, O, U vowels).
* **Cedilla diacritics**: ļ (on L), ņ (on N), m̧ (on M).
* **Combining characters**: some Marshallese characters use combining diacritics in Unicode.

None of these characters exist in ASCII, ISO-8859-1, or Windows-1252. They REQUIRE UTF-8 to be represented correctly.

### 17.2 The Operational Implications

For the Marshallese-Voices Bubbles client:

1. **HTML must be UTF-8.** Every page declares `<meta charset="utf-8">` as the first tag in `<head>`.
2. **HTTP Content-Type must include charset=utf-8.** nginx `charset utf-8;` directive set.
3. **Files must be saved as UTF-8 without BOM.** Joseph's editor configuration ensures this.
4. **Database must be UTF-8.** PostgreSQL database for the client was verified UTF-8 at creation.
5. **Forms must accept UTF-8.** All forms have `accept-charset="utf-8"` (defense in depth).
6. **Templates render UTF-8 strings directly.** No HTML entity escaping for Marshallese characters.

### 17.3 The Verification

```bash
# 1. HTTP layer
curl -sI https://marshallese-voices.example/ | grep -i content-type
# Expected: content-type: text/html; charset=utf-8

# 2. HTML layer
curl -s https://marshallese-voices.example/ | head -c 1024 | grep -i charset
# Expected: <meta charset="utf-8">

# 3. Sample page with Marshallese content
curl -s https://marshallese-voices.example/about | grep -oE 'M̧ajeļ' | head -1
# Expected: matches (UTF-8 preserved end to end)

# 4. Database (assumes a content table with Marshallese text)
sudo -u postgres psql -d marshallese -c "SELECT title FROM articles WHERE id = 1;"
# Should display Marshallese characters correctly
```

### 17.4 The Failure Mode

If UTF-8 alignment broke at any layer:

* **HTTP layer wrong**: browser uses HTTP encoding; HTML meta is ignored.
* **HTML layer wrong**: browser falls back to sniffing or default encoding.
* **File layer wrong**: characters in source code already corrupt.
* **Database layer wrong**: characters round trip but get lost on retrieval.
* **Form layer wrong**: submitted user content is corrupt before storage.

For the Marshallese-Voices client, every layer's correctness is verified at deployment and audited quarterly.

### 17.5 The Lesson For Other Bubbles Clients

Even sites that only display English content benefit from full UTF-8 alignment:

* Editorial sites pull content from sources (Word, Google Docs, etc) that produce smart quotes and em dashes (though Joseph removes the em dashes; the smart quotes remain).
* User generated content (comments, form submissions, reviews) may include emoji, accented characters, names with diacritics.
* International visitors may submit forms with characters in their native script.
* Customer data (names, addresses) often includes characters outside ASCII.

The Marshallese-Voices pattern is the gold standard. The same configuration applied to every Bubbles client site provides defense against the surprise UTF-8 content scenario.

---

## 18. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 18.1 Standard HTML page (the canonical pattern)

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Page Title</title>
    <!-- everything else -->
</head>
<body>
    <!-- content -->
</body>
</html>
```

### 18.2 Page with international content (Marshallese-Voices pattern)

```html
<!doctype html>
<html lang="mh">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Kajin M̧ajeļ</title>
</head>
<body>
    <h1>Kūņaā ōrraan</h1>
    <p>Kōn an aolep armej rōj baar kōj̧olok an...</p>
</body>
</html>
```

### 18.3 Form with explicit UTF-8 encoding

```html
<form action="/submit" method="post" accept-charset="utf-8" enctype="multipart/form-data">
    <label>Name: <input name="name" type="text"></label>
    <label>Message: <textarea name="message"></textarea></label>
    <button type="submit">Send</button>
</form>
```

### 18.4 nginx standard charset configuration

```nginx
# /etc/nginx/nginx.conf

http {
    # Set UTF-8 for text content types
    charset utf-8;
    charset_types
        text/html
        text/css
        text/xml
        text/plain
        application/json
        application/javascript
        application/xml
        application/atom+xml
        application/rss+xml;

    # Security: prevent MIME sniffing
    # (Set per server or globally; covered in framework-http-security-headers.md)

    # ... other directives
}
```

Reload:

```bash
nginx -t && systemctl reload nginx
```

### 18.5 PostgreSQL database creation with UTF-8

```sql
-- Create database with explicit UTF-8 encoding
CREATE DATABASE bubbles
    WITH ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0;
```

### 18.6 FastAPI HTMLResponse with explicit charset

```python
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

app = FastAPI()

@app.get("/", response_class=HTMLResponse)
async def homepage():
    return HTMLResponse(
        content="""<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Hello</title>
</head>
<body>
    <h1>Hello, world!</h1>
</body>
</html>""",
        # FastAPI defaults to text/html; charset=utf-8 for HTMLResponse
    )
```

### 18.7 FastAPI database connection with UTF-8

```python
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/bubbles?client_encoding=utf8"

from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10,
)
```

### 18.8 robots.txt as UTF-8 (without BOM)

```bash
# Save robots.txt as UTF-8 without BOM
cat > /var/www/sites/example.com/robots.txt <<'EOF'
User-agent: *
Disallow: /admin/

Sitemap: https://example.com/sitemap.xml
EOF

# Verify encoding
file -i /var/www/sites/example.com/robots.txt
# Expected: text/plain; charset=utf-8 (no BOM mention)
```

### 18.9 Convert legacy file to UTF-8

```bash
# Single file
iconv -f WINDOWS-1252 -t UTF-8 /path/to/legacy.html > /path/to/utf8.html

# Bulk convert all files in a directory
find /var/www/sites/example.com/ -name "*.html" -type f -exec bash -c '
    ENC=$(file -i "$1" | grep -oE "charset=[^ ;]+" | cut -d= -f2)
    if [ "$ENC" != "utf-8" ] && [ "$ENC" != "us-ascii" ]; then
        iconv -f "$ENC" -t UTF-8 "$1" > "$1.utf8" && mv "$1.utf8" "$1"
        echo "Converted: $1 from $ENC to UTF-8"
    fi
' _ {} \;
```

### 18.10 Strip BOM from files

```bash
# Single file
sed -i '1s/^\xEF\xBB\xBF//' /path/to/file.html

# Bulk strip BOM from all HTML files
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    sed -i '1s/^\xEF\xBB\xBF//' {} \;

# Verify no BOM remains
find /var/www/sites/example.com/ -name "*.html" -type f -exec file -i {} \; | grep "byte order mark"
# Should produce no output
```

---

## 19. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles charset configuration covering all five layers.

### 19.1 HTTP Layer (nginx)

```nginx
# /etc/nginx/nginx.conf (http context)

http {
    # ...

    # Charset declaration (UTF-8 for text types)
    charset utf-8;
    charset_types
        text/html
        text/css
        text/xml
        text/plain
        application/json
        application/javascript
        application/xml
        application/atom+xml
        application/rss+xml;

    # ... other directives
}
```

### 19.2 HTML Layer (template)

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <!-- everything else follows -->
</head>
<body>
    <!-- content -->
</body>
</html>
```

### 19.3 File System Layer

Editor configuration for hand coded sites:

* **VS Code**: settings.json contains `"files.encoding": "utf8"`.
* **vim**: `.vimrc` contains `set encoding=utf-8` and `set fileencoding=utf-8`.
* **Editorconfig**: `.editorconfig` in project root contains `charset = utf-8`.

Sample `.editorconfig`:

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
trim_trailing_whitespace = true
insert_final_newline = true
```

### 19.4 Database Layer (PostgreSQL)

```bash
# Verify all databases use UTF-8
sudo -u postgres psql -c "SELECT datname, pg_encoding_to_char(encoding) FROM pg_database WHERE datistemplate = false;"
# All entries should show UTF8

# For any new database
sudo -u postgres psql -c "CREATE DATABASE newclient WITH ENCODING='UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE=template0;"
```

### 19.5 Form Layer

```html
<form action="/submit" method="post" accept-charset="utf-8">
    <!-- form fields -->
</form>
```

### 19.6 Verification Script

```bash
#!/bin/bash
# /usr/local/bin/charset-audit.sh

URL=$1
DOMAIN=$(echo "$URL" | sed -E 's|https?://([^/]+).*|\1|')

echo "=== Charset audit for $URL ==="

# Layer 1: HTTP
CT=$(curl -sI "$URL" | grep -i content-type | tr -d '\r')
echo ""
echo "Layer 1 (HTTP): $CT"
[[ "$CT" =~ "charset=utf-8" ]] && echo "  Status: OK" || echo "  Status: MISSING charset=utf-8"

# Layer 2: HTML
echo ""
HEAD_BYTES=$(curl -s "$URL" | head -c 1024)
if echo "$HEAD_BYTES" | grep -qi 'charset="utf-8"\|charset=utf-8'; then
    POS=$(echo "$HEAD_BYTES" | grep -bo "charset" | head -1 | cut -d: -f1)
    echo "Layer 2 (HTML): meta charset found at byte $POS"
    [ "$POS" -lt 1024 ] && echo "  Status: OK (within 1024 bytes)" || echo "  Status: TOO LATE"
else
    echo "Layer 2 (HTML): MISSING meta charset in first 1024 bytes"
fi

# Layer 3: file (if domain matches a known Bubbles site)
SITE_DIR="/var/www/sites/$DOMAIN"
if [ -d "$SITE_DIR" ]; then
    echo ""
    echo "Layer 3 (file):"
    find "$SITE_DIR" -name "*.html" -type f | head -3 | while read f; do
        ENC=$(file -i "$f" | grep -oE "charset=[^ ;]+")
        echo "  $f: $ENC"
    done
fi

# Layer 4: database
echo ""
DB_ENC=$(sudo -u postgres psql -At -c "SHOW server_encoding;" 2>/dev/null)
echo "Layer 4 (db): server_encoding = $DB_ENC"
[ "$DB_ENC" = "UTF8" ] && echo "  Status: OK" || echo "  Status: NOT UTF8"

# Layer 5: forms
echo ""
FORMS=$(curl -s "$URL" | grep -oE 'accept-charset="[^"]+"' | head -3)
if [ -n "$FORMS" ]; then
    echo "Layer 5 (forms): $FORMS"
else
    echo "Layer 5 (forms): no accept-charset attributes (relies on page encoding)"
fi

echo ""
echo "=== End audit ==="
```

Make executable and use:

```bash
sudo chmod +x /usr/local/bin/charset-audit.sh
charset-audit.sh https://example.com/
```

---

## 20. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### HTML layer

1. [ ] Every HTML page has `<meta charset="utf-8">` declared.
2. [ ] Declaration is the FIRST tag inside `<head>`.
3. [ ] Declaration is within the first 1024 bytes of the document.
4. [ ] Declaration uses the HTML5 short form (not the legacy http-equiv form).
5. [ ] Declaration uses lowercase: `<meta charset="utf-8">` not `UTF-8`.

### HTTP layer (nginx)

6. [ ] `charset utf-8;` directive set in nginx (http or server context).
7. [ ] `charset_types` includes all text types served by the site.
8. [ ] Content-Type response header includes `charset=utf-8` for HTML.
9. [ ] Verify with `curl -sI` that Content-Type matches expected.

### File system layer

10. [ ] All HTML files saved as UTF-8 (verified via `file -i`).
11. [ ] No BOM in any HTML file.
12. [ ] All CSS files UTF-8 without BOM.
13. [ ] All JavaScript files UTF-8 without BOM.
14. [ ] All JSON files UTF-8 without BOM.
15. [ ] Editor configuration (`.editorconfig` or equivalent) enforces UTF-8.

### Database layer (PostgreSQL)

16. [ ] `server_encoding = UTF8` per `SHOW server_encoding;`.
17. [ ] Every database uses UTF8 encoding (`pg_database` table).
18. [ ] `client_encoding` set to UTF8 in FastAPI connection string.
19. [ ] Sample query returns UTF-8 characters correctly.
20. [ ] LC_COLLATE and LC_CTYPE set to a UTF-8 locale.

### Form layer

21. [ ] Forms have `accept-charset="utf-8"` attribute (defense in depth).
22. [ ] FastAPI Form processing receives UTF-8 correctly.
23. [ ] Form submissions with special characters round trip correctly.

### Content layer

24. [ ] No mojibake visible on any page (smart quotes, em dashes, accented characters).
25. [ ] Page titles in browser tab show correctly.
26. [ ] Page titles in Google search results show correctly.
27. [ ] Meta descriptions with special characters preserved.
28. [ ] Special characters render correctly in social media share previews.

### Marshallese-Voices specific (or any non Latin content)

29. [ ] Macron diacritics (ā, ē, ī, ō, ū) display correctly.
30. [ ] Cedilla diacritics (ļ, ņ, m̧) display correctly.
31. [ ] Combining characters preserved through copy paste.
32. [ ] Search functionality finds Marshallese terms (DB collation supports it).

### Security

33. [ ] `X-Content-Type-Options: nosniff` header set.
34. [ ] No UTF-7 vulnerability surface (charset always declared).
35. [ ] CSP header set (covered in framework-http-security-headers.md).

### Cross cutting

36. [ ] robots.txt file UTF-8 encoded.
37. [ ] sitemap.xml UTF-8 encoded.
38. [ ] All template includes/partials UTF-8 encoded.
39. [ ] All log files UTF-8.
40. [ ] All configuration files UTF-8.

### Tooling and process

41. [ ] Pre commit hook (or CI step) verifies UTF-8 encoding of source files.
42. [ ] Editor settings documented for new contributors.
43. [ ] Database backup/restore preserves UTF-8 through round trip.
44. [ ] `charset-audit.sh` script in `/usr/local/bin/`.

### Verification commands tested

45. [ ] `curl -sI URL | grep -i content-type` returns expected.
46. [ ] `curl -s URL | head -c 1024 | grep charset` finds declaration.
47. [ ] `file -i FILE` shows UTF-8 for all source files.
48. [ ] `psql -c "SHOW server_encoding;"` returns UTF8.
49. [ ] `nginx -t` passes.
50. [ ] `nginx -t && systemctl reload nginx` applied after any config changes.

A site that passes all 50 has correctly configured UTF-8 across all five operational layers and is resilient to surprise content with special characters.

---

## 21. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: `<meta charset>` placed after other tags in head.**
Symptom: characters render as mojibake in some browsers.
Why it breaks: charset declaration pushed past the 1024 byte cutoff.
Fix: place `<meta charset="utf-8">` as the FIRST tag inside `<head>`.

**Pitfall 2: HTTP Content-Type contains charset that differs from HTML.**
Symptom: browser ignores HTML meta tag; uses HTTP charset (which is wrong).
Why it breaks: HTTP wins over HTML in the encoding cascade.
Fix: ensure nginx `charset utf-8;` is set; both layers say UTF-8.

**Pitfall 3: BOM in HTML file.**
Symptom: weird whitespace at top of page, PHP "headers already sent" errors, validator complaints.
Why it breaks: BOM bytes appear in the rendered document.
Fix: strip BOM via `sed -i '1s/^\xEF\xBB\xBF//' file.html`.

**Pitfall 4: Database created with SQL_ASCII encoding.**
Symptom: special characters round trip through some queries but break on others.
Why it breaks: SQL_ASCII stores raw bytes with no encoding awareness.
Fix: dump, drop, recreate database with UTF8 encoding, restore.

**Pitfall 5: File saved as ISO-8859-1 with UTF-8 meta tag.**
Symptom: special characters in the file appear as mojibake.
Why it breaks: file bytes don't match declared encoding.
Fix: convert file with `iconv -f ISO-8859-1 -t UTF-8 file.html > new.html`.

**Pitfall 6: Mojibake fixed by adding more encoding declarations.**
Symptom: mojibake persists despite adding more `<meta charset>` tags.
Why it breaks: the issue is at a different layer; more meta tags do not help.
Fix: identify the actual broken layer (HTTP, file, database) and fix it there.

**Pitfall 7: Form submission corrupts special characters.**
Symptom: form data with é, ā, ñ comes through as â€™, Ä?, Ã±.
Why it breaks: form encoding inconsistent with page or server.
Fix: add `accept-charset="utf-8"` to form; verify FastAPI receives UTF-8.

**Pitfall 8: nginx default Content-Type without charset.**
Symptom: HTTP response is `Content-Type: text/html` (no charset).
Why it breaks: browser falls back to sniffing.
Fix: nginx `charset utf-8;` directive in http or server context.

**Pitfall 9: Smart quotes from Microsoft Word in editorial content.**
Symptom: copy paste from Word produces â€œ and â€ in published content.
Why it breaks: Word's smart quotes are Windows-1252 specific bytes interpreted as UTF-8 (or vice versa).
Fix: ensure entire pipeline is UTF-8; verify Word saved as UTF-8 before paste, or normalize smart quotes in content sanitization.

**Pitfall 10: Marshallese (or other non Latin) text shows as ??? in production.**
Symptom: development works, production shows question marks.
Why it breaks: a layer in the pipeline (often database or file) is not UTF-8.
Fix: audit all five layers; find the mismatch.

**Pitfall 11: Emoji in titles or content stripped or replaced.**
Symptom: 🎉 appears as ?? or is missing.
Why it breaks: a layer is not 4 byte UTF-8 capable (some databases need utf8mb4 in MySQL terms; PostgreSQL UTF8 handles all).
Fix: verify all layers handle 4 byte UTF-8.

**Pitfall 12: HTML entities used for characters that should be raw UTF-8.**
Symptom: source code is bloated with `&#xNNNN;` entities for normal accented characters.
Why it breaks: legacy practice; not wrong but inefficient.
Fix: declare UTF-8, use raw characters. Reserve entities for the five mandatory characters (&, <, >, ", ').

**Pitfall 13: charset declaration in `<meta http-equiv>` form only (no short form).**
Symptom: works but takes more bytes; some validators warn.
Why it breaks: legacy HTML4 syntax; verbose.
Fix: replace with `<meta charset="utf-8">`.

**Pitfall 14: nginx adds charset to binary files (images, PDFs).**
Symptom: image MIME types include unexpected charset parameter.
Why it breaks: `charset utf-8;` applies to types in `charset_types`; if image types are accidentally included.
Fix: ensure `charset_types` lists only text MIME types.

**Pitfall 15: Inconsistent encoding declarations across templates.**
Symptom: some pages have meta charset, others don't.
Why it breaks: inconsistent template hygiene.
Fix: enforce template includes; every layout has `<meta charset="utf-8">` as first head element.

---

## 22. DIAGNOSTIC COMMANDS

Reference of every command useful for charset investigation.

### Inspect a single URL

```bash
# Check HTTP Content-Type
curl -sI https://example.com/ | grep -i content-type

# Check HTML meta charset (first 1024 bytes)
curl -s https://example.com/ | head -c 1024 | grep -i charset

# Check exact byte position of charset declaration
curl -s https://example.com/ | head -c 2000 | grep -bn 'charset' | head -3

# Verify response body is valid UTF-8
curl -s https://example.com/ | iconv -f UTF-8 -t UTF-8 -c > /dev/null && echo "OK" || echo "BROKEN"
```

### Inspect a file

```bash
# Check encoding
file -i /var/www/sites/example.com/index.html

# Check for BOM
hexdump -C /var/www/sites/example.com/index.html | head -1
# A BOM is: ef bb bf at the start

# Check character composition
awk '{ for (i=1; i<=length($0); i++) { c=substr($0,i,1); ord=ord ? ord c : ord c; if (length(c) > 1) print c, "MULTIBYTE" } }' file.html
```

### Bulk file audit

```bash
# Find all non UTF-8 files
find /var/www/sites/ -type f \( -name "*.html" -o -name "*.css" -o -name "*.js" \) | while read f; do
    ENC=$(file -i "$f" | grep -oE "charset=[^ ;]+")
    if [ "$ENC" != "charset=utf-8" ] && [ "$ENC" != "charset=us-ascii" ]; then
        echo "$f: $ENC"
    fi
done

# Find files with BOM
find /var/www/sites/ -type f -exec file -i {} \; | grep "byte order mark"
```

### Database checks

```bash
# Server encoding
sudo -u postgres psql -c "SHOW server_encoding;"

# All databases
sudo -u postgres psql -c "SELECT datname, pg_encoding_to_char(encoding), datcollate, datctype FROM pg_database WHERE datistemplate = false;"

# Sample query with special characters
sudo -u postgres psql -d bubbles -c "SELECT title FROM articles WHERE title ~ '[āēīōū]' LIMIT 5;"

# Check byte vs character length to detect multi byte content
sudo -u postgres psql -d bubbles -c "SELECT id, title, octet_length(title) AS bytes, length(title) AS chars FROM articles WHERE octet_length(title) != length(title) LIMIT 10;"
```

### Convert files

```bash
# Convert single file
iconv -f WINDOWS-1252 -t UTF-8 input.html > output.html

# Detect encoding (uchardet must be installed: apt install uchardet)
uchardet input.html

# Strip BOM
sed -i '1s/^\xEF\xBB\xBF//' file.html
```

### Test with special characters

```bash
# Submit form with non ASCII
curl -X POST -d "name=Mojiba ké" -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    https://example.com/submit
echo "Status: $?"

# Inspect form submission encoding
curl -s -d "name=test" -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    https://example.com/submit | grep -i charset
```

### Apply nginx changes

```bash
nginx -t && systemctl reload nginx
```

### Editor and tooling verification

```bash
# Verify .editorconfig exists and sets charset
cat /var/www/sites/example.com/.editorconfig | grep -i charset

# Verify vim default
vim -c ":set encoding?" -c ":q" 2>&1 | grep encoding

# Verify VS Code settings (if accessible)
cat ~/.config/Code/User/settings.json | grep -i encoding
```

---

## 23. CROSS-REFERENCES

* [framework-http-content-headers.md](framework-http-content-headers.md) Section 5: the Content-Type HTTP header which carries the charset parameter; the precedence rule (HTTP wins over HTML meta).
* [framework-http-security-headers.md](framework-http-security-headers.md): `X-Content-Type-Options: nosniff` prevents charset based XSS attacks.
* [framework-html-meta-robots.md](framework-html-meta-robots.md): the first framework in the HTML signal track; the meta tag family.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Mojibake in titles and snippets hurts CTR which hurts rankings.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including charset declaration in every template.
* WHATWG HTML Living Standard - character encoding: https://html.spec.whatwg.org/multipage/parsing.html#determining-the-character-encoding
* MDN meta charset: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta#charset
* WHATWG Encoding Standard: https://encoding.spec.whatwg.org/
* OWASP XSS Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
* PostgreSQL Character Set Support: https://www.postgresql.org/docs/current/multibyte.html
* RFC 8259 (JSON, requires UTF-8): https://www.rfc-editor.org/rfc/rfc8259

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The canonical pattern

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">    <!-- FIRST inside head -->
    <!-- everything else follows -->
</head>
```

### The five layer rule

| Layer | Check | Expected |
|---|---|---|
| HTTP | `curl -sI URL | grep content-type` | `text/html; charset=utf-8` |
| HTML | `curl -s URL | head -c 1024 | grep charset` | `<meta charset="utf-8">` |
| File | `file -i path/file.html` | `charset=utf-8` |
| Database | `psql -c "SHOW server_encoding;"` | `UTF8` |
| Form | inspect HTML `<form>` | `accept-charset="utf-8"` |

### Five rules to memorize

1. **UTF-8 everywhere, no exceptions.**
2. **`<meta charset="utf-8">` is the FIRST tag inside `<head>`.**
3. **Place it within the first 1024 bytes (this is automatic if it's first in head).**
4. **No BOM in any file.**
5. **HTTP header wins over HTML meta tag; align both to UTF-8.**

### Five commands every operator should know

```bash
# 1. Check HTTP charset
curl -sI URL | grep -i content-type

# 2. Check HTML charset
curl -s URL | head -c 1024 | grep -i charset

# 3. Check file encoding
file -i /path/to/file.html

# 4. Check database encoding
sudo -u postgres psql -c "SHOW server_encoding;"

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. HTTP and HTML both UTF-8
URL=https://example.com/
HTTP_OK=$(curl -sI "$URL" | grep -i "charset=utf-8" | wc -l)
HTML_OK=$(curl -s "$URL" | head -c 1024 | grep -ic "charset=\"utf-8\"\|charset=utf-8")
[ "$HTTP_OK" = "1" ] && [ "$HTML_OK" -ge "1" ] && echo "OK" || echo "FAIL"

# 2. Sample file is UTF-8 no BOM
FILE=/var/www/sites/example.com/index.html
ENC=$(file -i "$FILE")
[[ "$ENC" =~ "charset=utf-8" ]] && ! [[ "$ENC" =~ "byte order mark" ]] && echo "OK" || echo "FAIL"

# 3. Database is UTF-8
DB_ENC=$(sudo -u postgres psql -At -c "SHOW server_encoding;" 2>/dev/null)
[ "$DB_ENC" = "UTF8" ] && echo "OK" || echo "FAIL"
```

If all three pass AND no mojibake visible in browser AND special character forms round trip correctly, the charset layer is correctly wired across the five operational layers.

---

End of framework-html-meta-charset.md.
